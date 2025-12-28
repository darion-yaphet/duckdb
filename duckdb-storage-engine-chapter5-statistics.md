# DuckDB 存储引擎深度解析：第五章 统计信息系统

## 5.1 概述

统计信息（Statistics）是现代分析型数据库的核心组件，它在查询优化和存储访问中扮演着至关重要的角色。DuckDB 的统计信息系统具有以下主要功能：

1. **Zonemap 过滤**：通过 min/max 值快速跳过不满足查询条件的数据块
2. **基数估计**：为查询优化器提供准确的行数估计
3. **压缩选择**：帮助选择最优的压缩算法
4. **常量检测**：识别常量列以启用特殊优化

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     DuckDB 统计信息系统架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      ColumnStatistics                            │   │
│  │   ┌─────────────────────┐   ┌──────────────────────────────┐    │   │
│  │   │   BaseStatistics    │   │    DistinctStatistics        │    │   │
│  │   │   (min/max/null)    │   │    (HyperLogLog 基数估计)     │    │   │
│  │   └─────────────────────┘   └──────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     StatisticsType 分发                          │   │
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐    │   │
│  │  │  NumericStats   │ │   StringStats   │ │   StructStats   │    │   │
│  │  │  (精确 min/max)  │ │ (截断 min/max)  │ │ (子字段统计)    │    │   │
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘    │   │
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐    │   │
│  │  │   ListStats     │ │   ArrayStats    │ │  GeometryStats  │    │   │
│  │  │  (元素统计)      │ │  (元素统计)     │ │   (边界框)      │    │   │
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        应用层                                    │   │
│  │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │   │
│  │   │ CheckZonemap   │  │ CardinalityEst │  │ CompressionSel │    │   │
│  │   │  (谓词过滤)    │  │  (基数估计)    │  │  (压缩选择)    │    │   │
│  │   └────────────────┘  └────────────────┘  └────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.2 BaseStatistics：统计信息基类

### 5.2.1 类设计

`BaseStatistics` 是所有统计信息的基类，使用 union 来高效存储不同类型的统计数据：

```cpp
// src/include/duckdb/storage/statistics/base_statistics.hpp

class BaseStatistics {
public:
    //! 该列的逻辑类型
    LogicalType type;

    //! 是否可能包含 NULL 值
    bool has_null;
    //! 是否可能包含非 NULL 值
    bool has_no_null;
    //! 近似的 distinct count
    idx_t distinct_count;

    //! 类型特定统计数据的 union
    union {
        NumericStatsData numeric_data;     // 数值类型
        StringStatsData string_data;       // 字符串类型
        GeometryStatsData geometry_data;   // 几何类型
        VariantStatsData variant_data;     // Variant 类型
    } stats_union;

    //! 嵌套类型的子统计信息
    unsafe_unique_array<BaseStatistics> child_stats;
};
```

### 5.2.2 统计类型分发

DuckDB 根据逻辑类型自动选择对应的统计类型：

```cpp
// src/storage/statistics/base_statistics.cpp

StatisticsType BaseStatistics::GetStatsType(const LogicalType &type) {
    if (type.id() == LogicalTypeId::SQLNULL) {
        return StatisticsType::BASE_STATS;
    }
    if (type.id() == LogicalTypeId::GEOMETRY) {
        return StatisticsType::GEOMETRY_STATS;
    }
    if (type.id() == LogicalTypeId::VARIANT) {
        return StatisticsType::VARIANT_STATS;
    }
    switch (type.InternalType()) {
    case PhysicalType::BOOL:
    case PhysicalType::INT8:
    case PhysicalType::INT16:
    case PhysicalType::INT32:
    case PhysicalType::INT64:
    case PhysicalType::UINT8:
    case PhysicalType::UINT16:
    case PhysicalType::UINT32:
    case PhysicalType::UINT64:
    case PhysicalType::INT128:
    case PhysicalType::UINT128:
    case PhysicalType::FLOAT:
    case PhysicalType::DOUBLE:
        return StatisticsType::NUMERIC_STATS;
    case PhysicalType::VARCHAR:
        return StatisticsType::STRING_STATS;
    case PhysicalType::STRUCT:
        return StatisticsType::STRUCT_STATS;
    case PhysicalType::LIST:
        return StatisticsType::LIST_STATS;
    case PhysicalType::ARRAY:
        return StatisticsType::ARRAY_STATS;
    case PhysicalType::BIT:
    case PhysicalType::INTERVAL:
    default:
        return StatisticsType::BASE_STATS;
    }
}
```

### 5.2.3 NULL 值追踪

DuckDB 使用两个布尔标志来精确追踪 NULL 值的存在：

```cpp
//! 是否可能包含 NULL 值
bool has_null;
//! 是否可能包含非 NULL 值
bool has_no_null;
```

这两个标志的组合提供了四种状态：

| has_null | has_no_null | 含义 |
|----------|-------------|------|
| false    | false       | 空段（InitializeEmpty） |
| true     | false       | 所有值都是 NULL |
| false    | true        | 所有值都非 NULL |
| true     | true        | 混合状态（Unknown） |

```cpp
void BaseStatistics::InitializeUnknown() {
    has_null = true;
    has_no_null = true;  // 最保守的状态
}

void BaseStatistics::InitializeEmpty() {
    has_null = false;
    has_no_null = false;  // 空段
}
```

### 5.2.4 统计信息合并

当合并多个段的统计信息时，DuckDB 采用保守策略：

```cpp
void BaseStatistics::Merge(const BaseStatistics &other) {
    // NULL 信息合并：如果任一方可能有 NULL，合并后也可能有
    has_null = has_null || other.has_null;
    has_no_null = has_no_null || other.has_no_null;

    // 根据统计类型分发到对应的合并函数
    switch (GetStatsType()) {
    case StatisticsType::NUMERIC_STATS:
        NumericStats::Merge(*this, other);  // 合并 min/max
        break;
    case StatisticsType::STRING_STATS:
        StringStats::Merge(*this, other);   // 合并字符串统计
        break;
    case StatisticsType::LIST_STATS:
        ListStats::Merge(*this, other);     // 合并列表子元素统计
        break;
    case StatisticsType::STRUCT_STATS:
        StructStats::Merge(*this, other);   // 合并结构体字段统计
        break;
    // ... 其他类型
    }
}
```

---

## 5.3 NumericStats：数值类型统计

### 5.3.1 数据结构

数值统计是最常用的统计类型，存储精确的最小值和最大值：

```cpp
// src/include/duckdb/storage/statistics/numeric_stats.hpp

struct NumericStatsData {
    //! 是否有最小值
    bool has_min;
    //! 是否有最大值
    bool has_max;
    //! 最小值（union 存储）
    NumericValueUnion min;
    //! 最大值（union 存储）
    NumericValueUnion max;
};

// 数值 union，支持所有数值类型
union NumericValueUnion {
    struct {
        bool boolean;
        int8_t tinyint;
        int16_t smallint;
        int32_t integer;
        int64_t bigint;
        uint8_t utinyint;
        uint16_t usmallint;
        uint32_t uinteger;
        uint64_t ubigint;
        hugeint_t hugeint;
        uhugeint_t uhugeint;
        float float_;
        double double_;
    } value_;
};
```

### 5.3.2 统计更新

更新数值统计时，DuckDB 会跟踪最小值和最大值：

```cpp
template <class T>
static inline void UpdateValue(T new_value, T &min, T &max) {
    min = LessThan::Operation(new_value, min) ? new_value : min;
    max = GreaterThan::Operation(new_value, max) ? new_value : max;
}

template <class T>
static inline void Update(NumericStatsData &nstats, T new_value) {
    UpdateValue<T>(new_value,
                   nstats.min.GetReferenceUnsafe<T>(),
                   nstats.max.GetReferenceUnsafe<T>());
}
```

### 5.3.3 Zonemap 检查（CheckZonemap）

Zonemap 是 DuckDB 用于快速过滤数据块的核心机制。通过比较查询条件与段的 min/max 统计信息，可以在不读取实际数据的情况下判断是否需要扫描该段：

```cpp
// src/storage/statistics/numeric_stats.cpp

template <class T>
FilterPropagateResult CheckZonemapTemplated(
    const BaseStatistics &stats,
    ExpressionType comparison_type,
    T min_value, T max_value, T constant)
{
    switch (comparison_type) {
    case ExpressionType::COMPARE_EQUAL:
    case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
        // 等值比较：常量必须在 [min, max] 范围内
        if (ConstantExactRange(min_value, max_value, constant)) {
            return FilterPropagateResult::FILTER_ALWAYS_TRUE;
        }
        if (ConstantValueInRange(min_value, max_value, constant)) {
            return FilterPropagateResult::NO_PRUNING_POSSIBLE;
        }
        return FilterPropagateResult::FILTER_ALWAYS_FALSE;

    case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
        // X >= C：只有当 max >= C 时才可能有匹配
        //         当 min >= C 时，所有值都满足
        if (GreaterThanEquals::Operation(min_value, constant)) {
            return FilterPropagateResult::FILTER_ALWAYS_TRUE;
        } else if (GreaterThanEquals::Operation(max_value, constant)) {
            return FilterPropagateResult::NO_PRUNING_POSSIBLE;
        } else {
            return FilterPropagateResult::FILTER_ALWAYS_FALSE;
        }

    case ExpressionType::COMPARE_LESSTHAN:
        // X < C：只有当 min < C 时才可能有匹配
        //        当 max < C 时，所有值都满足
        if (LessThan::Operation(max_value, constant)) {
            return FilterPropagateResult::FILTER_ALWAYS_TRUE;
        } else if (LessThan::Operation(min_value, constant)) {
            return FilterPropagateResult::NO_PRUNING_POSSIBLE;
        } else {
            return FilterPropagateResult::FILTER_ALWAYS_FALSE;
        }
    // ... 其他比较类型
    }
}
```

**FilterPropagateResult 枚举**定义了过滤结果：

```cpp
// src/include/duckdb/common/enums/filter_propagate_result.hpp

enum class FilterPropagateResult : uint8_t {
    NO_PRUNING_POSSIBLE = 0,   // 无法确定，需要扫描
    FILTER_ALWAYS_TRUE = 1,    // 所有行都满足条件
    FILTER_ALWAYS_FALSE = 2,   // 所有行都不满足条件（可跳过）
    FILTER_TRUE_OR_NULL = 3,   // 满足条件或为 NULL
    FILTER_FALSE_OR_NULL = 4   // 不满足条件或为 NULL
};
```

### 5.3.4 Zonemap 过滤示例

```
查询：SELECT * FROM t WHERE x > 50

段统计信息：
┌───────────────┬────────┬────────┬───────────────────┐
│    Segment    │  Min   │  Max   │   过滤结果        │
├───────────────┼────────┼────────┼───────────────────┤
│   Segment 1   │   10   │   30   │ ALWAYS_FALSE ✗    │
│   Segment 2   │   40   │   60   │ NO_PRUNING ⚠      │
│   Segment 3   │   70   │  100   │ ALWAYS_TRUE ✓     │
│   Segment 4   │   20   │   80   │ NO_PRUNING ⚠      │
└───────────────┴────────┴────────┴───────────────────┘

结果：
- Segment 1：跳过（max=30 < 50）
- Segment 2：需要扫描并过滤
- Segment 3：全部读取（min=70 > 50）
- Segment 4：需要扫描并过滤
```

---

## 5.4 StringStats：字符串类型统计

### 5.4.1 数据结构

字符串统计采用截断策略，只存储前 8 个字节：

```cpp
// src/include/duckdb/storage/statistics/string_stats.hpp

struct StringStatsData {
    //! 存储的 min/max 字符串最大长度
    constexpr static uint32_t MAX_STRING_MINMAX_SIZE = 8;

    //! 最小值的前 8 字节
    data_t min[MAX_STRING_MINMAX_SIZE];
    //! 最大值的前 8 字节
    data_t max[MAX_STRING_MINMAX_SIZE];
    //! 是否包含 Unicode 字符
    bool has_unicode;
    //! 是否有最大字符串长度统计
    bool has_max_string_length;
    //! 最大字符串长度
    uint32_t max_string_length;
};
```

**设计理由**：
- **空间效率**：只存储 8 字节，避免存储完整长字符串
- **比较效率**：8 字节足以进行快速前缀比较
- **Unicode 标志**：优化纯 ASCII 场景下的字符串处理

### 5.4.2 统计更新

```cpp
// src/storage/statistics/string_stats.cpp

void StringStats::Update(BaseStatistics &stats, const string_t &value) {
    auto data = const_data_ptr_cast(value.GetData());
    auto size = value.GetSize();

    // 截断到 8 字节
    data_t target[StringStatsData::MAX_STRING_MINMAX_SIZE];
    ConstructValue(data, size, target);

    auto &string_data = StringStats::GetDataUnsafe(stats);

    // 更新最小值
    if (StringValueComparison(target, MAX_STRING_MINMAX_SIZE, string_data.min) < 0) {
        memcpy(string_data.min, target, MAX_STRING_MINMAX_SIZE);
    }
    // 更新最大值
    if (StringValueComparison(target, MAX_STRING_MINMAX_SIZE, string_data.max) > 0) {
        memcpy(string_data.max, target, MAX_STRING_MINMAX_SIZE);
    }
    // 更新最大长度
    if (size > string_data.max_string_length) {
        string_data.max_string_length = UnsafeNumericCast<uint32_t>(size);
    }
    // 检测 Unicode
    if (stats.GetType().id() == LogicalTypeId::VARCHAR && !string_data.has_unicode) {
        auto unicode = Utf8Proc::Analyze(const_char_ptr_cast(data), size);
        if (unicode == UnicodeType::UTF8) {
            string_data.has_unicode = true;
        }
    }
}
```

### 5.4.3 字符串 Zonemap 检查

字符串比较基于字节序进行：

```cpp
FilterPropagateResult StringStats::CheckZonemap(
    const_data_ptr_t min_data, idx_t min_len,
    const_data_ptr_t max_data, idx_t max_len,
    ExpressionType comparison_type,
    const string &constant)
{
    auto data = const_data_ptr_cast(constant.c_str());
    idx_t size = constant.size();

    // 比较常量与 min/max
    int min_comp = StringValueComparison(data, MinValue(min_len, size), min_data);
    int max_comp = StringValueComparison(data, MinValue(max_len, size), max_data);

    switch (comparison_type) {
    case ExpressionType::COMPARE_EQUAL:
        // 等值比较：常量必须在 [min, max] 范围内
        if (min_comp >= 0 && max_comp <= 0) {
            return FilterPropagateResult::NO_PRUNING_POSSIBLE;
        } else {
            return FilterPropagateResult::FILTER_ALWAYS_FALSE;
        }
    // ... 其他比较类型
    }
}
```

**截断带来的保守性**：

由于只存储前 8 字节，字符串统计在某些情况下会更加保守：

```
示例：段中包含字符串 "abcdefgh_suffix" 和 "abcdefgh_prefix"
存储的统计信息：
  min = "abcdefgh"  （截断后）
  max = "abcdefgh"  （截断后）

查询 WHERE name = 'abcdefgh_other'
  - 常量前缀 "abcdefgh" 与 min/max 匹配
  - 结果：NO_PRUNING_POSSIBLE（需要扫描）
  - 这是保守但正确的行为
```

---

## 5.5 DistinctStatistics：基数估计

### 5.5.1 HyperLogLog 实现

DuckDB 使用 HyperLogLog (HLL) 算法来估计列的唯一值数量：

```cpp
// src/include/duckdb/storage/statistics/distinct_statistics.hpp

class DistinctStatistics {
public:
    //! HyperLogLog 数据结构
    unique_ptr<HyperLogLog> log;
    //! 采样到 HLL 中的值数量
    atomic<idx_t> sample_count;
    //! 插入的总值数量（采样前）
    atomic<idx_t> total_count;

private:
    //! 基础采样率（10%）
    static constexpr double BASE_SAMPLE_RATE = 0.1;
    //! 整数类型的采样率（30%，因为更可能是 join key）
    static constexpr double INTEGRAL_SAMPLE_RATE = 0.3;
};
```

### 5.5.2 采样更新

为了性能考虑，DuckDB 对输入数据进行采样：

```cpp
// src/storage/statistics/distinct_statistics.cpp

void DistinctStatistics::UpdateSample(Vector &new_data, idx_t count, Vector &hashes) {
    total_count += count;

    // 根据数据类型选择采样率
    const auto sample_rate = new_data.GetType().IsIntegral()
        ? INTEGRAL_SAMPLE_RATE   // 整数：30%
        : BASE_SAMPLE_RATE;      // 其他：10%

    // 计算实际采样数量
    count = MaxValue<idx_t>(
        LossyNumericCast<idx_t>(sample_rate * static_cast<double>(STANDARD_VECTOR_SIZE)),
        1);
    count = MinValue<idx_t>(count, original_count);

    UpdateInternal(new_data, count, hashes);
}

void DistinctStatistics::UpdateInternal(Vector &update, idx_t count, Vector &hashes) {
    sample_count += count;
    VectorOperations::Hash(update, hashes, count);
    log->Update(update, hashes, count);
}
```

### 5.5.3 基数估计算法

使用 Good-Turing 估计来校正采样误差：

```cpp
idx_t DistinctStatistics::GetCount() const {
    if (sample_count == 0 || total_count == 0) {
        return 0;
    }

    double u = static_cast<double>(MinValue<idx_t>(log->Count(), sample_count));
    double s = static_cast<double>(sample_count.load());
    double n = static_cast<double>(total_count.load());

    // 假设采样值中只出现一次的比例
    double u1 = pow(u / s, 2) * u;

    // Good-Turing 估计
    idx_t estimate = LossyNumericCast<idx_t>(u + u1 / s * (n - s));
    return MinValue<idx_t>(estimate, total_count);
}
```

**不支持的类型**：

```cpp
bool DistinctStatistics::TypeIsSupported(const LogicalType &type) {
    switch (type.InternalType()) {
    case PhysicalType::LIST:
    case PhysicalType::STRUCT:
    case PhysicalType::ARRAY:
        return false;  // 嵌套类型不支持
    case PhysicalType::BIT:
    case PhysicalType::BOOL:
        return false;  // 基数太小，无意义
    default:
        return true;
    }
}
```

---

## 5.6 嵌套类型统计

### 5.6.1 StructStats

结构体类型为每个字段维护独立的统计信息：

```cpp
// src/include/duckdb/storage/statistics/struct_stats.hpp

struct StructStats {
    //! 构造结构体统计信息
    static void Construct(BaseStatistics &stats);

    //! 创建未知统计信息（所有字段都是 Unknown）
    static BaseStatistics CreateUnknown(LogicalType type);

    //! 创建空统计信息（所有字段都是 Empty）
    static BaseStatistics CreateEmpty(LogicalType type);

    //! 获取子字段统计信息
    static const BaseStatistics &GetChildStats(const BaseStatistics &stats, idx_t i);
    static BaseStatistics &GetChildStats(BaseStatistics &stats, idx_t i);

    //! 设置子字段统计信息
    static void SetChildStats(BaseStatistics &stats, idx_t i, const BaseStatistics &new_stats);

    //! 合并统计信息
    static void Merge(BaseStatistics &stats, const BaseStatistics &other);
};
```

### 5.6.2 ListStats 和 ArrayStats

列表和数组类型为元素类型维护统计信息：

```cpp
// src/include/duckdb/storage/statistics/list_stats.hpp

struct ListStats {
    //! 获取元素统计信息
    static const BaseStatistics &GetChildStats(const BaseStatistics &stats);
    static BaseStatistics &GetChildStats(BaseStatistics &stats);

    //! 设置元素统计信息
    static void SetChildStats(BaseStatistics &stats, unique_ptr<BaseStatistics> new_stats);

    //! 合并：合并子元素的统计信息
    static void Merge(BaseStatistics &stats, const BaseStatistics &other);
};
```

**嵌套统计信息存储**：

```cpp
// 子统计信息存储在 child_stats 数组中
class BaseStatistics {
    // ...
    //! 嵌套类型的子统计信息
    unsafe_unique_array<BaseStatistics> child_stats;
};
```

---

## 5.7 ColumnStatistics：列级统计

### 5.7.1 类设计

`ColumnStatistics` 组合了 `BaseStatistics` 和 `DistinctStatistics`：

```cpp
// src/include/duckdb/storage/statistics/column_statistics.hpp

class ColumnStatistics {
public:
    //! 基础统计信息（min/max/null）
    BaseStatistics stats;

    //! 近似唯一值统计（HyperLogLog）
    unique_ptr<DistinctStatistics> distinct_stats;

public:
    //! 创建空统计信息
    static shared_ptr<ColumnStatistics> CreateEmptyStats(const LogicalType &type);

    //! 合并两个列统计信息
    void Merge(ColumnStatistics &other);

    //! 更新唯一值统计
    void UpdateDistinctStatistics(Vector &v, idx_t count, Vector &hashes);

    //! 获取基础统计
    BaseStatistics &Statistics();

    //! 获取唯一值统计
    bool HasDistinctStats();
    DistinctStatistics &DistinctStats();
};
```

### 5.7.2 SegmentStatistics

`SegmentStatistics` 是段级别的统计信息包装器：

```cpp
// src/include/duckdb/storage/statistics/segment_statistics.hpp

class SegmentStatistics {
public:
    explicit SegmentStatistics(LogicalType type);
    explicit SegmentStatistics(BaseStatistics statistics);

    //! 该段的类型特定统计信息
    BaseStatistics statistics;
};
```

---

## 5.8 TableFilter：统计信息驱动的过滤

### 5.8.1 过滤器层次结构

DuckDB 支持多种类型的表过滤器：

```cpp
// src/include/duckdb/planner/table_filter.hpp

enum class TableFilterType : uint8_t {
    CONSTANT_COMPARISON = 0,  // 常量比较 (=C, >C, >=C, <C, <=C)
    IS_NULL = 1,              // IS NULL
    IS_NOT_NULL = 2,          // IS NOT NULL
    CONJUNCTION_OR = 3,       // OR 组合
    CONJUNCTION_AND = 4,      // AND 组合
    STRUCT_EXTRACT = 5,       // 结构体字段过滤
    OPTIONAL_FILTER = 6,      // 可选过滤器
    IN_FILTER = 7,            // col IN (C1, C2, C3, ...)
    DYNAMIC_FILTER = 8,       // 运行时动态过滤器
    EXPRESSION_FILTER = 9,    // 任意表达式
    BLOOM_FILTER = 10,        // 布隆过滤器
};
```

### 5.8.2 过滤器接口

每种过滤器都实现 `CheckStatistics` 方法：

```cpp
class TableFilter {
public:
    TableFilterType filter_type;

    //! 使用统计信息检查是否可以跳过段
    virtual FilterPropagateResult CheckStatistics(BaseStatistics &stats) const = 0;

    //! 转换为字符串表示
    virtual string ToString(const string &column_name) const = 0;

    //! 复制过滤器
    virtual unique_ptr<TableFilter> Copy() const = 0;

    //! 转换为表达式
    virtual unique_ptr<Expression> ToExpression(const Expression &column) const = 0;
};
```

### 5.8.3 TableFilterSet

表过滤器集合管理多个列的过滤条件：

```cpp
class TableFilterSet {
public:
    //! 列索引 -> 过滤器映射
    map<idx_t, unique_ptr<TableFilter>> filters;

    //! 添加过滤器
    void PushFilter(const ColumnIndex &col_idx, unique_ptr<TableFilter> filter);

    //! 比较两个过滤器集合
    bool Equals(TableFilterSet &other);

    //! 复制过滤器集合
    unique_ptr<TableFilterSet> Copy() const;
};
```

---

## 5.9 统计信息在查询优化中的应用

### 5.9.1 谓词下推与段跳过

```
┌────────────────────────────────────────────────────────────────────────┐
│               统计信息驱动的查询优化流程                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SELECT * FROM orders WHERE amount > 1000                              │
│                                                                        │
│  ┌──────────────┐      ┌───────────────────────────────────────────┐  │
│  │ TableFilter  │      │              RowGroup 1                   │  │
│  │ amount > 1000│──────│  amount: [min=50, max=500]                │  │
│  └──────────────┘      │  CheckZonemap → FILTER_ALWAYS_FALSE       │  │
│         │              │  结果: 跳过整个 RowGroup                   │  │
│         │              └───────────────────────────────────────────┘  │
│         │                                                              │
│         │              ┌───────────────────────────────────────────┐  │
│         │              │              RowGroup 2                   │  │
│         └──────────────│  amount: [min=800, max=1500]              │  │
│                        │  CheckZonemap → NO_PRUNING_POSSIBLE       │  │
│                        │  结果: 扫描并过滤                          │  │
│                        └───────────────────────────────────────────┘  │
│                                                                        │
│                        ┌───────────────────────────────────────────┐  │
│                        │              RowGroup 3                   │  │
│                        │  amount: [min=2000, max=5000]             │  │
│                        │  CheckZonemap → FILTER_ALWAYS_TRUE        │  │
│                        │  结果: 全部读取（无需逐行过滤）            │  │
│                        └───────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.9.2 基数估计在 Join 优化中的应用

```
┌────────────────────────────────────────────────────────────────────────┐
│               DistinctStatistics 在 Join 中的应用                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id       │
│                                                                        │
│  orders 表统计：                                                        │
│    - 行数: 10,000,000                                                  │
│    - customer_id distinct: ~500,000 (HLL 估计)                         │
│                                                                        │
│  customers 表统计：                                                     │
│    - 行数: 1,000,000                                                   │
│    - id distinct: 1,000,000 (主键)                                     │
│                                                                        │
│  优化器决策：                                                           │
│    1. 估计 Join 结果基数: 10,000,000 × (500,000/1,000,000) ≈ 5,000,000 │
│    2. 选择 customers 作为 Build 侧（更小的表）                          │
│    3. 选择 Hash Join（基于基数估计）                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.9.3 常量检测优化

当统计信息显示某列为常量时，可以启用特殊优化：

```cpp
bool BaseStatistics::IsConstant() const {
    if (type.id() == LogicalTypeId::VALIDITY) {
        // 有效性掩码：全 NULL 或全非 NULL
        if (CanHaveNull() && !CanHaveNoNull()) {
            return true;
        }
        if (!CanHaveNull() && CanHaveNoNull()) {
            return true;
        }
        return false;
    }
    switch (GetStatsType()) {
    case StatisticsType::NUMERIC_STATS:
        // 数值类型：min == max
        return NumericStats::IsConstant(*this);
    default:
        break;
    }
    return false;
}

bool NumericStats::IsConstant(const BaseStatistics &stats) {
    return NumericStats::Max(stats) <= NumericStats::Min(stats);
}
```

---

## 5.10 统计信息序列化

### 5.10.1 序列化格式

DuckDB 使用属性编号来确保向前兼容：

```cpp
void BaseStatistics::Serialize(Serializer &serializer) const {
    serializer.WriteProperty(100, "has_null", has_null);
    serializer.WriteProperty(101, "has_no_null", has_no_null);
    serializer.WriteProperty(102, "distinct_count", distinct_count);
    serializer.WriteObject(103, "type_stats", [&](Serializer &serializer) {
        switch (GetStatsType()) {
        case StatisticsType::NUMERIC_STATS:
            NumericStats::Serialize(*this, serializer);
            break;
        case StatisticsType::STRING_STATS:
            StringStats::Serialize(*this, serializer);
            break;
        // ... 其他类型
        }
    });
}
```

### 5.10.2 NumericStats 序列化

```cpp
void NumericStats::Serialize(const BaseStatistics &stats, Serializer &serializer) {
    auto &numeric_stats = NumericStats::GetDataUnsafe(stats);
    serializer.WriteObject(200, "max", [&](Serializer &object) {
        SerializeNumericStatsValue(stats.GetType(), numeric_stats.min,
                                   numeric_stats.has_min, object);
    });
    serializer.WriteObject(201, "min", [&](Serializer &object) {
        SerializeNumericStatsValue(stats.GetType(), numeric_stats.max,
                                   numeric_stats.has_max, object);
    });
}
```

---

## 5.11 统计信息验证

DuckDB 在调试模式下验证统计信息的正确性：

```cpp
void BaseStatistics::Verify(Vector &vector, const SelectionVector &sel,
                            idx_t count, const bool ignore_has_null) const
{
    D_ASSERT(vector.GetType() == this->type);

    // 分发到类型特定的验证
    switch (GetStatsType()) {
    case StatisticsType::NUMERIC_STATS:
        NumericStats::Verify(*this, vector, sel, count);
        break;
    case StatisticsType::STRING_STATS:
        StringStats::Verify(*this, vector, sel, count);
        break;
    // ... 其他类型
    }

    // 验证 NULL 值一致性
    if (has_null && has_no_null) {
        return;  // Unknown 状态，无法验证
    }

    UnifiedVectorFormat vdata;
    vector.ToUnifiedFormat(count, vdata);

    for (idx_t i = 0; i < count; i++) {
        auto idx = sel.get_index(i);
        auto index = vdata.sel->get_index(idx);
        bool row_is_valid = vdata.validity.RowIsValid(index);

        if (row_is_valid && !has_no_null) {
            throw InternalException(
                "Statistics mismatch: vector labeled as having only NULL values, "
                "but vector contains valid values");
        }
        if (!row_is_valid && !has_null && !ignore_has_null) {
            throw InternalException(
                "Statistics mismatch: vector labeled as not having NULL values, "
                "but vector contains null values");
        }
    }
}
```

---

## 5.12 总结

### 5.12.1 统计类型速查表

| 统计类型 | 适用类型 | 存储内容 | 主要用途 |
|---------|---------|---------|---------|
| NumericStats | 整数、浮点数 | 精确 min/max | Zonemap 过滤 |
| StringStats | VARCHAR | 截断 min/max (8字节)、Unicode 标志 | 前缀过滤、编码选择 |
| DistinctStats | 非嵌套类型 | HyperLogLog | 基数估计、Join 优化 |
| StructStats | STRUCT | 子字段统计数组 | 嵌套查询优化 |
| ListStats | LIST | 元素类型统计 | 嵌套查询优化 |
| ArrayStats | ARRAY | 元素类型统计 | 嵌套查询优化 |

### 5.12.2 关键设计特点

1. **类型安全**：使用 union 和模板实现零开销的类型多态
2. **空间高效**：字符串只存储 8 字节前缀，使用采样的 HLL
3. **保守估计**：在无法确定时返回 NO_PRUNING_POSSIBLE
4. **嵌套支持**：通过 child_stats 数组支持任意嵌套结构
5. **可验证**：调试模式下自动验证统计信息正确性

### 5.12.3 核心源文件索引

| 文件 | 功能 |
|------|------|
| `src/include/duckdb/storage/statistics/base_statistics.hpp` | BaseStatistics 定义 |
| `src/storage/statistics/base_statistics.cpp` | BaseStatistics 实现 |
| `src/include/duckdb/storage/statistics/numeric_stats.hpp` | NumericStats 定义 |
| `src/storage/statistics/numeric_stats.cpp` | Zonemap 检查实现 |
| `src/include/duckdb/storage/statistics/string_stats.hpp` | StringStats 定义 |
| `src/storage/statistics/string_stats.cpp` | 字符串统计实现 |
| `src/include/duckdb/storage/statistics/distinct_statistics.hpp` | DistinctStatistics 定义 |
| `src/storage/statistics/distinct_statistics.cpp` | HyperLogLog 实现 |
| `src/include/duckdb/storage/statistics/column_statistics.hpp` | ColumnStatistics 定义 |
| `src/include/duckdb/planner/table_filter.hpp` | TableFilter 接口 |
| `src/include/duckdb/common/enums/filter_propagate_result.hpp` | 过滤结果枚举 |

---

**下一章预告**：第六章将深入探讨 DuckDB 的持久化与恢复机制，包括 WAL 日志、Checkpoint 流程和崩溃恢复。

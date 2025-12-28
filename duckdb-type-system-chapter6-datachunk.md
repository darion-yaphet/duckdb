# DuckDB 类型系统深度解析 - 第六章：DataChunk 和批量处理

## 引言

DataChunk 是 DuckDB 执行引擎的核心数据传输单元，表示关系的一个子集。它包含多个等长的向量（Vector），每个向量对应一列数据。本章将深入分析 DataChunk 的结构、操作以及在批量处理中的应用。

## DataChunk 概述

`DataChunk` 是一组向量的容器，所有向量具有相同的长度：

```cpp
// data_chunk.hpp
class DataChunk {
public:
    //! 数据向量集合
    vector<Vector> data;

private:
    idx_t count;              // 当前元组数量
    idx_t capacity;           // 最大容量
    idx_t initial_capacity;   // 初始容量（用于重置）
    vector<VectorCache> vector_caches;  // 向量缓存

public:
    // 大小访问
    inline idx_t size() const { return count; }
    inline idx_t ColumnCount() const { return data.size(); }
    inline idx_t GetCapacity() const { return capacity; }

    // 设置基数
    inline void SetCardinality(idx_t count_p) {
        D_ASSERT(count_p <= capacity);
        this->count = count_p;
    }

    // 值访问
    Value GetValue(idx_t col_idx, idx_t index) const;
    void SetValue(idx_t col_idx, idx_t index, const Value &val);

    // 类型信息
    vector<LogicalType> GetTypes() const;
};
```

## DataChunk 生命周期

### 创建与初始化

DataChunk 支持多种初始化方式：

```cpp
// 创建空 DataChunk
DataChunk chunk;

// 仅初始化类型，不分配数据
chunk.InitializeEmpty(types);

// 完整初始化（分配内存）
chunk.Initialize(allocator, types, capacity);

// 带选择性初始化
vector<bool> initialize = {true, false, true};  // 仅初始化第 0、2 列
chunk.Initialize(allocator, types, initialize, capacity);
```

### InitializeEmpty

仅设置向量类型，不分配数据空间：

```cpp
void DataChunk::InitializeEmpty(const vector<LogicalType> &types) {
    capacity = 0;
    count = 0;
    initial_capacity = 0;
    data.clear();
    for (auto &type : types) {
        data.emplace_back(Vector(type, nullptr));
    }
}
```

### Initialize

完整初始化，为每列分配数据空间：

```cpp
void DataChunk::Initialize(Allocator &allocator,
                            const vector<LogicalType> &types,
                            idx_t capacity) {
    D_ASSERT(data.empty());
    D_ASSERT(capacity <= STANDARD_VECTOR_SIZE);

    this->capacity = capacity;
    this->initial_capacity = capacity;
    this->count = 0;

    vector_caches.reserve(types.size());
    for (auto &type : types) {
        vector_caches.emplace_back(allocator, type, capacity);
        data.emplace_back(vector_caches.back());
    }
}
```

## 数据操作

### Reference（引用）

创建对另一个 DataChunk 的引用，共享数据：

```cpp
void DataChunk::Reference(DataChunk &other) {
    count = other.count;
    capacity = other.capacity;
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Reference(other.data[i]);
    }
}
```

### Move（移动）

转移数据所有权，销毁源 DataChunk：

```cpp
void DataChunk::Move(DataChunk &other) {
    data = std::move(other.data);
    vector_caches = std::move(other.vector_caches);
    count = other.count;
    capacity = other.capacity;
    initial_capacity = other.initial_capacity;
    other.Destroy();
}
```

### Append（追加）

将另一个 DataChunk 的数据追加到当前块：

```cpp
void DataChunk::Append(const DataChunk &other, bool resize,
                       SelectionVector *sel, idx_t append_count) {
    if (append_count == 0) {
        append_count = other.size();
    }

    // 检查空间
    idx_t new_count = count + append_count;
    if (new_count > capacity) {
        if (!resize) {
            throw OutOfRangeException("Cannot append to chunk - exceeds capacity");
        }
        // 扩展容量...
    }

    // 追加每列
    for (idx_t i = 0; i < ColumnCount(); i++) {
        // 复制数据到目标向量
        VectorOperations::Copy(other.data[i], data[i], sel, append_count, 0, count);
    }

    count = new_count;
}
```

### Copy（复制）

创建数据的深拷贝：

```cpp
void DataChunk::Copy(DataChunk &other, idx_t offset) const {
    D_ASSERT(ColumnCount() == other.ColumnCount());

    for (idx_t i = 0; i < ColumnCount(); i++) {
        VectorOperations::Copy(data[i], other.data[i], count, offset, 0);
    }
    other.SetCardinality(count - offset);
}

void DataChunk::Copy(DataChunk &other, const SelectionVector &sel,
                     const idx_t source_count, const idx_t offset) const {
    D_ASSERT(ColumnCount() == other.ColumnCount());

    for (idx_t i = 0; i < ColumnCount(); i++) {
        VectorOperations::Copy(data[i], other.data[i], sel, source_count, offset, 0);
    }
    other.SetCardinality(source_count - offset);
}
```

### Split / Fuse（分裂 / 融合）

将 DataChunk 从指定位置分裂为两个：

```cpp
void DataChunk::Split(DataChunk &other, idx_t split_idx) {
    D_ASSERT(other.ColumnCount() == 0);

    // 移动 split_idx 之后的列到 other
    for (idx_t i = split_idx; i < ColumnCount(); i++) {
        other.data.push_back(std::move(data[i]));
        other.vector_caches.push_back(std::move(vector_caches[i]));
    }

    // 调整当前块
    data.resize(split_idx);
    vector_caches.resize(split_idx);

    other.count = count;
    other.capacity = capacity;
}

// Fuse 是 Split 的逆操作
void DataChunk::Fuse(DataChunk &other) {
    for (idx_t i = 0; i < other.ColumnCount(); i++) {
        data.push_back(std::move(other.data[i]));
        vector_caches.push_back(std::move(other.vector_caches[i]));
    }
    other.Destroy();
}
```

### ReferenceColumns（列引用）

创建对指定列的引用：

```cpp
void DataChunk::ReferenceColumns(DataChunk &other,
                                  const vector<column_t> &column_ids) {
    D_ASSERT(column_ids.size() <= other.ColumnCount());
    InitializeEmpty(GetTypes());
    for (idx_t i = 0; i < column_ids.size(); i++) {
        data[i].Reference(other.data[column_ids[i]]);
    }
    SetCardinality(other.size());
    SetCapacity(other.GetCapacity());
}
```

## 批量操作

### Flatten（展平）

将所有向量转换为 FLAT_VECTOR：

```cpp
void DataChunk::Flatten() {
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Flatten(count);
    }
}
```

### Slice（切片）

创建数据子集：

```cpp
// 使用选择向量切片
void DataChunk::Slice(const SelectionVector &sel_vector, idx_t count) {
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Slice(sel_vector, count);
    }
    SetCardinality(count);
}

// 范围切片
void DataChunk::Slice(idx_t offset, idx_t slice_count) {
    D_ASSERT(offset + slice_count <= count);
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Slice(data[i], offset, offset + slice_count);
    }
    SetCardinality(slice_count);
}

// 从另一个 DataChunk 切片
void DataChunk::Slice(const DataChunk &other, const SelectionVector &sel,
                       idx_t count, idx_t col_offset) {
    D_ASSERT(other.ColumnCount() <= ColumnCount() - col_offset);
    for (idx_t i = 0; i < other.ColumnCount(); i++) {
        data[col_offset + i].Slice(other.data[i], sel, count);
    }
    SetCardinality(count);
}
```

### ToUnifiedFormat（统一格式转换）

将所有向量转换为统一访问格式：

```cpp
unsafe_unique_array<UnifiedVectorFormat> DataChunk::ToUnifiedFormat() {
    auto result = make_unsafe_uniq_array_uninitialized<UnifiedVectorFormat>(
        ColumnCount());

    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].ToUnifiedFormat(count, result[i]);
    }
    return result;
}
```

### Hash（哈希）

计算所有行的哈希值：

```cpp
void DataChunk::Hash(Vector &result) {
    D_ASSERT(result.GetType().id() == LogicalTypeId::HASH);
    VectorOperations::Hash(data[0], result, count);
    for (idx_t i = 1; i < ColumnCount(); i++) {
        VectorOperations::CombineHash(result, data[i], count);
    }
}

// 仅对指定列哈希
void DataChunk::Hash(vector<idx_t> &column_ids, Vector &result) {
    D_ASSERT(column_ids.size() > 0);
    VectorOperations::Hash(data[column_ids[0]], result, count);
    for (idx_t i = 1; i < column_ids.size(); i++) {
        VectorOperations::CombineHash(result, data[column_ids[i]], count);
    }
}
```

### Reset（重置）

重置到初始状态：

```cpp
void DataChunk::Reset() {
    count = 0;
    capacity = initial_capacity;
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].ResetFromCache(vector_caches[i]);
    }
}
```

## VectorCache

`VectorCache` 用于高效重用向量内存：

```cpp
class VectorCache {
    buffer_ptr<VectorBuffer> buffer;
    LogicalType type;
    idx_t capacity;

public:
    VectorCache(Allocator &allocator, const LogicalType &type,
                idx_t capacity = STANDARD_VECTOR_SIZE);

    // 从缓存重置向量
    void ResetFromCache(Vector &result);
};
```

向量缓存的作用：
- 避免重复内存分配
- 支持快速 Reset 操作
- 保持内存布局一致

## 数据流模型

DataChunk 在执行引擎中的数据流：

```
┌─────────────────────────────────────────────────────────────┐
│                      查询执行流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   TableScan                                                 │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│   │DataChunk│───▶│DataChunk│───▶│DataChunk│───▶ ...       │
│   │ (2048)  │    │ (2048)  │    │ (1024)  │               │
│   └─────────┘    └─────────┘    └─────────┘               │
│       │                                                     │
│       ▼                                                     │
│   Filter (基于选择向量过滤)                                   │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────┐    ┌─────────┐                               │
│   │DataChunk│───▶│DataChunk│───▶ ...                       │
│   │ (1500)  │    │ (800)   │                               │
│   └─────────┘    └─────────┘                               │
│       │                                                     │
│       ▼                                                     │
│   Projection (向量运算)                                      │
│       │                                                     │
│       ▼                                                     │
│   Aggregate (批量聚合)                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 执行引擎集成

### 操作符接口

每个物理操作符通过 DataChunk 交换数据：

```cpp
class PhysicalOperator {
public:
    // 获取下一批数据
    virtual OperatorResultType Execute(ExecutionContext &context,
                                        DataChunk &input,
                                        DataChunk &chunk,
                                        GlobalOperatorState &gstate,
                                        OperatorState &state);

    // 源操作符（如 TableScan）
    virtual SourceResultType GetData(ExecutionContext &context,
                                      DataChunk &chunk,
                                      GlobalSourceState &gstate,
                                      LocalSourceState &lstate);

    // 汇操作符（如 Insert）
    virtual SinkResultType Sink(ExecutionContext &context,
                                 GlobalSinkState &gstate,
                                 LocalSinkState &lstate,
                                 DataChunk &input);
};
```

### 批量处理示例

Filter 操作符的批量过滤：

```cpp
// 过滤操作
SelectionVector sel(STANDARD_VECTOR_SIZE);
idx_t result_count = 0;

// 获取过滤条件结果
UnifiedVectorFormat filter_data;
filter_vector.ToUnifiedFormat(count, filter_data);
auto filter = filter_data.GetData<bool>();

// 批量筛选
for (idx_t i = 0; i < count; i++) {
    idx_t idx = filter_data.sel->get_index(i);
    if (filter_data.validity.RowIsValid(idx) && filter[idx]) {
        sel.set_index(result_count++, i);
    }
}

// 应用选择向量
chunk.Slice(sel, result_count);
```

投影操作符的批量计算：

```cpp
// 对所有列应用表达式
for (idx_t col_idx = 0; col_idx < expressions.size(); col_idx++) {
    expressions[col_idx]->Execute(input, result.data[col_idx], count);
}
result.SetCardinality(count);
```

## 序列化与反序列化

DataChunk 支持序列化以用于分布式执行或持久化：

```cpp
void DataChunk::Serialize(Serializer &serializer,
                           bool compressed_serialization) const {
    // 写入列数和行数
    serializer.WriteProperty(100, "column_count", ColumnCount());
    serializer.WriteProperty(101, "count", count);

    // 序列化每列
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Serialize(serializer, count, compressed_serialization);
    }
}

void DataChunk::Deserialize(Deserializer &source) {
    idx_t column_count = source.ReadProperty<idx_t>(100, "column_count");
    idx_t row_count = source.ReadProperty<idx_t>(101, "count");

    for (idx_t i = 0; i < column_count; i++) {
        data[i].Deserialize(source, row_count);
    }
    SetCardinality(row_count);
}
```

## 调试与验证

### Verify（验证）

验证 DataChunk 的一致性：

```cpp
void DataChunk::Verify() {
#ifdef DEBUG
    D_ASSERT(count <= capacity);
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].Verify(count);
    }
#endif
}
```

### ToString / Print（打印）

用于调试输出：

```cpp
string DataChunk::ToString() const {
    string retval = "DataChunk - [" + std::to_string(ColumnCount()) + " columns]\n";
    retval += "- Size: " + std::to_string(count) + "\n";

    for (idx_t i = 0; i < ColumnCount(); i++) {
        retval += "Column " + std::to_string(i) + ": ";
        retval += data[i].ToString(count);
        retval += "\n";
    }
    return retval;
}

void DataChunk::Print() const {
    Printer::Print(ToString());
}
```

## 内存管理

### 容量管理

DataChunk 维护容量以优化内存使用：

```cpp
// 获取当前分配大小
idx_t DataChunk::GetAllocationSize() const {
    idx_t total = 0;
    for (idx_t i = 0; i < ColumnCount(); i++) {
        total += data[i].GetAllocationSize(capacity);
    }
    return total;
}
```

### 常量检测

检测是否所有向量都是常量（用于优化）：

```cpp
bool DataChunk::AllConstant() const {
    for (idx_t i = 0; i < ColumnCount(); i++) {
        if (data[i].GetVectorType() != VectorType::CONSTANT_VECTOR) {
            return false;
        }
    }
    return true;
}
```

## 标准向量大小

```cpp
// vector_size.hpp
constexpr idx_t STANDARD_VECTOR_SIZE = 2048;
```

选择 2048 作为标准大小的原因：
- 足够大以摊销函数调用开销
- 足够小以保持在 CPU L1/L2 缓存中
- 2 的幂便于对齐和位操作
- 平衡内存使用和处理效率

## 最佳实践

### 1. 尽量使用引用而非拷贝

```cpp
// 好：使用引用
chunk.Reference(source_chunk);

// 避免：不必要的拷贝
source_chunk.Copy(chunk);
```

### 2. 重用 DataChunk

```cpp
// 好：重用 DataChunk
chunk.Reset();
// 填充新数据...

// 避免：频繁分配新的 DataChunk
DataChunk new_chunk;
new_chunk.Initialize(allocator, types);
```

### 3. 利用常量向量

```cpp
// 检查是否可以使用常量优化
if (chunk.AllConstant()) {
    // 使用常量快速路径
}
```

### 4. 批量操作

```cpp
// 好：批量处理整个 DataChunk
VectorOperations::Add(chunk.data[0], chunk.data[1], result, chunk.size());

// 避免：逐行处理
for (idx_t i = 0; i < chunk.size(); i++) {
    result[i] = chunk.GetValue(0, i) + chunk.GetValue(1, i);
}
```

## 小结

本章详细介绍了 DataChunk 的结构和操作：

1. **DataChunk 结构**：
   - 向量集合 + 元数据（count, capacity）
   - VectorCache 用于高效内存重用

2. **核心操作**：
   - Initialize/InitializeEmpty：初始化
   - Reference/Move：引用和移动语义
   - Append/Copy：数据追加和复制
   - Split/Fuse：分裂和融合
   - Slice：切片操作
   - Flatten：展平向量
   - Hash：批量哈希

3. **执行引擎集成**：
   - 操作符间的数据传输单元
   - 支持 Source/Sink/Execute 模式
   - 选择向量实现零拷贝过滤

4. **性能优化**：
   - 标准大小 2048 行
   - 向量缓存避免重复分配
   - 常量检测启用快速路径
   - 引用语义减少拷贝

DataChunk 是 DuckDB 向量化执行的基石，通过批量处理显著提升查询性能。

下一章将探讨 Value 类型系统，了解如何在类型安全的前提下处理单个值。

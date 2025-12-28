# 第八章：类型转换系统

## 8.1 类型转换系统概述

DuckDB的类型转换系统是一个高度灵活和可扩展的框架，负责在不同数据类型之间进行转换。这个系统支持隐式转换（在表达式求值时自动进行）和显式转换（通过CAST操作符明确指定）。

### 8.1.1 设计目标

类型转换系统的核心设计目标包括：

1. **安全性**：转换过程中检测并处理溢出、精度损失等错误
2. **高性能**：支持向量化批量转换，避免逐行处理开销
3. **可扩展性**：允许扩展注册自定义类型转换函数
4. **灵活性**：支持严格模式和宽松模式的转换行为

### 8.1.2 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                    CastFunctionSet                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │             bind_functions (绑定函数链)                │  │
│  │  [MapCastFunction] → [DefaultCasts::GetDefault...]    │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    MapCastInfo                        │  │
│  │    casts[source_type_id][source_type]                 │  │
│  │         [target_type_id][target_type] → MapCastNode   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     BoundCastInfo                           │
│  ┌─────────────────┬─────────────────┬─────────────────┐    │
│  │   function      │  cast_data      │ init_local_state│    │
│  │ (cast_function_t)│(BoundCastData*) │(初始化函数)     │    │
│  └─────────────────┴─────────────────┴─────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## 8.2 CastFunctionSet架构

### 8.2.1 类定义

```cpp
// src/include/duckdb/function/cast/cast_function_set.hpp

class CastFunctionSet {
public:
    CastFunctionSet();
    explicit CastFunctionSet(DBConfig &config);

public:
    // 获取实例
    static CastFunctionSet &Get(ClientContext &context);
    static CastFunctionSet &Get(DatabaseInstance &db);

    // 获取转换函数（始终返回有效函数，因为NULL可以转换为任何类型）
    BoundCastInfo GetCastFunction(const LogicalType &source,
                                  const LogicalType &target,
                                  GetCastFunctionInput &input);

    // 计算隐式转换代价（-1表示不支持隐式转换）
    int64_t ImplicitCastCost(optional_ptr<ClientContext> context,
                             const LogicalType &source,
                             const LogicalType &target);

    // 注册自定义转换函数
    void RegisterCastFunction(const LogicalType &source,
                              const LogicalType &target,
                              BoundCastInfo function,
                              int64_t implicit_cast_cost = -1);

private:
    optional_ptr<DBConfig> config;
    vector<BindCastFunction> bind_functions;  // 绑定函数链
    optional_ptr<MapCastInfo> map_info;       // 自定义转换映射
};
```

### 8.2.2 转换函数查找流程

```cpp
// src/function/cast/cast_function_set.cpp

BoundCastInfo CastFunctionSet::GetCastFunction(const LogicalType &source,
                                               const LogicalType &target,
                                               GetCastFunctionInput &get_input) {
    // 相同类型：无操作转换
    if (source == target) {
        return DefaultCasts::NopCast;
    }

    // 从后向前遍历绑定函数链（后注册的优先）
    for (idx_t i = bind_functions.size(); i > 0; i--) {
        auto &bind_function = bind_functions[i - 1];
        BindCastInput input(*this, bind_function.info.get(), get_input.context);
        auto result = bind_function.function(input, source, target);
        if (result.function) {
            return result;  // 找到有效转换函数
        }
    }

    // 未找到转换：返回NULL转换（仅当所有值为NULL时成功）
    return DefaultCasts::TryVectorNullCast;
}
```

### 8.2.3 自定义转换注册

```cpp
void CastFunctionSet::RegisterCastFunction(const LogicalType &source,
                                           const LogicalType &target,
                                           MapCastNode node) {
    if (!map_info) {
        // 首次注册时创建映射结构
        auto info = make_uniq<MapCastInfo>();
        map_info = info.get();
        bind_functions.emplace_back(MapCastFunction, std::move(info));
    }
    map_info->AddEntry(source, target, std::move(node));
}
```

## 8.3 隐式转换与显式转换

### 8.3.1 隐式转换代价系统

隐式转换代价决定了类型推断时的优先级。较低的代价表示更优先的转换选择。

```cpp
// src/function/cast_rules.cpp

// 目标类型代价（决定隐式转换优先级）
static int64_t TargetTypeCost(const LogicalType &type) {
    switch (type.id()) {
    case LogicalTypeId::BIGINT:     return 101;  // 最优先的整数类型
    case LogicalTypeId::INTEGER:    return 102;
    case LogicalTypeId::HUGEINT:    return 103;
    case LogicalTypeId::DOUBLE:     return 104;
    case LogicalTypeId::DECIMAL:    return 105;
    case LogicalTypeId::TIMESTAMP:  return 120;
    case LogicalTypeId::VARCHAR:    return 149;  // 字符串优先级较低
    case LogicalTypeId::TEMPLATE:   return 1000000; // 模板类型最低优先级
    default:                        return 110;
    }
}
```

### 8.3.2 整数类型隐式转换规则

```cpp
// TINYINT 可以隐式转换的目标类型
static int64_t ImplicitCastTinyint(const LogicalType &to) {
    switch (to.id()) {
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
    case LogicalTypeId::DECIMAL:
        return TargetTypeCost(to);
    default:
        return -1;  // 不支持隐式转换
    }
}

// 无符号整数支持更多转换路径
static int64_t ImplicitCastUTinyint(const LogicalType &to) {
    switch (to.id()) {
    case LogicalTypeId::USMALLINT:
    case LogicalTypeId::UINTEGER:
    case LogicalTypeId::UBIGINT:
    case LogicalTypeId::SMALLINT:   // 可转换到更大的有符号类型
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::UHUGEINT:
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
    case LogicalTypeId::DECIMAL:
        return TargetTypeCost(to);
    default:
        return -1;
    }
}
```

### 8.3.3 复合类型隐式转换

```cpp
int64_t CastRules::ImplicitCast(const LogicalType &from, const LogicalType &to) {
    // LIST 类型递归检查子类型
    if (from.id() == LogicalTypeId::LIST && to.id() == LogicalTypeId::LIST) {
        auto child_cost = ImplicitCast(ListType::GetChildType(from),
                                       ListType::GetChildType(to));
        if (child_cost >= 1) {
            child_cost--;  // 偏好 LIST[X] -> LIST[VARCHAR] 而非 LIST[X] -> VARCHAR
        }
        return child_cost;
    }

    // STRUCT 类型按字段匹配
    if (from.id() == LogicalTypeId::STRUCT && to.id() == LogicalTypeId::STRUCT) {
        auto &source_children = StructType::GetChildTypes(from);
        auto &target_children = StructType::GetChildTypes(to);

        if (source_children.size() != target_children.size()) {
            return -1;  // 字段数不同
        }

        int64_t cost = -1;
        for (idx_t i = 0; i < source_children.size(); i++) {
            auto child_cost = ImplicitCast(source_children[i].second,
                                           target_children[i].second);
            if (child_cost == -1) return -1;
            cost = MaxValue(cost, child_cost);
        }
        return cost;
    }

    // UNION 类型：源成员必须是目标成员的子集
    if (from.id() == LogicalTypeId::UNION && to.id() == LogicalTypeId::UNION) {
        int64_t cost = -1;
        for (idx_t i = 0; i < UnionType::GetMemberCount(from); i++) {
            auto &from_name = UnionType::GetMemberName(from, i);
            bool found = false;
            for (idx_t j = 0; j < UnionType::GetMemberCount(to); j++) {
                if (StringUtil::CIEquals(from_name, UnionType::GetMemberName(to, j))) {
                    auto child_cost = ImplicitCast(UnionType::GetMemberType(from, i),
                                                   UnionType::GetMemberType(to, j));
                    cost = MaxValue(cost, child_cost);
                    found = true;
                    break;
                }
            }
            if (!found) return -1;
        }
        return cost;
    }

    // ... 基本类型转换规则
}
```

## 8.4 转换函数实现

### 8.4.1 BoundCastInfo结构

```cpp
// src/include/duckdb/function/cast/default_casts.hpp

// 转换函数签名
typedef bool (*cast_function_t)(Vector &source, Vector &result,
                                idx_t count, CastParameters &parameters);

// 局部状态初始化函数签名
typedef unique_ptr<FunctionLocalState> (*init_cast_local_state_t)(
    CastLocalStateParameters &parameters);

struct BoundCastInfo {
    cast_function_t function;           // 转换执行函数
    init_cast_local_state_t init_local_state;  // 状态初始化
    unique_ptr<BoundCastData> cast_data;       // 绑定时数据

    BoundCastInfo Copy() const;
};
```

### 8.4.2 转换参数

```cpp
struct CastParameters {
    optional_ptr<BoundCastData> cast_data;  // 绑定数据（如子类型转换信息）
    bool strict = false;                     // 严格模式标志
    string *error_message = nullptr;         // 错误消息输出
    optional_ptr<FunctionLocalState> local_state;  // 局部状态
    bool nullify_parent = false;             // 嵌套类型转换失败时置空父级
};
```

### 8.4.3 默认转换分发

```cpp
// src/function/cast/default_casts.cpp

BoundCastInfo DefaultCasts::GetDefaultCastFunction(BindCastInput &input,
                                                   const LogicalType &source,
                                                   const LogicalType &target) {
    // 目标为VARIANT时的特殊处理
    if (target.id() == LogicalTypeId::VARIANT) {
        return ImplicitToVariantCast(input, source, target);
    }

    // 任意类型到UNION的隐式转换
    if (source.id() != LogicalTypeId::UNION &&
        target.id() == LogicalTypeId::UNION) {
        return ImplicitToUnionCast(input, source, target);
    }

    // 根据源类型分发
    switch (source.id()) {
    case LogicalTypeId::BOOLEAN:
    case LogicalTypeId::TINYINT:
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
        return NumericCastSwitch(input, source, target);
    case LogicalTypeId::DECIMAL:
        return DecimalCastSwitch(input, source, target);
    case LogicalTypeId::DATE:
        return DateCastSwitch(input, source, target);
    case LogicalTypeId::TIMESTAMP:
        return TimestampCastSwitch(input, source, target);
    case LogicalTypeId::VARCHAR:
        return StringCastSwitch(input, source, target);
    case LogicalTypeId::LIST:
        return ListCastSwitch(input, source, target);
    case LogicalTypeId::STRUCT:
        return StructCastSwitch(input, source, target);
    case LogicalTypeId::MAP:
        return MapCastSwitch(input, source, target);
    // ... 其他类型
    default:
        return nullptr;
    }
}
```

## 8.5 向量化转换实现

### 8.5.1 VectorCastHelpers工具类

```cpp
// src/include/duckdb/function/cast/vector_cast_helpers.hpp

struct VectorCastHelpers {
    // 基本转换循环（无错误检查）
    template <class SRC, class DST, class OP>
    static bool TemplatedCastLoop(Vector &source, Vector &result,
                                  idx_t count, CastParameters &parameters) {
        UnaryExecutor::Execute<SRC, DST, OP>(source, result, count);
        return true;
    }

    // TryCast循环（带错误处理）
    template <class SRC, class DST, class OP>
    static bool TryCastLoop(Vector &source, Vector &result,
                            idx_t count, CastParameters &parameters) {
        VectorTryCastData input(result, parameters);
        UnaryExecutor::GenericExecute<SRC, DST, VectorTryCastOperator<OP>>(
            source, result, count, &input, parameters.error_message);
        return input.all_converted;
    }

    // 字符串转换
    template <class SRC, class OP, class RES = string_t>
    static bool StringCast(Vector &source, Vector &result,
                          idx_t count, CastParameters &parameters) {
        UnaryExecutor::GenericExecute<SRC, RES, VectorStringCastOperator<OP>>(
            source, result, count, (void *)&result);
        return true;
    }

    // DECIMAL转换
    template <class T>
    static bool ToDecimalCast(Vector &source, Vector &result,
                              idx_t count, CastParameters &parameters) {
        auto width = DecimalType::GetWidth(result.GetType());
        auto scale = DecimalType::GetScale(result.GetType());
        switch (result.GetType().InternalType()) {
        case PhysicalType::INT16:
            return TemplatedDecimalCast<T, int16_t, TryCastToDecimal>(
                source, result, count, parameters, width, scale);
        case PhysicalType::INT32:
            return TemplatedDecimalCast<T, int32_t, TryCastToDecimal>(
                source, result, count, parameters, width, scale);
        case PhysicalType::INT64:
            return TemplatedDecimalCast<T, int64_t, TryCastToDecimal>(
                source, result, count, parameters, width, scale);
        case PhysicalType::INT128:
            return TemplatedDecimalCast<T, hugeint_t, TryCastToDecimal>(
                source, result, count, parameters, width, scale);
        }
    }
};
```

### 8.5.2 TryCast操作符

```cpp
// 基本TryCast操作符（带错误处理）
template <class OP>
struct VectorTryCastOperator {
    template <class INPUT_TYPE, class RESULT_TYPE>
    static RESULT_TYPE Operation(INPUT_TYPE input, ValidityMask &mask,
                                 idx_t idx, void *dataptr) {
        RESULT_TYPE output;
        if (DUCKDB_LIKELY(OP::template Operation<INPUT_TYPE, RESULT_TYPE>(
                input, output))) {
            return output;
        }
        // 转换失败：记录错误
        auto data = reinterpret_cast<VectorTryCastData *>(dataptr);
        return HandleVectorCastError::Operation<RESULT_TYPE>(
            CastExceptionText<INPUT_TYPE, RESULT_TYPE>(input),
            mask, idx, *data);
    }
};

// 严格模式TryCast操作符
template <class OP>
struct VectorTryCastStrictOperator {
    template <class INPUT_TYPE, class RESULT_TYPE>
    static RESULT_TYPE Operation(INPUT_TYPE input, ValidityMask &mask,
                                 idx_t idx, void *dataptr) {
        auto data = reinterpret_cast<VectorTryCastData *>(dataptr);
        RESULT_TYPE output;
        // 传递strict参数
        if (DUCKDB_LIKELY(OP::template Operation<INPUT_TYPE, RESULT_TYPE>(
                input, output, data->parameters.strict))) {
            return output;
        }
        return HandleVectorCastError::Operation<RESULT_TYPE>(
            CastExceptionText<INPUT_TYPE, RESULT_TYPE>(input),
            mask, idx, *data);
    }
};
```

## 8.6 数值类型转换

### 8.6.1 数值转换矩阵

```cpp
// src/function/cast/numeric_casts.cpp

template <class SRC>
static BoundCastInfo InternalNumericCastSwitch(const LogicalType &source,
                                               const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::BOOLEAN:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, bool, NumericTryCast>);
    case LogicalTypeId::TINYINT:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, int8_t, NumericTryCast>);
    case LogicalTypeId::SMALLINT:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, int16_t, NumericTryCast>);
    case LogicalTypeId::INTEGER:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, int32_t, NumericTryCast>);
    case LogicalTypeId::BIGINT:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, int64_t, NumericTryCast>);
    case LogicalTypeId::HUGEINT:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, hugeint_t, NumericTryCast>);
    case LogicalTypeId::FLOAT:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, float, NumericTryCast>);
    case LogicalTypeId::DOUBLE:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<SRC, double, NumericTryCast>);
    case LogicalTypeId::DECIMAL:
        return BoundCastInfo(&VectorCastHelpers::ToDecimalCast<SRC>);
    case LogicalTypeId::VARCHAR:
        return BoundCastInfo(&VectorCastHelpers::StringCast<SRC, StringCast>);
    default:
        return DefaultCasts::TryVectorNullCast;
    }
}
```

### 8.6.2 DECIMAL转换特殊处理

DECIMAL转换需要处理精度和小数位数的变化：

```cpp
// src/function/cast/decimal_cast.cpp

// DECIMAL到DECIMAL：可能需要放大或缩小
template <class SOURCE, class POWERS_SOURCE>
static bool DecimalDecimalCastSwitch(Vector &source, Vector &result,
                                     idx_t count, CastParameters &parameters) {
    auto source_scale = DecimalType::GetScale(source.GetType());
    auto result_scale = DecimalType::GetScale(result.GetType());

    if (result_scale >= source_scale) {
        // 放大：乘以10的幂
        switch (result.GetType().InternalType()) {
        case PhysicalType::INT16:
            return TemplatedDecimalScaleUp<SOURCE, int16_t, POWERS_SOURCE, NumericHelper>(
                source, result, count, parameters);
        // ... 其他物理类型
        }
    } else {
        // 缩小：除以10的幂（带四舍五入）
        switch (result.GetType().InternalType()) {
        case PhysicalType::INT16:
            return TemplatedDecimalScaleDown<SOURCE, int16_t, POWERS_SOURCE>(
                source, result, count, parameters);
        // ... 其他物理类型
        }
    }
}

// 缩小时的四舍五入逻辑
struct DecimalScaleDownOperator {
    template <class INPUT_TYPE, class RESULT_TYPE>
    static RESULT_TYPE Operation(INPUT_TYPE input, ValidityMask &mask,
                                 idx_t idx, void *dataptr) {
        auto data = (DecimalScaleInput<INPUT_TYPE> *)dataptr;
        // 缩放后四舍五入
        const auto scaling = data->factor / 2;
        input /= scaling;
        if (input < 0) {
            input -= 1;
        } else {
            input += 1;
        }
        return Cast::Operation<INPUT_TYPE, RESULT_TYPE>(input / 2);
    }
};
```

## 8.7 字符串类型转换

### 8.7.1 字符串到其他类型

```cpp
// src/function/cast/string_cast.cpp

BoundCastInfo DefaultCasts::StringCastSwitch(BindCastInput &input,
                                             const LogicalType &source,
                                             const LogicalType &target) {
    switch (target.id()) {
    // 日期时间类型
    case LogicalTypeId::DATE:
        return BoundCastInfo(&VectorCastHelpers::TryCastErrorLoop<
            string_t, date_t, TryCastErrorMessage>);
    case LogicalTypeId::TIMESTAMP:
        return BoundCastInfo(&VectorCastHelpers::TryCastErrorLoop<
            string_t, timestamp_t, TryCastErrorMessage>);

    // 数值类型
    case LogicalTypeId::INTEGER:
        return BoundCastInfo(&VectorCastHelpers::TryCastStrictLoop<
            string_t, int32_t, TryCast>);
    case LogicalTypeId::DECIMAL:
        return BoundCastInfo(&VectorCastHelpers::ToDecimalCast<string_t>);

    // 二进制类型
    case LogicalTypeId::BLOB:
        return BoundCastInfo(&VectorCastHelpers::TryCastStringLoop<
            string_t, string_t, TryCastToBlob>);
    case LogicalTypeId::UUID:
        return BoundCastInfo(&VectorCastHelpers::TryCastStringLoop<
            string_t, hugeint_t, TryCastToUUID>);

    // 嵌套类型（需要绑定数据）
    case LogicalTypeId::LIST:
        return BoundCastInfo(
            &StringToNestedTypeCast<VectorStringToList>,
            ListBoundCastData::BindListToListCast(
                input, LogicalType::LIST(LogicalType::VARCHAR), target),
            ListBoundCastData::InitListLocalState);
    case LogicalTypeId::STRUCT:
        return BoundCastInfo(
            &StringToNestedTypeCast<VectorStringToStruct>,
            StructBoundCastData::BindStructToStructCast(
                input, InitVarcharStructType(target), target),
            StructBoundCastData::InitStructCastLocalState);

    default:
        return VectorStringCastNumericSwitch(input, source, target);
    }
}
```

### 8.7.2 字符串到LIST解析

```cpp
bool VectorStringToList::StringToNestedTypeCastLoop(
    const string_t *source_data, ValidityMask &source_mask,
    Vector &result, ValidityMask &result_mask, idx_t count,
    CastParameters &parameters, const SelectionVector *sel) {

    // 第一遍：计算总元素数
    idx_t total_list_size = 0;
    for (idx_t i = 0; i < count; i++) {
        idx_t idx = sel ? sel->get_index(i) : i;
        if (source_mask.RowIsValid(idx)) {
            total_list_size += VectorStringToList::CountPartsList(source_data[idx]);
        }
    }

    // 分配临时VARCHAR向量存储解析结果
    Vector varchar_vector(LogicalType::VARCHAR, total_list_size);
    ListVector::Reserve(result, total_list_size);

    // 第二遍：解析字符串并填充子元素
    auto list_data = ListVector::GetData(result);
    auto child_data = FlatVector::GetData<string_t>(varchar_vector);
    idx_t total = 0;

    for (idx_t i = 0; i < count; i++) {
        idx_t idx = sel ? sel->get_index(i) : i;
        if (!source_mask.RowIsValid(idx)) {
            result_mask.SetInvalid(i);
            continue;
        }

        list_data[i].offset = total;
        // 解析如 "[1, 2, 3]" 格式的字符串
        if (!VectorStringToList::SplitStringList(
                source_data[idx], child_data, total, varchar_vector)) {
            // 解析失败处理
            HandleVectorCastError::Operation<string_t>(error, result_mask, i, vector_cast_data);
        }
        list_data[i].length = total - list_data[i].offset;
    }

    // 转换子元素类型
    auto &result_child = ListVector::GetEntry(result);
    auto &cast_data = parameters.cast_data->Cast<ListBoundCastData>();
    return cast_data.child_cast_info.function(
        varchar_vector, result_child, total_list_size, child_parameters);
}
```

## 8.8 嵌套类型转换

### 8.8.1 LIST转换

```cpp
// src/function/cast/list_casts.cpp

// LIST到LIST转换（子类型转换）
bool ListCast::ListToListCast(Vector &source, Vector &result,
                              idx_t count, CastParameters &parameters) {
    auto &cast_data = parameters.cast_data->Cast<ListBoundCastData>();

    // 处理CONSTANT_VECTOR
    if (source.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
        if (ConstantVector::IsNull(source)) {
            ConstantVector::SetNull(result, true);
        } else {
            // 复制list_entry_t结构
            auto ldata = ConstantVector::GetData<list_entry_t>(source);
            auto tdata = ConstantVector::GetData<list_entry_t>(result);
            *tdata = *ldata;
        }
    } else {
        // FLAT_VECTOR处理
        source.Flatten(count);
        result.SetVectorType(VectorType::FLAT_VECTOR);
        FlatVector::SetValidity(result, FlatVector::Validity(source));

        auto ldata = FlatVector::GetData<list_entry_t>(source);
        auto tdata = FlatVector::GetData<list_entry_t>(result);
        for (idx_t i = 0; i < count; i++) {
            tdata[i] = ldata[i];
        }
    }

    // 转换子向量
    auto &source_child = ListVector::GetEntry(source);
    auto source_size = ListVector::GetListSize(source);
    ListVector::Reserve(result, source_size);
    auto &result_child = ListVector::GetEntry(result);

    CastParameters child_parameters(parameters, cast_data.child_cast_info.cast_data,
                                    parameters.local_state);
    return cast_data.child_cast_info.function(
        source_child, result_child, source_size, child_parameters);
}
```

### 8.8.2 STRUCT转换绑定数据

```cpp
// src/include/duckdb/function/cast/bound_cast_data.hpp

struct StructBoundCastData : public BoundCastData {
    StructBoundCastData(vector<BoundCastInfo> child_casts,
                        LogicalType target_p,
                        vector<idx_t> source_indexes_p,
                        vector<idx_t> target_indexes_p,
                        vector<idx_t> target_null_indexes_p)
        : child_cast_info(std::move(child_casts)),
          target(std::move(target_p)),
          source_indexes(std::move(source_indexes_p)),
          target_indexes(std::move(target_indexes_p)),
          target_null_indexes(std::move(target_null_indexes_p)) {}

    vector<BoundCastInfo> child_cast_info;  // 每个字段的转换函数
    LogicalType target;
    vector<idx_t> source_indexes;    // 源字段索引映射
    vector<idx_t> target_indexes;    // 目标字段索引映射
    vector<idx_t> target_null_indexes;  // 需要填充NULL的目标字段

    unique_ptr<BoundCastData> Copy() const override {
        vector<BoundCastInfo> copy_info;
        for (auto &info : child_cast_info) {
            copy_info.push_back(info.Copy());
        }
        return make_uniq<StructBoundCastData>(std::move(copy_info), target,
            source_indexes, target_indexes, target_null_indexes);
    }
};
```

### 8.8.3 MAP转换

```cpp
struct MapBoundCastData : public BoundCastData {
    MapBoundCastData(BoundCastInfo key_cast, BoundCastInfo value_cast)
        : key_cast(std::move(key_cast)),
          value_cast(std::move(value_cast)) {}

    BoundCastInfo key_cast;    // 键转换函数
    BoundCastInfo value_cast;  // 值转换函数

    static unique_ptr<BoundCastData> BindMapToMapCast(
        BindCastInput &input,
        const LogicalType &source,
        const LogicalType &target);
};
```

## 8.9 日期时间类型转换

### 8.9.1 时间戳转换矩阵

```cpp
// src/function/cast/time_casts.cpp

BoundCastInfo DefaultCasts::TimestampCastSwitch(BindCastInput &input,
                                                const LogicalType &source,
                                                const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::VARCHAR:
        return BoundCastInfo(&VectorCastHelpers::StringCast<timestamp_t, StringCast>);
    case LogicalTypeId::DATE:
        return BoundCastInfo(&VectorCastHelpers::TemplatedCastLoop<
            timestamp_t, date_t, Cast>);
    case LogicalTypeId::TIME:
        return BoundCastInfo(&VectorCastHelpers::TemplatedCastLoop<
            timestamp_t, dtime_t, Cast>);
    case LogicalTypeId::TIMESTAMP_TZ:
        // 微秒时间戳到带时区时间戳：重新解释（内部表示相同）
        return ReinterpretCast;
    case LogicalTypeId::TIMESTAMP_NS:
        // 微秒到纳秒：乘以1000
        return BoundCastInfo(&VectorCastHelpers::TemplatedCastLoop<
            timestamp_t, timestamp_t, CastTimestampUsToNs>);
    case LogicalTypeId::TIMESTAMP_MS:
        // 微秒到毫秒：除以1000
        return BoundCastInfo(&VectorCastHelpers::TemplatedCastLoop<
            timestamp_t, timestamp_t, CastTimestampUsToMs>);
    default:
        return TryVectorNullCast;
    }
}
```

### 8.9.2 DATE到TIMESTAMP转换

```cpp
BoundCastInfo DefaultCasts::DateCastSwitch(BindCastInput &input,
                                           const LogicalType &source,
                                           const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::VARCHAR:
        return BoundCastInfo(&VectorCastHelpers::StringCast<date_t, StringCast>);
    case LogicalTypeId::TIMESTAMP:
    case LogicalTypeId::TIMESTAMP_TZ:
        // DATE到TIMESTAMP：当天午夜
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<
            date_t, timestamp_t, TryCast>);
    case LogicalTypeId::TIMESTAMP_NS:
        return BoundCastInfo(&VectorCastHelpers::TryCastLoop<
            date_t, timestamp_ns_t, TryCastToTimestampNS>);
    default:
        return TryVectorNullCast;
    }
}
```

## 8.10 特殊转换

### 8.10.1 NopCast（无操作转换）

```cpp
bool DefaultCasts::NopCast(Vector &source, Vector &result,
                           idx_t count, CastParameters &parameters) {
    // 直接引用源向量，不做任何转换
    result.Reference(source);
    return true;
}
```

### 8.10.2 ReinterpretCast（重新解释转换）

```cpp
bool DefaultCasts::ReinterpretCast(Vector &source, Vector &result,
                                   idx_t count, CastParameters &parameters) {
    // 重新解释底层数据（物理表示相同）
    result.Reinterpret(source);
    return true;
}
```

### 8.10.3 TryVectorNullCast（NULL值转换）

```cpp
bool DefaultCasts::TryVectorNullCast(Vector &source, Vector &result,
                                     idx_t count, CastParameters &parameters) {
    bool success = true;
    // 只有所有值为NULL时才成功
    if (VectorOperations::HasNotNull(source, count)) {
        HandleCastError::AssignError(
            TryCast::UnimplementedCastMessage(source.GetType(), result.GetType()),
            parameters);
        success = false;
    }
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
    ConstantVector::SetNull(result, true);
    return success;
}
```

## 8.11 错误处理

### 8.11.1 HandleCastError

```cpp
void HandleCastError::AssignError(const string &error_message,
                                  CastParameters &parameters) {
    string column;
    if (parameters.cast_source && parameters.cast_source->HasAlias()) {
        column = " when casting from source column " + parameters.cast_source->alias;
    }

    if (!parameters.error_message) {
        // 严格模式：直接抛出异常
        throw ConversionException(parameters.query_location,
                                  error_message + column);
    }

    // 非严格模式：记录错误消息
    if (parameters.error_message->empty()) {
        *parameters.error_message = error_message + column;
    }
}
```

### 8.11.2 VectorTryCastData

```cpp
struct VectorTryCastData {
    VectorTryCastData(Vector &result, CastParameters &parameters)
        : result(result), parameters(parameters), all_converted(true) {}

    Vector &result;
    CastParameters &parameters;
    bool all_converted;  // 是否所有值都转换成功
};

struct HandleVectorCastError {
    template <class T>
    static T Operation(string error, ValidityMask &mask, idx_t idx,
                       VectorTryCastData &cast_data) {
        HandleCastError::AssignError(error, cast_data.parameters);
        cast_data.all_converted = false;
        mask.SetInvalid(idx);  // 标记该位置为NULL
        return NullValue<T>();
    }
};
```

## 8.12 扩展自定义转换

### 8.12.1 注册自定义转换函数

```cpp
// 示例：注册自定义类型到VARCHAR的转换
void RegisterCustomCast(DatabaseInstance &db) {
    auto &cast_functions = CastFunctionSet::Get(db);

    // 定义转换函数
    cast_function_t custom_to_varchar = [](Vector &source, Vector &result,
                                           idx_t count, CastParameters &params) -> bool {
        // 实现自定义转换逻辑
        auto source_data = FlatVector::GetData<custom_type_t>(source);
        auto result_data = FlatVector::GetData<string_t>(result);

        for (idx_t i = 0; i < count; i++) {
            if (FlatVector::IsNull(source, i)) {
                FlatVector::SetNull(result, i, true);
            } else {
                string str = custom_type_to_string(source_data[i]);
                result_data[i] = StringVector::AddString(result, str);
            }
        }
        return true;
    };

    // 注册转换函数（隐式转换代价为100）
    cast_functions.RegisterCastFunction(
        LogicalType::ANY,  // 可以使用具体类型
        LogicalType::VARCHAR,
        BoundCastInfo(custom_to_varchar),
        100  // 隐式转换代价
    );
}
```

### 8.12.2 绑定时转换函数

```cpp
// 需要在绑定时确定具体转换逻辑的情况
bind_cast_function_t custom_bind = [](BindCastInput &input,
                                      const LogicalType &source,
                                      const LogicalType &target) -> BoundCastInfo {
    // 根据具体类型信息选择转换策略
    if (source.id() == LogicalTypeId::LIST) {
        auto child_type = ListType::GetChildType(source);
        // 获取子类型转换函数
        auto child_cast = input.GetCastFunction(child_type, LogicalType::VARCHAR);
        // 返回带绑定数据的转换函数
        return BoundCastInfo(list_to_varchar_cast,
                             make_uniq<ListBoundCastData>(std::move(child_cast)),
                             ListBoundCastData::InitListLocalState);
    }
    return nullptr;  // 不处理
};

cast_functions.RegisterCastFunction(source_type, target_type, custom_bind, -1);
```

## 8.13 性能优化

### 8.13.1 常量向量优化

转换函数会检查源向量类型，对常量向量进行特殊处理：

```cpp
bool SomeCast(Vector &source, Vector &result, idx_t count, CastParameters &params) {
    if (source.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
        if (ConstantVector::IsNull(source)) {
            ConstantVector::SetNull(result, true);
            return true;
        }
        // 只转换一个值
        auto src = ConstantVector::GetData<SRC>(source);
        auto dst = ConstantVector::GetData<DST>(result);
        // ... 转换单个值
        return true;
    }

    // FLAT_VECTOR处理...
}
```

### 8.13.2 批量转换

使用UnaryExecutor进行高效的批量转换：

```cpp
template <class SRC, class DST, class OP>
static void Execute(Vector &source, Vector &result, idx_t count) {
    UnifiedVectorFormat sdata;
    source.ToUnifiedFormat(count, sdata);

    auto src_data = UnifiedVectorFormat::GetData<SRC>(sdata);
    auto dst_data = FlatVector::GetData<DST>(result);

    if (!sdata.validity.AllValid()) {
        // 有NULL值：需要检查每行
        for (idx_t i = 0; i < count; i++) {
            auto idx = sdata.sel->get_index(i);
            if (sdata.validity.RowIsValid(idx)) {
                dst_data[i] = OP::Operation(src_data[idx]);
            } else {
                FlatVector::SetNull(result, i, true);
            }
        }
    } else {
        // 无NULL值：直接转换
        for (idx_t i = 0; i < count; i++) {
            auto idx = sdata.sel->get_index(i);
            dst_data[i] = OP::Operation(src_data[idx]);
        }
    }
}
```

## 8.14 小结

DuckDB的类型转换系统通过精心设计实现了以下目标：

1. **灵活的转换注册机制**：支持默认转换和自定义转换的注册与覆盖
2. **高效的向量化执行**：所有转换都基于向量操作，避免逐行处理开销
3. **完整的错误处理**：支持严格模式和宽松模式，提供详细的错误信息
4. **智能的隐式转换**：基于代价的类型推断系统选择最优转换路径
5. **嵌套类型支持**：通过绑定数据和局部状态支持复杂嵌套类型的递归转换

类型转换系统与类型系统紧密集成，为DuckDB的SQL处理和表达式求值提供了坚实的基础。

# 第八章：类型转换函数体系

## 8.1 类型转换概述

类型转换是数据库系统中的基础功能，DuckDB 实现了一套完整的类型转换体系，支持数百种类型之间的转换。这套体系既支持隐式转换（自动类型提升），也支持显式转换（CAST 表达式）。

### 8.1.1 类型转换分类

```
┌──────────────────────────────────────────────────────────────────────┐
│                      类型转换分类                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 隐式转换（Implicit Cast）                                        │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ - 函数参数类型匹配                                          │  │
│     │ - 表达式类型统一                                            │  │
│     │ - INSERT 数据类型对齐                                       │  │
│     │ - 转换代价决定优先级                                         │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  2. 显式转换（Explicit Cast）                                        │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ - CAST(expr AS type)                                       │  │
│     │ - expr::type                                               │  │
│     │ - 用户明确指定目标类型                                       │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  3. TryCast（宽松转换）                                              │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ - TRY_CAST(expr AS type)                                   │  │
│     │ - 转换失败返回 NULL 而非抛异常                               │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.1.2 核心类结构

```cpp
// 绑定后的转换信息
struct BoundCastInfo {
    cast_function_t function;                    // 转换函数
    init_cast_local_state_t init_local_state;    // 局部状态初始化
    unique_ptr<BoundCastData> cast_data;         // 转换数据

    BoundCastInfo Copy() const;
};

// 转换函数集合
class CastFunctionSet {
public:
    //! 获取转换函数
    BoundCastInfo GetCastFunction(const LogicalType &source, const LogicalType &target,
                                  GetCastFunctionInput &input);

    //! 计算隐式转换代价
    int64_t ImplicitCastCost(optional_ptr<ClientContext> context,
                             const LogicalType &source, const LogicalType &target);

    //! 注册自定义转换函数
    void RegisterCastFunction(const LogicalType &source, const LogicalType &target,
                              BoundCastInfo function, int64_t implicit_cast_cost = -1);
};
```

## 8.2 CastFunctionSet 架构

### 8.2.1 转换函数查找

```cpp
// src/function/cast/cast_function_set.cpp
BoundCastInfo CastFunctionSet::GetCastFunction(const LogicalType &source,
                                                const LogicalType &target,
                                                GetCastFunctionInput &get_input) {
    // 相同类型：无操作转换
    if (source == target) {
        return DefaultCasts::NopCast;
    }

    // 从后向前遍历绑定函数（后注册的优先级高）
    for (idx_t i = bind_functions.size(); i > 0; i--) {
        auto &bind_function = bind_functions[i - 1];
        BindCastInput input(*this, bind_function.info.get(), get_input.context);
        input.query_location = get_input.query_location;

        auto result = bind_function.function(input, source, target);
        if (result.function) {
            return result;
        }
    }

    // 未找到转换函数：返回 NULL 转换
    return DefaultCasts::TryVectorNullCast;
}
```

### 8.2.2 隐式转换代价计算

```cpp
int64_t CastFunctionSet::ImplicitCastCost(optional_ptr<ClientContext> context,
                                           const LogicalType &source,
                                           const LogicalType &target) {
    // 首先检查已注册的自定义转换
    if (map_info) {
        auto entry = map_info->GetEntry(source, target);
        if (entry) {
            return entry->implicit_cast_cost;
        }
    }

    // 回退到默认隐式转换规则
    auto score = CastRules::ImplicitCast(source, target);

    // 特殊处理：旧版隐式转换到 VARCHAR
    if (score < 0 && source.id() != LogicalTypeId::BLOB &&
        target.id() == LogicalTypeId::VARCHAR) {
        if (context && DBConfig::GetSetting<OldImplicitCastingSetting>(*context)) {
            score = 10000000000;  // 非常高的代价
        }
    }

    return score;
}
```

### 8.2.3 自定义转换注册

```cpp
void CastFunctionSet::RegisterCastFunction(const LogicalType &source,
                                            const LogicalType &target,
                                            BoundCastInfo function,
                                            int64_t implicit_cast_cost) {
    RegisterCastFunction(source, target,
                         MapCastNode(std::move(function), implicit_cast_cost));
}

void CastFunctionSet::RegisterCastFunction(const LogicalType &source,
                                            const LogicalType &target,
                                            MapCastNode node) {
    if (!map_info) {
        // 创建转换映射表
        auto info = make_uniq<MapCastInfo>();
        map_info = info.get();
        bind_functions.emplace_back(MapCastFunction, std::move(info));
    }
    map_info->AddEntry(source, target, std::move(node));
}
```

## 8.3 DefaultCasts 默认转换

### 8.3.1 类型分发机制

```cpp
// src/function/cast/default_casts.cpp
BoundCastInfo DefaultCasts::GetDefaultCastFunction(BindCastInput &input,
                                                    const LogicalType &source,
                                                    const LogicalType &target) {
    D_ASSERT(source != target);

    // 特殊处理：转换到 VARIANT
    if (target.id() == LogicalTypeId::VARIANT) {
        return ImplicitToVariantCast(input, source, target);
    }

    // 特殊处理：转换到 UNION
    if (source.id() != LogicalTypeId::UNION &&
        source.id() != LogicalTypeId::SQLNULL &&
        target.id() == LogicalTypeId::UNION) {
        return ImplicitToUnionCast(input, source, target);
    }

    // 根据源类型分发
    switch (source.id()) {
    // 数值类型
    case LogicalTypeId::BOOLEAN:
    case LogicalTypeId::TINYINT:
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::UTINYINT:
    case LogicalTypeId::USMALLINT:
    case LogicalTypeId::UINTEGER:
    case LogicalTypeId::UBIGINT:
    case LogicalTypeId::UHUGEINT:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
        return NumericCastSwitch(input, source, target);

    // 小数类型
    case LogicalTypeId::DECIMAL:
        return DecimalCastSwitch(input, source, target);

    // 时间类型
    case LogicalTypeId::DATE:
        return DateCastSwitch(input, source, target);
    case LogicalTypeId::TIME:
        return TimeCastSwitch(input, source, target);
    case LogicalTypeId::TIMESTAMP:
        return TimestampCastSwitch(input, source, target);
    case LogicalTypeId::INTERVAL:
        return IntervalCastSwitch(input, source, target);

    // 字符串类型
    case LogicalTypeId::VARCHAR:
        return StringCastSwitch(input, source, target);
    case LogicalTypeId::BLOB:
        return BlobCastSwitch(input, source, target);

    // 复合类型
    case LogicalTypeId::LIST:
        return ListCastSwitch(input, source, target);
    case LogicalTypeId::STRUCT:
        return StructCastSwitch(input, source, target);
    case LogicalTypeId::MAP:
        return MapCastSwitch(input, source, target);
    case LogicalTypeId::ARRAY:
        return ArrayCastSwitch(input, source, target);
    case LogicalTypeId::UNION:
        return UnionCastSwitch(input, source, target);

    // 特殊类型
    case LogicalTypeId::SQLNULL:
        return NullTypeCast;
    case LogicalTypeId::ENUM:
        return EnumCastSwitch(input, source, target);
    case LogicalTypeId::UUID:
        return UUIDCastSwitch(input, source, target);

    default:
        return nullptr;
    }
}
```

### 8.3.2 特殊转换函数

```cpp
// 无操作转换（相同类型）
bool DefaultCasts::NopCast(Vector &source, Vector &result, idx_t count,
                            CastParameters &parameters) {
    result.Reference(source);
    return true;
}

// NULL 类型转换
static bool NullTypeCast(Vector &source, Vector &result, idx_t count,
                          CastParameters &parameters) {
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
    ConstantVector::SetNull(result, true);
    return true;
}

// 尝试 NULL 转换（仅当所有值都是 NULL 时成功）
bool DefaultCasts::TryVectorNullCast(Vector &source, Vector &result, idx_t count,
                                      CastParameters &parameters) {
    bool success = true;
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

// 重解释转换（二进制兼容类型）
bool DefaultCasts::ReinterpretCast(Vector &source, Vector &result, idx_t count,
                                    CastParameters &parameters) {
    result.Reinterpret(source);
    return true;
}
```

## 8.4 数值类型转换

### 8.4.1 NumericCastSwitch

```cpp
// src/function/cast/numeric_casts.cpp
BoundCastInfo NumericCastSwitch(BindCastInput &input, const LogicalType &source,
                                 const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::BOOLEAN:
        return NumericToBoolCast<SRC_TYPE>(source);
    case LogicalTypeId::TINYINT:
        return NumericToNumericCast<SRC_TYPE, int8_t>(source, target);
    case LogicalTypeId::SMALLINT:
        return NumericToNumericCast<SRC_TYPE, int16_t>(source, target);
    case LogicalTypeId::INTEGER:
        return NumericToNumericCast<SRC_TYPE, int32_t>(source, target);
    case LogicalTypeId::BIGINT:
        return NumericToNumericCast<SRC_TYPE, int64_t>(source, target);
    case LogicalTypeId::HUGEINT:
        return NumericToNumericCast<SRC_TYPE, hugeint_t>(source, target);
    case LogicalTypeId::FLOAT:
        return NumericToNumericCast<SRC_TYPE, float>(source, target);
    case LogicalTypeId::DOUBLE:
        return NumericToNumericCast<SRC_TYPE, double>(source, target);
    case LogicalTypeId::DECIMAL:
        return NumericToDecimalCast<SRC_TYPE>(source, target);
    case LogicalTypeId::VARCHAR:
        return NumericToStringCast<SRC_TYPE>(source);
    // ...
    default:
        return nullptr;
    }
}
```

### 8.4.2 数值类型转换模板

```cpp
template <class SRC_TYPE, class DST_TYPE>
static bool NumericToNumericCastFunction(Vector &source, Vector &result, idx_t count,
                                          CastParameters &parameters) {
    if (std::is_same<SRC_TYPE, DST_TYPE>::value) {
        // 相同类型直接引用
        result.Reference(source);
        return true;
    }

    // 使用通用执行器进行转换
    UnaryExecutor::Execute<SRC_TYPE, DST_TYPE>(
        source, result, count,
        [&](SRC_TYPE input) -> DST_TYPE {
            DST_TYPE output;
            if (!TryCast::Operation<SRC_TYPE, DST_TYPE>(input, output)) {
                HandleCastError::AssignError(
                    CastExceptionMessage<SRC_TYPE, DST_TYPE>(input), parameters);
            }
            return output;
        });
    return true;
}
```

### 8.4.3 溢出检查

```cpp
template <class SRC, class DST>
bool TryCast::Operation(SRC input, DST &result) {
    // 检查范围
    if (input < NumericLimits<DST>::Minimum() ||
        input > NumericLimits<DST>::Maximum()) {
        return false;
    }
    result = static_cast<DST>(input);
    return true;
}

// 特化：BIGINT → INTEGER
template <>
bool TryCast::Operation(int64_t input, int32_t &result) {
    if (input < NumericLimits<int32_t>::Minimum() ||
        input > NumericLimits<int32_t>::Maximum()) {
        return false;
    }
    result = static_cast<int32_t>(input);
    return true;
}
```

## 8.5 字符串转换

### 8.5.1 StringCastSwitch

```cpp
// src/function/cast/string_cast.cpp
BoundCastInfo StringCastSwitch(BindCastInput &input, const LogicalType &source,
                                const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::DATE:
        return StringToDateCast(input, source, target);
    case LogicalTypeId::TIME:
        return StringToTimeCast(input, source, target);
    case LogicalTypeId::TIMESTAMP:
        return StringToTimestampCast(input, source, target);
    case LogicalTypeId::BOOLEAN:
        return StringToBoolCast;
    case LogicalTypeId::TINYINT:
        return StringToNumericCast<int8_t>();
    case LogicalTypeId::SMALLINT:
        return StringToNumericCast<int16_t>();
    case LogicalTypeId::INTEGER:
        return StringToNumericCast<int32_t>();
    case LogicalTypeId::BIGINT:
        return StringToNumericCast<int64_t>();
    case LogicalTypeId::FLOAT:
        return StringToNumericCast<float>();
    case LogicalTypeId::DOUBLE:
        return StringToNumericCast<double>();
    case LogicalTypeId::DECIMAL:
        return StringToDecimalCast(input, source, target);
    case LogicalTypeId::UUID:
        return StringToUUIDCast;
    case LogicalTypeId::LIST:
        return StringToListCast(input, source, target);
    case LogicalTypeId::STRUCT:
        return StringToStructCast(input, source, target);
    case LogicalTypeId::MAP:
        return StringToMapCast(input, source, target);
    // ...
    default:
        return nullptr;
    }
}
```

### 8.5.2 字符串解析模板

```cpp
template <class T>
bool TrySimpleIntegerCast(const char *buf, idx_t len, T &result, bool strict) {
    // 跳过空白
    idx_t pos = 0;
    while (pos < len && StringUtil::CharacterIsSpace(buf[pos])) {
        pos++;
    }

    // 解析符号
    bool negative = false;
    if (pos < len && buf[pos] == '-') {
        negative = true;
        pos++;
    } else if (pos < len && buf[pos] == '+') {
        pos++;
    }

    // 解析数字
    T value = 0;
    bool has_digit = false;
    while (pos < len && StringUtil::CharacterIsDigit(buf[pos])) {
        has_digit = true;
        T digit = buf[pos] - '0';

        // 溢出检查
        if (value > (NumericLimits<T>::Maximum() - digit) / 10) {
            return false;
        }
        value = value * 10 + digit;
        pos++;
    }

    if (!has_digit) {
        return false;
    }

    // 严格模式：不允许尾随字符
    if (strict) {
        while (pos < len && StringUtil::CharacterIsSpace(buf[pos])) {
            pos++;
        }
        if (pos != len) {
            return false;
        }
    }

    result = negative ? -value : value;
    return true;
}
```

## 8.6 复合类型转换

### 8.6.1 LIST 转换

```cpp
// src/function/cast/list_casts.cpp
BoundCastInfo ListCastSwitch(BindCastInput &input, const LogicalType &source,
                              const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::LIST:
        return ListToListCast(input, source, target);
    case LogicalTypeId::ARRAY:
        return ListToArrayCast(input, source, target);
    case LogicalTypeId::VARCHAR:
        return ListToVarcharCast(input, source, target);
    default:
        return nullptr;
    }
}

static BoundCastInfo ListToListCast(BindCastInput &input, const LogicalType &source,
                                     const LogicalType &target) {
    auto &source_child = ListType::GetChildType(source);
    auto &target_child = ListType::GetChildType(target);

    // 获取子元素转换函数
    auto child_cast = input.GetCastFunction(source_child, target_child);

    // 创建转换数据
    auto cast_data = make_uniq<ListBoundCastData>(std::move(child_cast));

    return BoundCastInfo(ListToListCastFunction, std::move(cast_data));
}

static bool ListToListCastFunction(Vector &source, Vector &result, idx_t count,
                                    CastParameters &parameters) {
    auto &cast_data = parameters.cast_data->Cast<ListBoundCastData>();

    // 获取列表数据
    auto &source_entries = ListVector::GetEntry(source);
    auto &result_entries = ListVector::GetEntry(result);
    auto source_size = ListVector::GetListSize(source);

    // 转换子元素
    CastParameters child_params(parameters, cast_data.child_cast_info.cast_data.get());
    if (!cast_data.child_cast_info.function(source_entries, result_entries,
                                             source_size, child_params)) {
        return false;
    }

    // 复制列表元数据
    ListVector::SetListSize(result, source_size);
    if (source.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
    }

    return true;
}
```

### 8.6.2 STRUCT 转换

```cpp
// src/function/cast/struct_cast.cpp
BoundCastInfo StructCastSwitch(BindCastInput &input, const LogicalType &source,
                                const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::STRUCT:
        return StructToStructCast(input, source, target);
    case LogicalTypeId::VARCHAR:
        return StructToVarcharCast(input, source, target);
    default:
        return nullptr;
    }
}

static BoundCastInfo StructToStructCast(BindCastInput &input, const LogicalType &source,
                                         const LogicalType &target) {
    auto &source_children = StructType::GetChildTypes(source);
    auto &target_children = StructType::GetChildTypes(target);

    if (source_children.size() != target_children.size()) {
        throw InvalidInputException("Cannot cast struct with %d fields to struct with %d fields",
                                    source_children.size(), target_children.size());
    }

    // 为每个字段创建转换函数
    vector<BoundCastInfo> child_casts;
    for (idx_t i = 0; i < source_children.size(); i++) {
        auto child_cast = input.GetCastFunction(source_children[i].second,
                                                 target_children[i].second);
        child_casts.push_back(std::move(child_cast));
    }

    auto cast_data = make_uniq<StructBoundCastData>(std::move(child_casts));
    return BoundCastInfo(StructToStructCastFunction, std::move(cast_data));
}
```

### 8.6.3 MAP 转换

```cpp
// src/function/cast/map_cast.cpp
BoundCastInfo MapCastSwitch(BindCastInput &input, const LogicalType &source,
                             const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::MAP:
        return MapToMapCast(input, source, target);
    case LogicalTypeId::VARCHAR:
        return MapToVarcharCast(input, source, target);
    case LogicalTypeId::STRUCT:
        return MapToStructCast(input, source, target);
    default:
        return nullptr;
    }
}

static BoundCastInfo MapToMapCast(BindCastInput &input, const LogicalType &source,
                                   const LogicalType &target) {
    auto &source_key = MapType::KeyType(source);
    auto &source_val = MapType::ValueType(source);
    auto &target_key = MapType::KeyType(target);
    auto &target_val = MapType::ValueType(target);

    // 获取键和值的转换函数
    auto key_cast = input.GetCastFunction(source_key, target_key);
    auto val_cast = input.GetCastFunction(source_val, target_val);

    auto cast_data = make_uniq<MapBoundCastData>(std::move(key_cast), std::move(val_cast));
    return BoundCastInfo(MapToMapCastFunction, std::move(cast_data));
}
```

## 8.7 时间类型转换

### 8.7.1 时间类型转换矩阵

```
┌──────────────────────────────────────────────────────────────────────┐
│                      时间类型转换矩阵                                  │
├───────────────────┬────────────────────────────────────────────────┤
│ 源类型            │ 可转换目标类型                                   │
├───────────────────┼────────────────────────────────────────────────┤
│ DATE              │ TIMESTAMP, TIMESTAMP_TZ, VARCHAR               │
│ TIME              │ TIME_TZ, VARCHAR                               │
│ TIMESTAMP         │ DATE, TIME, TIMESTAMP_TZ, TIMESTAMP_NS,        │
│                   │ TIMESTAMP_MS, TIMESTAMP_SEC, VARCHAR           │
│ TIMESTAMP_TZ      │ TIMESTAMP, DATE, TIME, VARCHAR                 │
│ INTERVAL          │ VARCHAR                                        │
└───────────────────┴────────────────────────────────────────────────┘
```

### 8.7.2 时间转换实现

```cpp
// src/function/cast/time_casts.cpp
BoundCastInfo DateCastSwitch(BindCastInput &input, const LogicalType &source,
                              const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::TIMESTAMP:
        return DateToTimestampCast;
    case LogicalTypeId::TIMESTAMP_TZ:
        return DateToTimestampTzCast;
    case LogicalTypeId::TIMESTAMP_NS:
        return DateToTimestampNsCast;
    case LogicalTypeId::VARCHAR:
        return DateToVarcharCast;
    default:
        return nullptr;
    }
}

static bool DateToTimestampCast(Vector &source, Vector &result, idx_t count,
                                 CastParameters &parameters) {
    UnaryExecutor::Execute<date_t, timestamp_t>(
        source, result, count,
        [](date_t date) -> timestamp_t {
            return Timestamp::FromDatetime(date, dtime_t(0));
        });
    return true;
}

BoundCastInfo TimestampCastSwitch(BindCastInput &input, const LogicalType &source,
                                   const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::DATE:
        return TimestampToDateCast;
    case LogicalTypeId::TIME:
        return TimestampToTimeCast;
    case LogicalTypeId::TIMESTAMP_TZ:
        return TimestampToTimestampTzCast(input, source, target);
    case LogicalTypeId::TIMESTAMP_NS:
        return TimestampToTimestampNsCast;
    case LogicalTypeId::TIMESTAMP_MS:
        return TimestampToTimestampMsCast;
    case LogicalTypeId::TIMESTAMP_SEC:
        return TimestampToTimestampSecCast;
    case LogicalTypeId::VARCHAR:
        return TimestampToVarcharCast;
    default:
        return nullptr;
    }
}
```

## 8.8 DECIMAL 类型转换

### 8.8.1 DECIMAL 精度处理

```cpp
// src/function/cast/decimal_cast.cpp
BoundCastInfo DecimalCastSwitch(BindCastInput &input, const LogicalType &source,
                                 const LogicalType &target) {
    auto source_width = DecimalType::GetWidth(source);
    auto source_scale = DecimalType::GetScale(source);

    switch (target.id()) {
    case LogicalTypeId::TINYINT:
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::HUGEINT:
        return DecimalToIntegerCast(input, source, target, source_width, source_scale);
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
        return DecimalToFloatCast(input, source, target, source_width, source_scale);
    case LogicalTypeId::DECIMAL:
        return DecimalToDecimalCast(input, source, target);
    case LogicalTypeId::VARCHAR:
        return DecimalToVarcharCast(source_width, source_scale);
    default:
        return nullptr;
    }
}
```

### 8.8.2 DECIMAL 精度转换

```cpp
static BoundCastInfo DecimalToDecimalCast(BindCastInput &input,
                                           const LogicalType &source,
                                           const LogicalType &target) {
    auto source_width = DecimalType::GetWidth(source);
    auto source_scale = DecimalType::GetScale(source);
    auto target_width = DecimalType::GetWidth(target);
    auto target_scale = DecimalType::GetScale(target);

    // 确定物理存储类型
    auto source_type = DecimalType::GetInternalType(source);
    auto target_type = DecimalType::GetInternalType(target);

    // 计算比例因子
    int64_t scale_diff = target_scale - source_scale;

    if (scale_diff == 0 && source_type == target_type) {
        // 无需转换
        return DefaultCasts::ReinterpretCast;
    }

    // 创建带精度信息的转换数据
    auto cast_data = make_uniq<DecimalCastData>(source_width, source_scale,
                                                 target_width, target_scale);
    return BoundCastInfo(DecimalToDecimalCastFunction, std::move(cast_data));
}
```

## 8.9 枚举类型转换

### 8.9.1 ENUM 转换

```cpp
// src/function/cast/enum_casts.cpp
BoundCastInfo EnumCastSwitch(BindCastInput &input, const LogicalType &source,
                              const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::VARCHAR:
        return EnumToVarcharCast(input, source, target);
    case LogicalTypeId::ENUM:
        return EnumToEnumCast(input, source, target);
    default:
        return nullptr;
    }
}

static BoundCastInfo EnumToVarcharCast(BindCastInput &input, const LogicalType &source,
                                        const LogicalType &target) {
    auto &enum_type = EnumType::GetValuesVector(source);

    return BoundCastInfo([&enum_type](Vector &source, Vector &result, idx_t count,
                                       CastParameters &parameters) {
        // 使用枚举值向量进行查找
        VectorOperations::Gather::Set(enum_type, source, result, count);
        return true;
    });
}
```

## 8.10 UNION 类型转换

### 8.10.1 UNION 类型转换逻辑

```cpp
// src/function/cast/union_casts.cpp
BoundCastInfo UnionCastSwitch(BindCastInput &input, const LogicalType &source,
                               const LogicalType &target) {
    switch (target.id()) {
    case LogicalTypeId::UNION:
        return UnionToUnionCast(input, source, target);
    case LogicalTypeId::VARCHAR:
        return UnionToVarcharCast(input, source, target);
    default:
        // 尝试转换为成员类型
        return UnionToMemberCast(input, source, target);
    }
}

static BoundCastInfo UnionToUnionCast(BindCastInput &input, const LogicalType &source,
                                       const LogicalType &target) {
    auto source_count = UnionType::GetMemberCount(source);
    auto target_count = UnionType::GetMemberCount(target);

    // 为每个源成员找到对应的目标成员
    vector<idx_t> member_mapping;
    vector<BoundCastInfo> member_casts;

    for (idx_t i = 0; i < source_count; i++) {
        auto &source_member_name = UnionType::GetMemberName(source, i);
        auto &source_member_type = UnionType::GetMemberType(source, i);

        bool found = false;
        for (idx_t j = 0; j < target_count; j++) {
            auto &target_member_name = UnionType::GetMemberName(target, j);
            if (StringUtil::CIEquals(source_member_name, target_member_name)) {
                member_mapping.push_back(j);
                auto &target_member_type = UnionType::GetMemberType(target, j);
                member_casts.push_back(
                    input.GetCastFunction(source_member_type, target_member_type));
                found = true;
                break;
            }
        }

        if (!found) {
            throw InvalidInputException("Cannot cast union: member '%s' not found in target",
                                        source_member_name);
        }
    }

    auto cast_data = make_uniq<UnionCastData>(std::move(member_mapping),
                                               std::move(member_casts));
    return BoundCastInfo(UnionToUnionCastFunction, std::move(cast_data));
}
```

## 8.11 错误处理

### 8.11.1 转换错误处理

```cpp
// src/function/cast/default_casts.cpp
void HandleCastError::AssignError(const string &error_message,
                                   CastParameters &parameters) {
    AssignError(error_message, parameters.error_message,
                parameters.cast_source, parameters.query_location);
}

void HandleCastError::AssignError(const string &error_message,
                                   string *error_message_ptr,
                                   optional_ptr<const Expression> cast_source,
                                   optional_idx error_location) {
    string column;
    if (cast_source && cast_source->HasAlias()) {
        column = " when casting from source column " + cast_source->alias;
    }

    if (!error_message_ptr) {
        // 严格模式：直接抛异常
        throw ConversionException(error_location, error_message + column);
    }

    // TryCast 模式：记录错误
    if (error_message_ptr->empty()) {
        *error_message_ptr = error_message + column;
    }
}
```

### 8.11.2 TryCast 实现

```cpp
// TryCast 返回布尔值表示是否成功
template <class SRC, class DST>
bool TryCastFunction(Vector &source, Vector &result, idx_t count,
                     CastParameters &parameters) {
    UnaryExecutor::ExecuteWithNulls<SRC, DST>(
        source, result, count,
        [&](SRC input, ValidityMask &mask, idx_t idx) -> DST {
            DST output;
            if (!TryCast::Operation<SRC, DST>(input, output)) {
                mask.SetInvalid(idx);
                return DST();
            }
            return output;
        });
    return true;
}
```

## 8.12 本章小结

本章深入分析了 DuckDB 的类型转换函数体系：

1. **CastFunctionSet 架构**：
   - 管理所有转换函数
   - 支持自定义转换注册
   - 计算隐式转换代价

2. **DefaultCasts 分发机制**：
   - 基于源类型的 switch 分发
   - 支持所有内置类型的转换

3. **数值类型转换**：
   - 模板化的类型安全转换
   - 溢出检查和范围验证

4. **字符串转换**：
   - 字符串到任意类型的解析
   - 任意类型到字符串的格式化

5. **复合类型转换**：
   - LIST/STRUCT/MAP/ARRAY 的递归转换
   - 子元素类型匹配和转换

6. **时间类型转换**：
   - DATE/TIME/TIMESTAMP 之间的转换
   - 时区感知转换

7. **特殊类型转换**：
   - DECIMAL 精度处理
   - ENUM 值查找
   - UNION 成员映射

8. **错误处理**：
   - 严格模式抛异常
   - TryCast 模式返回 NULL

类型转换体系是 DuckDB 类型系统的基础，支持 SQL 的灵活类型语义。

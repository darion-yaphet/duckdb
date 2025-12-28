# DuckDB 类型系统深度解析 - 第一章：类型系统架构概述

## 引言

类型系统是数据库系统的基础设施，它定义了数据的表示方式、存储格式以及操作语义。DuckDB 采用了一种精心设计的双层类型架构，将 SQL 语义层面的逻辑类型与物理存储层面的类型分离，既保证了 SQL 兼容性，又实现了高效的向量化执行。本章将深入剖析 DuckDB 类型系统的整体架构设计。

## 双层类型设计

DuckDB 的类型系统采用了**逻辑类型（LogicalType）**与**物理类型（PhysicalType）**分离的双层设计。这种设计的核心思想是：

```
┌─────────────────────────────────────────────────────────────────┐
│                      SQL 语义层                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  LogicalTypeId: BOOLEAN, INTEGER, VARCHAR, DATE, ...    │    │
│  │                 DECIMAL, LIST, STRUCT, MAP, ENUM, ...   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            ↓ 映射                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PhysicalType: BOOL, INT8/16/32/64, VARCHAR, LIST, ...  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                      物理存储层                                  │
└─────────────────────────────────────────────────────────────────┘
```

- **LogicalTypeId**：表示 SQL 语义层面的类型标识，如 `DATE`、`TIMESTAMP`、`DECIMAL(18,2)` 等
- **PhysicalType**：表示底层存储时实际使用的数据格式，如 `INT32`、`INT64`、`VARCHAR` 等

这种分离带来以下优势：

1. **语义与存储解耦**：同一个物理类型可以承载多种逻辑类型（如 `INT64` 同时用于 `BIGINT`、`TIME`、`TIMESTAMP`）
2. **类型元信息扩展**：通过 `ExtraTypeInfo` 机制，可以为类型附加额外信息（如 `DECIMAL` 的精度和小数位数）
3. **向量化执行友好**：物理类型数量有限且固定，便于实现高效的向量化操作

## PhysicalType 枚举

物理类型定义在 `src/include/duckdb/common/types.hpp` 中，表示数据在内存中的实际存储格式：

```cpp
enum class PhysicalType : uint8_t {
    BOOL = 1,      // 布尔值
    UINT8 = 2,     // 无符号8位整数
    INT8 = 3,      // 有符号8位整数
    UINT16 = 4,    // 无符号16位整数
    INT16 = 5,     // 有符号16位整数
    UINT32 = 6,    // 无符号32位整数
    INT32 = 7,     // 有符号32位整数
    UINT64 = 8,    // 无符号64位整数
    INT64 = 9,     // 有符号64位整数
    FLOAT = 10,    // 单精度浮点数
    DOUBLE = 11,   // 双精度浮点数
    UINT128 = 12,  // 无符号128位整数
    INT128 = 13,   // 有符号128位整数
    INTERVAL = 14, // 时间间隔
    VARCHAR = 15,  // 变长字符串
    // 复杂类型
    STRUCT = 17,   // 结构体（嵌套类型）
    LIST = 18,     // 列表
    ARRAY = 33,    // 固定长度数组
    BIT = 20,      // 位串
    // 特殊类型
    UNKNOWN = 21,  // 未知类型
    INVALID = 255  // 无效类型
};
```

每种物理类型都有确定的内存大小，通过 `GetTypeIdSize` 函数获取：

```cpp
idx_t GetTypeIdSize(PhysicalType type) {
    switch (type) {
    case PhysicalType::BOOL:
        return sizeof(bool);
    case PhysicalType::INT8:
        return sizeof(int8_t);
    case PhysicalType::INT16:
        return sizeof(int16_t);
    case PhysicalType::INT32:
        return sizeof(int32_t);
    case PhysicalType::INT64:
        return sizeof(int64_t);
    case PhysicalType::INT128:
        return sizeof(hugeint_t);
    case PhysicalType::VARCHAR:
        return sizeof(string_t);  // 16字节的内联字符串结构
    case PhysicalType::LIST:
        return sizeof(list_entry_t);  // offset + len
    case PhysicalType::STRUCT:
    case PhysicalType::ARRAY:
        return 0;  // 没有自己的数据，子类型决定
    // ...
    }
}
```

## LogicalTypeId 枚举

逻辑类型标识定义了 SQL 层面的类型语义：

```cpp
enum class LogicalTypeId : uint8_t {
    INVALID = 0,
    SQLNULL = 1,      // NULL 类型
    UNKNOWN = 2,      // 未解析的参数类型
    ANY = 3,          // 任意类型（用于函数重载）

    // 布尔类型
    BOOLEAN = 10,

    // 整数类型
    TINYINT = 11,     // INT8
    SMALLINT = 12,    // INT16
    INTEGER = 13,     // INT32
    BIGINT = 14,      // INT64
    UTINYINT = 46,    // UINT8
    USMALLINT = 47,   // UINT16
    UINTEGER = 48,    // UINT32
    UBIGINT = 49,     // UINT64
    HUGEINT = 50,     // INT128
    UHUGEINT = 51,    // UINT128

    // 浮点类型
    FLOAT = 20,
    DOUBLE = 21,
    DECIMAL = 22,

    // 日期时间类型
    DATE = 23,
    TIME = 24,
    TIMESTAMP = 25,
    TIME_TZ = 29,
    TIMESTAMP_TZ = 30,
    TIMESTAMP_SEC = 44,
    TIMESTAMP_MS = 31,
    TIMESTAMP_NS = 32,
    TIME_NS = 66,

    // 字符串类型
    VARCHAR = 33,
    CHAR = 53,
    BLOB = 34,
    BIT = 43,

    // 嵌套类型
    STRUCT = 37,
    LIST = 38,
    MAP = 39,
    ARRAY = 62,
    UNION = 55,
    VARIANT = 65,

    // 其他类型
    UUID = 40,
    ENUM = 42,
    INTERVAL = 35,
    USER = 52,        // 用户自定义类型

    // 特殊内部类型
    TABLE = 54,       // 表类型
    LAMBDA = 57,      // Lambda 函数
    AGGREGATE_STATE = 58,  // 聚合状态
    // ...
};
```

## LogicalType 类设计

`LogicalType` 是类型系统的核心类，封装了类型的完整信息：

```cpp
class LogicalType {
    friend struct ExtraTypeInfo;
public:
    // 构造函数
    LogicalType();
    LogicalType(LogicalTypeId id);
    LogicalType(LogicalTypeId id, shared_ptr<ExtraTypeInfo> type_info_p);

    // 类型标识访问
    LogicalTypeId id() const { return id_; }
    PhysicalType InternalType() const { return physical_type_; }

    // 辅助类型信息
    ExtraTypeInfo *AuxInfo() const { return type_info_.get(); }
    shared_ptr<ExtraTypeInfo> GetAuxInfoShrPtr() const { return type_info_; }

    // 类型属性判断
    bool IsIntegral() const;
    bool IsSigned() const;
    bool IsUnsigned() const;
    bool IsFloating() const;
    bool IsNumeric() const;
    bool IsTemporal() const;
    bool IsValid() const;
    bool IsComplete() const;

    // 类型别名
    void SetAlias(string alias);
    string GetAlias() const;
    bool HasAlias() const;

    // 扩展类型信息
    bool HasExtensionInfo() const;
    optional_ptr<const ExtensionTypeInfo> GetExtensionInfo() const;

    // 类型工厂方法
    static LogicalType DECIMAL(uint8_t width, uint8_t scale);
    static LogicalType LIST(const LogicalType &child);
    static LogicalType STRUCT(child_list_t<LogicalType> children);
    static LogicalType MAP(LogicalType key, LogicalType value);
    static LogicalType UNION(child_list_t<LogicalType> members);
    static LogicalType ARRAY(const LogicalType &child, optional_idx size);
    static LogicalType ENUM(Vector &ordered_data, idx_t size);
    static LogicalType USER(const string &user_type_name);

private:
    LogicalTypeId id_;
    PhysicalType physical_type_;
    shared_ptr<ExtraTypeInfo> type_info_;

    // 根据 LogicalTypeId 确定 PhysicalType
    PhysicalType GetInternalType();
};
```

### 核心成员变量

1. **id_**：逻辑类型标识符，表示 SQL 语义类型
2. **physical_type_**：物理类型，表示实际存储格式
3. **type_info_**：指向 `ExtraTypeInfo` 的智能指针，存储类型的额外元信息

### 逻辑类型到物理类型的映射

`GetInternalType()` 方法实现了从逻辑类型到物理类型的映射规则：

```cpp
PhysicalType LogicalType::GetInternalType() {
    switch (id_) {
    case LogicalTypeId::BOOLEAN:
        return PhysicalType::BOOL;

    // 整数类型直接映射
    case LogicalTypeId::TINYINT:
        return PhysicalType::INT8;
    case LogicalTypeId::SMALLINT:
        return PhysicalType::INT16;
    case LogicalTypeId::INTEGER:
        return PhysicalType::INT32;
    case LogicalTypeId::BIGINT:
        return PhysicalType::INT64;

    // 日期时间类型映射到整数
    case LogicalTypeId::DATE:
        return PhysicalType::INT32;  // 从 epoch 开始的天数
    case LogicalTypeId::TIME:
    case LogicalTypeId::TIMESTAMP:
    case LogicalTypeId::TIMESTAMP_TZ:
        return PhysicalType::INT64;  // 微秒精度

    // DECIMAL 根据精度选择物理类型
    case LogicalTypeId::DECIMAL: {
        auto width = DecimalType::GetWidth(*this);
        if (width <= Decimal::MAX_WIDTH_INT16) {
            return PhysicalType::INT16;
        } else if (width <= Decimal::MAX_WIDTH_INT32) {
            return PhysicalType::INT32;
        } else if (width <= Decimal::MAX_WIDTH_INT64) {
            return PhysicalType::INT64;
        } else {
            return PhysicalType::INT128;
        }
    }

    // 嵌套类型
    case LogicalTypeId::STRUCT:
    case LogicalTypeId::UNION:
    case LogicalTypeId::VARIANT:
        return PhysicalType::STRUCT;
    case LogicalTypeId::LIST:
    case LogicalTypeId::MAP:
        return PhysicalType::LIST;
    case LogicalTypeId::ARRAY:
        return PhysicalType::ARRAY;

    // ENUM 根据字典大小选择物理类型
    case LogicalTypeId::ENUM:
        return EnumType::GetPhysicalType(*this);

    // ...
    }
}
```

这个映射有几个关键设计点：

1. **日期时间使用整数存储**：`DATE` 用 `INT32` 存储天数，时间戳用 `INT64` 存储微秒数
2. **DECIMAL 自适应存储**：根据精度自动选择最小的能容纳的整数类型
3. **嵌套类型共享物理类型**：`STRUCT`、`UNION`、`VARIANT` 都使用 `PhysicalType::STRUCT`
4. **ENUM 动态选择**：根据枚举值数量选择 `UINT8`、`UINT16` 或 `UINT32`

## ExtraTypeInfo 扩展机制

对于需要额外元数据的类型（如 `DECIMAL` 的精度、`LIST` 的子类型），DuckDB 通过 `ExtraTypeInfo` 及其子类来存储：

```cpp
enum class ExtraTypeInfoType : uint8_t {
    INVALID_TYPE_INFO = 0,
    GENERIC_TYPE_INFO = 1,     // 通用信息（如别名）
    DECIMAL_TYPE_INFO = 2,     // DECIMAL 的 width/scale
    STRING_TYPE_INFO = 3,      // 字符串的 collation
    LIST_TYPE_INFO = 4,        // LIST 的子类型
    STRUCT_TYPE_INFO = 5,      // STRUCT 的字段列表
    ENUM_TYPE_INFO = 6,        // ENUM 的字典
    USER_TYPE_INFO = 7,        // 用户类型名称
    AGGREGATE_STATE_TYPE_INFO = 8,  // 聚合状态
    ARRAY_TYPE_INFO = 9,       // ARRAY 的子类型和大小
    ANY_TYPE_INFO = 10,        // ANY 类型的约束
    INTEGER_LITERAL_TYPE_INFO = 11, // 整数字面量
    TEMPLATE_TYPE_INFO = 12,   // 模板类型名
    GEO_TYPE_INFO = 13         // 地理类型
};
```

### ExtraTypeInfo 基类

```cpp
struct ExtraTypeInfo {
    ExtraTypeInfoType type;              // 类型标识
    string alias;                         // 类型别名
    unique_ptr<ExtensionTypeInfo> extension_info;  // 扩展信息

    explicit ExtraTypeInfo(ExtraTypeInfoType type);
    explicit ExtraTypeInfo(ExtraTypeInfoType type, string alias);

    // 相等性比较
    bool Equals(ExtraTypeInfo *other_p) const;

    // 序列化支持
    virtual void Serialize(Serializer &serializer) const;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);

    // 复制方法
    virtual shared_ptr<ExtraTypeInfo> Copy() const;
    virtual shared_ptr<ExtraTypeInfo> DeepCopy() const;

    // 类型安全转换
    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }

protected:
    virtual bool EqualsInternal(ExtraTypeInfo *other_p) const;
};
```

### 具体子类实现

#### DecimalTypeInfo

```cpp
struct DecimalTypeInfo : public ExtraTypeInfo {
    uint8_t width;   // 总位数（1-38）
    uint8_t scale;   // 小数位数

    DecimalTypeInfo(uint8_t width_p, uint8_t scale_p)
        : ExtraTypeInfo(ExtraTypeInfoType::DECIMAL_TYPE_INFO),
          width(width_p), scale(scale_p) {
        D_ASSERT(width_p >= scale_p);
    }

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override {
        auto &other = other_p->Cast<DecimalTypeInfo>();
        return width == other.width && scale == other.scale;
    }
};
```

#### ListTypeInfo

```cpp
struct ListTypeInfo : public ExtraTypeInfo {
    LogicalType child_type;  // 列表元素类型

    explicit ListTypeInfo(LogicalType child_type_p)
        : ExtraTypeInfo(ExtraTypeInfoType::LIST_TYPE_INFO),
          child_type(std::move(child_type_p)) {}

    // 支持深拷贝（递归复制子类型）
    shared_ptr<ExtraTypeInfo> DeepCopy() const override {
        return make_shared_ptr<ListTypeInfo>(child_type.DeepCopy());
    }

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override {
        auto &other = other_p->Cast<ListTypeInfo>();
        return child_type == other.child_type;
    }
};
```

#### StructTypeInfo

```cpp
struct StructTypeInfo : public ExtraTypeInfo {
    child_list_t<LogicalType> child_types;  // 字段名和类型列表

    explicit StructTypeInfo(child_list_t<LogicalType> child_types_p)
        : ExtraTypeInfo(ExtraTypeInfoType::STRUCT_TYPE_INFO),
          child_types(std::move(child_types_p)) {}

    shared_ptr<ExtraTypeInfo> DeepCopy() const override {
        child_list_t<LogicalType> copied_child_types;
        for (const auto &child_type : child_types) {
            copied_child_types.emplace_back(
                child_type.first,
                child_type.second.DeepCopy()
            );
        }
        return make_shared_ptr<StructTypeInfo>(std::move(copied_child_types));
    }
};
```

#### EnumTypeInfo

```cpp
struct EnumTypeInfo : public ExtraTypeInfo {
    EnumTypeInfo(Vector &values_insert_order_p, idx_t dict_size_p);

    const Vector &GetValuesInsertOrder() const;
    const idx_t &GetDictSize() const;

    // 根据枚举大小确定存储类型
    static PhysicalType DictType(idx_t size) {
        if (size <= NumericLimits<uint8_t>::Maximum()) {
            return PhysicalType::UINT8;
        } else if (size <= NumericLimits<uint16_t>::Maximum()) {
            return PhysicalType::UINT16;
        } else if (size <= NumericLimits<uint32_t>::Maximum()) {
            return PhysicalType::UINT32;
        }
        throw InternalException("Enum size too large");
    }

    // 创建 ENUM 类型
    static LogicalType CreateType(Vector &ordered_data, idx_t size);

protected:
    Vector values_insert_order;  // 枚举值向量
    EnumDictType dict_type;
    idx_t dict_size;
};
```

#### ArrayTypeInfo

```cpp
struct ArrayTypeInfo : public ExtraTypeInfo {
    LogicalType child_type;  // 元素类型
    uint32_t size;           // 固定大小

    explicit ArrayTypeInfo(LogicalType child_type_p, uint32_t size_p)
        : ExtraTypeInfo(ExtraTypeInfoType::ARRAY_TYPE_INFO),
          child_type(std::move(child_type_p)), size(size_p) {}

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override {
        auto &other = other_p->Cast<ArrayTypeInfo>();
        return child_type == other.child_type && size == other.size;
    }
};
```

## 类型辅助类

为了方便访问特定类型的元信息，DuckDB 提供了一系列静态辅助类：

### DecimalType

```cpp
struct DecimalType {
    static uint8_t GetWidth(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::DECIMAL);
        auto info = type.AuxInfo();
        return info->Cast<DecimalTypeInfo>().width;
    }

    static uint8_t GetScale(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::DECIMAL);
        auto info = type.AuxInfo();
        return info->Cast<DecimalTypeInfo>().scale;
    }

    static uint8_t MaxWidth() {
        return DecimalWidth<hugeint_t>::max;  // 38
    }
};
```

### ListType

```cpp
struct ListType {
    static const LogicalType &GetChildType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::LIST ||
                 type.id() == LogicalTypeId::MAP);
        auto info = type.AuxInfo();
        return info->Cast<ListTypeInfo>().child_type;
    }
};
```

### StructType

```cpp
struct StructType {
    static const child_list_t<LogicalType> &GetChildTypes(const LogicalType &type);
    static const LogicalType &GetChildType(const LogicalType &type, idx_t index);
    static const string &GetChildName(const LogicalType &type, idx_t index);
    static idx_t GetChildCount(const LogicalType &type);
    static bool IsUnnamed(const LogicalType &type);  // 是否为匿名结构体
};
```

### MapType

```cpp
struct MapType {
    static const LogicalType &KeyType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::MAP);
        return StructType::GetChildTypes(ListType::GetChildType(type))[0].second;
    }

    static const LogicalType &ValueType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::MAP);
        return StructType::GetChildTypes(ListType::GetChildType(type))[1].second;
    }
};
```

### UnionType

```cpp
struct UnionType {
    static const idx_t MAX_UNION_MEMBERS = 256;

    static const LogicalType &GetMemberType(const LogicalType &type, idx_t index);
    static const string &GetMemberName(const LogicalType &type, idx_t index);
    static idx_t GetMemberCount(const LogicalType &type);
};
```

### ArrayType

```cpp
struct ArrayType {
    static constexpr idx_t MAX_ARRAY_SIZE = 100000;

    static const LogicalType &GetChildType(const LogicalType &type);
    static idx_t GetSize(const LogicalType &type);
    static bool IsAnySize(const LogicalType &type);  // 大小为0表示任意大小
    static LogicalType ConvertToList(const LogicalType &type);  // 转换为 LIST
};
```

### EnumType

```cpp
struct EnumType {
    static const Vector &GetValuesInsertOrder(const LogicalType &type);
    static idx_t GetSize(const LogicalType &type);
    static PhysicalType GetPhysicalType(const LogicalType &type);
    static int64_t GetPos(const LogicalType &type, const string_t &key);
    static string_t GetString(const LogicalType &type, idx_t pos);
};
```

## 类型工厂方法

`LogicalType` 提供了丰富的静态工厂方法来创建各种类型：

### 基础类型常量

```cpp
// 预定义的基础类型常量
constexpr const LogicalTypeId LogicalType::BOOLEAN;
constexpr const LogicalTypeId LogicalType::TINYINT;
constexpr const LogicalTypeId LogicalType::SMALLINT;
constexpr const LogicalTypeId LogicalType::INTEGER;
constexpr const LogicalTypeId LogicalType::BIGINT;
constexpr const LogicalTypeId LogicalType::FLOAT;
constexpr const LogicalTypeId LogicalType::DOUBLE;
constexpr const LogicalTypeId LogicalType::VARCHAR;
constexpr const LogicalTypeId LogicalType::DATE;
constexpr const LogicalTypeId LogicalType::TIMESTAMP;
// ... 更多预定义类型
```

### 复杂类型工厂

```cpp
// DECIMAL 类型
LogicalType LogicalType::DECIMAL(uint8_t width, uint8_t scale) {
    D_ASSERT(width >= scale);
    auto type_info = make_shared_ptr<DecimalTypeInfo>(width, scale);
    return LogicalType(LogicalTypeId::DECIMAL, std::move(type_info));
}

// LIST 类型
LogicalType LogicalType::LIST(const LogicalType &child) {
    auto info = make_shared_ptr<ListTypeInfo>(child);
    return LogicalType(LogicalTypeId::LIST, std::move(info));
}

// STRUCT 类型
LogicalType LogicalType::STRUCT(child_list_t<LogicalType> children) {
    auto info = make_shared_ptr<StructTypeInfo>(std::move(children));
    return LogicalType(LogicalTypeId::STRUCT, std::move(info));
}

// MAP 类型（内部使用 LIST<STRUCT<key, value>>）
LogicalType LogicalType::MAP(LogicalType key, LogicalType value) {
    child_list_t<LogicalType> child_types;
    child_types.emplace_back("key", std::move(key));
    child_types.emplace_back("value", std::move(value));
    auto child = LogicalType::STRUCT(child_types);
    auto info = make_shared_ptr<ListTypeInfo>(child);
    return LogicalType(LogicalTypeId::MAP, std::move(info));
}

// UNION 类型（带隐藏的 tag 字段）
LogicalType LogicalType::UNION(child_list_t<LogicalType> members) {
    D_ASSERT(!members.empty());
    D_ASSERT(members.size() <= UnionType::MAX_UNION_MEMBERS);
    // 在开头插入隐藏的 tag 字段
    members.insert(members.begin(), {"", LogicalType::UTINYINT});
    auto info = make_shared_ptr<StructTypeInfo>(std::move(members));
    return LogicalType(LogicalTypeId::UNION, std::move(info));
}

// ARRAY 类型（固定长度）
LogicalType LogicalType::ARRAY(const LogicalType &child, optional_idx size) {
    if (!size.IsValid()) {
        // 不完整的 ARRAY 类型，用于绑定
        auto info = make_shared_ptr<ArrayTypeInfo>(child, 0);
        return LogicalType(LogicalTypeId::ARRAY, std::move(info));
    } else {
        auto array_size = size.GetIndex();
        D_ASSERT(array_size > 0 && array_size <= ArrayType::MAX_ARRAY_SIZE);
        auto info = make_shared_ptr<ArrayTypeInfo>(child, array_size);
        return LogicalType(LogicalTypeId::ARRAY, std::move(info));
    }
}

// ENUM 类型
LogicalType LogicalType::ENUM(Vector &ordered_data, idx_t size) {
    return EnumTypeInfo::CreateType(ordered_data, size);
}
```

## 类型比较与合并

### 类型相等性

`LogicalType` 的相等性比较同时考虑类型标识和额外信息：

```cpp
bool LogicalType::operator==(const LogicalType &rhs) const {
    if (id_ != rhs.id_) {
        return false;
    }
    return EqualTypeInfo(rhs);
}

bool LogicalType::EqualTypeInfo(const LogicalType &rhs) const {
    if (type_info_.get() == rhs.type_info_.get()) {
        return true;
    }
    if (type_info_) {
        return type_info_->Equals(rhs.type_info_.get());
    } else {
        return rhs.type_info_->Equals(type_info_.get());
    }
}
```

### 类型合并

当需要确定两个类型的公共超类型时（如 `UNION ALL` 或 `CASE WHEN`），使用 `MaxLogicalType`：

```cpp
LogicalType LogicalType::MaxLogicalType(ClientContext &context,
                                         const LogicalType &left,
                                         const LogicalType &right) {
    LogicalType result;
    if (!TryGetMaxLogicalType(context, left, right, result)) {
        throw NotImplementedException(
            "Cannot combine types %s and %s - an explicit cast is required",
            left.ToString(), right.ToString());
    }
    return result;
}
```

合并规则包括：

1. **NULL 类型**：`NULL` 与任何类型合并取另一方
2. **数值类型**：选择能容纳两者的最小类型（如 `INT32` + `INT64` → `INT64`）
3. **DECIMAL**：合并精度和小数位数
4. **嵌套类型**：递归合并子类型
5. **不兼容类型**：抛出异常或使用 `VARCHAR`

## 类型属性判断

`LogicalType` 提供了丰富的类型属性判断方法：

```cpp
// 是否为整数类型
bool LogicalType::IsIntegral() const {
    switch (id_) {
    case LogicalTypeId::TINYINT:
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::UTINYINT:
    case LogicalTypeId::USMALLINT:
    case LogicalTypeId::UINTEGER:
    case LogicalTypeId::UBIGINT:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::UHUGEINT:
        return true;
    default:
        return false;
    }
}

// 是否为数值类型
bool LogicalType::IsNumeric() const {
    return IsIntegral() || IsFloating() || id_ == LogicalTypeId::DECIMAL;
}

// 是否为时态类型
bool LogicalType::IsTemporal() const {
    switch (id_) {
    case LogicalTypeId::DATE:
    case LogicalTypeId::TIME:
    case LogicalTypeId::TIMESTAMP:
    case LogicalTypeId::TIME_TZ:
    case LogicalTypeId::TIMESTAMP_TZ:
    // ...
        return true;
    default:
        return false;
    }
}

// 是否为完整类型（不含 ANY、UNKNOWN 等）
bool LogicalType::IsComplete() const {
    return !TypeVisitor::Contains(*this, [](const LogicalType &type) {
        switch (type.id()) {
        case LogicalTypeId::INVALID:
        case LogicalTypeId::UNKNOWN:
        case LogicalTypeId::ANY:
        case LogicalTypeId::TEMPLATE:
            return true;
        // 检查嵌套类型是否有正确的 type_info
        case LogicalTypeId::LIST:
        case LogicalTypeId::MAP:
            if (!type.AuxInfo() ||
                type.AuxInfo()->type != ExtraTypeInfoType::LIST_TYPE_INFO) {
                return true;
            }
            break;
        // ...
        }
        return false;
    });
}
```

## 类型集合

为了方便函数签名匹配，`LogicalType` 提供了类型集合：

```cpp
// 所有数值类型
const vector<LogicalType> LogicalType::Numeric() {
    return {TINYINT, SMALLINT, INTEGER, BIGINT, HUGEINT,
            FLOAT, DOUBLE, LogicalTypeId::DECIMAL,
            UTINYINT, USMALLINT, UINTEGER, UBIGINT, UHUGEINT};
}

// 所有整数类型
const vector<LogicalType> LogicalType::Integral() {
    return {TINYINT, SMALLINT, INTEGER, BIGINT, HUGEINT,
            UTINYINT, USMALLINT, UINTEGER, UBIGINT, UHUGEINT};
}

// 浮点类型
const vector<LogicalType> LogicalType::Real() {
    return {FLOAT, DOUBLE};
}

// 所有类型
const vector<LogicalType> LogicalType::AllTypes() {
    return {BOOLEAN, TINYINT, SMALLINT, INTEGER, BIGINT, DATE,
            TIMESTAMP, DOUBLE, FLOAT, VARCHAR, BLOB, BIT,
            INTERVAL, HUGEINT, LogicalTypeId::DECIMAL,
            // ... 更多类型
    };
}
```

## 小结

本章介绍了 DuckDB 类型系统的核心架构设计：

1. **双层类型设计**：`LogicalTypeId`（SQL 语义）与 `PhysicalType`（物理存储）分离
2. **LogicalType 类**：封装类型标识、物理类型和扩展信息
3. **ExtraTypeInfo 机制**：通过继承体系为复杂类型存储元数据
4. **类型辅助类**：`DecimalType`、`ListType`、`StructType` 等便捷访问接口
5. **类型工厂方法**：提供统一的复杂类型创建入口
6. **类型比较与合并**：支持类型相等性判断和公共超类型计算

这种设计使得 DuckDB 能够：
- 保持 SQL 标准的类型语义
- 实现高效的向量化执行
- 支持灵活的类型扩展
- 确保类型安全的序列化与反序列化

下一章将深入分析标量类型的具体实现，包括整数、浮点数、DECIMAL 以及字符串类型的内部结构。

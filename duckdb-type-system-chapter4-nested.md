# DuckDB 类型系统深度解析 - 第四章：嵌套类型系统

## 引言

嵌套类型是现代分析型数据库的重要特性，允许在单个列中存储复杂的数据结构。DuckDB 提供了完整的嵌套类型支持，包括 LIST（列表）、STRUCT（结构体）、MAP（映射）、UNION（联合）和 ARRAY（固定长度数组）。本章将深入分析这些嵌套类型的实现机制。

## 嵌套类型概览

DuckDB 的嵌套类型家族：

| SQL 类型 | LogicalTypeId | PhysicalType | 描述 |
|---------|--------------|--------------|------|
| LIST | LIST | LIST | 可变长度的同质列表 |
| STRUCT | STRUCT | STRUCT | 命名字段的异质集合 |
| MAP | MAP | LIST | 键值对集合 |
| UNION | UNION | STRUCT | 带标签的类型联合 |
| ARRAY | ARRAY | ARRAY | 固定长度的同质数组 |

## LIST 类型

LIST 是最基础的嵌套类型，表示可变长度的同质元素序列。

### list_entry_t 结构

LIST 数据使用 `list_entry_t` 结构存储每个列表的边界信息：

```cpp
// types.hpp
struct list_entry_t {
    uint64_t offset;  // 在子向量中的起始位置
    uint64_t length;  // 列表中的元素数量

    list_entry_t() = default;
    list_entry_t(uint64_t offset, uint64_t length)
        : offset(offset), length(length) {}

    inline constexpr bool operator==(const list_entry_t &other) const {
        return offset == other.offset && length == other.length;
    }
};
```

### ListTypeInfo 类型信息

LIST 类型需要存储子元素类型：

```cpp
// extra_type_info.hpp
struct ListTypeInfo : public ExtraTypeInfo {
    LogicalType child_type;  // 子元素类型

    explicit ListTypeInfo(LogicalType child_type_p)
        : ExtraTypeInfo(ExtraTypeInfoType::LIST_TYPE_INFO),
          child_type(std::move(child_type_p)) {}

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

### LIST 类型创建

```cpp
// types.cpp
LogicalType LogicalType::LIST(const LogicalType &child) {
    auto info = make_shared_ptr<ListTypeInfo>(child);
    return LogicalType(LogicalTypeId::LIST, std::move(info));
}
```

### ListType 辅助类

```cpp
struct ListType {
    static const LogicalType &GetChildType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::LIST ||
                 type.id() == LogicalTypeId::MAP);
        auto info = type.AuxInfo();
        D_ASSERT(info);
        return info->Cast<ListTypeInfo>().child_type;
    }
};
```

### LIST 存储模型

```
LIST<INTEGER> 示例：[[1,2,3], [4,5], [6]]

主向量（list_entry_t 数组）：
┌──────────────────────────────────────────┐
│ Row 0: offset=0, length=3  → [1,2,3]     │
│ Row 1: offset=3, length=2  → [4,5]       │
│ Row 2: offset=5, length=1  → [6]         │
└──────────────────────────────────────────┘

子向量（INTEGER 数组）：
┌──────────────────────────────────────────┐
│ [0]=1, [1]=2, [2]=3, [3]=4, [4]=5, [5]=6 │
└──────────────────────────────────────────┘
```

## STRUCT 类型

STRUCT 是一个包含命名字段的复合类型，每个字段可以有不同的类型。

### StructTypeInfo 类型信息

```cpp
// extra_type_info.hpp
struct StructTypeInfo : public ExtraTypeInfo {
    child_list_t<LogicalType> child_types;  // 字段名和类型的列表

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

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override {
        auto &other = other_p->Cast<StructTypeInfo>();
        return child_types == other.child_types;
    }
};
```

### child_list_t 类型定义

```cpp
// types.hpp
template <class T>
using child_list_t = vector<pair<string, T>>;
```

### STRUCT 类型创建

```cpp
LogicalType LogicalType::STRUCT(child_list_t<LogicalType> children) {
    auto info = make_shared_ptr<StructTypeInfo>(std::move(children));
    return LogicalType(LogicalTypeId::STRUCT, std::move(info));
}
```

### StructType 辅助类

```cpp
struct StructType {
    // 获取所有子类型
    static const child_list_t<LogicalType> &GetChildTypes(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::STRUCT ||
                 type.id() == LogicalTypeId::UNION);
        auto info = type.AuxInfo();
        D_ASSERT(info);
        return info->Cast<StructTypeInfo>().child_types;
    }

    // 获取指定索引的子类型
    static const LogicalType &GetChildType(const LogicalType &type, idx_t index) {
        return GetChildTypes(type)[index].second;
    }

    // 获取指定索引的字段名
    static const string &GetChildName(const LogicalType &type, idx_t index) {
        return GetChildTypes(type)[index].first;
    }

    // 获取子字段数量
    static idx_t GetChildCount(const LogicalType &type) {
        return GetChildTypes(type).size();
    }

    // 检查是否为匿名结构体（所有字段名为空）
    static bool IsUnnamed(const LogicalType &type) {
        auto &child_types = GetChildTypes(type);
        for (auto &child_type : child_types) {
            if (!child_type.first.empty()) {
                return false;
            }
        }
        return true;
    }
};
```

### STRUCT 存储模型

```
STRUCT(name VARCHAR, age INTEGER) 示例：
[{name: 'Alice', age: 30}, {name: 'Bob', age: 25}]

STRUCT 向量本身没有数据，由子向量组成：

子向量 0 (name - VARCHAR):
┌──────────────────────────┐
│ Row 0: 'Alice'           │
│ Row 1: 'Bob'             │
└──────────────────────────┘

子向量 1 (age - INTEGER):
┌──────────────────────────┐
│ Row 0: 30                │
│ Row 1: 25                │
└──────────────────────────┘
```

## MAP 类型

MAP 是键值对的集合，内部表示为 `LIST<STRUCT<key, value>>`。

### MAP 类型创建

```cpp
LogicalType LogicalType::MAP(LogicalType key, LogicalType value) {
    // MAP 内部是 LIST<STRUCT<key, value>>
    child_list_t<LogicalType> child_types;
    child_types.emplace_back("key", std::move(key));
    child_types.emplace_back("value", std::move(value));

    auto child = LogicalType::STRUCT(std::move(child_types));
    auto info = make_shared_ptr<ListTypeInfo>(std::move(child));
    return LogicalType(LogicalTypeId::MAP, std::move(info));
}
```

### MapType 辅助类

```cpp
struct MapType {
    static const LogicalType &KeyType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::MAP);
        // MAP 的 child 是 STRUCT<key, value>
        return StructType::GetChildTypes(ListType::GetChildType(type))[0].second;
    }

    static const LogicalType &ValueType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::MAP);
        return StructType::GetChildTypes(ListType::GetChildType(type))[1].second;
    }
};
```

### MAP 存储模型

```
MAP(VARCHAR, INTEGER) 示例：
{'a': 1, 'b': 2}

物理上是 LIST<STRUCT<key VARCHAR, value INTEGER>>：

主向量 (list_entry_t):
┌──────────────────────────────────────────┐
│ Row 0: offset=0, length=2                │
└──────────────────────────────────────────┘

子向量 (STRUCT<key, value>):
  └── key 向量 (VARCHAR):
      ┌──────────────────────┐
      │ [0]='a', [1]='b'     │
      └──────────────────────┘
  └── value 向量 (INTEGER):
      ┌──────────────────────┐
      │ [0]=1, [1]=2         │
      └──────────────────────┘
```

## UNION 类型

UNION 是带标签的类型联合，每个值是多个可能类型之一。

### UNION 类型创建

```cpp
LogicalType LogicalType::UNION(child_list_t<LogicalType> members) {
    D_ASSERT(!members.empty());
    D_ASSERT(members.size() <= UnionType::MAX_UNION_MEMBERS);

    // 在开头插入隐藏的 tag 字段
    // tag 用于标识当前值是哪个成员类型
    members.insert(members.begin(), {"", LogicalType::UTINYINT});

    auto info = make_shared_ptr<StructTypeInfo>(std::move(members));
    return LogicalType(LogicalTypeId::UNION, std::move(info));
}
```

### union_tag_t 类型

```cpp
// types.hpp
using union_tag_t = uint8_t;
```

### UnionType 辅助类

```cpp
struct UnionType {
    static const idx_t MAX_UNION_MEMBERS = 256;  // 最多 256 个成员

    // 获取成员类型（跳过隐藏的 tag 字段）
    static const LogicalType &GetMemberType(const LogicalType &type, idx_t index) {
        D_ASSERT(type.id() == LogicalTypeId::UNION);
        // index + 1 跳过 tag 字段
        return StructType::GetChildType(type, index + 1);
    }

    // 获取成员名称
    static const string &GetMemberName(const LogicalType &type, idx_t index) {
        D_ASSERT(type.id() == LogicalTypeId::UNION);
        return StructType::GetChildName(type, index + 1);
    }

    // 获取成员数量（不包括 tag）
    static idx_t GetMemberCount(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::UNION);
        return StructType::GetChildCount(type) - 1;
    }
};
```

### UNION 存储模型

```
UNION(str VARCHAR, num INTEGER) 示例：
[union_value('hello'), union_value(42)]

物理上是 STRUCT<tag UTINYINT, str VARCHAR, num INTEGER>：

tag 向量 (UTINYINT):
┌──────────────────────┐
│ [0]=0 (str 类型)      │
│ [1]=1 (num 类型)      │
└──────────────────────┘

str 向量 (VARCHAR):
┌──────────────────────┐
│ [0]='hello'          │
│ [1]=NULL (无效)       │
└──────────────────────┘

num 向量 (INTEGER):
┌──────────────────────┐
│ [0]=NULL (无效)       │
│ [1]=42               │
└──────────────────────┘
```

## ARRAY 类型

ARRAY 是固定长度的同质数组，与 LIST 不同，所有行的数组大小必须相同。

### ArrayTypeInfo 类型信息

```cpp
// extra_type_info.hpp
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

### ARRAY 类型创建

```cpp
LogicalType LogicalType::ARRAY(const LogicalType &child, optional_idx size) {
    if (!size.IsValid()) {
        // 不完整的 ARRAY 类型，用于绑定阶段
        auto info = make_shared_ptr<ArrayTypeInfo>(child, 0);
        return LogicalType(LogicalTypeId::ARRAY, std::move(info));
    } else {
        auto array_size = size.GetIndex();
        D_ASSERT(array_size > 0 && array_size <= ArrayType::MAX_ARRAY_SIZE);
        auto info = make_shared_ptr<ArrayTypeInfo>(child, array_size);
        return LogicalType(LogicalTypeId::ARRAY, std::move(info));
    }
}
```

### ArrayType 辅助类

```cpp
struct ArrayType {
    static constexpr idx_t MAX_ARRAY_SIZE = 100000;  // 最大数组大小

    // 获取子类型
    static const LogicalType &GetChildType(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::ARRAY);
        auto info = type.AuxInfo();
        D_ASSERT(info);
        return info->Cast<ArrayTypeInfo>().child_type;
    }

    // 获取数组大小
    static idx_t GetSize(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::ARRAY);
        auto info = type.AuxInfo();
        D_ASSERT(info);
        return info->Cast<ArrayTypeInfo>().size;
    }

    // 检查是否为任意大小（大小为 0）
    static bool IsAnySize(const LogicalType &type) {
        return GetSize(type) == 0;
    }

    // 转换为 LIST 类型
    static LogicalType ConvertToList(const LogicalType &type) {
        D_ASSERT(type.id() == LogicalTypeId::ARRAY);
        return LogicalType::LIST(GetChildType(type));
    }
};
```

### ARRAY 存储模型

```
ARRAY(INTEGER, 3) 示例：[[1,2,3], [4,5,6]]

ARRAY 向量本身没有数据，只有子向量：

子向量 (INTEGER):
┌────────────────────────────────────────────┐
│ [0]=1, [1]=2, [2]=3, [3]=4, [4]=5, [5]=6   │
└────────────────────────────────────────────┘

Row 0 对应子向量 [0:3]
Row 1 对应子向量 [3:6]
```

## ENUM 类型

ENUM 是枚举类型，将字符串值映射到整数索引。

### EnumTypeInfo 类型信息

```cpp
// extra_type_info.hpp
enum EnumDictType : uint8_t {
    INVALID = 0,
    VECTOR_DICT = 1  // 使用向量存储字典
};

struct EnumTypeInfo : public ExtraTypeInfo {
    explicit EnumTypeInfo(Vector &values_insert_order_p, idx_t dict_size_p);

    const EnumDictType &GetEnumDictType() const;
    const Vector &GetValuesInsertOrder() const;
    const idx_t &GetDictSize() const;

    // 根据枚举大小确定存储类型
    static PhysicalType DictType(idx_t size) {
        if (size <= NumericLimits<uint8_t>::Maximum()) {
            return PhysicalType::UINT8;   // 0-255 个枚举值
        } else if (size <= NumericLimits<uint16_t>::Maximum()) {
            return PhysicalType::UINT16;  // 256-65535 个枚举值
        } else if (size <= NumericLimits<uint32_t>::Maximum()) {
            return PhysicalType::UINT32;  // 更多枚举值
        }
        throw InternalException("Enum size too large");
    }

    static LogicalType CreateType(Vector &ordered_data, idx_t size);

protected:
    Vector values_insert_order;  // 枚举值向量
    EnumDictType dict_type;
    idx_t dict_size;
};
```

### EnumType 辅助类

```cpp
struct EnumType {
    // 获取枚举值向量
    static const Vector &GetValuesInsertOrder(const LogicalType &type);

    // 获取枚举大小
    static idx_t GetSize(const LogicalType &type);

    // 获取物理存储类型
    static PhysicalType GetPhysicalType(const LogicalType &type);

    // 根据键查找位置
    static int64_t GetPos(const LogicalType &type, const string_t &key);

    // 根据位置获取字符串
    static string_t GetString(const LogicalType &type, idx_t pos);
};
```

### ENUM 物理类型选择

```cpp
// types.cpp
case LogicalTypeId::ENUM: {
    D_ASSERT(type_info_);
    return EnumType::GetPhysicalType(*this);
}
```

## 嵌套类型与物理类型映射

```cpp
PhysicalType LogicalType::GetInternalType() {
    switch (id_) {
    // STRUCT 和 UNION 共享物理类型
    case LogicalTypeId::STRUCT:
    case LogicalTypeId::UNION:
    case LogicalTypeId::VARIANT:
        return PhysicalType::STRUCT;

    // LIST 和 MAP 共享物理类型
    case LogicalTypeId::LIST:
    case LogicalTypeId::MAP:
        return PhysicalType::LIST;

    // ARRAY 有自己的物理类型
    case LogicalTypeId::ARRAY:
        return PhysicalType::ARRAY;

    // ENUM 根据大小选择
    case LogicalTypeId::ENUM:
        return EnumType::GetPhysicalType(*this);
    // ...
    }
}
```

## 类型大小计算

```cpp
idx_t GetTypeIdSize(PhysicalType type) {
    switch (type) {
    case PhysicalType::LIST:
        return sizeof(list_entry_t);  // 16 bytes (offset + length)

    case PhysicalType::STRUCT:
    case PhysicalType::ARRAY:
        return 0;  // 没有自己的数据，由子类型决定

    // ...
    }
}
```

## 嵌套类型深拷贝

嵌套类型的拷贝需要递归处理子类型：

```cpp
// ExtraTypeInfo::DeepCopy 示例
shared_ptr<ExtraTypeInfo> ListTypeInfo::DeepCopy() const {
    return make_shared_ptr<ListTypeInfo>(child_type.DeepCopy());
}

shared_ptr<ExtraTypeInfo> StructTypeInfo::DeepCopy() const {
    child_list_t<LogicalType> copied_child_types;
    for (const auto &child_type : child_types) {
        copied_child_types.emplace_back(
            child_type.first,              // 字段名
            child_type.second.DeepCopy()   // 递归深拷贝类型
        );
    }
    return make_shared_ptr<StructTypeInfo>(std::move(copied_child_types));
}

shared_ptr<ExtraTypeInfo> ArrayTypeInfo::DeepCopy() const {
    return make_shared_ptr<ArrayTypeInfo>(child_type.DeepCopy(), size);
}
```

## 类型相等性比较

嵌套类型的相等性需要递归比较：

```cpp
bool ListTypeInfo::EqualsInternal(ExtraTypeInfo *other_p) const {
    auto &other = other_p->Cast<ListTypeInfo>();
    return child_type == other.child_type;
}

bool StructTypeInfo::EqualsInternal(ExtraTypeInfo *other_p) const {
    auto &other = other_p->Cast<StructTypeInfo>();
    return child_types == other.child_types;  // 比较字段名和类型
}

bool ArrayTypeInfo::EqualsInternal(ExtraTypeInfo *other_p) const {
    auto &other = other_p->Cast<ArrayTypeInfo>();
    return child_type == other.child_type && size == other.size;
}
```

## 嵌套类型序列化

嵌套类型需要递归序列化子类型信息：

```cpp
void ListTypeInfo::Serialize(Serializer &serializer) const {
    ExtraTypeInfo::Serialize(serializer);
    child_type.Serialize(serializer);
}

shared_ptr<ExtraTypeInfo> ListTypeInfo::Deserialize(Deserializer &source) {
    auto child_type = LogicalType::Deserialize(source);
    return make_shared_ptr<ListTypeInfo>(std::move(child_type));
}

void StructTypeInfo::Serialize(Serializer &serializer) const {
    ExtraTypeInfo::Serialize(serializer);
    serializer.WriteProperty(100, "child_types", child_types);
}

void ArrayTypeInfo::Serialize(Serializer &serializer) const {
    ExtraTypeInfo::Serialize(serializer);
    child_type.Serialize(serializer);
    serializer.WriteProperty(100, "size", size);
}
```

## 类型完整性检查

嵌套类型需要检查所有子类型是否完整：

```cpp
bool LogicalType::IsComplete() const {
    return !TypeVisitor::Contains(*this, [](const LogicalType &type) {
        switch (type.id()) {
        // 不完整的类型标识
        case LogicalTypeId::INVALID:
        case LogicalTypeId::UNKNOWN:
        case LogicalTypeId::ANY:
        case LogicalTypeId::TEMPLATE:
            return true;

        // 检查 LIST/MAP 是否有正确的 type_info
        case LogicalTypeId::LIST:
        case LogicalTypeId::MAP:
            if (!type.AuxInfo() ||
                type.AuxInfo()->type != ExtraTypeInfoType::LIST_TYPE_INFO) {
                return true;
            }
            break;

        // 检查 STRUCT/UNION 是否有正确的 type_info
        case LogicalTypeId::STRUCT:
        case LogicalTypeId::UNION:
            if (!type.AuxInfo() ||
                type.AuxInfo()->type != ExtraTypeInfoType::STRUCT_TYPE_INFO) {
                return true;
            }
            break;

        // 检查 ARRAY 是否有正确的 type_info
        case LogicalTypeId::ARRAY:
            if (!type.AuxInfo() ||
                type.AuxInfo()->type != ExtraTypeInfoType::ARRAY_TYPE_INFO) {
                return true;
            }
            break;
        // ...
        }
        return false;
    });
}
```

## 嵌套类型的 SQL 语法

```sql
-- LIST
CREATE TABLE t1 (col1 INTEGER[]);
INSERT INTO t1 VALUES ([1, 2, 3]), ([4, 5]);
SELECT col1[1] FROM t1;  -- 索引访问

-- STRUCT
CREATE TABLE t2 (col2 STRUCT(name VARCHAR, age INTEGER));
INSERT INTO t2 VALUES ({'name': 'Alice', 'age': 30});
SELECT col2.name FROM t2;  -- 字段访问

-- MAP
CREATE TABLE t3 (col3 MAP(VARCHAR, INTEGER));
INSERT INTO t3 VALUES (MAP {'a': 1, 'b': 2});
SELECT col3['a'] FROM t3;  -- 键访问

-- UNION
CREATE TABLE t4 (col4 UNION(str VARCHAR, num INTEGER));
INSERT INTO t4 VALUES (union_value('hello'::VARCHAR));
SELECT union_tag(col4), union_extract(col4, 'str') FROM t4;

-- ARRAY (固定长度)
CREATE TABLE t5 (col5 INTEGER[3]);
INSERT INTO t5 VALUES ([1, 2, 3]);
```

## 小结

本章详细分析了 DuckDB 的嵌套类型系统：

1. **LIST 类型**：使用 `list_entry_t`（offset + length）引用子向量中的元素
2. **STRUCT 类型**：命名字段集合，由多个子向量组成
3. **MAP 类型**：内部表示为 `LIST<STRUCT<key, value>>`
4. **UNION 类型**：带标签的类型联合，使用隐藏的 tag 字段标识当前类型
5. **ARRAY 类型**：固定长度数组，通过行号和大小计算子向量偏移
6. **ENUM 类型**：字符串到整数的映射，根据大小选择物理类型

关键设计特点：

- **类型信息分离**：通过 `ExtraTypeInfo` 子类存储嵌套类型的元信息
- **复用物理类型**：MAP 复用 LIST，UNION 复用 STRUCT
- **递归操作**：深拷贝、相等性比较、序列化都需要递归处理
- **完整性验证**：类型系统确保所有嵌套层级都有正确的类型信息

下一章将探讨向量化存储模型，了解这些类型如何在 Vector 中高效存储和访问。

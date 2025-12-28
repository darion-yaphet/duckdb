# 第十章：特殊类型与扩展

## 10.1 特殊类型概述

DuckDB的类型系统除了标准SQL类型外，还包含一系列特殊类型，用于支持函数重载、类型推断、聚合状态存储等高级功能。这些特殊类型在编译期或执行期有特殊语义，但通常不直接暴露给最终用户。

```
特殊类型分类：
┌─────────────────────────────────────────────────────────────────┐
│  内部类型（Internal Types）                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ ANY          - 通配符类型，匹配任意类型                        │ │
│  │ UNKNOWN      - 未解析的参数类型                               │ │
│  │ SQLNULL      - SQL NULL字面量                                │ │
│  │ POINTER      - 内部指针类型                                   │ │
│  │ INVALID      - 无效类型标记                                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  函数类型（Function Types）                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ TEMPLATE         - 模板类型参数                               │ │
│  │ LAMBDA           - Lambda函数类型                             │ │
│  │ TABLE            - 表值函数返回类型                            │ │
│  │ AGGREGATE_STATE  - 聚合函数中间状态                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  字面量类型（Literal Types）                                      │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ INTEGER_LITERAL  - 整数字面量（未确定具体类型）                 │ │
│  │ STRING_LITERAL   - 字符串字面量（未确定具体类型）               │ │
│  └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  扩展类型（Extension Types）                                      │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ VARIANT     - 动态类型容器                                    │ │
│  │ GEO         - 地理空间类型（扩展提供）                         │ │
│  │ User Types  - 用户自定义类型                                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## 10.2 ANY类型

### 10.2.1 ANY类型定义

ANY类型是一个特殊的通配符类型，用于函数定义时表示可以接受任意类型的参数。

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct AnyTypeInfo : public ExtraTypeInfo {
    AnyTypeInfo(LogicalType target_type, idx_t cast_score);

    LogicalType target_type;  // 目标类型约束（可选）
    idx_t cast_score;         // 隐式转换代价

public:
    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;
    shared_ptr<ExtraTypeInfo> DeepCopy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};
```

### 10.2.2 ANY类型的使用

```cpp
// src/include/duckdb/common/types.hpp

struct AnyType {
    // 获取ANY类型的目标约束类型
    DUCKDB_API static LogicalType GetTargetType(const LogicalType &type);

    // 获取转换代价分数
    DUCKDB_API static idx_t GetCastScore(const LogicalType &type);
};

// 使用示例：定义接受任意类型的函数
ScalarFunction any_func("my_func",
    {LogicalType::ANY},     // 参数：任意类型
    LogicalType::VARCHAR,   // 返回：VARCHAR
    MyFuncImplementation);
```

### 10.2.3 ANY类型在函数绑定中的作用

```cpp
// src/function/function_binder.cpp

// ANY类型匹配任意输入类型
if (expected_type.id() == LogicalTypeId::ANY) {
    auto target = AnyType::GetTargetType(expected_type);
    if (target.id() != LogicalTypeId::INVALID) {
        // 有目标约束：需要可转换到目标类型
        auto cost = CastRules::ImplicitCast(actual_type, target);
        if (cost >= 0) {
            return cost;
        }
    }
    // 无约束：接受任意类型
    return AnyType::GetCastScore(expected_type);
}
```

## 10.3 TEMPLATE类型

### 10.3.1 模板类型定义

TEMPLATE类型用于定义多态函数，同一模板名称在函数内必须解析为相同的具体类型。

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct TemplateTypeInfo : public ExtraTypeInfo {
    explicit TemplateTypeInfo(string name_p);

    // 模板名称（如 "T", "KEY_TYPE"）
    // 用于区分同一函数内的不同模板参数
    // 绑定器将同名模板解析为相同的具体类型
    string name;

public:
    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};
```

### 10.3.2 模板类型使用示例

```cpp
// 定义模板函数：list_filter(LIST<T>, LAMBDA) -> LIST<T>
// T必须在输入和输出中解析为相同类型

auto template_t = LogicalType::TEMPLATE("T");

ScalarFunction list_filter("list_filter",
    {LogicalType::LIST(template_t), LogicalType::LAMBDA},
    LogicalType::LIST(template_t),
    ListFilterFunction);
```

### 10.3.3 模板名称访问

```cpp
// src/include/duckdb/common/types.hpp

struct TemplateType {
    // 获取模板类型的名称
    DUCKDB_API static const string &GetName(const LogicalType &type);
};

// 使用示例
auto &name = TemplateType::GetName(template_type);  // 返回 "T"
```

## 10.4 INTEGER_LITERAL类型

### 10.4.1 整数字面量类型

INTEGER_LITERAL类型表示尚未确定具体类型的整数字面量。根据实际值的大小，可以隐式转换为不同的整数类型。

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct IntegerLiteralTypeInfo : public ExtraTypeInfo {
    explicit IntegerLiteralTypeInfo(Value constant_value);

    Value constant_value;  // 实际的整数值

public:
    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};
```

### 10.4.2 整数字面量类型操作

```cpp
// src/include/duckdb/common/types.hpp

struct IntegerLiteral {
    // 返回此整数字面量"偏好"的类型（根据值的大小）
    DUCKDB_API static LogicalType GetType(const LogicalType &type);

    // 检查整数字面量是否适合目标数值类型
    DUCKDB_API static bool FitsInType(const LogicalType &type, const LogicalType &target);
};
```

### 10.4.3 类型推断规则

```cpp
// src/function/cast_rules.cpp

if (from.id() == LogicalTypeId::INTEGER_LITERAL) {
    // 整数字面量有一个底层类型 - 这个类型总是匹配
    if (IntegerLiteral::GetType(from).id() == to.id()) {
        return 0;  // 完美匹配，无代价
    }

    // 整数字面量可以低代价转换到任何整数类型，但仅当字面量适合时
    if (IntegerLiteral::FitsInType(from, to)) {
        // 偏好 BIGINT, INTEGER, ...
        auto target_cost = TargetTypeCost(to);
        return target_cost - 90;  // 降低代价以优先选择
    }

    // 其他情况使用字面量偏好类型的转换规则
    return CastRules::ImplicitCast(IntegerLiteral::GetType(from), to);
}
```

## 10.5 STRING_LITERAL类型

### 10.5.1 字符串字面量类型

STRING_LITERAL类型表示尚未确定具体类型的字符串字面量。它可以隐式转换为DATE、TIMESTAMP、UUID等多种类型。

```cpp
// 字符串字面量的隐式转换规则
if (from.id() == LogicalTypeId::STRING_LITERAL) {
    // 字符串字面量可以低代价转换到任何有效类型
    // 但目标类型必须完全解析
    if (!LogicalTypeIsValid(to)) {
        return -1;  // 不能转换到 LIST(ANY) 等未解析类型
    }

    if (to.id() == LogicalTypeId::VARCHAR && to.GetAlias().empty()) {
        return 1;  // 到VARCHAR的转换代价最低
    }

    return 20;  // 到其他类型的固定代价
}
```

## 10.6 AGGREGATE_STATE类型

### 10.6.1 聚合状态类型定义

AGGREGATE_STATE类型用于存储聚合函数的中间状态，支持分布式聚合场景。

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct AggregateStateTypeInfo : public ExtraTypeInfo {
    explicit AggregateStateTypeInfo(aggregate_state_t state_type_p);

    aggregate_state_t state_type;  // 聚合状态描述

public:
    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};

// 聚合状态描述
struct aggregate_state_t {
    string function_name;           // 聚合函数名
    LogicalType return_type;        // 返回类型
    vector<LogicalType> bound_argument_types;  // 绑定的参数类型
};
```

### 10.6.2 创建聚合状态类型

```cpp
// src/include/duckdb/common/types.hpp

// 创建聚合状态类型
DUCKDB_API static LogicalType AGGREGATE_STATE(aggregate_state_t state_type);

// 使用示例
aggregate_state_t state;
state.function_name = "sum";
state.return_type = LogicalType::BIGINT;
state.bound_argument_types = {LogicalType::BIGINT};

auto agg_state_type = LogicalType::AGGREGATE_STATE(state);
```

### 10.6.3 分布式聚合支持

```sql
-- 创建聚合状态
SELECT finalize(combine(states))
FROM (
    SELECT partial_agg(value) as states
    FROM partitioned_table
    GROUP BY partition_id
);
```

## 10.7 LAMBDA类型

### 10.7.1 Lambda函数类型

LAMBDA类型表示匿名函数，用于高阶函数如`list_filter`、`list_transform`等。

```cpp
// LogicalTypeId::LAMBDA = 106

// Lambda类型的物理表示
// Lambda本身不直接存储数据，而是在表达式层面处理
```

### 10.7.2 Lambda表达式示例

```sql
-- 使用lambda过滤列表
SELECT list_filter([1, 2, 3, 4, 5], x -> x > 2);
-- 结果: [3, 4, 5]

-- 使用lambda转换列表
SELECT list_transform([1, 2, 3], x -> x * 2);
-- 结果: [2, 4, 6]

-- 嵌套lambda
SELECT list_filter(list_transform([1, 2, 3, 4], x -> x * x), y -> y > 5);
-- 结果: [9, 16]
```

## 10.8 VARIANT类型

### 10.8.1 动态类型容器

VARIANT类型是一个动态类型容器，可以存储任意类型的值，类似于JSON但支持DuckDB的完整类型系统。

```cpp
// src/common/types.cpp

LogicalType LogicalType::VARIANT() {
    child_list_t<LogicalType> children;

    // 键数组：存储每行的类型标签
    children.emplace_back("keys", LogicalType::LIST(LogicalTypeId::VARCHAR));

    // 值数组：存储实际数据（每种类型一个列表）
    children.emplace_back("children",
        LogicalType::LIST(LogicalType::LIST(LogicalTypeId::BLOB)));

    auto info = make_shared_ptr<StructTypeInfo>(std::move(children));
    return LogicalType(LogicalTypeId::VARIANT, std::move(info));
}
```

### 10.8.2 VARIANT类型特性

```cpp
// VARIANT类型转换优先级最高
if (right.id() == LogicalTypeId::VARIANT) {
    result = right;
    return true;
}
if (left.id() == LogicalTypeId::VARIANT) {
    result = left;
    return true;
}

// VARIANT可以隐式从任何类型转换
static int64_t ImplicitCastVariant(const LogicalType &to) {
    return TargetTypeCost(to);  // 可以转换到任何类型
}
```

### 10.8.3 使用场景

```sql
-- VARIANT可以存储混合类型数据
CREATE TABLE mixed_data (
    id INTEGER,
    data VARIANT
);

INSERT INTO mixed_data VALUES
    (1, 42),
    (2, 'hello'),
    (3, [1, 2, 3]),
    (4, {'name': 'test'});
```

## 10.9 用户自定义类型

### 10.9.1 UserTypeInfo

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct UserTypeInfo : public ExtraTypeInfo {
    explicit UserTypeInfo(string name_p);
    UserTypeInfo(string name_p, vector<Value> modifiers_p);
    UserTypeInfo(string catalog_p, string schema_p, string name_p, vector<Value> modifiers_p);

    string catalog;              // 目录名
    string schema;               // 模式名
    string user_type_name;       // 类型名
    vector<Value> user_type_modifiers;  // 类型修饰符

public:
    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};
```

### 10.9.2 TypeCatalogEntry

```cpp
// src/catalog/catalog_entry/type_catalog_entry.cpp

class TypeCatalogEntry : public StandardEntry {
public:
    TypeCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema, CreateTypeInfo &info);

    LogicalType user_type;           // 类型定义
    bind_type_function_t bind_function;  // 可选的绑定函数

    unique_ptr<CatalogEntry> Copy(ClientContext &context) const;
    unique_ptr<CreateInfo> GetInfo() const;
    string ToSQL() const;
};
```

### 10.9.3 创建用户定义类型

```sql
-- 创建简单别名类型
CREATE TYPE email AS VARCHAR;

-- 创建ENUM类型
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

-- 创建复合类型
CREATE TYPE address AS STRUCT(
    street VARCHAR,
    city VARCHAR,
    zip INTEGER
);

-- 使用用户定义类型
CREATE TABLE users (
    id INTEGER,
    contact_email email,
    current_mood mood,
    home_address address
);
```

## 10.10 扩展类型系统

### 10.10.1 ExtensionTypeInfo

```cpp
// src/include/duckdb/common/extension_type_info.hpp

struct LogicalTypeModifier {
    explicit LogicalTypeModifier(Value value_p) : value(std::move(value_p)) {}

    string ToString() const {
        return label.empty() ? value.ToString() : label;
    }

    Value value;    // 修饰符值
    string label;   // 可选的显示标签

    void Serialize(Serializer &serializer) const;
    static LogicalTypeModifier Deserialize(Deserializer &source);
};

struct ExtensionTypeInfo {
    vector<LogicalTypeModifier> modifiers;        // 类型修饰符列表
    unordered_map<string, Value> properties;      // 类型属性

    void Serialize(Serializer &serializer) const;
    static unique_ptr<ExtensionTypeInfo> Deserialize(Deserializer &source);
    static bool Equals(optional_ptr<ExtensionTypeInfo> rhs, optional_ptr<ExtensionTypeInfo> lhs);
};
```

### 10.10.2 地理空间类型

```cpp
// src/include/duckdb/common/extra_type_info.hpp

struct GeoTypeInfo : public ExtraTypeInfo {
public:
    GeoTypeInfo();

    void Serialize(Serializer &serializer) const override;
    static shared_ptr<ExtraTypeInfo> Deserialize(Deserializer &source);
    shared_ptr<ExtraTypeInfo> Copy() const override;

protected:
    bool EqualsInternal(ExtraTypeInfo *other_p) const override;
};
```

### 10.10.3 扩展注册自定义类型

```cpp
// 扩展注册自定义类型示例
void MyExtension::Load(DatabaseInstance &db) {
    // 创建类型信息
    CreateTypeInfo type_info;
    type_info.name = "my_type";
    type_info.type = LogicalType::BLOB;  // 底层存储类型
    type_info.type.SetAlias("my_type");  // 设置别名

    // 注册到目录
    auto &catalog = Catalog::GetCatalog(db);
    catalog.CreateType(context, type_info);

    // 注册类型相关函数
    ExtensionUtil::RegisterFunction(db, MyTypeFunction());
    ExtensionUtil::RegisterCast(db, from_type, to_type, cast_func);
}
```

## 10.11 POINTER类型

### 10.11.1 内部指针类型

POINTER类型是纯内部类型，用于在向量间传递指针。

```cpp
// LogicalTypeId::POINTER = 51
// PhysicalType 对应 POINTER

// POINTER类型通常用于：
// 1. 行位置引用（row_locations向量）
// 2. 内部数据结构指针传递
// 3. 哈希表桶指针
```

### 10.11.2 使用场景

```cpp
// 创建指针向量
Vector row_locations(LogicalType::POINTER, count);
auto locations = FlatVector::GetData<data_ptr_t>(row_locations);

// 填充指针
for (idx_t i = 0; i < count; i++) {
    locations[i] = GetRowPointer(i);
}

// 使用指针进行Gather操作
tuple_data.Gather(row_locations, sel, count, result, target_sel);
```

## 10.12 TABLE类型

### 10.12.1 表值类型

TABLE类型表示表值函数的返回类型，包含一组列定义。

```cpp
// LogicalTypeId::TABLE = 107

// TABLE类型用于表值函数
// 如 generate_series、read_csv_auto 等
TableFunction generate_series(
    "generate_series",
    {LogicalType::BIGINT, LogicalType::BIGINT},
    LogicalType::TABLE,  // 返回表
    GenerateSeriesFunction
);
```

## 10.13 类型别名系统

### 10.13.1 类型别名

任何LogicalType都可以附加一个别名，用于用户自定义类型的表示。

```cpp
// 设置类型别名
LogicalType my_type = LogicalType::INTEGER;
my_type.SetAlias("my_int");

// 获取类型别名
string alias = my_type.GetAlias();  // "my_int"

// 检查是否有别名
bool has_alias = my_type.HasAlias();  // true

// 别名在隐式转换中的作用
int64_t CastRules::ImplicitCast(const LogicalType &from, const LogicalType &to) {
    if (from.GetAlias() != to.GetAlias()) {
        // 别名不同，不能隐式转换
        return -1;
    }
    // ... 继续检查其他规则
}
```

### 10.13.2 别名解析

```cpp
// 解析用户类型名到实际类型
LogicalType Catalog::GetType(ClientContext &context, const string &schema, const string &name) {
    auto &type_entry = GetEntry<TypeCatalogEntry>(context, schema, name);
    return type_entry.user_type;
}
```

## 10.14 类型验证

### 10.14.1 类型有效性检查

```cpp
// src/function/cast_rules.cpp

bool LogicalTypeIsValid(const LogicalType &type) {
    switch (type.id()) {
    case LogicalTypeId::STRUCT:
    case LogicalTypeId::UNION:
    case LogicalTypeId::VARIANT:
    case LogicalTypeId::LIST:
    case LogicalTypeId::MAP:
    case LogicalTypeId::ARRAY:
    case LogicalTypeId::DECIMAL:
        // 这些类型只有在有辅助信息时才有效
        if (!type.AuxInfo()) {
            return false;
        }
        break;
    default:
        break;
    }

    switch (type.id()) {
    case LogicalTypeId::ANY:
    case LogicalTypeId::INVALID:
    case LogicalTypeId::UNKNOWN:
        return false;

    case LogicalTypeId::STRUCT: {
        // 递归验证所有子类型
        auto child_count = StructType::GetChildCount(type);
        for (idx_t i = 0; i < child_count; i++) {
            if (!LogicalTypeIsValid(StructType::GetChildType(type, i))) {
                return false;
            }
        }
        return true;
    }

    case LogicalTypeId::LIST:
    case LogicalTypeId::MAP:
        return LogicalTypeIsValid(ListType::GetChildType(type));

    case LogicalTypeId::ARRAY:
        return LogicalTypeIsValid(ArrayType::GetChildType(type));

    default:
        return true;
    }
}
```

## 10.15 类型排序优先级

### 10.15.1 类型排序代价

```cpp
// src/common/types.cpp

static uint8_t GetTypeIdCost(LogicalTypeId type_id) {
    switch (type_id) {
    // 基本类型
    case LogicalTypeId::BOOLEAN:    return 10;
    case LogicalTypeId::TINYINT:    return 20;
    case LogicalTypeId::SMALLINT:   return 30;
    case LogicalTypeId::INTEGER:    return 40;
    case LogicalTypeId::BIGINT:     return 50;

    // 日期时间
    case LogicalTypeId::DATE:       return 60;
    case LogicalTypeId::TIMESTAMP:  return 70;

    // 字符串
    case LogicalTypeId::VARCHAR:    return 110;
    case LogicalTypeId::BLOB:       return 120;

    // 嵌套类型
    case LogicalTypeId::STRUCT:     return 125;
    case LogicalTypeId::MAP:        return 127;

    // 特殊类型
    case LogicalTypeId::VARIANT:
    case LogicalTypeId::UNION:
    case LogicalTypeId::TABLE:      return 150;

    // 函数类型
    case LogicalTypeId::LAMBDA:     return 200;

    default:                        return 100;
    }
}
```

## 10.16 小结

DuckDB的特殊类型系统提供了丰富的功能：

1. **内部类型**：ANY、UNKNOWN、POINTER等支持类型推断和内部操作
2. **模板类型**：TEMPLATE类型实现多态函数
3. **字面量类型**：INTEGER_LITERAL、STRING_LITERAL实现灵活的类型推断
4. **函数类型**：LAMBDA、TABLE支持高阶函数和表值函数
5. **状态类型**：AGGREGATE_STATE支持分布式聚合
6. **动态类型**：VARIANT支持混合类型数据
7. **用户类型**：通过目录系统支持自定义类型
8. **扩展类型**：ExtensionTypeInfo支持扩展添加新类型

这些特殊类型与标准SQL类型紧密集成，共同构成了DuckDB强大而灵活的类型系统，支持从简单查询到复杂分析的各种使用场景。

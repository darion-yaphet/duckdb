# DuckDB Catalog 系统深度解析（六）：默认条目与内置对象

## 引言

DuckDB 包含大量内置对象：数百个标量函数、聚合函数、表函数，以及系统视图和类型别名。如果在启动时将这些对象全部加载到 Catalog，会造成显著的启动延迟和内存开销。DuckDB 采用 **DefaultGenerator 懒加载机制**，只在首次访问时才创建这些内置条目，实现了按需加载的设计理念。

本章将深入分析 DefaultGenerator 的设计原理、各类默认生成器的实现细节，以及内置函数的注册机制。

## 1. DefaultGenerator 基础架构

### 1.1 设计动机

考虑 DuckDB 的内置对象规模：

- **标量函数**：200+ 个（如 `upper`, `lower`, `substring` 等）
- **聚合函数**：50+ 个（如 `sum`, `avg`, `count` 等）
- **表函数**：30+ 个（如 `read_csv`, `duckdb_tables` 等）
- **系统视图**：60+ 个（pg_catalog 和 information_schema 视图）
- **类型别名**：77 个（如 `int` → `INTEGER`, `text` → `VARCHAR` 等）

如果每次启动都创建这些对象，会带来以下问题：
1. 启动时间增加（解析、绑定每个对象）
2. 内存占用增加（每个条目都有版本链开销）
3. 扫描性能下降（CatalogSet 变大）

### 1.2 DefaultGenerator 抽象基类

```
源文件: src/include/duckdb/catalog/default/default_generator.hpp
```

```cpp
class DefaultGenerator {
public:
    explicit DefaultGenerator(Catalog &catalog);
    virtual ~DefaultGenerator();

    Catalog &catalog;
    atomic<bool> created_all_entries;   // 是否已创建所有条目

public:
    // 按名称创建默认条目，不存在则返回 nullptr
    virtual unique_ptr<CatalogEntry> CreateDefaultEntry(ClientContext &context,
                                                         const string &entry_name);
    virtual unique_ptr<CatalogEntry> CreateDefaultEntry(CatalogTransaction transaction,
                                                         const string &entry_name);
    // 获取所有默认条目名称列表
    virtual vector<string> GetDefaultEntries() = 0;
};
```

关键设计点：

1. **按需创建**：`CreateDefaultEntry` 在首次查询时被调用
2. **完整枚举**：`GetDefaultEntries` 支持 `SHOW ALL` 类操作
3. **原子标记**：`created_all_entries` 避免重复创建

### 1.3 与 CatalogSet 的集成

DefaultGenerator 通过 `CatalogSet::SetDefaultGenerator` 与 CatalogSet 关联：

```cpp
// src/catalog/catalog_set.cpp
void CatalogSet::SetDefaultGenerator(unique_ptr<DefaultGenerator> defaults_p) {
    lock_guard<mutex> lock(catalog_lock);
    defaults = std::move(defaults_p);
}
```

当查询不存在的条目时，CatalogSet 会尝试创建默认条目：

```cpp
optional_ptr<CatalogEntry> CatalogSet::CreateDefaultEntry(CatalogTransaction transaction,
                                                           const string &name,
                                                           unique_lock<mutex> &read_lock) {
    // 检查是否有默认生成器
    if (!defaults || defaults->created_all_entries) {
        return nullptr;
    }

    // 释放锁以避免死锁（CreateDefaultEntry 可能访问其他 CatalogSet）
    read_lock.unlock();

    // 尝试创建默认条目
    auto entry = defaults->CreateDefaultEntry(transaction, name);

    read_lock.lock();
    if (!entry) {
        return nullptr;
    }

    // 将条目添加到 CatalogSet
    auto result = CreateCommittedEntry(std::move(entry));
    return result;
}
```

懒加载流程图：

```
          查询 "upper" 函数
                │
                ▼
    ┌─────────────────────────┐
    │  CatalogSet::GetEntry   │
    │  查找 map["upper"]      │
    └─────────────────────────┘
                │
         不存在 │
                ▼
    ┌─────────────────────────┐
    │  CreateDefaultEntry     │
    │  检查 DefaultGenerator  │
    └─────────────────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │  DefaultFunctionGen     │
    │  查找内置宏定义         │
    │  解析并创建 MacroEntry  │
    └─────────────────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │  CreateCommittedEntry   │
    │  永久添加到 CatalogSet  │
    └─────────────────────────┘
                │
                ▼
           返回条目
```

## 2. DefaultSchemaGenerator：内置 Schema

### 2.1 内置 Schema 定义

```
源文件: src/catalog/default/default_schemas.cpp
```

DuckDB 定义了两个内置的系统 Schema：

```cpp
struct DefaultSchema {
    const char *name;
};

static const DefaultSchema internal_schemas[] = {
    {"information_schema"},   // SQL 标准信息模式
    {"pg_catalog"},           // PostgreSQL 兼容模式
    {nullptr}
};
```

### 2.2 创建默认 Schema

```cpp
unique_ptr<CatalogEntry> DefaultSchemaGenerator::CreateDefaultEntry(
    CatalogTransaction transaction, const string &entry_name) {

    if (IsDefaultSchema(entry_name)) {
        CreateSchemaInfo info;
        info.schema = StringUtil::Lower(entry_name);
        info.internal = true;  // 标记为内部对象
        return make_uniq_base<CatalogEntry, DuckSchemaEntry>(catalog, info);
    }
    return nullptr;
}
```

这些 Schema 的特点：
- `internal = true`：不会在 `SHOW SCHEMAS` 中显示（除非明确指定）
- 拥有各自的 DefaultGenerator：提供系统视图和兼容函数

### 2.3 Schema 层次结构

```
┌─────────────────────────────────────────────────────┐
│                     DuckCatalog                     │
├─────────────────────────────────────────────────────┤
│  schemas (CatalogSet)                               │
│  ├── DefaultSchemaGenerator                         │
│  │                                                  │
│  ├── "main" (用户默认 Schema)                       │
│  │   ├── tables: DefaultViewGenerator               │
│  │   ├── functions: DefaultFunctionGenerator        │
│  │   └── types: DefaultTypeGenerator                │
│  │                                                  │
│  ├── "information_schema" (懒加载)                  │
│  │   └── tables: DefaultViewGenerator               │
│  │       └── "tables", "columns", "schemata"...     │
│  │                                                  │
│  └── "pg_catalog" (懒加载)                          │
│      ├── tables: DefaultViewGenerator               │
│      │   └── "pg_class", "pg_type", "pg_proc"...    │
│      └── functions: DefaultFunctionGenerator        │
│          └── "format_type", "pg_get_viewdef"...     │
└─────────────────────────────────────────────────────┘
```

## 3. DefaultFunctionGenerator：内置宏函数

### 3.1 DefaultMacro 结构

```
源文件: src/catalog/default/default_functions.cpp
```

DuckDB 使用宏定义来实现许多内置函数：

```cpp
struct DefaultMacro {
    const char *schema;                      // 所属 Schema
    const char *name;                        // 函数名
    const char *parameters[8];               // 参数列表
    DefaultNamedParameter named_parameters[8]; // 命名参数
    const char *macro;                       // 宏表达式
};
```

### 3.2 内置宏示例

```cpp
static const DefaultMacro internal_macros[] = {
    // 系统信息函数
    {DEFAULT_SCHEMA, "current_user", {nullptr}, {{nullptr, nullptr}}, "'duckdb'"},
    {DEFAULT_SCHEMA, "current_catalog", {nullptr}, {{nullptr, nullptr}},
     "main.current_database()"},

    // PostgreSQL 兼容函数
    {"pg_catalog", "pg_typeof", {"expression", nullptr}, {{nullptr, nullptr}},
     "lower(typeof(expression))"},
    {"pg_catalog", "format_type", {"type_oid", "typemod", nullptr}, {{nullptr, nullptr}},
     "(select format_pg_type(logical_type, type_name) from duckdb_types() t "
     "where t.type_oid=type_oid)..."},

    // 列表操作函数
    {DEFAULT_SCHEMA, "list_append", {"l", "e", nullptr}, {{nullptr, nullptr}},
     "list_concat(l, list_value(e))"},
    {DEFAULT_SCHEMA, "array_reverse", {"l", nullptr}, {{nullptr, nullptr}},
     "list_reverse(l)"},

    // 权限检查（PostgreSQL 兼容，始终返回 true）
    {"pg_catalog", "has_table_privilege", {"table", "privilege", nullptr},
     {{nullptr, nullptr}}, "true"},

    // 数学函数
    {DEFAULT_SCHEMA, "round_even", {"x", "n", nullptr}, {{nullptr, nullptr}},
     "CASE ((abs(x) * power(10, n+1)) % 10) WHEN 5 "
     "THEN round(x/2, n) * 2 ELSE round(x, n) END"},

    // 聚合列表函数
    {DEFAULT_SCHEMA, "list_sum", {"l", nullptr}, {{nullptr, nullptr}},
     "list_aggr(l, 'sum')"},
    {DEFAULT_SCHEMA, "list_avg", {"l", nullptr}, {{nullptr, nullptr}},
     "list_aggr(l, 'avg')"},

    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}
};
```

### 3.3 宏解析与创建

```cpp
unique_ptr<CreateMacroInfo> DefaultFunctionGenerator::CreateInternalMacroInfo(
    array_ptr<const DefaultMacro> macros) {

    auto type = CatalogType::MACRO_ENTRY;
    auto bind_info = make_uniq<CreateMacroInfo>(type);

    for (auto &default_macro : macros) {
        // 解析宏表达式
        auto expressions = Parser::ParseExpressionList(default_macro.macro);
        D_ASSERT(expressions.size() == 1);

        auto function = make_uniq<ScalarMacroFunction>(std::move(expressions[0]));

        // 添加位置参数
        for (idx_t param_idx = 0; default_macro.parameters[param_idx] != nullptr; param_idx++) {
            function->parameters.push_back(
                make_uniq<ColumnRefExpression>(default_macro.parameters[param_idx]));
        }

        // 添加命名参数（带默认值）
        for (idx_t named_idx = 0; default_macro.named_parameters[named_idx].name != nullptr;
             named_idx++) {
            const auto &named_param = default_macro.named_parameters[named_idx];
            auto expr_list = Parser::ParseExpressionList(named_param.default_value);
            function->parameters.push_back(make_uniq<ColumnRefExpression>(named_param.name));
            function->default_parameters.insert(
                make_pair(named_param.name, std::move(expr_list[0])));
        }

        bind_info->macros.push_back(std::move(function));
    }

    bind_info->schema = macros[0].schema;
    bind_info->name = macros[0].name;
    bind_info->temporary = true;
    bind_info->internal = true;

    return bind_info;
}
```

### 3.4 函数重载支持

同名函数可以有多个重载版本：

```cpp
static unique_ptr<CreateFunctionInfo> GetDefaultFunction(const string &input_schema,
                                                          const string &input_name) {
    auto schema = StringUtil::Lower(input_schema);
    auto name = StringUtil::Lower(input_name);

    for (idx_t index = 0; internal_macros[index].name != nullptr; index++) {
        if (DefaultFunctionMatches(internal_macros[index], schema, name)) {
            // 找到函数！继续迭代找出所有重载
            idx_t overload_count;
            for (overload_count = 1; internal_macros[index + overload_count].name;
                 overload_count++) {
                if (!DefaultFunctionMatches(internal_macros[index + overload_count],
                                            schema, name)) {
                    break;
                }
            }
            // 创建包含所有重载的函数集
            return DefaultFunctionGenerator::CreateInternalMacroInfo(
                array_ptr<const DefaultMacro>(internal_macros + index, overload_count));
        }
    }
    return nullptr;
}
```

例如 `pg_get_constraintdef` 有两个重载：

```cpp
{"pg_catalog", "pg_get_constraintdef", {"constraint_oid", nullptr}, ...},
{"pg_catalog", "pg_get_constraintdef", {"constraint_oid", "pretty_bool", nullptr}, ...},
```

## 4. DefaultViewGenerator：系统视图

### 4.1 DefaultView 结构

```
源文件: src/catalog/default/default_views.cpp
```

```cpp
struct DefaultView {
    const char *schema;   // 所属 Schema
    const char *name;     // 视图名称
    const char *sql;      // 视图定义 SQL
};
```

### 4.2 系统视图定义

```cpp
static const DefaultView internal_views[] = {
    // main schema 兼容视图
    {DEFAULT_SCHEMA, "pragma_database_list",
     "SELECT database_oid AS seq, database_name AS name, path AS file "
     "FROM duckdb_databases() WHERE NOT internal ORDER BY 1"},

    {DEFAULT_SCHEMA, "sqlite_master",
     "select 'table' \"type\", table_name \"name\", ... from duckdb_tables "
     "union all select 'view' \"type\", view_name \"name\", ... from duckdb_views "
     "union all select 'index' \"type\", index_name \"name\", ... from duckdb_indexes;"},

    // DuckDB 元数据视图
    {DEFAULT_SCHEMA, "duckdb_tables",
     "SELECT * FROM duckdb_tables() WHERE NOT internal"},
    {DEFAULT_SCHEMA, "duckdb_columns",
     "SELECT * FROM duckdb_columns() WHERE NOT internal"},
    {DEFAULT_SCHEMA, "duckdb_indexes",
     "SELECT * FROM duckdb_indexes()"},

    // pg_catalog 兼容视图
    {"pg_catalog", "pg_class",
     "SELECT table_oid oid, table_name relname, schema_oid relnamespace, "
     "... FROM duckdb_tables() "
     "UNION ALL SELECT view_oid oid, view_name relname, ... FROM duckdb_views() "
     "UNION ALL SELECT sequence_oid oid, sequence_name relname, ... FROM duckdb_sequences() "
     "UNION ALL SELECT index_oid oid, index_name relname, ... FROM duckdb_indexes()"},

    {"pg_catalog", "pg_type",
     "SELECT ... format_pg_type(logical_type, type_name) typname, "
     "schema_oid typnamespace, ... FROM duckdb_types() WHERE type_oid IS NOT NULL;"},

    {"pg_catalog", "pg_proc",
     "SELECT f.function_oid oid, function_name proname, s.oid pronamespace, "
     "... FROM duckdb_functions() f LEFT JOIN duckdb_schemas() s USING (database_name, schema_name)"},

    // information_schema 标准视图
    {"information_schema", "tables",
     "SELECT database_name table_catalog, schema_name table_schema, table_name, "
     "CASE WHEN temporary THEN 'LOCAL TEMPORARY' ELSE 'BASE TABLE' END table_type, "
     "... FROM duckdb_tables() "
     "UNION ALL SELECT ... 'VIEW' table_type, ... FROM duckdb_views;"},

    {"information_schema", "columns",
     "SELECT database_name table_catalog, schema_name table_schema, table_name, "
     "column_name, column_index ordinal_position, column_default, "
     "CASE WHEN is_nullable THEN 'YES' ELSE 'NO' END is_nullable, "
     "data_type, ... FROM duckdb_columns;"},

    {"information_schema", "schemata",
     "SELECT database_name catalog_name, schema_name, 'duckdb' schema_owner, "
     "... FROM duckdb_schemas()"},

    {nullptr, nullptr, nullptr}
};
```

### 4.3 视图创建流程

```cpp
static unique_ptr<CreateViewInfo> GetDefaultView(ClientContext &context,
                                                   const string &input_schema,
                                                   const string &input_name) {
    auto schema = StringUtil::Lower(input_schema);
    auto name = StringUtil::Lower(input_name);

    for (idx_t index = 0; internal_views[index].name != nullptr; index++) {
        if (internal_views[index].schema == schema &&
            internal_views[index].name == name) {

            auto result = make_uniq<CreateViewInfo>();
            result->schema = schema;
            result->view_name = name;
            result->sql = internal_views[index].sql;
            result->temporary = true;
            result->internal = true;

            // 解析 SQL 并绑定视图
            return CreateViewInfo::FromSelect(context, std::move(result));
        }
    }
    return nullptr;
}
```

视图创建涉及完整的 SQL 解析和绑定，因此比宏函数更耗时。但由于懒加载，只有实际查询 `information_schema.tables` 时才会触发这个过程。

### 4.4 系统视图层次

```
┌─────────────────────────────────────────────────────────────────┐
│                         系统视图架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  duckdb_tables()  ◄──────┬──────► main.duckdb_tables            │
│  duckdb_columns() ◄──────┤                                      │
│  duckdb_indexes() ◄──────┤       information_schema.tables      │
│  duckdb_views()   ◄──────┤       information_schema.columns     │
│  duckdb_types()   ◄──────┤                                      │
│  duckdb_schemas() ◄──────┘       pg_catalog.pg_class            │
│                                  pg_catalog.pg_type             │
│     (表函数)                      pg_catalog.pg_attribute        │
│                                                                 │
│                                   (系统视图：SQL 封装)           │
└─────────────────────────────────────────────────────────────────┘
```

核心表函数（如 `duckdb_tables()`）直接访问 Catalog 数据，而系统视图只是对这些表函数的 SQL 封装，提供标准化的列名和过滤条件。

## 5. DefaultTypeGenerator：类型别名

### 5.1 内置类型定义

```
源文件: src/include/duckdb/catalog/default/builtin_types/types.hpp
```

DuckDB 定义了 77 个类型别名，映射到核心 LogicalTypeId：

```cpp
struct DefaultType {
    const char *name;
    LogicalTypeId type;
};

static constexpr const builtin_type_array BUILTIN_TYPES{{
    // 数值类型别名
    {"decimal", LogicalTypeId::DECIMAL},
    {"dec", LogicalTypeId::DECIMAL},
    {"numeric", LogicalTypeId::DECIMAL},

    // 整数类型别名
    {"bigint", LogicalTypeId::BIGINT},
    {"int8", LogicalTypeId::BIGINT},
    {"int64", LogicalTypeId::BIGINT},
    {"long", LogicalTypeId::BIGINT},

    {"integer", LogicalTypeId::INTEGER},
    {"int", LogicalTypeId::INTEGER},
    {"int4", LogicalTypeId::INTEGER},
    {"int32", LogicalTypeId::INTEGER},
    {"signed", LogicalTypeId::INTEGER},

    {"smallint", LogicalTypeId::SMALLINT},
    {"int2", LogicalTypeId::SMALLINT},
    {"short", LogicalTypeId::SMALLINT},

    {"tinyint", LogicalTypeId::TINYINT},
    {"int1", LogicalTypeId::TINYINT},

    // 无符号整数
    {"ubigint", LogicalTypeId::UBIGINT},
    {"uint64", LogicalTypeId::UBIGINT},
    {"uinteger", LogicalTypeId::UINTEGER},
    {"uint32", LogicalTypeId::UINTEGER},

    // 字符串类型别名
    {"varchar", LogicalTypeId::VARCHAR},
    {"string", LogicalTypeId::VARCHAR},
    {"text", LogicalTypeId::VARCHAR},
    {"char", LogicalTypeId::VARCHAR},
    {"bpchar", LogicalTypeId::VARCHAR},
    {"nvarchar", LogicalTypeId::VARCHAR},

    // 二进制类型别名
    {"blob", LogicalTypeId::BLOB},
    {"bytea", LogicalTypeId::BLOB},
    {"varbinary", LogicalTypeId::BLOB},
    {"binary", LogicalTypeId::BLOB},

    // 时间类型别名
    {"timestamp", LogicalTypeId::TIMESTAMP},
    {"datetime", LogicalTypeId::TIMESTAMP},
    {"timestamp_us", LogicalTypeId::TIMESTAMP},
    {"timestamptz", LogicalTypeId::TIMESTAMP_TZ},

    // 布尔类型别名
    {"boolean", LogicalTypeId::BOOLEAN},
    {"bool", LogicalTypeId::BOOLEAN},
    {"logical", LogicalTypeId::BOOLEAN},

    // 浮点类型别名
    {"float", LogicalTypeId::FLOAT},
    {"real", LogicalTypeId::FLOAT},
    {"float4", LogicalTypeId::FLOAT},
    {"double", LogicalTypeId::DOUBLE},
    {"float8", LogicalTypeId::DOUBLE},

    // 其他类型
    {"uuid", LogicalTypeId::UUID},
    {"guid", LogicalTypeId::UUID},
    {"geometry", LogicalTypeId::GEOMETRY},
    {"variant", LogicalTypeId::VARIANT},
    {"varint", LogicalTypeId::BIGNUM},
    // ...
}};
```

### 5.2 类型别名创建

```cpp
// src/catalog/default/default_types.cpp

LogicalTypeId DefaultTypeGenerator::GetDefaultType(const string &name) {
    auto &internal_types = BUILTIN_TYPES;
    for (auto &type : internal_types) {
        if (StringUtil::CIEquals(name, type.name)) {  // 大小写不敏感
            return type.type;
        }
    }
    return LogicalType::INVALID;
}

unique_ptr<CatalogEntry> DefaultTypeGenerator::CreateDefaultEntry(
    ClientContext &context, const string &entry_name) {

    // 类型别名只在 main schema 中有效
    if (schema.name != DEFAULT_SCHEMA) {
        return nullptr;
    }

    auto type_id = GetDefaultType(entry_name);
    if (type_id == LogicalTypeId::INVALID) {
        return nullptr;
    }

    CreateTypeInfo info;
    info.name = entry_name;
    info.type = LogicalType(type_id);
    info.internal = true;
    info.temporary = true;

    return make_uniq_base<CatalogEntry, TypeCatalogEntry>(catalog, schema, info);
}
```

### 5.3 使用示例

```sql
-- 以下写法等价
CREATE TABLE t1 (id int);
CREATE TABLE t2 (id integer);
CREATE TABLE t3 (id int4);
CREATE TABLE t4 (id int32);

-- 类型别名解析
SELECT typeof(1::int8);      -- BIGINT
SELECT typeof('hello'::text); -- VARCHAR
```

## 6. DefaultTableFunctionGenerator：表宏

### 6.1 表宏定义

```
源文件: src/catalog/default/default_table_functions.cpp
```

表宏是返回表的宏函数：

```cpp
struct DefaultTableMacro {
    const char *schema;
    const char *name;
    const char *parameters[8];
    DefaultNamedParameter named_parameters[8];
    const char *macro;  // SELECT 语句
};

static const DefaultTableMacro internal_table_macros[] = {
    // 直方图函数
    {DEFAULT_SCHEMA, "histogram", {"source", "col_name", nullptr},
     {{"bin_count", "10"}, {"technique", "'auto'"}, {nullptr, nullptr}},
     R"(
SELECT
   CASE
   WHEN is_histogram_other_bin(bin) THEN '(other values)'
   WHEN ... THEN bin::VARCHAR
   WHEN row_number() over () = 1 THEN concat('x <= ', bin::VARCHAR)
   ELSE concat(lag(bin::VARCHAR) over (), ' < x <= ', bin::VARCHAR)
   END AS bin,
   count,
   bar(count, 0, max(count) over ()) AS bar
FROM histogram_values(source, col_name, bin_count := bin_count, technique := technique);
)"},

    // 日志解析函数
    {DEFAULT_SCHEMA, "duckdb_logs_parsed", {"log_type"}, {},
     R"(
SELECT * EXCLUDE (message), UNNEST(parse_duckdb_log_message(log_type, message))
FROM duckdb_logs(denormalized_table=1)
WHERE type ILIKE log_type
)"},

    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}
};
```

### 6.2 表宏创建

```cpp
unique_ptr<CreateMacroInfo> DefaultTableFunctionGenerator::CreateTableMacroInfo(
    const DefaultTableMacro &default_macro) {

    Parser parser;
    parser.ParseQuery(default_macro.macro);

    if (parser.statements.size() != 1 ||
        parser.statements[0]->type != StatementType::SELECT_STATEMENT) {
        throw InternalException("Expected a single select statement");
    }

    auto node = std::move(parser.statements[0]->Cast<SelectStatement>().node);
    auto result = make_uniq<TableMacroFunction>(std::move(node));

    return CreateInternalTableMacroInfo(default_macro, std::move(result));
}
```

## 7. DuckSchemaEntry 中的生成器配置

### 7.1 初始化流程

```
源文件: src/catalog/catalog_entry/duck_schema_entry.cpp
```

DuckSchemaEntry 构造函数配置各个 CatalogSet 的 DefaultGenerator：

```cpp
DuckSchemaEntry::DuckSchemaEntry(Catalog &catalog, CreateSchemaInfo &info)
    : SchemaCatalogEntry(catalog, info),
      // tables 使用 DefaultViewGenerator（系统视图）
      tables(catalog, catalog.IsSystemCatalog()
             ? make_uniq<DefaultViewGenerator>(catalog, *this)
             : nullptr),
      indexes(catalog),
      // table_functions 使用 DefaultTableFunctionGenerator
      table_functions(catalog, catalog.IsSystemCatalog()
                      ? make_uniq<DefaultTableFunctionGenerator>(catalog, *this)
                      : nullptr),
      copy_functions(catalog),
      // functions 使用 DefaultFunctionGenerator（宏函数）
      functions(catalog, catalog.IsSystemCatalog()
                ? make_uniq<DefaultFunctionGenerator>(catalog, *this)
                : nullptr),
      sequences(catalog),
      collations(catalog),
      // types 始终使用 DefaultTypeGenerator（类型别名）
      types(catalog, make_uniq<DefaultTypeGenerator>(catalog, *this)) {
}
```

关键设计：

1. **条件配置**：只有 System Catalog 的 Schema 才配置视图/函数生成器
2. **类型始终启用**：所有 Schema 都支持类型别名
3. **扩展无侵入**：用户数据库的 Schema 不会受默认对象影响

### 7.2 IsSystemCatalog 判断

```cpp
bool Catalog::IsSystemCatalog() const {
    return GetAttached().IsSystem();
}
```

System Catalog 是 DuckDB 启动时自动创建的内部数据库，用于存储内置函数和系统视图。

## 8. BuiltinFunctions：核心函数注册

### 8.1 与 DefaultGenerator 的区别

DuckDB 有两种内置函数注册方式：

| 特性 | DefaultGenerator (宏) | BuiltinFunctions |
|------|---------------------|------------------|
| 实现方式 | SQL 宏表达式 | C++ 函数指针 |
| 加载时机 | 懒加载 | 启动时注册 |
| 性能 | 需要表达式求值 | 直接调用 |
| 适用场景 | 组合函数、兼容函数 | 核心计算函数 |

### 8.2 BuiltinFunctions 注册

```
源文件: src/function/built_in_functions.cpp
```

```cpp
BuiltinFunctions::BuiltinFunctions(CatalogTransaction transaction, Catalog &catalog)
    : transaction(transaction), catalog(catalog) {
}

void BuiltinFunctions::AddFunction(ScalarFunctionSet set) {
    CreateScalarFunctionInfo info(std::move(set));
    info.internal = true;
    catalog.CreateFunction(transaction, info);
}

void BuiltinFunctions::AddFunction(AggregateFunction function) {
    CreateAggregateFunctionInfo info(std::move(function));
    info.internal = true;
    catalog.CreateFunction(transaction, info);
}
```

### 8.3 FunctionList 静态注册

```
源文件: src/function/function_list.cpp
```

DuckDB 使用宏生成函数列表：

```cpp
struct StaticFunctionDefinition {
    const char *name;
    const char *alias_of;
    const char *parameters;
    const char *description;
    const char *example;
    const char *categories;
    get_scalar_function_t get_function;
    get_scalar_function_set_t get_function_set;
    get_aggregate_function_t get_aggregate_function;
    get_aggregate_function_set_t get_aggregate_function_set;
};

// 函数列表（由 scripts/generate_functions.py 生成）
static const StaticFunctionDefinition function[] = {
    DUCKDB_SCALAR_FUNCTION_SET(OperatorAddFun),
    DUCKDB_SCALAR_FUNCTION_SET(OperatorSubtractFun),
    DUCKDB_SCALAR_FUNCTION_SET(OperatorMultiplyFun),
    DUCKDB_SCALAR_FUNCTION(LowerFun),
    DUCKDB_SCALAR_FUNCTION(UpperFun),
    DUCKDB_SCALAR_FUNCTION_SET(SubstringFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(SumFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(AvgFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(CountFun),
    // ... 200+ 个函数
    FINAL_FUNCTION
};
```

### 8.4 DuckCatalog 初始化

```cpp
// src/catalog/duck_catalog.cpp
void DuckCatalog::Initialize(bool load_builtin) {
    // 创建默认 Schema
    CreateSchemaInfo info;
    info.schema = DEFAULT_SCHEMA;
    info.internal = true;
    auto main_schema = CreateSchemaInternal(data->system_transaction, info);

    if (load_builtin) {
        // 注册核心函数
        FunctionList::RegisterFunctions(*this, data->system_transaction);
    }
}
```

## 9. 扩展函数自动加载

### 9.1 扩展占位符机制

DuckDB 支持函数的自动加载——当用户调用未加载扩展中的函数时，自动加载该扩展：

```cpp
// src/function/built_in_functions.cpp
struct ExtensionFunctionInfo : public ScalarFunctionInfo {
    explicit ExtensionFunctionInfo(string extension_p)
        : extension(std::move(extension_p)) {}
    string extension;
};

unique_ptr<FunctionData> BindExtensionFunction(
    ClientContext &context, ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments) {

    auto &function_info = bound_function.GetExtraFunctionInfo()
                            .Cast<ExtensionFunctionInfo>();
    auto &extension_name = function_info.extension;
    auto &db = *context.db;

    if (!ExtensionHelper::CanAutoloadExtension(extension_name)) {
        throw BinderException(
            "Function \"%s\" is in extension \"%s\" which is not loaded "
            "and could not be auto-loaded",
            bound_function.name, extension_name);
    }

    // 自动加载扩展
    ExtensionHelper::AutoLoadExtension(db, extension_name);

    // 从 Catalog 获取真正的函数
    auto &catalog = Catalog::GetSystemCatalog(db);
    auto &function_entry = catalog.GetEntry<ScalarFunctionCatalogEntry>(
        context, DEFAULT_SCHEMA, bound_function.name);

    // 用扩展函数替换占位符
    bound_function = function_entry.functions.GetFunctionByArguments(
        context, bound_function.arguments);

    // 调用真正的绑定函数
    if (!bound_function.HasBindCallback()) {
        return nullptr;
    }
    return bound_function.GetBindCallback()(context, bound_function, arguments);
}
```

### 9.2 扩展占位符注册

```cpp
void BuiltinFunctions::RegisterExtensionOverloads() {
    ScalarFunctionSet current_set;

    for (auto &entry : EXTENSION_FUNCTION_OVERLOADS) {
        // 解析函数签名
        vector<LogicalType> arguments;
        auto splits = StringUtil::Split(entry.signature, ">");
        auto return_type = DBConfig::ParseLogicalType(splits[1]);
        // ...

        // 创建占位符函数
        ScalarFunction function(entry.name, std::move(arguments),
                                 std::move(return_type),
                                 nullptr,              // 无实现
                                 BindExtensionFunction); // 绑定时加载扩展
        function.SetExtraFunctionInfo<ExtensionFunctionInfo>(entry.extension);

        current_set.AddFunction(std::move(function));
    }

    AddExtensionFunction(std::move(current_set));
}
```

这种设计允许用户无缝使用扩展函数：

```sql
-- 首次调用时自动加载 json 扩展
SELECT json_extract('{"a": 1}', '$.a');
```

## 10. CreateDefaultEntries：全量创建

### 10.1 遍历所有默认条目

当执行 `SHOW ALL FUNCTIONS` 或类似命令时，需要显示所有内置函数：

```cpp
void CatalogSet::CreateDefaultEntries(CatalogTransaction transaction,
                                       unique_lock<mutex> &read_lock) {
    if (!defaults || defaults->created_all_entries) {
        return;
    }

    // 获取所有默认条目名称
    auto default_entries = defaults->GetDefaultEntries();

    for (auto &default_entry : default_entries) {
        auto entry_value = map.GetEntry(default_entry);
        if (!entry_value) {
            // 创建条目（可能访问其他 CatalogSet，需要释放锁）
            read_lock.unlock();
            auto entry = defaults->CreateDefaultEntry(transaction, default_entry);
            if (!entry) {
                throw InternalException("Failed to create default entry for %s",
                                         default_entry);
            }

            read_lock.lock();
            CreateCommittedEntry(std::move(entry));
        }
    }

    // 标记已创建所有条目，避免重复
    defaults->created_all_entries = true;
}
```

### 10.2 性能考虑

全量创建是一次性操作，之后 `created_all_entries` 标志会阻止重复创建：

```cpp
if (!defaults || defaults->created_all_entries) {
    return nullptr;  // 快速返回
}
```

## 11. 总结

### 11.1 DefaultGenerator 设计优势

1. **启动速度**：避免解析数百个内置对象
2. **内存效率**：只加载实际使用的对象
3. **扩展性**：新增内置对象只需添加数组元素
4. **兼容性**：通过 pg_catalog 和 information_schema 提供标准接口

### 11.2 生成器类型总结

| 生成器类型 | 生成对象 | 典型数量 | 示例 |
|-----------|---------|---------|------|
| DefaultSchemaGenerator | 系统 Schema | 2 | pg_catalog, information_schema |
| DefaultFunctionGenerator | 宏函数 | 170+ | current_user, list_append |
| DefaultViewGenerator | 系统视图 | 60+ | duckdb_tables, pg_class |
| DefaultTypeGenerator | 类型别名 | 77 | int→INTEGER, text→VARCHAR |
| DefaultTableFunctionGenerator | 表宏 | 3 | histogram, duckdb_logs_parsed |

### 11.3 函数注册对比

```
┌─────────────────────────────────────────────────────────────────┐
│                      DuckDB 内置函数架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              BuiltinFunctions (启动注册)                   │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │ │
│  │  │ 算术运算符   │ │ 字符串函数  │ │ 聚合函数    │          │ │
│  │  │ +, -, *, /  │ │ lower,upper │ │ sum,avg,max │          │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘          │ │
│  │                   (C++ 实现，高性能)                        │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              DefaultGenerator (懒加载)                      │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │ │
│  │  │ 组合函数    │ │ 兼容函数    │ │ 系统视图    │          │ │
│  │  │ list_sum    │ │ pg_typeof   │ │ pg_class    │          │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘          │ │
│  │                   (SQL 宏实现，灵活扩展)                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Extension Functions (自动加载)                 │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │ │
│  │  │ json_*      │ │ parquet_*   │ │ icu_*       │          │ │
│  │  │ (json ext)  │ │ (parquet)   │ │ (icu ext)   │          │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘          │ │
│  │                   (占位符 + 按需加载)                       │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

这种分层设计在功能完整性、启动性能和运行时效率之间取得了良好的平衡。

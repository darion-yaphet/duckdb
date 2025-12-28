# DuckDB Catalog 系统深度解析 - 第五章：名称解析与搜索路径

本章深入分析 DuckDB 的名称解析机制，包括 CatalogSearchPath 搜索路径、三级名称解析（catalog.schema.name）以及 EntryLookupInfo 查找策略。

## 5.1 名称解析概述

### 5.1.1 三级命名结构

DuckDB 使用三级名称来唯一标识对象：

```
catalog.schema.name

示例:
- memory.main.customers        → 数据库 memory 中 main schema 的 customers 表
- system.pg_catalog.pg_class   → 系统 catalog 中的 pg_class 表
- temp.main.tmp_data           → 临时 catalog 中的临时表
```

### 5.1.2 名称解析挑战

用户通常只提供部分名称：

```sql
-- 只提供名称
SELECT * FROM customers;

-- 提供 schema.name
SELECT * FROM main.customers;

-- 完整的三级名称
SELECT * FROM memory.main.customers;
```

名称解析的任务是根据搜索路径找到完整的对象定位。

---

## 5.2 CatalogSearchPath

### 5.2.1 核心结构

```cpp
// src/include/duckdb/catalog/catalog_search_path.hpp

struct CatalogSearchEntry {
    string catalog;  // 数据库名
    string schema;   // Schema 名

    CatalogSearchEntry(string catalog, string schema);
    string ToString() const;

    static CatalogSearchEntry Parse(const string &input);
    static vector<CatalogSearchEntry> ParseList(const string &input);
};

class CatalogSearchPath {
public:
    CatalogSearchPath(ClientContext &context);

    //! 设置搜索路径
    void Set(CatalogSearchEntry new_value, CatalogSetPathType set_type);
    void Set(vector<CatalogSearchEntry> new_paths, CatalogSetPathType set_type);

    //! 获取当前搜索路径
    const vector<CatalogSearchEntry> &Get() const;

    //! 获取默认值
    const CatalogSearchEntry &GetDefault() const;
    string GetDefaultSchema(const string &catalog) const;
    string GetDefaultCatalog(const string &schema) const;

    //! 检查 schema 是否在搜索路径中
    bool SchemaInSearchPath(ClientContext &context,
                            const string &catalog_name,
                            const string &schema_name) const;

private:
    ClientContext &context;
    vector<CatalogSearchEntry> paths;      // 完整路径（包含自动添加的）
    vector<CatalogSearchEntry> set_paths;  // 用户设置的路径
};
```

### 5.2.2 搜索路径初始化

```cpp
void CatalogSearchPath::SetPathsInternal(vector<CatalogSearchEntry> new_paths) {
    this->set_paths = std::move(new_paths);

    paths.clear();
    paths.reserve(set_paths.size() + 3);

    // 1. 始终先搜索临时 Catalog
    paths.emplace_back(TEMP_CATALOG, DEFAULT_SCHEMA);  // temp.main

    // 2. 用户设置的路径
    for (auto &path : set_paths) {
        paths.push_back(path);
    }

    // 3. 默认数据库的 main schema
    paths.emplace_back(INVALID_CATALOG, DEFAULT_SCHEMA);  // <default>.main

    // 4. 系统 Catalog
    paths.emplace_back(SYSTEM_CATALOG, DEFAULT_SCHEMA);   // system.main
    paths.emplace_back(SYSTEM_CATALOG, "pg_catalog");     // system.pg_catalog
}
```

### 5.2.3 默认搜索路径

```
默认搜索路径顺序:

┌─────────────────────────────────────────────────────────────────┐
│                     搜索顺序                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. temp.main              ← 临时对象（会话级）                  │
│  2. <user_set_paths>       ← 用户通过 SET search_path 设置      │
│  3. <default_db>.main      ← 默认数据库的 main schema           │
│  4. system.main            ← 系统内置对象                       │
│  5. system.pg_catalog      ← PostgreSQL 兼容对象                │
│                                                                  │
│  示例（用户设置了 memory.public）:                               │
│  [temp.main, memory.public, memory.main, system.main, ...]      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 SET search_path 语句

### 5.3.1 设置方式

```cpp
enum class CatalogSetPathType : uint8_t {
    SET_SCHEMA,      // SET schema = 'xxx'（只能设置一个）
    SET_SCHEMAS,     // SET search_path = 'xxx,yyy'（可以多个）
    SET_DIRECTLY     // 内部使用，不验证
};
```

### 5.3.2 Set 方法

```cpp
void CatalogSearchPath::Set(
    vector<CatalogSearchEntry> new_paths,
    CatalogSetPathType set_type) {

    // SET schema 只能设置一个
    if (set_type == CatalogSetPathType::SET_SCHEMA && new_paths.size() != 1) {
        throw CatalogException("SET schema can set only 1 schema");
    }

    for (auto &path : new_paths) {
        // 验证 schema 存在
        auto schema_entry = Catalog::GetSchema(
            context, path.catalog, path.schema, OnEntryNotFound::RETURN_NULL);

        if (schema_entry) {
            // 找到 schema，补全 catalog 名
            if (path.catalog.empty()) {
                path.catalog = GetDefault().catalog;
            }
            continue;
        }

        // 如果只提供了 schema 名，尝试作为 catalog 名解析
        if (path.catalog.empty()) {
            auto catalog = Catalog::GetCatalogEntry(context, path.schema);
            if (catalog) {
                auto schema = catalog->GetSchema(
                    context, catalog->GetDefaultSchema(),
                    OnEntryNotFound::RETURN_NULL);
                if (schema) {
                    path.catalog = std::move(path.schema);
                    path.schema = schema->name;
                    continue;
                }
            }
        }

        throw CatalogException("No catalog + schema named \"%s\" found.",
                               path.ToString());
    }

    // 不允许设置为系统内部 schema
    if (set_type == CatalogSetPathType::SET_SCHEMA) {
        if (new_paths[0].catalog == TEMP_CATALOG ||
            new_paths[0].catalog == SYSTEM_CATALOG) {
            throw CatalogException("Cannot set to internal schema");
        }
    }

    SetPathsInternal(std::move(new_paths));
}
```

### 5.3.3 SQL 示例

```sql
-- 设置默认 schema
SET schema = 'public';

-- 设置搜索路径（多个 schema）
SET search_path = 'analytics,public,shared';

-- 带 catalog 的路径
SET search_path = 'mydb.analytics,mydb.public';

-- 查看当前搜索路径
SELECT * FROM duckdb_settings() WHERE name = 'search_path';
```

---

## 5.4 名称解析流程

### 5.4.1 TryLookupEntry

```cpp
// src/catalog/catalog.cpp

CatalogEntryLookup Catalog::TryLookupEntry(
    CatalogEntryRetriever &retriever,
    const string &catalog,
    const string &schema,
    const EntryLookupInfo &lookup_info,
    OnEntryNotFound if_not_found) {

    auto &context = retriever.GetContext();

    // 情况1: 指定了完整的 catalog.schema
    if (!schema.empty()) {
        auto result = TryLookupEntryInternal(
            retriever, catalog, schema, lookup_info);
        if (result.Found() || if_not_found == OnEntryNotFound::RETURN_NULL) {
            return result;
        }
        // 未找到，生成错误信息
        auto error = CreateMissingEntryError(
            context, lookup_info, catalog, schema);
        throw CatalogException(error);
    }

    // 情况2: 只提供了名称，使用搜索路径
    return TryLookupEntryFromSearchPath(
        retriever, catalog, lookup_info, if_not_found);
}
```

### 5.4.2 TryLookupEntryFromSearchPath

```cpp
CatalogEntryLookup Catalog::TryLookupEntryFromSearchPath(
    CatalogEntryRetriever &retriever,
    const string &catalog,
    const EntryLookupInfo &lookup_info,
    OnEntryNotFound if_not_found) {

    auto &context = retriever.GetContext();
    auto &search_path = ClientData::Get(context).catalog_search_path;

    // 遍历搜索路径
    for (auto &entry : search_path.Get()) {
        // 确定要搜索的 catalog
        string search_catalog = entry.catalog;
        if (IsInvalidCatalog(search_catalog)) {
            search_catalog = DatabaseManager::GetDefaultDatabase(context);
        }

        // 如果指定了 catalog，只搜索该 catalog
        if (!catalog.empty() && !StringUtil::CIEquals(catalog, search_catalog)) {
            continue;
        }

        // 在当前 catalog.schema 中查找
        auto result = TryLookupEntryInternal(
            retriever, search_catalog, entry.schema, lookup_info);

        if (result.Found()) {
            return result;
        }
    }

    // 未找到
    if (if_not_found == OnEntryNotFound::RETURN_NULL) {
        return CatalogEntryLookup();
    }

    // 生成错误信息
    auto error = CreateMissingEntryError(
        context, lookup_info, catalog, string());
    throw CatalogException(error);
}
```

### 5.4.3 解析流程图

```
输入: "customers" (只提供名称)

                    ┌─────────────────────────────────────────┐
                    │ TryLookupEntryFromSearchPath            │
                    └─────────────────┬───────────────────────┘
                                      │
         搜索路径: [temp.main, memory.public, memory.main, system.main, system.pg_catalog]
                                      │
                    ┌─────────────────┴───────────────────────┐
                    │                                         │
                    ▼                                         │
           ┌────────────────┐                                │
           │ temp.main      │ ← 查找 customers                │
           │ 找到? ─NO──────┼─────────────────────────────────┤
           └────────────────┘                                │
                    │                                         │
                    ▼                                         │
           ┌────────────────┐                                │
           │ memory.public  │ ← 查找 customers                │
           │ 找到? ─NO──────┼─────────────────────────────────┤
           └────────────────┘                                │
                    │                                         │
                    ▼                                         │
           ┌────────────────┐                                │
           │ memory.main    │ ← 查找 customers                │
           │ 找到? ─YES─────┼────► 返回 memory.main.customers │
           └────────────────┘                                │
                                                              │
                    ▼ (如果都未找到)                           │
           ┌────────────────┐                                │
           │ 抛出           │                                │
           │ CatalogException│                                │
           └────────────────┘                                │
```

---

## 5.5 EntryLookupInfo

### 5.5.1 结构定义

```cpp
// src/include/duckdb/catalog/entry_lookup_info.hpp

struct EntryLookupInfo {
    CatalogType type;              // 查找的条目类型
    string name;                   // 条目名称
    QueryErrorContext error_ctx;   // 错误上下文（用于错误消息）

    EntryLookupInfo(CatalogType type, const string &name,
                    QueryErrorContext error_context = {});

    const string &GetEntryName() const;
    CatalogType GetCatalogType() const;
    const QueryErrorContext &GetErrorContext() const;
};
```

### 5.5.2 使用示例

```cpp
// 查找表
EntryLookupInfo lookup_info(CatalogType::TABLE_ENTRY, "customers");
auto result = catalog.GetEntry(context, "main", lookup_info,
                               OnEntryNotFound::THROW_EXCEPTION);

// 查找函数
EntryLookupInfo func_lookup(CatalogType::SCALAR_FUNCTION_ENTRY, "my_func");
auto func = catalog.GetEntry(context, "", func_lookup,
                             OnEntryNotFound::RETURN_NULL);
```

---

## 5.6 CatalogLookupBehavior

### 5.6.1 行为定义

```cpp
enum class CatalogLookupBehavior : uint8_t {
    STANDARD = 0,       // 标准查找
    LOWER_PRIORITY = 1, // 降低优先级
    NEVER = 2           // 不查找此 catalog
};
```

### 5.6.2 用途

用于控制特定 catalog 在查找时的行为：

```cpp
// 某些扩展可能希望降低其 catalog 的优先级
// 或完全禁止在某些 catalog 中查找

CatalogLookupBehavior GetCatalogLookupBehavior(
    const string &catalog_name) {

    if (catalog_name == TEMP_CATALOG) {
        return CatalogLookupBehavior::STANDARD;
    }
    // ... 根据配置返回不同行为
}
```

---

## 5.7 获取默认值

### 5.7.1 GetDefault

获取默认的 catalog 和 schema：

```cpp
const CatalogSearchEntry &CatalogSearchPath::GetDefault() const {
    const auto &paths = Get();
    D_ASSERT(paths.size() >= 2);
    // 返回第二个元素（跳过 temp）
    return paths[1];
}
```

### 5.7.2 GetDefaultSchema

获取指定 catalog 的默认 schema：

```cpp
string CatalogSearchPath::GetDefaultSchema(const string &catalog) const {
    for (auto &path : paths) {
        if (path.catalog == TEMP_CATALOG) {
            continue;  // 跳过临时 catalog
        }
        if (StringUtil::CIEquals(path.catalog, catalog)) {
            return path.schema;
        }
    }
    return DEFAULT_SCHEMA;  // 默认返回 "main"
}
```

### 5.7.3 GetDefaultCatalog

获取指定 schema 的默认 catalog：

```cpp
string CatalogSearchPath::GetDefaultCatalog(const string &schema) const {
    // 默认 schema（如 pg_catalog）属于系统 catalog
    if (DefaultSchemaGenerator::IsDefaultSchema(schema)) {
        return SYSTEM_CATALOG;
    }

    for (auto &path : paths) {
        if (path.catalog == TEMP_CATALOG) {
            continue;
        }
        if (StringUtil::CIEquals(path.schema, schema)) {
            return path.catalog;
        }
    }
    return INVALID_CATALOG;
}
```

---

## 5.8 搜索路径解析

### 5.8.1 CatalogSearchEntry::Parse

解析单个搜索路径条目：

```cpp
CatalogSearchEntry CatalogSearchEntry::Parse(const string &input) {
    idx_t pos = 0;
    auto result = ParseInternal(input, pos);
    if (pos < input.size()) {
        throw ParserException("Expected a single entry");
    }
    return result;
}

// 支持的格式:
// "schema"          → (empty, schema)
// "catalog.schema"  → (catalog, schema)
// "\"my.schema\""   → (empty, "my.schema") 引号处理
```

### 5.8.2 CatalogSearchEntry::ParseList

解析搜索路径列表：

```cpp
vector<CatalogSearchEntry> CatalogSearchEntry::ParseList(const string &input) {
    idx_t pos = 0;
    vector<CatalogSearchEntry> result;
    while (pos < input.size()) {
        auto entry = ParseInternal(input, pos);
        result.push_back(entry);
    }
    return result;
}

// 示例:
// "public,analytics,shared"
//   → [(empty, public), (empty, analytics), (empty, shared)]
//
// "db1.public,db2.main"
//   → [(db1, public), (db2, main)]
```

---

## 5.9 SchemaInSearchPath

检查 schema 是否在搜索路径中：

```cpp
bool CatalogSearchPath::SchemaInSearchPath(
    ClientContext &context,
    const string &catalog_name,
    const string &schema_name) const {

    for (auto &path : paths) {
        if (!StringUtil::CIEquals(path.schema, schema_name)) {
            continue;
        }
        if (StringUtil::CIEquals(path.catalog, catalog_name)) {
            return true;
        }
        // 处理 INVALID_CATALOG（代表默认数据库）
        if (IsInvalidCatalog(path.catalog) &&
            StringUtil::CIEquals(catalog_name,
                                 DatabaseManager::GetDefaultDatabase(context))) {
            return true;
        }
    }
    return false;
}
```

---

## 5.10 错误处理与建议

### 5.10.1 CreateMissingEntryError

生成友好的错误消息：

```cpp
static CatalogException CreateMissingEntryError(
    ClientContext &context,
    const EntryLookupInfo &lookup_info,
    const string &catalog,
    const string &schema) {

    string entry_name = lookup_info.GetEntryName();
    CatalogType type = lookup_info.GetCatalogType();

    // 查找相似名称
    SimilarCatalogEntry similar = FindSimilarEntry(
        context, type, entry_name, catalog, schema);

    string error_message = StringUtil::Format(
        "%s \"%s\" does not exist",
        CatalogTypeToString(type), entry_name);

    // 添加建议
    if (!similar.name.empty() && similar.score > 0.5) {
        error_message += StringUtil::Format(
            "\nDid you mean \"%s\"?", similar.name);
    }

    return CatalogException(lookup_info.GetErrorContext(), error_message);
}
```

### 5.10.2 错误消息示例

```
错误: Table "customrs" does not exist
Did you mean "customers"?

Catalog Error: Schema "publc" does not exist
Available schemas: main, public, information_schema

Catalog Error: No catalog + schema named "nonexistent" found.
```

---

## 5.11 本章小结

本章详细分析了 DuckDB 的名称解析机制：

1. **三级命名结构**：
   - 格式：`catalog.schema.name`
   - 支持部分名称，使用搜索路径补全

2. **CatalogSearchPath**：
   - 管理搜索路径列表
   - 默认顺序：temp → 用户设置 → 默认数据库 → 系统
   - 支持 SET schema 和 SET search_path

3. **名称解析流程**：
   - 指定 schema 时直接查找
   - 只有名称时遍历搜索路径
   - 第一个匹配的结果被返回

4. **EntryLookupInfo**：
   - 封装查找类型和名称
   - 携带错误上下文

5. **默认值获取**：
   - `GetDefault()`：默认 catalog.schema
   - `GetDefaultSchema()`：指定 catalog 的默认 schema
   - `GetDefaultCatalog()`：指定 schema 的默认 catalog

6. **错误处理**：
   - 提供相似名称建议
   - 友好的错误消息

---

## 5.12 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog_search_path.hpp` | CatalogSearchPath 定义 |
| `src/catalog/catalog_search_path.cpp` | CatalogSearchPath 实现 |
| `src/include/duckdb/catalog/entry_lookup_info.hpp` | EntryLookupInfo 定义 |
| `src/catalog/catalog_entry_retriever.cpp` | 条目检索器 |
| `src/catalog/catalog.cpp` | Catalog 名称解析逻辑 |

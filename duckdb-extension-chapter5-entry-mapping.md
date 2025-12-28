# 第五章：扩展入口与函数映射

## 概述

DuckDB 的扩展系统需要解决两个核心问题：如何定义扩展的入口点，以及如何在用户调用未加载扩展的函数时自动加载相应扩展。本章将深入分析扩展入口定义、函数映射表、自动发现机制和别名系统的实现。

## 扩展入口点定义

### C++ ABI 入口

C++ 扩展的入口函数遵循以下命名规范：

```
<extension_name>_duckdb_cpp_init
```

例如，`parquet` 扩展的入口函数名为 `parquet_duckdb_cpp_init`。

入口函数签名：

```cpp
typedef void (*ext_init_fun_t)(ExtensionLoader &);

// 扩展实现示例
extern "C" {
void parquet_duckdb_cpp_init(ExtensionLoader &loader) {
    // 注册函数、类型等
    ParquetExtension::Load(loader);
}
}
```

### C ABI 入口

C ABI 扩展使用不同的入口命名：

```
<extension_name>_init_c_api
```

入口函数签名：

```cpp
typedef bool (*ext_init_c_api_fun_t)(duckdb_extension_info info,
                                      duckdb_extension_access *access);

// C 扩展实现示例
bool my_extension_init_c_api(duckdb_extension_info info,
                              duckdb_extension_access *access) {
    // 获取 API
    auto api = (duckdb_ext_api_v1 *)access->get_api(info, "v1.0.0");
    if (!api) {
        access->set_error(info, "Failed to get API");
        return false;
    }

    // 注册功能
    // ...

    return true;
}
```

### 入口加载流程

```cpp
// extension_load.cpp
void ExtensionHelper::LoadExternalExtensionInternal(...) {
    auto extension_init_result = InitialLoad(db, fs, extension);

    // 根据 ABI 类型选择入口
    if (extension_init_result.abi_type == ExtensionABIType::CPP) {
        auto init_fun_name = extension_init_result.filebase + "_duckdb_cpp_init";
        ext_init_fun_t init_fun = TryLoadFunctionFromDLL<ext_init_fun_t>(
            extension_init_result.lib_hdl, init_fun_name,
            extension_init_result.filename);

        if (!init_fun) {
            throw IOException("Extension did not contain entrypoint '%s'",
                              init_fun_name);
        }

        ExtensionLoader loader(info);
        (*init_fun)(loader);
        loader.FinalizeLoad();
    } else if (extension_init_result.abi_type == ExtensionABIType::C_STRUCT ||
               extension_init_result.abi_type == ExtensionABIType::C_STRUCT_UNSTABLE) {
        auto init_fun_name = extension_init_result.filebase + "_init_c_api";
        ext_init_c_api_fun_t init_fun_capi = TryLoadFunctionFromDLL<
            ext_init_c_api_fun_t>(...);

        DuckDBExtensionLoadState load_state(db, extension_init_result);
        auto access = ExtensionAccess::CreateAccessStruct();
        (*init_fun_capi)(load_state.ToCStruct(), &access);
    }
}
```

## 扩展函数映射表

### 数据结构定义

DuckDB 在编译时生成一个静态映射表，记录所有已知扩展提供的功能：

```cpp
// extension_entries.hpp

// 基础名称映射
struct ExtensionEntry {
    char name[48];       // 条目名称
    char extension[48];  // 所属扩展
};

// 函数映射（包含类型）
struct ExtensionFunctionEntry {
    char name[48];       // 函数名
    char extension[48];  // 所属扩展
    CatalogType type;    // 函数类型（标量/聚合/表函数等）
};

// 函数重载映射（包含签名）
struct ExtensionFunctionOverloadEntry {
    char name[48];       // 函数名
    char extension[48];  // 所属扩展
    CatalogType type;    // 函数类型
    char signature[96];  // 函数签名
};
```

### 函数映射表

```cpp
static constexpr ExtensionFunctionEntry EXTENSION_FUNCTIONS[] = {
    // 运算符
    {"!__postfix", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"&", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"**", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"->>", "json", CatalogType::SCALAR_FUNCTION_ENTRY},

    // 数学函数
    {"abs", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"acos", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"sqrt", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY},

    // 聚合函数
    {"avg", "core_functions", CatalogType::AGGREGATE_FUNCTION_ENTRY},
    {"sum", "core_functions", CatalogType::AGGREGATE_FUNCTION_ENTRY},
    {"approx_count_distinct", "core_functions", CatalogType::AGGREGATE_FUNCTION_ENTRY},

    // 表函数
    {"delta_scan", "delta", CatalogType::TABLE_FUNCTION_ENTRY},
    {"iceberg_scan", "iceberg", CatalogType::TABLE_FUNCTION_ENTRY},
    {"parquet_scan", "parquet", CatalogType::TABLE_FUNCTION_ENTRY},

    // JSON 扩展
    {"from_json", "json", CatalogType::SCALAR_FUNCTION_ENTRY},
    {"array_to_json", "json", CatalogType::SCALAR_FUNCTION_ENTRY},

    // TPC-H/TPC-DS
    {"dbgen", "tpch", CatalogType::TABLE_FUNCTION_ENTRY},
    {"dsdgen", "tpcds", CatalogType::TABLE_FUNCTION_ENTRY},

    // ...
};
```

### 重载映射表

对于有多个重载版本的函数，使用重载映射表：

```cpp
static constexpr ExtensionFunctionOverloadEntry EXTENSION_FUNCTION_OVERLOADS[] = {
    // 不同输入类型的 year 函数
    {"year", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY, "[DATE]>BIGINT"},
    {"year", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY, "[INTERVAL]>BIGINT"},
    {"year", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY, "[TIMESTAMP]>BIGINT"},
    {"year", "icu", CatalogType::SCALAR_FUNCTION_ENTRY, "[TIMESTAMPTZ]>BIGINT"},

    // strftime 的多种形式
    {"strftime", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY, "[DATE,VARCHAR]>VARCHAR"},
    {"strftime", "core_functions", CatalogType::SCALAR_FUNCTION_ENTRY, "[TIMESTAMP,VARCHAR]>VARCHAR"},
    {"strftime", "icu", CatalogType::SCALAR_FUNCTION_ENTRY, "[TIMESTAMPTZ,VARCHAR]>VARCHAR"},

    // ...
};
```

## 多维度映射表

### 设置映射

```cpp
static constexpr ExtensionEntry EXTENSION_SETTINGS[] = {
    // HTTP/S3 相关设置
    {"s3_access_key_id", "httpfs"},
    {"s3_endpoint", "httpfs"},
    {"s3_region", "httpfs"},
    {"s3_secret_access_key", "httpfs"},

    // Azure 设置
    {"azure_account_name", "azure"},
    {"azure_storage_connection_string", "azure"},

    // ICU 设置
    {"calendar", "icu"},
    {"timezone", "icu"},

    // Parquet 设置
    {"binary_as_string", "parquet"},
    {"parquet_metadata_cache", "parquet"},

    // ...
};
```

### 类型映射

```cpp
static constexpr ExtensionEntry EXTENSION_TYPES[] = {
    {"json", "json"},
    {"inet", "inet"},
    {"geometry", "spatial"},
};
```

### 排序规则映射

```cpp
static constexpr ExtensionEntry EXTENSION_COLLATIONS[] = {
    {"af", "icu"},    {"am", "icu"},    {"ar", "icu"},
    {"de", "icu"},    {"en", "icu"},    {"en_us", "icu"},
    {"fr", "icu"},    {"ja", "icu"},    {"ko", "icu"},
    {"zh", "icu"},    {"zh_cn", "icu"}, {"zh_tw", "icu"},
    // ... 100+ 种排序规则
};
```

### 文件前缀映射

```cpp
static constexpr ExtensionEntry EXTENSION_FILE_PREFIXES[] = {
    {"http://", "httpfs"},
    {"https://", "httpfs"},
    {"s3://", "httpfs"},
    {"s3a://", "httpfs"},
    {"s3n://", "httpfs"},
    {"gcs://", "httpfs"},
    {"gs://", "httpfs"},
    {"r2://", "httpfs"},
    {"azure://", "azure"},
    {"az://", "azure"},
    {"abfss://", "azure"},
    {"hf://", "httpfs"},  // Hugging Face
};
```

### 文件后缀映射

```cpp
static constexpr ExtensionEntry EXTENSION_FILE_POSTFIXES[] = {
    {".parquet", "parquet"},
    {".json", "json"},
    {".jsonl", "json"},
    {".ndjson", "json"},
    {".shp", "spatial"},
    {".gpkg", "spatial"},
    {".fgb", "spatial"},
    {".xlsx", "excel"},
    {".avro", "avro"},
};
```

### Copy 函数映射

```cpp
static constexpr ExtensionEntry EXTENSION_COPY_FUNCTIONS[] = {
    {"parquet", "parquet"},
    {"json", "json"},
    {"avro", "avro"},
};
```

### Secret 类型映射

```cpp
static constexpr ExtensionEntry EXTENSION_SECRET_TYPES[] = {
    {"aws", "httpfs"},
    {"azure", "azure"},
    {"ducklake", "ducklake"},
    {"gcs", "httpfs"},
    {"huggingface", "httpfs"},
    {"iceberg", "iceberg"},
    {"mysql", "mysql_scanner"},
    {"postgres", "postgres_scanner"},
    {"r2", "httpfs"},
    {"s3", "httpfs"},
};
```

### Secret Provider 映射

```cpp
static constexpr ExtensionEntry EXTENSION_SECRET_PROVIDERS[] = {
    {"s3/config", "httpfs"},
    {"gcs/config", "httpfs"},
    {"s3/credential_chain", "aws"},
    {"azure/config", "azure"},
    {"azure/credential_chain", "azure"},
    {"mysql/config", "mysql_scanner"},
    {"postgres/config", "postgres_scanner"},
};
```

## 查找函数实现

### FindExtensionInEntries

在名称映射表中查找：

```cpp
template <idx_t N>
static string FindExtensionInEntries(const string &name,
                                      const ExtensionEntry (&entries)[N]) {
    auto lcase = StringUtil::Lower(name);

    auto it = std::find_if(entries, entries + N,
        [&](const ExtensionEntry &element) {
            return element.name == lcase;
        });

    if (it != entries + N) {
        return it->extension;
    }
    return "";  // 未找到
}
```

### FindExtensionInFunctionEntries

在函数映射表中查找，返回所有匹配的扩展：

```cpp
template <size_t N>
static vector<pair<string, CatalogType>>
FindExtensionInFunctionEntries(const string &name,
                                const ExtensionFunctionEntry (&entries)[N]) {
    auto lcase = StringUtil::Lower(name);

    vector<pair<string, CatalogType>> result;
    for (idx_t i = 0; i < N; i++) {
        auto &element = entries[i];
        if (element.name == lcase) {
            result.emplace_back(element.extension, element.type);
        }
    }
    return result;
}
```

### 自动加载触发

```cpp
template <idx_t N>
static void TryAutoloadFromEntry(DatabaseInstance &db, const string &entry,
                                  const ExtensionEntry (&entries)[N]) {
    auto &dbconfig = DBConfig::GetConfig(db);

#ifndef DUCKDB_DISABLE_EXTENSION_LOAD
    if (dbconfig.options.autoload_known_extensions) {
        auto extension_name = ExtensionHelper::FindExtensionInEntries(entry, entries);
        if (ExtensionHelper::CanAutoloadExtension(extension_name)) {
            ExtensionHelper::AutoLoadExtension(db, extension_name);
        }
    }
#endif
}
```

## 别名系统

### 别名定义

```cpp
// extension_alias.cpp
static const ExtensionAlias internal_aliases[] = {
    {"http", "httpfs"},        // http:// 协议
    {"https", "httpfs"},       // https:// 协议
    {"md", "motherduck"},      // MotherDuck 简写
    {"mysql", "mysql_scanner"},
    {"s3", "httpfs"},          // S3 协议
    {"postgres", "postgres_scanner"},
    {"sqlite", "sqlite_scanner"},
    {"sqlite3", "sqlite_scanner"},
    {"uc_catalog", "unity_catalog"},  // 兼容旧名称
    {nullptr, nullptr}
};
```

### 别名解析

```cpp
string ExtensionHelper::ApplyExtensionAlias(const string &extension_name) {
    auto lname = StringUtil::Lower(extension_name);

    for (idx_t index = 0; internal_aliases[index].alias; index++) {
        if (lname == internal_aliases[index].alias) {
            return internal_aliases[index].extension;
        }
    }
    return lname;  // 未找到别名，返回原名
}
```

### 别名查询 API

```cpp
idx_t ExtensionHelper::ExtensionAliasCount() {
    idx_t index;
    for (index = 0; internal_aliases[index].alias != nullptr; index++) {
    }
    return index;
}

ExtensionAlias ExtensionHelper::GetExtensionAlias(idx_t index) {
    D_ASSERT(index < ExtensionAliasCount());
    return internal_aliases[index];
}
```

## 自动加载扩展列表

### AUTOLOADABLE_EXTENSIONS

定义哪些扩展可以被自动加载：

```cpp
static constexpr const char *AUTOLOADABLE_EXTENSIONS[] = {
    "avro",
    "aws",
    "azure",
    "autocomplete",
    "core_functions",
    "delta",
    "ducklake",
    "encodings",
    "excel",
    "fts",
    "httpfs",
    "iceberg",
    "inet",
    "icu",
    "json",
    "motherduck",
    "mysql_scanner",
    "parquet",
    "sqlite_scanner",
    "sqlsmith",
    "postgres_scanner",
    "tpcds",
    "tpch",
    "uc_catalog",
    "ui"
};
```

### 判断是否可自动加载

```cpp
bool ExtensionHelper::CanAutoloadExtension(const string &ext_name) {
#ifdef DUCKDB_DISABLE_EXTENSION_LOAD
    return false;
#endif

    if (ext_name.empty()) {
        return false;
    }

    for (const auto &ext : AUTOLOADABLE_EXTENSIONS) {
        if (ext_name == ext) {
            return true;
        }
    }
    return false;
}
```

## 默认扩展列表

### 定义

```cpp
static const DefaultExtension internal_extensions[] = {
    {"core_functions", "Core function library",
     DUCKDB_EXTENSION_CORE_FUNCTIONS_LINKED},
    {"icu", "Adds support for time zones and collations",
     DUCKDB_EXTENSION_ICU_LINKED},
    {"excel", "Adds support for Excel-like format strings",
     DUCKDB_EXTENSION_EXCEL_LINKED},
    {"parquet", "Adds support for reading and writing parquet files",
     DUCKDB_EXTENSION_PARQUET_LINKED},
    {"tpch", "Adds TPC-H data generation and query support",
     DUCKDB_EXTENSION_TPCH_LINKED},
    {"tpcds", "Adds TPC-DS data generation and query support",
     DUCKDB_EXTENSION_TPCDS_LINKED},
    {"httpfs", "Adds support for reading and writing files over HTTP(S)",
     DUCKDB_EXTENSION_HTTPFS_LINKED},
    {"json", "Adds support for JSON operations",
     DUCKDB_EXTENSION_JSON_LINKED},
    {"jemalloc", "Overwrites system allocator with JEMalloc",
     DUCKDB_EXTENSION_JEMALLOC_LINKED},
    {"autocomplete", "Adds support for autocomplete in the shell",
     DUCKDB_EXTENSION_AUTOCOMPLETE_LINKED},

    // 仅动态加载的扩展
    {"motherduck", "Enables motherduck integration", false},
    {"mysql_scanner", "Adds support for connecting to MySQL", false},
    {"sqlite_scanner", "Adds support for SQLite database files", false},
    {"postgres_scanner", "Adds support for connecting to Postgres", false},
    {"inet", "Adds support for IP-related data types", false},
    {"spatial", "Geospatial extension", false},
    {"aws", "Provides features that depend on AWS SDK", false},
    {"azure", "Adds filesystem abstraction for Azure blob storage", false},
    {"iceberg", "Adds support for Apache Iceberg", false},
    {"vss", "Adds indexing for Vector Similarity Search", false},
    {"delta", "Adds support for Delta Lake", false},
    {"fts", "Adds support for Full-Text Search Indexes", false},
    {"ui", "Adds local UI for DuckDB", false},
    {"ducklake", "Adds support for DuckLake", false},

    {nullptr, nullptr, false}  // 终止标记
};
```

### 查询 API

```cpp
idx_t ExtensionHelper::DefaultExtensionCount() {
    idx_t index;
    for (index = 0; internal_extensions[index].name != nullptr; index++) {
    }
    return index;
}

DefaultExtension ExtensionHelper::GetDefaultExtension(idx_t index) {
    D_ASSERT(index < DefaultExtensionCount());
    return internal_extensions[index];
}
```

## 函数映射表生成

### 自动生成流程

函数映射表由脚本自动生成：

```bash
# 1. 启用生成标志构建
GENERATE_EXTENSION_ENTRIES=1 make debug

# 2. 运行生成脚本
python3 scripts/generate_extensions_function.py \
    --extensions icu \
    --shell build/debug/duckdb \
    --extension_repository build/debug/repository
```

生成脚本会：
1. 加载指定扩展
2. 查询所有注册的函数、类型、设置
3. 输出到 `extension_entries.hpp`

### 映射表用途

```
                    用户查询: SELECT json_array(1,2,3)
                                    │
                                    ▼
                            Catalog 查找函数
                                    │
                        ┌───────────┴───────────┐
                        ▼                       ▼
                    已加载                   未找到
                        │                       │
                        ▼                       ▼
                    执行                  EXTENSION_FUNCTIONS 查找
                                               │
                                               ▼
                                    {"json_array", "json", ...}
                                               │
                                               ▼
                                    CanAutoloadExtension("json")
                                               │
                                               ▼ true
                                    AutoLoadExtension("json")
                                               │
                                               ▼
                                    重新执行查询
```

## 应用场景

### 设置自动加载

```cpp
// database.cpp
void DatabaseInstance::SetExtensionSettings(...) {
    for (auto &option : options) {
        auto &name = option.first;
        auto &value = option.second;

        // 查找设置所属扩展
        auto extension_name = ExtensionHelper::FindExtensionInEntries(
            name, EXTENSION_SETTINGS);

        if (extension_name.empty()) {
            continue;
        }

        // 尝试自动加载扩展
        if (!ExtensionHelper::TryAutoLoadExtension(*this, extension_name)) {
            throw InvalidInputException(
                "To set the %s setting, the %s extension needs to be loaded",
                name, extension_name);
        }
    }
}
```

### Catalog 函数查找

```cpp
// catalog.cpp
void Catalog::LookupFunction(...) {
    // 先在已加载的 catalog 中查找
    auto entry = catalog.GetEntry(...);
    if (entry) {
        return entry;
    }

    // 未找到，尝试自动加载
    auto extensions = ExtensionHelper::FindExtensionInFunctionEntries(
        name, EXTENSION_FUNCTIONS);

    for (auto &[ext_name, type] : extensions) {
        if (ExtensionHelper::CanAutoloadExtension(ext_name)) {
            ExtensionHelper::AutoLoadExtension(db, ext_name);
            // 重新查找
            entry = catalog.GetEntry(...);
            if (entry) {
                return entry;
            }
        }
    }

    // 仍未找到，抛出错误
    throw CatalogException("Function %s not found", name);
}
```

### Secret 管理器集成

```cpp
// secret_manager.cpp
void SecretManager::CreateSecret(...) {
    // 查找 secret 类型所属扩展
    auto extension_name = ExtensionHelper::FindExtensionInEntries(
        type, EXTENSION_SECRET_TYPES);

    if (!extension_name.empty()) {
        ExtensionHelper::TryAutoloadFromEntry(db, type, EXTENSION_SECRET_TYPES);
    }
}
```

## 映射表维度总结

| 映射表 | 用途 | 触发场景 |
|-------|------|---------|
| EXTENSION_FUNCTIONS | 函数查找 | 调用未知函数 |
| EXTENSION_FUNCTION_OVERLOADS | 函数重载 | 特定签名匹配 |
| EXTENSION_SETTINGS | 设置查找 | SET 语句 |
| EXTENSION_TYPES | 类型查找 | 使用未知类型 |
| EXTENSION_COLLATIONS | 排序规则 | 字符串排序 |
| EXTENSION_COPY_FUNCTIONS | Copy 函数 | COPY 语句 |
| EXTENSION_SECRET_TYPES | Secret 类型 | 创建 Secret |
| EXTENSION_SECRET_PROVIDERS | Provider 查找 | Secret Provider |
| EXTENSION_FILE_PREFIXES | 文件协议 | 访问 URL |
| EXTENSION_FILE_POSTFIXES | 文件后缀 | 读取文件 |
| EXTENSION_FILE_CONTAINS | 文件包含 | URL 参数 |
| AUTOLOADABLE_EXTENSIONS | 可自动加载 | 自动加载判断 |

## 小结

本章分析了 DuckDB 扩展的入口点定义和函数映射机制：

1. **入口点规范**：C++ 扩展使用 `*_duckdb_cpp_init`，C 扩展使用 `*_init_c_api`
2. **静态映射表**：编译时生成，覆盖函数、设置、类型、排序规则等多个维度
3. **查找函数**：`FindExtensionInEntries` 和 `FindExtensionInFunctionEntries` 提供高效查找
4. **别名系统**：支持扩展名称简写和兼容旧名称
5. **自动加载**：基于映射表实现按需加载扩展

这套机制确保了用户可以透明地使用扩展功能，无需手动管理扩展加载。

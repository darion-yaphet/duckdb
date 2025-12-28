# 第九章：Pragma 与系统函数

## 9.1 概述

DuckDB 提供了丰富的 Pragma 命令和系统函数，用于数据库配置、元数据查询、调试诊断等系统级操作。本章深入分析 PragmaFunction 的设计、系统表函数的实现，以及 CopyFunction 的数据导入导出机制。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      系统函数层次结构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐    ┌─────────────────┐    ┌──────────────────┐   │
│   │ Pragma      │    │ System Table    │    │ Copy             │   │
│   │ Functions   │    │ Functions       │    │ Functions        │   │
│   └──────┬──────┘    └────────┬────────┘    └────────┬─────────┘   │
│          │                    │                      │              │
│   ┌──────┴──────┐     ┌───────┴────────┐    ┌───────┴────────┐     │
│   │ Statement   │     │ duckdb_tables  │    │ CSV Copy       │     │
│   │ (无参数)    │     │ duckdb_columns │    │ Parquet Copy   │     │
│   └─────────────┘     │ duckdb_settings│    │ JSON Copy      │     │
│   ┌─────────────┐     │ pragma_*       │    └────────────────┘     │
│   │ Call        │     └────────────────┘                           │
│   │ (带参数)    │                                                  │
│   └─────────────┘                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 9.2 PragmaFunction 设计

### 9.2.1 核心结构定义

PragmaFunction 继承自 `SimpleNamedParameterFunction`，支持带参数和不带参数两种调用形式：

```cpp
// src/include/duckdb/function/pragma_function.hpp

//! Return a substitute query to execute instead of this pragma statement
typedef string (*pragma_query_t)(ClientContext &context, const FunctionParameters &parameters);
//! Execute the main pragma function
typedef void (*pragma_function_t)(ClientContext &context, const FunctionParameters &parameters);

class PragmaFunction : public SimpleNamedParameterFunction {
public:
    // Call - 带参数的 PRAGMA 调用
    static PragmaFunction PragmaCall(const string &name, pragma_query_t query,
                                     vector<LogicalType> arguments,
                                     LogicalType varargs = LogicalType::INVALID);
    static PragmaFunction PragmaCall(const string &name, pragma_function_t function,
                                     vector<LogicalType> arguments,
                                     LogicalType varargs = LogicalType::INVALID);

    // Statement - 无参数的 PRAGMA 语句
    static PragmaFunction PragmaStatement(const string &name, pragma_query_t query);
    static PragmaFunction PragmaStatement(const string &name, pragma_function_t function);

public:
    PragmaType type;                           // Call 或 Statement
    pragma_query_t query;                      // 返回查询字符串
    pragma_function_t function;                // 直接执行函数
    named_parameter_type_map_t named_parameters;
};
```

### 9.2.2 Pragma 类型分类

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pragma 分类矩阵                               │
├─────────────────┬────────────────────┬──────────────────────────┤
│     维度        │      Query 类型     │     Function 类型        │
├─────────────────┼────────────────────┼──────────────────────────┤
│ Statement       │ PRAGMA show_tables │ PRAGMA enable_profiling  │
│ (无参数)        │ → SELECT * FROM... │ → 修改配置状态           │
├─────────────────┼────────────────────┼──────────────────────────┤
│ Call            │ PRAGMA table_info  │ (较少使用)               │
│ (带参数)        │ ('tbl')            │                          │
│                 │ → SELECT * FROM    │                          │
│                 │   pragma_table_info│                          │
└─────────────────┴────────────────────┴──────────────────────────┘
```

**Query 类型**：返回替代 SQL 查询字符串，由执行器执行
**Function 类型**：直接执行操作，通常用于修改配置

## 9.3 执行型 Pragma 实现

执行型 Pragma 直接修改客户端或数据库配置，不返回查询结果：

```cpp
// src/function/pragma/pragma_functions.cpp

static void PragmaEnableProfilingStatement(ClientContext &context,
                                           const FunctionParameters &parameters) {
    auto &config = ClientConfig::GetConfig(context);
    config.enable_profiler = true;
    config.emit_profiler_output = true;
}

static void PragmaDisableProfiling(ClientContext &context,
                                   const FunctionParameters &parameters) {
    auto &config = ClientConfig::GetConfig(context);
    config.enable_profiler = false;
}

static void PragmaEnableOptimizer(ClientContext &context,
                                  const FunctionParameters &parameters) {
    ClientConfig::GetConfig(context).enable_optimizer = true;
}

static void PragmaDisableOptimizer(ClientContext &context,
                                   const FunctionParameters &parameters) {
    ClientConfig::GetConfig(context).enable_optimizer = false;
}
```

### 9.3.1 配置型 Pragma 注册

```cpp
void PragmaFunctions::RegisterFunction(BuiltinFunctions &set) {
    RegisterEnableProfiling(set);

    // 分析器控制
    set.AddFunction(PragmaFunction::PragmaStatement("disable_profile", PragmaDisableProfiling));
    set.AddFunction(PragmaFunction::PragmaStatement("disable_profiling", PragmaDisableProfiling));

    // 验证模式
    set.AddFunction(PragmaFunction::PragmaStatement("enable_verification", PragmaEnableVerification));
    set.AddFunction(PragmaFunction::PragmaStatement("disable_verification", PragmaDisableVerification));

    // 优化器控制
    set.AddFunction(PragmaFunction::PragmaStatement("enable_optimizer", PragmaEnableOptimizer));
    set.AddFunction(PragmaFunction::PragmaStatement("disable_optimizer", PragmaDisableOptimizer));

    // 进度条控制
    set.AddFunction(PragmaFunction::PragmaStatement("enable_progress_bar", PragmaEnableProgressBar));
    set.AddFunction(PragmaFunction::PragmaStatement("disable_progress_bar", PragmaDisableProgressBar));

    // 检查点控制
    set.AddFunction(PragmaFunction::PragmaStatement("force_checkpoint", PragmaForceCheckpoint));
    set.AddFunction(PragmaFunction::PragmaStatement("enable_checkpoint_on_shutdown",
                                                     PragmaEnableCheckpointOnShutdown));
}
```

## 9.4 查询型 Pragma 实现

查询型 Pragma 返回 SQL 查询字符串，系统会执行该查询并返回结果：

```cpp
// src/function/pragma/pragma_queries.cpp

static string PragmaTableInfo(ClientContext &context, const FunctionParameters &parameters) {
    return StringUtil::Format("SELECT * FROM pragma_table_info(%s);",
                              KeywordHelper::WriteQuoted(parameters.values[0].ToString(), '\''));
}

static string PragmaShowTables(ClientContext &context, const FunctionParameters &parameters) {
    return PragmaShowTables();  // 返回复杂的 CTE 查询
}

string PragmaShowTables(const string &database, const string &schema) {
    string where_clause = "";
    vector<string> where_conditions;
    if (!database.empty()) {
        where_conditions.push_back(
            StringUtil::Format("lower(database_name) = lower(%s)", SQLString(database)));
    }
    if (where_conditions.empty()) {
        where_conditions.push_back("in_search_path(database_name, schema_name)");
    }
    where_clause = "WHERE " + StringUtil::Join(where_conditions, " AND ");

    // 使用 CTE 合并表和视图
    string query = R"EOF(
    with "tables" as (
        SELECT table_name as "name" FROM duckdb_tables
        )EOF" + where_clause + R"EOF(
    ), "views" as (
        SELECT view_name as "name" FROM duckdb_views
        )EOF" + where_clause + R"EOF(
    ), db_objects as (
        SELECT "name" FROM "tables"
        UNION ALL
        SELECT "name" FROM "views"
    )
    SELECT "name" FROM db_objects ORDER BY "name";)EOF";

    return query;
}

static string PragmaVersion(ClientContext &context, const FunctionParameters &parameters) {
    return "SELECT * FROM pragma_version();";
}

static string PragmaDatabaseSize(ClientContext &context, const FunctionParameters &parameters) {
    return "SELECT * FROM pragma_database_size();";
}
```

### 9.4.1 Pragma 查询注册

```cpp
void PragmaQueries::RegisterFunction(BuiltinFunctions &set) {
    // 表信息查询
    set.AddFunction(PragmaFunction::PragmaCall("table_info", PragmaTableInfo,
                                                {LogicalType::VARCHAR}));
    set.AddFunction(PragmaFunction::PragmaCall("storage_info", PragmaStorageInfo,
                                                {LogicalType::VARCHAR}));
    set.AddFunction(PragmaFunction::PragmaCall("show", PragmaShow,
                                                {LogicalType::VARCHAR}));

    // 元数据查询
    set.AddFunction(PragmaFunction::PragmaStatement("show_tables", PragmaShowTables));
    set.AddFunction(PragmaFunction::PragmaStatement("show_databases", PragmaShowDatabases));
    set.AddFunction(PragmaFunction::PragmaStatement("database_list", PragmaDatabaseList));
    set.AddFunction(PragmaFunction::PragmaStatement("functions", PragmaFunctionsQuery));

    // 版本信息
    set.AddFunction(PragmaFunction::PragmaStatement("version", PragmaVersion));
    set.AddFunction(PragmaFunction::PragmaStatement("platform", PragmaPlatform));

    // 数据库操作
    set.AddFunction(PragmaFunction::PragmaCall("import_database", PragmaImportDatabase,
                                                {LogicalType::VARCHAR}));
    set.AddFunction(PragmaFunction::PragmaCall("copy_database", PragmaCopyDatabase,
                                                {LogicalType::VARCHAR, LogicalType::VARCHAR}));
}
```

## 9.5 系统表函数

系统表函数提供元数据查询能力，作为 TableFunction 实现，支持完整的表函数特性。

### 9.5.1 pragma_table_info 实现

```cpp
// src/function/table/system/pragma_table_info.cpp

struct PragmaTableFunctionData : public TableFunctionData {
    explicit PragmaTableFunctionData(CatalogEntry &entry_p, bool is_table_info)
        : entry(entry_p), is_table_info(is_table_info) {
    }
    CatalogEntry &entry;
    bool is_table_info;
};

struct PragmaTableOperatorData : public GlobalTableFunctionState {
    PragmaTableOperatorData() : offset(0) {}
    idx_t offset;
};

// Schema 定义
struct PragmaTableInfoHelper {
    static void GetSchema(vector<LogicalType> &return_types, vector<string> &names) {
        names.emplace_back("cid");       return_types.emplace_back(LogicalType::INTEGER);
        names.emplace_back("name");      return_types.emplace_back(LogicalType::VARCHAR);
        names.emplace_back("type");      return_types.emplace_back(LogicalType::VARCHAR);
        names.emplace_back("notnull");   return_types.emplace_back(LogicalType::BOOLEAN);
        names.emplace_back("dflt_value");return_types.emplace_back(LogicalType::VARCHAR);
        names.emplace_back("pk");        return_types.emplace_back(LogicalType::BOOLEAN);
    }
};

// 绑定：解析表名，获取目录条目
template <bool IS_PRAGMA_TABLE_INFO>
static unique_ptr<FunctionData> PragmaTableInfoBind(ClientContext &context,
                                                     TableFunctionBindInput &input,
                                                     vector<LogicalType> &return_types,
                                                     vector<string> &names) {
    if (IS_PRAGMA_TABLE_INFO) {
        PragmaTableInfoHelper::GetSchema(return_types, names);
    } else {
        PragmaShowHelper::GetSchema(return_types, names);
    }

    auto qname = QualifiedName::Parse(input.inputs[0].GetValue<string>());
    Binder::BindSchemaOrCatalog(context, qname.catalog, qname.schema);
    auto &entry = Catalog::GetEntry(context, CatalogType::TABLE_ENTRY,
                                    qname.catalog, qname.schema, qname.name);
    return make_uniq<PragmaTableFunctionData>(entry, IS_PRAGMA_TABLE_INFO);
}

// 执行：返回列信息
static void PragmaTableInfoFunction(ClientContext &context, TableFunctionInput &data_p,
                                     DataChunk &output) {
    auto &bind_data = data_p.bind_data->Cast<PragmaTableFunctionData>();
    auto &state = data_p.global_state->Cast<PragmaTableOperatorData>();

    switch (bind_data.entry.type) {
    case CatalogType::TABLE_ENTRY:
        PragmaTableInfoTable(state, bind_data.entry.Cast<TableCatalogEntry>(),
                            output, bind_data.is_table_info);
        break;
    case CatalogType::VIEW_ENTRY:
        PragmaTableInfoView(state, bind_data.entry.Cast<ViewCatalogEntry>(),
                           output, bind_data.is_table_info);
        break;
    }
}
```

### 9.5.2 pragma_version 实现

```cpp
// src/function/table/version/pragma_version.cpp

struct PragmaVersionData : public GlobalTableFunctionState {
    PragmaVersionData() : finished(false) {}
    bool finished;
};

static unique_ptr<FunctionData> PragmaVersionBind(ClientContext &context,
                                                   TableFunctionBindInput &input,
                                                   vector<LogicalType> &return_types,
                                                   vector<string> &names) {
    names.emplace_back("library_version");  return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("source_id");        return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("codename");         return_types.emplace_back(LogicalType::VARCHAR);
    return nullptr;
}

static void PragmaVersionFunction(ClientContext &context, TableFunctionInput &data_p,
                                   DataChunk &output) {
    auto &data = data_p.global_state->Cast<PragmaVersionData>();
    if (data.finished) {
        return;
    }
    output.SetCardinality(1);
    output.SetValue(0, 0, DuckDB::LibraryVersion());  // "v1.2.0"
    output.SetValue(1, 0, DuckDB::SourceID());        // Git commit hash
    output.SetValue(2, 0, DuckDB::ReleaseCodename()); // "Histrionicus"
    data.finished = true;
}

// 版本代号映射
const char *DuckDB::ReleaseCodename() {
    if (StringUtil::Contains(DUCKDB_VERSION, "-dev")) {
        return "Development Version";
    }
    if (StringUtil::StartsWith(DUCKDB_VERSION, "v1.2.")) return "Histrionicus";
    if (StringUtil::StartsWith(DUCKDB_VERSION, "v1.3.")) return "Ossivalis";
    if (StringUtil::StartsWith(DUCKDB_VERSION, "v1.4.")) return "Andium";
    if (StringUtil::StartsWith(DUCKDB_VERSION, "v1.5.")) return "Variegata";
    return "Unknown Version";
}
```

## 9.6 duckdb_* 系统表函数

### 9.6.1 duckdb_tables 实现

```cpp
// src/function/table/system/duckdb_tables.cpp

struct DuckDBTablesData : public GlobalTableFunctionState {
    DuckDBTablesData() : offset(0) {}
    vector<reference<CatalogEntry>> entries;
    idx_t offset;
};

static unique_ptr<FunctionData> DuckDBTablesBind(ClientContext &context,
                                                  TableFunctionBindInput &input,
                                                  vector<LogicalType> &return_types,
                                                  vector<string> &names) {
    // 定义返回列
    names.emplace_back("database_name");  return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("database_oid");   return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("schema_name");    return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("schema_oid");     return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("table_name");     return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("table_oid");      return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("comment");        return_types.emplace_back(LogicalType::VARCHAR);
    names.emplace_back("tags");
    return_types.emplace_back(LogicalType::MAP(LogicalType::VARCHAR, LogicalType::VARCHAR));
    names.emplace_back("internal");       return_types.emplace_back(LogicalType::BOOLEAN);
    names.emplace_back("temporary");      return_types.emplace_back(LogicalType::BOOLEAN);
    names.emplace_back("has_primary_key");return_types.emplace_back(LogicalType::BOOLEAN);
    names.emplace_back("estimated_size"); return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("column_count");   return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("index_count");    return_types.emplace_back(LogicalType::BIGINT);
    names.emplace_back("sql");            return_types.emplace_back(LogicalType::VARCHAR);
    return nullptr;
}

unique_ptr<GlobalTableFunctionState> DuckDBTablesInit(ClientContext &context,
                                                       TableFunctionInitInput &input) {
    auto result = make_uniq<DuckDBTablesData>();
    // 扫描所有 schema 的表
    auto schemas = Catalog::GetAllSchemas(context);
    for (auto &schema : schemas) {
        schema.get().Scan(context, CatalogType::TABLE_ENTRY,
                         [&](CatalogEntry &entry) { result->entries.push_back(entry); });
    }
    return std::move(result);
}

void DuckDBTablesFunction(ClientContext &context, TableFunctionInput &data_p, DataChunk &output) {
    auto &data = data_p.global_state->Cast<DuckDBTablesData>();
    if (data.offset >= data.entries.size()) {
        return;
    }

    idx_t count = 0;
    while (data.offset < data.entries.size() && count < STANDARD_VECTOR_SIZE) {
        auto &entry = data.entries[data.offset++].get();
        if (entry.type != CatalogType::TABLE_ENTRY) continue;

        auto &table = entry.Cast<TableCatalogEntry>();
        auto storage_info = table.GetStorageInfo(context);

        idx_t col = 0;
        output.SetValue(col++, count, table.catalog.GetName());
        output.SetValue(col++, count, Value::BIGINT(table.catalog.GetOid()));
        output.SetValue(col++, count, Value(table.schema.name));
        output.SetValue(col++, count, Value::BIGINT(table.schema.oid));
        output.SetValue(col++, count, Value(table.name));
        output.SetValue(col++, count, Value::BIGINT(table.oid));
        output.SetValue(col++, count, Value(table.comment));
        output.SetValue(col++, count, Value::MAP(table.tags));
        output.SetValue(col++, count, Value::BOOLEAN(table.internal));
        output.SetValue(col++, count, Value::BOOLEAN(table.temporary));
        output.SetValue(col++, count, Value::BOOLEAN(table.HasPrimaryKey()));
        // ... 更多列
        count++;
    }
    output.SetCardinality(count);
}
```

### 9.6.2 duckdb_settings 实现

```cpp
// src/function/table/system/duckdb_settings.cpp

struct DuckDBSettingValue {
    string name;
    Value value;
    string description;
    string input_type;
    string scope;
    vector<Value> aliases;
};

unique_ptr<GlobalTableFunctionState> DuckDBSettingsInit(ClientContext &context,
                                                         TableFunctionInitInput &input) {
    auto result = make_uniq<DuckDBSettingsData>();

    // 收集别名映射
    unordered_map<idx_t, vector<Value>> aliases;
    for (idx_t i = 0; i < DBConfig::GetAliasCount(); i++) {
        auto alias = DBConfig::GetAliasByIndex(i);
        aliases[alias->option_index].emplace_back(alias->alias);
    }

    // 收集内置配置选项
    auto &config = DBConfig::GetConfig(context);
    auto options_count = DBConfig::GetOptionCount();
    for (idx_t i = 0; i < options_count; i++) {
        auto option = DBConfig::GetOptionByIndex(i);
        DuckDBSettingValue value;
        value.name = option->name;
        value.value = option->get_setting ? option->get_setting(context)
                                          : option->default_value;
        value.description = option->description;
        value.input_type = option->parameter_type;
        value.scope = EnumUtil::ToString(
            option->set_global ? SettingScope::GLOBAL : SettingScope::LOCAL);
        result->settings.push_back(std::move(value));
    }

    // 收集扩展配置选项
    for (auto &ext_param : config.extension_parameters) {
        Value setting_val;
        context.TryGetCurrentSetting(ext_param.first, setting_val);
        DuckDBSettingValue value;
        value.name = ext_param.first;
        value.value = std::move(setting_val);
        value.description = ext_param.second.description;
        value.input_type = ext_param.second.type.ToString();
        result->settings.push_back(std::move(value));
    }

    std::sort(result->settings.begin(), result->settings.end());
    return std::move(result);
}
```

## 9.7 CopyFunction 数据导入导出

CopyFunction 是 DuckDB 的数据导入导出框架，支持 CSV、Parquet、JSON 等多种格式。

### 9.7.1 CopyFunction 核心结构

```cpp
// src/include/duckdb/function/copy_function.hpp

class CopyFunction : public Function {
public:
    explicit CopyFunction(const string &name);

    // 写入回调函数
    copy_to_plan_t plan;                      // 计划重写
    copy_to_select_t copy_to_select;          // 选择列表处理
    copy_to_bind_t copy_to_bind;              // 绑定
    copy_options_t copy_options;              // 选项定义
    copy_to_initialize_local_t copy_to_initialize_local;   // 本地状态初始化
    copy_to_initialize_global_t copy_to_initialize_global; // 全局状态初始化
    copy_to_sink_t copy_to_sink;              // 数据接收
    copy_to_combine_t copy_to_combine;        // 状态合并
    copy_to_finalize_t copy_to_finalize;      // 完成处理

    // 批量写入支持
    copy_prepare_batch_t prepare_batch;
    copy_flush_batch_t flush_batch;
    copy_desired_batch_size_t desired_batch_size;

    // 文件轮转支持
    copy_rotate_files_t rotate_files;
    copy_rotate_next_file_t rotate_next_file;

    // 读取回调函数
    copy_from_bind_t copy_from_bind;
    TableFunction copy_from_function;

    // 序列化支持
    copy_to_serialize_t serialize;
    copy_to_deserialize_t deserialize;

    string extension;  // 文件扩展名
    shared_ptr<CopyFunctionInfo> function_info;
};
```

### 9.7.2 CopyFunction 执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      COPY TO 执行流程                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  COPY table TO 'file.csv' (FORMAT CSV, HEADER)                         │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────┐                                                    │
│  │ Bind Options    │  解析 FORMAT, HEADER 等选项                        │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ copy_to_bind    │  绑定写入参数，创建 FunctionData                   │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ initialize_     │  创建文件句柄，初始化写入器                        │
│  │ global          │                                                    │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ initialize_     │  每个线程创建本地缓冲区                            │
│  │ local           │                                                    │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ copy_to_sink    │  接收 DataChunk，写入缓冲区                        │
│  │ (per chunk)     │                                                    │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ copy_to_combine │  合并各线程的本地状态                              │
│  └────────┬────────┘                                                    │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ copy_to_        │  刷新缓冲区，关闭文件                              │
│  │ finalize        │                                                    │
│  └─────────────────┘                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.7.3 COPY 绑定实现

```cpp
// src/planner/binder/statement/bind_copy.cpp

BoundStatement Binder::BindCopyTo(CopyStatement &stmt, const CopyFunction &function,
                                   CopyToType copy_to_type) {
    if (function.plan) {
        // 计划重写
        return function.plan(*this, stmt);
    }

    auto &copy_info = *stmt.info;
    auto select_node = Bind(*copy_info.select_statement->Copy());

    // 处理通用选项
    bool use_tmp_file = true;
    CopyOverwriteMode overwrite_mode = CopyOverwriteMode::COPY_ERROR_ON_CONFLICT;
    FilenamePattern filename_pattern;
    bool per_thread_output = false;
    optional_idx file_size_bytes;
    vector<idx_t> partition_cols;

    for (auto &option : original_options) {
        auto loption = StringUtil::Lower(option.first);
        if (loption == "use_tmp_file") {
            use_tmp_file = GetBooleanArg(context, option.second);
        } else if (loption == "overwrite" || loption == "append") {
            // 处理覆写模式
        } else if (loption == "per_thread_output") {
            per_thread_output = GetBooleanArg(context, option.second);
        } else if (loption == "partition_by") {
            partition_cols = ParseColumnsOrdered(converted, select_node.names, loption);
        }
        // ... 更多选项处理
    }

    // 调用格式特定的绑定函数
    auto function_data = function.copy_to_bind(context, bind_input,
                                               names_to_write, types_to_write);

    // 创建 LogicalCopyToFile 算子
    auto copy = make_uniq<LogicalCopyToFile>(function, std::move(function_data),
                                              std::move(stmt.info));
    copy->file_path = file_path;
    copy->use_tmp_file = use_tmp_file;
    copy->overwrite_mode = overwrite_mode;
    copy->per_thread_output = per_thread_output;
    copy->partition_columns = std::move(partition_cols);

    copy->AddChild(std::move(select_node.plan));

    BoundStatement result;
    result.names = GetCopyFunctionReturnNames(copy->return_type);
    result.types = GetCopyFunctionReturnLogicalTypes(copy->return_type);
    result.plan = std::move(copy);
    return result;
}

BoundStatement Binder::BindCopyFrom(CopyStatement &stmt, const CopyFunction &function) {
    // 生成 INSERT INTO table SELECT * FROM copy_function(file)
    InsertStatement insert;
    insert.table = stmt.info->table;
    insert.schema = stmt.info->schema;
    insert.columns = stmt.info->select_list;

    auto insert_statement = Bind(insert);
    auto &bound_insert = insert_statement.plan->Cast<LogicalInsert>();

    // 调用格式特定的绑定
    auto function_data = function.copy_from_bind(context, input,
                                                  expected_names, bound_insert.expected_types);

    // 创建 LogicalGet 从文件读取
    auto get = make_uniq<LogicalGet>(GenerateTableIndex(), std::move(copy_from_function),
                                      std::move(function_data), bound_insert.expected_types,
                                      expected_names);
    insert_statement.plan->children.push_back(std::move(get));

    result.plan = std::move(insert_statement.plan);
    return result;
}
```

### 9.7.4 CopyFunction 选项验证

```cpp
case_insensitive_map_t<CopyOption> Binder::GetFullCopyOptionsList(const CopyFunction &function,
                                                                   CopyOptionMode mode) {
    case_insensitive_map_t<CopyOption> copy_options;
    CopyOptionsInput input(copy_options);
    function.copy_options(context, input);  // 格式特定选项

    // 添加通用写入选项
    if (mode != CopyOptionMode::READ_ONLY) {
        copy_options["use_tmp_file"] = CopyOption(LogicalType::BOOLEAN, CopyOptionMode::WRITE_ONLY);
        copy_options["overwrite"] = CopyOption(LogicalType::BOOLEAN, CopyOptionMode::WRITE_ONLY);
        copy_options["append"] = CopyOption(LogicalType::BOOLEAN, CopyOptionMode::WRITE_ONLY);
        copy_options["filename_pattern"] = CopyOption(LogicalType::VARCHAR, CopyOptionMode::WRITE_ONLY);
        copy_options["per_thread_output"] = CopyOption(LogicalType::BOOLEAN, CopyOptionMode::WRITE_ONLY);
        copy_options["file_size_bytes"] = CopyOption(LogicalType::ANY, CopyOptionMode::WRITE_ONLY);
        copy_options["partition_by"] = CopyOption(LogicalType::ANY, CopyOptionMode::WRITE_ONLY);
        copy_options["return_files"] = CopyOption(LogicalType::BOOLEAN, CopyOptionMode::WRITE_ONLY);
    }
    return copy_options;
}
```

## 9.8 系统函数分类

### 9.8.1 系统表函数一览

| 函数名 | 用途 | 返回内容 |
|--------|------|----------|
| `duckdb_tables()` | 表元数据 | 表名、OID、列数、索引数 |
| `duckdb_columns()` | 列元数据 | 列名、类型、约束、默认值 |
| `duckdb_views()` | 视图元数据 | 视图名、SQL 定义 |
| `duckdb_indexes()` | 索引元数据 | 索引名、类型、唯一性 |
| `duckdb_schemas()` | Schema 元数据 | Schema 名、所属数据库 |
| `duckdb_databases()` | 数据库列表 | 数据库名、路径、只读标志 |
| `duckdb_functions()` | 函数列表 | 函数名、类型、参数 |
| `duckdb_settings()` | 配置项列表 | 配置名、当前值、描述 |
| `duckdb_extensions()` | 扩展列表 | 扩展名、版本、加载状态 |
| `duckdb_types()` | 类型列表 | 类型名、类别、内部表示 |
| `duckdb_constraints()` | 约束列表 | 约束类型、关联表/列 |
| `duckdb_dependencies()` | 依赖关系 | 依赖类型、源/目标对象 |
| `duckdb_memory()` | 内存使用 | 分配器、已用/可用内存 |
| `duckdb_temporary_files()` | 临时文件 | 文件路径、大小 |

### 9.8.2 Pragma 表函数一览

| 函数名 | 用途 | 参数 |
|--------|------|------|
| `pragma_table_info(name)` | 表结构信息 | 表名 |
| `pragma_show(name)` | 表结构（MySQL 风格） | 表名 |
| `pragma_storage_info(name)` | 存储信息 | 表名 |
| `pragma_database_size()` | 数据库大小 | 无 |
| `pragma_version()` | 版本信息 | 无 |
| `pragma_platform()` | 平台信息 | 无 |
| `pragma_collations()` | 排序规则列表 | 无 |
| `pragma_metadata_info()` | 元数据信息 | 无 |
| `pragma_user_agent()` | 用户代理 | 无 |

## 9.9 实现模式总结

### 9.9.1 Pragma vs 表函数

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pragma 实现策略                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRAGMA table_info('tbl')                                       │
│       │                                                         │
│       │ pragma_query_t                                          │
│       ▼                                                         │
│  "SELECT * FROM pragma_table_info('tbl')"                       │
│       │                                                         │
│       │ 执行器                                                  │
│       ▼                                                         │
│  pragma_table_info (TableFunction)                              │
│       │                                                         │
│       │ bind → init → function                                  │
│       ▼                                                         │
│  返回结果集                                                      │
│                                                                 │
│  设计优势：                                                      │
│  - Pragma 提供简洁语法：PRAGMA table_info('tbl')                │
│  - 表函数提供完整能力：JOIN、WHERE、聚合                         │
│  - 两种入口共享同一实现                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.9.2 系统表函数实现模式

```cpp
// 标准实现模式
class SystemTableFunction {
    // 1. 定义状态结构
    struct FunctionData : TableFunctionData { /* 绑定时数据 */ };
    struct FunctionState : GlobalTableFunctionState {
        vector<Entry> entries;
        idx_t offset;
    };

    // 2. Bind: 定义返回列
    static unique_ptr<FunctionData> Bind(ClientContext &context,
                                          TableFunctionBindInput &input,
                                          vector<LogicalType> &return_types,
                                          vector<string> &names);

    // 3. Init: 收集数据
    static unique_ptr<GlobalTableFunctionState> Init(ClientContext &context,
                                                      TableFunctionInitInput &input);

    // 4. Function: 返回数据
    static void Function(ClientContext &context, TableFunctionInput &data_p,
                         DataChunk &output);

    // 5. Register: 注册函数
    static void RegisterFunction(BuiltinFunctions &set) {
        TableFunction func("name", {}, Function);
        func.bind = Bind;
        func.init_global = Init;
        set.AddFunction(func);
    }
};
```

## 9.10 小结

本章详细介绍了 DuckDB 的 Pragma 命令和系统函数实现：

1. **PragmaFunction 双模式设计**：Statement（无参数）和 Call（带参数）两种调用形式，Query（返回 SQL）和 Function（直接执行）两种执行模式

2. **系统表函数层次结构**：
   - `duckdb_*` 系列提供完整元数据查询
   - `pragma_*` 系列提供特定信息查询
   - Pragma 命令通过生成查询字符串复用表函数

3. **CopyFunction 数据通道**：完整的数据导入导出框架，支持多种格式、并行写入、文件分区等高级特性

4. **统一的实现模式**：Bind-Init-Function 三阶段设计，Catalog 扫描与迭代返回的状态管理模式

这些系统级功能为数据库管理、调试诊断、数据迁移提供了强大支持，是 DuckDB 可用性的重要保障。

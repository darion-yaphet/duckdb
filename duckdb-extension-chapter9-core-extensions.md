# 第九章：核心扩展实现分析

## 9.1 概述

DuckDB 的核心扩展展示了扩展系统的最佳实践。通过分析 `core_functions`、`parquet`、`json` 和 `icu` 这四个典型扩展的实现，我们可以深入理解扩展开发的设计模式、代码组织方式和功能注册策略。

```
+-------------------+     +-------------------+     +-------------------+
|  core_functions   |     |      parquet      |     |       json        |
+-------------------+     +-------------------+     +-------------------+
| - 标量函数        |     | - 文件读写        |     | - JSON 解析       |
| - 聚合函数        |     | - 元数据查询      |     | - 路径提取        |
| - 日期函数        |     | - 加密支持        |     | - 类型转换        |
| - 字符串函数      |     | - 压缩编解码      |     | - 表函数          |
+-------------------+     +-------------------+     +-------------------+
         |                        |                        |
         +------------------------+------------------------+
                                  |
                          ExtensionLoader
                                  |
                          DuckDB 核心引擎
```

## 9.2 core_functions 扩展

### 9.2.1 扩展入口实现

`core_functions` 扩展是 DuckDB 内置函数的集合，采用最简洁的扩展模式：

```cpp
// extension/core_functions/core_functions_extension.cpp

static void LoadInternal(ExtensionLoader &loader) {
    // 通过 FunctionList 批量注册函数
    FunctionList::RegisterExtensionFunctions(
        loader,
        CoreFunctionList::GetFunctionList()
    );
}

void CoreFunctionsExtension::Load(ExtensionLoader &loader) {
    LoadInternal(loader);
}

std::string CoreFunctionsExtension::Name() {
    return "core_functions";
}

std::string CoreFunctionsExtension::Version() const {
#ifdef EXT_VERSION_CORE_FUNCTIONS
    return EXT_VERSION_CORE_FUNCTIONS;
#else
    return "";
#endif
}

// C++ 入口宏
extern "C" {
DUCKDB_CPP_EXTENSION_ENTRY(core_functions, loader) {
    duckdb::LoadInternal(loader);
}
}
```

### 9.2.2 函数列表管理

核心函数采用静态定义列表的方式管理所有函数：

```cpp
// extension/core_functions/function_list.cpp

// 宏定义：标量函数注册
#define DUCKDB_SCALAR_FUNCTION_BASE(_PARAM, _NAME, _ALIAS_OF)       \
    { _NAME, _ALIAS_OF, _PARAM::Parameters, _PARAM::Description,   \
      _PARAM::Example, _PARAM::Categories, _PARAM::GetFunction,    \
      nullptr, nullptr, nullptr }

#define DUCKDB_SCALAR_FUNCTION(_PARAM) \
    DUCKDB_SCALAR_FUNCTION_BASE(_PARAM, _PARAM::Name, _PARAM::Name)

// 宏定义：聚合函数注册
#define DUCKDB_AGGREGATE_FUNCTION_BASE(_PARAM, _NAME, _ALIAS_OF)   \
    { _NAME, _ALIAS_OF, _PARAM::Parameters, _PARAM::Description,   \
      _PARAM::Example, _PARAM::Categories, nullptr, nullptr,       \
      _PARAM::GetFunction, nullptr }

// 静态函数列表（由脚本自动生成）
static const StaticFunctionDefinition core_functions[] = {
    DUCKDB_SCALAR_FUNCTION(FactorialOperatorFun),
    DUCKDB_SCALAR_FUNCTION_SET(BitwiseAndFun),
    DUCKDB_SCALAR_FUNCTION(AcosFun),
    DUCKDB_AGGREGATE_FUNCTION(ApproxCountDistinctFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(AvgFun),
    // ... 数百个函数定义
    FINAL_FUNCTION
};
```

### 9.2.3 函数分类结构

core_functions 按功能域组织函数：

```
extension/core_functions/
├── aggregate/                    # 聚合函数
│   ├── algebraic/               # 代数聚合（avg, stddev, corr）
│   ├── distributive/            # 分布式聚合（sum, count, min/max）
│   ├── holistic/                # 整体聚合（quantile, mode）
│   ├── nested/                  # 嵌套聚合（list, histogram）
│   └── regression/              # 回归分析（regr_slope, regr_r2）
├── scalar/                      # 标量函数
│   ├── date/                    # 日期函数
│   ├── string/                  # 字符串函数
│   ├── math/                    # 数学函数
│   ├── list/                    # 列表函数
│   ├── map/                     # 映射函数
│   ├── struct/                  # 结构体函数
│   └── generic/                 # 通用函数
├── function_list.cpp            # 函数列表定义
└── core_functions_extension.cpp # 扩展入口
```

### 9.2.4 函数定义模板

每个函数通过结构体定义其元数据和实现：

```cpp
// 示例：abs 函数定义
struct AbsFun {
    static constexpr const char *Name = "abs";
    static constexpr const char *Parameters = "x";
    static constexpr const char *Description =
        "Returns the absolute value of x";
    static constexpr const char *Example = "abs(-5)";
    static constexpr const char *Categories[] = {"math"};

    static ScalarFunctionSet GetFunctions() {
        ScalarFunctionSet set(Name);
        // 添加不同类型的重载
        set.AddFunction(ScalarFunction({LogicalType::TINYINT},
            LogicalType::TINYINT, AbsFunction<int8_t>));
        set.AddFunction(ScalarFunction({LogicalType::INTEGER},
            LogicalType::INTEGER, AbsFunction<int32_t>));
        set.AddFunction(ScalarFunction({LogicalType::DOUBLE},
            LogicalType::DOUBLE, AbsFunction<double>));
        return set;
    }
};
```

## 9.3 parquet 扩展

### 9.3.1 扩展结构概览

parquet 扩展是功能最丰富的核心扩展之一，提供完整的 Parquet 文件格式支持：

```cpp
// extension/parquet/parquet_extension.cpp

static void LoadInternal(ExtensionLoader &loader) {
    auto &db_instance = loader.GetDatabaseInstance();
    auto &fs = db_instance.GetFileSystem();

    // 1. 注册 ZSTD 文件系统
    fs.RegisterSubSystem(FileCompressionType::ZSTD,
        make_uniq<ZStdFileSystem>());

    // 2. 注册读取函数
    auto scan_fun = ParquetScanFunction::GetFunctionSet();
    scan_fun.name = "read_parquet";
    loader.RegisterFunction(scan_fun);
    scan_fun.name = "parquet_scan";
    loader.RegisterFunction(scan_fun);

    // 3. 注册元数据查询函数
    ParquetMetaDataFunction meta_fun;
    loader.RegisterFunction(
        MultiFileReader::CreateFunctionSet(meta_fun));

    ParquetSchemaFunction schema_fun;
    loader.RegisterFunction(
        MultiFileReader::CreateFunctionSet(schema_fun));

    // 4. 注册 COPY 函数
    CopyFunction function("parquet");
    function.copy_to_bind = ParquetWriteBind;
    function.copy_to_sink = ParquetWriteSink;
    function.copy_from_function = scan_fun.functions[0];
    // ... 配置其他回调
    loader.RegisterFunction(function);

    // 5. 注册替换扫描
    auto &config = DBConfig::GetConfig(db_instance);
    config.replacement_scans.emplace_back(ParquetScanReplacement);

    // 6. 注册扩展选项
    config.AddExtensionOption("binary_as_string",
        "In Parquet files, interpret binary data as a string.",
        LogicalType::BOOLEAN, Value(false));
}
```

### 9.3.2 表函数注册模式

Parquet 扩展注册多种表函数：

```cpp
// 表函数类型及用途
+---------------------------+-----------------------------------+
| 函数名                     | 功能                               |
+---------------------------+-----------------------------------+
| read_parquet / parquet_scan| 读取 Parquet 文件数据              |
| parquet_metadata          | 查询文件元数据                      |
| parquet_schema            | 查询 Schema 结构                   |
| parquet_key_value_metadata| 查询键值元数据                      |
| parquet_file_metadata     | 查询文件级元数据                    |
| parquet_bloom_probe       | Bloom Filter 探测                  |
+---------------------------+-----------------------------------+
```

### 9.3.3 COPY 函数实现

COPY 函数支持 Parquet 格式的导入导出：

```cpp
// 写入绑定数据
struct ParquetWriteBindData : public TableFunctionData {
    vector<LogicalType> sql_types;
    vector<string> column_names;
    duckdb_parquet::CompressionCodec::type codec =
        duckdb_parquet::CompressionCodec::SNAPPY;
    idx_t row_group_size = DEFAULT_ROW_GROUP_SIZE;
    bool enable_bloom_filters = true;
    double bloom_filter_false_positive_ratio = 0.01;
    // ... 其他配置
};

// COPY 选项定义
static void ParquetListCopyOptions(
    ClientContext &context,
    CopyOptionsInput &input
) {
    auto &copy_options = input.options;
    copy_options["row_group_size"] =
        CopyOption(LogicalType::UBIGINT, CopyOptionMode::READ_WRITE);
    copy_options["compression"] =
        CopyOption(LogicalType::VARCHAR, CopyOptionMode::READ_WRITE);
    copy_options["field_ids"] =
        CopyOption(LogicalType::ANY, CopyOptionMode::WRITE_ONLY);
    // ... 更多选项
}
```

### 9.3.4 替换扫描机制

允许直接从文件名查询 Parquet：

```cpp
static unique_ptr<TableRef> ParquetScanReplacement(
    ClientContext &context,
    ReplacementScanInput &input,
    optional_ptr<ReplacementScanData> data
) {
    auto table_name = ReplacementScan::GetFullPath(input);

    // 检查文件扩展名
    if (!ReplacementScan::CanReplace(table_name, {"parquet"})) {
        return nullptr;
    }

    // 创建表函数引用
    auto table_function = make_uniq<TableFunctionRef>();
    vector<unique_ptr<ParsedExpression>> children;
    children.push_back(make_uniq<ConstantExpression>(Value(table_name)));
    table_function->function = make_uniq<FunctionExpression>(
        "parquet_scan", std::move(children));

    return std::move(table_function);
}

// 使用示例：
// SELECT * FROM 'data.parquet';
// 自动转换为: SELECT * FROM parquet_scan('data.parquet');
```

### 9.3.5 读取器架构

Parquet 读取器采用分层设计：

```
ParquetReader
├── FileMetaData          # 文件元数据
├── ThriftFileTransport   # Thrift 传输层
└── ColumnReader          # 列读取器
    ├── StructColumnReader    # 结构体列
    ├── ListColumnReader      # 列表列
    ├── StringColumnReader    # 字符串列
    ├── DecimalColumnReader   # 十进制列
    └── TemplatedColumnReader # 模板化基础类型
```

```cpp
// 读取器初始化
static shared_ptr<ParquetFileMetadataCache> LoadMetadata(
    ClientContext &context,
    Allocator &allocator,
    CachingFileHandle &file_handle,
    const shared_ptr<const ParquetEncryptionConfig> &encryption_config,
    const EncryptionUtil &encryption_util,
    optional_idx footer_size
) {
    auto file_proto = CreateThriftFileProtocol(context, file_handle, false);
    auto &transport = reinterpret_cast<ThriftFileTransport &>(
        *file_proto->getTransport());

    // 验证文件魔数
    if (memcmp(buffer + 4, "PAR1", 4) == 0) {
        footer_encrypted = false;
    } else if (memcmp(buffer + 4, "PARE", 4) == 0) {
        footer_encrypted = true;  // 加密的 Parquet
    }
    // ... 解析元数据
}
```

## 9.4 json 扩展

### 9.4.1 扩展入口实现

json 扩展提供完整的 JSON 处理能力：

```cpp
// extension/json/json_extension.cpp

static void LoadInternal(ExtensionLoader &loader) {
    // 1. 注册 JSON 类型
    auto json_type = LogicalType::JSON();
    loader.RegisterType(LogicalType::JSON_TYPE_NAME, std::move(json_type));

    // 2. 注册类型转换
    JSONFunctions::RegisterSimpleCastFunctions(loader);
    JSONFunctions::RegisterJSONCreateCastFunctions(loader);
    JSONFunctions::RegisterJSONTransformCastFunctions(loader);

    // 3. 注册标量函数
    for (auto &fun : JSONFunctions::GetScalarFunctions()) {
        loader.RegisterFunction(fun);
    }

    // 4. 注册表函数
    for (auto &fun : JSONFunctions::GetTableFunctions()) {
        loader.RegisterFunction(fun);
    }

    // 5. 注册 COPY 函数
    auto copy_fun = JSONFunctions::GetJSONCopyFunction();
    loader.RegisterFunction(copy_fun);
    copy_fun.extension = "ndjson";
    copy_fun.name = "ndjson";
    loader.RegisterFunction(copy_fun);

    // 6. 注册替换扫描
    DBConfig::GetConfig(loader.GetDatabaseInstance())
        .replacement_scans.emplace_back(JSONFunctions::ReadJSONReplacement);

    // 7. 注册 SQL 宏
    for (idx_t index = 0; JSON_MACROS[index].name != nullptr; index++) {
        auto info = DefaultFunctionGenerator::CreateInternalMacroInfo(
            JSON_MACROS[index]);
        loader.RegisterFunction(*info);
    }
}
```

### 9.4.2 函数分类

JSON 扩展提供丰富的函数集：

```cpp
vector<ScalarFunctionSet> JSONFunctions::GetScalarFunctions() {
    vector<ScalarFunctionSet> functions;

    // 提取函数
    AddAliases({"json_extract", "json_extract_path"},
        GetExtractFunction(), functions);
    AddAliases({"json_extract_string", "json_extract_path_text", "->>"},
        GetExtractStringFunction(), functions);

    // 创建函数
    functions.push_back(GetArrayFunction());
    functions.push_back(GetObjectFunction());
    AddAliases({"to_json", "json_quote"},
        GetToJSONFunction(), functions);

    // 结构/转换函数
    functions.push_back(GetStructureFunction());
    AddAliases({"json_transform", "from_json"},
        GetTransformFunction(), functions);

    // 其他函数
    functions.push_back(GetArrayLengthFunction());
    functions.push_back(GetContainsFunction());
    functions.push_back(GetKeysFunction());
    functions.push_back(GetTypeFunction());
    functions.push_back(GetValidFunction());

    return functions;
}
```

### 9.4.3 类型转换注册

JSON 扩展定义了精细的类型转换规则：

```cpp
void JSONFunctions::RegisterSimpleCastFunctions(ExtensionLoader &loader) {
    auto &db = loader.GetDatabaseInstance();

    // JSON -> VARCHAR：基本免费（重解释）
    loader.RegisterCastFunction(
        LogicalType::JSON(),
        LogicalType::VARCHAR,
        DefaultCasts::ReinterpretCast,
        1  // 低成本
    );

    // VARCHAR -> JSON：需要解析验证
    const auto varchar_to_json_cost =
        CastFunctionSet::ImplicitCastCost(
            db, LogicalType::SQLNULL, LogicalTypeId::STRUCT) + 1;

    BoundCastInfo varchar_to_json_info(
        CastVarcharToJSON,        // 转换函数
        nullptr,                  // 绑定数据
        JSONFunctionLocalState::InitCastLocalState  // 本地状态
    );

    loader.RegisterCastFunction(
        LogicalType::VARCHAR,
        LogicalType::JSON(),
        std::move(varchar_to_json_info),
        varchar_to_json_cost
    );

    // JSON[] -> VARCHAR：特殊处理防止引号转义
    loader.RegisterCastFunction(
        LogicalType::LIST(LogicalType::JSON()),
        LogicalTypeId::VARCHAR,
        CastJSONListToVarchar,
        json_list_to_varchar_cost
    );
}
```

### 9.4.4 SQL 宏定义

json 扩展通过 SQL 宏简化常用操作：

```cpp
static const DefaultMacro JSON_MACROS[] = {
    {DEFAULT_SCHEMA,
     "json_group_array",
     {"x", nullptr},
     {{nullptr, nullptr}},
     "CAST('[' || string_agg("
         "CASE WHEN x IS NULL THEN 'null'::JSON ELSE to_json(x) END, ',') "
     "|| ']' AS JSON)"},

    {DEFAULT_SCHEMA,
     "json_group_object",
     {"n", "v", nullptr},
     {{nullptr, nullptr}},
     "CAST('{' || string_agg("
         "to_json(n::VARCHAR) || ':' || "
         "CASE WHEN v IS NULL THEN 'null'::JSON ELSE to_json(v) END, ',') "
     "|| '}' AS JSON)"},

    {DEFAULT_SCHEMA,
     "json_group_structure",
     {"x", nullptr},
     {{nullptr, nullptr}},
     "json_structure(json_group_array(x))->0"},

    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}
};
```

### 9.4.5 路径表达式处理

JSON 扩展支持灵活的路径语法：

```cpp
JSONPathType JSONReadFunctionData::CheckPath(
    const Value &path_val,
    string &path,
    idx_t &len
) {
    // 获取路径字符串
    auto path_str = path_val.GetValueUnsafe<string_t>();
    len = path_str.GetSize();
    const auto ptr = path_str.GetData();
    JSONPathType path_type = JSONPathType::REGULAR;

    if (len != 0) {
        if (*ptr == '/' || *ptr == '$') {
            // JSON Path 或 JSON Pointer 格式
            path = string(ptr, len);
        } else if (path_val.type().IsIntegral()) {
            // 整数索引：转换为数组访问
            path = "$[" + string(ptr, len) + "]";
        } else if (memchr(ptr, '"', len)) {
            // 包含引号：JSON Pointer 格式
            path = "/" + string(ptr, len);
        } else {
            // 简单键名：转换为 JSON Path
            path = "$.\"" + string(ptr, len) + "\"";
        }

        // 验证并检测通配符
        if (*path.c_str() == '$') {
            path_type = JSONCommon::ValidatePath(path.c_str(), len, true);
        }
    }
    return path_type;
}
```

## 9.5 icu 扩展

### 9.5.1 扩展入口实现

icu 扩展提供国际化支持：

```cpp
// extension/icu/icu_extension.cpp

static void LoadInternal(ExtensionLoader &loader) {
    // 1. 注册所有 ICU 支持的排序规则
    int32_t count;
    auto locales = icu::Collator::getAvailableLocales(count);
    for (int32_t i = 0; i < count; i++) {
        string collation;
        const auto &locale = locales[i];

        if (string(locale.getCountry()).empty()) {
            collation = locale.getLanguage();
        } else {
            collation = locale.getLanguage() +
                string("_") + locale.getCountry();
        }
        collation = StringUtil::Lower(collation);

        CreateCollationInfo info(collation,
            GetICUCollateFunction(collation, ""), false, false);
        loader.RegisterCollation(info);
    }

    // 2. 注册特殊排序规则
    CreateCollationInfo noaccent_info(
        "icu_noaccent",
        GetICUCollateFunction("noaccent", "und-u-ks-level1-kc-true"),
        false, false);
    loader.RegisterCollation(noaccent_info);

    // 3. 注册排序键函数
    ScalarFunction sort_key(
        "icu_sort_key",
        {LogicalType::VARCHAR, LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        ICUCollateFunction,
        ICUSortKeyBind
    );
    loader.RegisterFunction(sort_key);

    // 4. 注册时区配置
    auto &config = DBConfig::GetConfig(loader.GetDatabaseInstance());
    config.AddExtensionOption(
        "TimeZone",
        "The current time zone",
        LogicalType::VARCHAR,
        Value(tz_string),
        SetICUTimeZone
    );

    // 5. 注册日期时间函数
    RegisterICUCurrentFunctions(loader);
    RegisterICUDateAddFunctions(loader);
    RegisterICUDatePartFunctions(loader);
    RegisterICUDateSubFunctions(loader);
    RegisterICUDateTruncFunctions(loader);
    RegisterICUMakeDateFunctions(loader);
    RegisterICUStrptimeFunctions(loader);
    RegisterICUTimeBucketFunctions(loader);
    RegisterICUTimeZoneFunctions(loader);

    // 6. 注册日历配置
    config.AddExtensionOption(
        "Calendar",
        "The current calendar",
        LogicalType::VARCHAR,
        Value(cal->getType()),
        SetICUCalendar
    );
}
```

### 9.5.2 排序规则实现

ICU 排序规则生成可比较的排序键：

```cpp
// 排序规则函数绑定数据
struct IcuBindData : public FunctionData {
    duckdb::unique_ptr<icu::Collator> collator;
    string language;
    string country;
    string tag;  // 如 "und-u-ks-level1-kc-true"

    explicit IcuBindData(string tag_p) : tag(std::move(tag_p)) {
        UErrorCode status = U_ZERO_ERROR;
        UCollator *ucollator = ucol_open(tag.c_str(), &status);
        if (U_FAILURE(status)) {
            throw InvalidInputException(
                "Failed to create ICU collator with tag %s: %s",
                tag, u_errorName(status));
        }
        collator = unique_ptr<icu::Collator>(
            icu::Collator::fromUCollator(ucollator));
    }
};

// 排序函数实现
static void ICUCollateFunction(
    DataChunk &args,
    ExpressionState &state,
    Vector &result
) {
    const char HEX_TABLE[] = {
        '0', '1', '2', '3', '4', '5', '6', '7',
        '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'
    };

    auto &info = state.expr.Cast<BoundFunctionExpression>()
        .bind_info->Cast<IcuBindData>();
    auto &collator = *info.collator;

    UnaryExecutor::Execute<string_t, string_t>(
        args.data[0], result, args.size(),
        [&](string_t input) {
            // 获取 ICU 排序键
            const auto string_size =
                ICUGetSortKey(collator, input, buffer, buffer_size);

            // 转换为十六进制字符串
            auto str_result = StringVector::EmptyString(
                result, (string_size - 1) * 2);
            auto str_data = str_result.GetDataWriteable();

            for (idx_t i = 0; i < string_size - 1; i++) {
                uint8_t byte = uint8_t(buffer[i]);
                str_data[i * 2] = HEX_TABLE[byte / 16];
                str_data[i * 2 + 1] = HEX_TABLE[byte % 16];
            }

            str_result.Finalize();
            return str_result;
        });
}
```

### 9.5.3 时区处理

ICU 扩展提供完整的时区支持：

```cpp
// 时区设置验证
static void SetICUTimeZone(
    ClientContext &context,
    SetScope scope,
    Value &parameter
) {
    auto tz_str = StringValue::Get(parameter);
    tz_str = NormalizeTimeZone(tz_str);  // 标准化时区名
    ICUHelpers::GetTimeZone(tz_str);      // 验证时区存在
    parameter = Value(tz_str);
}

// 时区标准化（处理特殊格式）
static string NormalizeTimeZone(const string &tz_str) {
    if (GetKnownTimeZone(tz_str)) {
        return tz_str;
    }

    // 处理 UTC±NN00 格式到 Etc/GMT±N
    if (tz_str.compare(0, 3, "UTC") == 0) {
        // UTC+8 -> Etc/GMT-8（符号取反）
        string mapped = "Etc/GMT";
        if (tz_str[3] == '+') {
            mapped += '-';
        } else if (tz_str[3] == '-') {
            mapped += '+';
        }
        // ... 处理数字部分
        if (GetKnownTimeZone(mapped)) {
            return mapped;
        }
    }
    return tz_str;
}
```

### 9.5.4 ICU 函数组织

icu 扩展的日期时间函数分布在多个文件：

```
extension/icu/
├── icu_extension.cpp      # 扩展入口
├── icu-dateadd.cpp        # 日期加法
├── icu-datesub.cpp        # 日期减法
├── icu-datepart.cpp       # 日期部分提取
├── icu-datetrunc.cpp      # 日期截断
├── icu-makedate.cpp       # 日期构造
├── icu-strptime.cpp       # 字符串解析
├── icu-timebucket.cpp     # 时间桶
├── icu-timezone.cpp       # 时区函数
├── icu-current.cpp        # 当前时间
├── icu-table-range.cpp    # 表生成器
└── third_party/icu/       # ICU 库源码
```

## 9.6 扩展开发最佳实践

### 9.6.1 代码组织模式

通过分析核心扩展，总结出最佳实践：

```
my_extension/
├── my_extension.hpp           # 扩展类声明
├── my_extension.cpp           # 扩展入口
├── include/
│   ├── my_functions.hpp       # 函数声明
│   └── my_types.hpp           # 类型定义
├── functions/
│   ├── scalar/                # 标量函数实现
│   ├── aggregate/             # 聚合函数实现
│   └── table/                 # 表函数实现
└── CMakeLists.txt             # 构建配置
```

### 9.6.2 函数注册模式

推荐的函数注册模式：

```cpp
// 1. 定义函数获取接口
class MyFunctions {
public:
    static vector<ScalarFunctionSet> GetScalarFunctions();
    static vector<TableFunctionSet> GetTableFunctions();
    static void RegisterCastFunctions(ExtensionLoader &loader);
};

// 2. 在入口处批量注册
static void LoadInternal(ExtensionLoader &loader) {
    // 批量注册标量函数
    for (auto &fun : MyFunctions::GetScalarFunctions()) {
        loader.RegisterFunction(fun);
    }

    // 批量注册表函数
    for (auto &fun : MyFunctions::GetTableFunctions()) {
        loader.RegisterFunction(fun);
    }

    // 注册类型转换
    MyFunctions::RegisterCastFunctions(loader);
}
```

### 9.6.3 类型注册模式

自定义类型注册：

```cpp
static void LoadInternal(ExtensionLoader &loader) {
    // 1. 创建并注册类型
    auto my_type = LogicalType::USER("MY_TYPE");
    loader.RegisterType("MY_TYPE", std::move(my_type));

    // 2. 注册类型转换
    loader.RegisterCastFunction(
        LogicalType::VARCHAR,
        LogicalType::USER("MY_TYPE"),
        MyTypeCastFunction,
        100  // 转换成本
    );

    loader.RegisterCastFunction(
        LogicalType::USER("MY_TYPE"),
        LogicalType::VARCHAR,
        MyTypeToStringFunction,
        50
    );
}
```

### 9.6.4 配置选项注册

扩展配置的最佳实践：

```cpp
static void LoadInternal(ExtensionLoader &loader) {
    auto &config = DBConfig::GetConfig(loader.GetDatabaseInstance());

    // 简单配置
    config.AddExtensionOption(
        "my_extension.cache_size",
        "Cache size in bytes",
        LogicalType::BIGINT,
        Value(1024 * 1024)  // 默认 1MB
    );

    // 带验证回调的配置
    config.AddExtensionOption(
        "my_extension.mode",
        "Operation mode: 'fast' or 'safe'",
        LogicalType::VARCHAR,
        Value("safe"),
        ValidateMode  // 验证回调
    );
}

// 验证回调
static void ValidateMode(
    ClientContext &context,
    SetScope scope,
    Value &parameter
) {
    auto mode = StringValue::Get(parameter);
    if (mode != "fast" && mode != "safe") {
        throw InvalidInputException(
            "Invalid mode: '%s'. Must be 'fast' or 'safe'.", mode);
    }
}
```

## 9.7 本章小结

通过分析四个核心扩展的实现，我们总结出以下关键点：

| 扩展 | 核心特性 | 注册方式 | 关键技术 |
|------|---------|---------|---------|
| core_functions | 大规模函数集 | 静态列表批量注册 | 宏定义模板 |
| parquet | 文件格式支持 | 多类型函数注册 | 替换扫描、COPY 函数 |
| json | 类型+函数 | 类型转换+标量/表函数 | SQL 宏、路径表达式 |
| icu | 国际化 | 排序规则+配置选项 | ICU 库集成 |

核心扩展的共同特点：
1. **模块化组织**：按功能域划分源文件
2. **批量注册**：通过循环/列表统一注册函数
3. **类型扩展**：定义自定义类型并注册转换函数
4. **配置集成**：提供扩展特定的配置选项
5. **替换扫描**：支持从文件名直接查询
6. **完整测试**：每个功能配套测试用例

下一章我们将介绍扩展开发实战，从零开始创建一个完整的 DuckDB 扩展。

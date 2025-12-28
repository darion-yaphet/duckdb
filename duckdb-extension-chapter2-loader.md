# DuckDB 扩展系统深度解析（二）：ExtensionLoader 注册接口

## 引言

ExtensionLoader 是扩展向 DuckDB 注册功能的核心接口。通过这个统一的注册层，扩展可以添加标量函数、聚合函数、表函数、自定义类型、类型转换等各类功能。本章将详细分析 ExtensionLoader 的设计与实现。

## 1. ExtensionLoader 核心结构

### 1.1 类定义

```cpp
// src/include/duckdb/main/extension/extension_loader.hpp

class ExtensionLoader {
    friend class DuckDB;
    friend class ExtensionHelper;

public:
    //! 从活跃加载状态构造
    explicit ExtensionLoader(ExtensionActiveLoad &load_info);
    //! 从数据库实例和扩展名构造
    ExtensionLoader(DatabaseInstance &db, const string &extension_name);

    //! 获取关联的数据库实例
    DUCKDB_API DatabaseInstance &GetDatabaseInstance();

    //! 设置扩展描述信息
    DUCKDB_API void SetDescription(const string &description);

private:
    DatabaseInstance &db;           // 数据库实例引用
    string extension_name;          // 扩展名称
    string extension_description;   // 扩展描述
    optional_ptr<ExtensionInfo> extension_info;  // 扩展信息指针
};
```

### 1.2 构造与初始化

```cpp
// src/main/extension/extension_loader.cpp

ExtensionLoader::ExtensionLoader(ExtensionActiveLoad &load_info)
    : db(load_info.db),
      extension_name(load_info.extension_name),
      extension_info(load_info.info) {
}

ExtensionLoader::ExtensionLoader(DatabaseInstance &db, const string &name)
    : db(db), extension_name(name) {
}

DatabaseInstance &ExtensionLoader::GetDatabaseInstance() {
    return db;
}
```

### 1.3 描述设置与最终化

```cpp
void ExtensionLoader::SetDescription(const string &description) {
    extension_description = description;
}

void ExtensionLoader::FinalizeLoad() {
    // 如果提供了描述，存储到扩展信息中
    if (!extension_description.empty() && extension_info) {
        auto info = make_uniq<ExtensionLoadedInfo>();
        info->description = extension_description;
        extension_info->load_info = std::move(info);
    }
}
```

## 2. 标量函数注册

### 2.1 注册接口

```cpp
//! 注册单个标量函数
DUCKDB_API void RegisterFunction(ScalarFunction function);
//! 注册标量函数集（多重载）
DUCKDB_API void RegisterFunction(ScalarFunctionSet function);
//! 使用完整创建信息注册
DUCKDB_API void RegisterFunction(CreateScalarFunctionInfo info);
```

### 2.2 实现细节

```cpp
void ExtensionLoader::RegisterFunction(ScalarFunction function) {
    // 将单个函数包装为函数集
    ScalarFunctionSet set(function.name);
    set.AddFunction(std::move(function));
    RegisterFunction(std::move(set));
}

void ExtensionLoader::RegisterFunction(ScalarFunctionSet function) {
    CreateScalarFunctionInfo info(std::move(function));
    // 冲突时合并重载，而非报错
    info.on_conflict = OnCreateConflict::ALTER_ON_CONFLICT;
    RegisterFunction(std::move(info));
}

void ExtensionLoader::RegisterFunction(CreateScalarFunctionInfo function) {
    D_ASSERT(!function.functions.name.empty());
    // 获取系统 Catalog
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    // 获取系统事务
    auto data = CatalogTransaction::GetSystemTransaction(db);
    // 创建函数
    system_catalog.CreateFunction(data, function);
}
```

### 2.3 使用示例

```cpp
// 注册简单标量函数
void MyExtension::Load(ExtensionLoader &loader) {
    // 方式1：注册单个函数
    ScalarFunction add_one(
        "add_one",                          // 函数名
        {LogicalType::INTEGER},             // 参数类型
        LogicalType::INTEGER,               // 返回类型
        [](DataChunk &args, ExpressionState &state, Vector &result) {
            auto &input = args.data[0];
            UnaryExecutor::Execute<int32_t, int32_t>(
                input, result, args.size(),
                [](int32_t val) { return val + 1; });
        });
    loader.RegisterFunction(add_one);

    // 方式2：注册多重载函数集
    ScalarFunctionSet double_func("double_it");
    double_func.AddFunction(ScalarFunction(
        {LogicalType::INTEGER}, LogicalType::INTEGER, DoubleInteger));
    double_func.AddFunction(ScalarFunction(
        {LogicalType::DOUBLE}, LogicalType::DOUBLE, DoubleDouble));
    loader.RegisterFunction(double_func);
}
```

## 3. 聚合函数注册

### 3.1 注册接口

```cpp
//! 注册单个聚合函数
DUCKDB_API void RegisterFunction(AggregateFunction function);
//! 注册聚合函数集
DUCKDB_API void RegisterFunction(AggregateFunctionSet function);
//! 使用完整创建信息注册
DUCKDB_API void RegisterFunction(CreateAggregateFunctionInfo info);
```

### 3.2 实现细节

```cpp
void ExtensionLoader::RegisterFunction(AggregateFunction function) {
    AggregateFunctionSet set(function.name);
    set.AddFunction(std::move(function));
    RegisterFunction(std::move(set));
}

void ExtensionLoader::RegisterFunction(AggregateFunctionSet function) {
    CreateAggregateFunctionInfo info(std::move(function));
    info.on_conflict = OnCreateConflict::ALTER_ON_CONFLICT;
    RegisterFunction(std::move(info));
}

void ExtensionLoader::RegisterFunction(CreateAggregateFunctionInfo function) {
    D_ASSERT(!function.functions.name.empty());
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateFunction(data, function);
}
```

### 3.3 使用示例

```cpp
// 自定义聚合函数示例：计算乘积
struct ProductState {
    double product;
    bool is_set;
};

AggregateFunction CreateProductFunction() {
    return AggregateFunction(
        "my_product",
        {LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        // state_size
        AggregateFunction::StateSize<ProductState>,
        // initialize
        AggregateFunction::StateInitialize<ProductState, ProductStateInit>,
        // update
        [](Vector inputs[], AggregateInputData &aggr_input,
           idx_t input_count, Vector &state_vector, idx_t count) {
            // 更新聚合状态
            UnaryAggregateExecutor::Execute<double, ProductState>(
                inputs[0], state_vector, count,
                [](ProductState &state, double input) {
                    if (!state.is_set) {
                        state.product = input;
                        state.is_set = true;
                    } else {
                        state.product *= input;
                    }
                });
        },
        // combine
        AggregateFunction::StateCombine<ProductState, ProductCombine>,
        // finalize
        AggregateFunction::StateFinalize<ProductState, double, ProductFinalize>
    );
}

void MyExtension::Load(ExtensionLoader &loader) {
    loader.RegisterFunction(CreateProductFunction());
}
```

## 4. 表函数注册

### 4.1 注册接口

```cpp
//! 注册单个表函数
DUCKDB_API void RegisterFunction(TableFunction function);
//! 注册表函数集
DUCKDB_API void RegisterFunction(TableFunctionSet function);
//! 使用完整创建信息注册
DUCKDB_API void RegisterFunction(CreateTableFunctionInfo info);
```

### 4.2 实现细节

```cpp
void ExtensionLoader::RegisterFunction(TableFunction function) {
    TableFunctionSet set(function.name);
    set.AddFunction(std::move(function));
    RegisterFunction(std::move(set));
}

void ExtensionLoader::RegisterFunction(TableFunctionSet function) {
    D_ASSERT(!function.name.empty());
    CreateTableFunctionInfo info(std::move(function));
    info.on_conflict = OnCreateConflict::ALTER_ON_CONFLICT;
    RegisterFunction(std::move(info));
}

void ExtensionLoader::RegisterFunction(CreateTableFunctionInfo info) {
    D_ASSERT(!info.functions.name.empty());
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateFunction(data, info);
}
```

### 4.3 使用示例

```cpp
// 自定义表函数示例：生成数字序列
struct RangeBindData : public TableFunctionData {
    int64_t start;
    int64_t end;
    int64_t step;

    unique_ptr<FunctionData> Copy() const override {
        auto result = make_uniq<RangeBindData>();
        result->start = start;
        result->end = end;
        result->step = step;
        return result;
    }

    bool Equals(const FunctionData &other) const override {
        auto &o = other.Cast<RangeBindData>();
        return start == o.start && end == o.end && step == o.step;
    }
};

struct RangeGlobalState : public GlobalTableFunctionState {
    int64_t current;
};

TableFunction CreateRangeFunction() {
    TableFunction func("my_range", {LogicalType::BIGINT, LogicalType::BIGINT},
                       RangeFunction);

    // 绑定函数：确定返回列
    func.bind = [](ClientContext &context, TableFunctionBindInput &input,
                   vector<LogicalType> &return_types, vector<string> &names)
        -> unique_ptr<FunctionData> {
        auto bind_data = make_uniq<RangeBindData>();
        bind_data->start = input.inputs[0].GetValue<int64_t>();
        bind_data->end = input.inputs[1].GetValue<int64_t>();
        bind_data->step = 1;

        return_types.push_back(LogicalType::BIGINT);
        names.push_back("value");

        return bind_data;
    };

    // 全局状态初始化
    func.init_global = [](ClientContext &context,
                          TableFunctionInitInput &input)
        -> unique_ptr<GlobalTableFunctionState> {
        auto &bind_data = input.bind_data->Cast<RangeBindData>();
        auto state = make_uniq<RangeGlobalState>();
        state->current = bind_data.start;
        return state;
    };

    return func;
}

void RangeFunction(ClientContext &context, TableFunctionInput &data,
                   DataChunk &output) {
    auto &bind_data = data.bind_data->Cast<RangeBindData>();
    auto &state = data.global_state->Cast<RangeGlobalState>();

    idx_t count = 0;
    auto result_data = FlatVector::GetData<int64_t>(output.data[0]);

    while (state.current < bind_data.end && count < STANDARD_VECTOR_SIZE) {
        result_data[count++] = state.current;
        state.current += bind_data.step;
    }

    output.SetCardinality(count);
}
```

## 5. Pragma 与 Copy 函数注册

### 5.1 Pragma 函数

```cpp
//! 注册 Pragma 函数
DUCKDB_API void RegisterFunction(PragmaFunction function);
//! 注册 Pragma 函数集
DUCKDB_API void RegisterFunction(PragmaFunctionSet function);

// 实现
void ExtensionLoader::RegisterFunction(PragmaFunction function) {
    D_ASSERT(!function.name.empty());
    PragmaFunctionSet set(function.name);
    set.AddFunction(std::move(function));
    RegisterFunction(std::move(set));
}

void ExtensionLoader::RegisterFunction(PragmaFunctionSet function) {
    D_ASSERT(!function.name.empty());
    auto function_name = function.name;
    CreatePragmaFunctionInfo info(std::move(function_name), std::move(function));
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreatePragmaFunction(data, info);
}
```

### 5.2 Copy 函数

```cpp
//! 注册 Copy 函数（导入导出格式）
DUCKDB_API void RegisterFunction(CopyFunction function);

void ExtensionLoader::RegisterFunction(CopyFunction function) {
    CreateCopyFunctionInfo info(std::move(function));
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateCopyFunction(data, info);
}
```

### 5.3 宏函数

```cpp
//! 注册宏函数
DUCKDB_API void RegisterFunction(CreateMacroInfo &info);

void ExtensionLoader::RegisterFunction(CreateMacroInfo &info) {
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateFunction(data, info);
}
```

## 6. 类型与转换注册

### 6.1 自定义类型注册

```cpp
//! 注册新类型
DUCKDB_API void RegisterType(string type_name, LogicalType type,
                             bind_logical_type_function_t bind_function = nullptr);

void ExtensionLoader::RegisterType(string type_name, LogicalType type,
                                   bind_logical_type_function_t bind_modifiers) {
    D_ASSERT(!type_name.empty());
    CreateTypeInfo info(std::move(type_name), std::move(type), bind_modifiers);
    info.temporary = true;   // 扩展类型标记为临时
    info.internal = true;    // 标记为内部类型
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateType(data, info);
}
```

### 6.2 类型转换注册

```cpp
//! 使用绑定函数注册转换
DUCKDB_API void RegisterCastFunction(const LogicalType &source,
                                     const LogicalType &target,
                                     bind_cast_function_t function,
                                     int64_t implicit_cast_cost = -1);

//! 使用 BoundCastInfo 注册转换
DUCKDB_API void RegisterCastFunction(const LogicalType &source,
                                     const LogicalType &target,
                                     BoundCastInfo function,
                                     int64_t implicit_cast_cost = -1);

// 实现
void ExtensionLoader::RegisterCastFunction(const LogicalType &source,
                                           const LogicalType &target,
                                           bind_cast_function_t bind_function,
                                           int64_t implicit_cast_cost) {
    auto &config = DBConfig::GetConfig(db);
    auto &casts = config.GetCastFunctions();
    casts.RegisterCastFunction(source, target, bind_function, implicit_cast_cost);
}

void ExtensionLoader::RegisterCastFunction(const LogicalType &source,
                                           const LogicalType &target,
                                           BoundCastInfo function,
                                           int64_t implicit_cast_cost) {
    auto &config = DBConfig::GetConfig(db);
    auto &casts = config.GetCastFunctions();
    casts.RegisterCastFunction(source, target, std::move(function), implicit_cast_cost);
}
```

### 6.3 使用示例

```cpp
void MyExtension::Load(ExtensionLoader &loader) {
    // 注册自定义类型
    LogicalType my_point_type = LogicalType::STRUCT({
        {"x", LogicalType::DOUBLE},
        {"y", LogicalType::DOUBLE}
    });
    loader.RegisterType("POINT2D", my_point_type);

    // 注册类型转换：VARCHAR -> POINT2D
    loader.RegisterCastFunction(
        LogicalType::VARCHAR,
        my_point_type,
        [](BindCastInput &input, const LogicalType &source,
           const LogicalType &target) -> BoundCastInfo {
            return BoundCastInfo(ParsePoint2D);
        },
        100  // 隐式转换代价
    );
}
```

## 7. 排序规则注册

### 7.1 注册接口

```cpp
//! 注册排序规则
DUCKDB_API void RegisterCollation(CreateCollationInfo &info);

void ExtensionLoader::RegisterCollation(CreateCollationInfo &info) {
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    // 冲突时忽略
    info.on_conflict = OnCreateConflict::IGNORE_ON_CONFLICT;
    system_catalog.CreateCollation(data, info);

    // 同时注册为函数，用于序列化
    CreateScalarFunctionInfo finfo(info.function);
    finfo.on_conflict = OnCreateConflict::IGNORE_ON_CONFLICT;
    system_catalog.CreateFunction(data, finfo);
}
```

### 7.2 使用示例

```cpp
// 注册不区分大小写的排序规则
void MyExtension::Load(ExtensionLoader &loader) {
    ScalarFunction lower_func("my_nocase", {LogicalType::VARCHAR},
                              LogicalType::VARCHAR, LowerCaseFunction);

    CreateCollationInfo info("my_nocase", lower_func,
                             true,   // combinable
                             false); // not_required_for_equality
    loader.RegisterCollation(info);
}
```

## 8. Secret 系统集成

### 8.1 Secret 类型注册

```cpp
//! 注册 Secret 类型
DUCKDB_API void RegisterSecretType(SecretType secret_type);

void ExtensionLoader::RegisterSecretType(SecretType secret_type) {
    auto &config = DBConfig::GetConfig(db);
    config.secret_manager->RegisterSecretType(secret_type);
}
```

### 8.2 Secret 创建函数

```cpp
//! 注册 Secret 创建函数
DUCKDB_API void RegisterFunction(CreateSecretFunction function);

void ExtensionLoader::RegisterFunction(CreateSecretFunction function) {
    D_ASSERT(!function.secret_type.empty());
    auto &config = DBConfig::GetConfig(db);
    config.secret_manager->RegisterSecretFunction(
        std::move(function),
        OnCreateConflict::ERROR_ON_CONFLICT);
}
```

### 8.3 使用示例

```cpp
void MyExtension::Load(ExtensionLoader &loader) {
    // 注册 Secret 类型
    SecretType my_secret_type;
    my_secret_type.name = "my_cloud";
    my_secret_type.deserializer = MySecretDeserialize;
    my_secret_type.default_provider = "config";
    loader.RegisterSecretType(my_secret_type);

    // 注册 Secret 创建函数
    CreateSecretFunction secret_func;
    secret_func.secret_type = "my_cloud";
    secret_func.provider = "config";
    secret_func.named_parameters = {
        {"access_key", LogicalType::VARCHAR},
        {"secret_key", LogicalType::VARCHAR}
    };
    secret_func.function = CreateMyCloudSecret;
    loader.RegisterFunction(secret_func);
}
```

## 9. 函数重载扩展

### 9.1 添加重载接口

已有函数可以通过 AddFunctionOverload 添加新重载：

```cpp
//! 添加标量函数重载
DUCKDB_API void AddFunctionOverload(ScalarFunction function);
DUCKDB_API void AddFunctionOverload(ScalarFunctionSet function);
//! 添加表函数重载
DUCKDB_API void AddFunctionOverload(TableFunctionSet function);
```

### 9.2 实现细节

```cpp
void ExtensionLoader::AddFunctionOverload(ScalarFunction function) {
    auto &scalar_function = GetFunction(function.name);
    scalar_function.functions.AddFunction(std::move(function));
}

void ExtensionLoader::AddFunctionOverload(ScalarFunctionSet functions) {
    D_ASSERT(!functions.name.empty());
    auto &scalar_function = GetFunction(functions.name);
    for (auto &function : functions.functions) {
        function.name = functions.name;
        scalar_function.functions.AddFunction(std::move(function));
    }
}

void ExtensionLoader::AddFunctionOverload(TableFunctionSet functions) {
    auto &table_function = GetTableFunction(functions.name);
    for (auto &function : functions.functions) {
        function.name = functions.name;
        table_function.functions.AddFunction(std::move(function));
    }
}
```

### 9.3 获取已有函数

```cpp
//! 获取标量函数（不存在则抛异常）
ScalarFunctionCatalogEntry &ExtensionLoader::GetFunction(const string &name) {
    auto catalog_entry = TryGetFunction(name);
    if (!catalog_entry) {
        throw InvalidInputException(
            "Function with name \"%s\" not found", name);
    }
    return catalog_entry->Cast<ScalarFunctionCatalogEntry>();
}

//! 尝试获取标量函数（不存在返回 nullptr）
optional_ptr<CatalogEntry> ExtensionLoader::TryGetFunction(const string &name) {
    return TryGetEntry(db, name, CatalogType::SCALAR_FUNCTION_ENTRY);
}

//! 获取表函数
TableFunctionCatalogEntry &ExtensionLoader::GetTableFunction(const string &name) {
    auto catalog_entry = TryGetTableFunction(name);
    if (!catalog_entry) {
        throw InvalidInputException(
            "Function with name \"%s\" not found", name);
    }
    return catalog_entry->Cast<TableFunctionCatalogEntry>();
}
```

## 10. 注册方法速查表

| 方法 | 注册对象 | 冲突策略 | 说明 |
|-----|---------|---------|------|
| `RegisterFunction(ScalarFunction)` | 标量函数 | ALTER | 合并重载 |
| `RegisterFunction(ScalarFunctionSet)` | 标量函数集 | ALTER | 合并重载 |
| `RegisterFunction(AggregateFunction)` | 聚合函数 | ALTER | 合并重载 |
| `RegisterFunction(AggregateFunctionSet)` | 聚合函数集 | ALTER | 合并重载 |
| `RegisterFunction(TableFunction)` | 表函数 | ALTER | 合并重载 |
| `RegisterFunction(TableFunctionSet)` | 表函数集 | ALTER | 合并重载 |
| `RegisterFunction(PragmaFunction)` | Pragma | ERROR | 冲突报错 |
| `RegisterFunction(CopyFunction)` | Copy | ERROR | 冲突报错 |
| `RegisterFunction(CreateMacroInfo)` | 宏函数 | ERROR | 冲突报错 |
| `RegisterType` | 类型 | - | 临时内部类型 |
| `RegisterCastFunction` | 类型转换 | - | 添加到转换集 |
| `RegisterCollation` | 排序规则 | IGNORE | 冲突忽略 |
| `RegisterSecretType` | Secret类型 | - | 注册到 SecretManager |
| `AddFunctionOverload` | 函数重载 | - | 扩展已有函数 |

## 小结

本章详细介绍了 ExtensionLoader 的注册接口：

1. **统一注册层**：所有类型的功能通过 ExtensionLoader 统一注册
2. **冲突处理**：函数默认 ALTER 合并重载，Pragma/Copy 冲突报错
3. **Catalog 集成**：注册内容存入系统 Catalog，供查询使用
4. **重载扩展**：支持为已有函数添加新类型重载

下一章将分析扩展的安装机制，包括仓库系统、下载流程和安装信息管理。

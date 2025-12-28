# DuckDB 函数系统深度解析（一）：架构概述

## 引言

函数是数据库系统的核心计算单元。DuckDB 实现了一套完整的函数体系，支持标量函数、聚合函数、表函数、宏函数等多种类型。本章将从整体架构角度分析 DuckDB 函数系统的设计理念、类型层次和关键抽象。

## 1. 函数类型分类

### 1.1 函数类型概览

DuckDB 支持以下主要函数类型：

| 函数类型 | 描述 | 示例 |
|---------|------|------|
| ScalarFunction | 标量函数，逐行计算 | `upper()`, `+`, `substring()` |
| AggregateFunction | 聚合函数，多行聚合 | `sum()`, `count()`, `avg()` |
| TableFunction | 表函数，返回表数据 | `read_csv()`, `generate_series()` |
| MacroFunction | 宏函数，SQL 表达式展开 | `current_user`, `list_sum()` |
| PragmaFunction | Pragma 函数，系统命令 | `pragma_table_info()` |
| CopyFunction | 复制函数，数据导入导出 | `csv`, `parquet` |

### 1.2 类层次结构

```
源文件: src/include/duckdb/function/function.hpp
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Function                                   │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  - name: string           // 函数名                              ││
│  │  - extra_info: string     // 额外信息                            ││
│  │  - catalog_name: string   // 所属 Catalog                        ││
│  │  - schema_name: string    // 所属 Schema                         ││
│  └─────────────────────────────────────────────────────────────────┘│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                ┌───────────────┴───────────────┐
                │                               │
                ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────────────┐
│      SimpleFunction       │   │        PragmaFunction             │
│  ┌─────────────────────┐  │   └───────────────────────────────────┘
│  │ - arguments         │  │
│  │ - varargs           │  │
│  └─────────────────────┘  │
└───────────────┬───────────┘
                │
    ┌───────────┴───────────────────────────┐
    │                                       │
    ▼                                       ▼
┌─────────────────────────┐   ┌─────────────────────────────────────┐
│ SimpleNamedParameter    │   │       BaseScalarFunction            │
│ Function                │   │  ┌─────────────────────────────────┐│
│ ┌─────────────────────┐ │   │  │ - return_type: LogicalType      ││
│ │ - named_parameters  │ │   │  │ - stability: FunctionStability  ││
│ └─────────────────────┘ │   │  │ - null_handling                 ││
└───────────┬─────────────┘   │  │ - errors: FunctionErrors        ││
            │                 │  │ - collation_handling            ││
            ▼                 │  └─────────────────────────────────┘│
┌─────────────────────────┐   └──────────────────┬──────────────────┘
│     TableFunction       │                      │
└─────────────────────────┘          ┌───────────┴───────────┐
                                     │                       │
                                     ▼                       ▼
                          ┌─────────────────────┐ ┌─────────────────────┐
                          │   ScalarFunction    │ │  AggregateFunction  │
                          └─────────────────────┘ └─────────────────────┘
```

## 2. 核心类定义

### 2.1 Function 基类

```cpp
// src/include/duckdb/function/function.hpp

class Function {
public:
    explicit Function(string name);
    virtual ~Function();

    //! 函数名称
    string name;
    //! 额外信息（用于区分同名函数）
    string extra_info;
    //! 所属 Catalog 名称
    string catalog_name;
    //! 所属 Schema 名称
    string schema_name;

public:
    //! 格式化函数签名：name(arg1, arg2, ...)
    static string CallToString(const string &catalog_name,
                               const string &schema_name,
                               const string &name,
                               const vector<LogicalType> &arguments,
                               const LogicalType &varargs = LogicalType::INVALID);

    //! 格式化带返回类型：name(arg1, arg2..) -> return_type
    static string CallToString(const string &catalog_name,
                               const string &schema_name,
                               const string &name,
                               const vector<LogicalType> &arguments,
                               const LogicalType &varargs,
                               const LogicalType &return_type);
};
```

### 2.2 SimpleFunction：参数管理

```cpp
class SimpleFunction : public Function {
public:
    SimpleFunction(string name, vector<LogicalType> arguments,
                   LogicalType varargs = LogicalType(LogicalTypeId::INVALID));

    //! 参数类型列表
    vector<LogicalType> arguments;
    //! 原始参数（EraseArgument 调用后保留）
    vector<LogicalType> original_arguments;
    //! 变参类型（无变参时为 INVALID）
    LogicalType varargs;

public:
    bool HasVarArgs() const;
    virtual string ToString() const;
};
```

### 2.3 BaseScalarFunction：标量函数基类

```cpp
class BaseScalarFunction : public SimpleFunction {
public:
    BaseScalarFunction(string name,
                       vector<LogicalType> arguments,
                       LogicalType return_type,
                       FunctionStability stability,
                       LogicalType varargs = LogicalType(LogicalTypeId::INVALID),
                       FunctionNullHandling null_handling = FunctionNullHandling::DEFAULT_NULL_HANDLING,
                       FunctionErrors errors = FunctionErrors::CANNOT_ERROR);

public:
    //! 返回类型
    LogicalType return_type;
    //! 函数稳定性
    FunctionStability stability;
    //! NULL 处理模式
    FunctionNullHandling null_handling;
    //! 错误处理模式
    FunctionErrors errors;
    //! 排序规则处理
    FunctionCollationHandling collation_handling;

    // Setter/Getter 方法...
};
```

## 3. 函数属性系统

### 3.1 FunctionStability：函数稳定性

```cpp
enum class FunctionStability : uint8_t {
    CONSISTENT = 0,             // 相同输入始终返回相同结果
    VOLATILE = 1,               // 结果可能每行变化（如 random()）
    CONSISTENT_WITHIN_QUERY = 2 // 同一查询内结果一致（如 now()）
};
```

稳定性影响优化器行为：

| 稳定性 | 常量折叠 | 表达式缓存 | 并行执行 |
|--------|---------|-----------|---------|
| CONSISTENT | ✅ | ✅ | ✅ |
| CONSISTENT_WITHIN_QUERY | ❌ | ✅（查询级）| ✅ |
| VOLATILE | ❌ | ❌ | ✅ |

示例：
```sql
-- CONSISTENT: 可以常量折叠
SELECT upper('hello');  -- 编译期计算为 'HELLO'

-- CONSISTENT_WITHIN_QUERY: 查询内缓存
SELECT now(), now();    -- 两次返回相同值

-- VOLATILE: 每次调用可能不同
SELECT random(), random();  -- 两个不同的随机数
```

### 3.2 FunctionNullHandling：NULL 处理

```cpp
enum class FunctionNullHandling : uint8_t {
    DEFAULT_NULL_HANDLING = 0,  // NULL 输入 → NULL 输出
    SPECIAL_HANDLING = 1        // 函数自行处理 NULL
};
```

默认模式下，执行器自动处理 NULL：
- 检测输入中的 NULL
- 直接设置输出为 NULL
- 跳过函数调用

特殊处理模式用于需要感知 NULL 的函数：
```sql
-- COALESCE 需要感知 NULL
SELECT coalesce(NULL, 1);  -- 返回 1，不是 NULL

-- IFNULL 同样
SELECT ifnull(NULL, 'default');  -- 返回 'default'
```

### 3.3 FunctionErrors：错误处理

```cpp
enum class FunctionErrors : uint8_t {
    CANNOT_ERROR = 0,           // 函数不会抛出错误
    CAN_THROW_RUNTIME_ERROR = 1 // 函数可能抛出运行时错误
};
```

这个标记帮助优化器判断是否可以安全地重排表达式：
```sql
-- 如果 division 可能抛错，不能把它移到 WHERE 之前
SELECT 1/x FROM t WHERE x != 0;
```

### 3.4 FunctionCollationHandling：排序规则处理

```cpp
enum class FunctionCollationHandling : uint8_t {
    PROPAGATE_COLLATIONS = 0,       // 传播输入的排序规则
    PUSH_COMBINABLE_COLLATIONS = 1, // 推送可组合的排序规则
    IGNORE_COLLATIONS = 2           // 忽略排序规则
};
```

## 4. 函数数据与状态

### 4.1 FunctionData：绑定时状态

```cpp
struct FunctionData {
    virtual ~FunctionData();

    //! 复制绑定数据
    virtual unique_ptr<FunctionData> Copy() const = 0;
    //! 比较绑定数据是否相等
    virtual bool Equals(const FunctionData &other) const = 0;
    //! 是否支持语句缓存
    virtual bool SupportStatementCache() const;

    template <class TARGET>
    TARGET &Cast() { ... }
};
```

FunctionData 在绑定阶段创建，存储函数执行所需的上下文信息：

```cpp
// 示例：正则表达式函数的绑定数据
struct RegexpMatchesBindData : public FunctionData {
    unique_ptr<RE2> regex;       // 编译后的正则表达式
    bool case_insensitive;       // 是否忽略大小写

    unique_ptr<FunctionData> Copy() const override {
        auto result = make_uniq<RegexpMatchesBindData>();
        result->regex = make_uniq<RE2>(regex->pattern());
        result->case_insensitive = case_insensitive;
        return result;
    }
};
```

### 4.2 FunctionLocalState：线程局部状态

```cpp
struct FunctionLocalState {
    virtual ~FunctionLocalState();

    template <class TARGET>
    TARGET &Cast() { ... }
};
```

用于存储线程私有的执行状态，避免并发访问问题：

```cpp
// 示例：字符串聚合的局部状态
struct StringAggLocalState : public FunctionLocalState {
    string buffer;  // 线程私有的字符串缓冲区
};
```

### 4.3 TableFunctionData：表函数数据

```cpp
struct TableFunctionData : public FunctionData {
    // 投影列 ID（支持投影下推）
    vector<idx_t> column_ids;

    unique_ptr<FunctionData> Copy() const override;
    bool Equals(const FunctionData &other) const override;
};
```

## 5. 函数集合与重载

### 5.1 FunctionSet 模板

```
源文件: src/include/duckdb/function/function_set.hpp
```

```cpp
template <class T>
class FunctionSet {
public:
    explicit FunctionSet(string name) : name(std::move(name)) {}

    //! 函数集名称
    string name;
    //! 重载函数列表
    vector<T> functions;

public:
    void AddFunction(T function) {
        functions.push_back(std::move(function));
    }

    idx_t Size() { return functions.size(); }

    T GetFunctionByOffset(idx_t offset) {
        D_ASSERT(offset < functions.size());
        return functions[offset];
    }

    //! 合并函数集（用于扩展添加重载）
    bool MergeFunctionSet(FunctionSet<T> new_functions, bool override = false) {
        for (auto &new_func : new_functions.functions) {
            bool overwritten = false;
            for (auto &func : functions) {
                if (new_func.Equal(func)) {
                    // 重载已存在
                    if (override) {
                        overwritten = true;
                        func = new_func;
                    } else {
                        return false;  // 冲突
                    }
                    break;
                }
            }
            if (!overwritten) {
                functions.push_back(new_func);
            }
        }
        return true;
    }
};
```

### 5.2 具体函数集类型

```cpp
class ScalarFunctionSet : public FunctionSet<ScalarFunction> {
public:
    explicit ScalarFunctionSet(string name);
    explicit ScalarFunctionSet(ScalarFunction fun);

    //! 根据参数类型选择最佳重载
    ScalarFunction GetFunctionByArguments(ClientContext &context,
                                          const vector<LogicalType> &arguments);
};

class AggregateFunctionSet : public FunctionSet<AggregateFunction> {
public:
    explicit AggregateFunctionSet(string name);

    AggregateFunction GetFunctionByArguments(ClientContext &context,
                                             const vector<LogicalType> &arguments);
};

class TableFunctionSet : public FunctionSet<TableFunction> {
public:
    explicit TableFunctionSet(string name);

    TableFunction GetFunctionByArguments(ClientContext &context,
                                         const vector<LogicalType> &arguments);
};
```

### 5.3 重载示例

```cpp
// 创建 + 运算符的重载集
ScalarFunctionSet add_set("+");

// 整数加法
add_set.AddFunction(ScalarFunction(
    {LogicalType::INTEGER, LogicalType::INTEGER},
    LogicalType::INTEGER,
    AddIntegerFunction));

// 浮点加法
add_set.AddFunction(ScalarFunction(
    {LogicalType::DOUBLE, LogicalType::DOUBLE},
    LogicalType::DOUBLE,
    AddDoubleFunction));

// 字符串连接
add_set.AddFunction(ScalarFunction(
    {LogicalType::VARCHAR, LogicalType::VARCHAR},
    LogicalType::VARCHAR,
    ConcatFunction));
```

## 6. 函数注册机制

### 6.1 FunctionList 静态注册

```
源文件: src/function/function_list.cpp
```

DuckDB 使用静态数组定义内置函数：

```cpp
struct StaticFunctionDefinition {
    const char *name;               // 函数名
    const char *alias_of;           // 别名目标
    const char *parameters;         // 参数描述
    const char *description;        // 功能描述
    const char *example;            // 使用示例
    const char *categories;         // 分类
    get_scalar_function_t get_function;        // 获取单函数
    get_scalar_function_set_t get_function_set; // 获取函数集
    // ... 聚合函数等
};

static const StaticFunctionDefinition function[] = {
    DUCKDB_SCALAR_FUNCTION(AbsFun),
    DUCKDB_SCALAR_FUNCTION_SET(OperatorAddFun),
    DUCKDB_SCALAR_FUNCTION(LowerFun),
    DUCKDB_SCALAR_FUNCTION(UpperFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(SumFun),
    DUCKDB_AGGREGATE_FUNCTION_SET(CountFun),
    // ... 200+ 个函数
    FINAL_FUNCTION
};
```

### 6.2 BuiltinFunctions 启动注册

```
源文件: src/function/built_in_functions.cpp
```

```cpp
class BuiltinFunctions {
public:
    BuiltinFunctions(CatalogTransaction transaction, Catalog &catalog);

    void AddFunction(ScalarFunctionSet set) {
        CreateScalarFunctionInfo info(std::move(set));
        info.internal = true;
        catalog.CreateFunction(transaction, info);
    }

    void AddFunction(AggregateFunction function) {
        CreateAggregateFunctionInfo info(std::move(function));
        info.internal = true;
        catalog.CreateFunction(transaction, info);
    }

    // 注册所有内置函数
    void Initialize();
};
```

### 6.3 DefaultGenerator 懒加载

对于宏函数，DuckDB 使用懒加载机制：

```cpp
// src/catalog/default/default_functions.cpp

static const DefaultMacro internal_macros[] = {
    {DEFAULT_SCHEMA, "current_user", {nullptr}, {{nullptr, nullptr}}, "'duckdb'"},
    {DEFAULT_SCHEMA, "current_catalog", {nullptr}, {{nullptr, nullptr}},
     "main.current_database()"},
    {"pg_catalog", "pg_typeof", {"expression", nullptr}, {{nullptr, nullptr}},
     "lower(typeof(expression))"},
    // ... 170+ 宏定义
    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}
};
```

懒加载流程：

```
   查询 "current_user"
          │
          ▼
┌─────────────────────────┐
│  CatalogSet::GetEntry   │
│  查找 functions map     │
└─────────────────────────┘
          │
     不存在 │
          ▼
┌─────────────────────────┐
│  DefaultFunctionGen     │
│  搜索 internal_macros   │
│  解析 SQL 表达式        │
│  创建 MacroCatalogEntry │
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

## 7. 函数执行流程概览

### 7.1 从 SQL 到执行

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SELECT upper(name) FROM t                    │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Parser                                     │
│                    FunctionExpression("upper")                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Binder                                     │
│  1. Catalog 查找 "upper" → ScalarFunctionCatalogEntry               │
│  2. FunctionBinder 选择最佳重载                                      │
│  3. 调用 bind 回调创建 FunctionData                                  │
│  4. 生成 BoundFunctionExpression                                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Planner                                     │
│                    LogicalProjection                                 │
│                    └── BoundFunctionExpression                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        PhysicalPlanner                               │
│                    PhysicalProjection                                │
│                    └── ExpressionExecutor                            │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Execution                                     │
│  1. init_local_state 初始化线程状态                                  │
│  2. 对每个 DataChunk:                                               │
│     a. NULL 处理（DEFAULT_NULL_HANDLING 模式）                       │
│     b. 调用 function(DataChunk, ExpressionState, Vector)            │
│  3. 返回结果向量                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 函数回调时机

| 阶段 | 回调 | 用途 |
|------|------|------|
| 绑定 | `bind` | 类型推导，创建 FunctionData |
| 绑定 | `bind_lambda` | Lambda 参数类型绑定 |
| 优化 | `statistics` | 统计信息传播 |
| 执行初始化 | `init_local_state` | 创建线程局部状态 |
| 执行 | `function` | 主计算逻辑 |
| 序列化 | `serialize` | 计划持久化 |
| 反序列化 | `deserialize` | 计划恢复 |

## 8. 与 Catalog 的集成

### 8.1 函数 CatalogEntry 类型

```cpp
// 函数相关的 CatalogType
enum class CatalogType : uint8_t {
    SCALAR_FUNCTION_ENTRY,    // 标量函数
    AGGREGATE_FUNCTION_ENTRY, // 聚合函数
    TABLE_FUNCTION_ENTRY,     // 表函数
    PRAGMA_FUNCTION_ENTRY,    // Pragma 函数
    MACRO_ENTRY,              // 标量宏
    TABLE_MACRO_ENTRY,        // 表宏
    COPY_FUNCTION_ENTRY,      // 复制函数
    // ...
};
```

### 8.2 Schema 中的函数存储

```cpp
class DuckSchemaEntry : public SchemaCatalogEntry {
    // 标量函数和宏存储在 functions
    CatalogSet functions;

    // 表函数存储在 table_functions
    CatalogSet table_functions;

    // Pragma 函数存储在独立集合
    CatalogSet pragma_functions;

    // Copy 函数
    CatalogSet copy_functions;
};
```

### 8.3 函数查找

```cpp
// 查找标量函数
auto &entry = catalog.GetEntry<ScalarFunctionCatalogEntry>(
    context, DEFAULT_SCHEMA, "upper");

// 获取函数集
ScalarFunctionSet &functions = entry.functions;

// 根据参数选择重载
ScalarFunction func = functions.GetFunctionByArguments(
    context, {LogicalType::VARCHAR});
```

## 9. 总结

### 9.1 设计原则

1. **类型安全**：完整的类型层次确保编译期检查
2. **可扩展性**：FunctionSet 支持动态添加重载
3. **性能优化**：稳定性标记支持常量折叠和缓存
4. **懒加载**：DefaultGenerator 减少启动开销
5. **并行友好**：FunctionLocalState 隔离线程状态

### 9.2 函数类型选择指南

| 场景 | 推荐类型 |
|------|---------|
| 逐行计算 | ScalarFunction |
| 多行聚合 | AggregateFunction |
| 返回表数据 | TableFunction |
| SQL 表达式组合 | ScalarMacroFunction |
| SQL 查询组合 | TableMacroFunction |
| 系统配置 | PragmaFunction |

### 9.3 核心类关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Function System                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │ FunctionSet │───▶│  Function   │───▶│    FunctionData         │ │
│  │             │    │             │    │  (Bind-time state)      │ │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘ │
│         │                  │                        │               │
│         │                  │                        ▼               │
│         │                  │           ┌─────────────────────────┐ │
│         │                  │           │  FunctionLocalState     │ │
│         │                  │           │  (Thread-local state)   │ │
│         │                  │           └─────────────────────────┘ │
│         │                  │                                        │
│         ▼                  ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FunctionBinder                            │   │
│  │  - 重载解析                                                  │   │
│  │  - 类型转换                                                  │   │
│  │  - 参数绑定                                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   ExpressionExecutor                         │   │
│  │  - 向量化执行                                                │   │
│  │  - NULL 处理                                                 │   │
│  │  - 结果生成                                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

下一章将深入分析标量函数的实现机制，包括向量化执行器和典型函数实现。

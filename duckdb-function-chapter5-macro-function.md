# 第五章：宏函数与 SQL 级抽象

## 5.1 宏函数概述

宏函数是 DuckDB 函数系统中独特的一类，它在 SQL 层面提供了表达式级别的抽象和复用能力。与传统的编译时执行的标量函数和聚合函数不同，宏函数在绑定阶段进行展开，本质上是一种参数化的表达式模板。

### 5.1.1 宏与普通函数的区别

```
┌──────────────────────────────────────────────────────────────────────┐
│                    宏函数 vs 普通函数对比                              │
├─────────────────────────┬────────────────────────────────────────────┤
│         特性            │    宏函数          │    普通函数            │
├─────────────────────────┼────────────────────┼────────────────────────┤
│ 定义语言                │    SQL 表达式      │    C++ 代码            │
│ 处理时机                │    绑定期展开      │    执行期调用          │
│ 参数传递                │    表达式替换      │    值传递              │
│ 类型检查                │    展开后检查      │    签名匹配            │
│ 可扩展性                │    用户可定义      │    需重新编译          │
│ 性能特征                │    零调用开销      │    函数调用开销        │
│ 适用场景                │    组合逻辑        │    核心计算            │
└─────────────────────────┴────────────────────┴────────────────────────┘
```

宏函数的核心价值在于：
1. **表达式复用**：将常用的 SQL 表达式封装为可复用的模板
2. **零调用开销**：绑定期展开，执行时无函数调用开销
3. **用户可定义**：无需修改 C++ 代码即可扩展功能
4. **PostgreSQL 兼容**：支持大量 PostgreSQL 系统函数的模拟

### 5.1.2 宏类型体系

DuckDB 支持两种主要的宏类型：

```cpp
// src/include/duckdb/function/macro_function.hpp
enum class MacroType : uint8_t {
    VOID_MACRO = 0,     // 保留类型
    TABLE_MACRO = 1,    // 表宏：返回表结果
    SCALAR_MACRO = 2    // 标量宏：返回单个表达式
};
```

**标量宏（Scalar Macro）**：返回单个表达式值
```sql
-- 定义
CREATE MACRO add_one(x) AS x + 1;
-- 使用
SELECT add_one(5);  -- 展开为: SELECT 5 + 1
```

**表宏（Table Macro）**：返回完整的查询结果
```sql
-- 定义
CREATE MACRO top_n(tbl, n) AS TABLE
    SELECT * FROM tbl LIMIT n;
-- 使用
SELECT * FROM top_n(my_table, 10);
```

## 5.2 MacroFunction 基类设计

### 5.2.1 核心数据结构

```cpp
// src/include/duckdb/function/macro_function.hpp
class MacroFunction {
public:
    //! 宏类型：SCALAR_MACRO 或 TABLE_MACRO
    MacroType type;

    //! 参数列表（ColumnRefExpression 形式）
    vector<unique_ptr<ParsedExpression>> parameters;

    //! 默认参数值映射（参数名 → 默认表达式）
    InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> default_parameters;

    //! 参数类型列表（支持类型化宏）
    vector<LogicalType> types;

public:
    //! 绑定宏函数调用，选择最佳匹配的重载
    static MacroBindResult BindMacroFunction(
        Binder &binder,
        const vector<unique_ptr<MacroFunction>> &macro_functions,
        const string &name,
        FunctionExpression &function_expr,
        vector<unique_ptr<ParsedExpression>> &positional_arguments,
        InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> &named_arguments,
        idx_t depth);

    //! 创建虚拟绑定表，用于参数替换
    static unique_ptr<DummyBinding> CreateDummyBinding(
        const MacroFunction &macro_def,
        const string &name,
        vector<unique_ptr<ParsedExpression>> &positional_arguments,
        InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> &named_arguments);
};
```

### 5.2.2 参数系统设计

宏函数支持灵活的参数系统：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      宏参数系统                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 位置参数（Positional Parameters）                               │
│     CREATE MACRO foo(a, b, c) AS ...                                │
│     调用: foo(1, 2, 3)                                              │
│                                                                     │
│  2. 命名参数（Named Parameters）                                    │
│     调用: foo(a := 1, c := 3, b := 2)                               │
│                                                                     │
│  3. 默认参数（Default Parameters）                                  │
│     CREATE MACRO bar(x, y := 10) AS x + y                           │
│     调用: bar(5) → 展开为 5 + 10                                     │
│                                                                     │
│  4. 类型化参数（Typed Parameters）                                  │
│     CREATE MACRO typed_add(x INTEGER, y INTEGER) AS x + y           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

参数在 `parameters` 向量中以 `ColumnRefExpression` 形式存储：

```cpp
// 创建宏时设置参数
for (idx_t param_idx = 0; default_macro.parameters[param_idx] != nullptr; param_idx++) {
    function->parameters.push_back(
        make_uniq<ColumnRefExpression>(default_macro.parameters[param_idx]));
}

// 设置默认参数
for (idx_t named_idx = 0; default_macro.named_parameters[named_idx].name != nullptr; named_idx++) {
    const auto &named_param = default_macro.named_parameters[named_idx];
    auto expr_list = Parser::ParseExpressionList(named_param.default_value);
    function->parameters.push_back(make_uniq<ColumnRefExpression>(named_param.name));
    function->default_parameters.insert(make_pair(named_param.name, std::move(expr_list[0])));
}
```

## 5.3 标量宏实现

### 5.3.1 ScalarMacroFunction 结构

```cpp
// src/include/duckdb/function/scalar_macro_function.hpp
class ScalarMacroFunction : public MacroFunction {
public:
    static constexpr const MacroType TYPE = MacroType::SCALAR_MACRO;

    //! 宏体表达式
    unique_ptr<ParsedExpression> expression;

public:
    explicit ScalarMacroFunction(unique_ptr<ParsedExpression> expression);

    unique_ptr<MacroFunction> Copy() const override;
    string ToSQL() const override;
};
```

标量宏的核心是一个 `ParsedExpression`，表示宏体的 SQL 表达式：

```cpp
// src/function/scalar_macro_function.cpp
ScalarMacroFunction::ScalarMacroFunction(unique_ptr<ParsedExpression> expression)
    : MacroFunction(MacroType::SCALAR_MACRO), expression(std::move(expression)) {
}

string ScalarMacroFunction::ToSQL() const {
    auto expression_copy = expression->Copy();
    RemoveQualificationRecursive(expression_copy);
    return MacroFunction::ToSQL() + StringUtil::Format("(%s)", expression_copy->ToString());
}
```

### 5.3.2 标量宏绑定流程

标量宏的绑定在 `ExpressionBinder::BindMacro` 中完成：

```cpp
// src/planner/binder/expression/bind_macro_expression.cpp
BindResult ExpressionBinder::BindMacro(FunctionExpression &function,
                                       ScalarMacroCatalogEntry &macro_func,
                                       idx_t depth,
                                       unique_ptr<ParsedExpression> &expr) {
    auto stack_checker = StackCheck(*expr, 3);

    // 1. 展开宏表达式
    UnfoldMacroExpression(function, macro_func, expr, depth);

    // 2. 绑定展开后的表达式
    return BindExpression(expr, depth);
}
```

`UnfoldMacroExpression` 是核心展开逻辑：

```cpp
void ExpressionBinder::UnfoldMacroExpression(FunctionExpression &function,
                                             ScalarMacroCatalogEntry &macro_func,
                                             unique_ptr<ParsedExpression> &expr,
                                             idx_t depth) {
    // 1. 分离位置参数和命名参数
    vector<unique_ptr<ParsedExpression>> positional_arguments;
    InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> named_arguments;

    // 2. 绑定宏函数，选择最佳匹配的重载
    auto bind_result = MacroFunction::BindMacroFunction(
        binder, macro_func.macros, macro_func.name, function,
        positional_arguments, named_arguments, depth);

    auto &macro_def = macro_func.macros[bind_result.function_idx.GetIndex()]
                          ->Cast<ScalarMacroFunction>();

    // 3. 创建虚拟绑定表
    auto new_macro_binding = MacroFunction::CreateDummyBinding(
        macro_def, macro_func.name, positional_arguments, named_arguments);
    macro_binding = new_macro_binding.get();

    // 4. 复制宏体表达式
    expr = macro_def.expression->Copy();

    // 5. 限定宏参数引用
    auto dummy_binder = Binder::CreateBinder(context);
    dummy_binder->macro_binding = new_macro_binding.get();
    ExpressionBinder::QualifyColumnNames(*dummy_binder, expr);

    // 6. 替换参数引用为实际参数
    vector<unordered_set<string>> lambda_params;
    ReplaceMacroParameters(expr, lambda_params);
}
```

### 5.3.3 参数替换机制

参数替换是宏展开的核心步骤：

```cpp
// src/planner/binder/expression/bind_macro_expression.cpp
void ExpressionBinder::ReplaceMacroParameters(unique_ptr<ParsedExpression> &expr,
                                              vector<unordered_set<string>> &lambda_params) {
    switch (expr->GetExpressionClass()) {
    case ExpressionClass::COLUMN_REF: {
        auto &col_ref = expr->Cast<ColumnRefExpression>();

        // 跳过 Lambda 参数
        if (LambdaExpression::IsLambdaParameter(lambda_params, col_ref.GetName())) {
            return;
        }

        // 检查是否为宏参数引用
        bool bind_macro_parameter = false;
        if (col_ref.IsQualified()) {
            // 带限定符的引用：检查表名是否为虚拟绑定表
            if (col_ref.GetTableName().find(DummyBinding::DUMMY_NAME) != string::npos) {
                bind_macro_parameter = true;
            }
        } else {
            // 不带限定符：检查是否匹配宏参数
            bind_macro_parameter = macro_binding->HasMatchingBinding(col_ref.GetColumnName());
        }

        // 替换为实际参数表达式
        if (bind_macro_parameter) {
            expr = macro_binding->ParamToArg(col_ref);
        }
        return;
    }
    case ExpressionClass::FUNCTION: {
        // Lambda 函数需要特殊处理
        auto &function = expr->Cast<FunctionExpression>();
        if (function.IsLambdaFunction()) {
            return ReplaceMacroParametersInLambda(function, lambda_params);
        }
        break;
    }
    case ExpressionClass::SUBQUERY: {
        // 递归处理子查询
        auto &sq = expr->Cast<SubqueryExpression>().subquery;
        ParsedExpressionIterator::EnumerateQueryNodeChildren(
            *sq->node, [&](unique_ptr<ParsedExpression> &child) {
                ReplaceMacroParameters(child, lambda_params);
            });
        break;
    }
    default:
        break;
    }

    // 递归处理子表达式
    ParsedExpressionIterator::EnumerateChildren(
        *expr, [&](unique_ptr<ParsedExpression> &child) {
            ReplaceMacroParameters(child, lambda_params);
        });
}
```

## 5.4 表宏实现

### 5.4.1 TableMacroFunction 结构

```cpp
// src/include/duckdb/function/table_macro_function.hpp
class TableMacroFunction : public MacroFunction {
public:
    static constexpr const MacroType TYPE = MacroType::TABLE_MACRO;

    //! 查询节点（SELECT 语句的 AST）
    unique_ptr<QueryNode> query_node;

public:
    explicit TableMacroFunction(unique_ptr<QueryNode> query_node);

    string ToSQL() const override;
};
```

表宏存储的是完整的 `QueryNode`，而不仅仅是表达式：

```cpp
// src/function/table_macro_function.cpp
TableMacroFunction::TableMacroFunction(unique_ptr<QueryNode> query_node)
    : MacroFunction(MacroType::TABLE_MACRO), query_node(std::move(query_node)) {
}

string TableMacroFunction::ToSQL() const {
    return MacroFunction::ToSQL() + StringUtil::Format("TABLE (%s)", query_node->ToString());
}
```

### 5.4.2 表宏绑定流程

表宏的绑定在 `Binder::BindTableMacro` 中完成：

```cpp
// src/planner/binder/query_node/bind_table_macro_node.cpp
unique_ptr<QueryNode> Binder::BindTableMacro(FunctionExpression &function,
                                              TableMacroCatalogEntry &macro_func,
                                              idx_t depth) {
    // 1. 分离位置参数和命名参数
    vector<unique_ptr<ParsedExpression>> positional_arguments;
    InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> named_arguments;

    // 2. 绑定宏函数
    auto bind_result = MacroFunction::BindMacroFunction(
        *this, macro_func.macros, macro_func.name, function,
        positional_arguments, named_arguments, depth);

    auto &macro_def = *macro_func.macros[bind_result.function_idx.GetIndex()];

    // 3. 创建虚拟绑定表
    auto new_macro_binding = MacroFunction::CreateDummyBinding(
        macro_def, macro_func.name, positional_arguments, named_arguments);
    new_macro_binding->arguments = &positional_arguments;

    // 4. 创建 ExpressionBinder 进行参数替换
    auto eb = ExpressionBinder(*this, this->context);
    eb.macro_binding = new_macro_binding.get();

    // 5. 复制查询节点并替换参数
    auto node = macro_def.Cast<TableMacroFunction>().query_node->Copy();
    vector<unordered_set<string>> lambda_params;
    ParsedExpressionIterator::EnumerateQueryNodeChildren(
        *node, [&](unique_ptr<ParsedExpression> &child) {
            eb.ReplaceMacroParameters(child, lambda_params);
        });

    return node;
}
```

## 5.5 DummyBinding 虚拟绑定机制

### 5.5.1 DummyBinding 结构

`DummyBinding` 是宏参数替换的关键组件：

```cpp
// src/include/duckdb/planner/table_binding.hpp
struct DummyBinding : public Binding {
public:
    static constexpr const BindingType TYPE = BindingType::DUMMY;
    // 虚拟表名常量
    static constexpr const char *DUMMY_NAME = "0_macro_parameters";

public:
    DummyBinding(vector<LogicalType> types, vector<string> names, string dummy_name);

    //! 宏参数表达式列表
    vector<unique_ptr<ParsedExpression>> *arguments;
    //! 虚拟绑定表名称
    string dummy_name;

public:
    //! 绑定列引用
    BindResult Bind(ColumnRefExpression &col_ref, idx_t depth) override;
    //! 绑定 Lambda 引用
    BindResult Bind(LambdaRefExpression &lambda_ref, idx_t depth);

    //! 将列引用转换为参数表达式
    unique_ptr<ParsedExpression> ParamToArg(ColumnRefExpression &col_ref);
};
```

### 5.5.2 虚拟绑定创建过程

```cpp
// src/function/macro_function.cpp
unique_ptr<DummyBinding> MacroFunction::CreateDummyBinding(
    const MacroFunction &macro_def,
    const string &name,
    vector<unique_ptr<ParsedExpression>> &positional_arguments,
    InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> &named_arguments) {

    // 1. 构建类型列表
    vector<LogicalType> types = macro_def.types;
    types.resize(macro_def.parameters.size(), LogicalType::UNKNOWN);

    // 2. 构建参数名列表
    vector<string> names;
    for (idx_t i = 0; i < positional_arguments.size(); i++) {
        names.push_back(macro_def.parameters[i]->Cast<ColumnRefExpression>().GetColumnName());
    }

    // 3. 将命名参数合并到位置参数中
    for (auto &kv : named_arguments) {
        names.push_back(kv.first);
        positional_arguments.push_back(std::move(kv.second));
    }

    // 4. 创建 DummyBinding
    auto res = make_uniq<DummyBinding>(types, names, name);
    res->arguments = &positional_arguments;
    return res;
}
```

### 5.5.3 参数替换过程图解

```
┌───────────────────────────────────────────────────────────────────────┐
│                      宏展开过程                                        │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  原始调用:                                                            │
│  ┌─────────────────────────────────────┐                             │
│  │ SELECT add_one(5)                   │                             │
│  └─────────────────────────────────────┘                             │
│                    │                                                  │
│                    ▼                                                  │
│  宏定义:                                                              │
│  ┌─────────────────────────────────────┐                             │
│  │ CREATE MACRO add_one(x) AS x + 1    │                             │
│  │   parameters: [x]                   │                             │
│  │   expression: x + 1                 │                             │
│  └─────────────────────────────────────┘                             │
│                    │                                                  │
│                    ▼                                                  │
│  创建 DummyBinding:                                                   │
│  ┌─────────────────────────────────────┐                             │
│  │ names: ["x"]                        │                             │
│  │ arguments: [5]  (实际参数表达式)     │                             │
│  │ dummy_name: "add_one"               │                             │
│  └─────────────────────────────────────┘                             │
│                    │                                                  │
│                    ▼                                                  │
│  限定参数引用:                                                        │
│  ┌─────────────────────────────────────┐                             │
│  │ x + 1 → 0_macro_parameters.x + 1    │                             │
│  └─────────────────────────────────────┘                             │
│                    │                                                  │
│                    ▼                                                  │
│  参数替换 (ParamToArg):                                               │
│  ┌─────────────────────────────────────┐                             │
│  │ 0_macro_parameters.x + 1 → 5 + 1    │                             │
│  └─────────────────────────────────────┘                             │
│                    │                                                  │
│                    ▼                                                  │
│  展开后的查询:                                                        │
│  ┌─────────────────────────────────────┐                             │
│  │ SELECT 5 + 1                        │                             │
│  └─────────────────────────────────────┘                             │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## 5.6 宏绑定与重载解析

### 5.6.1 宏重载支持

DuckDB 支持同名宏的重载，类似于函数重载：

```cpp
// src/include/duckdb/catalog/catalog_entry/macro_catalog_entry.hpp
class MacroCatalogEntry : public FunctionEntry {
public:
    //! 存储多个重载版本
    vector<unique_ptr<MacroFunction>> macros;
};
```

### 5.6.2 重载解析算法

`BindMacroFunction` 实现了宏的重载解析：

```cpp
// src/function/macro_function.cpp
MacroBindResult MacroFunction::BindMacroFunction(
    Binder &binder,
    const vector<unique_ptr<MacroFunction>> &functions,
    const string &name,
    FunctionExpression &function_expr,
    vector<unique_ptr<ParsedExpression>> &positional_arguments,
    InsertionOrderPreservingMap<unique_ptr<ParsedExpression>> &named_arguments,
    idx_t depth) {

    // 1. 确定是否需要类型绑定
    bool requires_bind = false;
    for (auto &function : functions) {
        for (const auto &type : function->types) {
            if (type.id() != LogicalTypeId::UNKNOWN) {
                requires_bind = true;
                break;
            }
        }
    }

    // 2. 分离位置参数和命名参数
    vector<LogicalType> positional_arg_types;
    InsertionOrderPreservingMap<LogicalType> named_arg_types;
    for (auto &arg : function_expr.children) {
        // 绑定参数以获取类型（如果需要）
        LogicalType arg_type = LogicalType::UNKNOWN;
        if (requires_bind) {
            // ... 绑定表达式获取类型
        }

        if (!arg->GetAlias().empty()) {
            // 命名参数
            named_arg_types.insert(arg->GetAlias(), std::move(arg_type));
            named_arguments[arg->GetAlias()] = std::move(arg);
        } else {
            // 位置参数
            positional_arguments.push_back(std::move(arg));
            positional_arg_types.push_back(std::move(arg_type));
        }
    }

    // 3. 计算每个候选宏的匹配代价
    auto lowest_cost = NumericLimits<idx_t>::Maximum();
    vector<idx_t> result_indices;

    for (idx_t function_idx = 0; function_idx < functions.size(); function_idx++) {
        auto &function = functions[function_idx];

        // 检查参数数量是否匹配
        if (positional_arguments.size() > function->parameters.size()) {
            continue;
        }

        // 计算类型转换代价
        idx_t macro_cost = 0;
        for (idx_t param_idx = 0; param_idx < positional_arguments.size(); param_idx++) {
            const auto &param_type = parameter_types[param_idx];
            if (param_type == LogicalType::UNKNOWN) {
                macro_cost += 1000000;  // 无类型信息时给予高代价
            } else {
                auto cast_cost = CastFunctionSet::ImplicitCastCost(
                    binder.context, positional_arg_types[param_idx], param_type);
                if (cast_cost < 0) {
                    break;  // 无法转换
                }
                macro_cost += cast_cost;
            }
        }

        // 选择代价最低的候选
        if (macro_cost <= lowest_cost) {
            if (macro_cost < lowest_cost) {
                lowest_cost = macro_cost;
                result_indices.clear();
            }
            result_indices.push_back(function_idx);
        }
    }

    // 4. 处理无匹配或多匹配情况
    if (result_indices.size() != 1) {
        // 生成错误信息
        return MacroBindResult(error);
    }

    // 5. 为类型化参数添加显式类型转换
    const auto &macro_def = *functions[result_indices[0]];
    for (idx_t param_idx = 0; param_idx < positional_arguments.size(); param_idx++) {
        if (parameter_types[param_idx] != LogicalType::UNKNOWN) {
            auto &arg = positional_arguments[param_idx];
            arg = make_uniq<CastExpression>(parameter_types[param_idx], std::move(arg));
        }
    }

    return MacroBindResult(result_indices[0]);
}
```

### 5.6.3 匹配代价计算

```
┌──────────────────────────────────────────────────────────────────────┐
│                      宏匹配代价计算                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  代价计算规则:                                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 1. 无类型参数 (UNKNOWN): 代价 += 1000000                        │ │
│  │ 2. 精确类型匹配: 代价 += 0                                      │ │
│  │ 3. 隐式转换可行: 代价 += ImplicitCastCost()                     │ │
│  │ 4. 无法转换: 跳过此候选                                         │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  选择策略:                                                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ - 选择代价最低的候选                                            │ │
│  │ - 如果多个候选代价相同，报告歧义错误                             │ │
│  │ - 建议用户添加显式类型转换或使用命名参数                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## 5.7 内置宏系统

### 5.7.1 内置宏定义表

DuckDB 通过静态数组定义大量内置宏：

```cpp
// src/catalog/default/default_functions.cpp
static const DefaultMacro internal_macros[] = {
    // 用户标识相关
    {DEFAULT_SCHEMA, "current_role", {nullptr}, {{nullptr, nullptr}}, "'duckdb'"},
    {DEFAULT_SCHEMA, "current_user", {nullptr}, {{nullptr, nullptr}}, "'duckdb'"},
    {DEFAULT_SCHEMA, "current_catalog", {nullptr}, {{nullptr, nullptr}}, "main.current_database()"},
    {DEFAULT_SCHEMA, "user", {nullptr}, {{nullptr, nullptr}}, "current_user"},
    {DEFAULT_SCHEMA, "session_user", {nullptr}, {{nullptr, nullptr}}, "'duckdb'"},

    // PostgreSQL 兼容函数
    {"pg_catalog", "pg_typeof", {"expression", nullptr}, {{nullptr, nullptr}},
     "lower(typeof(expression))"},
    {"pg_catalog", "current_database", {nullptr}, {{nullptr, nullptr}},
     "system.main.current_database()"},
    {"pg_catalog", "pg_get_viewdef", {"oid", nullptr}, {{nullptr, nullptr}},
     "(select sql from duckdb_views() v where v.view_oid=oid)"},

    // 数学函数
    {DEFAULT_SCHEMA, "round_even", {"x", "n", nullptr}, {{nullptr, nullptr}},
     "CASE ((abs(x) * power(10, n+1)) % 10) WHEN 5 THEN round(x/2, n) * 2 ELSE round(x, n) END"},
    {DEFAULT_SCHEMA, "nullif", {"a", "b", nullptr}, {{nullptr, nullptr}},
     "CASE WHEN a=b THEN NULL ELSE a END"},

    // 列表操作
    {DEFAULT_SCHEMA, "list_append", {"l", "e", nullptr}, {{nullptr, nullptr}},
     "list_concat(l, list_value(e))"},
    {DEFAULT_SCHEMA, "list_prepend", {"e", "l", nullptr}, {{nullptr, nullptr}},
     "list_concat(list_value(e), l)"},
    {DEFAULT_SCHEMA, "list_reverse", {"l", nullptr}, {{nullptr, nullptr}}, "l[:-:-1]"},

    // 列表聚合
    {DEFAULT_SCHEMA, "list_avg", {"l", nullptr}, {{nullptr, nullptr}}, "list_aggr(l, 'avg')"},
    {DEFAULT_SCHEMA, "list_sum", {"l", nullptr}, {{nullptr, nullptr}}, "list_aggr(l, 'sum')"},
    {DEFAULT_SCHEMA, "list_min", {"l", nullptr}, {{nullptr, nullptr}}, "list_aggr(l, 'min')"},
    {DEFAULT_SCHEMA, "list_max", {"l", nullptr}, {{nullptr, nullptr}}, "list_aggr(l, 'max')"},

    // 日期函数
    {DEFAULT_SCHEMA, "date_add", {"date", "interval", nullptr}, {{nullptr, nullptr}},
     "date + interval"},

    // ... 更多内置宏
    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}
};
```

### 5.7.2 DefaultMacro 结构

```cpp
struct DefaultMacro {
    const char *schema;                    // 模式名
    const char *name;                      // 宏名
    const char *parameters[8];             // 参数列表（最多 8 个）
    DefaultNamedParameter named_parameters[8];  // 命名参数
    const char *macro;                     // 宏体表达式
};

struct DefaultNamedParameter {
    const char *name;
    const char *default_value;
};
```

### 5.7.3 懒加载机制

内置宏通过 `DefaultFunctionGenerator` 懒加载：

```cpp
// src/catalog/default/default_functions.cpp
class DefaultFunctionGenerator : public DefaultGenerator {
public:
    DefaultFunctionGenerator(Catalog &catalog, SchemaCatalogEntry &schema);

    unique_ptr<CatalogEntry> CreateDefaultEntry(ClientContext &context,
                                                 const string &entry_name) override {
        auto info = GetDefaultFunction(schema.name, entry_name);
        if (info) {
            return make_uniq_base<CatalogEntry, ScalarMacroCatalogEntry>(
                catalog, schema, info->Cast<CreateMacroInfo>());
        }
        return nullptr;
    }
};
```

懒加载过程：

```
┌──────────────────────────────────────────────────────────────────────┐
│                      内置宏懒加载流程                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 首次引用 current_user:                                           │
│     ┌────────────────────────────────────────┐                      │
│     │ SELECT current_user                    │                      │
│     └────────────────────────────────────────┘                      │
│                       │                                              │
│                       ▼                                              │
│  2. Catalog 查找失败:                                                │
│     ┌────────────────────────────────────────┐                      │
│     │ CatalogSet::GetEntry("current_user")   │                      │
│     │ → 未找到                               │                      │
│     └────────────────────────────────────────┘                      │
│                       │                                              │
│                       ▼                                              │
│  3. 触发 DefaultGenerator:                                          │
│     ┌────────────────────────────────────────┐                      │
│     │ DefaultFunctionGenerator::             │                      │
│     │   CreateDefaultEntry("current_user")   │                      │
│     └────────────────────────────────────────┘                      │
│                       │                                              │
│                       ▼                                              │
│  4. 查找 internal_macros 表:                                         │
│     ┌────────────────────────────────────────┐                      │
│     │ GetDefaultFunction("main", "current_user") │                  │
│     │ → 找到定义: "'duckdb'"                  │                      │
│     └────────────────────────────────────────┘                      │
│                       │                                              │
│                       ▼                                              │
│  5. 创建 CatalogEntry:                                               │
│     ┌────────────────────────────────────────┐                      │
│     │ ScalarMacroCatalogEntry 创建并缓存     │                      │
│     └────────────────────────────────────────┘                      │
│                       │                                              │
│                       ▼                                              │
│  6. 后续引用直接命中缓存                                              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## 5.8 宏 Catalog 条目

### 5.8.1 MacroCatalogEntry 基类

```cpp
// src/include/duckdb/catalog/catalog_entry/macro_catalog_entry.hpp
class MacroCatalogEntry : public FunctionEntry {
public:
    MacroCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema, CreateMacroInfo &info);

    //! 宏函数列表（支持重载）
    vector<unique_ptr<MacroFunction>> macros;

public:
    unique_ptr<CreateInfo> GetInfo() const override;
    string ToSQL() const override;
};
```

### 5.8.2 标量宏和表宏条目

```cpp
// 标量宏条目
class ScalarMacroCatalogEntry : public MacroCatalogEntry {
public:
    static constexpr const CatalogType Type = CatalogType::MACRO_ENTRY;
    static constexpr const char *Name = "macro function";
};

// 表宏条目
class TableMacroCatalogEntry : public MacroCatalogEntry {
public:
    static constexpr const CatalogType Type = CatalogType::TABLE_MACRO_ENTRY;
    static constexpr const char *Name = "table macro function";
};
```

### 5.8.3 CreateMacroInfo 信息结构

```cpp
// src/include/duckdb/parser/parsed_data/create_macro_info.hpp
struct CreateMacroInfo : public CreateFunctionInfo {
    explicit CreateMacroInfo(CatalogType type);

    //! 宏函数列表（支持多个重载）
    vector<unique_ptr<MacroFunction>> macros;

    unique_ptr<CreateInfo> Copy() const override;
    string ToString() const override;
    void Serialize(Serializer &serializer) const override;
    static unique_ptr<CreateInfo> Deserialize(Deserializer &deserializer);
};
```

## 5.9 Lambda 与宏的交互

### 5.9.1 Lambda 参数保护

当宏体包含 Lambda 表达式时，需要特殊处理以避免错误替换 Lambda 参数：

```cpp
// src/planner/binder/expression/bind_macro_expression.cpp
void ExpressionBinder::ReplaceMacroParametersInLambda(
    FunctionExpression &function,
    vector<unordered_set<string>> &lambda_params) {

    for (auto &child : function.children) {
        if (child->GetExpressionClass() != ExpressionClass::LAMBDA) {
            // 非 Lambda 子表达式正常替换
            ReplaceMacroParameters(child, lambda_params);
            continue;
        }

        // Lambda 表达式特殊处理
        auto &lambda_expr = child->Cast<LambdaExpression>();
        string error_message;
        auto column_ref_expressions = lambda_expr.ExtractColumnRefExpressions(error_message);

        if (!error_message.empty()) {
            // 可能是 JSON 函数，替换 LHS 和 RHS
            ReplaceMacroParameters(lambda_expr.lhs, lambda_params);
            ReplaceMacroParameters(lambda_expr.expr, lambda_params);
            continue;
        }

        // 记录当前层的 Lambda 参数名
        lambda_params.emplace_back();
        for (const auto &column_ref_expr : column_ref_expressions) {
            const auto &column_ref = column_ref_expr.get().Cast<ColumnRefExpression>();
            lambda_params.back().emplace(column_ref.GetName());
        }

        // 只替换 RHS（Lambda 体），不替换 LHS（Lambda 参数）
        ReplaceMacroParameters(lambda_expr.expr, lambda_params);

        lambda_params.pop_back();
    }
}
```

### 5.9.2 Lambda 参数跳过逻辑

```cpp
void ExpressionBinder::ReplaceMacroParameters(unique_ptr<ParsedExpression> &expr,
                                              vector<unordered_set<string>> &lambda_params) {
    switch (expr->GetExpressionClass()) {
    case ExpressionClass::COLUMN_REF: {
        auto &col_ref = expr->Cast<ColumnRefExpression>();

        // 关键：检查是否为 Lambda 参数
        if (LambdaExpression::IsLambdaParameter(lambda_params, col_ref.GetName())) {
            return;  // 不替换 Lambda 参数
        }

        // ... 正常的宏参数替换
    }
    // ...
    }
}
```

## 5.10 宏的性能特征

### 5.10.1 零调用开销

由于宏在绑定阶段展开，执行时没有函数调用开销：

```
┌──────────────────────────────────────────────────────────────────────┐
│                    执行模型对比                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  普通函数执行:                                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ for each row:                                                  │ │
│  │   push arguments                                               │ │
│  │   call function                                                │ │
│  │   execute function body                                        │ │
│  │   pop return value                                             │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  宏展开后:                                                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ for each row:                                                  │ │
│  │   execute inlined expression (无调用开销)                       │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.10.2 表达式内联优化

宏展开后的表达式可以参与后续优化：

```sql
-- 原始查询
SELECT round_even(price, 2) FROM products WHERE price > 100;

-- 宏展开后
SELECT CASE ((abs(price) * power(10, 3)) % 10)
       WHEN 5 THEN round(price/2, 2) * 2
       ELSE round(price, 2) END
FROM products
WHERE price > 100;

-- 优化器可以进一步优化：
-- 1. 常量折叠：power(10, 3) → 1000
-- 2. 表达式简化
-- 3. 与其他优化规则结合
```

### 5.10.3 适用场景

| 场景 | 宏函数 | 普通函数 |
|------|--------|----------|
| 简单表达式组合 | ✓ 推荐 | 可行 |
| 复杂计算逻辑 | 可行 | ✓ 推荐 |
| 需要状态管理 | ✗ 不支持 | ✓ 支持 |
| 用户可定义 | ✓ SQL 级别 | 需要 C++ |
| PostgreSQL 兼容 | ✓ 推荐 | 可行 |

## 5.11 本章小结

本章深入分析了 DuckDB 的宏函数系统：

1. **宏类型体系**：支持标量宏（ScalarMacroFunction）和表宏（TableMacroFunction），分别用于表达式级和查询级的抽象

2. **参数系统**：支持位置参数、命名参数、默认参数和类型化参数，提供灵活的调用方式

3. **绑定期展开**：宏在绑定阶段通过 `UnfoldMacroExpression` 展开，实现零调用开销

4. **DummyBinding 机制**：通过虚拟绑定表实现参数名到参数表达式的映射和替换

5. **重载解析**：类似普通函数的重载解析，基于类型转换代价选择最佳匹配

6. **内置宏系统**：通过 `internal_macros` 静态表和 `DefaultFunctionGenerator` 懒加载实现大量内置函数

7. **Lambda 交互**：特殊处理 Lambda 参数以避免错误替换

宏函数作为 SQL 级别的抽象机制，为 DuckDB 提供了强大的可扩展性和 PostgreSQL 兼容性，同时保持了执行效率。

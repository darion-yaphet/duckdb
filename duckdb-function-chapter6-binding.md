# 第六章：函数绑定与重载解析

## 6.1 函数绑定概述

函数绑定是 DuckDB 查询处理流程中的关键步骤，负责将 SQL 查询中的函数调用与具体的函数实现关联起来。这个过程类似于 C++ 的函数重载解析，但需要处理 SQL 的动态类型特性和隐式类型转换。

### 6.1.1 绑定的核心任务

```
┌──────────────────────────────────────────────────────────────────────┐
│                      函数绑定核心任务                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 重载解析                                                          │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ 从 FunctionSet 中选择与参数类型最匹配的函数                  │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  2. 类型转换                                                          │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ 计算隐式类型转换代价，必要时插入 CAST 表达式                  │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  3. 模板类型解析                                                      │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ 将 TEMPLATE 类型解析为具体类型                              │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  4. 错误诊断                                                          │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ 无匹配或多匹配时生成清晰的错误信息                           │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.1.2 FunctionBinder 类结构

```cpp
// src/include/duckdb/function/function_binder.hpp
class FunctionBinder {
public:
    explicit FunctionBinder(Binder &binder);
    explicit FunctionBinder(ClientContext &context);

    optional_ptr<Binder> binder;
    ClientContext &context;

public:
    //! 标量函数重载解析
    optional_idx BindFunction(const string &name, ScalarFunctionSet &functions,
                              const vector<LogicalType> &arguments, ErrorData &error);

    //! 聚合函数重载解析
    optional_idx BindFunction(const string &name, AggregateFunctionSet &functions,
                              const vector<LogicalType> &arguments, ErrorData &error);

    //! 表函数重载解析
    optional_idx BindFunction(const string &name, TableFunctionSet &functions,
                              const vector<LogicalType> &arguments, ErrorData &error);

    //! Pragma 函数重载解析
    optional_idx BindFunction(const string &name, PragmaFunctionSet &functions,
                              vector<Value> &parameters, ErrorData &error);

    //! 完整的标量函数绑定
    unique_ptr<Expression> BindScalarFunction(ScalarFunctionCatalogEntry &func,
                                              vector<unique_ptr<Expression>> children,
                                              ErrorData &error, bool is_operator = false);

    //! 完整的聚合函数绑定
    unique_ptr<BoundAggregateExpression> BindAggregateFunction(
        AggregateFunction bound_function,
        vector<unique_ptr<Expression>> children,
        unique_ptr<Expression> filter = nullptr,
        AggregateType aggr_type = AggregateType::NON_DISTINCT);

    //! 类型转换
    void CastToFunctionArguments(SimpleFunction &function,
                                 vector<unique_ptr<Expression>> &children);

    //! 模板类型解析
    void ResolveTemplateTypes(BaseScalarFunction &bound_function,
                              const vector<unique_ptr<Expression>> &children);

    //! 检查模板类型是否已解析
    void CheckTemplateTypesResolved(const BaseScalarFunction &bound_function);

private:
    //! 计算绑定代价
    optional_idx BindFunctionCost(const SimpleFunction &func,
                                  const vector<LogicalType> &arguments);
    optional_idx BindVarArgsFunctionCost(const SimpleFunction &func,
                                         const vector<LogicalType> &arguments);
};
```

## 6.2 重载解析算法

### 6.2.1 候选函数筛选

重载解析分为两个阶段：筛选和选择。

```cpp
// src/function/function_binder.cpp
template <class T>
vector<idx_t> FunctionBinder::BindFunctionsFromArguments(
    const string &name, FunctionSet<T> &functions,
    const vector<LogicalType> &arguments, ErrorData &error) {

    optional_idx best_function;
    idx_t lowest_cost = NumericLimits<idx_t>::Maximum();
    vector<idx_t> candidate_functions;

    // 遍历所有重载版本
    for (idx_t f_idx = 0; f_idx < functions.functions.size(); f_idx++) {
        auto &func = functions.functions[f_idx];

        // 计算绑定代价
        auto bind_cost = BindFunctionCost(func, arguments);

        if (!bind_cost.IsValid()) {
            // 无法自动转换，跳过此候选
            continue;
        }

        auto cost = bind_cost.GetIndex();
        if (cost == lowest_cost) {
            // 代价相同，添加到候选列表
            candidate_functions.push_back(f_idx);
            continue;
        }

        if (cost > lowest_cost) {
            // 代价更高，跳过
            continue;
        }

        // 找到更好的候选
        candidate_functions.clear();
        lowest_cost = cost;
        best_function = f_idx;
    }

    if (!best_function.IsValid()) {
        // 没有找到匹配的函数
        error = ErrorData(BinderException::NoMatchingFunction(
            catalog_name, schema_name, name, arguments, candidates));
        return candidate_functions;
    }

    candidate_functions.push_back(best_function.GetIndex());
    return candidate_functions;
}
```

### 6.2.2 绑定代价计算

代价计算是重载解析的核心：

```cpp
// src/function/function_binder.cpp
optional_idx FunctionBinder::BindFunctionCost(const SimpleFunction &func,
                                              const vector<LogicalType> &arguments) {
    // 处理变参函数
    if (func.HasVarArgs()) {
        return BindVarArgsFunctionCost(func, arguments);
    }

    // 参数数量不匹配
    if (func.arguments.size() != arguments.size()) {
        return optional_idx();
    }

    idx_t cost = 0;
    bool has_parameter = false;

    for (idx_t i = 0; i < arguments.size(); i++) {
        // UNKNOWN 类型（参数占位符）特殊处理
        if (arguments[i].id() == LogicalTypeId::UNKNOWN) {
            has_parameter = true;
            continue;
        }

        // 计算隐式转换代价
        int64_t cast_cost = CastFunctionSet::ImplicitCastCost(
            context, arguments[i], func.arguments[i]);

        if (cast_cost >= 0) {
            cost += idx_t(cast_cost);
        } else {
            // 无法隐式转换
            return optional_idx();
        }
    }

    // 存在参数占位符时返回 0 代价
    if (has_parameter) {
        return 0;
    }

    return cost;
}
```

变参函数的代价计算：

```cpp
optional_idx FunctionBinder::BindVarArgsFunctionCost(const SimpleFunction &func,
                                                      const vector<LogicalType> &arguments) {
    // 参数数量至少要满足非变参部分
    if (arguments.size() < func.arguments.size()) {
        return optional_idx();
    }

    idx_t cost = 0;
    for (idx_t i = 0; i < arguments.size(); i++) {
        // 确定目标类型：固定参数或变参类型
        LogicalType arg_type = i < func.arguments.size()
                                   ? func.arguments[i]
                                   : func.varargs;

        if (arguments[i] == arg_type) {
            continue;  // 精确匹配
        }

        int64_t cast_cost = CastFunctionSet::ImplicitCastCost(
            context, arguments[i], arg_type);

        if (cast_cost >= 0) {
            cost += idx_t(cast_cost);
        } else {
            return optional_idx();  // 无法转换
        }
    }

    return cost;
}
```

### 6.2.3 多候选处理

当存在多个代价相同的候选函数时：

```cpp
template <class T>
optional_idx FunctionBinder::MultipleCandidateException(
    const string &catalog_name, const string &schema_name, const string &name,
    FunctionSet<T> &functions, vector<idx_t> &candidate_functions,
    const vector<LogicalType> &arguments, ErrorData &error) {

    D_ASSERT(functions.functions.size() > 1);

    // 生成调用字符串
    string call_str = Function::CallToString(catalog_name, schema_name, name, arguments);

    // 生成候选函数列表
    string candidate_str;
    for (auto &conf : candidate_functions) {
        T f = functions.GetFunctionByOffset(conf);
        candidate_str += "\t" + f.ToString() + "\n";
    }

    // 返回错误信息
    error = ErrorData(
        ExceptionType::BINDER,
        StringUtil::Format(
            "Could not choose a best candidate function for the function call \"%s\". "
            "In order to select one, please add explicit type casts.\n"
            "\tCandidate functions:\n%s",
            call_str, candidate_str));

    return optional_idx();
}
```

## 6.3 隐式类型转换规则

### 6.3.1 CastRules 类

`CastRules` 定义了所有类型之间的隐式转换规则：

```cpp
// src/include/duckdb/function/cast_rules.hpp
class CastRules {
public:
    //! 返回从 from 到 to 的隐式转换代价，-1 表示不可转换
    static int64_t ImplicitCast(const LogicalType &from, const LogicalType &to);
};
```

### 6.3.2 目标类型代价

不同目标类型有不同的偏好程度：

```cpp
// src/function/cast_rules.cpp
static int64_t TargetTypeCost(const LogicalType &type) {
    switch (type.id()) {
    case LogicalTypeId::BIGINT:
        return 101;   // 首选整数类型
    case LogicalTypeId::INTEGER:
        return 102;
    case LogicalTypeId::HUGEINT:
        return 103;
    case LogicalTypeId::DOUBLE:
        return 104;   // 浮点类型代价稍高
    case LogicalTypeId::DECIMAL:
        return 105;
    case LogicalTypeId::BIGNUM:
        return 106;
    case LogicalTypeId::TIMESTAMP_NS:
        return 119;   // 时间戳类型
    case LogicalTypeId::TIMESTAMP:
        return 120;
    case LogicalTypeId::VARCHAR:
        return 149;   // VARCHAR 代价最高
    case LogicalTypeId::STRUCT:
    case LogicalTypeId::MAP:
    case LogicalTypeId::LIST:
    case LogicalTypeId::UNION:
    case LogicalTypeId::ARRAY:
        return 160;   // 复合类型
    case LogicalTypeId::TEMPLATE:
        return 1000000;  // 模板类型代价极高
    default:
        return 110;
    }
}
```

### 6.3.3 数值类型转换规则

```cpp
// 整数类型转换链
static int64_t ImplicitCastTinyint(const LogicalType &to) {
    switch (to.id()) {
    case LogicalTypeId::SMALLINT:
    case LogicalTypeId::INTEGER:
    case LogicalTypeId::BIGINT:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
    case LogicalTypeId::DECIMAL:
    case LogicalTypeId::BIGNUM:
        return TargetTypeCost(to);
    default:
        return -1;
    }
}

// 更大的整数类型转换更受限
static int64_t ImplicitCastBigint(const LogicalType &to) {
    switch (to.id()) {
    case LogicalTypeId::FLOAT:
    case LogicalTypeId::DOUBLE:
    case LogicalTypeId::HUGEINT:
    case LogicalTypeId::DECIMAL:
    case LogicalTypeId::BIGNUM:
        return TargetTypeCost(to);
    default:
        return -1;
    }
}
```

类型转换图解：

```
┌──────────────────────────────────────────────────────────────────────┐
│                      隐式类型转换链                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   整数类型（向上转换）:                                               │
│   TINYINT → SMALLINT → INTEGER → BIGINT → HUGEINT                   │
│      ↓          ↓          ↓         ↓         ↓                     │
│      └──────────┴──────────┴─────────┴─────────┘                     │
│                           ↓                                          │
│                     FLOAT → DOUBLE                                   │
│                           ↓                                          │
│                        DECIMAL                                       │
│                                                                      │
│   时间类型:                                                          │
│   DATE → TIMESTAMP → TIMESTAMP_TZ                                   │
│                ↓                                                     │
│          TIMESTAMP_NS                                                │
│                                                                      │
│   特殊类型:                                                          │
│   SQLNULL → (任意类型)                                               │
│   UNKNOWN → (任意类型, 代价 0)                                        │
│   STRING_LITERAL → (任意有效类型, 低代价)                             │
│   INTEGER_LITERAL → (匹配的整数类型, 代价 0)                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.3.4 复合类型转换规则

```cpp
int64_t CastRules::ImplicitCast(const LogicalType &from, const LogicalType &to) {
    // LIST → LIST：子类型必须可转换
    if (from.id() == LogicalTypeId::LIST && to.id() == LogicalTypeId::LIST) {
        auto child_cost = ImplicitCast(
            ListType::GetChildType(from),
            ListType::GetChildType(to));
        if (child_cost >= 1) {
            child_cost--;  // 偏好 LIST → LIST 而非 LIST → VARCHAR
        }
        return child_cost;
    }

    // ARRAY → ARRAY：子类型可转换且大小匹配
    if (from.id() == LogicalTypeId::ARRAY && to.id() == LogicalTypeId::ARRAY) {
        auto from_size = ArrayType::GetSize(from);
        auto to_size = ArrayType::GetSize(to);
        auto to_is_any_size = ArrayType::IsAnySize(to);

        if (from_size == to_size || to_is_any_size) {
            return ImplicitCast(ArrayType::GetChildType(from),
                                ArrayType::GetChildType(to));
        }
        return -1;
    }

    // ARRAY → LIST：允许转换
    if (from.id() == LogicalTypeId::ARRAY && to.id() == LogicalTypeId::LIST) {
        auto child_cost = ImplicitCast(
            ArrayType::GetChildType(from),
            ListType::GetChildType(to));
        return child_cost >= 0 ? child_cost + 1 : -1;
    }

    // STRUCT → STRUCT：字段递归匹配
    if (from.id() == LogicalTypeId::STRUCT && to.id() == LogicalTypeId::STRUCT) {
        // 命名结构体按名称匹配，未命名按位置匹配
        // ...
    }

    // UNION → UNION：源成员必须是目标成员的子集
    if (from.id() == LogicalTypeId::UNION && to.id() == LogicalTypeId::UNION) {
        // ...
    }
}
```

### 6.3.5 字面量特殊处理

```cpp
int64_t CastRules::ImplicitCast(const LogicalType &from, const LogicalType &to) {
    // NULL 可以转换为任意类型
    if (from.id() == LogicalTypeId::SQLNULL) {
        return TargetTypeCost(to);
    }

    // UNKNOWN（参数占位符）零代价转换
    if (from.id() == LogicalTypeId::UNKNOWN) {
        return 0;
    }

    // 字符串字面量：低代价转换
    if (from.id() == LogicalTypeId::STRING_LITERAL) {
        if (!LogicalTypeIsValid(to)) {
            return -1;  // 目标类型不完整
        }
        if (to.id() == LogicalTypeId::VARCHAR) {
            return 1;   // 转 VARCHAR 代价最低
        }
        return 20;      // 其他类型代价稍高
    }

    // 整数字面量：优先匹配合适的类型
    if (from.id() == LogicalTypeId::INTEGER_LITERAL) {
        // 如果字面量值恰好适合目标类型
        if (IntegerLiteral::FitsInType(from, to)) {
            return TargetTypeCost(to) - 90;  // 给予奖励
        }
        // 否则使用字面量的首选类型规则
        return CastRules::ImplicitCast(IntegerLiteral::GetType(from), to);
    }
}
```

## 6.4 模板类型系统

### 6.4.1 模板类型概述

DuckDB 支持类似 C++ 的模板类型，允许定义泛型函数：

```cpp
// 使用模板类型定义泛型函数
ScalarFunction("array_length", {LogicalType::TEMPLATE}, LogicalType::BIGINT,
               ArrayLengthFunction);
```

### 6.4.2 模板类型推导

```cpp
// src/function/function_binder.cpp
void FunctionBinder::ResolveTemplateTypes(BaseScalarFunction &bound_function,
                                          const vector<unique_ptr<Expression>> &children) {
    case_insensitive_map_t<vector<LogicalType>> bindings;
    vector<reference<LogicalType>> to_substitute;

    // 1. 从参数推导模板类型
    for (idx_t i = 0; i < bound_function.arguments.size(); i++) {
        auto &param = bound_function.arguments[i];

        if (param.IsTemplated()) {
            auto actual = ExpressionBinder::GetExpressionReturnType(*children[i]);
            InferTemplateType(context, param, actual, bindings,
                              *children[i], bound_function);
            to_substitute.emplace_back(param);
        }
    }

    // 2. 处理模板化的变参
    if (bound_function.varargs.IsTemplated()) {
        for (idx_t i = bound_function.arguments.size(); i < children.size(); i++) {
            auto actual = ExpressionBinder::GetExpressionReturnType(*children[i]);
            InferTemplateType(context, bound_function.varargs, actual, bindings,
                              *children[i], bound_function);
        }
        to_substitute.emplace_back(bound_function.varargs);
    }

    // 3. 处理模板化的返回类型
    if (bound_function.GetReturnType().IsTemplated()) {
        to_substitute.emplace_back(bound_function.GetReturnType());
    }

    // 4. 替换所有模板类型为具体类型
    for (auto &templated_type : to_substitute) {
        SubstituteTemplateType(templated_type, bindings, bound_function.name);
    }
}
```

### 6.4.3 模板类型推断逻辑

```cpp
static void InferTemplateType(ClientContext &context,
                              const LogicalType &source,
                              const LogicalType &target,
                              case_insensitive_map_t<vector<LogicalType>> &bindings,
                              const Expression &current_expr,
                              const BaseScalarFunction &function) {

    // 目标是 UNKNOWN/SQLNULL：将模板绑定为该类型
    if (target.id() == LogicalTypeId::UNKNOWN ||
        target.id() == LogicalTypeId::SQLNULL) {
        TypeVisitor::Contains(source, [&](const LogicalType &child) {
            if (child.id() == LogicalTypeId::TEMPLATE) {
                const auto index = TemplateType::GetName(child);
                if (bindings.find(index) == bindings.end()) {
                    bindings[index] = {target.id()};
                }
            }
            return false;
        });
        return;
    }

    // 源是模板类型：绑定或统一
    if (source.id() == LogicalTypeId::TEMPLATE) {
        const auto &index = TemplateType::GetName(source);
        auto it = bindings.find(index);

        if (it == bindings.end()) {
            // 首次绑定
            bindings[index] = {target};
            return;
        }

        if (it->second.back() == target) {
            // 已绑定为相同类型
            return;
        }

        // 尝试类型统一（提升）
        LogicalType result;
        if (LogicalType::TryGetMaxLogicalType(context, it->second.back(), target, result)) {
            if (it->second.back() != result) {
                it->second.push_back(target);
                it->second.push_back(std::move(result));
            }
            return;
        }

        // 类型不兼容
        throw BinderException(...);
    }

    // 嵌套类型：递归推断
    if (source.IsNested() && target.IsNested()) {
        switch (source.id()) {
        case LogicalTypeId::LIST:
        case LogicalTypeId::ARRAY:
            InferTemplateType(context,
                              ListType::GetChildType(source),
                              ListType::GetChildType(target),
                              bindings, current_expr, function);
            break;
        case LogicalTypeId::MAP:
            InferTemplateType(context, MapType::KeyType(source),
                              MapType::KeyType(target), ...);
            InferTemplateType(context, MapType::ValueType(source),
                              MapType::ValueType(target), ...);
            break;
        // ...
        }
    }
}
```

### 6.4.4 模板类型替换

```cpp
static void SubstituteTemplateType(LogicalType &type,
                                   case_insensitive_map_t<vector<LogicalType>> &bindings,
                                   const string &function_name) {
    type = TypeVisitor::VisitReplace(type, [&](const LogicalType &t) -> LogicalType {
        if (t.id() == LogicalTypeId::TEMPLATE) {
            const auto index = TemplateType::GetName(t);
            auto it = bindings.find(index);
            if (it != bindings.end()) {
                // 返回规范化的具体类型
                return LogicalType::NormalizeType(it->second.back());
            }
            // 未解析的模板类型在后续检查中报错
        }
        return t;
    });
}
```

## 6.5 类型转换插入

### 6.5.1 CastToFunctionArguments

在选定函数后，需要为参数插入必要的类型转换：

```cpp
// src/function/function_binder.cpp
void FunctionBinder::CastToFunctionArguments(SimpleFunction &function,
                                             vector<unique_ptr<Expression>> &children) {
    // 预处理 ANY 类型
    for (auto &arg : function.arguments) {
        PrepareTypeForCast(arg);
    }
    PrepareTypeForCast(function.varargs);

    for (idx_t i = 0; i < children.size(); i++) {
        auto target_type = i < function.arguments.size()
                               ? function.arguments[i]
                               : function.varargs;

        // 验证目标类型
        if (target_type.id() == LogicalTypeId::STRING_LITERAL ||
            target_type.id() == LogicalTypeId::INTEGER_LITERAL) {
            throw InternalException("Function returned literal type");
        }
        target_type.Verify();

        // Lambda 子表达式不转换
        if (children[i]->return_type.id() == LogicalTypeId::LAMBDA) {
            continue;
        }

        // 检查是否需要转换
        auto cast_result = RequiresCast(children[i]->return_type, target_type);

        if (cast_result == LogicalTypeComparisonResult::DIFFERENT_TYPES) {
            // 插入 CAST 表达式
            children[i] = BoundCastExpression::AddCastToType(
                context, std::move(children[i]), target_type);
        }
    }
}
```

### 6.5.2 类型比较逻辑

```cpp
static LogicalTypeComparisonResult RequiresCast(const LogicalType &source_type,
                                                 const LogicalType &target_type) {
    // ANY 类型不需要转换
    if (target_type.id() == LogicalTypeId::ANY) {
        return LogicalTypeComparisonResult::TARGET_IS_ANY;
    }

    // 相同类型不需要转换
    if (source_type == target_type) {
        return LogicalTypeComparisonResult::IDENTICAL_TYPE;
    }

    // 嵌套类型递归检查
    if (source_type.id() == LogicalTypeId::LIST &&
        target_type.id() == LogicalTypeId::LIST) {
        return RequiresCast(ListType::GetChildType(source_type),
                            ListType::GetChildType(target_type));
    }

    if (source_type.id() == LogicalTypeId::ARRAY &&
        target_type.id() == LogicalTypeId::ARRAY) {
        return RequiresCast(ArrayType::GetChildType(source_type),
                            ArrayType::GetChildType(target_type));
    }

    return LogicalTypeComparisonResult::DIFFERENT_TYPES;
}
```

## 6.6 完整绑定流程

### 6.6.1 标量函数绑定

```cpp
// src/function/function_binder.cpp
unique_ptr<Expression> FunctionBinder::BindScalarFunction(
    ScalarFunction bound_function,
    vector<unique_ptr<Expression>> children,
    bool is_operator,
    optional_ptr<Binder> binder) {

    // 1. 解析模板类型
    ResolveTemplateTypes(bound_function, children);

    // 2. 调用 bind 回调
    unique_ptr<FunctionData> bind_info;
    if (bound_function.HasBindCallback()) {
        bind_info = bound_function.GetBindCallback()(context, bound_function, children);
    } else if (bound_function.HasBindExtendedCallback()) {
        ScalarFunctionBindInput bind_input(*binder);
        bind_info = bound_function.GetBindExtendedCallback()(
            bind_input, bound_function, children);
    }

    // 3. 检查模板类型是否已解析
    CheckTemplateTypesResolved(bound_function);

    // 4. 处理排序规则
    HandleCollations(context, bound_function, children);

    // 5. 插入类型转换
    CastToFunctionArguments(bound_function, children);

    // 6. 创建绑定的函数表达式
    auto return_type = bound_function.GetReturnType();
    auto result_func = make_uniq<BoundFunctionExpression>(
        std::move(return_type), std::move(bound_function),
        std::move(children), std::move(bind_info), is_operator);

    // 7. 处理表达式绑定回调
    if (result_func->function.HasBindExpressionCallback()) {
        FunctionBindExpressionInput input(context, result_func->bind_info.get(),
                                          result_func->children);
        auto result = result_func->function.GetBindExpressionCallback()(input);
        if (result) {
            return result;
        }
    }

    return std::move(result_func);
}
```

### 6.6.2 NULL 处理优化

```cpp
unique_ptr<Expression> FunctionBinder::BindScalarFunction(
    ScalarFunctionCatalogEntry &func,
    vector<unique_ptr<Expression>> children,
    ErrorData &error, bool is_operator, optional_ptr<Binder> binder) {

    // 选择最佳匹配
    auto best_function = BindFunction(func.name, func.functions, children, error);
    if (!best_function.IsValid()) {
        return nullptr;
    }

    auto bound_function = func.functions.GetFunctionByOffset(best_function.GetIndex());

    // NULL 处理优化
    const auto return_type_if_null = bound_function.GetReturnType().IsComplete()
                                         ? bound_function.GetReturnType()
                                         : LogicalType::SQLNULL;

    if (bound_function.GetNullHandling() == FunctionNullHandling::DEFAULT_NULL_HANDLING) {
        for (auto &child : children) {
            // 如果参数类型是 SQLNULL，直接返回 NULL
            if (child->return_type == LogicalTypeId::SQLNULL) {
                return make_uniq<BoundConstantExpression>(Value(return_type_if_null));
            }

            // 如果参数是常量且值为 NULL
            if (!child->IsFoldable()) {
                continue;
            }
            Value result;
            if (!ExpressionExecutor::TryEvaluateScalar(context, *child, result)) {
                continue;
            }
            if (result.IsNull()) {
                return make_uniq<BoundConstantExpression>(Value(return_type_if_null));
            }
        }
    }

    return BindScalarFunction(bound_function, std::move(children), is_operator, binder);
}
```

### 6.6.3 聚合函数绑定

```cpp
unique_ptr<BoundAggregateExpression> FunctionBinder::BindAggregateFunction(
    AggregateFunction bound_function,
    vector<unique_ptr<Expression>> children,
    unique_ptr<Expression> filter,
    AggregateType aggr_type) {

    // 1. 解析模板类型
    ResolveTemplateTypes(bound_function, children);

    // 2. 调用 bind 回调
    unique_ptr<FunctionData> bind_info;
    if (bound_function.HasBindCallback()) {
        bind_info = bound_function.GetBindCallback()(context, bound_function, children);
        // bind 可能移除了一些参数
        children.resize(MinValue(bound_function.arguments.size(), children.size()));
    }

    // 3. 检查模板类型
    CheckTemplateTypesResolved(bound_function);

    // 4. 插入类型转换
    CastToFunctionArguments(bound_function, children);

    // 5. 创建绑定的聚合表达式
    return make_uniq<BoundAggregateExpression>(
        std::move(bound_function), std::move(children),
        std::move(filter), std::move(bind_info), aggr_type);
}
```

## 6.7 排序规则处理

### 6.7.1 排序规则传播

```cpp
static void HandleCollations(ClientContext &context, ScalarFunction &bound_function,
                             vector<unique_ptr<Expression>> &children) {
    switch (bound_function.GetCollationHandling()) {
    case FunctionCollationHandling::IGNORE_COLLATIONS:
        // 忽略排序规则
        break;

    case FunctionCollationHandling::PROPAGATE_COLLATIONS:
        // 传播排序规则到返回类型
        PropagateCollations(context, bound_function, children);
        break;

    case FunctionCollationHandling::PUSH_COMBINABLE_COLLATIONS:
        // 传播并下推可组合的排序规则
        PushCollations(context, bound_function, children, CollationType::COMBINABLE_COLLATIONS);
        break;

    default:
        throw InternalException("Unrecognized collation handling");
    }
}
```

### 6.7.2 排序规则提取

```cpp
static string ExtractCollation(const vector<unique_ptr<Expression>> &children) {
    string collation;
    for (auto &arg : children) {
        if (!RequiresCollationPropagation(arg->return_type)) {
            continue;  // 非 VARCHAR 列
        }

        auto child_collation = StringType::GetCollation(arg->return_type);
        if (collation.empty()) {
            collation = child_collation;
        } else if (!child_collation.empty() && collation != child_collation) {
            throw BinderException("Cannot combine types with different collation!");
        }
    }
    return collation;
}
```

## 6.8 绑定流程图解

```
┌──────────────────────────────────────────────────────────────────────┐
│                      函数绑定完整流程                                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SQL 查询: SELECT upper(name) FROM users                             │
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 1. 提取参数类型                                                  ││
│  │    arguments = [VARCHAR]                                        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 2. 遍历 FunctionSet 中的所有重载                                 ││
│  │    upper(VARCHAR) → VARCHAR  ← 精确匹配，代价 0                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 3. 选择代价最低的候选                                            ││
│  │    best_function = upper(VARCHAR) → VARCHAR                     ││
│  └─────────────────────────────────────────────────────────────────┘│
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 4. 调用 bind 回调（如果有）                                      ││
│  │    bind_info = function.bind(context, function, children)       ││
│  └─────────────────────────────────────────────────────────────────┘│
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 5. 插入必要的类型转换                                            ││
│  │    CastToFunctionArguments(function, children)                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                     │                                                │
│                     ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 6. 创建 BoundFunctionExpression                                 ││
│  │    return_type = VARCHAR                                        ││
│  │    function = upper                                             ││
│  │    children = [BoundColumnRefExpression(name)]                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## 6.9 错误诊断

### 6.9.1 无匹配错误

```cpp
// 当没有找到匹配的函数时
error = ErrorData(BinderException::NoMatchingFunction(
    catalog_name, schema_name, name, arguments, candidates));

// 生成的错误信息示例：
// No function matches the given name and argument types 'add(VARCHAR, INTEGER)'.
// Candidates:
//   add(INTEGER, INTEGER) -> INTEGER
//   add(DOUBLE, DOUBLE) -> DOUBLE
```

### 6.9.2 多候选错误

```cpp
// 当存在多个代价相同的候选时
error = ErrorData(
    ExceptionType::BINDER,
    "Could not choose a best candidate function for the function call 'foo(1, 2)'. "
    "In order to select one, please add explicit type casts.\n"
    "    Candidate functions:\n"
    "        foo(INTEGER, INTEGER) -> INTEGER\n"
    "        foo(BIGINT, BIGINT) -> BIGINT\n");
```

### 6.9.3 模板类型未解析错误

```cpp
void FunctionBinder::CheckTemplateTypesResolved(const BaseScalarFunction &bound_function) {
    for (const auto &arg : bound_function.arguments) {
        VerifyTemplateType(arg, bound_function.name);
    }
    VerifyTemplateType(bound_function.varargs, bound_function.name);
    VerifyTemplateType(bound_function.GetReturnType(), bound_function.name);
}

static void VerifyTemplateType(const LogicalType &type, const string &function_name) {
    TypeVisitor::Contains(type, [&](const LogicalType &type) {
        if (type.id() == LogicalTypeId::TEMPLATE) {
            throw BinderException(
                "Function '%s' has a template parameter type '%s' "
                "that could not be resolved to a concrete type",
                function_name, TemplateType::GetName(type));
        }
        return false;
    });
}
```

## 6.10 本章小结

本章深入分析了 DuckDB 的函数绑定与重载解析机制：

1. **FunctionBinder 类**：统一处理标量函数、聚合函数、表函数和 Pragma 函数的绑定

2. **重载解析算法**：
   - 遍历所有候选函数计算绑定代价
   - 选择代价最低的候选
   - 处理多候选歧义情况

3. **类型转换规则**：
   - `CastRules` 定义完整的隐式转换规则
   - 目标类型代价决定了转换优先级
   - 支持数值、时间、复合类型的转换链

4. **模板类型系统**：
   - 支持泛型函数定义
   - 从实际参数推导模板类型
   - 支持嵌套类型的模板推导

5. **类型转换插入**：
   - `CastToFunctionArguments` 在需要时插入 CAST 表达式
   - 处理 ANY 类型和 Lambda 表达式特例

6. **完整绑定流程**：
   - 模板解析 → bind 回调 → 模板检查 → 排序规则 → 类型转换 → 创建表达式

7. **排序规则处理**：
   - 支持忽略、传播、下推三种模式
   - 确保字符串比较的一致性

函数绑定是连接 SQL 语法和底层函数实现的桥梁，其设计直接影响用户体验和类型安全性。

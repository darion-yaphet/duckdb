# DuckDB 查询优化器深度解析 - 第一章：优化器架构与表达式重写

## 1.1 优化器概述

DuckDB 的查询优化器负责将逻辑计划转换为更高效的等价形式。优化器采用基于规则的多阶段优化架构，包含 31 个内置优化阶段，按照精心设计的顺序依次执行。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DuckDB 优化器流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  逻辑计划输入 (LogicalOperator Tree)                                │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────┐                   │
│  │ 扩展预优化 (pre_optimize_function)          │                   │
│  └────────────────────┬────────────────────────┘                   │
│                       │                                             │
│                       ▼                                             │
│  ┌─────────────────────────────────────────────┐                   │
│  │ 内置优化器 (RunBuiltInOptimizers)           │                   │
│  │                                             │                   │
│  │  1. ExpressionRewriter (表达式重写)         │                   │
│  │  2. CTE Inlining                            │                   │
│  │  3. Filter Pushdown/Pullup                  │                   │
│  │  4. Join Order Optimization                 │                   │
│  │  5. Column Lifetime Analysis                │                   │
│  │  6. Statistics Propagation                  │                   │
│  │  ... (共 31 个阶段)                         │                   │
│  └────────────────────┬────────────────────────┘                   │
│                       │                                             │
│                       ▼                                             │
│  ┌─────────────────────────────────────────────┐                   │
│  │ 扩展后优化 (optimize_function)              │                   │
│  └────────────────────┬────────────────────────┘                   │
│                       │                                             │
│                       ▼                                             │
│  优化后的逻辑计划                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 源码位置

| 组件 | 文件路径 |
|------|----------|
| Optimizer 类 | `src/optimizer/optimizer.cpp` |
| ExpressionRewriter | `src/optimizer/expression_rewriter.cpp` |
| Rule 基类 | `src/include/duckdb/optimizer/rule.hpp` |
| ExpressionMatcher | `src/include/duckdb/optimizer/matcher/expression_matcher.hpp` |
| 各种规则实现 | `src/optimizer/rule/*.cpp` |

---

## 1.2 Optimizer 类设计

`Optimizer` 类是优化器的入口点，负责协调所有优化阶段的执行。

### 1.2.1 类结构

```cpp
class Optimizer {
public:
    Optimizer(Binder &binder, ClientContext &context);

    // 主入口：执行所有优化
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan);

    // 检查优化器是否被禁用
    bool OptimizerDisabled(OptimizerType type);

    // 辅助函数：绑定标量函数（用于规则创建新表达式）
    unique_ptr<Expression> BindScalarFunction(const string &name, ...);

private:
    ClientContext &context;
    Binder &binder;
    ExpressionRewriter rewriter;  // 表达式重写器
    unique_ptr<LogicalOperator> plan;  // 当前计划

    void RunBuiltInOptimizers();
    void RunOptimizer(OptimizerType type, const std::function<void()> &callback);
    void Verify(LogicalOperator &op);
};
```

### 1.2.2 优化器初始化

在构造函数中，`Optimizer` 向 `ExpressionRewriter` 注册所有表达式重写规则：

```cpp
Optimizer::Optimizer(Binder &binder, ClientContext &context)
    : context(context), binder(binder), rewriter(context) {
    // 规则注册顺序很重要！
    rewriter.rules.push_back(make_uniq<ConstantOrderNormalizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DistributivityRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ArithmeticSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<CaseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConjunctionSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DatePartSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DateTruncSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InClauseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EqualOrNullSimplification>(rewriter));
    rewriter.rules.push_back(make_uniq<MoveConstantsRule>(rewriter));
    rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EnumComparisonRule>(rewriter));
    rewriter.rules.push_back(make_uniq<JoinDependentFilterRule>(rewriter));
    rewriter.rules.push_back(make_uniq<TimeStampComparison>(context, rewriter));
}
```

### 1.2.3 优化执行流程

`Optimize()` 方法是优化器的主入口：

```cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan_p) {
    // 1. 验证输入计划
    Verify(*plan_p);
    this->plan = std::move(plan_p);

    // 2. 运行扩展的预优化函数
    for (auto &pre_optimizer_extension : DBConfig::GetConfig(context).optimizer_extensions) {
        RunOptimizer(OptimizerType::EXTENSION, [&]() {
            if (pre_optimizer_extension.pre_optimize_function) {
                pre_optimizer_extension.pre_optimize_function(input, plan);
            }
        });
    }

    // 3. 运行内置优化器
    RunBuiltInOptimizers();

    // 4. 运行扩展的后优化函数
    for (auto &optimizer_extension : DBConfig::GetConfig(context).optimizer_extensions) {
        RunOptimizer(OptimizerType::EXTENSION, [&]() {
            if (optimizer_extension.optimize_function) {
                optimizer_extension.optimize_function(input, plan);
            }
        });
    }

    // 5. 最终验证
    Planner::VerifyPlan(context, plan);
    return std::move(plan);
}
```

### 1.2.4 RunOptimizer 封装

每个优化阶段都通过 `RunOptimizer()` 执行，提供：
- 禁用检查
- 性能分析集成
- 优化后验证

```cpp
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
    // 检查是否被禁用
    if (OptimizerDisabled(type)) {
        return;
    }

    // 开始性能分析
    auto &profiler = QueryProfiler::Get(context);
    profiler.StartPhase(MetricsUtils::GetOptimizerMetricByType(type));

    // 执行优化
    callback();

    // 结束性能分析
    profiler.EndPhase();

    // 验证优化后的计划
    if (plan) {
        Verify(*plan);
    }
}
```

### 1.2.5 跳过简单语句

对于某些简单语句，优化器会跳过优化以提高性能：

```cpp
void Optimizer::RunBuiltInOptimizers() {
    switch (plan->type) {
    case LogicalOperatorType::LOGICAL_TRANSACTION:
    case LogicalOperatorType::LOGICAL_PRAGMA:
    case LogicalOperatorType::LOGICAL_SET:
    case LogicalOperatorType::LOGICAL_ATTACH:
    case LogicalOperatorType::LOGICAL_UPDATE_EXTENSIONS:
    case LogicalOperatorType::LOGICAL_CREATE_SECRET:
    case LogicalOperatorType::LOGICAL_EXTENSION_OPERATOR:
        // 跳过优化简单且常见的计划
        if (plan->children.empty()) {
            return;
        }
        break;
    default:
        break;
    }
    // ... 执行优化阶段
}
```

---

## 1.3 优化阶段执行顺序

内置优化器按照以下顺序执行（顺序经过精心设计）：

```
┌────────────────────────────────────────────────────────────────────┐
│                    内置优化阶段执行顺序                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  阶段 1: EXPRESSION_REWRITER     ─── 表达式重写（本章重点）         │
│       │                                                            │
│       ▼                                                            │
│  阶段 2: CTE_INLINING            ─── CTE 内联 (第一次)             │
│       │                                                            │
│       ▼                                                            │
│  阶段 3: SUM_REWRITER            ─── SUM(x+C) 重写                 │
│       │                                                            │
│       ▼                                                            │
│  阶段 4: FILTER_PULLUP           ─── 谓词上拉                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 5: FILTER_PUSHDOWN         ─── 谓词下推                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 6: CTE_FILTER_PUSHER       ─── CTE 过滤下推                  │
│       │                                                            │
│       ▼                                                            │
│  阶段 7: REGEX_RANGE             ─── 正则范围过滤                  │
│       │                                                            │
│       ▼                                                            │
│  阶段 8: IN_CLAUSE               ─── IN 子句重写                   │
│       │                                                            │
│       ▼                                                            │
│  阶段 9: DELIMINATOR             ─── Delim 优化                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 10: CTE_INLINING           ─── CTE 内联 (第二次)             │
│       │                                                            │
│       ▼                                                            │
│  阶段 11: EMPTY_RESULT_PULLUP    ─── 空结果上拉                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 12: JOIN_ORDER             ─── Join 顺序优化                 │
│       │                                                            │
│       ▼                                                            │
│  阶段 13: JOIN_ELIMINATION       ─── Join 消除                     │
│       │                                                            │
│       ▼                                                            │
│  阶段 14: UNNEST_REWRITER        ─── UNNEST 重写                   │
│       │                                                            │
│       ▼                                                            │
│  阶段 15: UNUSED_COLUMNS         ─── 无用列移除                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 16: DUPLICATE_GROUPS       ─── 重复分组移除                  │
│       │                                                            │
│       ▼                                                            │
│  阶段 17: COMMON_SUBEXPRESSIONS  ─── 公共子表达式                  │
│       │                                                            │
│       ▼                                                            │
│  阶段 18: COLUMN_LIFETIME        ─── 列生命周期 (第一次)           │
│       │                                                            │
│       ▼                                                            │
│  阶段 19: BUILD_SIDE_PROBE_SIDE  ─── Build/Probe 选择              │
│       │                                                            │
│       ▼                                                            │
│  阶段 20: COMMON_SUBPLAN         ─── 公共子计划                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 21: LIMIT_PUSHDOWN         ─── Limit 下推                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 22: ROW_GROUP_PRUNER       ─── Row Group 裁剪                │
│       │                                                            │
│       ▼                                                            │
│  阶段 23: SAMPLING_PUSHDOWN      ─── 采样下推                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 24: TOP_N                  ─── TopN 优化                     │
│       │                                                            │
│       ▼                                                            │
│  阶段 25: LATE_MATERIALIZATION   ─── 延迟物化                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 26: STATISTICS_PROPAGATION ─── 统计传播                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 27: TOP_N_WINDOW_ELIMINATION─── TopN 窗口消除                │
│       │                                                            │
│       ▼                                                            │
│  阶段 28: COMMON_AGGREGATE       ─── 公共聚合                      │
│       │                                                            │
│       ▼                                                            │
│  阶段 29: COLUMN_LIFETIME        ─── 列生命周期 (第二次)           │
│       │                                                            │
│       ▼                                                            │
│  阶段 30: REORDER_FILTER         ─── 过滤重排序                    │
│       │                                                            │
│       ▼                                                            │
│  阶段 31: JOIN_FILTER_PUSHDOWN   ─── Join 过滤下推                 │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 为什么某些优化要执行两次？

注意到 `CTE_INLINING` 和 `COLUMN_LIFETIME` 各执行了两次：

1. **CTE_INLINING 两次执行**：
   - 第一次：在谓词优化之前，尝试内联简单的 CTE
   - 第二次：在 DELIMINATOR 之后，处理子查询去相关化产生的新 CTE

2. **COLUMN_LIFETIME 两次执行**：
   - 第一次：在 Build/Probe 选择之前，提供列使用信息
   - 第二次：在所有优化完成后，最终确定列投影

---

## 1.4 ExpressionRewriter 框架

`ExpressionRewriter` 是表达式重写的核心框架，采用访问者模式遍历计划树中的所有表达式，并应用注册的规则进行转换。

### 1.4.1 类结构

```cpp
class ExpressionRewriter : public LogicalOperatorVisitor {
public:
    explicit ExpressionRewriter(ClientContext &context) : context(context) {}

    // 注册的规则列表
    vector<unique_ptr<Rule>> rules;
    ClientContext &context;

    // 访问者模式入口
    void VisitOperator(LogicalOperator &op) override;
    void VisitExpression(unique_ptr<Expression> *expression) override;

    // 工具函数：创建 constant_or_null 表达式
    static unique_ptr<Expression> ConstantOrNull(unique_ptr<Expression> child, Value value);

private:
    // 应用规则到表达式
    static unique_ptr<Expression> ApplyRules(LogicalOperator &op,
                                             const vector<reference<Rule>> &rules,
                                             unique_ptr<Expression> expr,
                                             bool &changes_made,
                                             bool is_root = false);

    optional_ptr<LogicalOperator> op;
    vector<reference<Rule>> to_apply_rules;
};
```

### 1.4.2 执行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ExpressionRewriter 执行流程                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  VisitOperator(op)                                                  │
│       │                                                             │
│       ├──→ 1. 递归访问子算子                                         │
│       │    VisitOperatorChildren(op)                                │
│       │                                                             │
│       ├──→ 2. 准备规则列表                                           │
│       │    to_apply_rules = rules                                   │
│       │                                                             │
│       ├──→ 3. 访问算子的所有表达式                                   │
│       │    VisitOperatorExpressions(op)                             │
│       │         │                                                   │
│       │         └──→ VisitExpression(expression)                    │
│       │              重复直到无变化 {                                │
│       │                  ApplyRules(op, rules, expr)                │
│       │              }                                              │
│       │                                                             │
│       └──→ 4. 特殊处理 LogicalFilter                                 │
│            if (op.type == LOGICAL_FILTER) {                         │
│                filter.SplitPredicates();  // 重新拆分谓词            │
│            }                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.4.3 规则应用算法

`ApplyRules` 是核心算法，实现了规则的匹配和应用：

```cpp
unique_ptr<Expression> ExpressionRewriter::ApplyRules(
    LogicalOperator &op,
    const vector<reference<Rule>> &rules,
    unique_ptr<Expression> expr,
    bool &changes_made,
    bool is_root) {

    // 尝试每个规则
    for (auto &rule : rules) {
        vector<reference<Expression>> bindings;

        // 1. 模式匹配
        if (rule.get().root->Match(*expr, bindings)) {
            bool rule_made_change = false;
            auto alias = expr->alias;

            // 2. 应用规则
            auto result = rule.get().Apply(op, bindings, rule_made_change, is_root);

            if (result) {
                // 规则产生了新表达式
                changes_made = true;
                if (!alias.empty()) {
                    result->alias = std::move(alias);
                }
                // 递归：在新表达式上重新应用规则
                return ApplyRules(op, rules, std::move(result), changes_made);
            } else if (rule_made_change) {
                // 规则修改了现有表达式（通过 bindings 引用）
                changes_made = true;
                return expr;  // 返回让外层重新处理
            }
            // 规则匹配但未做修改，继续尝试下一个规则
        }
    }

    // 3. 没有规则能应用于根节点，递归处理子表达式
    ExpressionIterator::EnumerateChildren(*expr, [&](unique_ptr<Expression> &child) {
        child = ApplyRules(op, rules, std::move(child), changes_made);
    });

    return expr;
}
```

### 1.4.4 不动点迭代

`VisitExpression` 实现了不动点迭代，确保规则被充分应用：

```cpp
void ExpressionRewriter::VisitExpression(unique_ptr<Expression> *expression) {
    bool changes_made;
    do {
        changes_made = false;
        *expression = ApplyRules(*op, to_apply_rules,
                                 std::move(*expression),
                                 changes_made,
                                 true);
    } while (changes_made);  // 持续应用直到稳定
}
```

这种迭代确保了规则之间的交互效果：
- 规则 A 可能创造规则 B 的应用机会
- 规则 B 的结果可能又能被规则 A 优化

---

## 1.5 Rule 抽象与模式匹配

### 1.5.1 Rule 基类

每个优化规则继承自 `Rule` 基类：

```cpp
class Rule {
public:
    explicit Rule(ExpressionRewriter &rewriter) : rewriter(rewriter) {}
    virtual ~Rule() {}

    ExpressionRewriter &rewriter;

    // 模式匹配器：定义规则匹配的表达式模式
    unique_ptr<ExpressionMatcher> root;

    // 获取执行上下文
    ClientContext &GetContext() const;

    // 应用规则：bindings 包含匹配到的表达式
    // 返回值：
    //   - 非空：替换原表达式
    //   - 空 + fixed_point=true：原表达式已被修改
    //   - 空 + fixed_point=false：无变化
    virtual unique_ptr<Expression> Apply(LogicalOperator &op,
                                         vector<reference<Expression>> &bindings,
                                         bool &fixed_point,
                                         bool is_root) = 0;
};
```

### 1.5.2 ExpressionMatcher 层次结构

DuckDB 提供了丰富的表达式匹配器：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ExpressionMatcher 继承层次                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ExpressionMatcher (基类)                                           │
│       │                                                             │
│       ├── ConstantExpressionMatcher                                 │
│       │   匹配 BoundConstantExpression                              │
│       │                                                             │
│       ├── FoldableConstantMatcher                                   │
│       │   匹配可折叠表达式（标量、非聚合、非窗口）                    │
│       │                                                             │
│       ├── StableExpressionMatcher                                   │
│       │   匹配稳定表达式（非 volatile）                              │
│       │                                                             │
│       ├── ComparisonExpressionMatcher                               │
│       │   匹配比较表达式，支持子匹配器                               │
│       │                                                             │
│       ├── ConjunctionExpressionMatcher                              │
│       │   匹配 AND/OR 表达式                                        │
│       │                                                             │
│       ├── FunctionExpressionMatcher                                 │
│       │   匹配函数调用，支持函数名匹配                               │
│       │                                                             │
│       ├── AggregateExpressionMatcher                                │
│       │   匹配聚合函数                                              │
│       │                                                             │
│       ├── CastExpressionMatcher                                     │
│       │   匹配类型转换                                              │
│       │                                                             │
│       ├── CaseExpressionMatcher                                     │
│       │   匹配 CASE 表达式                                          │
│       │                                                             │
│       ├── InClauseExpressionMatcher                                 │
│       │   匹配 IN 表达式                                            │
│       │                                                             │
│       └── ExpressionEqualityMatcher                                 │
│           匹配与给定表达式相等的表达式                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.5.3 ExpressionMatcher 基类

```cpp
class ExpressionMatcher {
public:
    explicit ExpressionMatcher(ExpressionClass type = ExpressionClass::INVALID)
        : expr_class(type) {}

    // 模式匹配：成功则将表达式添加到 bindings
    virtual bool Match(Expression &expr, vector<reference<Expression>> &bindings);

    // 匹配的表达式类（INVALID 表示任意）
    ExpressionClass expr_class;

    // 可选：表达式类型匹配器（如 COMPARE_EQUAL）
    unique_ptr<ExpressionTypeMatcher> expr_type;

    // 可选：返回类型匹配器
    unique_ptr<TypeMatcher> type;
};
```

### 1.5.4 SetMatcher::Policy

对于有子匹配器的复合匹配器，`Policy` 控制匹配策略：

```cpp
enum class Policy {
    INVALID,     // 未设置
    ORDERED,     // 子表达式必须按顺序匹配
    UNORDERED,   // 子表达式可以任意顺序匹配
    SOME,        // 至少一个子表达式匹配每个匹配器
    SOME_ORDERED // SOME + 按顺序
};
```

**示例**：匹配 `a + 0` 或 `0 + a`

```cpp
auto op = make_uniq<FunctionExpressionMatcher>();
op->function = make_uniq<SpecificFunctionMatcher>("+");
op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
op->matchers.push_back(make_uniq<ExpressionMatcher>());
op->policy = SetMatcher::Policy::SOME;  // 允许任意顺序
```

### 1.5.5 FunctionMatcher

用于匹配函数名：

```cpp
// 匹配特定函数
class SpecificFunctionMatcher : public FunctionMatcher {
    string name;  // 如 "date_part"
};

// 匹配多个函数之一
class ManyFunctionMatcher : public FunctionMatcher {
    unordered_set<string> names;  // 如 {"+", "-", "*"}
};
```

---

## 1.6 常量折叠规则 (ConstantFoldingRule)

常量折叠是最基础的优化，将编译时可计算的表达式替换为常量。

### 1.6.1 规则定义

```cpp
// 自定义匹配器：匹配可折叠但非常量的表达式
class ConstantFoldingExpressionMatcher : public FoldableConstantMatcher {
public:
    bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
        // 排除已经是常量的表达式（无需再折叠）
        if (expr.GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
            return false;
        }
        return FoldableConstantMatcher::Match(expr, bindings);
    }
};

ConstantFoldingRule::ConstantFoldingRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    root = make_uniq<ConstantFoldingExpressionMatcher>();
}
```

### 1.6.2 规则应用

```cpp
unique_ptr<Expression> ConstantFoldingRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get();

    // 使用 ExpressionExecutor 计算表达式的值
    Value result_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
        return nullptr;  // 计算失败（如除零）
    }

    // 返回常量表达式
    return make_uniq<BoundConstantExpression>(result_value);
}
```

### 1.6.3 示例

```sql
-- 优化前
SELECT * FROM t WHERE a > 1 + 2 * 3

-- 优化后
SELECT * FROM t WHERE a > 7
```

### 1.6.4 可折叠表达式

`FoldableConstantMatcher` 匹配满足以下条件的表达式：
- 是标量表达式
- 不包含聚合函数
- 不包含窗口函数
- 不包含参数引用
- 不包含子查询

---

## 1.7 算术简化规则 (ArithmeticSimplificationRule)

利用代数恒等式简化算术表达式。

### 1.7.1 规则定义

```cpp
ArithmeticSimplificationRule::ArithmeticSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {
    auto op = make_uniq<FunctionExpressionMatcher>();

    // 匹配：一个常量 + 一个任意表达式
    op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    op->matchers.push_back(make_uniq<ExpressionMatcher>());
    op->policy = SetMatcher::Policy::SOME;  // 顺序不重要

    // 匹配这些函数
    op->function = make_uniq<ManyFunctionMatcher>(
        unordered_set<string> {"+", "-", "*", "//"});

    // 只处理整数类型
    op->type = make_uniq<IntegerTypeMatcher>();
    op->matchers[0]->type = make_uniq<IntegerTypeMatcher>();
    op->matchers[1]->type = make_uniq<IntegerTypeMatcher>();

    root = std::move(op);
}
```

### 1.7.2 规则应用

```cpp
unique_ptr<Expression> ArithmeticSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
    auto &constant = bindings[1].get().Cast<BoundConstantExpression>();

    // 确定常量在哪一侧
    idx_t constant_child = root.children[0].get() == &constant ? 0 : 1;

    // NULL 参与的任何算术运算结果都是 NULL
    if (constant.value.IsNull()) {
        return make_uniq<BoundConstantExpression>(Value(root.return_type));
    }

    auto &func_name = root.function.name;

    if (func_name == "+") {
        if (constant.value == 0) {
            // x + 0 = 0 + x = x
            return std::move(root.children[1 - constant_child]);
        }
    } else if (func_name == "-") {
        if (constant_child == 1 && constant.value == 0) {
            // x - 0 = x （注意：0 - x 不能简化）
            return std::move(root.children[0]);
        }
    } else if (func_name == "*") {
        if (constant.value == 1) {
            // x * 1 = 1 * x = x
            return std::move(root.children[1 - constant_child]);
        } else if (constant.value == 0) {
            // x * 0 = 0（但需要处理 NULL：NULL * 0 = NULL）
            return ExpressionRewriter::ConstantOrNull(
                std::move(root.children[1 - constant_child]),
                Value::Numeric(root.return_type, 0));
        }
    } else if (func_name == "//") {
        if (constant_child == 1) {
            if (constant.value == 1) {
                // x / 1 = x
                return std::move(root.children[0]);
            } else if (constant.value == 0) {
                // x / 0 = NULL
                return make_uniq<BoundConstantExpression>(Value(root.return_type));
            }
        }
    }

    return nullptr;  // 无法简化
}
```

### 1.7.3 ConstantOrNull 辅助函数

处理 `x * 0` 这类情况：当 x 为 NULL 时结果为 NULL，否则为 0。

```cpp
unique_ptr<Expression> ExpressionRewriter::ConstantOrNull(
    unique_ptr<Expression> child, Value value) {
    // 生成 constant_or_null(value, child) 函数调用
    // 语义：如果 child 为 NULL 则返回 NULL，否则返回 value
    ...
}
```

### 1.7.4 示例

```sql
-- 优化前
SELECT a + 0, b * 1, c - 0, d * 0 FROM t

-- 优化后
SELECT a, b, c, constant_or_null(0, d) FROM t
```

---

## 1.8 比较简化规则 (ComparisonSimplificationRule)

优化涉及类型转换的比较表达式。

### 1.8.1 规则定义

```cpp
ComparisonSimplificationRule::ComparisonSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {
    auto op = make_uniq<ComparisonExpressionMatcher>();
    op->matchers.push_back(make_uniq<FoldableConstantMatcher>());
    op->policy = SetMatcher::Policy::SOME;
    root = std::move(op);
}
```

### 1.8.2 规则应用

核心思想：将 `CAST(column) COMP constant` 转换为 `column COMP CAST(constant)`

```cpp
unique_ptr<Expression> ComparisonSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &expr = bindings[0].get().Cast<BoundComparisonExpression>();
    auto &constant_expr = bindings[1].get();

    // 计算常量值
    Value constant_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), constant_expr, constant_value)) {
        return nullptr;
    }

    // NULL 比较（非 IS DISTINCT FROM）返回 NULL
    if (constant_value.IsNull() &&
        expr.GetExpressionType() != ExpressionType::COMPARE_NOT_DISTINCT_FROM &&
        expr.GetExpressionType() != ExpressionType::COMPARE_DISTINCT_FROM) {
        return make_uniq<BoundConstantExpression>(Value(LogicalType::BOOLEAN));
    }

    // 检查是否有 CAST
    auto column_ref_expr = (expr.left.get() != &constant_expr) ? expr.left.get() : expr.right.get();
    if (column_ref_expr->GetExpressionClass() == ExpressionClass::BOUND_CAST) {
        auto &cast_expression = column_ref_expr->Cast<BoundCastExpression>();
        auto target_type = cast_expression.source_type();

        // 检查转换是否可逆
        if (!BoundCastExpression::CastIsInvertible(target_type, cast_expression.return_type)) {
            return nullptr;
        }

        // 尝试将常量转换为目标类型
        Value cast_constant;
        if (!constant_value.TryCastAs(rewriter.context, target_type, cast_constant, ...)) {
            return nullptr;
        }

        // 检查常量转换是否可逆
        if (!cast_constant.IsNull() &&
            !BoundCastExpression::CastIsInvertible(cast_expression.return_type, target_type)) {
            return nullptr;
        }

        // 应用优化：移除 CAST，转换常量
        auto child_expression = std::move(cast_expression.child);
        auto new_constant_expr = make_uniq<BoundConstantExpression>(cast_constant);

        // 更新比较表达式
        if (column_ref_left) {
            expr.left = std::move(child_expression);
            expr.right = std::move(new_constant_expr);
        } else {
            expr.left = std::move(new_constant_expr);
            expr.right = std::move(child_expression);
        }
        changes_made = true;
    }
    return nullptr;
}
```

### 1.8.3 示例

```sql
-- 优化前（假设 id 是 INTEGER 列）
SELECT * FROM t WHERE CAST(id AS BIGINT) = 100

-- 优化后
SELECT * FROM t WHERE id = 100
```

### 1.8.4 可逆性检查

不是所有转换都是可逆的，例如：
- `FLOAT -> INTEGER`：有精度损失，不可逆
- `INTEGER -> BIGINT`：可逆
- `VARCHAR -> INTEGER`：不可逆

---

## 1.9 逻辑简化规则 (ConjunctionSimplificationRule)

简化 AND/OR 表达式中的常量条件。

### 1.9.1 规则定义

```cpp
ConjunctionSimplificationRule::ConjunctionSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {
    auto op = make_uniq<ConjunctionExpressionMatcher>();
    op->matchers.push_back(make_uniq<FoldableConstantMatcher>());
    op->policy = SetMatcher::Policy::SOME;
    root = std::move(op);
}
```

### 1.9.2 规则应用

```cpp
unique_ptr<Expression> ConjunctionSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &conjunction = bindings[0].get().Cast<BoundConjunctionExpression>();
    auto &constant_expr = bindings[1].get();

    // 计算常量值
    Value constant_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), constant_expr, constant_value)) {
        return nullptr;
    }
    constant_value = constant_value.DefaultCastAs(LogicalType::BOOLEAN);

    // NULL 无法简化
    if (constant_value.IsNull()) {
        return nullptr;
    }

    if (conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
        if (!BooleanValue::Get(constant_value)) {
            // FALSE AND x = FALSE
            return make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
        } else {
            // TRUE AND x = x
            return RemoveExpression(conjunction, constant_expr);
        }
    } else {
        // OR
        if (!BooleanValue::Get(constant_value)) {
            // FALSE OR x = x
            return RemoveExpression(conjunction, constant_expr);
        } else {
            // TRUE OR x = TRUE
            return make_uniq<BoundConstantExpression>(Value::BOOLEAN(true));
        }
    }
}

// 从 conjunction 中移除指定表达式
unique_ptr<Expression> ConjunctionSimplificationRule::RemoveExpression(
    BoundConjunctionExpression &conj, const Expression &expr) {

    for (idx_t i = 0; i < conj.children.size(); i++) {
        if (conj.children[i].get() == &expr) {
            conj.children.erase_at(i);
            break;
        }
    }

    // 如果只剩一个子表达式，直接返回它
    if (conj.children.size() == 1) {
        return std::move(conj.children[0]);
    }
    return nullptr;
}
```

### 1.9.3 简化规则总结

| 表达式 | 简化结果 |
|--------|----------|
| `TRUE AND x` | `x` |
| `FALSE AND x` | `FALSE` |
| `TRUE OR x` | `TRUE` |
| `FALSE OR x` | `x` |
| `x AND x` | `x` |
| `NULL AND/OR x` | 不简化 |

### 1.9.4 示例

```sql
-- 优化前
SELECT * FROM t WHERE 1=1 AND a > 5
SELECT * FROM t WHERE 1=0 OR b < 10

-- 优化后
SELECT * FROM t WHERE a > 5
SELECT * FROM t WHERE b < 10
```

---

## 1.10 CASE 简化规则 (CaseSimplificationRule)

简化 CASE WHEN 表达式中的常量条件。

### 1.10.1 规则定义

```cpp
CaseSimplificationRule::CaseSimplificationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    root = make_uniq<CaseExpressionMatcher>();
}
```

### 1.10.2 规则应用

```cpp
unique_ptr<Expression> CaseSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get().Cast<BoundCaseExpression>();

    // 遍历所有 WHEN 子句
    for (idx_t i = 0; i < root.case_checks.size(); i++) {
        auto &case_check = root.case_checks[i];

        if (case_check.when_expr->IsFoldable()) {
            // 计算 WHEN 条件
            auto constant_value = ExpressionExecutor::EvaluateScalar(
                GetContext(), *case_check.when_expr);
            auto condition = constant_value.DefaultCastAs(LogicalType::BOOLEAN);

            if (condition.IsNull() || !BooleanValue::Get(condition)) {
                // 条件恒为 FALSE 或 NULL：移除此分支
                root.case_checks.erase_at(i);
                i--;
            } else {
                // 条件恒为 TRUE：
                // 将 THEN 移动到 ELSE，移除后续所有分支
                root.else_expr = std::move(case_check.then_expr);
                root.case_checks.erase(
                    root.case_checks.begin() + NumericCast<int64_t>(i),
                    root.case_checks.end());
                break;
            }
        }
    }

    // 如果没有 WHEN 子句了，直接返回 ELSE
    if (root.case_checks.empty()) {
        return std::move(root.else_expr);
    }

    return nullptr;
}
```

### 1.10.3 示例

```sql
-- 优化前
SELECT CASE
    WHEN 1=0 THEN 'never'
    WHEN 1=1 THEN 'always'
    ELSE 'else'
END

-- 优化后
SELECT 'always'
```

---

## 1.11 LIKE 优化规则 (LikeOptimizationRule)

将特定模式的 LIKE 转换为更高效的字符串函数。

### 1.11.1 规则定义

```cpp
LikeOptimizationRule::LikeOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    auto func = make_uniq<FunctionExpressionMatcher>();
    func->matchers.push_back(make_uniq<ExpressionMatcher>());
    func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    func->policy = SetMatcher::Policy::ORDERED;
    // 匹配 LIKE ("~~") 和 NOT LIKE ("!~~")
    func->function = make_uniq<ManyFunctionMatcher>(
        unordered_set<string> {"!~~", "~~"});
    root = std::move(func);
}
```

### 1.11.2 模式识别

```cpp
// 检查是否为纯文本（无通配符）
static bool PatternIsConstant(const string &pattern) {
    for (idx_t i = 0; i < pattern.size(); i++) {
        if (pattern[i] == '%' || pattern[i] == '_') {
            return false;
        }
    }
    return true;
}

// 检查是否为前缀模式：'abc%'
static bool PatternIsPrefix(const string &pattern) {
    // 结尾必须是 %，其他位置不能有 % 或 _
    ...
}

// 检查是否为后缀模式：'%abc'
static bool PatternIsSuffix(const string &pattern) {
    // 开头必须是 %，其他位置不能有 % 或 _
    ...
}

// 检查是否为包含模式：'%abc%'
static bool PatternIsContains(const string &pattern) {
    // 开头和结尾都是 %，中间不能有 % 或 _
    ...
}
```

### 1.11.3 规则应用

```cpp
unique_ptr<Expression> LikeOptimizationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
    auto &constant_expr = bindings[2].get().Cast<BoundConstantExpression>();

    auto constant_value = ExpressionExecutor::EvaluateScalar(GetContext(), constant_expr);
    auto &patt_str = StringValue::Get(constant_value);
    bool is_not_like = root.function.name == "!~~";

    if (PatternIsConstant(patt_str)) {
        // 'abc' → 等值比较
        return make_uniq<BoundComparisonExpression>(
            is_not_like ? ExpressionType::COMPARE_NOTEQUAL : ExpressionType::COMPARE_EQUAL,
            std::move(root.children[0]), std::move(root.children[1]));

    } else if (PatternIsPrefix(patt_str)) {
        // 'abc%' → prefix(column, 'abc')
        return ApplyRule(root, PrefixFun::GetFunction(), patt_str, is_not_like);

    } else if (PatternIsSuffix(patt_str)) {
        // '%abc' → suffix(column, 'abc')
        return ApplyRule(root, SuffixFun::GetFunction(), patt_str, is_not_like);

    } else if (PatternIsContains(patt_str)) {
        // '%abc%' → contains(column, 'abc')
        return ApplyRule(root, GetStringContains(), patt_str, is_not_like);
    }

    return nullptr;
}
```

### 1.11.4 转换规则总结

| LIKE 模式 | 优化后 |
|-----------|--------|
| `'abc'` | `column = 'abc'` |
| `'abc%'` | `prefix(column, 'abc')` |
| `'%abc'` | `suffix(column, 'abc')` |
| `'%abc%'` | `contains(column, 'abc')` |
| `NOT LIKE 'abc%'` | `NOT prefix(column, 'abc')` |

### 1.11.5 性能提升原理

- **等值比较**：可以使用索引
- **前缀匹配**：可以使用索引范围扫描（如 ART 索引）
- **后缀/包含**：避免正则表达式引擎开销

---

## 1.12 正则表达式优化规则 (RegexOptimizationRule)

将简单正则表达式转换为 LIKE 或字符串函数。

### 1.12.1 规则定义

```cpp
RegexOptimizationRule::RegexOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    auto func = make_uniq<FunctionExpressionMatcher>();
    func->function = make_uniq<SpecificFunctionMatcher>("regexp_matches");
    func->policy = SetMatcher::Policy::SOME_ORDERED;
    func->matchers.push_back(make_uniq<ExpressionMatcher>());
    func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    root = std::move(func);
}
```

### 1.12.2 正则分析

使用 RE2 库解析正则表达式，识别可优化的模式：

```cpp
unique_ptr<Expression> RegexOptimizationRule::Apply(...) {
    duckdb_re2::RE2 pattern(patt_str, parsed_options);

    // 纯文字模式 → contains
    if (pattern.Regexp()->op() == duckdb_re2::kRegexpLiteralString ||
        pattern.Regexp()->op() == duckdb_re2::kRegexpLiteral) {
        // 'abc' → contains(column, 'abc')
        auto parameter = make_uniq<BoundConstantExpression>(Value(escaped_like_string));
        auto contains = make_uniq<BoundFunctionExpression>(..., GetStringContains(), ...);
        return contains;
    }

    // 复合模式 → 尝试转换为 LIKE
    if (pattern.Regexp()->op() == duckdb_re2::kRegexpConcat) {
        LikeString like_string = LikeMatchFromRegex(pattern);
        if (like_string.exists) {
            // 转换为 LIKE 表达式
            auto like_expression = make_uniq<BoundFunctionExpression>(
                ..., LikeFun::GetFunction(), ...);
            return like_expression;
        }
    }

    return nullptr;
}
```

### 1.12.3 支持的正则模式

| 正则模式 | RE2 操作符 | 转换结果 |
|----------|-----------|----------|
| `abc` | kRegexpLiteralString | `contains(column, 'abc')` |
| `^abc` | BeginText + Literal | `LIKE 'abc%'` |
| `abc$` | Literal + EndText | `LIKE '%abc'` |
| `^abc$` | BeginText + Literal + EndText | `LIKE 'abc'` |
| `.*abc.*` | Star + Literal + Star | `LIKE '%abc%'` |
| `.` | kRegexpAnyChar | `LIKE '_'` |

### 1.12.4 示例

```sql
-- 优化前
SELECT * FROM t WHERE regexp_matches(name, 'Smith')

-- 优化后
SELECT * FROM t WHERE contains(name, 'Smith')

-- 优化前
SELECT * FROM t WHERE regexp_matches(email, '^admin')

-- 优化后
SELECT * FROM t WHERE email LIKE 'admin%'
```

---

## 1.13 分配律规则 (DistributivityRule)

利用分配律简化 OR 表达式。

### 1.13.1 规则原理

```
(A AND X) OR (B AND X) → X AND (A OR B)
```

这种转换的好处：
- 公共因子 X 只需计算一次
- 可能产生更好的谓词下推机会

### 1.13.2 规则定义

```cpp
DistributivityRule::DistributivityRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    root = make_uniq<ExpressionMatcher>();
    root->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::CONJUNCTION_OR);
}
```

### 1.13.3 规则应用

```cpp
unique_ptr<Expression> DistributivityRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &initial_or = bindings[0].get().Cast<BoundConjunctionExpression>();

    // 1. 构建第一个分支的表达式集合
    expression_set_t candidate_set;
    AddExpressionSet(*initial_or.children[0], candidate_set);

    // 2. 与其他分支求交集，找出公共因子
    for (idx_t i = 1; i < initial_or.children.size(); i++) {
        expression_set_t next_set;
        AddExpressionSet(*initial_or.children[i], next_set);

        expression_set_t intersect_result;
        for (auto &expr : candidate_set) {
            if (next_set.find(expr) != next_set.end()) {
                intersect_result.insert(expr);
            }
        }
        candidate_set = intersect_result;
    }

    if (candidate_set.empty()) {
        return nullptr;  // 没有公共因子
    }

    // 3. 提取公共因子，构建新的 AND 表达式
    auto new_root = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND);

    for (auto &expr : candidate_set) {
        // 从每个 OR 分支中提取公共表达式
        auto result = ExtractExpression(initial_or, 0, expr.get());
        for (idx_t i = 1; i < initial_or.children.size(); i++) {
            ExtractExpression(initial_or, i, *result);
        }
        new_root->children.push_back(std::move(result));
    }

    // 4. 处理剩余表达式
    // ...（见完整实现）

    return std::move(new_root);
}
```

### 1.13.4 示例

```sql
-- 优化前
SELECT * FROM t
WHERE (a = 1 AND b = 2) OR (a = 1 AND c = 3)

-- 优化后
SELECT * FROM t
WHERE a = 1 AND (b = 2 OR c = 3)

-- 特殊情况：X OR (X AND A) = X
-- 优化前
SELECT * FROM t WHERE a = 1 OR (a = 1 AND b = 2)

-- 优化后
SELECT * FROM t WHERE a = 1
```

---

## 1.14 常量移动规则 (MoveConstantsRule)

将比较表达式中的算术运算移动到常量侧。

### 1.14.1 规则原理

```
x + 1 > 10  →  x > 9
x - 5 = 3  →  x = 8
x * 2 < 100 → x < 50
```

### 1.14.2 规则定义

```cpp
MoveConstantsRule::MoveConstantsRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    auto op = make_uniq<ComparisonExpressionMatcher>();

    // 外层常量
    op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    op->policy = SetMatcher::Policy::UNORDERED;

    // 内层算术表达式
    auto arithmetic = make_uniq<FunctionExpressionMatcher>();
    arithmetic->function = make_uniq<ManyFunctionMatcher>(
        unordered_set<string> {"+", "-", "*"});  // 注意：不包含除法
    arithmetic->type = make_uniq<IntegerTypeMatcher>();

    // 算术表达式的参数：一个常量 + 一个任意表达式
    arithmetic->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    arithmetic->matchers.push_back(make_uniq<ExpressionMatcher>());
    arithmetic->policy = SetMatcher::Policy::SOME;

    op->matchers.push_back(std::move(arithmetic));
    root = std::move(op);
}
```

### 1.14.3 规则应用（加法示例）

```cpp
if (op_type == "+") {
    // [x + 1 COMP 10] 或 [1 + x COMP 10]
    // 转换为：x COMP (10 - 1)
    if (!Hugeint::TrySubtractInPlace(outer_value, inner_value)) {
        return nullptr;  // 溢出
    }
    auto result_value = Value::HUGEINT(outer_value);
    if (!result_value.DefaultTryCastAs(constant_type)) {
        // 结果超出类型范围
        if (comparison.GetExpressionType() == ExpressionType::COMPARE_EQUAL) {
            // x + 5 = 3（x 为无符号数）→ 永远为 false
            return ExpressionRewriter::ConstantOrNull(
                std::move(arithmetic.children[arithmetic_child_index]),
                Value::BOOLEAN(false));
        }
        return nullptr;
    }
    outer_constant.value = std::move(result_value);
}
```

### 1.14.4 减法的特殊处理

减法需要考虑操作数顺序：

```cpp
if (op_type == "-") {
    if (arithmetic_child_index == 0) {
        // [x - 1 COMP 10] → [x COMP 11]
        outer_value = outer_value + inner_value;
    } else {
        // [1 - x COMP 10] → [x COMP -9]
        // 并且需要翻转比较操作符！
        outer_value = inner_value - outer_value;
        comparison.SetExpressionType(FlipComparisonExpression(comparison.GetExpressionType()));
    }
}
```

### 1.14.5 乘法的处理

```cpp
if (op_type == "*") {
    // 不能处理乘以 0
    if (inner_value == 0) {
        return nullptr;
    }

    // 必须能整除
    if (outer_value % inner_value != 0) {
        // x * 3 = 10 永远为 false（整数除法不能得到 10/3）
        if (comparison.GetExpressionType() == ExpressionType::COMPARE_EQUAL) {
            return ExpressionRewriter::ConstantOrNull(..., Value::BOOLEAN(false));
        }
        return nullptr;
    }

    // 负数乘法需要翻转比较符
    if (inner_value < 0) {
        comparison.SetExpressionType(FlipComparisonExpression(...));
    }

    outer_constant.value = outer_value / inner_value;
}
```

### 1.14.6 为什么不处理除法？

整数除法有截断问题：

```sql
x / 2 = 3  -- 不能简化为 x = 6
           -- 因为 x = 6 或 x = 7 都满足 x / 2 = 3
```

### 1.14.7 示例

```sql
-- 优化前
SELECT * FROM t WHERE x + 5 > 10

-- 优化后
SELECT * FROM t WHERE x > 5

-- 优化前
SELECT * FROM t WHERE 100 - x < 50

-- 优化后（翻转比较符）
SELECT * FROM t WHERE x > 50
```

---

## 1.15 日期函数简化规则

### 1.15.1 DatePartSimplificationRule

将 `date_part('specifier', column)` 转换为专用函数：

```cpp
unique_ptr<Expression> DatePartSimplificationRule::Apply(...) {
    auto specifier = GetDatePartSpecifier(StringValue::Get(constant));
    string new_function_name;

    switch (specifier) {
    case DatePartSpecifier::YEAR:      new_function_name = "year"; break;
    case DatePartSpecifier::MONTH:     new_function_name = "month"; break;
    case DatePartSpecifier::DAY:       new_function_name = "day"; break;
    case DatePartSpecifier::HOUR:      new_function_name = "hour"; break;
    case DatePartSpecifier::MINUTE:    new_function_name = "minute"; break;
    case DatePartSpecifier::SECOND:    new_function_name = "second"; break;
    case DatePartSpecifier::WEEK:      new_function_name = "week"; break;
    case DatePartSpecifier::DOW:       new_function_name = "dayofweek"; break;
    case DatePartSpecifier::DOY:       new_function_name = "dayofyear"; break;
    // ... 更多
    default:
        return nullptr;
    }

    // 绑定新函数
    return binder.BindScalarFunction(DEFAULT_SCHEMA, new_function_name,
                                     std::move(children), error);
}
```

**示例**：

```sql
-- 优化前
SELECT date_part('year', created_at) FROM orders

-- 优化后
SELECT year(created_at) FROM orders
```

### 1.15.2 DateTruncSimplificationRule

类似地，将 `date_trunc('specifier', column)` 转换为专用函数。

---

## 1.16 其他表达式规则

### 1.16.1 InClauseSimplificationRule

优化 IN 表达式中的类型转换：

```cpp
// 优化前：CAST(column) IN (const1, const2, ...)
// 优化后：column IN (CAST(const1), CAST(const2), ...)

// 将 CAST 从探测侧（每行都要执行）移动到常量侧（只执行一次）
```

### 1.16.2 EmptyNeedleRemovalRule

处理空字符串参数：

```cpp
// prefix(x, '') → constant_or_null(x, true)
// contains(x, '') → constant_or_null(x, true)
// suffix(x, '') → constant_or_null(x, true)
```

### 1.16.3 EnumComparisonRule

优化枚举类型比较：

```cpp
// ENUM_COLUMN = 'value' → ENUM_COLUMN = ENUM_VALUE
// 避免运行时字符串比较
```

### 1.16.4 EqualOrNullSimplification

简化 `IS NOT DISTINCT FROM` 表达式中的常量：

```cpp
// column IS NOT DISTINCT FROM NULL → column IS NULL
```

### 1.16.5 ConstantOrderNormalizationRule

规范化比较表达式的操作数顺序，使常量在右侧：

```cpp
// 5 = x → x = 5
// 便于后续规则匹配
```

---

## 1.17 规则系统的设计思想

### 1.17.1 规则排序的重要性

规则在 `Optimizer` 构造函数中的注册顺序影响效率：

1. **ConstantOrderNormalizationRule** 首先执行，确保常量在右侧
2. **ConstantFoldingRule** 紧随其后，折叠常量表达式
3. **DistributivityRule** 在算术简化之前，可能产生更多简化机会
4. **特定函数规则**（如 LIKE、Regex）在通用规则之后

### 1.17.2 不动点迭代

规则之间可能相互启用：

```
原表达式: (1 + 2) * x + 0

第1轮: ConstantFolding → 3 * x + 0
第2轮: ArithmeticSimplification → 3 * x
第3轮: 无变化，停止
```

### 1.17.3 安全性保证

规则必须保证语义等价：
- 处理 NULL 值的特殊情况
- 考虑溢出
- 检查类型转换的可逆性

### 1.17.4 扩展性

可以通过以下方式添加新规则：
1. 继承 `Rule` 基类
2. 定义 `ExpressionMatcher` 模式
3. 实现 `Apply` 方法
4. 在 `Optimizer` 构造函数中注册

---

## 1.18 本章小结

本章深入分析了 DuckDB 优化器的整体架构和表达式重写系统：

### 关键组件

| 组件 | 职责 |
|------|------|
| Optimizer | 优化器入口，协调所有优化阶段 |
| ExpressionRewriter | 表达式重写框架，应用规则到表达式 |
| Rule | 规则基类，定义匹配模式和转换逻辑 |
| ExpressionMatcher | 模式匹配器，识别可优化的表达式结构 |

### 核心规则

| 规则 | 优化类型 |
|------|----------|
| ConstantFoldingRule | 常量折叠 |
| ArithmeticSimplificationRule | 代数恒等式 |
| ComparisonSimplificationRule | 比较优化 |
| ConjunctionSimplificationRule | 逻辑简化 |
| CaseSimplificationRule | CASE 简化 |
| LikeOptimizationRule | LIKE → 字符串函数 |
| RegexOptimizationRule | 正则 → LIKE/contains |
| DistributivityRule | 分配律 |
| MoveConstantsRule | 常量移动 |

### 设计亮点

1. **规则系统**：清晰的抽象，易于扩展
2. **模式匹配**：灵活的 ExpressionMatcher 层次结构
3. **不动点迭代**：确保规则充分应用
4. **安全性**：完善的 NULL 和边界条件处理
5. **性能分析**：集成 QueryProfiler

下一章将介绍谓词优化（Filter Pushdown、Filter Pullup、Filter Combiner），这是查询优化中最重要的技术之一。

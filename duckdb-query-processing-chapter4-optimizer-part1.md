# DuckDB 查询处理深度解析（四）：优化器（上）- 表达式重写与谓词优化

## 前言

在前三章中，我们深入分析了 DuckDB 的解析器、绑定器和逻辑计划生成器。本章我们进入查询优化的核心领域——优化器（Optimizer）。

DuckDB 的优化器采用基于规则（Rule-Based）和基于代价（Cost-Based）相结合的混合优化策略。本章作为优化器上篇，重点讲解：

1. **优化器整体架构**：优化阶段顺序和控制机制
2. **表达式重写框架**：ExpressionRewriter 和 Rule 模式匹配
3. **表达式简化规则**：常量折叠、算术简化、逻辑简化等
4. **谓词优化**：谓词下推、谓词上拉、谓词合并
5. **列裁剪**：移除未使用的列
6. **公共子表达式消除**：CSE 优化

下一章将继续讲解 Join 顺序优化、基数估计和代价模型等高级主题。

---

## 1. 优化器整体架构

### 1.1 Optimizer 类设计

DuckDB 的优化器位于 `src/optimizer/` 目录，核心类是 `Optimizer`：

```cpp
// src/include/duckdb/optimizer/optimizer.hpp

class Optimizer {
public:
    Optimizer(Binder &binder, ClientContext &context);

    //! 执行优化，返回优化后的逻辑计划
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan);

    //! 获取客户端上下文
    ClientContext &GetContext();

    //! 检查特定优化器是否被禁用
    bool OptimizerDisabled(OptimizerType type);
    static bool OptimizerDisabled(ClientContext &context, OptimizerType type);

private:
    //! 运行内置优化器
    void RunBuiltInOptimizers();

    //! 运行单个优化器
    void RunOptimizer(OptimizerType type, const std::function<void()> &callback);

    //! 验证逻辑计划的正确性
    void Verify(LogicalOperator &op);

    //! 绑定标量函数的辅助方法
    unique_ptr<Expression> BindScalarFunction(const string &name,
                                               vector<unique_ptr<Expression>> children);

private:
    ClientContext &context;
    Binder &binder;
    ExpressionRewriter rewriter;
    unique_ptr<LogicalOperator> plan;
};
```

核心设计要点：

1. **Binder 引用**：优化过程中可能需要创建新的表达式，需要使用 Binder 分配新的表索引
2. **ExpressionRewriter**：内置的表达式重写器，包含所有表达式重写规则
3. **ClientContext**：用于获取配置信息和执行上下文
4. **优化器开关**：可以通过配置禁用特定的优化器

### 1.2 优化阶段顺序

`RunBuiltInOptimizers()` 方法定义了完整的优化流程：

```cpp
void Optimizer::RunBuiltInOptimizers() {
    // 跳过简单计划的优化
    switch (plan->type) {
    case LogicalOperatorType::LOGICAL_TRANSACTION:
    case LogicalOperatorType::LOGICAL_PRAGMA:
    case LogicalOperatorType::LOGICAL_SET:
        if (plan->children.empty()) {
            return;  // 无需优化
        }
        break;
    default:
        break;
    }

    // === 第一阶段：表达式简化 ===
    // 1. 表达式重写（常量折叠、算术简化等）
    RunOptimizer(OptimizerType::EXPRESSION_REWRITER, [&]() {
        rewriter.VisitOperator(*plan);
    });

    // 2. CTE 内联优化
    RunOptimizer(OptimizerType::CTE_INLINING, [&]() {
        CTEInlining cte_inlining(*this);
        plan = cte_inlining.Optimize(std::move(plan));
    });

    // 3. SUM 重写优化：SUM(x + C) -> SUM(x) + C * COUNT(x)
    RunOptimizer(OptimizerType::SUM_REWRITER, [&]() {
        SumRewriterOptimizer optimizer(*this);
        optimizer.Optimize(plan);
    });

    // === 第二阶段：谓词优化 ===
    // 4. 谓词上拉
    RunOptimizer(OptimizerType::FILTER_PULLUP, [&]() {
        FilterPullup filter_pullup;
        plan = filter_pullup.Rewrite(std::move(plan));
    });

    // 5. 谓词下推
    RunOptimizer(OptimizerType::FILTER_PUSHDOWN, [&]() {
        FilterPushdown filter_pushdown(*this);
        unordered_set<idx_t> top_bindings;
        filter_pushdown.CheckMarkToSemi(*plan, top_bindings);
        plan = filter_pushdown.Rewrite(std::move(plan));
    });

    // 6. CTE 谓词推送
    RunOptimizer(OptimizerType::CTE_FILTER_PUSHER, [&]() {
        CTEFilterPusher cte_filter_pusher(*this);
        plan = cte_filter_pusher.Optimize(std::move(plan));
    });

    // 7. 正则表达式范围优化
    RunOptimizer(OptimizerType::REGEX_RANGE, [&]() {
        RegexRangeFilter regex_opt;
        plan = regex_opt.Rewrite(std::move(plan));
    });

    // 8. IN 子句重写
    RunOptimizer(OptimizerType::IN_CLAUSE, [&]() {
        InClauseRewriter ic_rewriter(context, *this);
        plan = ic_rewriter.Rewrite(std::move(plan));
    });

    // 9. Deliminator 优化（移除冗余的 DelimGet/DelimJoin）
    RunOptimizer(OptimizerType::DELIMINATOR, [&]() {
        Deliminator deliminator;
        plan = deliminator.Optimize(std::move(plan));
    });

    // 10. 第二次 CTE 内联（在其他优化后可能有新机会）
    RunOptimizer(OptimizerType::CTE_INLINING, [&]() {
        CTEInlining cte_inlining(*this);
        plan = cte_inlining.Optimize(std::move(plan));
    });

    // 11. 空结果上拉
    RunOptimizer(OptimizerType::EMPTY_RESULT_PULLUP, [&]() {
        EmptyResultPullup empty_result_pullup;
        plan = empty_result_pullup.Optimize(std::move(plan));
    });

    // === 第三阶段：Join 优化 ===
    // 12. Join 顺序优化（包括基数估计和代价模型）
    RunOptimizer(OptimizerType::JOIN_ORDER, [&]() {
        JoinOrderOptimizer optimizer(context);
        plan = optimizer.Optimize(std::move(plan));
    });

    // 13. Join 消除（移除冗余 Join）
    RunOptimizer(OptimizerType::JOIN_ELIMINATION, [&]() {
        JoinElimination join_elimination;
        plan = join_elimination.Optimize(std::move(plan));
    });

    // 14. Unnest 重写
    RunOptimizer(OptimizerType::UNNEST_REWRITER, [&]() {
        UnnestRewriter unnest_rewriter;
        plan = unnest_rewriter.Optimize(std::move(plan));
    });

    // === 第四阶段：列和表达式优化 ===
    // 15. 移除未使用的列
    RunOptimizer(OptimizerType::UNUSED_COLUMNS, [&]() {
        RemoveUnusedColumns unused(binder, context, true);
        unused.VisitOperator(*plan);
    });

    // 16. 移除重复的 GROUP BY 列
    RunOptimizer(OptimizerType::DUPLICATE_GROUPS, [&]() {
        RemoveDuplicateGroups remove;
        remove.VisitOperator(*plan);
    });

    // 17. 公共子表达式消除
    RunOptimizer(OptimizerType::COMMON_SUBEXPRESSIONS, [&]() {
        CommonSubExpressionOptimizer cse_optimizer(binder);
        cse_optimizer.VisitOperator(*plan);
    });

    // 18. 列生命周期分析
    RunOptimizer(OptimizerType::COLUMN_LIFETIME, [&]() {
        ColumnLifetimeAnalyzer column_lifetime(*this, *plan, true);
        column_lifetime.VisitOperator(*plan);
    });

    // 19. Build/Probe 侧优化
    RunOptimizer(OptimizerType::BUILD_SIDE_PROBE_SIDE, [&]() {
        BuildProbeSideOptimizer build_probe_side_optimizer(context, *plan);
        build_probe_side_optimizer.VisitOperator(*plan);
    });

    // 20. 公共子计划优化
    RunOptimizer(OptimizerType::COMMON_SUBPLAN, [&]() {
        CommonSubplanOptimizer common_subplan_optimizer(*this);
        plan = common_subplan_optimizer.Optimize(std::move(plan));
    });

    // === 第五阶段：后期优化 ===
    // 21. LIMIT 下推
    RunOptimizer(OptimizerType::LIMIT_PUSHDOWN, [&]() {
        LimitPushdown limit_pushdown;
        plan = limit_pushdown.Optimize(std::move(plan));
    });

    // 22. Row Group 剪枝
    RunOptimizer(OptimizerType::ROW_GROUP_PRUNER, [&]() {
        RowGroupPruner row_group_pruner(context);
        plan = row_group_pruner.Optimize(std::move(plan));
    });

    // 23. 采样下推
    RunOptimizer(OptimizerType::SAMPLING_PUSHDOWN, [&]() {
        SamplingPushdown sampling_pushdown;
        plan = sampling_pushdown.Optimize(std::move(plan));
    });

    // 24. TopN 优化（ORDER BY + LIMIT -> TopN）
    RunOptimizer(OptimizerType::TOP_N, [&]() {
        TopN topn(context);
        plan = topn.Optimize(std::move(plan));
    });

    // 25. 延迟物化
    RunOptimizer(OptimizerType::LATE_MATERIALIZATION, [&]() {
        LateMaterialization late_materialization(*this);
        plan = late_materialization.Optimize(std::move(plan));
    });

    // 26. 统计信息传播
    column_binding_map_t<unique_ptr<BaseStatistics>> statistics_map;
    RunOptimizer(OptimizerType::STATISTICS_PROPAGATION, [&]() {
        StatisticsPropagator propagator(*this, *plan);
        propagator.PropagateStatistics(plan);
        statistics_map = propagator.GetStatisticsMap();
    });

    // 27. TopN 窗口消除
    RunOptimizer(OptimizerType::TOP_N_WINDOW_ELIMINATION, [&]() {
        TopNWindowElimination topn_window_elimination(context, *this, &statistics_map);
        plan = topn_window_elimination.Optimize(std::move(plan));
    });

    // 28. 公共聚合优化
    RunOptimizer(OptimizerType::COMMON_AGGREGATE, [&]() {
        CommonAggregateOptimizer common_aggregate;
        common_aggregate.VisitOperator(*plan);
    });

    // 29. 第二次列生命周期分析
    RunOptimizer(OptimizerType::COLUMN_LIFETIME, [&]() {
        ColumnLifetimeAnalyzer column_lifetime(*this, *plan, true);
        column_lifetime.VisitOperator(*plan);
    });

    // 30. 过滤器重排序
    RunOptimizer(OptimizerType::REORDER_FILTER, [&]() {
        ExpressionHeuristics expression_heuristics(*this);
        plan = expression_heuristics.Rewrite(std::move(plan));
    });

    // 31. Join 过滤器下推
    RunOptimizer(OptimizerType::JOIN_FILTER_PUSHDOWN, [&]() {
        JoinFilterPushdownOptimizer join_filter_pushdown(*this);
        join_filter_pushdown.VisitOperator(*plan);
    });
}
```

### 1.3 优化器执行流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DuckDB Optimizer Pipeline                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入: LogicalOperator Tree                                             │
│         │                                                               │
│         ▼                                                               │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    第一阶段: 表达式简化                          │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  1. ExpressionRewriter - 常量折叠、算术简化、逻辑简化          │    │
│  │  2. CTE Inlining - CTE 内联                                    │    │
│  │  3. SUM Rewriter - SUM 表达式重写                               │    │
│  └────────────────────────────────────────────────────────────────┘    │
│         │                                                               │
│         ▼                                                               │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    第二阶段: 谓词优化                            │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  4. Filter Pullup - 谓词上拉                                    │    │
│  │  5. Filter Pushdown - 谓词下推                                  │    │
│  │  6. CTE Filter Pusher - CTE 谓词推送                           │    │
│  │  7. Regex Range - 正则范围优化                                  │    │
│  │  8. IN Clause Rewriter - IN 子句重写                           │    │
│  │  9. Deliminator - 移除冗余 DelimJoin                           │    │
│  │ 10. Empty Result Pullup - 空结果优化                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│         │                                                               │
│         ▼                                                               │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    第三阶段: Join 优化                          │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │ 12. Join Order Optimizer - Join 顺序优化（代价模型）           │    │
│  │ 13. Join Elimination - 移除冗余 Join                           │    │
│  │ 14. Unnest Rewriter - Unnest 重写                              │    │
│  └────────────────────────────────────────────────────────────────┘    │
│         │                                                               │
│         ▼                                                               │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    第四阶段: 列和表达式优化                      │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │ 15. Remove Unused Columns - 移除未使用的列                     │    │
│  │ 16. Remove Duplicate Groups - 移除重复 GROUP BY                │    │
│  │ 17. CSE Optimizer - 公共子表达式消除                           │    │
│  │ 18. Column Lifetime Analyzer - 列生命周期分析                  │    │
│  │ 19. Build/Probe Side Optimizer - Build 侧选择                  │    │
│  └────────────────────────────────────────────────────────────────┘    │
│         │                                                               │
│         ▼                                                               │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    第五阶段: 后期优化                            │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │ 21. Limit Pushdown - LIMIT 下推                                │    │
│  │ 24. TopN Optimizer - ORDER BY + LIMIT 优化                     │    │
│  │ 25. Late Materialization - 延迟物化                            │    │
│  │ 26. Statistics Propagation - 统计信息传播                      │    │
│  │ 27. TopN Window Elimination - 窗口函数优化                     │    │
│  │ 30. Reorder Filter - 过滤器重排序                              │    │
│  │ 31. Join Filter Pushdown - Join 过滤器下推                     │    │
│  └────────────────────────────────────────────────────────────────┘    │
│         │                                                               │
│         ▼                                                               │
│  输出: Optimized LogicalOperator Tree                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.4 优化器开关机制

每个优化器都有对应的 `OptimizerType`，可以通过配置禁用：

```cpp
// 检查优化器是否被禁用
bool Optimizer::OptimizerDisabled(ClientContext &context_p, OptimizerType type) {
    auto &config = DBConfig::GetConfig(context_p);
    return config.options.disabled_optimizers.find(type) !=
           config.options.disabled_optimizers.end();
}

// 运行优化器时自动跳过被禁用的优化器
void Optimizer::RunOptimizer(OptimizerType type, const std::function<void()> &callback) {
    if (OptimizerDisabled(type)) {
        return;  // 优化器被禁用，跳过
    }

    // 记录性能信息
    auto &profiler = QueryProfiler::Get(context);
    profiler.StartPhase(MetricsUtils::GetOptimizerMetricByType(type));

    callback();  // 执行优化

    profiler.EndPhase();

    // 验证优化结果
    if (plan) {
        Verify(*plan);
    }
}
```

用户可以通过 SQL 命令禁用特定优化器：

```sql
-- 禁用谓词下推
SET disabled_optimizers = 'filter_pushdown';

-- 禁用多个优化器
SET disabled_optimizers = 'filter_pushdown,join_order';
```

---

## 2. 表达式重写框架

### 2.1 ExpressionRewriter 设计

`ExpressionRewriter` 是 DuckDB 表达式优化的核心框架，采用规则驱动的模式：

```cpp
// src/include/duckdb/optimizer/expression_rewriter.hpp

class ExpressionRewriter : public LogicalOperatorVisitor {
public:
    explicit ExpressionRewriter(ClientContext &context) : context(context) {}

    //! 访问算子并重写其表达式
    void VisitOperator(LogicalOperator &op) override;

    //! 访问并重写单个表达式
    void VisitExpression(unique_ptr<Expression> *expression) override;

    //! 应用规则到表达式
    static unique_ptr<Expression> ApplyRules(LogicalOperator &op,
                                              const vector<reference<Rule>> &rules,
                                              unique_ptr<Expression> expr,
                                              bool &changes_made,
                                              bool is_root = false);

    //! 创建 ConstantOrNull 表达式的辅助方法
    static unique_ptr<Expression> ConstantOrNull(unique_ptr<Expression> child, Value value);

public:
    //! 所有注册的重写规则
    vector<unique_ptr<Rule>> rules;

    //! 当前要应用的规则（可能是 rules 的子集）
    vector<reference<Rule>> to_apply_rules;

    //! 客户端上下文
    ClientContext &context;

    //! 当前正在处理的算子
    LogicalOperator *op = nullptr;
};
```

### 2.2 规则执行流程

表达式重写的核心逻辑在 `ApplyRules` 方法中：

```cpp
unique_ptr<Expression> ExpressionRewriter::ApplyRules(
    LogicalOperator &op,
    const vector<reference<Rule>> &rules,
    unique_ptr<Expression> expr,
    bool &changes_made,
    bool is_root) {

    // 遍历所有规则
    for (auto &rule : rules) {
        vector<reference<Expression>> bindings;

        // 尝试匹配规则
        if (rule.get().root->Match(*expr, bindings)) {
            // 规则匹配成功！尝试应用
            bool rule_made_change = false;
            auto alias = expr->alias;

            auto result = rule.get().Apply(op, bindings, rule_made_change, is_root);

            if (result) {
                // 规则产生了新的表达式
                changes_made = true;

                // 保留原始别名
                if (!alias.empty()) {
                    result->alias = std::move(alias);
                }

                // 递归应用规则到新表达式
                return ApplyRules(op, rules, std::move(result), changes_made);

            } else if (rule_made_change) {
                // 规则修改了子表达式但没有替换根
                changes_made = true;
                return expr;
            }
            // 规则没有做任何修改，继续下一个规则
        }
    }

    // 没有规则能够修改当前表达式
    // 递归处理子表达式
    ExpressionIterator::EnumerateChildren(*expr, [&](unique_ptr<Expression> &child) {
        child = ApplyRules(op, rules, std::move(child), changes_made);
    });

    return expr;
}
```

### 2.3 算子级别的重写

`VisitOperator` 方法在算子级别协调表达式重写：

```cpp
void ExpressionRewriter::VisitOperator(LogicalOperator &op) {
    // 先递归处理子算子
    VisitOperatorChildren(op);

    // 设置当前算子上下文
    this->op = &op;

    // 准备要应用的规则
    to_apply_rules.clear();
    for (auto &rule : rules) {
        to_apply_rules.push_back(*rule);
    }

    // 重写算子的所有表达式
    VisitOperatorExpressions(op);

    // 特殊处理：LogicalFilter 需要重新分割谓词
    if (op.type == LogicalOperatorType::LOGICAL_FILTER) {
        auto &filter = op.Cast<LogicalFilter>();
        filter.SplitPredicates();  // 分割 AND 连接的谓词
    }
}

void ExpressionRewriter::VisitExpression(unique_ptr<Expression> *expression) {
    bool changes_made;
    do {
        changes_made = false;
        *expression = ApplyRules(*op, to_apply_rules, std::move(*expression),
                                  changes_made, true);
    } while (changes_made);  // 重复直到没有更多改变
}
```

### 2.4 Rule 基类与模式匹配

每个重写规则继承自 `Rule` 基类：

```cpp
// src/include/duckdb/optimizer/rule.hpp

class Rule {
public:
    explicit Rule(ExpressionRewriter &rewriter) : rewriter(rewriter) {}
    virtual ~Rule() {}

    //! 规则的根匹配器
    unique_ptr<ExpressionMatcher> root;

    //! 应用规则
    //! @param bindings 匹配到的表达式绑定
    //! @param changes_made 输出参数，指示是否有修改
    //! @param is_root 是否在表达式树的根节点
    virtual unique_ptr<Expression> Apply(LogicalOperator &op,
                                          vector<reference<Expression>> &bindings,
                                          bool &changes_made,
                                          bool is_root) = 0;

    //! 获取客户端上下文
    ClientContext &GetContext() const;

protected:
    ExpressionRewriter &rewriter;
};
```

### 2.5 ExpressionMatcher 模式匹配

DuckDB 提供了丰富的模式匹配器来识别可优化的表达式：

```cpp
// 基础匹配器
class ExpressionMatcher {
public:
    virtual ~ExpressionMatcher() {}

    //! 尝试匹配表达式
    virtual bool Match(Expression &expr, vector<reference<Expression>> &bindings);

    //! 可选的类型匹配器
    unique_ptr<TypeMatcher> type;

    //! 可选的表达式类型过滤
    ExpressionClass expr_class = ExpressionClass::INVALID;

    //! 可选的表达式类型过滤
    ExpressionType expr_type = ExpressionType::INVALID;
};

// 常量表达式匹配器
class ConstantExpressionMatcher : public ExpressionMatcher {
public:
    ConstantExpressionMatcher() {
        expr_class = ExpressionClass::BOUND_CONSTANT;
    }
};

// 可折叠常量匹配器（匹配 IsFoldable() == true 的表达式）
class FoldableConstantMatcher : public ExpressionMatcher {
public:
    bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
        if (!expr.IsFoldable()) {
            return false;
        }
        bindings.push_back(expr);
        return true;
    }
};

// 函数表达式匹配器
class FunctionExpressionMatcher : public ExpressionMatcher {
public:
    FunctionExpressionMatcher() {
        expr_class = ExpressionClass::BOUND_FUNCTION;
    }

    //! 子表达式匹配器
    vector<unique_ptr<ExpressionMatcher>> matchers;

    //! 匹配策略
    SetMatcher::Policy policy = SetMatcher::Policy::UNORDERED;

    //! 函数名匹配器
    unique_ptr<FunctionMatcher> function;
};

// 逻辑表达式匹配器
class ConjunctionExpressionMatcher : public ExpressionMatcher {
public:
    ConjunctionExpressionMatcher() {
        expr_class = ExpressionClass::BOUND_CONJUNCTION;
    }

    vector<unique_ptr<ExpressionMatcher>> matchers;
    SetMatcher::Policy policy = SetMatcher::Policy::UNORDERED;
};
```

---

## 3. 表达式简化规则

### 3.1 优化器注册的所有规则

在 `Optimizer` 构造函数中注册了 21 个表达式重写规则：

```cpp
Optimizer::Optimizer(Binder &binder, ClientContext &context)
    : context(context), binder(binder), rewriter(context) {

    // 常量相关
    rewriter.rules.push_back(make_uniq<ConstantOrderNormalizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DistributivityRule>(rewriter));

    // 算术简化
    rewriter.rules.push_back(make_uniq<ArithmeticSimplificationRule>(rewriter));

    // 逻辑简化
    rewriter.rules.push_back(make_uniq<CaseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConjunctionSimplificationRule>(rewriter));

    // 日期时间简化
    rewriter.rules.push_back(make_uniq<DatePartSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DateTruncSimplificationRule>(rewriter));

    // 比较简化
    rewriter.rules.push_back(make_uniq<ComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InClauseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EqualOrNullSimplification>(rewriter));
    rewriter.rules.push_back(make_uniq<MoveConstantsRule>(rewriter));

    // 字符串优化
    rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));

    // 聚合优化
    rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));

    // 类型特定优化
    rewriter.rules.push_back(make_uniq<EnumComparisonRule>(rewriter));
    rewriter.rules.push_back(make_uniq<JoinDependentFilterRule>(rewriter));
    rewriter.rules.push_back(make_uniq<TimeStampComparison>(context, rewriter));
}
```

### 3.2 常量折叠（Constant Folding）

常量折叠是最基本的优化，在编译时计算常量表达式：

```cpp
// src/optimizer/rule/constant_folding.cpp

//! 自定义匹配器：匹配可折叠但非常量的表达式
class ConstantFoldingExpressionMatcher : public FoldableConstantMatcher {
public:
    bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
        // 不匹配已经是常量的表达式（无法进一步折叠）
        if (expr.GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
            return false;
        }
        return FoldableConstantMatcher::Match(expr, bindings);
    }
};

ConstantFoldingRule::ConstantFoldingRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    auto op = make_uniq<ConstantFoldingExpressionMatcher>();
    root = std::move(op);
}

unique_ptr<Expression> ConstantFoldingRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get();

    // 验证表达式确实可折叠且不是常量
    D_ASSERT(root.IsFoldable() &&
             root.GetExpressionType() != ExpressionType::VALUE_CONSTANT);

    // 使用 ExpressionExecutor 计算表达式的值
    Value result_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
        return nullptr;  // 计算失败，不做优化
    }

    // 验证结果类型一致
    D_ASSERT(result_value.type().InternalType() == root.return_type.InternalType());

    // 用常量表达式替换原表达式
    return make_uniq<BoundConstantExpression>(result_value);
}
```

**优化示例：**

```sql
-- 优化前
SELECT a + (1 + 2) FROM t

-- 优化后
SELECT a + 3 FROM t
```

### 3.3 算术简化（Arithmetic Simplification）

简化涉及特殊值（0, 1, NULL）的算术运算：

```cpp
// src/optimizer/rule/arithmetic_simplification.cpp

ArithmeticSimplificationRule::ArithmeticSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {

    // 匹配模式：算术函数(常量, 任意表达式) 或 (任意表达式, 常量)
    auto op = make_uniq<FunctionExpressionMatcher>();
    op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    op->matchers.push_back(make_uniq<ExpressionMatcher>());
    op->policy = SetMatcher::Policy::SOME;  // 任意顺序匹配

    // 只匹配加减乘除
    op->function = make_uniq<ManyFunctionMatcher>(
        unordered_set<string> {"+", "-", "*", "//"});

    // 只处理整数类型
    op->type = make_uniq<IntegerTypeMatcher>();
    op->matchers[0]->type = make_uniq<IntegerTypeMatcher>();
    op->matchers[1]->type = make_uniq<IntegerTypeMatcher>();

    root = std::move(op);
}

unique_ptr<Expression> ArithmeticSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
    auto &constant = bindings[1].get().Cast<BoundConstantExpression>();

    // 确定常量在哪一侧
    idx_t constant_child = root.children[0].get() == &constant ? 0 : 1;

    // 处理 NULL：任何涉及 NULL 的算术运算结果都是 NULL
    if (constant.value.IsNull()) {
        return make_uniq<BoundConstantExpression>(Value(root.return_type));
    }

    auto &func_name = root.function.name;

    if (func_name == "+") {
        // x + 0 = 0 + x = x
        if (constant.value == 0) {
            return std::move(root.children[1 - constant_child]);
        }
    }
    else if (func_name == "-") {
        // x - 0 = x（注意：0 - x 不能简化）
        if (constant_child == 1 && constant.value == 0) {
            return std::move(root.children[1 - constant_child]);
        }
    }
    else if (func_name == "*") {
        // x * 1 = 1 * x = x
        if (constant.value == 1) {
            return std::move(root.children[1 - constant_child]);
        }
        // x * 0 = 0 * x = 0 或 NULL（如果 x 是 NULL）
        else if (constant.value == 0) {
            return ExpressionRewriter::ConstantOrNull(
                std::move(root.children[1 - constant_child]),
                Value::Numeric(root.return_type, 0));
        }
    }
    else if (func_name == "//") {  // 整数除法
        if (constant_child == 1) {  // x // 常量
            // x // 1 = x
            if (constant.value == 1) {
                return std::move(root.children[1 - constant_child]);
            }
            // x // 0 = NULL（除以零）
            else if (constant.value == 0) {
                return make_uniq<BoundConstantExpression>(Value(root.return_type));
            }
        }
    }

    return nullptr;  // 无法简化
}
```

**优化示例：**

```sql
-- 优化前
SELECT a + 0, b * 1, c * 0, d // 1 FROM t

-- 优化后
SELECT a, b, constant_or_null(0, c), d FROM t
```

### 3.4 逻辑表达式简化（Conjunction Simplification）

简化 AND/OR 表达式中的常量条件：

```cpp
// src/optimizer/rule/conjunction_simplification.cpp

ConjunctionSimplificationRule::ConjunctionSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {

    // 匹配模式：AND/OR 表达式包含可折叠的子表达式
    auto op = make_uniq<ConjunctionExpressionMatcher>();
    op->matchers.push_back(make_uniq<FoldableConstantMatcher>());
    op->policy = SetMatcher::Policy::SOME;
    root = std::move(op);
}

unique_ptr<Expression> ConjunctionSimplificationRule::RemoveExpression(
    BoundConjunctionExpression &conj,
    const Expression &expr) {

    // 从子表达式列表中移除指定表达式
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

unique_ptr<Expression> ConjunctionSimplificationRule::Apply(
    LogicalOperator &op,
    vector<reference<Expression>> &bindings,
    bool &changes_made,
    bool is_root) {

    auto &conjunction = bindings[0].get().Cast<BoundConjunctionExpression>();
    auto &constant_expr = bindings[1].get();

    // 计算常量表达式的值
    Value constant_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), constant_expr, constant_value)) {
        return nullptr;
    }

    constant_value = constant_value.DefaultCastAs(LogicalType::BOOLEAN);

    // NULL 值不能简化
    if (constant_value.IsNull()) {
        return nullptr;
    }

    bool is_true = BooleanValue::Get(constant_value);

    if (conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
        if (!is_true) {
            // FALSE AND ... = FALSE
            return make_uniq<BoundConstantExpression>(Value::BOOLEAN(false));
        } else {
            // TRUE AND x AND y = x AND y
            return RemoveExpression(conjunction, constant_expr);
        }
    }
    else {  // OR
        D_ASSERT(conjunction.GetExpressionType() == ExpressionType::CONJUNCTION_OR);
        if (!is_true) {
            // FALSE OR x OR y = x OR y
            return RemoveExpression(conjunction, constant_expr);
        } else {
            // TRUE OR ... = TRUE
            return make_uniq<BoundConstantExpression>(Value::BOOLEAN(true));
        }
    }
}
```

**优化示例：**

```sql
-- 优化前
SELECT * FROM t WHERE TRUE AND a > 10 AND FALSE OR TRUE

-- 优化后
SELECT * FROM t WHERE FALSE    -- FALSE AND ... = FALSE
                               -- 实际上整个 WHERE 会被进一步优化
```

### 3.5 其他重要规则

#### CASE 简化

```cpp
// 简化 CASE WHEN TRUE THEN x ELSE y END -> x
// 简化 CASE WHEN FALSE THEN x ELSE y END -> y
// 简化 CASE WHEN cond THEN x ELSE x END -> x
```

#### 比较简化

```cpp
// 简化 x = x -> NOT(x IS NULL)（对于非 NULL 类型直接是 TRUE）
// 简化 x <> x -> FALSE（对于非 NULL 类型）
// 简化 x > x -> FALSE
// 简化 x >= x -> NOT(x IS NULL)
```

#### LIKE 优化

```cpp
// 简化 x LIKE 'abc' -> x = 'abc'（无通配符）
// 简化 x LIKE '%' -> x IS NOT NULL
// 简化 x LIKE 'abc%' -> x >= 'abc' AND x < 'abd'
```

#### 正则表达式优化

```cpp
// 简化 regexp_matches(x, '^abc') -> prefix(x, 'abc')
// 简化 regexp_matches(x, 'abc$') -> suffix(x, 'abc')
// 简化 regexp_matches(x, 'abc') -> contains(x, 'abc')（简单字面量）
```

### 3.6 规则执行顺序的重要性

规则的注册顺序很重要，因为优化会迭代应用直到没有更多变化：

1. **ConstantOrderNormalization**：首先规范化常量位置（如 `1 + x` -> `x + 1`）
2. **ConstantFolding**：折叠常量子表达式
3. **ArithmeticSimplification**：简化算术运算
4. **其他规则**：处理更复杂的模式

这种顺序确保了：
- 常量被规范化后，后续规则更容易匹配
- 简单优化先做，为复杂优化创造机会

---

## 4. 谓词优化

### 4.1 谓词下推（Filter Pushdown）

谓词下推是最重要的优化之一，将过滤条件尽可能推到数据源附近执行。

#### FilterPushdown 类设计

```cpp
// src/include/duckdb/optimizer/filter_pushdown.hpp

class FilterPushdown {
public:
    //! 过滤器结构
    struct Filter {
        unique_ptr<Expression> filter;
        unordered_set<idx_t> bindings;  // 涉及的表

        void ExtractBindings();  // 提取绑定信息
    };

    FilterPushdown(Optimizer &optimizer, bool convert_mark_joins = true);

    //! 重写逻辑计划，下推过滤器
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);

    //! 添加过滤器
    FilterResult AddFilter(unique_ptr<Expression> expr);

    //! 生成过滤器
    void GenerateFilters();

    //! 将过滤器推入 FilterCombiner
    FilterResult PushFilters();

private:
    //! 针对不同算子类型的下推方法
    unique_ptr<LogicalOperator> PushdownFilter(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownCrossProduct(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownProjection(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownAggregate(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownGet(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownSetOperation(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownDistinct(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownLimit(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownWindow(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownUnnest(unique_ptr<LogicalOperator> op);

    //! 针对不同 Join 类型的下推方法
    unique_ptr<LogicalOperator> PushdownInnerJoin(unique_ptr<LogicalOperator> op, ...);
    unique_ptr<LogicalOperator> PushdownLeftJoin(unique_ptr<LogicalOperator> op, ...);
    unique_ptr<LogicalOperator> PushdownMarkJoin(unique_ptr<LogicalOperator> op, ...);
    unique_ptr<LogicalOperator> PushdownSemiAntiJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownSingleJoin(unique_ptr<LogicalOperator> op, ...);
    unique_ptr<LogicalOperator> PushdownOuterJoin(unique_ptr<LogicalOperator> op, ...);

    //! 完成下推（处理无法继续下推的情况）
    unique_ptr<LogicalOperator> FinishPushdown(unique_ptr<LogicalOperator> op);

private:
    Optimizer &optimizer;
    FilterCombiner combiner;        // 谓词合并器
    vector<unique_ptr<Filter>> filters;  // 待下推的过滤器
    bool convert_mark_joins;        // 是否转换 Mark Join 为 Semi Join
};
```

#### 下推主逻辑

```cpp
unique_ptr<LogicalOperator> FilterPushdown::Rewrite(unique_ptr<LogicalOperator> op) {
    D_ASSERT(!combiner.HasFilters());

    // 根据算子类型分发到不同的下推方法
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        return PushdownAggregate(std::move(op));

    case LogicalOperatorType::LOGICAL_FILTER:
        return PushdownFilter(std::move(op));

    case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
        return PushdownCrossProduct(std::move(op));

    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
    case LogicalOperatorType::LOGICAL_ANY_JOIN:
    case LogicalOperatorType::LOGICAL_ASOF_JOIN:
    case LogicalOperatorType::LOGICAL_DELIM_JOIN:
        return PushdownJoin(std::move(op));

    case LogicalOperatorType::LOGICAL_PROJECTION:
        return PushdownProjection(std::move(op));

    case LogicalOperatorType::LOGICAL_INTERSECT:
    case LogicalOperatorType::LOGICAL_EXCEPT:
    case LogicalOperatorType::LOGICAL_UNION:
        return PushdownSetOperation(std::move(op));

    case LogicalOperatorType::LOGICAL_DISTINCT:
        return PushdownDistinct(std::move(op));

    case LogicalOperatorType::LOGICAL_ORDER_BY:
        // ORDER BY 直接透过
        op->children[0] = Rewrite(std::move(op->children[0]));
        return op;

    case LogicalOperatorType::LOGICAL_MATERIALIZED_CTE:
        // CTE 左侧不能下推，右侧可以
        {
            FilterPushdown pushdown(optimizer, convert_mark_joins);
            op->children[0] = pushdown.Rewrite(std::move(op->children[0]));
            op->children[1] = Rewrite(std::move(op->children[1]));
            return op;
        }

    case LogicalOperatorType::LOGICAL_GET:
        return PushdownGet(std::move(op));

    case LogicalOperatorType::LOGICAL_LIMIT:
        return PushdownLimit(std::move(op));

    case LogicalOperatorType::LOGICAL_WINDOW:
        return PushdownWindow(std::move(op));

    case LogicalOperatorType::LOGICAL_UNNEST:
        return PushdownUnnest(std::move(op));

    default:
        return FinishPushdown(std::move(op));
    }
}
```

#### Join 下推策略

不同类型的 Join 有不同的下推规则：

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownJoin(unique_ptr<LogicalOperator> op) {
    auto &join = op->Cast<LogicalJoin>();

    // 获取左右两侧涉及的表
    unordered_set<idx_t> left_bindings, right_bindings;
    LogicalJoin::GetTableReferences(*op->children[0], left_bindings);
    LogicalJoin::GetTableReferences(*op->children[1], right_bindings);

    unique_ptr<LogicalOperator> result;
    switch (join.join_type) {
    case JoinType::OUTER:
        // 外连接：需要特殊处理
        result = PushdownOuterJoin(std::move(op), left_bindings, right_bindings);
        break;

    case JoinType::INNER:
        // 内连接：可以自由下推到两侧
        if (op->type == LogicalOperatorType::LOGICAL_ASOF_JOIN) {
            // AsOf Join 特殊处理，右侧不能下推
            result = PushdownLeftJoin(std::move(op), left_bindings, right_bindings);
        } else {
            result = PushdownInnerJoin(std::move(op), left_bindings, right_bindings);
        }
        break;

    case JoinType::LEFT:
        // 左连接：只能下推到左侧
        result = PushdownLeftJoin(std::move(op), left_bindings, right_bindings);
        break;

    case JoinType::MARK:
        // Mark Join
        result = PushdownMarkJoin(std::move(op), left_bindings, right_bindings);
        break;

    case JoinType::SINGLE:
        // Single Join
        result = PushdownSingleJoin(std::move(op), left_bindings, right_bindings);
        break;

    case JoinType::SEMI:
    case JoinType::ANTI:
        // Semi/Anti Join
        result = PushdownSemiAntiJoin(std::move(op));
        break;

    default:
        // 不支持的 Join 类型，停止下推
        return FinishPushdown(std::move(op));
    }

    return result;
}
```

#### 下推到 TableScan

谓词最终可以下推到表扫描：

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownGet(unique_ptr<LogicalOperator> op) {
    auto &get = op->Cast<LogicalGet>();

    // 将过滤器推入 FilterCombiner
    if (PushFilters() == FilterResult::UNSATISFIABLE) {
        // 过滤器不可满足，返回空结果
        return make_uniq<LogicalEmptyResult>(std::move(op));
    }

    // 生成表过滤器
    auto &column_ids = get.GetColumnIds();
    vector<FilterPushdownResult> pushdown_results;
    auto table_filters = combiner.GenerateTableScanFilters(column_ids, pushdown_results);

    // 合并到 LogicalGet 的 table_filters
    for (auto &entry : table_filters.filters) {
        get.table_filters.PushFilter(entry.first, std::move(entry.second));
    }

    // 生成剩余过滤器
    GenerateFilters();

    // 将未能下推的过滤器保留为 LogicalFilter
    return PushFinalFilters(std::move(op));
}
```

**谓词下推示例：**

```sql
-- 原始查询
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'China' AND o.amount > 1000

-- 下推后的逻辑计划
LogicalJoin(INNER, o.customer_id = c.id)
├── LogicalFilter(o.amount > 1000)
│   └── LogicalGet(orders)
└── LogicalFilter(c.country = 'China')
    └── LogicalGet(customers)
```

### 4.2 谓词合并（Filter Combiner）

`FilterCombiner` 负责分析和合并谓词，支持等价类推导和范围推导。

#### FilterCombiner 核心设计

```cpp
// src/include/duckdb/optimizer/filter_combiner.hpp

class FilterCombiner {
public:
    //! 存储表达式与常量的比较信息
    struct ExpressionValueInformation {
        ExpressionType comparison_type;  // 比较类型
        Value constant;                   // 常量值
    };

    FilterCombiner(ClientContext &context);

    //! 添加过滤器
    FilterResult AddFilter(unique_ptr<Expression> expr);
    FilterResult AddFilter(Expression &expr);

    //! 生成优化后的过滤器
    void GenerateFilters(const std::function<void(unique_ptr<Expression>)> &callback);

    //! 生成可下推到 TableScan 的过滤器
    TableFilterSet GenerateTableScanFilters(const vector<ColumnIndex> &column_ids,
                                            vector<FilterPushdownResult> &pushdown_results);

    //! 检查是否有过滤器
    bool HasFilters();

private:
    //! 获取或创建表达式节点
    Expression &GetNode(Expression &expr);

    //! 获取表达式的等价类集合
    idx_t GetEquivalenceSet(Expression &expr);

    //! 添加常量比较
    FilterResult AddConstantComparison(vector<ExpressionValueInformation> &info_list,
                                       ExpressionValueInformation info);

    //! 添加比较过滤器
    FilterResult AddBoundComparisonFilter(Expression &expr);

    //! 添加传递性过滤器
    FilterResult AddTransitiveFilters(BoundComparisonExpression &comparison, bool is_root = true);

    //! 查找传递性过滤器
    unique_ptr<Expression> FindTransitiveFilter(Expression &expr);

private:
    ClientContext &context;

    //! 存储的表达式（保证唯一性）
    expression_map_t<unique_ptr<Expression>> stored_expressions;

    //! 表达式到等价类的映射
    expression_map_t<idx_t> equivalence_set_map;

    //! 等价类到表达式列表的映射
    unordered_map<idx_t, vector<reference<Expression>>> equivalence_map;

    //! 等价类的常量约束
    unordered_map<idx_t, vector<ExpressionValueInformation>> constant_values;

    //! 不能进一步优化的过滤器
    vector<unique_ptr<Expression>> remaining_filters;

    //! 等价类索引计数器
    idx_t set_index = 0;
};
```

#### 等价类推导

FilterCombiner 通过等价类来推导新的过滤条件：

```cpp
FilterResult FilterCombiner::AddBoundComparisonFilter(Expression &expr) {
    auto &comparison = expr.Cast<BoundComparisonExpression>();

    // 检查一侧是否是标量
    bool left_is_scalar = comparison.left->IsFoldable();
    bool right_is_scalar = comparison.right->IsFoldable();

    if (left_is_scalar || right_is_scalar) {
        // 表达式与常量比较
        auto &node = GetNode(left_is_scalar ? *comparison.right : *comparison.left);
        idx_t equivalence_set = GetEquivalenceSet(node);

        auto &scalar = left_is_scalar ? comparison.left : comparison.right;
        Value constant_value;
        if (!ExpressionExecutor::TryEvaluateScalar(context, *scalar, constant_value)) {
            return FilterResult::UNSUPPORTED;
        }

        // NULL 比较总是不可满足
        if (constant_value.IsNull()) {
            return FilterResult::UNSATISFIABLE;
        }

        // 创建约束信息
        ExpressionValueInformation info;
        info.comparison_type = left_is_scalar ?
            FlipComparisonExpression(comparison.GetExpressionType()) :
            comparison.GetExpressionType();
        info.constant = constant_value;

        // 添加到等价类的常量约束
        auto &info_list = constant_values.find(equivalence_set)->second;
        auto ret = AddConstantComparison(info_list, info);

        // 尝试添加传递性过滤器
        auto &non_scalar = left_is_scalar ? *comparison.right : *comparison.left;
        auto transitive_filter = FindTransitiveFilter(non_scalar);
        if (transitive_filter != nullptr) {
            auto transitive_result = AddTransitiveFilters(
                transitive_filter->Cast<BoundComparisonExpression>());
            // ...
        }

        return ret;
    }
    else {
        // 两个非标量表达式的比较（如 a = b）
        if (expr.GetExpressionType() != ExpressionType::COMPARE_EQUAL) {
            return FilterResult::UNSUPPORTED;
        }

        auto &left_node = GetNode(*comparison.left);
        auto &right_node = GetNode(*comparison.right);

        if (left_node.Equals(right_node)) {
            return FilterResult::UNSUPPORTED;
        }

        // 合并两个等价类
        auto left_equivalence_set = GetEquivalenceSet(left_node);
        auto right_equivalence_set = GetEquivalenceSet(right_node);

        if (left_equivalence_set == right_equivalence_set) {
            // 已经在同一等价类，冗余过滤器
            return FilterResult::SUCCESS;
        }

        // 合并等价类
        auto &left_bucket = equivalence_map.find(left_equivalence_set)->second;
        auto &right_bucket = equivalence_map.find(right_equivalence_set)->second;

        for (auto &right_expr : right_bucket) {
            equivalence_set_map[right_expr] = left_equivalence_set;
            left_bucket.push_back(right_expr);
        }

        // 合并常量约束
        auto &left_constant_bucket = constant_values.find(left_equivalence_set)->second;
        auto &right_constant_bucket = constant_values.find(right_equivalence_set)->second;

        for (auto &right_constant : right_constant_bucket) {
            if (AddConstantComparison(left_constant_bucket, right_constant) ==
                FilterResult::UNSATISFIABLE) {
                return FilterResult::UNSATISFIABLE;
            }
        }
    }

    return FilterResult::SUCCESS;
}
```

#### 范围推导与矛盾检测

```cpp
// 比较两个值约束，检测矛盾或冗余
ValueComparisonResult CompareValueInformation(
    ExpressionValueInformation &left,
    ExpressionValueInformation &right) {

    if (left.comparison_type == ExpressionType::COMPARE_EQUAL) {
        // 左边是等值约束
        bool prune_right_side = false;

        switch (right.comparison_type) {
        case ExpressionType::COMPARE_LESSTHAN:
            prune_right_side = left.constant < right.constant;
            break;
        case ExpressionType::COMPARE_LESSTHANOREQUALTO:
            prune_right_side = left.constant <= right.constant;
            break;
        case ExpressionType::COMPARE_GREATERTHAN:
            prune_right_side = left.constant > right.constant;
            break;
        case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
            prune_right_side = left.constant >= right.constant;
            break;
        case ExpressionType::COMPARE_NOTEQUAL:
            prune_right_side = left.constant != right.constant;
            break;
        case ExpressionType::COMPARE_EQUAL:
            prune_right_side = left.constant == right.constant;
            break;
        }

        if (prune_right_side) {
            return ValueComparisonResult::PRUNE_RIGHT;  // 可以移除右边约束
        } else {
            return ValueComparisonResult::UNSATISFIABLE_CONDITION;  // 矛盾！
        }
    }
    // ... 更多比较逻辑

    // 检测范围矛盾：x < 5 AND x > 10
    else if (IsLessThan(left.comparison_type) && IsGreaterThan(right.comparison_type)) {
        if (left.constant >= right.constant) {
            return ValueComparisonResult::PRUNE_NOTHING;
        } else {
            return ValueComparisonResult::UNSATISFIABLE_CONDITION;  // 矛盾！
        }
    }

    // ... 更多情况
}
```

**等价类推导示例：**

```sql
-- 原始查询
SELECT * FROM t WHERE a = b AND b = c AND a = 10

-- FilterCombiner 推导：
-- 等价类: {a, b, c}
-- 常量约束: a = 10
-- 推导出: b = 10, c = 10

-- 优化后
SELECT * FROM t WHERE a = 10 AND b = 10 AND c = 10
```

**矛盾检测示例：**

```sql
-- 原始查询
SELECT * FROM t WHERE a = 10 AND a = 20

-- FilterCombiner 检测到矛盾：
-- a = 10 与 a = 20 不可能同时满足

-- 优化后：返回 LogicalEmptyResult
```

### 4.3 谓词上拉（Filter Pullup）

谓词上拉与下推相反，将分散的过滤条件合并到一起：

```cpp
// src/include/duckdb/optimizer/filter_pullup.hpp

class FilterPullup {
public:
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);

private:
    unique_ptr<LogicalOperator> PullupFilter(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupFromLeft(unique_ptr<LogicalOperator> op);

    //! 合并过滤器
    void MergeFilters(LogicalFilter &filter, vector<unique_ptr<Expression>> &new_filters);

    vector<unique_ptr<Expression>> filters_expr_pullup;
};
```

上拉通常在下推之前执行，确保所有过滤器都被收集到一起，然后统一下推。

---

## 5. 列裁剪（Remove Unused Columns）

列裁剪优化移除查询中未使用的列，减少数据扫描和传输。

### 5.1 RemoveUnusedColumns 设计

```cpp
// src/include/duckdb/optimizer/remove_unused_columns.hpp

class RemoveUnusedColumns : public BaseColumnPruner {
public:
    RemoveUnusedColumns(Binder &binder, ClientContext &context,
                        bool is_root = false);

    //! 访问算子
    void VisitOperator(LogicalOperator &op) override;

protected:
    //! 清除未使用的表达式
    template <class T>
    void ClearUnusedExpressions(vector<T> &list, idx_t table_idx, bool replace = true);

    //! 移除 LogicalGet 中未使用的列
    void RemoveColumnsFromLogicalGet(LogicalGet &get);

private:
    Binder &binder;
    ClientContext &context;
    bool everything_referenced;  // 是否所有列都被引用
};

class BaseColumnPruner : public LogicalOperatorVisitor {
protected:
    //! 列引用记录
    struct ReferencedColumn {
        vector<reference<BoundColumnRefExpression>> bindings;
        vector<ColumnIndex> child_columns;  // 用于结构体字段裁剪
    };

    //! 列绑定到引用记录的映射
    column_binding_map_t<ReferencedColumn> column_references;

    //! 添加列绑定
    void AddBinding(BoundColumnRefExpression &col);
    void AddBinding(BoundColumnRefExpression &col, ColumnIndex child_column);

    //! 替换列绑定
    void ReplaceBinding(ColumnBinding current_binding, ColumnBinding new_binding);

    //! 处理结构体字段提取
    bool HandleStructExtract(Expression &expr);

    //! 要传递的子列信息
    vector<ColumnIndex> deliver_child;
};
```

### 5.2 列裁剪逻辑

```cpp
void RemoveUnusedColumns::VisitOperator(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
        auto &aggr = op.Cast<LogicalAggregate>();

        // 多个 grouping set 需要保留所有列
        bool new_root = (aggr.grouping_sets.size() > 1);

        if (!everything_referenced && !new_root) {
            // 移除未使用的聚合表达式
            ClearUnusedExpressions(aggr.expressions, aggr.aggregate_index);

            // 如果所有表达式都被移除，添加 COUNT(*)
            if (aggr.expressions.empty() && aggr.groups.empty()) {
                auto count_star_fun = CountStarFun::GetFunction();
                FunctionBinder function_binder(context);
                aggr.expressions.push_back(
                    function_binder.BindAggregateFunction(count_star_fun, {}, nullptr,
                                                         AggregateType::NON_DISTINCT));
            }
        }

        // 递归处理子节点
        RemoveUnusedColumns remove(binder, context, new_root);
        remove.VisitOperatorExpressions(op);
        remove.VisitOperator(*op.children[0]);
        return;
    }

    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
        if (everything_referenced) {
            break;
        }

        auto &comp_join = op.Cast<LogicalComparisonJoin>();
        if (comp_join.join_type != JoinType::INNER) {
            break;
        }

        // 对于内连接的等值条件，可以用左侧列替换右侧列
        for (auto &cond : comp_join.conditions) {
            if (cond.comparison != ExpressionType::COMPARE_EQUAL) {
                continue;
            }
            // ... 替换逻辑
        }
        break;
    }

    case LogicalOperatorType::LOGICAL_PROJECTION: {
        if (!everything_referenced) {
            auto &proj = op.Cast<LogicalProjection>();

            // 移除未使用的投影表达式
            ClearUnusedExpressions(proj.expressions, proj.table_index);

            // 至少保留一个表达式
            if (proj.expressions.empty()) {
                proj.expressions.push_back(
                    make_uniq<BoundConstantExpression>(Value::INTEGER(42)));
            }
        }

        // 递归处理
        RemoveUnusedColumns remove(binder, context);
        remove.VisitOperatorExpressions(op);
        remove.VisitOperator(*op.children[0]);
        return;
    }

    case LogicalOperatorType::LOGICAL_GET: {
        LogicalOperatorVisitor::VisitOperatorExpressions(op);
        auto &get = op.Cast<LogicalGet>();
        RemoveColumnsFromLogicalGet(get);
        return;
    }

    case LogicalOperatorType::LOGICAL_DISTINCT: {
        auto &distinct = op.Cast<LogicalDistinct>();
        if (distinct.distinct_type == DistinctType::DISTINCT_ON) {
            break;  // DISTINCT ON 需要保留列
        }
        // 普通 DISTINCT 需要所有投影列
        everything_referenced = true;
        break;
    }

    // ... 其他算子类型
    }

    LogicalOperatorVisitor::VisitOperatorExpressions(op);
    LogicalOperatorVisitor::VisitOperatorChildren(op);
}
```

### 5.3 从 TableScan 移除未使用的列

```cpp
void RemoveUnusedColumns::RemoveColumnsFromLogicalGet(LogicalGet &get) {
    if (everything_referenced) {
        return;
    }

    if (!get.function.projection_pushdown) {
        return;  // 数据源不支持投影下推
    }

    auto final_column_ids = get.GetColumnIds();

    // 创建列选择向量
    vector<idx_t> proj_sel;
    for (idx_t col_idx = 0; col_idx < final_column_ids.size(); col_idx++) {
        proj_sel.push_back(col_idx);
    }
    auto col_sel = proj_sel;

    // 移除未使用的列
    ClearUnusedExpressions(proj_sel, get.table_index, false);

    // 处理过滤器引用的列
    vector<unique_ptr<Expression>> filter_expressions;
    for (auto &filter : get.table_filters.filters) {
        // ... 确保过滤器引用的列不被移除
    }

    // 再次清理，包括仅用于过滤的列
    ClearUnusedExpressions(col_sel, get.table_index);

    // 设置新的列 ID
    vector<ColumnIndex> column_ids;
    for (idx_t idx = 0; idx < col_sel.size(); idx++) {
        // ... 构建新的列索引
    }

    if (column_ids.empty()) {
        // 没有列被使用（如 EXISTS 查询），只扫描行 ID
        column_ids.emplace_back(get.GetAnyColumn());
    }

    get.SetColumnIds(std::move(column_ids));

    // 设置投影映射
    if (get.function.filter_prune) {
        get.projection_ids.clear();
        // ... 设置投影 ID
    }
}
```

**列裁剪示例：**

```sql
-- 原始查询（表有 10 列）
SELECT name, age FROM users WHERE country = 'China'

-- 列裁剪后只扫描 3 列
LogicalProjection([name, age])
└── LogicalFilter(country = 'China')
    └── LogicalGet(users, columns=[name, age, country])
```

---

## 6. 公共子表达式消除（CSE）

### 6.1 CSE 优化器设计

```cpp
// src/include/duckdb/optimizer/cse_optimizer.hpp

class CommonSubExpressionOptimizer : public LogicalOperatorVisitor {
public:
    explicit CommonSubExpressionOptimizer(Binder &binder) : binder(binder) {}

    void VisitOperator(LogicalOperator &op) override;

private:
    //! 提取公共子表达式
    void ExtractCommonSubExpresions(LogicalOperator &op);

    //! 统计表达式出现次数
    void CountExpressions(Expression &expr, CSEReplacementState &state);

    //! 执行 CSE 替换
    void PerformCSEReplacement(unique_ptr<Expression> &expr_ptr, CSEReplacementState &state);

    Binder &binder;
};

//! CSE 状态
struct CSEReplacementState {
    //! 新投影的表索引
    idx_t projection_index;

    //! 表达式到出现次数的映射
    expression_map_t<CSENode> expression_count;

    //! 列绑定到投影索引的映射
    column_binding_map_t<idx_t> column_map;

    //! 结果投影的表达式
    vector<unique_ptr<Expression>> expressions;

    //! 缓存的表达式（保持有效引用）
    vector<unique_ptr<Expression>> cached_expressions;

    //! 短路表达式追踪
    bool short_circuited = false;
};
```

### 6.2 CSE 执行流程

```cpp
void CommonSubExpressionOptimizer::VisitOperator(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_PROJECTION:
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        // 只在投影和聚合中提取 CSE
        ExtractCommonSubExpresions(op);
        break;
    default:
        break;
    }

    LogicalOperatorVisitor::VisitOperator(op);
}

void CommonSubExpressionOptimizer::ExtractCommonSubExpresions(LogicalOperator &op) {
    D_ASSERT(op.children.size() == 1);

    // 第一遍：统计每个表达式的出现次数
    CSEReplacementState state;
    LogicalOperatorVisitor::EnumerateExpressions(op, [&](unique_ptr<Expression> *child) {
        CountExpressions(**child, state);
    });

    // 检查是否有表达式出现多次
    bool perform_replacement = false;
    for (auto &expr : state.expression_count) {
        if (expr.second.count > 1) {
            perform_replacement = true;
            break;
        }
    }

    if (!perform_replacement) {
        return;  // 没有 CSE
    }

    // 第二遍：执行替换
    state.projection_index = binder.GenerateTableIndex();
    LogicalOperatorVisitor::EnumerateExpressions(op, [&](unique_ptr<Expression> *child) {
        PerformCSEReplacement(*child, state);
    });

    D_ASSERT(state.expressions.size() > 0);

    // 创建新的投影节点
    auto projection = make_uniq<LogicalProjection>(state.projection_index,
                                                    std::move(state.expressions));
    if (op.children[0]->has_estimated_cardinality) {
        projection->SetEstimatedCardinality(op.children[0]->estimated_cardinality);
    }

    projection->children.push_back(std::move(op.children[0]));
    op.children[0] = std::move(projection);
}
```

### 6.3 表达式计数（处理短路表达式）

```cpp
void CommonSubExpressionOptimizer::CountExpressions(Expression &expr, CSEReplacementState &state) {
    // 跳过简单表达式
    switch (expr.GetExpressionClass()) {
    case ExpressionClass::BOUND_COLUMN_REF:
    case ExpressionClass::BOUND_CONSTANT:
    case ExpressionClass::BOUND_PARAMETER:
        return;
    default:
        break;
    }

    // 聚合表达式不能移动到投影
    // 非 volatile 表达式可以考虑 CSE
    if (expr.GetExpressionClass() != ExpressionClass::BOUND_AGGREGATE && !expr.IsVolatile()) {
        auto node = state.expression_count.find(expr);
        if (node == state.expression_count.end()) {
            // 首次遇到此表达式
            if (!state.short_circuited) {
                // 只有在非短路上下文中才计数
                state.expression_count[expr] = CSENode();
            }
        } else {
            // 表达式出现多次
            node->second.count++;
        }
    }

    // 处理短路表达式（AND、OR、CASE）
    switch (expr.GetExpressionClass()) {
    case ExpressionClass::BOUND_CONJUNCTION:
    case ExpressionClass::BOUND_CASE: {
        const auto save_short_circuit = state.short_circuited;

        ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) {
            CountExpressions(child, state);
            state.short_circuited = true;  // 后续参数是短路的
        });

        state.short_circuited = save_short_circuit;
        break;
    }
    default:
        // 递归计数子表达式
        ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) {
            CountExpressions(child, state);
        });
        break;
    }
}
```

### 6.4 执行 CSE 替换

```cpp
void CommonSubExpressionOptimizer::PerformCSEReplacement(
    unique_ptr<Expression> &expr_ptr,
    CSEReplacementState &state) {

    Expression &expr = *expr_ptr;

    // 处理列引用
    if (expr.GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF) {
        auto &bound_column_ref = expr.Cast<BoundColumnRefExpression>();

        auto column_entry = state.column_map.find(bound_column_ref.binding);
        if (column_entry == state.column_map.end()) {
            // 首次遇到：添加到投影
            idx_t new_column_index = state.expressions.size();
            state.column_map[bound_column_ref.binding] = new_column_index;

            state.expressions.push_back(make_uniq<BoundColumnRefExpression>(
                bound_column_ref.GetAlias(),
                bound_column_ref.return_type,
                bound_column_ref.binding));

            bound_column_ref.binding = ColumnBinding(state.projection_index, new_column_index);
        } else {
            // 已存在：更新绑定
            bound_column_ref.binding = ColumnBinding(state.projection_index, column_entry->second);
        }
        return;
    }

    // 检查是否是 CSE 候选
    if (state.expression_count.find(expr) != state.expression_count.end()) {
        auto &node = state.expression_count[expr];

        if (node.count > 1) {
            // 表达式出现多次！移动到投影
            auto alias = expr.GetAlias();
            auto type = expr.return_type;

            if (!node.column_index.IsValid()) {
                // 首次：添加到投影
                node.column_index = state.expressions.size();
                state.expressions.push_back(std::move(expr_ptr));
            } else {
                // 已添加：缓存原表达式
                state.cached_expressions.push_back(std::move(expr_ptr));
            }

            // 用列引用替换
            expr_ptr = make_uniq<BoundColumnRefExpression>(
                alias, type,
                ColumnBinding(state.projection_index, node.column_index.GetIndex()));
            return;
        }
    }

    // 表达式只出现一次，递归处理子表达式
    ExpressionIterator::EnumerateChildren(expr, [&](unique_ptr<Expression> &child) {
        PerformCSEReplacement(child, state);
    });
}
```

**CSE 优化示例：**

```sql
-- 原始查询
SELECT (a + b) * 2, (a + b) * 3, (a + b) + 1 FROM t

-- CSE 优化后
-- 新增投影: cse_0 = a + b
LogicalProjection([cse_0 * 2, cse_0 * 3, cse_0 + 1])
└── LogicalProjection([a + b AS cse_0])
    └── LogicalGet(t)
```

---

## 7. 优化示例：完整流程

让我们通过一个完整的例子来展示优化过程：

```sql
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
  AND o.date >= '2024-01-01'
  AND 1 = 1
GROUP BY c.name
HAVING SUM(o.amount) > 1000
ORDER BY SUM(o.amount) DESC
LIMIT 10;
```

### 7.1 初始逻辑计划

```
LogicalLimit(10)
└── LogicalOrder(SUM(o.amount) DESC)
    └── LogicalFilter(SUM(o.amount) > 1000)  -- HAVING
        └── LogicalAggregate(GROUP BY c.name, SUM(o.amount))
            └── LogicalFilter(c.country = 'China' AND o.date >= '2024-01-01' AND 1 = 1)
                └── LogicalJoin(INNER, c.id = o.customer_id)
                    ├── LogicalGet(customers)
                    └── LogicalGet(orders)
```

### 7.2 表达式重写阶段

**常量折叠：**
- `1 = 1` → `TRUE`

**逻辑简化：**
- `... AND TRUE` → `...`

**结果：**
```
LogicalLimit(10)
└── LogicalOrder(SUM(o.amount) DESC)
    └── LogicalFilter(SUM(o.amount) > 1000)
        └── LogicalAggregate(GROUP BY c.name, SUM(o.amount))
            └── LogicalFilter(c.country = 'China' AND o.date >= '2024-01-01')
                └── LogicalJoin(INNER, c.id = o.customer_id)
                    ├── LogicalGet(customers)
                    └── LogicalGet(orders)
```

### 7.3 谓词下推阶段

**Filter Pullup：** 收集所有过滤条件

**Filter Pushdown：**
- `c.country = 'China'` 下推到 customers 表扫描
- `o.date >= '2024-01-01'` 下推到 orders 表扫描

**结果：**
```
LogicalLimit(10)
└── LogicalOrder(SUM(o.amount) DESC)
    └── LogicalFilter(SUM(o.amount) > 1000)
        └── LogicalAggregate(GROUP BY c.name, SUM(o.amount))
            └── LogicalJoin(INNER, c.id = o.customer_id)
                ├── LogicalGet(customers, filter=[country = 'China'])
                └── LogicalGet(orders, filter=[date >= '2024-01-01'])
```

### 7.4 列裁剪阶段

**分析列使用：**
- customers 表：需要 `id`（JOIN）、`name`（SELECT/GROUP BY）、`country`（过滤）
- orders 表：需要 `customer_id`（JOIN）、`amount`（聚合）、`date`（过滤）

**移除未使用的列：**
```
LogicalLimit(10)
└── LogicalOrder(SUM(o.amount) DESC)
    └── LogicalFilter(SUM(o.amount) > 1000)
        └── LogicalAggregate(GROUP BY c.name, SUM(o.amount))
            └── LogicalJoin(INNER, c.id = o.customer_id)
                ├── LogicalGet(customers, columns=[id, name], filter=[country = 'China'])
                └── LogicalGet(orders, columns=[customer_id, amount], filter=[date >= '2024-01-01'])
```

### 7.5 后期优化

**TopN 优化：** ORDER BY + LIMIT → TopN
```
LogicalTopN(10, SUM(o.amount) DESC)
└── LogicalFilter(SUM(o.amount) > 1000)
    └── LogicalAggregate(GROUP BY c.name, SUM(o.amount))
        └── LogicalJoin(INNER, c.id = o.customer_id)
            ├── LogicalGet(customers, columns=[id, name], filter=[country = 'China'])
            └── LogicalGet(orders, columns=[customer_id, amount], filter=[date >= '2024-01-01'])
```

---

## 8. 总结

本章详细分析了 DuckDB 优化器的表达式重写和谓词优化机制：

### 8.1 核心优化技术

| 优化技术 | 作用 | 实现类 |
|---------|------|--------|
| 表达式重写 | 简化表达式，消除冗余计算 | `ExpressionRewriter` |
| 常量折叠 | 编译时计算常量表达式 | `ConstantFoldingRule` |
| 算术简化 | 简化 +0, *1, *0 等 | `ArithmeticSimplificationRule` |
| 逻辑简化 | 简化 AND TRUE, OR FALSE 等 | `ConjunctionSimplificationRule` |
| 谓词下推 | 将过滤条件推到数据源 | `FilterPushdown` |
| 谓词合并 | 等价类推导，范围推导 | `FilterCombiner` |
| 列裁剪 | 移除未使用的列 | `RemoveUnusedColumns` |
| CSE 消除 | 提取公共子表达式 | `CommonSubExpressionOptimizer` |

### 8.2 优化顺序的设计原则

1. **表达式简化优先**：为后续优化创造机会
2. **谓词优化居中**：收集、合并、下推过滤条件
3. **结构优化在后**：Join 顺序、列裁剪需要完整信息
4. **后期优化收尾**：TopN、统计传播、过滤器重排序

### 8.3 下一章预告

下一章将继续讲解优化器的高级主题：
- Join 顺序优化器（JoinOrderOptimizer）
- 基数估计（CardinalityEstimator）
- 代价模型（Cost Model）
- 统计信息传播（StatisticsPropagator）
- 其他高级优化技术

---

## 附录：相关源文件索引

| 组件 | 主要文件 |
|------|----------|
| Optimizer | `src/optimizer/optimizer.cpp` |
| ExpressionRewriter | `src/optimizer/expression_rewriter.cpp` |
| Rule 基类 | `src/include/duckdb/optimizer/rule.hpp` |
| 常量折叠 | `src/optimizer/rule/constant_folding.cpp` |
| 算术简化 | `src/optimizer/rule/arithmetic_simplification.cpp` |
| 逻辑简化 | `src/optimizer/rule/conjunction_simplification.cpp` |
| Filter Pushdown | `src/optimizer/filter_pushdown.cpp` |
| Filter Combiner | `src/optimizer/filter_combiner.cpp` |
| 列裁剪 | `src/optimizer/remove_unused_columns.cpp` |
| CSE 优化 | `src/optimizer/cse_optimizer.cpp` |

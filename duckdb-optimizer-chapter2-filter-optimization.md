# DuckDB 查询优化器深度解析 - 第二章：谓词优化

## 2.1 谓词优化概述

谓词优化是查询优化中最重要的技术之一，通过将过滤条件尽可能靠近数据源来减少处理的数据量。DuckDB 的谓词优化系统由三个核心组件组成：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    谓词优化组件架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FilterPullup                              │   │
│  │    谓词上拉：从子树中提取过滤器                              │   │
│  │    时机：在 FilterPushdown 之前执行                         │   │
│  └───────────────────────┬─────────────────────────────────────┘   │
│                          │                                         │
│                          ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FilterPushdown                            │   │
│  │    谓词下推：将过滤器推向数据源                              │   │
│  │    核心逻辑：根据算子类型决定下推策略                        │   │
│  └───────────────────────┬─────────────────────────────────────┘   │
│                          │                                         │
│                          ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FilterCombiner                            │   │
│  │    过滤器合并：等价类推导、范围优化、矛盾检测                │   │
│  │    用于 FilterPushdown 内部的智能过滤器管理                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 源码位置

| 组件 | 文件路径 |
|------|----------|
| FilterPushdown | `src/optimizer/filter_pushdown.cpp` |
| 各算子下推实现 | `src/optimizer/pushdown/*.cpp` |
| FilterPullup | `src/optimizer/filter_pullup.cpp` |
| 各算子上拉实现 | `src/optimizer/pullup/*.cpp` |
| FilterCombiner | `src/optimizer/filter_combiner.cpp` |

---

## 2.2 FilterPushdown 架构

`FilterPushdown` 是谓词下推的核心类，负责将过滤条件尽可能推向数据源。

### 2.2.1 类结构

```cpp
class FilterPushdown {
public:
    explicit FilterPushdown(Optimizer &optimizer, bool convert_mark_joins = true);

    // 主入口：执行谓词下推
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);

    // 预处理：检查 Mark Join 是否可以转换为 Semi Join
    void CheckMarkToSemi(LogicalOperator &op, unordered_set<idx_t> &table_bindings);

    // 过滤器结构
    struct Filter {
        unordered_set<idx_t> bindings;  // 过滤器引用的表索引
        unique_ptr<Expression> filter;   // 过滤表达式

        void ExtractBindings();  // 从表达式中提取表绑定
    };

private:
    Optimizer &optimizer;
    FilterCombiner combiner;           // 过滤器合并器
    bool convert_mark_joins;           // 是否转换 Mark Join
    vector<unique_ptr<Filter>> filters; // 待下推的过滤器列表

    // 各算子的下推方法
    unique_ptr<LogicalOperator> PushdownAggregate(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownFilter(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownProjection(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownGet(unique_ptr<LogicalOperator> op);
    // ... 更多下推方法

    // 辅助方法
    FilterResult AddFilter(unique_ptr<Expression> expr);
    void GenerateFilters();
    unique_ptr<LogicalOperator> FinishPushdown(unique_ptr<LogicalOperator> op);
};
```

### 2.2.2 主入口 Rewrite

`Rewrite` 方法根据算子类型分派到不同的处理函数：

```cpp
unique_ptr<LogicalOperator> FilterPushdown::Rewrite(unique_ptr<LogicalOperator> op) {
    D_ASSERT(!combiner.HasFilters());

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

    case LogicalOperatorType::LOGICAL_GET:
        return PushdownGet(std::move(op));

    case LogicalOperatorType::LOGICAL_ORDER_BY:
        // 直接穿透 ORDER BY
        op->children[0] = Rewrite(std::move(op->children[0]));
        return op;

    case LogicalOperatorType::LOGICAL_MATERIALIZED_CTE: {
        // CTE 左侧（物化部分）需要独立下推
        FilterPushdown pushdown(optimizer, convert_mark_joins);
        op->children[0] = pushdown.Rewrite(std::move(op->children[0]));
        // 右侧（使用部分）继续当前下推
        op->children[1] = Rewrite(std::move(op->children[1]));
        return op;
    }

    default:
        return FinishPushdown(std::move(op));
    }
}
```

### 2.2.3 Filter 结构与绑定提取

每个待下推的过滤器包含表达式和它引用的表索引：

```cpp
struct Filter {
    unordered_set<idx_t> bindings;  // 引用的表索引集合
    unique_ptr<Expression> filter;   // 过滤表达式

    void ExtractBindings() {
        bindings.clear();
        // 从表达式中提取所有列引用的表索引
        LogicalJoin::GetExpressionBindings(*filter, bindings);
    }
};
```

表绑定用于判断过滤器可以下推到哪一侧：

```cpp
// 示例：判断过滤器属于 Join 的哪一侧
JoinSide side = JoinSide::GetJoinSide(filter->bindings, left_bindings, right_bindings);

// 可能的结果：
// JoinSide::LEFT   - 只引用左表
// JoinSide::RIGHT  - 只引用右表
// JoinSide::BOTH   - 引用两侧（如 Join 条件）
// JoinSide::NONE   - 不引用任何表（常量表达式）
```

---

## 2.3 Join 下推策略

Join 是最复杂的下推场景，不同类型的 Join 有不同的下推规则。

### 2.3.1 PushdownJoin 分派

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownJoin(unique_ptr<LogicalOperator> op) {
    auto &join = op->Cast<LogicalJoin>();

    // 获取左右两侧的表绑定
    unordered_set<idx_t> left_bindings, right_bindings;
    LogicalJoin::GetTableReferences(*op->children[0], left_bindings);
    LogicalJoin::GetTableReferences(*op->children[1], right_bindings);

    // 根据 Join 类型选择下推策略
    switch (join.join_type) {
    case JoinType::INNER:
        return PushdownInnerJoin(std::move(op), left_bindings, right_bindings);

    case JoinType::LEFT:
        return PushdownLeftJoin(std::move(op), left_bindings, right_bindings);

    case JoinType::OUTER:
        return PushdownOuterJoin(std::move(op), left_bindings, right_bindings);

    case JoinType::MARK:
        return PushdownMarkJoin(std::move(op), left_bindings, right_bindings);

    case JoinType::SEMI:
    case JoinType::ANTI:
        return PushdownSemiAntiJoin(std::move(op));

    default:
        return FinishPushdown(std::move(op));
    }
}
```

### 2.3.2 Inner Join 下推

Inner Join 是最灵活的，过滤器可以自由下推到任意一侧：

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownInnerJoin(
    unique_ptr<LogicalOperator> op,
    unordered_set<idx_t> &left_bindings,
    unordered_set<idx_t> &right_bindings) {

    auto &join = op->Cast<LogicalJoin>();

    // 1. 将 Join 条件转换为过滤器
    if (op->type == LogicalOperatorType::LOGICAL_ANY_JOIN) {
        auto &any_join = join.Cast<LogicalAnyJoin>();
        if (AddFilter(std::move(any_join.condition)) == FilterResult::UNSATISFIABLE) {
            // 条件永假，返回空结果
            return make_uniq<LogicalEmptyResult>(std::move(op));
        }
    } else {
        // Comparison Join
        auto &comp_join = join.Cast<LogicalComparisonJoin>();
        for (auto &cond : comp_join.conditions) {
            auto condition = JoinCondition::CreateExpression(std::move(cond));
            if (AddFilter(std::move(condition)) == FilterResult::UNSATISFIABLE) {
                return make_uniq<LogicalEmptyResult>(std::move(op));
            }
        }
    }

    // 2. 生成过滤器（从 FilterCombiner 中提取）
    GenerateFilters();

    // 3. 将 Inner Join 转换为 Cross Product
    auto cross_product = make_uniq<LogicalCrossProduct>(
        std::move(op->children[0]),
        std::move(op->children[1]));

    // 4. 继续下推 Cross Product
    return PushdownCrossProduct(std::move(cross_product));
}
```

**关键思想**：Inner Join 被转换为 Cross Product + Filter，然后 Filter 可以被分别下推到两侧。

```
优化前:
    InnerJoin (a.id = b.id AND b.x > 5)
       /        \
   Scan(a)    Scan(b)

优化后:
    CrossProduct
       /        \
   Scan(a)    Filter(b.x > 5)
                  |
               Scan(b)
   + Filter(a.id = b.id) 留在上层或转为 Join
```

### 2.3.3 Left Join 下推

Left Join 的下推规则更加复杂：

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownLeftJoin(
    unique_ptr<LogicalOperator> op,
    unordered_set<idx_t> &left_bindings,
    unordered_set<idx_t> &right_bindings) {

    FilterPushdown left_pushdown(optimizer, convert_mark_joins);
    FilterPushdown right_pushdown(optimizer, convert_mark_joins);

    // 创建 FilterCombiner 用于传递性推导
    FilterCombiner filter_combiner(optimizer);
    if (isComparison) {
        // 添加 Join 条件到 combiner
        for (auto &cond : comparison_join.conditions) {
            filter_combiner.AddFilter(make_uniq<BoundComparisonExpression>(...));
        }
    }

    vector<unique_ptr<Filter>> remaining_filters;

    for (idx_t i = 0; i < filters.size(); i++) {
        auto side = JoinSide::GetJoinSide(filters[i]->bindings, left_bindings, right_bindings);

        if (side == JoinSide::LEFT) {
            // 只引用左表：可以安全下推到左侧
            if (isComparison) {
                // 添加到 combiner 以推导传递性过滤
                filter_combiner.AddFilter(filters[i]->filter->Copy());
            }
            left_pushdown.filters.push_back(std::move(filters[i]));
        } else {
            // 引用右表或两侧：检查是否可以转换为 Inner Join
            if (FilterRemovesNull(optimizer.context, optimizer.rewriter,
                                  filters[i]->filter.get(), right_bindings)) {
                // 过滤器会排除 NULL 值，可以转换为 Inner Join！
                join.join_type = JoinType::INNER;
                // 恢复所有过滤器，重新处理
                for (auto &f : left_pushdown.filters) {
                    filters.push_back(std::move(f));
                }
                return PushdownInnerJoin(std::move(op), left_bindings, right_bindings);
            }
            remaining_filters.push_back(std::move(filters[i]));
        }
    }

    // 从 FilterCombiner 中推导右侧过滤器
    // 例如：Join 条件是 a.id = b.id，左侧有 a.id = 5
    // 则可以推导出 b.id = 5 推到右侧
    filter_combiner.GenerateFilters([&](unique_ptr<Expression> filter) {
        if (JoinSide::GetJoinSide(*filter, left_bindings, right_bindings) == JoinSide::RIGHT) {
            right_pushdown.AddFilter(std::move(filter));
        }
    });

    // 递归下推到子节点
    op->children[0] = left_pushdown.Rewrite(std::move(op->children[0]));
    op->children[1] = right_pushdown.Rewrite(std::move(op->children[1]));

    // 处理剩余过滤器
    return PushFinalFilters(std::move(op));
}
```

### 2.3.4 Left Join → Inner Join 转换

关键函数 `FilterRemovesNull` 检查过滤器是否会排除 NULL 值：

```cpp
static bool FilterRemovesNull(ClientContext &context, ExpressionRewriter &rewriter,
                              Expression *expr, unordered_set<idx_t> &right_bindings) {
    // 1. 复制表达式
    auto copy = expr->Copy();

    // 2. 将右表的列引用替换为 NULL
    copy = ReplaceColRefWithNull(std::move(copy), right_bindings);

    // 3. 使用表达式重写器简化
    auto filter = make_uniq<LogicalFilter>();
    filter->expressions.push_back(std::move(copy));
    rewriter.VisitOperator(*filter);

    // 4. 检查结果是否为 FALSE 或 NULL
    for (auto &expr : filter->expressions) {
        if (!expr->IsFoldable()) {
            return false;
        }
        auto val = ExpressionExecutor::EvaluateScalar(context, *expr)
                       .DefaultCastAs(LogicalType::BOOLEAN);
        if (val.IsNull() || !BooleanValue::Get(val)) {
            // 当右表为 NULL 时，过滤器结果为 FALSE/NULL
            // 意味着 Left Join 产生的 NULL 行会被过滤掉
            // 因此可以安全转换为 Inner Join
            return true;
        }
    }
    return false;
}
```

**示例**：

```sql
-- 原始查询
SELECT * FROM a LEFT JOIN b ON a.id = b.id WHERE b.x > 5

-- 分析：
-- 当 b 的行不匹配时，b.x 为 NULL
-- b.x > 5 对于 NULL 结果为 FALSE/NULL
-- 所以 Left Join 产生的 NULL 行会被过滤掉

-- 优化后（转换为 Inner Join）
SELECT * FROM a INNER JOIN b ON a.id = b.id WHERE b.x > 5
```

### 2.3.5 下推规则总结

| Join 类型 | 左侧下推 | 右侧下推 | 说明 |
|-----------|----------|----------|------|
| Inner | 完全支持 | 完全支持 | 可以转换为 Cross Product + Filter |
| Left | 完全支持 | 仅支持从 Join 条件推导 | 可能转换为 Inner |
| Right | 仅支持从 Join 条件推导 | 完全支持 | 类似 Left |
| Full Outer | 不支持 | 不支持 | 必须保留 NULL |
| Semi/Anti | 完全支持 | 独立下推 | 右侧不影响输出 |
| Mark | 取决于使用方式 | 独立下推 | 可能转换为 Semi |

---

## 2.4 下推到扫描 (PushdownGet)

谓词下推的最终目标是将过滤器推到表扫描层。

### 2.4.1 实现概述

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownGet(unique_ptr<LogicalOperator> op) {
    auto &get = op->Cast<LogicalGet>();

    // 1. 检查表函数是否支持过滤器下推
    if (get.function.pushdown_complex_filter || get.function.filter_pushdown) {
        // 处理参数化查询
        for (auto &filter : filters) {
            if (filter->filter->HasParameter()) {
                BoundParameterExpression::InvalidateRecursive(*filter->filter);
            }
        }
    }

    // 2. 尝试复杂过滤器下推（如 Parquet 的谓词下推）
    if (get.function.pushdown_complex_filter) {
        vector<unique_ptr<Expression>> expressions;
        for (auto &filter : filters) {
            expressions.push_back(std::move(filter->filter));
        }
        filters.clear();

        // 调用表函数的自定义下推逻辑
        get.function.pushdown_complex_filter(optimizer.context, get,
                                             get.bind_data.get(), expressions);

        // 重新生成未下推的过滤器
        for (auto &expr : expressions) {
            auto f = make_uniq<Filter>();
            f->filter = std::move(expr);
            f->ExtractBindings();
            filters.push_back(std::move(f));
        }
    }

    // 3. 生成 TableFilterSet
    if (get.function.filter_pushdown) {
        if (PushFilters() == FilterResult::UNSATISFIABLE) {
            return make_uniq<LogicalEmptyResult>(std::move(op));
        }

        vector<FilterPushdownResult> pushdown_results;
        get.table_filters = combiner.GenerateTableScanFilters(
            get.GetColumnIds(), pushdown_results);

        GenerateFilters();

        // 4. 尝试下推通用表达式
        for (idx_t i = 0; i < filters.size(); ++i) {
            auto pushdown_result = pushdown_results[i];
            if (pushdown_result != FilterPushdownResult::NO_PUSHDOWN) {
                continue;  // 已下推
            }
            auto &expr = *filters[i]->filter;
            if (expr.IsVolatile() || expr.CanThrow()) {
                continue;  // 不能下推 volatile 或可能抛异常的表达式
            }
            pushdown_result = combiner.TryPushdownGenericExpression(get, expr);
            if (pushdown_result == FilterPushdownResult::PUSHED_DOWN_FULLY) {
                filters.erase_at(i);
                pushdown_results.erase_at(i);
                i--;
            }
        }
    }

    return FinishPushdown(std::move(op));
}
```

### 2.4.2 TableFilterSet 生成

`FilterCombiner::GenerateTableScanFilters` 生成可以在扫描时执行的过滤器：

```cpp
TableFilterSet FilterCombiner::GenerateTableScanFilters(
    const vector<ColumnIndex> &column_ids,
    vector<FilterPushdownResult> &pushdown_results) {

    TableFilterSet table_filters;

    // 1. 处理常量比较过滤器
    for (auto &constant_value : constant_values) {
        auto expr_id = constant_value.first;
        auto &const_list = constant_value.second;
        TryPushdownConstantFilter(table_filters, column_ids, expr_id, const_list);
    }

    // 2. 处理 LIKE、IN、OR 等特殊过滤器
    for (idx_t i = 0; i < remaining_filters.size(); i++) {
        auto &remaining_filter = remaining_filters[i];
        auto pushdown_result = TryPushdownExpression(
            table_filters, column_ids, *remaining_filter);

        if (pushdown_result == FilterPushdownResult::PUSHED_DOWN_FULLY) {
            remaining_filters.erase_at(i--);
        } else {
            pushdown_results.push_back(pushdown_result);
        }
    }

    return table_filters;
}
```

### 2.4.3 支持的 TableFilter 类型

| Filter 类型 | 说明 | 示例 |
|-------------|------|------|
| ConstantFilter | 常量比较 | `x > 5`, `x = 'abc'` |
| IsNullFilter | NULL 检查 | `x IS NULL` |
| IsNotNullFilter | 非 NULL 检查 | `x IS NOT NULL` |
| InFilter | IN 列表 | `x IN (1, 2, 3)` |
| ConjunctionAndFilter | AND 组合 | `x > 1 AND x < 10` |
| ConjunctionOrFilter | OR 组合 | `x = 1 OR x = 2` |
| OptionalFilter | 可选过滤（用于 Zonemap） | 非密集 IN 列表 |
| StructFilter | 嵌套结构字段 | `struct.field > 5` |
| ExpressionFilter | 通用表达式 | 复杂表达式 |

### 2.4.4 特殊过滤器下推

**LIKE 过滤器**：

```cpp
FilterPushdownResult FilterCombiner::TryPushdownLikeFilter(
    TableFilterSet &table_filters,
    const vector<ColumnIndex> &column_ids,
    Expression &expr) {

    auto &func = expr.Cast<BoundFunctionExpression>();
    if (func.function.name != "~~") {
        return FilterPushdownResult::NO_PUSHDOWN;
    }

    auto &like_string = StringValue::Get(constant_value_expr.value);

    // 提取前缀
    string prefix;
    bool equality = true;
    for (char const &c : like_string) {
        if (c == '%' || c == '_') {
            equality = false;
            break;
        }
        prefix += c;
    }

    if (equality) {
        // 'abc' → 等值过滤
        auto equal_filter = make_uniq<ConstantFilter>(
            ExpressionType::COMPARE_EQUAL, Value(prefix));
        table_filters.PushFilter(column_index, std::move(equal_filter));
        return FilterPushdownResult::PUSHED_DOWN_FULLY;
    }

    // 'abc%' → 范围过滤 (>= 'abc' AND < 'abd')
    auto lower_bound = make_uniq<ConstantFilter>(
        ExpressionType::COMPARE_GREATERTHANOREQUALTO, Value(prefix));
    prefix[prefix.size() - 1]++;
    auto upper_bound = make_uniq<ConstantFilter>(
        ExpressionType::COMPARE_LESSTHAN, Value(prefix));

    table_filters.PushFilter(column_index, std::move(lower_bound));
    table_filters.PushFilter(column_index, std::move(upper_bound));
    return FilterPushdownResult::PUSHED_DOWN_PARTIALLY;  // 仍需执行 LIKE
}
```

**IN 过滤器**：

```cpp
FilterPushdownResult FilterCombiner::TryPushdownInFilter(
    TableFilterSet &table_filters,
    const vector<ColumnIndex> &column_ids,
    Expression &expr) {

    // 收集 IN 列表的值
    vector<Value> in_list;
    for (idx_t i = 1; i < func.children.size(); i++) {
        auto &const_value_expr = func.children[i]->Cast<BoundConstantExpression>();
        in_list.push_back(const_value_expr.value);
    }

    // 检查是否为密集范围（如 1, 2, 3, 4, 5）
    if (type.IsIntegral() && IsDenseRange(in_list)) {
        // 转换为范围过滤：x >= 1 AND x <= 5
        auto lower_bound = make_uniq<ConstantFilter>(
            ExpressionType::COMPARE_GREATERTHANOREQUALTO, std::move(in_list.front()));
        auto upper_bound = make_uniq<ConstantFilter>(
            ExpressionType::COMPARE_LESSTHANOREQUALTO, std::move(in_list.back()));
        table_filters.PushFilter(column_index, std::move(lower_bound));
        table_filters.PushFilter(column_index, std::move(upper_bound));
        return FilterPushdownResult::PUSHED_DOWN_FULLY;
    }

    // 非密集范围：使用 OptionalFilter 进行 Zonemap 检查
    auto optional_filter = make_uniq<OptionalFilter>();
    auto in_filter = make_uniq<InFilter>(std::move(in_list));
    optional_filter->child_filter = std::move(in_filter);
    table_filters.PushFilter(column_index, std::move(optional_filter));
    return FilterPushdownResult::PUSHED_DOWN_PARTIALLY;
}
```

---

## 2.5 FilterPullup 架构

`FilterPullup` 在 `FilterPushdown` 之前执行，负责将嵌套在计划树中的过滤器提取出来。

### 2.5.1 为什么需要 FilterPullup？

```sql
-- 示例查询
SELECT * FROM (
    SELECT * FROM a WHERE a.x > 5
) sub
JOIN b ON sub.id = b.id
WHERE sub.y < 10
```

在这个查询中：
1. 内层有 `a.x > 5` 过滤器
2. 外层有 `sub.y < 10` 过滤器
3. 还有 Join 条件 `sub.id = b.id`

`FilterPullup` 将所有过滤器提取到统一位置，然后 `FilterPushdown` 可以进行全局优化。

### 2.5.2 类结构

```cpp
class FilterPullup {
public:
    explicit FilterPullup(bool pullup = false, bool add_column = false)
        : can_pullup(pullup), can_add_column(add_column) {}

    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);

private:
    vector<unique_ptr<Expression>> filters_expr_pullup;  // 收集的过滤器
    bool can_pullup = false;      // 是否允许上拉（遇到分叉时设置）
    bool can_add_column = false;  // 是否可以添加列（用于集合操作）

    unique_ptr<LogicalOperator> PullupFilter(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupProjection(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupInnerJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupFromLeft(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupBothSide(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PullupSetOperation(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> FinishPullup(unique_ptr<LogicalOperator> op);
};
```

### 2.5.3 主入口

```cpp
unique_ptr<LogicalOperator> FilterPullup::Rewrite(unique_ptr<LogicalOperator> op) {
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_FILTER:
        return PullupFilter(std::move(op));

    case LogicalOperatorType::LOGICAL_PROJECTION:
        return PullupProjection(std::move(op));

    case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
        return PullupCrossProduct(std::move(op));

    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
    case LogicalOperatorType::LOGICAL_ANY_JOIN:
    case LogicalOperatorType::LOGICAL_DELIM_JOIN:
    case LogicalOperatorType::LOGICAL_ASOF_JOIN:
        return PullupJoin(std::move(op));

    case LogicalOperatorType::LOGICAL_INTERSECT:
    case LogicalOperatorType::LOGICAL_EXCEPT:
        return PullupSetOperation(std::move(op));

    case LogicalOperatorType::LOGICAL_DISTINCT:
        return PullupDistinct(std::move(op));

    case LogicalOperatorType::LOGICAL_ORDER_BY:
        // 直接穿透
        op->children[0] = Rewrite(std::move(op->children[0]));
        return op;

    default:
        return FinishPullup(std::move(op));
    }
}
```

### 2.5.4 Inner Join 上拉

```cpp
unique_ptr<LogicalOperator> FilterPullup::PullupInnerJoin(unique_ptr<LogicalOperator> op) {
    // 1. 从两侧上拉过滤器
    op = PullupBothSide(std::move(op));

    // 2. 收集已上拉的过滤器
    vector<unique_ptr<Expression>> expressions;
    if (op->type == LogicalOperatorType::LOGICAL_FILTER) {
        expressions = std::move(op->expressions);
        op = std::move(op->children[0]);
    }

    // 3. 提取 Join 条件
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
        auto &comp_join = op->Cast<LogicalComparisonJoin>();
        for (auto &cond : comp_join.conditions) {
            expressions.push_back(make_uniq<BoundComparisonExpression>(
                cond.comparison, std::move(cond.left), std::move(cond.right)));
        }
        break;
    }
    case LogicalOperatorType::LOGICAL_ANY_JOIN: {
        auto &any_join = op->Cast<LogicalAnyJoin>();
        expressions.push_back(std::move(any_join.condition));
        break;
    }
    }

    // 4. 转换为 Cross Product
    op = make_uniq<LogicalCrossProduct>(
        std::move(op->children[0]),
        std::move(op->children[1]));

    // 5. 如果可以上拉，收集过滤器；否则生成 Filter 节点
    if (can_pullup) {
        for (auto &expr : expressions) {
            filters_expr_pullup.push_back(std::move(expr));
        }
    } else {
        op = GeneratePullupFilter(std::move(op), expressions);
    }

    return op;
}
```

### 2.5.5 上拉规则总结

| 算子类型 | 上拉行为 |
|----------|----------|
| Filter | 提取过滤器表达式 |
| Inner Join | 提取 Join 条件，转换为 Cross Product |
| Left Join | 只从左侧上拉 |
| Cross Product | 从两侧上拉 |
| Projection | 穿透上拉，更新列引用 |
| Distinct | 穿透上拉 |
| Set Operations | 特殊处理，需要考虑列映射 |

---

## 2.6 FilterCombiner 详解

`FilterCombiner` 是谓词优化的智能核心，负责：
1. 等价类管理
2. 传递性过滤器推导
3. 范围合并
4. 矛盾检测

### 2.6.1 类结构

```cpp
class FilterCombiner {
public:
    struct ExpressionValueInformation {
        Value constant;
        ExpressionType comparison_type;
    };

    FilterResult AddFilter(unique_ptr<Expression> expr);

    void GenerateFilters(const std::function<void(unique_ptr<Expression> filter)> &callback);

    bool HasFilters();

    TableFilterSet GenerateTableScanFilters(
        const vector<ColumnIndex> &column_ids,
        vector<FilterPushdownResult> &pushdown_results);

private:
    // 存储的表达式（去重）
    expression_map_t<unique_ptr<Expression>> stored_expressions;

    // 表达式到等价类的映射
    expression_map_t<idx_t> equivalence_set_map;

    // 等价类中的常量值
    map<idx_t, vector<ExpressionValueInformation>> constant_values;

    // 等价类中的表达式
    map<idx_t, vector<reference<Expression>>> equivalence_map;

    // 未能处理的过滤器
    vector<unique_ptr<Expression>> remaining_filters;

    idx_t set_index = 0;
};
```

### 2.6.2 等价类管理

等价类是 FilterCombiner 的核心概念：

```cpp
// 获取或创建表达式的等价类
idx_t FilterCombiner::GetEquivalenceSet(Expression &expr) {
    auto entry = equivalence_set_map.find(expr);
    if (entry == equivalence_set_map.end()) {
        // 创建新的等价类
        idx_t index = set_index++;
        equivalence_set_map[expr] = index;
        equivalence_map[index].push_back(expr);
        constant_values.insert(make_pair(index, vector<ExpressionValueInformation>()));
        return index;
    } else {
        return entry->second;
    }
}
```

当遇到 `a = b` 这样的等式时，合并等价类：

```cpp
// 处理两个非标量表达式的比较 (a = b)
if (expr.GetExpressionType() == ExpressionType::COMPARE_EQUAL) {
    auto &left_node = GetNode(*comparison.left);
    auto &right_node = GetNode(*comparison.right);

    auto left_equivalence_set = GetEquivalenceSet(left_node);
    auto right_equivalence_set = GetEquivalenceSet(right_node);

    if (left_equivalence_set == right_equivalence_set) {
        // 已经在同一等价类中
        return FilterResult::SUCCESS;
    }

    // 合并等价类：将右侧的所有表达式和常量移到左侧
    auto &left_bucket = equivalence_map.find(left_equivalence_set)->second;
    auto &right_bucket = equivalence_map.find(right_equivalence_set)->second;

    for (auto &right_expr : right_bucket) {
        equivalence_set_map[right_expr] = left_equivalence_set;
        left_bucket.push_back(right_expr);
    }

    // 合并常量值
    auto &left_constants = constant_values.find(left_equivalence_set)->second;
    auto &right_constants = constant_values.find(right_equivalence_set)->second;

    for (auto &right_constant : right_constants) {
        if (AddConstantComparison(left_constants, right_constant) == FilterResult::UNSATISFIABLE) {
            return FilterResult::UNSATISFIABLE;
        }
    }
}
```

### 2.6.3 范围合并与剪枝

```cpp
FilterResult FilterCombiner::AddConstantComparison(
    vector<ExpressionValueInformation> &info_list,
    ExpressionValueInformation info) {

    if (info.constant.IsNull()) {
        // 与 NULL 比较永远为假
        return FilterResult::UNSATISFIABLE;
    }

    for (idx_t i = 0; i < info_list.size(); i++) {
        auto comparison = CompareValueInformation(info_list[i], info);

        switch (comparison) {
        case ValueComparisonResult::PRUNE_LEFT:
            // 新条件更严格，移除旧条件
            // 例如：已有 x > 5，新条件 x > 7，移除 x > 5
            info_list.erase_at(i);
            i--;
            break;

        case ValueComparisonResult::PRUNE_RIGHT:
            // 旧条件更严格，忽略新条件
            // 例如：已有 x > 7，新条件 x > 5，忽略 x > 5
            return FilterResult::SUCCESS;

        case ValueComparisonResult::UNSATISFIABLE_CONDITION:
            // 条件矛盾
            // 例如：已有 x = 5，新条件 x > 6，矛盾
            return FilterResult::UNSATISFIABLE;

        default:
            // 不能剪枝，继续检查
            break;
        }
    }

    // 添加新条件
    info_list.push_back(info);
    return FilterResult::SUCCESS;
}
```

### 2.6.4 条件比较逻辑

```cpp
ValueComparisonResult CompareValueInformation(
    ExpressionValueInformation &left,
    ExpressionValueInformation &right) {

    // 等值比较优先级最高
    if (left.comparison_type == ExpressionType::COMPARE_EQUAL) {
        // x = 5 可以剪枝其他条件
        switch (right.comparison_type) {
        case ExpressionType::COMPARE_LESSTHAN:
            // x = 5 AND x < 10 → 保留 x = 5
            return (left.constant < right.constant)
                ? ValueComparisonResult::PRUNE_RIGHT
                : ValueComparisonResult::UNSATISFIABLE_CONDITION;
        case ExpressionType::COMPARE_GREATERTHAN:
            // x = 5 AND x > 3 → 保留 x = 5
            return (left.constant > right.constant)
                ? ValueComparisonResult::PRUNE_RIGHT
                : ValueComparisonResult::UNSATISFIABLE_CONDITION;
        case ExpressionType::COMPARE_EQUAL:
            // x = 5 AND x = 5 → 保留一个
            // x = 5 AND x = 6 → 矛盾
            return (left.constant == right.constant)
                ? ValueComparisonResult::PRUNE_RIGHT
                : ValueComparisonResult::UNSATISFIABLE_CONDITION;
        // ...
        }
    }

    // 同方向范围比较
    if (IsGreaterThan(left.comparison_type) && IsGreaterThan(right.comparison_type)) {
        // 例如：x > 5 和 x > 7
        if (left.constant > right.constant) {
            // x > 7 更严格，剪枝 x > 5
            return ValueComparisonResult::PRUNE_RIGHT;
        } else if (left.constant < right.constant) {
            // x > 5 被 x > 7 覆盖
            return ValueComparisonResult::PRUNE_LEFT;
        } else {
            // 相等时，> 比 >= 更严格
            if (left.comparison_type == ExpressionType::COMPARE_GREATERTHANOREQUALTO) {
                return ValueComparisonResult::PRUNE_LEFT;
            } else {
                return ValueComparisonResult::PRUNE_RIGHT;
            }
        }
    }

    // 相反方向范围比较
    if (IsLessThan(left.comparison_type) && IsGreaterThan(right.comparison_type)) {
        // 例如：x < 5 和 x > 7 → 矛盾
        // 例如：x < 10 和 x > 5 → 可以共存
        if (left.constant >= right.constant) {
            return ValueComparisonResult::PRUNE_NOTHING;
        } else {
            return ValueComparisonResult::UNSATISFIABLE_CONDITION;
        }
    }

    // ... 更多情况
}
```

### 2.6.5 传递性过滤器推导

```cpp
FilterResult FilterCombiner::AddTransitiveFilters(
    BoundComparisonExpression &comparison, bool is_root) {

    // 只处理 >, >=, <, <=
    if (!IsGreaterThan(comparison.GetExpressionType()) &&
        !IsLessThan(comparison.GetExpressionType())) {
        return FilterResult::UNSUPPORTED;
    }

    auto &left_node = GetNode(*comparison.left);
    auto &right_node = GetNode(*comparison.right);

    auto left_equivalence_set = GetEquivalenceSet(left_node);
    auto right_equivalence_set = GetEquivalenceSet(right_node);

    auto &left_constants = constant_values.find(left_equivalence_set)->second;
    auto &right_constants = constant_values.find(right_equivalence_set)->second;

    // 遍历右侧的常量条件
    for (const auto &right_constant : right_constants) {
        ExpressionValueInformation info;
        info.constant = right_constant.constant;

        // 根据比较类型和已有条件推导新条件
        if (right_constant.comparison_type == ExpressionType::COMPARE_EQUAL) {
            // 例如：j >= i 且 i = 10 → j >= 10
            info.comparison_type = comparison.GetExpressionType();
        } else if (comparison.GetExpressionType() == ExpressionType::COMPARE_GREATERTHANOREQUALTO &&
                   IsGreaterThan(right_constant.comparison_type)) {
            // 例如：j >= i 且 i > 10 → j > 10
            info.comparison_type = right_constant.comparison_type;
        }
        // ... 更多情况

        // 添加到左侧的常量列表
        if (AddConstantComparison(left_constants, info) == FilterResult::UNSATISFIABLE) {
            return FilterResult::UNSATISFIABLE;
        }
    }

    return FilterResult::SUCCESS;
}
```

**示例**：

```sql
-- 原始条件
WHERE a.id = b.id AND a.id = 100

-- 经过 FilterCombiner 处理后
-- 等价类: {a.id, b.id}
-- 常量条件: {= 100}

-- 生成的过滤器：
-- a.id = 100
-- b.id = 100  (传递推导)
```

### 2.6.6 过滤器生成

```cpp
void FilterCombiner::GenerateFilters(
    const std::function<void(unique_ptr<Expression> filter)> &callback) {

    // 1. 输出未能处理的过滤器
    for (auto &filter : remaining_filters) {
        callback(std::move(filter));
    }
    remaining_filters.clear();

    // 2. 为每个等价类生成过滤器
    for (auto &entry : equivalence_map) {
        auto &entries = entry.second;
        auto &constant_list = constant_values.find(entry.first)->second;

        // 生成等式过滤器（等价类内的表达式两两相等）
        for (idx_t i = 0; i < entries.size(); i++) {
            for (idx_t k = i + 1; k < entries.size(); k++) {
                auto comparison = make_uniq<BoundComparisonExpression>(
                    ExpressionType::COMPARE_EQUAL,
                    entries[i].get().Copy(),
                    entries[k].get().Copy());
                callback(std::move(comparison));
            }

            // 生成常量比较过滤器
            // 尝试合并为 BETWEEN
            auto lower_index = optional_idx::Invalid();
            auto upper_index = optional_idx::Invalid();

            for (idx_t k = 0; k < constant_list.size(); k++) {
                auto &info = constant_list[k];
                if (IsGreaterThan(info.comparison_type)) {
                    lower_index = k;
                } else if (IsLessThan(info.comparison_type)) {
                    upper_index = k;
                } else {
                    // 等值或不等值，直接输出
                    callback(make_uniq<BoundComparisonExpression>(
                        info.comparison_type,
                        entries[i].get().Copy(),
                        make_uniq<BoundConstantExpression>(info.constant)));
                }
            }

            // 如果有上下界，生成 BETWEEN
            if (lower_index.IsValid() && upper_index.IsValid()) {
                auto between = make_uniq<BoundBetweenExpression>(
                    entries[i].get().Copy(),
                    make_uniq<BoundConstantExpression>(constant_list[lower_index.GetIndex()].constant),
                    make_uniq<BoundConstantExpression>(constant_list[upper_index.GetIndex()].constant),
                    lower_inclusive, upper_inclusive);
                callback(std::move(between));
            } else if (lower_index.IsValid()) {
                // 只有下界
                callback(make_uniq<BoundComparisonExpression>(...));
            } else if (upper_index.IsValid()) {
                // 只有上界
                callback(make_uniq<BoundComparisonExpression>(...));
            }
        }
    }

    // 清空状态
    stored_expressions.clear();
    equivalence_set_map.clear();
    constant_values.clear();
    equivalence_map.clear();
}
```

---

## 2.7 其他算子下推

### 2.7.1 Aggregate 下推

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownAggregate(unique_ptr<LogicalOperator> op) {
    auto &aggr = op->Cast<LogicalAggregate>();

    // 分离过滤器
    // - 只引用分组键的过滤器可以下推
    // - 引用聚合结果的过滤器必须保留
    vector<unique_ptr<Filter>> remaining_filters;

    for (auto &filter : filters) {
        // 检查过滤器是否只引用分组键
        bool can_pushdown = true;
        for (auto &binding : filter->bindings) {
            // ... 检查 binding 是否在 aggr.groups 中
        }

        if (can_pushdown) {
            // 重写列引用，指向分组键的原始列
            // ...
        } else {
            remaining_filters.push_back(std::move(filter));
        }
    }

    // 递归下推到子节点
    op->children[0] = Rewrite(std::move(op->children[0]));

    // 处理剩余过滤器
    filters = std::move(remaining_filters);
    return PushFinalFilters(std::move(op));
}
```

### 2.7.2 Projection 下推

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownProjection(unique_ptr<LogicalOperator> op) {
    auto &proj = op->Cast<LogicalProjection>();

    // 替换过滤器中的列引用
    // 将投影输出的列引用替换为投影表达式
    for (auto &filter : filters) {
        // 例如：过滤器引用 proj.col_0
        // 如果 proj.expressions[0] = a.x + 1
        // 则替换为 a.x + 1
        // ...
    }

    // 继续下推
    return Rewrite(std::move(op->children[0]));
}
```

### 2.7.3 Set Operation 下推

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownSetOperation(unique_ptr<LogicalOperator> op) {
    // UNION：可以下推到两侧
    // INTERSECT：可以下推到两侧
    // EXCEPT：只能下推到左侧

    // 需要将列引用映射到各侧对应的列
    // ...
}
```

---

## 2.8 优化示例

### 示例 1：Inner Join 优化

```sql
-- 原始查询
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'China' AND o.amount > 1000

-- 初始计划
Filter (c.country = 'China' AND o.amount > 1000)
    InnerJoin (o.customer_id = c.id)
        TableScan(orders)
        TableScan(customers)

-- FilterPullup 后
CrossProduct
    TableScan(orders)
    TableScan(customers)
+ Filter (c.country = 'China' AND o.amount > 1000 AND o.customer_id = c.id)

-- FilterPushdown 后
CrossProduct
    Filter (o.amount > 1000)
        TableScan(orders) [table_filter: amount > 1000]
    Filter (c.country = 'China')
        TableScan(customers) [table_filter: country = 'China']
+ Filter (o.customer_id = c.id)

-- Join Order Optimizer 会将 Cross Product + Filter 转回 Join
```

### 示例 2：Left Join → Inner Join

```sql
-- 原始查询
SELECT * FROM a
LEFT JOIN b ON a.id = b.id
WHERE b.value > 10

-- 分析：
-- b.value > 10 会排除 b 为 NULL 的行
-- 因此 Left Join 可以安全转换为 Inner Join

-- 优化后
SELECT * FROM a
INNER JOIN b ON a.id = b.id
WHERE b.value > 10
```

### 示例 3：传递性过滤推导

```sql
-- 原始查询
SELECT * FROM a
JOIN b ON a.id = b.id
WHERE a.id = 100

-- FilterCombiner 分析：
-- 等价类: {a.id, b.id}
-- 常量: {= 100}

-- 生成过滤器：
-- a.id = 100 (原始)
-- b.id = 100 (推导)

-- 优化后
SELECT * FROM a
JOIN b ON a.id = b.id
WHERE a.id = 100 AND b.id = 100
```

### 示例 4：矛盾检测

```sql
-- 原始查询
SELECT * FROM t WHERE x > 10 AND x < 5

-- FilterCombiner 检测到：
-- x > 10 且 x < 5 矛盾

-- 优化后
LogicalEmptyResult  -- 直接返回空结果
```

---

## 2.9 本章小结

本章详细分析了 DuckDB 的谓词优化系统，包括三个核心组件：

### FilterPushdown

| 功能 | 说明 |
|------|------|
| 算子感知下推 | 根据算子类型选择不同的下推策略 |
| Join 类型转换 | Left Join → Inner Join（当过滤器排除 NULL） |
| TableFilter 生成 | 将过滤器转换为 TableFilterSet 用于扫描 |

### FilterPullup

| 功能 | 说明 |
|------|------|
| Inner Join 拆分 | 将 Join 转换为 Cross Product + Filter |
| 统一过滤器位置 | 将嵌套的过滤器提取到顶层 |
| 为后续优化准备 | 配合 FilterPushdown 和 Join Order 使用 |

### FilterCombiner

| 功能 | 说明 |
|------|------|
| 等价类管理 | 合并相等表达式到同一等价类 |
| 范围合并 | 合并多个范围条件（如 x > 5 AND x < 10 → BETWEEN） |
| 传递性推导 | 从 a=b AND a=5 推导出 b=5 |
| 矛盾检测 | 检测不可满足的条件（如 x > 10 AND x < 5） |
| TableFilter 生成 | 生成用于扫描的优化过滤器 |

### 设计亮点

1. **模块化架构**：Pullup、Pushdown、Combiner 各司其职
2. **完整的 Join 支持**：处理所有 Join 类型的不同语义
3. **智能优化**：Left Join → Inner Join 转换
4. **深度集成**：与存储层的 TableFilter 无缝对接
5. **正确性保证**：完善的 NULL 处理和语义保持

下一章将介绍 Join 顺序优化，这是另一个关键的查询优化技术。

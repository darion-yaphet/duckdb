# DuckDB 优化器深度解析：第四章 - 聚合与 Distinct 优化

## 4.1 章节概述

聚合操作（Aggregate）和去重操作（Distinct）是 OLAP 查询中最常见的操作之一。DuckDB 针对这两类操作实现了多种优化策略，包括：

- **重复聚合消除**：删除重复的聚合表达式
- **ORDER BY 聚合优化**：将有序聚合转换为 `arg_min/arg_max`
- **DISTINCT 聚合优化**：移除不必要的 DISTINCT 修饰符
- **聚合过滤下推**：将 GROUP BY 列上的过滤条件下推
- **TopN 优化**：将 ORDER BY + LIMIT 融合为 TopN
- **未使用列移除**：删除未引用的聚合表达式

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     聚合与 Distinct 优化架构                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入：LogicalAggregate / LogicalDistinct                               │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 1. Expression Rewriting (表达式级优化)                        │      │
│  │    - OrderedAggregateOptimizer: 有序聚合 → arg_min/max       │      │
│  │    - DistinctAggregateOptimizer: 移除无用 DISTINCT           │      │
│  │    - DistinctWindowedOptimizer: 窗口函数 DISTINCT 优化       │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 2. CommonAggregateOptimizer (重复聚合消除)                     │      │
│  │    - 检测相同的聚合表达式                                      │      │
│  │    - 删除重复并重映射引用                                      │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 3. FilterPushdown (过滤下推)                                  │      │
│  │    - 将 GROUP BY 列过滤条件下推                               │      │
│  │    - 处理 Grouping Sets 特殊情况                              │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 4. RemoveUnusedColumns (未使用列移除)                         │      │
│  │    - 移除未引用的聚合表达式                                    │      │
│  │    - 空聚合时插入 COUNT(*)                                    │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 5. TopN Optimizer (ORDER BY + LIMIT 优化)                     │      │
│  │    - 将 ORDER BY + LIMIT 融合为 LogicalTopN                   │      │
│  │    - 动态过滤下推到扫描层                                      │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 6. CompressedMaterialization (压缩物化)                       │      │
│  │    - 压缩 GROUP BY 列                                         │      │
│  │    - 压缩 DISTINCT 目标列                                     │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  输出：优化后的聚合/去重计划                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.2 CommonAggregateOptimizer：重复聚合消除

### 4.2.1 优化目标

消除 LogicalAggregate 中重复的聚合表达式，避免重复计算：

```sql
-- 原始查询
SELECT SUM(a), SUM(a), AVG(b), SUM(a) FROM t GROUP BY c;
-- 有三个重复的 SUM(a)

-- 优化后等价于
SELECT SUM(a), SUM(a), AVG(b), SUM(a) FROM t GROUP BY c;
-- 只计算一次 SUM(a)，其他引用复用结果
```

### 4.2.2 实现机制

```cpp
// src/optimizer/common_aggregate_optimizer.cpp

class CommonAggregateOptimizer : public LogicalOperatorVisitor {
public:
    void VisitOperator(LogicalOperator &op) override;

private:
    //! 聚合表达式到新绑定的映射
    column_binding_map_t<ColumnBinding> aggregate_map;

    void ExtractCommonAggregates(LogicalAggregate &aggr);
};
```

### 4.2.3 核心算法

```cpp
void CommonAggregateOptimizer::ExtractCommonAggregates(LogicalAggregate &aggr) {
    expression_map_t<idx_t> aggregate_remap;  // 表达式 → 首次出现的索引
    idx_t total_erased = 0;

    for (idx_t i = 0; i < aggr.expressions.size(); i++) {
        idx_t original_index = i + total_erased;
        auto entry = aggregate_remap.find(*aggr.expressions[i]);

        if (entry == aggregate_remap.end()) {
            // 首次出现，记录索引
            aggregate_remap[*aggr.expressions[i]] = i;

            if (i != original_index) {
                // 之前有表达式被删除，需要重映射绑定
                ColumnBinding original_binding(aggr.aggregate_index, original_index);
                ColumnBinding new_binding(aggr.aggregate_index, i);
                aggregate_map[original_binding] = new_binding;
            }
        } else {
            // 重复表达式，删除并重映射
            total_erased++;
            aggr.expressions.erase_at(i);
            i--;

            ColumnBinding original_binding(aggr.aggregate_index, original_index);
            ColumnBinding new_binding(aggr.aggregate_index, entry->second);
            aggregate_map[original_binding] = new_binding;
        }
    }
}
```

### 4.2.4 引用更新

删除重复聚合后，需要更新所有对删除表达式的引用：

```cpp
unique_ptr<Expression> CommonAggregateOptimizer::VisitReplace(BoundColumnRefExpression &expr,
                                                              unique_ptr<Expression> *expr_ptr) {
    // 检查此列引用是否指向已被重映射的聚合
    auto entry = aggregate_map.find(expr.binding);
    if (entry != aggregate_map.end()) {
        // 更新绑定到新位置
        expr.binding = entry->second;
    }
    return nullptr;
}
```

```
重复聚合消除示例：

原始聚合表达式：
[0] SUM(a)
[1] SUM(a)     ← 与 [0] 重复
[2] AVG(b)
[3] SUM(a)     ← 与 [0] 重复

优化后：
[0] SUM(a)
[1] AVG(b)

重映射：
  原始 [1] → 新 [0]
  原始 [2] → 新 [1]
  原始 [3] → 新 [0]
```

---

## 4.3 OrderedAggregateOptimizer：有序聚合优化

### 4.3.1 优化目标

将带 ORDER BY 子句的特定聚合函数转换为更高效的实现：

```sql
-- 原始查询（使用 FIRST/LAST 聚合）
SELECT FIRST(value ORDER BY time DESC) FROM t GROUP BY id;

-- 转换为
SELECT arg_min_null(value, create_sort_key(time, 'DESC')) FROM t GROUP BY id;
```

### 4.3.2 转换规则

| 原始聚合 | 转换后 |
|---------|-------|
| `FIRST(x ORDER BY ...)` | `arg_min_null(x, create_sort_key(...))` |
| `LAST(x ORDER BY ...)` | `arg_max_null(x, create_sort_key(...))` |
| `ARBITRARY(x ORDER BY ...)` | `arg_min_null(x, create_sort_key(...))` |
| `ANY_VALUE(x ORDER BY ...)` | `arg_min(x, create_sort_key(...))` |

### 4.3.3 实现

```cpp
// src/optimizer/rule/ordered_aggregate_optimizer.cpp

unique_ptr<Expression> OrderedAggregateOptimizer::Apply(
    ClientContext &context, BoundAggregateExpression &aggr,
    vector<unique_ptr<Expression>> &groups,
    optional_ptr<vector<GroupingSet>> grouping_sets, bool &changes_made) {

    if (!aggr.order_bys) {
        return nullptr;  // 没有 ORDER BY
    }

    // 检查聚合函数是否依赖顺序
    if (aggr.function.GetOrderDependent() == AggregateOrderDependent::NOT_ORDER_DEPENDENT) {
        // 不依赖顺序，直接移除 ORDER BY
        aggr.order_bys.reset();
        changes_made = true;
        return nullptr;
    }

    // 尝试简化 ORDER BY（如果 ORDER BY 列在 GROUP BY 中）
    if (aggr.order_bys->Simplify(groups, grouping_sets)) {
        aggr.order_bys.reset();
        changes_made = true;
        return nullptr;
    }

    // 将 FIRST/LAST 转换为 arg_min/arg_max
    const auto &aggr_name = aggr.function.name;
    string arg_xxx_name;
    if (aggr_name == "last") {
        arg_xxx_name = "arg_max_null";
    } else if (aggr_name == "first" || aggr_name == "arbitrary") {
        arg_xxx_name = "arg_min_null";
    } else if (aggr_name == "any_value") {
        arg_xxx_name = "arg_min";
    } else {
        return nullptr;
    }

    // 创建 create_sort_key 函数调用
    FunctionBinder binder(context);
    vector<unique_ptr<Expression>> sort_children;
    for (auto &order : aggr.order_bys->orders) {
        sort_children.emplace_back(std::move(order.expression));
        sort_children.emplace_back(
            make_uniq<BoundConstantExpression>(Value(order.GetOrderModifier())));
    }
    aggr.order_bys.reset();

    auto sort_key = binder.BindScalarFunction(DEFAULT_SCHEMA, "create_sort_key",
                                               std::move(sort_children), error);

    // 创建新的 arg_xxx 聚合
    auto &children = aggr.children;
    children.emplace_back(std::move(sort_key));

    return binder.BindAggregateFunction(bound_function, std::move(children),
                                        std::move(aggr.filter), aggr.aggr_type);
}
```

```
有序聚合转换示例：

FIRST(value ORDER BY time DESC, id ASC)
    ↓
arg_min_null(value, create_sort_key(time, 'DESC', id, 'ASC'))

create_sort_key 函数：
- 将多列排序键编码为单个可比较的二进制字符串
- 支持 ASC/DESC、NULLS FIRST/LAST 等选项
```

---

## 4.4 DistinctAggregateOptimizer：DISTINCT 优化

### 4.4.1 优化目标

移除不必要的 DISTINCT 修饰符：

```sql
-- 原始查询
SELECT COUNT(DISTINCT a), MAX(DISTINCT b) FROM t;
-- MAX 不受 DISTINCT 影响

-- 优化后
SELECT COUNT(DISTINCT a), MAX(b) FROM t;
```

### 4.4.2 函数分类

```cpp
enum class AggregateDistinctDependent {
    NOT_DISTINCT_DEPENDENT,  // 不受 DISTINCT 影响 (MAX, MIN)
    DISTINCT_DEPENDENT       // 受 DISTINCT 影响 (COUNT, SUM, AVG)
};
```

**不受 DISTINCT 影响的聚合：**
- `MAX` - 最大值不变
- `MIN` - 最小值不变
- `FIRST` - 首个值不变
- `LAST` - 最后值不变
- `ANY_VALUE` - 任意值不变

**受 DISTINCT 影响的聚合：**
- `COUNT` - 计数会减少
- `SUM` - 总和会变化
- `AVG` - 平均值会变化

### 4.4.3 实现

```cpp
// src/optimizer/rule/distinct_aggregate_optimizer.cpp

unique_ptr<Expression> DistinctAggregateOptimizer::Apply(
    ClientContext &context, BoundAggregateExpression &aggr, bool &changes_made) {

    if (!aggr.IsDistinct()) {
        return nullptr;  // 没有 DISTINCT
    }

    if (aggr.function.GetDistinctDependent() == AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT) {
        // 不受 DISTINCT 影响，移除修饰符
        aggr.aggr_type = AggregateType::NON_DISTINCT;
        changes_made = true;
    }

    return nullptr;
}
```

### 4.4.4 窗口函数 DISTINCT 优化

```cpp
// 同样适用于窗口函数
unique_ptr<Expression> DistinctWindowedOptimizer::Apply(
    ClientContext &context, BoundWindowExpression &wexpr, bool &changes_made) {

    if (!wexpr.distinct) {
        return nullptr;
    }

    if (!wexpr.aggregate) {
        return nullptr;  // 不是聚合窗口函数
    }

    if (wexpr.aggregate->GetDistinctDependent() == AggregateDistinctDependent::NOT_DISTINCT_DEPENDENT) {
        wexpr.distinct = false;
        changes_made = true;
    }

    return nullptr;
}
```

---

## 4.5 聚合过滤下推

### 4.5.1 下推规则

对于 GROUP BY 列上的过滤条件，可以下推到聚合的子节点：

```sql
-- 原始查询
SELECT SUM(amount)
FROM orders
GROUP BY customer_id
HAVING customer_id > 100;

-- 优化后
SELECT SUM(amount)
FROM orders
WHERE customer_id > 100  -- 过滤条件下推
GROUP BY customer_id;
```

### 4.5.2 实现

```cpp
// src/optimizer/pushdown/pushdown_aggregate.cpp

unique_ptr<LogicalOperator> FilterPushdown::PushdownAggregate(unique_ptr<LogicalOperator> op) {
    auto &aggr = op->Cast<LogicalAggregate>();

    FilterPushdown child_pushdown(optimizer, convert_mark_joins);

    for (idx_t i = 0; i < filters.size(); i++) {
        auto &f = *filters[i];

        // 检查过滤条件是否引用聚合结果
        if (f.bindings.find(aggr.aggregate_index) != f.bindings.end()) {
            continue;  // 引用聚合结果，不能下推
        }

        // 检查过滤条件是否引用 GROUPINGS 函数
        if (f.bindings.find(aggr.groupings_index) != f.bindings.end()) {
            continue;  // 引用 GROUPINGS，不能下推
        }

        // 空分组不能下推
        if (aggr.groups.empty()) {
            continue;
        }

        // 检查过滤条件是否在所有 Grouping Sets 中
        vector<ColumnBinding> bindings;
        ExtractFilterBindings(*f.filter, bindings);

        bool can_pushdown = true;
        for (auto &grp : aggr.grouping_sets) {
            for (auto &binding : bindings) {
                if (grp.find(binding.column_index) == grp.end()) {
                    can_pushdown = false;
                    break;
                }
            }
            if (!can_pushdown) break;
        }

        if (!can_pushdown) {
            continue;
        }

        // 可以下推：将 GROUP BY 列引用替换为原始表达式
        f.filter = ReplaceGroupBindings(aggr, std::move(f.filter));

        if (child_pushdown.AddFilter(std::move(f.filter)) == FilterResult::UNSATISFIABLE) {
            return make_uniq<LogicalEmptyResult>(std::move(op));
        }

        filters.erase_at(i);
        i--;
    }

    // 递归下推到子节点
    op->children[0] = child_pushdown.Rewrite(std::move(op->children[0]));
    return FinishPushdown(std::move(op));
}
```

### 4.5.3 Grouping Sets 限制

对于多个 Grouping Sets，过滤条件必须在所有集合中都存在才能下推：

```sql
-- 查询使用 CUBE
SELECT SUM(amount)
FROM orders
GROUP BY CUBE(region, year)
HAVING region = 'ASIA';

-- region 不在所有 grouping sets 中
-- Grouping Sets: {region, year}, {region}, {year}, {}
-- 过滤条件只能在 {region, year} 和 {region} 中下推
-- 因此不能下推！
```

---

## 4.6 Distinct 过滤下推

### 4.6.1 下推规则

普通 DISTINCT 可以直接下推过滤条件：

```cpp
// src/optimizer/pushdown/pushdown_distinct.cpp

unique_ptr<LogicalOperator> FilterPushdown::PushdownDistinct(unique_ptr<LogicalOperator> op) {
    auto &distinct = op->Cast<LogicalDistinct>();

    if (!distinct.order_by) {
        // 普通 DISTINCT，直接下推
        op->children[0] = Rewrite(std::move(op->children[0]));
        return op;
    }

    // DISTINCT ON 暂不支持下推
    return FinishPushdown(std::move(op));
}
```

### 4.6.2 DISTINCT ON 限制

`DISTINCT ON` 需要保留排序顺序，因此不能随意下推：

```sql
-- DISTINCT ON 需要特殊处理
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, order_date DESC;
```

---

## 4.7 RemoveUnusedColumns：未使用列移除

### 4.7.1 聚合列移除

删除未被引用的聚合表达式：

```cpp
// src/optimizer/remove_unused_columns.cpp

case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
    auto &aggr = op.Cast<LogicalAggregate>();

    // 多 Grouping Sets 时需要所有列
    bool new_root = (aggr.grouping_sets.size() > 1);

    if (!everything_referenced && !new_root) {
        // 移除未使用的聚合表达式
        ClearUnusedExpressions(aggr.expressions, aggr.aggregate_index);

        if (aggr.expressions.empty() && aggr.groups.empty()) {
            // 所有聚合都被移除，插入 COUNT(*)
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
```

### 4.7.2 空聚合处理

当所有聚合表达式都被移除时，需要插入 `COUNT(*)` 保持语义：

```sql
-- 原始查询
SELECT 1 FROM t GROUP BY a;

-- 聚合表达式列表为空，但需要保持分组语义
-- 插入 COUNT(*) 确保正确的分组行为
```

### 4.7.3 DISTINCT 列处理

DISTINCT 需要保留所有列用于去重计算：

```cpp
case LogicalOperatorType::LOGICAL_DISTINCT: {
    auto &distinct = op.Cast<LogicalDistinct>();

    if (distinct.distinct_type == DistinctType::DISTINCT_ON) {
        break;  // DISTINCT ON 有明确的列引用
    }

    // 普通 DISTINCT 需要所有列
    everything_referenced = true;
    break;
}
```

---

## 4.8 TopN Optimizer

### 4.8.1 优化目标

将 `ORDER BY + LIMIT` 组合转换为 TopN 算子，避免完整排序：

```sql
-- 原始查询
SELECT * FROM orders ORDER BY amount DESC LIMIT 10;

-- 逻辑计划转换
-- Limit(10)               →  TopN(10, amount DESC)
--   └── Order(amount DESC)       └── Scan(orders)
--         └── Scan(orders)
```

### 4.8.2 优化条件

```cpp
// src/optimizer/topn_optimizer.cpp

bool TopN::CanOptimize(LogicalOperator &op, optional_ptr<ClientContext> context) {
    if (op.type != LogicalOperatorType::LOGICAL_LIMIT) {
        return false;
    }

    auto &limit = op.Cast<LogicalLimit>();

    // 需要常量 LIMIT
    if (limit.limit_val.Type() != LimitNodeType::CONSTANT_VALUE) {
        return false;
    }

    // OFFSET 需要是常量或不存在
    if (limit.offset_val.Type() == LimitNodeType::EXPRESSION_VALUE) {
        return false;
    }

    // 检查是否应该使用完整排序
    if (child_op->has_estimated_cardinality) {
        auto constant_limit = static_cast<double>(limit.limit_val.GetConstantValue());
        if (limit.offset_val.Type() == LimitNodeType::CONSTANT_VALUE) {
            constant_limit += static_cast<double>(limit.offset_val.GetConstantValue());
        }
        auto child_card = static_cast<double>(child_op->estimated_cardinality);

        // 如果 limit > 0.7% 的基数且 > 5000，使用完整排序更快
        bool limit_is_large = constant_limit > 5000;
        if (constant_limit > child_card * 0.007 && limit_is_large) {
            return false;  // 不优化
        }
    }

    // 子节点必须是 ORDER BY
    auto child = op.children[0].get();
    while (child->type == LogicalOperatorType::LOGICAL_PROJECTION) {
        child = child->children[0].get();
    }

    return child->type == LogicalOperatorType::LOGICAL_ORDER_BY;
}
```

### 4.8.3 TopN 创建

```cpp
unique_ptr<LogicalOperator> TopN::Optimize(unique_ptr<LogicalOperator> op) {
    if (CanOptimize(*op, &context)) {
        vector<unique_ptr<LogicalOperator>> projections;

        // 收集 LIMIT 和 ORDER BY 之间的投影
        auto child = std::move(op->children[0]);
        while (child->type == LogicalOperatorType::LOGICAL_PROJECTION) {
            auto tmp = std::move(child->children[0]);
            projections.push_back(std::move(child));
            child = std::move(tmp);
        }

        auto &order_by = child->Cast<LogicalOrder>();
        auto &limit = op->Cast<LogicalLimit>();

        // 创建 TopN 算子
        auto limit_val = limit.limit_val.GetConstantValue();
        idx_t offset_val = 0;
        if (limit.offset_val.Type() == LimitNodeType::CONSTANT_VALUE) {
            offset_val = limit.offset_val.GetConstantValue();
        }

        auto topn = make_uniq<LogicalTopN>(std::move(order_by.orders), limit_val, offset_val);
        topn->AddChild(std::move(order_by.children[0]));
        op = std::move(topn);

        // 重建投影链
        while (!projections.empty()) {
            auto node = std::move(projections.back());
            node->children[0] = std::move(op);
            op = std::move(node);
            projections.pop_back();
        }
    }

    // 设置动态过滤
    if (op->type == LogicalOperatorType::LOGICAL_TOP_N) {
        PushdownDynamicFilters(op->Cast<LogicalTopN>());
    }

    // 递归处理子节点
    for (auto &child : op->children) {
        child = Optimize(std::move(child));
    }
    return op;
}
```

### 4.8.4 动态过滤下推

TopN 可以将边界值作为动态过滤条件下推到扫描层：

```cpp
void TopN::PushdownDynamicFilters(LogicalTopN &op) {
    // 仅支持整数和字符串类型
    bool nulls_first = op.orders[0].null_order == OrderByNullType::NULLS_FIRST;
    auto &type = op.orders[0].expression->return_type;
    if (!TypeIsIntegral(type.InternalType()) && type.id() != LogicalTypeId::VARCHAR) {
        return;
    }

    // 仅支持 ORDER BY [列引用]
    if (op.orders[0].expression->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
        return;
    }

    // 创建动态过滤
    ExpressionType comparison_type;
    if (op.orders[0].type == OrderType::ASCENDING) {
        // 升序：过滤 C <= boundary
        comparison_type = op.orders.size() == 1 ?
            ExpressionType::COMPARE_LESSTHAN :
            ExpressionType::COMPARE_LESSTHANOREQUALTO;
    } else {
        // 降序：过滤 C >= boundary
        comparison_type = op.orders.size() == 1 ?
            ExpressionType::COMPARE_GREATERTHAN :
            ExpressionType::COMPARE_GREATERTHANOREQUALTO;
    }

    // 创建并下推动态过滤器
    auto filter_data = make_shared_ptr<DynamicFilterData>();
    op.dynamic_filter = filter_data;

    for (auto &target : pushdown_targets) {
        auto dynamic_filter = make_uniq<DynamicFilter>(filter_data);
        target.get.table_filters.PushFilter(column_index, std::move(dynamic_filter));
    }
}
```

```
TopN 动态过滤示例：

SELECT * FROM orders ORDER BY amount DESC LIMIT 10;

执行过程：
1. 扫描开始时，无边界过滤
2. 找到前 10 个最大值后，堆中最小值 = 1000
3. 更新动态过滤：amount > 1000
4. 后续扫描跳过 amount <= 1000 的行
5. 边界持续更新，过滤效果越来越好
```

---

## 4.9 CompressedMaterialization：压缩物化

### 4.9.1 聚合压缩

对 GROUP BY 列进行类型压缩以减少内存使用：

```cpp
// src/optimizer/compressed_materialization/compress_aggregate.cpp

void CompressedMaterialization::CompressAggregate(unique_ptr<LogicalOperator> &op) {
    auto &aggregate = op->Cast<LogicalAggregate>();

    // 多 Grouping Sets 暂不支持
    if (aggregate.grouping_sets.size() > 1) {
        return;
    }

    auto &groups = aggregate.groups;
    auto &group_stats = aggregate.group_stats;

    if (groups.empty() || group_stats.empty()) {
        return;
    }

    // 收集引用的绑定（不能压缩）
    column_binding_set_t referenced_bindings;
    for (auto &group : groups) {
        if (group->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
            GetReferencedBindings(*group, referenced_bindings);
        }
    }

    // 聚合函数中的绑定也不能压缩
    for (auto &expr : aggregate.expressions) {
        auto &aggr_expr = expr->Cast<BoundAggregateExpression>();
        for (auto &child : aggr_expr.children) {
            GetReferencedBindings(*child, referenced_bindings);
        }
    }

    // 创建压缩信息
    CompressedMaterializationInfo info(*op, {0}, referenced_bindings);

    // 尝试压缩
    CreateProjections(op, info);
}
```

### 4.9.2 DISTINCT 压缩

```cpp
// src/optimizer/compressed_materialization/compress_distinct.cpp

void CompressedMaterialization::CompressDistinct(unique_ptr<LogicalOperator> &op) {
    auto &distinct = op->Cast<LogicalDistinct>();
    auto &distinct_targets = distinct.distinct_targets;

    // 收集不能压缩的绑定
    column_binding_set_t referenced_bindings;
    for (auto &target : distinct_targets) {
        if (target->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) {
            GetReferencedBindings(*target, referenced_bindings);
        }
    }

    // 创建压缩信息
    CompressedMaterializationInfo info(*op, {0}, referenced_bindings);

    // 创建绑定映射
    const auto bindings = distinct.GetColumnBindings();
    for (idx_t col_idx = 0; col_idx < bindings.size(); col_idx++) {
        info.binding_map.emplace(bindings[col_idx],
                                  CMBindingInfo(bindings[col_idx], types[col_idx]));
    }

    CreateProjections(op, info);
}
```

```
压缩物化示例：

原始查询：GROUP BY varchar_column
VARCHAR 类型可能占用大量内存

压缩优化：
1. 分析列统计信息
2. 如果 distinct count 较小，创建字典编码
3. 用整数代替字符串进行分组
4. 结果输出时解压

压缩效果：
- 减少 Hash Table 内存占用
- 提高比较操作速度
```

---

## 4.10 优化流程整合

### 4.10.1 优化器调用顺序

在 `Optimizer::Optimize()` 中，聚合相关优化的执行顺序：

```cpp
// src/optimizer/optimizer.cpp

// 阶段 1: 表达式重写（包含聚合优化规则）
ExpressionRewriter rewriter(*this);
rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));
rewriter.VisitOperator(*plan);

// 阶段 2: 过滤下推（包含聚合过滤下推）
FilterPushdown filter_pushdown(*this);
plan = filter_pushdown.Rewrite(std::move(plan));

// 阶段 3: 未使用列移除
RemoveUnusedColumns unused(binder, context);
unused.VisitOperator(*plan);

// 阶段 4: TopN 优化
TopN topn(context);
plan = topn.Optimize(std::move(plan));

// 阶段 5: 重复聚合消除
CommonAggregateOptimizer common_aggregate;
common_aggregate.VisitOperator(*plan);

// 阶段 6: 压缩物化
CompressedMaterialization compress(context);
compress.Compress(plan);
```

### 4.10.2 完整优化示例

```sql
-- 原始查询
SELECT region, SUM(amount), MAX(amount), SUM(amount)
FROM orders
WHERE year = 2023
GROUP BY region
HAVING region != 'UNKNOWN'
ORDER BY SUM(amount) DESC
LIMIT 10;
```

**优化步骤：**

1. **DistinctAggregateOptimizer**：无变化（没有 DISTINCT 聚合）

2. **FilterPushdown**：
   - `region != 'UNKNOWN'` 下推到 GROUP BY 之下
   - `year = 2023` 继续下推到扫描

3. **RemoveUnusedColumns**：无变化（所有聚合都被使用）

4. **CommonAggregateOptimizer**：
   - 检测到两个 `SUM(amount)` 重复
   - 删除一个，重映射引用

5. **TopN**：
   - 将 `ORDER BY + LIMIT` 融合为 TopN
   - 设置动态过滤

6. **CompressedMaterialization**：
   - 如果 `region` 列 distinct count 较小
   - 创建字典压缩

```
最终计划：
TopN(10, SUM(amount) DESC)
├── DynamicFilter(SUM(amount) > boundary)
└── Aggregate
    ├── Groups: region
    ├── Aggregates: SUM(amount), MAX(amount)  # 只计算一次 SUM
    └── Filter: region != 'UNKNOWN'
        └── Scan: orders
            └── Filter: year = 2023
```

---

## 4.11 源文件索引

| 组件 | 文件路径 |
|------|---------|
| CommonAggregateOptimizer | `src/optimizer/common_aggregate_optimizer.cpp` |
| OrderedAggregateOptimizer | `src/optimizer/rule/ordered_aggregate_optimizer.cpp` |
| DistinctAggregateOptimizer | `src/optimizer/rule/distinct_aggregate_optimizer.cpp` |
| 聚合过滤下推 | `src/optimizer/pushdown/pushdown_aggregate.cpp` |
| Distinct 过滤下推 | `src/optimizer/pushdown/pushdown_distinct.cpp` |
| TopN Optimizer | `src/optimizer/topn_optimizer.cpp` |
| RemoveUnusedColumns | `src/optimizer/remove_unused_columns.cpp` |
| 聚合压缩 | `src/optimizer/compressed_materialization/compress_aggregate.cpp` |
| Distinct 压缩 | `src/optimizer/compressed_materialization/compress_distinct.cpp` |

---

## 4.12 本章小结

本章详细介绍了 DuckDB 针对聚合和 Distinct 操作的优化策略：

1. **CommonAggregateOptimizer**：
   - 检测并消除重复的聚合表达式
   - 自动重映射列引用

2. **OrderedAggregateOptimizer**：
   - 移除顺序无关聚合的 ORDER BY
   - 将 FIRST/LAST 转换为 arg_min/arg_max

3. **DistinctAggregateOptimizer**：
   - 移除不必要的 DISTINCT 修饰符
   - 支持聚合函数和窗口函数

4. **聚合过滤下推**：
   - 将 GROUP BY 列过滤条件下推
   - 正确处理 Grouping Sets

5. **TopN 优化**：
   - ORDER BY + LIMIT 融合
   - 动态边界过滤下推

6. **压缩物化**：
   - GROUP BY 列压缩
   - DISTINCT 目标列压缩

关键设计特点：
- **多阶段优化**：不同优化在合适的阶段执行
- **语义保持**：空聚合时插入 COUNT(*) 保持分组语义
- **代价感知**：TopN 根据基数估算选择是否优化
- **动态过滤**：运行时更新过滤边界提高效率

下一章将介绍子查询优化。

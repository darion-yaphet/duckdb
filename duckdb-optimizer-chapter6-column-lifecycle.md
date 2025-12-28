# DuckDB 查询优化器深度解析：第六章 - 列生命周期优化

## 6.1 列生命周期优化概述

列生命周期优化是 DuckDB 优化器中减少数据传输和处理量的关键技术。其核心思想是：**只处理真正需要的列，并尽早丢弃不再使用的列**。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        列生命周期优化组件                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ RemoveUnusedColumns                                              │   │
│  │ • 删除未被引用的列                                                │   │
│  │ • 更新 LogicalGet 的 column_ids                                  │   │
│  │ • 移除未使用的聚合表达式                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ColumnLifetimeAnalyzer                                           │   │
│  │ • 分析列的使用范围                                                │   │
│  │ • 生成 projection_map 提前丢弃不需要的列                          │   │
│  │ • 优化 Join/Order/Filter 的列传递                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ RemoveDuplicateGroups                                            │   │
│  │ • 移除重复的 GROUP BY 列                                          │   │
│  │ • 更新列绑定映射                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ColumnBindingReplacer                                            │   │
│  │ • 计划变更后更新列绑定                                            │   │
│  │ • 支持类型替换                                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ LateMaterialization                                              │   │
│  │ • 延迟物化：先获取 row-id，后获取数据列                           │   │
│  │ • 适用于 TopN/Limit/Sample                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ CompressedMaterialization                                        │   │
│  │ • 物化时压缩数据                                                  │   │
│  │ • 使用统计信息优化存储类型                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.1.1 ColumnBinding 基础

在深入优化算法之前，先理解 DuckDB 的列绑定机制：

```cpp
// src/include/duckdb/planner/column_binding.hpp
struct ColumnBinding {
    idx_t table_index;   // 表/算子的索引
    idx_t column_index;  // 列在表中的位置

    bool operator==(const ColumnBinding &other) const {
        return table_index == other.table_index &&
               column_index == other.column_index;
    }
};
```

每个算子会输出一组列，这些列由 `table_index` 和 `column_index` 唯一标识。当列在计划树中向上传递时，需要追踪每个列的来源和引用。

---

## 6.2 RemoveUnusedColumns

`RemoveUnusedColumns` 是列裁剪的核心优化器，它遍历逻辑计划，识别并移除未被引用的列。

### 6.2.1 类设计

```cpp
// src/include/duckdb/optimizer/remove_unused_columns.hpp
struct ReferencedColumn {
    vector<reference<BoundColumnRefExpression>> bindings;  // 引用此列的表达式
    vector<ColumnIndex> child_columns;  // 子列索引（用于 STRUCT 类型）
};

class RemoveUnusedColumns : public BaseColumnPruner {
public:
    RemoveUnusedColumns(Binder &binder, ClientContext &context, bool is_root = false)
        : binder(binder), context(context), everything_referenced(is_root) {}

    void VisitOperator(LogicalOperator &op) override;

private:
    Binder &binder;
    ClientContext &context;
    bool everything_referenced;  // 根节点默认引用所有列

    // 核心数据结构：跟踪哪些列被引用
    column_binding_map_t<ReferencedColumn> column_references;
};
```

### 6.2.2 算法流程

```cpp
// src/optimizer/remove_unused_columns.cpp:61
void RemoveUnusedColumns::VisitOperator(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
        auto &aggr = op.Cast<LogicalAggregate>();

        if (!everything_referenced) {
            // 删除未引用的聚合表达式
            ClearUnusedExpressions(aggr.expressions, aggr.aggregate_index);

            if (aggr.expressions.empty() && aggr.groups.empty()) {
                // 如果所有聚合都被删除，添加 COUNT(*)
                // 确保聚合算子仍然有输出
                auto count_star_fun = CountStarFun::GetFunction();
                aggr.expressions.push_back(
                    function_binder.BindAggregateFunction(count_star_fun, {}, nullptr));
            }
        }

        // 递归处理子节点（重新开始引用收集）
        RemoveUnusedColumns remove(binder, context, false);
        remove.VisitOperatorExpressions(op);
        remove.VisitOperator(*op.children[0]);
        return;
    }

    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
        auto &comp_join = op.Cast<LogicalComparisonJoin>();

        if (comp_join.join_type == JoinType::INNER) {
            // 对于内连接的等值条件 (X = Y)，
            // 可以将右侧列的引用替换为左侧列的引用
            // 这减少了从 Hash Table 中提取的列数
            for (auto &cond : comp_join.conditions) {
                if (cond.comparison != ExpressionType::COMPARE_EQUAL) continue;
                if (!IsColumnRef(cond.left) || !IsColumnRef(cond.right)) continue;
                if (cond.left->return_type.IsFloating()) continue;  // 浮点数有 +0/-0 问题

                auto &lhs_col = cond.left->Cast<BoundColumnRefExpression>();
                auto &rhs_col = cond.right->Cast<BoundColumnRefExpression>();

                // 将右侧列的所有引用替换为左侧列
                auto colrefs = column_references.find(rhs_col.binding);
                if (colrefs != column_references.end()) {
                    for (auto &entry : colrefs->second.bindings) {
                        entry.get().binding = lhs_col.binding;
                        AddBinding(entry.get());
                    }
                    column_references.erase(rhs_col.binding);
                }
            }
        }
        break;
    }

    case LogicalOperatorType::LOGICAL_PROJECTION: {
        auto &proj = op.Cast<LogicalProjection>();

        if (!everything_referenced) {
            // 删除未引用的投影表达式
            ClearUnusedExpressions(proj.expressions, proj.table_index);

            if (proj.expressions.empty()) {
                // 保留至少一个常量（如 EXISTS 子查询）
                proj.expressions.push_back(
                    make_uniq<BoundConstantExpression>(Value::INTEGER(42)));
            }
        }

        // 递归处理投影表达式引用的列
        RemoveUnusedColumns remove(binder, context);
        for (auto &expr : proj.expressions) {
            remove.VisitExpression(&expr);
        }
        remove.VisitOperator(*op.children[0]);
        return;
    }

    case LogicalOperatorType::LOGICAL_GET: {
        auto &get = op.Cast<LogicalGet>();
        RemoveColumnsFromLogicalGet(get);
        return;
    }
    // ...
    }
}
```

### 6.2.3 LogicalGet 列裁剪

```cpp
// src/optimizer/remove_unused_columns.cpp:309
void RemoveUnusedColumns::RemoveColumnsFromLogicalGet(LogicalGet &get) {
    if (everything_referenced) return;
    if (!get.function.projection_pushdown) return;  // 表函数不支持列裁剪

    auto final_column_ids = get.GetColumnIds();

    // 创建列索引的选择向量
    vector<idx_t> proj_sel;
    for (idx_t col_idx = 0; col_idx < final_column_ids.size(); col_idx++) {
        proj_sel.push_back(col_idx);
    }

    // 清除未使用的列
    ClearUnusedExpressions(proj_sel, get.table_index, false);

    // 处理表过滤器引用的列（即使未在输出中使用，也需要扫描）
    for (auto &filter : get.table_filters.filters) {
        // 确保过滤器列被保留
        VisitExpression(&filter_expr);
    }

    // 构建新的 column_ids
    vector<ColumnIndex> column_ids;
    for (idx_t idx = 0; idx < col_sel.size(); idx++) {
        column_ids.emplace_back(final_column_ids[col_sel[idx]]);
    }

    if (column_ids.empty()) {
        // 至少扫描一列（用于 EXISTS 等场景）
        column_ids.emplace_back(get.GetAnyColumn());
    }

    get.SetColumnIds(std::move(column_ids));
}
```

### 6.2.4 STRUCT 列的处理

对于 STRUCT 类型，DuckDB 支持子列级别的裁剪：

```cpp
// src/optimizer/remove_unused_columns.cpp:396
bool BaseColumnPruner::HandleStructExtractRecursive(
    Expression &expr,
    optional_ptr<BoundColumnRefExpression> &colref,
    vector<idx_t> &indexes) {

    auto &function = expr.Cast<BoundFunctionExpression>();
    if (function.function.name != "struct_extract_at" &&
        function.function.name != "struct_extract" &&
        function.function.name != "array_extract") {
        return false;
    }

    auto &bind_data = function.bind_info->Cast<StructExtractBindData>();
    indexes.push_back(bind_data.index);

    // 递归处理嵌套的 struct_extract
    if (function.children[0]->GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF) {
        colref = &function.children[0]->Cast<BoundColumnRefExpression>();
        return true;
    }

    return HandleStructExtractRecursive(*function.children[0], colref, indexes);
}
```

**示例：**

```sql
-- 对于 STRUCT 类型的列，只提取需要的子列
CREATE TABLE users (id INT, info STRUCT(name VARCHAR, age INT, email VARCHAR));

-- 只需要 info.name
SELECT info.name FROM users;

-- 列裁剪后，只扫描 id 和 info.name 子列
```

---

## 6.3 ColumnLifetimeAnalyzer

`ColumnLifetimeAnalyzer` 进一步优化列的生命周期，通过生成 `projection_map` 来指示算子只需要传递哪些列。

### 6.3.1 类设计

```cpp
// src/include/duckdb/optimizer/column_lifetime_analyzer.hpp
class ColumnLifetimeAnalyzer : public LogicalOperatorVisitor {
public:
    explicit ColumnLifetimeAnalyzer(Optimizer &optimizer, LogicalOperator &root,
                                     bool is_root = false)
        : optimizer(optimizer), root(root), everything_referenced(is_root) {}

    void VisitOperator(LogicalOperator &op) override;

private:
    Optimizer &optimizer;
    LogicalOperator &root;
    bool everything_referenced;
    column_binding_set_t column_references;  // 被引用的列集合

    void ExtractUnusedColumnBindings(const vector<ColumnBinding> &bindings,
                                      column_binding_set_t &unused_bindings);
    static void GenerateProjectionMap(vector<ColumnBinding> bindings,
                                       column_binding_set_t &unused_bindings,
                                       vector<idx_t> &projection_map);
};
```

### 6.3.2 Projection Map 生成

`projection_map` 告诉执行器只需要传递哪些列：

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:28
void ColumnLifetimeAnalyzer::GenerateProjectionMap(
    vector<ColumnBinding> bindings,
    column_binding_set_t &unused_bindings,
    vector<idx_t> &projection_map) {

    projection_map.clear();
    if (unused_bindings.empty()) return;

    // 只添加被使用的列到 projection_map
    for (idx_t i = 0; i < bindings.size(); i++) {
        if (unused_bindings.find(bindings[i]) == unused_bindings.end()) {
            projection_map.push_back(i);
        }
    }

    // 如果所有列都被使用，清空 map（表示传递所有列）
    if (projection_map.size() == bindings.size()) {
        projection_map.clear();
    }
}
```

### 6.3.3 针对不同算子的处理

```cpp
// src/optimizer/column_lifetime_analyzer.cpp:57
void ColumnLifetimeAnalyzer::VisitOperator(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN: {
        auto &comp_join = op.Cast<LogicalComparisonJoin>();
        if (everything_referenced) break;

        // 只对 Hash Join 生成 projection_map
        idx_t has_range = 0;
        if (!comp_join.HasEquality(has_range)) return;

        // 找出左右子节点中未使用的列
        column_binding_set_t lhs_unused, rhs_unused;
        ExtractUnusedColumnBindings(op.children[0]->GetColumnBindings(), lhs_unused);
        ExtractUnusedColumnBindings(op.children[1]->GetColumnBindings(), rhs_unused);

        StandardVisitOperator(op);

        // 生成左右 projection_map
        GenerateProjectionMap(op.children[0]->GetColumnBindings(),
                              lhs_unused, comp_join.left_projection_map);
        GenerateProjectionMap(op.children[1]->GetColumnBindings(),
                              rhs_unused, comp_join.right_projection_map);
        return;
    }

    case LogicalOperatorType::LOGICAL_ORDER_BY: {
        auto &order = op.Cast<LogicalOrder>();
        if (everything_referenced) break;

        column_binding_set_t unused_bindings;
        ExtractUnusedColumnBindings(op.children[0]->GetColumnBindings(), unused_bindings);

        StandardVisitOperator(op);

        // ORDER BY 只需要传递上层需要的列 + 排序键
        GenerateProjectionMap(op.children[0]->GetColumnBindings(),
                              unused_bindings, order.projection_map);
        return;
    }

    case LogicalOperatorType::LOGICAL_FILTER: {
        auto &filter = op.Cast<LogicalFilter>();
        if (everything_referenced) break;

        column_binding_set_t unused_bindings;
        ExtractUnusedColumnBindings(op.children[0]->GetColumnBindings(), unused_bindings);

        StandardVisitOperator(op);

        // Filter 可以立即丢弃后续不需要的列
        GenerateProjectionMap(op.children[0]->GetColumnBindings(),
                              unused_bindings, filter.projection_map);
        return;
    }
    // ...
    }
}
```

### 6.3.4 使用示例

```sql
-- 原始查询
SELECT name FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 100
ORDER BY o.date;

-- 列生命周期分析结果:
-- 1. Scan orders: 需要 customer_id, amount, date, name? (取决于 name 来源)
-- 2. Scan customers: 需要 id (用于 Join), name
-- 3. Join: left_projection_map 只包含 name, date 的索引
--          right_projection_map 只包含 name 的索引
-- 4. Filter: projection_map 只包含 name, date
-- 5. Order: projection_map 只包含 name
```

---

## 6.4 RemoveDuplicateGroups

`RemoveDuplicateGroups` 优化器移除 GROUP BY 子句中重复的列。

### 6.4.1 问题场景

```sql
-- 由于子查询展开或其他优化，可能产生重复的分组列
SELECT customer_id, customer_id, SUM(amount)
FROM orders
GROUP BY customer_id, customer_id;  -- 重复的分组列
```

### 6.4.2 实现

```cpp
// src/optimizer/remove_duplicate_groups.cpp:21
void RemoveDuplicateGroups::VisitAggregate(LogicalAggregate &aggr) {
    if (!aggr.grouping_functions.empty()) return;  // GROUPING() 函数需要保留所有组

    auto &groups = aggr.groups;

    // 使用 map 检测重复列
    column_binding_map_t<idx_t> duplicate_map;
    vector<pair<idx_t, idx_t>> duplicates;

    for (idx_t group_idx = 0; group_idx < groups.size(); group_idx++) {
        const auto &group = groups[group_idx];
        if (group->GetExpressionType() != ExpressionType::BOUND_COLUMN_REF) continue;

        const auto &colref = group->Cast<BoundColumnRefExpression>();
        const auto &binding = colref.binding;

        auto it = duplicate_map.find(binding);
        if (it == duplicate_map.end()) {
            duplicate_map.emplace(binding, group_idx);
        } else {
            // 发现重复，记录 (保留的索引, 要删除的索引)
            duplicates.emplace_back(it->second, group_idx);
        }
    }

    if (duplicates.empty()) return;

    // 从后向前删除重复的组（避免索引偏移）
    sort(duplicates.begin(), duplicates.end(),
         [](const pair<idx_t, idx_t> &lhs, const pair<idx_t, idx_t> &rhs) {
             return lhs.second > rhs.second;
         });

    // 构建绑定映射
    column_binding_map_t<ColumnBinding> group_binding_map;
    for (idx_t group_idx = 0; group_idx < groups.size(); group_idx++) {
        group_binding_map.emplace(ColumnBinding(aggr.group_index, group_idx),
                                   ColumnBinding(aggr.group_index, group_idx));
    }

    for (auto &duplicate : duplicates) {
        const auto &remaining_idx = duplicate.first;
        const auto &removed_idx = duplicate.second;

        // 删除重复的组
        groups.erase_at(removed_idx);

        // 更新 grouping_sets
        for (auto &grouping_set : aggr.grouping_sets) {
            if (grouping_set.erase(removed_idx) != 0) {
                grouping_set.insert(remaining_idx);
            }
            // 更新大于 removed_idx 的索引
            // ...
        }

        // 更新绑定映射
        auto it = group_binding_map.find(ColumnBinding(aggr.group_index, removed_idx));
        it->second.column_index = remaining_idx;

        for (auto &map_entry : group_binding_map) {
            if (map_entry.second.column_index > removed_idx) {
                map_entry.second.column_index--;
            }
        }
    }

    // 替换所有引用旧绑定的表达式
    for (const auto &map_entry : group_binding_map) {
        auto it = column_references.find(map_entry.first);
        if (it != column_references.end()) {
            for (auto expr : it->second) {
                expr.get().binding = map_entry.second;
            }
        }
    }
}
```

---

## 6.5 ColumnBindingReplacer

`ColumnBindingReplacer` 是一个工具类，用于在计划变更后批量更新列绑定。

### 6.5.1 设计

```cpp
// src/include/duckdb/optimizer/column_binding_replacer.hpp
struct ReplacementBinding {
    ColumnBinding old_binding;
    ColumnBinding new_binding;
    bool replace_type;       // 是否也替换类型
    LogicalType new_type;
};

class ColumnBindingReplacer : LogicalOperatorVisitor {
public:
    vector<ReplacementBinding> replacement_bindings;
    optional_ptr<LogicalOperator> stop_operator;  // 在此算子停止遍历

    void VisitOperator(LogicalOperator &op) override;
    void VisitExpression(unique_ptr<Expression> *expression) override;
};
```

### 6.5.2 实现

```cpp
// src/optimizer/column_binding_replacer.cpp:18
void ColumnBindingReplacer::VisitOperator(LogicalOperator &op) {
    if (stop_operator && stop_operator.get() == &op) {
        return;  // 到达停止点
    }
    VisitOperatorChildren(op);
    VisitOperatorExpressions(op);
}

void ColumnBindingReplacer::VisitExpression(unique_ptr<Expression> *expression) {
    auto &expr = *expression;
    if (expr->GetExpressionClass() == ExpressionClass::BOUND_COLUMN_REF) {
        auto &bound_column_ref = expr->Cast<BoundColumnRefExpression>();

        // 查找并替换绑定
        for (const auto &replace_binding : replacement_bindings) {
            if (bound_column_ref.binding == replace_binding.old_binding) {
                bound_column_ref.binding = replace_binding.new_binding;
                if (replace_binding.replace_type) {
                    bound_column_ref.return_type = replace_binding.new_type;
                }
            }
        }
    }
    VisitExpressionChildren(**expression);
}
```

### 6.5.3 使用场景

```cpp
// 典型使用模式：在插入新投影后更新绑定
void SomeOptimizer::InsertProjection(unique_ptr<LogicalOperator> &op) {
    // 保存原始绑定
    auto old_bindings = op->GetColumnBindings();

    // 创建新投影
    auto projection = make_uniq<LogicalProjection>(new_table_index, ...);
    projection->children.push_back(std::move(op));
    op = std::move(projection);

    // 获取新绑定
    auto new_bindings = op->GetColumnBindings();

    // 创建替换器
    ColumnBindingReplacer replacer;
    for (idx_t i = 0; i < old_bindings.size(); i++) {
        replacer.replacement_bindings.emplace_back(old_bindings[i], new_bindings[i]);
    }

    // 不遍历新插入的投影
    replacer.stop_operator = op.get();

    // 更新计划树中的所有引用
    replacer.VisitOperator(root);
}
```

---

## 6.6 Late Materialization (延迟物化)

Late Materialization 是一种高级优化技术，特别适用于 TopN/Limit/Sample 场景。

### 6.6.1 核心思想

```
传统执行:
  Scan(所有列) -> Filter -> Sort -> TopN -> 输出

延迟物化:
  1. Scan(row_id + 排序列) -> Filter -> Sort -> TopN -> 获取 row_id 集合
  2. 用 row_id 回表获取需要的列
  3. 输出
```

### 6.6.2 适用条件

```cpp
// src/optimizer/late_materialization.cpp:153
bool LateMaterialization::TryLateMaterialization(unique_ptr<LogicalOperator> &op) {
    // 条件1: 排序/过滤需要的列少于表的总列数
    if (column_references.size() >= get.GetColumnIds().size()) {
        return false;
    }

    // 条件2: 表支持延迟物化
    if (!get.function.late_materialization) {
        return false;
    }

    // 条件3: 表支持 row_id 获取
    if (!get.function.get_row_id_columns) {
        return false;
    }

    row_id_column_ids = get.function.get_row_id_columns(context, get.bind_data.get());
    // ... 执行转换
}
```

### 6.6.3 转换过程

```cpp
// src/optimizer/late_materialization.cpp:234
unique_ptr<LogicalOperator> LateMaterialization::Optimize(unique_ptr<LogicalOperator> op) {
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_TOP_N: {
        auto &top_n = op->Cast<LogicalTopN>();
        if (top_n.limit > max_row_count) break;

        if (TryLateMaterialization(op)) {
            // 转换成功，返回新计划
            // 新计划结构:
            // Projection(最终列)
            //   -> Order(恢复顺序)
            //     -> SemiJoin(row_id)
            //       -> LHS: Scan(所有列)
            //       -> RHS: TopN -> Scan(row_id + 排序列)
            return op;
        }
        break;
    }
    // LIMIT 和 SAMPLE 类似处理
    }
    return op;
}
```

### 6.6.4 示例

```sql
-- 原始查询: 获取按日期排序的前10条订单
SELECT * FROM orders
ORDER BY date
LIMIT 10;

-- 假设 orders 表有 20 列

-- 优化前: 扫描所有20列，排序，取前10
-- 优化后:
-- 1. 扫描 (row_id, date)，排序，取前10的 row_id
-- 2. 用 row_id 回表获取10行的20列
-- 3. 按 row_id 排序恢复原顺序
```

```
优化后的计划:
┌──────────────────────────────┐
│ Projection (final columns)   │
├──────────────────────────────┤
│ Order (by row_id)            │
├──────────────────────────────┤
│ Semi Join (row_id)           │
├──────────────────────────────┤
│ ├── Scan orders (all cols)   │ -- LHS: 回表获取所有列
│ └── TopN (limit 10)          │ -- RHS: 只需 row_id + date
│     └── Scan orders          │
│         (row_id, date)       │
└──────────────────────────────┘
```

---

## 6.7 Compressed Materialization (压缩物化)

Compressed Materialization 利用统计信息，在物化阶段对列进行压缩。

### 6.7.1 核心思想

```
对于需要物化的数据（如排序、聚合），可以:
1. 根据统计信息确定值的范围
2. 使用更小的类型存储数据
3. 完成操作后解压回原始类型
```

### 6.7.2 压缩策略

```cpp
// src/optimizer/compressed_materialization.cpp:345
unique_ptr<CompressExpression> CompressedMaterialization::GetIntegralCompress(
    unique_ptr<Expression> input, const BaseStatistics &stats) {

    const auto &type = input->return_type;

    // 条件1: 类型至少是 2 字节
    if (GetTypeIdSize(type.InternalType()) == 1) {
        return nullptr;
    }

    LogicalType cast_type;
    Value min;

    if (!stats.CanHaveNoNull()) {
        // 全是 NULL，使用最小类型
        cast_type = LogicalType::UTINYINT;
    } else if (NumericStats::HasMinMax(stats)) {
        // 根据范围选择最小类型
        auto range_value = GetIntegralRangeValue(context, type, stats);

        const auto range = UBigIntValue::Get(range_value);
        if (range <= NumericLimits<uint8_t>().Maximum()) {
            cast_type = LogicalType::UTINYINT;
        } else if (range <= NumericLimits<uint16_t>().Maximum()) {
            cast_type = LogicalType::USMALLINT;
        } else if (range <= NumericLimits<uint32_t>().Maximum()) {
            cast_type = LogicalType::UINTEGER;
        } else {
            cast_type = LogicalType::UBIGINT;
        }

        min = NumericStats::Min(stats);
    } else {
        return nullptr;  // 没有统计信息
    }

    // 检查压缩是否有益
    if (GetTypeIdSize(cast_type.InternalType()) >= GetTypeIdSize(type.InternalType())) {
        return nullptr;
    }

    // 创建压缩函数: compressed = (value - min)
    auto compress_function = CMIntegralCompressFun::GetFunction(type, cast_type);
    // ...
}
```

### 6.7.3 字符串压缩

```cpp
// src/optimizer/compressed_materialization.cpp:408
unique_ptr<CompressExpression> CompressedMaterialization::GetStringCompress(
    unique_ptr<Expression> input, const BaseStatistics &stats) {

    // 根据最大字符串长度选择类型
    if (StringStats::HasMaxStringLength(stats)) {
        max_string_length = StringStats::MaxStringLength(stats);

        // 短字符串可以压缩为整数类型
        for (const auto &compressed_type : CMUtils::StringTypes()) {
            if (max_string_length < GetTypeIdSize(compressed_type.InternalType())) {
                cast_type = compressed_type;
                break;
            }
        }
    }
    // 创建字符串压缩函数
    // ...
}
```

### 6.7.4 适用场景

```cpp
// src/optimizer/compressed_materialization.cpp:72
void CompressedMaterialization::Compress(unique_ptr<LogicalOperator> &op) {
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        CompressAggregate(op);  // 压缩分组键
        break;
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
        CompressComparisonJoin(op);  // 压缩 Join 键
        break;
    case LogicalOperatorType::LOGICAL_DISTINCT:
        CompressDistinct(op);  // 压缩 DISTINCT 列
        break;
    case LogicalOperatorType::LOGICAL_ORDER_BY:
        CompressOrder(op);  // 压缩排序键
        break;
    }
}
```

### 6.7.5 示例

```sql
-- 假设 age 列统计信息: min=0, max=120
SELECT COUNT(*) FROM users GROUP BY age;

-- 压缩优化:
-- 原始: age 是 INTEGER (4 bytes)
-- 压缩: 范围 0-120 只需 UTINYINT (1 byte)

-- 压缩后的执行:
-- 1. 添加压缩投影: compressed_age = CAST(age - 0 AS UTINYINT)
-- 2. 使用 compressed_age 进行分组
-- 3. 添加解压投影: age = CAST(compressed_age AS INTEGER) + 0
```

---

## 6.8 优化器调用顺序

列生命周期优化在优化器流程中的位置：

```cpp
// src/optimizer/optimizer.cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan) {
    // ... 早期优化 ...

    // 列生命周期优化（在物理计划生成前）
    RunOptimizer(OptimizerType::REMOVE_DUPLICATE_GROUPS, [&]() {
        RemoveDuplicateGroups groups;
        groups.VisitOperator(*plan);
    });

    RunOptimizer(OptimizerType::REMOVE_UNUSED_COLUMNS, [&]() {
        RemoveUnusedColumns unused(binder, context);
        unused.VisitOperator(*plan);
    });

    // 统计信息传播后
    RunOptimizer(OptimizerType::COLUMN_LIFETIME, [&]() {
        ColumnLifetimeAnalyzer analyzer(*this, *plan);
        analyzer.VisitOperator(*plan);
    });

    RunOptimizer(OptimizerType::LATE_MATERIALIZATION, [&]() {
        LateMaterialization late(*this);
        plan = late.Optimize(std::move(plan));
    });

    RunOptimizer(OptimizerType::COMPRESSED_MATERIALIZATION, [&]() {
        CompressedMaterialization compress(*this, *plan, statistics_map);
        compress.Compress(plan);
    });

    // ... 后续优化 ...
}
```

---

## 6.9 源码索引

| 组件 | 文件路径 | 主要功能 |
|------|----------|----------|
| RemoveUnusedColumns | `src/optimizer/remove_unused_columns.cpp` | 移除未使用的列 |
| ColumnLifetimeAnalyzer | `src/optimizer/column_lifetime_analyzer.cpp` | 生成 projection_map |
| ColumnBindingReplacer | `src/optimizer/column_binding_replacer.cpp` | 更新列绑定 |
| RemoveDuplicateGroups | `src/optimizer/remove_duplicate_groups.cpp` | 移除重复分组列 |
| LateMaterialization | `src/optimizer/late_materialization.cpp` | 延迟物化优化 |
| CompressedMaterialization | `src/optimizer/compressed_materialization.cpp` | 压缩物化优化 |
| 头文件 | `src/include/duckdb/optimizer/` | 接口定义 |

---

## 6.10 本章小结

本章详细介绍了 DuckDB 的列生命周期优化机制：

1. **RemoveUnusedColumns**: 核心列裁剪优化器，删除未被引用的列，支持 STRUCT 子列级别裁剪
2. **ColumnLifetimeAnalyzer**: 生成 projection_map，让算子提前丢弃不需要的列
3. **RemoveDuplicateGroups**: 移除 GROUP BY 中的重复列
4. **ColumnBindingReplacer**: 计划变更后的列绑定更新工具
5. **Late Materialization**: 延迟物化，先处理轻量级操作，后回表获取数据
6. **Compressed Materialization**: 利用统计信息压缩物化数据

这些优化共同作用，显著减少了查询执行过程中的数据传输量和处理量：

- **减少 I/O**: 只扫描需要的列
- **减少内存使用**: 更早丢弃不需要的列
- **提高 CPU 效率**: 压缩后的数据更紧凑，cache 友好
- **优化 Join 性能**: 减少 Hash Table 中存储的列数

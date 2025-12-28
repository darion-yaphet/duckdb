# DuckDB 查询优化器深度解析：第七章 - 统计信息与基数估计

## 7.1 统计信息概述

统计信息是查询优化器做出明智决策的基础。DuckDB 的统计传播系统 (StatisticsPropagator) 在逻辑计划中自底向上传播统计信息，用于：

1. **谓词消除**: 如果统计信息表明过滤条件永远为真/假，可以消除该过滤
2. **基数估计**: 估算每个算子输出的行数，用于代价计算
3. **计划优化**: 基于统计信息进行更激进的优化决策
4. **聚合计算**: 直接使用统计信息计算某些聚合，无需扫描数据

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       统计传播系统架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ StatisticsPropagator                                             │   │
│  │ • 自底向上遍历逻辑计划                                            │   │
│  │ • 为每个算子收集/传播统计信息                                      │   │
│  │ • 更新 statistics_map                                            │   │
│  └────────────────────────────────────────────────────────────────┬┘   │
│                                                                    │    │
│           ┌────────────────────┬───────────────────────┐          │    │
│           │                    │                       │          │    │
│           ▼                    ▼                       ▼          │    │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │    │
│  │ Operator Stats   │ │ Expression Stats │ │ BaseStatistics  │      │    │
│  │ 传播             │ │ 传播             │ │ 类型            │      │    │
│  ├─────────────────┤ ├─────────────────┤ ├─────────────────┤      │    │
│  │ • Get           │ │ • ColumnRef     │ │ • NumericStats  │      │    │
│  │ • Filter        │ │ • Comparison    │ │ • StringStats   │      │    │
│  │ • Join          │ │ • Function      │ │ • NodeStatistics│      │    │
│  │ • Aggregate     │ │ • Constant      │ │                 │      │    │
│  │ • Projection    │ │ • Cast          │ │                 │      │    │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │    │
│                                                                    │    │
│  ┌─────────────────────────────────────────────────────────────────┘    │
│  │ FilterPropagateResult                                                │
│  │ • FILTER_ALWAYS_TRUE: 过滤器永远为真                                 │
│  │ • FILTER_ALWAYS_FALSE: 过滤器永远为假                                │
│  │ • FILTER_TRUE_OR_NULL: 过滤器为真或 NULL                             │
│  │ • FILTER_FALSE_OR_NULL: 过滤器为假或 NULL                            │
│  │ • NO_PRUNING_POSSIBLE: 无法确定                                      │
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7.2 统计信息类型

### 7.2.1 BaseStatistics

`BaseStatistics` 是所有统计信息的基类：

```cpp
// src/include/duckdb/storage/statistics/base_statistics.hpp
class BaseStatistics {
public:
    // 统计信息类型
    StatisticsType GetStatsType() const;

    // NULL 值相关
    bool CanHaveNull() const;
    bool CanHaveNoNull() const;
    void Set(StatsInfo info);  // 设置 CAN_HAVE_NULL_VALUES / CANNOT_HAVE_NULL_VALUES

    // 合并统计信息
    void Merge(const BaseStatistics &other);

    // 创建空统计信息
    static BaseStatistics CreateEmpty(LogicalType type);
    static BaseStatistics CreateUnknown(LogicalType type);
};
```

### 7.2.2 NumericStats

用于数值类型的统计信息：

```cpp
// src/include/duckdb/storage/statistics/numeric_stats.hpp
class NumericStats {
public:
    // 检查是否有 min/max 信息
    static bool HasMinMax(const BaseStatistics &stats);

    // 获取 min/max 值
    static Value Min(const BaseStatistics &stats);
    static Value Max(const BaseStatistics &stats);

    // 设置 min/max 值
    static void SetMin(BaseStatistics &stats, const Value &value);
    static void SetMax(BaseStatistics &stats, const Value &value);
};
```

### 7.2.3 StringStats

用于字符串类型的统计信息：

```cpp
// src/include/duckdb/storage/statistics/string_stats.hpp
class StringStats {
public:
    // 获取字符串的 min/max
    static string_t Min(const BaseStatistics &stats);
    static string_t Max(const BaseStatistics &stats);

    // 最大字符串长度
    static bool HasMaxStringLength(const BaseStatistics &stats);
    static uint32_t MaxStringLength(const BaseStatistics &stats);
};
```

### 7.2.4 NodeStatistics

表示算子级别的统计信息（主要是基数）：

```cpp
// src/include/duckdb/storage/statistics/node_statistics.hpp
struct NodeStatistics {
    idx_t estimated_cardinality;  // 估计的行数
    idx_t max_cardinality;        // 最大可能行数
    bool has_estimated_cardinality;
    bool has_max_cardinality;
};
```

---

## 7.3 StatisticsPropagator 类

### 7.3.1 类设计

```cpp
// src/include/duckdb/optimizer/statistics_propagator.hpp
class StatisticsPropagator {
public:
    StatisticsPropagator(Optimizer &optimizer, LogicalOperator &root);

    // 主入口：传播整个计划的统计信息
    unique_ptr<NodeStatistics> PropagateStatistics(unique_ptr<LogicalOperator> &node_ptr);

    // 获取统计信息映射
    column_binding_map_t<unique_ptr<BaseStatistics>> GetStatisticsMap();

private:
    Optimizer &optimizer;
    ClientContext &context;
    optional_ptr<LogicalOperator> root;

    // 核心数据结构：列绑定 -> 统计信息
    column_binding_map_t<unique_ptr<BaseStatistics>> statistics_map;

    // 当前节点的统计信息
    unique_ptr<NodeStatistics> node_stats;
};
```

### 7.3.2 主传播函数

```cpp
// src/optimizer/statistics_propagator.cpp:40
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(
    LogicalOperator &node, unique_ptr<LogicalOperator> &node_ptr) {

    unique_ptr<NodeStatistics> result;

    // 根据算子类型分发到特定的传播函数
    switch (node.type) {
    case LogicalOperatorType::LOGICAL_GET:
        result = PropagateStatistics(node.Cast<LogicalGet>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_FILTER:
        result = PropagateStatistics(node.Cast<LogicalFilter>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_PROJECTION:
        result = PropagateStatistics(node.Cast<LogicalProjection>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
    case LogicalOperatorType::LOGICAL_JOIN:
        result = PropagateStatistics(node.Cast<LogicalJoin>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        result = PropagateStatistics(node.Cast<LogicalAggregate>(), node_ptr);
        break;
    // ... 其他算子
    default:
        result = PropagateChildren(node, node_ptr);
    }

    // 统计传播后，尝试压缩物化
    if (!optimizer.OptimizerDisabled(OptimizerType::COMPRESSED_MATERIALIZATION)) {
        CompressedMaterialization compressed(optimizer, *root, statistics_map);
        compressed.Compress(node_ptr);
    }

    return result;
}
```

---

## 7.4 算子级别的统计传播

### 7.4.1 LogicalGet 统计传播

从存储层获取初始统计信息：

```cpp
// src/optimizer/statistics/operator/propagate_get.cpp:114
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(
    LogicalGet &get, unique_ptr<LogicalOperator> &node_ptr) {

    // 1. 获取基数估计
    if (get.function.cardinality) {
        node_stats = get.function.cardinality(context, get.bind_data.get());
    }

    // 2. 获取列统计信息
    if (get.function.statistics) {
        auto &column_ids = get.GetColumnIds();
        for (idx_t i = 0; i < column_ids.size(); i++) {
            auto stats = get.function.statistics(
                context, get.bind_data.get(), column_ids[i].GetPrimaryIndex());
            if (stats) {
                ColumnBinding binding(get.table_index, i);
                statistics_map.insert(make_pair(binding, std::move(stats)));
            }
        }
    }

    // 3. 应用表过滤器并更新统计信息
    for (auto &filter : get.table_filters.filters) {
        auto &stats = statistics_map[binding];
        auto propagate_result = PropagateTableFilter(stats_binding, stats, *filter);

        switch (propagate_result) {
        case FilterPropagateResult::FILTER_ALWAYS_TRUE:
            // 过滤器永远为真，可以移除
            get.table_filters.filters.erase(column);
            break;
        case FilterPropagateResult::FILTER_ALWAYS_FALSE:
            // 过滤器永远为假，替换为空结果
            ReplaceWithEmptyResult(node_ptr);
            return make_uniq<NodeStatistics>(0U, 0U);
        default:
            // 更新统计信息
            UpdateFilterStatistics(stats, *filter);
            break;
        }
    }

    return std::move(node_stats);
}
```

### 7.4.2 Filter 统计传播

处理过滤条件并更新统计信息：

```cpp
// src/optimizer/statistics/operator/propagate_filter.cpp:248
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(
    LogicalFilter &filter, unique_ptr<LogicalOperator> &node_ptr) {

    // 1. 先传播子节点的统计信息
    node_stats = PropagateStatistics(filter.children[0]);

    // 2. 如果子节点变成空结果，则当前节点也是空结果
    if (filter.children[0]->type == LogicalOperatorType::LOGICAL_EMPTY_RESULT) {
        ReplaceWithEmptyResult(node_ptr);
        return make_uniq<NodeStatistics>(0U, 0U);
    }

    // 3. 处理每个过滤条件
    for (idx_t i = 0; i < filter.expressions.size(); i++) {
        auto &condition = filter.expressions[i];
        auto prune_result = HandleFilter(condition);

        switch (prune_result) {
        case FilterPropagateResult::FILTER_ALWAYS_TRUE:
            // 条件永远为真，移除
            filter.expressions.erase_at(i);
            i--;
            if (filter.expressions.empty() && filter.projection_map.empty()) {
                // 所有条件都移除了，用子节点替换
                node_ptr = std::move(filter.children[0]);
            }
            break;

        case FilterPropagateResult::FILTER_FALSE_OR_NULL:
            // 条件永远为假，整个结果为空
            ReplaceWithEmptyResult(node_ptr);
            return make_uniq<NodeStatistics>(0U, 0U);

        default:
            // 无法消除，更新统计信息
            break;
        }
    }

    return std::move(node_stats);
}
```

### 7.4.3 Join 统计传播

处理连接条件，更新统计并可能消除连接：

```cpp
// src/optimizer/statistics/operator/propagate_join.cpp:17
void StatisticsPropagator::PropagateStatistics(
    LogicalComparisonJoin &join, unique_ptr<LogicalOperator> &node_ptr) {

    for (idx_t i = 0; i < join.conditions.size(); i++) {
        auto &condition = join.conditions[i];

        // 获取左右两侧的统计信息
        const auto stats_left = PropagateExpression(condition.left);
        const auto stats_right = PropagateExpression(condition.right);

        if (stats_left && stats_right) {
            // 根据统计信息判断条件是否可以裁剪
            auto prune_result = PropagateComparison(
                *stats_left, *stats_right, condition.comparison);

            // 保存 Join 统计信息（用于 Perfect Hash Join）
            join.join_stats.push_back(stats_left->ToUnique());
            join.join_stats.push_back(stats_right->ToUnique());

            switch (prune_result) {
            case FilterPropagateResult::FILTER_ALWAYS_FALSE:
                // 条件永远为假
                switch (join.join_type) {
                case JoinType::INNER:
                case JoinType::SEMI:
                    // Inner/Semi Join 结果为空
                    ReplaceWithEmptyResult(node_ptr);
                    return;
                case JoinType::ANTI:
                    // Anti Join 返回左侧
                    node_ptr = std::move(join.children[0]);
                    return;
                case JoinType::LEFT:
                    // Left Join 右侧为空
                    ReplaceWithEmptyResult(join.children[1]);
                    return;
                // ...
                }
                break;

            case FilterPropagateResult::FILTER_ALWAYS_TRUE:
                if (join.conditions.size() > 1) {
                    // 多个条件，移除这个永真的条件
                    join.conditions.erase_at(i);
                    join.join_stats.clear();
                    i--;
                } else {
                    // 唯一的条件永远为真，转换为 Cross Product
                    if (join.join_type == JoinType::INNER) {
                        auto cross = LogicalCrossProduct::Create(
                            std::move(join.children[0]),
                            std::move(join.children[1]));
                        node_ptr = std::move(cross);
                        return;
                    }
                }
                break;
            }
        }

        // 更新两侧的统计信息
        if (join.join_type == JoinType::INNER || join.join_type == JoinType::SEMI) {
            UpdateFilterStatistics(*condition.left, *condition.right, condition.comparison);

            // 尝试将统计信息下推为过滤器
            CreateFilterFromJoinStats(join.children[0], condition.left, ...);
            CreateFilterFromJoinStats(join.children[1], condition.right, ...);
        }
    }
}
```

### 7.4.4 Aggregate 统计传播

聚合算子的统计传播和优化：

```cpp
// src/optimizer/statistics/operator/propagate_aggregate.cpp:236
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(
    LogicalAggregate &aggr, unique_ptr<LogicalOperator> &node_ptr) {

    // 1. 传播子节点统计
    node_stats = PropagateStatistics(aggr.children[0]);

    // 2. 传播分组键的统计信息
    aggr.group_stats.resize(aggr.groups.size());
    for (idx_t group_idx = 0; group_idx < aggr.groups.size(); group_idx++) {
        auto stats = PropagateExpression(aggr.groups[group_idx]);
        aggr.group_stats[group_idx] = stats ? stats->ToUnique() : nullptr;

        if (stats) {
            ColumnBinding group_binding(aggr.group_index, group_idx);
            statistics_map[group_binding] = std::move(stats);
        }
    }

    // 3. 传播聚合表达式的统计信息
    for (idx_t agg_idx = 0; agg_idx < aggr.expressions.size(); agg_idx++) {
        auto stats = PropagateExpression(aggr.expressions[agg_idx]);
        if (stats) {
            ColumnBinding agg_binding(aggr.aggregate_index, agg_idx);
            statistics_map[agg_binding] = std::move(stats);
        }
    }

    // 4. 尝试直接使用统计信息计算聚合结果
    TryExecuteAggregates(aggr, node_ptr);

    return std::move(node_stats);
}
```

---

## 7.5 表达式级别的统计传播

### 7.5.1 比较表达式

```cpp
// src/optimizer/statistics/expression/propagate_comparison.cpp:8
FilterPropagateResult StatisticsPropagator::PropagateComparison(
    BaseStatistics &lstats, BaseStatistics &rstats, ExpressionType comparison) {

    // 只处理数值类型
    if (!lstats.GetType().IsNumeric()) {
        return FilterPropagateResult::NO_PRUNING_POSSIBLE;
    }

    if (!NumericStats::HasMinMax(lstats) || !NumericStats::HasMinMax(rstats)) {
        return FilterPropagateResult::NO_PRUNING_POSSIBLE;
    }

    bool has_null = lstats.CanHaveNull() || rstats.CanHaveNull();

    switch (comparison) {
    case ExpressionType::COMPARE_EQUAL:
        // l = r: 如果 l.min > r.max 或 r.min > l.max，则永远为假
        if (NumericStats::Min(lstats) > NumericStats::Max(rstats) ||
            NumericStats::Min(rstats) > NumericStats::Max(lstats)) {
            return has_null ? FilterPropagateResult::FILTER_FALSE_OR_NULL
                           : FilterPropagateResult::FILTER_ALWAYS_FALSE;
        }
        return FilterPropagateResult::NO_PRUNING_POSSIBLE;

    case ExpressionType::COMPARE_GREATERTHAN:
        // l > r: 如果 l.min > r.max，则永远为真
        if (NumericStats::Min(lstats) > NumericStats::Max(rstats)) {
            return has_null ? FilterPropagateResult::FILTER_TRUE_OR_NULL
                           : FilterPropagateResult::FILTER_ALWAYS_TRUE;
        }
        // 如果 r.min >= l.max，则永远为假
        if (NumericStats::Min(rstats) >= NumericStats::Max(lstats)) {
            return has_null ? FilterPropagateResult::FILTER_FALSE_OR_NULL
                           : FilterPropagateResult::FILTER_ALWAYS_FALSE;
        }
        return FilterPropagateResult::NO_PRUNING_POSSIBLE;

    // 类似处理其他比较类型...
    }
}
```

### 7.5.2 函数表达式

函数可以注册统计传播回调：

```cpp
// src/optimizer/statistics/expression/propagate_function.cpp:6
unique_ptr<BaseStatistics> StatisticsPropagator::PropagateExpression(
    BoundFunctionExpression &func, unique_ptr<Expression> &expr_ptr) {

    // 收集参数的统计信息
    vector<BaseStatistics> stats;
    for (idx_t i = 0; i < func.children.size(); i++) {
        auto stat = PropagateExpression(func.children[i]);
        if (!stat) {
            stats.push_back(BaseStatistics::CreateUnknown(func.children[i]->return_type));
        } else {
            stats.push_back(stat->Copy());
        }
    }

    // 调用函数的统计回调
    if (func.function.HasStatisticsCallback()) {
        FunctionStatisticsInput input(func, func.bind_info.get(), stats, &expr_ptr);
        return func.function.GetStatisticsCallback()(context, input);
    }

    return nullptr;
}
```

---

## 7.6 统计信息的更新

### 7.6.1 过滤条件更新统计

当过滤条件被应用后，需要更新列的统计信息：

```cpp
// src/optimizer/statistics/operator/propagate_filter.cpp:49
void StatisticsPropagator::UpdateFilterStatistics(
    BaseStatistics &stats, ExpressionType comparison_type, const Value &constant) {

    // 非 DISTINCT 比较移除 NULL
    if (!IsCompareDistinct(comparison_type)) {
        stats.Set(StatsInfo::CANNOT_HAVE_NULL_VALUES);
    }

    if (!stats.GetType().IsNumeric() || !NumericStats::HasMinMax(stats)) {
        return;
    }

    switch (comparison_type) {
    case ExpressionType::COMPARE_LESSTHAN:
    case ExpressionType::COMPARE_LESSTHANOREQUALTO:
        // X < constant 或 X <= constant
        // max 变为 constant
        NumericStats::SetMax(stats, constant);
        break;

    case ExpressionType::COMPARE_GREATERTHAN:
    case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
        // X > constant 或 X >= constant
        // min 变为 constant
        NumericStats::SetMin(stats, constant);
        break;

    case ExpressionType::COMPARE_EQUAL:
        // X = constant
        // min 和 max 都变为 constant
        NumericStats::SetMin(stats, constant);
        NumericStats::SetMax(stats, constant);
        break;
    }
}
```

### 7.6.2 两列比较更新统计

当两列进行比较时，更新两列的统计信息：

```cpp
// src/optimizer/statistics/operator/propagate_filter.cpp:87
void StatisticsPropagator::UpdateFilterStatistics(
    BaseStatistics &lstats, BaseStatistics &rstats, ExpressionType comparison_type) {

    // 示例: left = [-50, 250], right = [-100, 100]

    switch (comparison_type) {
    case ExpressionType::COMPARE_LESSTHAN:
    case ExpressionType::COMPARE_LESSTHANOREQUALTO:
        // LEFT < RIGHT 或 LEFT <= RIGHT
        // left.max 最多等于 right.max
        if (NumericStats::Max(lstats) > NumericStats::Max(rstats)) {
            NumericStats::SetMax(lstats, NumericStats::Max(rstats));
        }
        // right.min 最多等于 left.min
        if (NumericStats::Min(rstats) < NumericStats::Min(lstats)) {
            NumericStats::SetMin(rstats, NumericStats::Min(lstats));
        }
        // 结果: left: [-50, 100], right: [-50, 100]
        break;

    case ExpressionType::COMPARE_EQUAL:
        // LEFT = RIGHT
        // 取最紧的边界
        if (NumericStats::Min(lstats) > NumericStats::Min(rstats)) {
            NumericStats::SetMin(rstats, NumericStats::Min(lstats));
        } else {
            NumericStats::SetMin(lstats, NumericStats::Min(rstats));
        }
        if (NumericStats::Max(lstats) < NumericStats::Max(rstats)) {
            NumericStats::SetMax(rstats, NumericStats::Max(lstats));
        } else {
            NumericStats::SetMax(lstats, NumericStats::Max(rstats));
        }
        break;
    // ...
    }
}
```

---

## 7.7 基于统计的聚合优化

### 7.7.1 直接从统计计算聚合

对于简单的 COUNT(*)、MIN、MAX 聚合，如果表支持分区统计，可以直接使用统计信息计算结果：

```cpp
// src/optimizer/statistics/operator/propagate_aggregate.cpp:97
void StatisticsPropagator::TryExecuteAggregates(
    LogicalAggregate &aggr, unique_ptr<LogicalOperator> &node_ptr) {

    // 条件1: 无分组键
    if (!aggr.groups.empty()) return;

    // 条件2: 检查聚合函数类型
    vector<idx_t> count_star_idxs;
    vector<ColumnBinding> min_max_bindings;
    vector<unique_ptr<ValueComparator>> comparators;

    for (auto &aggr_expr : aggr.expressions) {
        const string &fun_name = aggr_expr.function.name;
        if (fun_name == "min" || fun_name == "max") {
            // MIN/MAX 必须是简单列引用
            if (aggr_expr.children[0]->type != ExpressionType::BOUND_COLUMN_REF) {
                return;
            }
            min_max_bindings.push_back(col_ref.binding);
            comparators.push_back(GetComparator(fun_name, col_ref.return_type));
        } else if (fun_name == "count_star") {
            count_star_idxs.push_back(i);
        } else {
            // 其他聚合函数不支持
            return;
        }
    }

    // 条件3: 跳过 Projection 找到 LogicalGet
    auto &get = FindGet(aggr);
    if (!get.function.get_partition_stats) return;
    if (!get.table_filters.filters.empty()) return;

    // 获取分区统计
    auto partition_stats = get.function.get_partition_stats(context, input);
    if (partition_stats.empty()) return;

    // 使用统计信息计算结果
    vector<unique_ptr<Expression>> agg_results;

    // 计算 MIN/MAX
    for (idx_t agg_idx = 0; agg_idx < min_max_bindings.size(); agg_idx++) {
        Value agg_result;
        // 从第一个分区获取初始值
        TryGetValueFromStats(partition_stats[0], column_index, *comparator, agg_result);

        // 与其他分区比较
        for (idx_t i = 1; i < partition_stats.size(); i++) {
            Value rhs;
            TryGetValueFromStats(partition_stats[i], column_index, *comparator, rhs);
            if (!comparator->Compare(agg_result, rhs)) {
                agg_result = rhs;
            }
        }
        agg_results.push_back(make_uniq<BoundConstantExpression>(agg_result));
    }

    // 计算 COUNT(*)
    if (!count_star_idxs.empty()) {
        idx_t count = 0;
        for (const auto &stats : partition_stats) {
            if (stats.count_type == CountType::COUNT_APPROXIMATE) {
                return;  // 只支持精确计数
            }
            count += stats.count;
        }
        for (auto idx : count_star_idxs) {
            agg_results.insert(..., make_uniq<BoundConstantExpression>(Value::BIGINT(count)));
        }
    }

    // 替换聚合为常量表达式
    auto expression_get = make_uniq<LogicalExpressionGet>(..., std::move(agg_results));
    node_ptr = std::move(expression_get);
}
```

### 7.7.2 示例

```sql
-- 查询
SELECT COUNT(*), MIN(age), MAX(age) FROM users;

-- 如果 users 表有 3 个分区，统计信息如下:
-- 分区1: count=1000, age: [18, 35]
-- 分区2: count=2000, age: [22, 50]
-- 分区3: count=1500, age: [20, 65]

-- 直接从统计计算:
-- COUNT(*) = 1000 + 2000 + 1500 = 4500
-- MIN(age) = min(18, 22, 20) = 18
-- MAX(age) = max(35, 50, 65) = 65

-- 替换为:
SELECT 4500 AS "count", 18 AS "min", 65 AS "max";
```

---

## 7.8 基数估计在 Join Order 中的应用

统计信息在 Join Order 优化中用于估计中间结果的基数：

```cpp
// src/optimizer/join_order/cardinality_estimator.cpp
double CardinalityEstimator::EstimateCardinality(JoinRelationSet &set) {
    // 基于等价类的基数估计

    // 基础公式: card(A ⋈ B) = card(A) * card(B) / max(distinct(A.key), distinct(B.key))

    // 对于多表连接，使用乘法/除法公式
    // numerator = product of all base table cardinalities
    // denominator = product of equivalence class max distincts

    double numerator = 1.0;
    for (auto &rel : set.relations) {
        numerator *= GetBaseCardinality(rel);
    }

    double denominator = 1.0;
    for (auto &equiv_class : equivalence_classes) {
        if (equiv_class.InvolvesRelations(set)) {
            denominator *= equiv_class.GetMaxDistinct();
        }
    }

    return numerator / denominator;
}
```

---

## 7.9 统计传播与优化的交互

### 7.9.1 从 Join 统计创建过滤器

当 Join 条件更新了统计信息后，可以将更紧的范围下推为过滤器：

```cpp
// src/optimizer/statistics/operator/propagate_join.cpp:323
void StatisticsPropagator::CreateFilterFromJoinStats(
    unique_ptr<LogicalOperator> &child,
    unique_ptr<Expression> &expr,
    const BaseStatistics &stats_before,
    const BaseStatistics &stats_after) {

    // 只处理整数列
    if (!expr->return_type.IsIntegral()) return;
    if (!NumericStats::HasMinMax(stats_before) || !NumericStats::HasMinMax(stats_after)) return;

    auto min_before = NumericStats::Min(stats_before);
    auto max_before = NumericStats::Max(stats_before);
    auto min_after = NumericStats::Min(stats_after);
    auto max_after = NumericStats::Max(stats_after);

    vector<unique_ptr<Expression>> filter_exprs;

    // 如果 min 增加了，添加 >= 过滤器
    if (min_after > min_before) {
        filter_exprs.push_back(make_uniq<BoundComparisonExpression>(
            ExpressionType::COMPARE_GREATERTHANOREQUALTO,
            expr->Copy(),
            make_uniq<BoundConstantExpression>(min_after)));
    }

    // 如果 max 减少了，添加 <= 过滤器
    if (max_after < max_before) {
        filter_exprs.push_back(make_uniq<BoundComparisonExpression>(
            ExpressionType::COMPARE_LESSTHANOREQUALTO,
            expr->Copy(),
            make_uniq<BoundConstantExpression>(max_after)));
    }

    if (filter_exprs.empty()) return;

    // 创建并下推过滤器
    auto filter = make_uniq<LogicalFilter>();
    filter->children.push_back(std::move(child));
    child = std::move(filter);

    for (auto &expr : filter_exprs) {
        child->expressions.push_back(std::move(expr));
    }

    // 下推过滤器
    FilterPushdown filter_pushdown(optimizer, false);
    child = filter_pushdown.Rewrite(std::move(child));
}
```

### 7.9.2 示例

```sql
-- 原始查询
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'China';

-- 统计信息:
-- orders.customer_id: [1, 1000000]
-- customers.id: [1, 50000] (过滤后 country='China')

-- 优化过程:
-- 1. Join 条件 o.customer_id = c.id
-- 2. c.id 范围 [1, 50000] 传播到 o.customer_id
-- 3. 创建新过滤器: o.customer_id <= 50000
-- 4. 下推到 orders 扫描

-- 优化后的计划:
-- Join
--   ├── Filter(customer_id <= 50000)
--   │     └── Scan orders
--   └── Filter(country = 'China')
--         └── Scan customers
```

---

## 7.10 源码索引

| 组件 | 文件路径 | 主要功能 |
|------|----------|----------|
| StatisticsPropagator | `src/optimizer/statistics_propagator.cpp` | 主统计传播器 |
| Get 传播 | `src/optimizer/statistics/operator/propagate_get.cpp` | 扫描算子统计 |
| Filter 传播 | `src/optimizer/statistics/operator/propagate_filter.cpp` | 过滤算子统计 |
| Join 传播 | `src/optimizer/statistics/operator/propagate_join.cpp` | 连接算子统计 |
| Aggregate 传播 | `src/optimizer/statistics/operator/propagate_aggregate.cpp` | 聚合算子统计 |
| Comparison 传播 | `src/optimizer/statistics/expression/propagate_comparison.cpp` | 比较表达式 |
| Function 传播 | `src/optimizer/statistics/expression/propagate_function.cpp` | 函数表达式 |
| BaseStatistics | `src/include/duckdb/storage/statistics/base_statistics.hpp` | 基础统计类 |
| NumericStats | `src/include/duckdb/storage/statistics/numeric_stats.hpp` | 数值统计 |
| StringStats | `src/include/duckdb/storage/statistics/string_stats.hpp` | 字符串统计 |

---

## 7.11 本章小结

本章详细介绍了 DuckDB 的统计信息系统和基数估计机制：

1. **统计信息类型**: BaseStatistics、NumericStats、StringStats、NodeStatistics
2. **统计传播器**: StatisticsPropagator 自底向上传播统计信息
3. **算子级别传播**: Get、Filter、Join、Aggregate 等算子的特定处理
4. **表达式级别传播**: 比较、函数等表达式的统计推导
5. **统计更新**: 根据过滤条件收紧统计范围
6. **优化应用**:
   - 消除永真/永假的过滤条件
   - 直接从统计计算聚合结果
   - 从 Join 统计创建新的过滤器
   - 为 Join Order 提供基数估计

统计信息是优化器的眼睛，使优化器能够做出基于数据特征的明智决策，而不是盲目地应用规则。

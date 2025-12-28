# DuckDB 查询优化器深度解析：第九章 - 物理计划生成

## 9.1 章节概述

物理计划生成是查询优化的最后一步，将优化后的逻辑计划转换为可执行的物理计划。这一阶段不仅涉及算子映射，还包括关键的算法选择决策。

```
┌─────────────────────────────────────────────────────────────────┐
│                    物理计划生成流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ 优化后的     │    │    类型      │    │    列绑定    │      │
│  │ 逻辑计划     │───▶│    解析      │───▶│    解析      │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                 │               │
│                                                 ▼               │
│                                          ┌──────────────┐      │
│                                          │   物理计划   │      │
│                                          │    生成      │      │
│                                          └──────┬───────┘      │
│                                                 │               │
│                                                 ▼               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    算法选择决策                          │   │
│  │                                                         │   │
│  │  Join:                                                  │   │
│  │  • Hash Join (等值 + 大数据量)                          │   │
│  │  • Nested Loop Join (小数据量)                          │   │
│  │  • Merge Join (范围条件)                                │   │
│  │  • IEJoin (两个范围条件)                                │   │
│  │                                                         │   │
│  │  Aggregate:                                             │   │
│  │  • Ungrouped Aggregate (无分组)                         │   │
│  │  • Perfect Hash Aggregate (整型小范围)                  │   │
│  │  • Partitioned Aggregate (分区对齐)                     │   │
│  │  • Hash Aggregate (通用)                                │   │
│  │                                                         │   │
│  │  Distinct:                                              │   │
│  │  • Hash Aggregate 实现                                  │   │
│  │  • 需要时添加 FIRST 聚合                                │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                 │               │
│                                                 ▼               │
│                                          ┌──────────────┐      │
│                                          │ PhysicalPlan │      │
│                                          │    (输出)    │      │
│                                          └──────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.2 PhysicalPlanGenerator 架构

### 9.2.1 核心类设计

```cpp
// src/include/duckdb/execution/physical_plan_generator.hpp

class PhysicalPlan {
public:
    explicit PhysicalPlan(Allocator &allocator) : arena(allocator) {}

    // 创建物理算子（使用 Arena 分配器）
    template <class T, class... ARGS>
    PhysicalOperator &Make(ARGS &&... args) {
        static_assert(std::is_base_of<PhysicalOperator, T>::value);
        auto ptr = arena.Make<T>(*this, std::forward<ARGS>(args)...);
        ops.push_back(*ptr);
        return *ptr;
    }

    PhysicalOperator &Root() { return *root; }
    void SetRoot(PhysicalOperator &op) { root = op; }

private:
    ArenaAllocator arena;                          // 内存分配器
    vector<reference<PhysicalOperator>> ops;       // 所有算子引用
    optional_ptr<PhysicalOperator> root;           // 根算子
};

class PhysicalPlanGenerator {
public:
    explicit PhysicalPlanGenerator(ClientContext &context);

    // 主入口：从逻辑计划生成物理计划
    unique_ptr<PhysicalPlan> Plan(unique_ptr<LogicalOperator> logical);

    // 创建特定算子的物理计划
    PhysicalOperator &CreatePlan(LogicalOperator &op);

    // 辅助方法
    template <class T, class... ARGS>
    PhysicalOperator &Make(ARGS &&... args) {
        return physical_plan->Make<T>(std::forward<ARGS>(args)...);
    }

private:
    ClientContext &context;
    unique_ptr<PhysicalPlan> physical_plan;

    // CTE 相关状态
    unordered_map<idx_t, shared_ptr<ColumnDataCollection>> recursive_cte_tables;
    unordered_map<idx_t, shared_ptr<ColumnDataCollection>> recurring_cte_tables;
    unordered_map<idx_t, vector<const_reference<PhysicalOperator>>> materialized_ctes;
};
```

### 9.2.2 生成流程

```cpp
// src/execution/physical_plan_generator.cpp

unique_ptr<PhysicalPlan> PhysicalPlanGenerator::Plan(unique_ptr<LogicalOperator> op) {
    auto &plan = ResolveAndPlan(std::move(op));
    plan.Verify();
    return std::move(physical_plan);
}

PhysicalOperator &PhysicalPlanGenerator::ResolveAndPlan(unique_ptr<LogicalOperator> op) {
    auto &profiler = QueryProfiler::Get(context);

    // Step 1: 解析算子类型
    profiler.StartPhase(MetricType::PHYSICAL_PLANNER_RESOLVE_TYPES);
    op->ResolveOperatorTypes();
    profiler.EndPhase();

    // Step 2: 列绑定解析
    // 将 BoundColumnRefExpression 转换为 BoundReferenceExpression
    profiler.StartPhase(MetricType::PHYSICAL_PLANNER_COLUMN_BINDING);
    ColumnBindingResolver resolver;
    resolver.VisitOperator(*op);
    profiler.EndPhase();

    // Step 3: 创建物理计划
    profiler.StartPhase(MetricType::PHYSICAL_PLANNER_CREATE_PLAN);
    physical_plan = PlanInternal(*op);
    profiler.EndPhase();

    return physical_plan->Root();
}
```

### 9.2.3 算子映射分发

```cpp
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOperator &op) {
    switch (op.type) {
    // 扫描算子
    case LogicalOperatorType::LOGICAL_GET:
        return CreatePlan(op.Cast<LogicalGet>());
    case LogicalOperatorType::LOGICAL_DUMMY_SCAN:
        return CreatePlan(op.Cast<LogicalDummyScan>());

    // 投影和过滤
    case LogicalOperatorType::LOGICAL_PROJECTION:
        return CreatePlan(op.Cast<LogicalProjection>());
    case LogicalOperatorType::LOGICAL_FILTER:
        return CreatePlan(op.Cast<LogicalFilter>());

    // Join 算子
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
    case LogicalOperatorType::LOGICAL_ASOF_JOIN:
    case LogicalOperatorType::LOGICAL_DELIM_JOIN:
        return CreatePlan(op.Cast<LogicalComparisonJoin>());
    case LogicalOperatorType::LOGICAL_ANY_JOIN:
        return CreatePlan(op.Cast<LogicalAnyJoin>());
    case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
        return CreatePlan(op.Cast<LogicalCrossProduct>());

    // 聚合和排序
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        return CreatePlan(op.Cast<LogicalAggregate>());
    case LogicalOperatorType::LOGICAL_ORDER_BY:
        return CreatePlan(op.Cast<LogicalOrder>());
    case LogicalOperatorType::LOGICAL_TOP_N:
        return CreatePlan(op.Cast<LogicalTopN>());

    // ... 其他算子类型
    default:
        throw InternalException("Unimplemented logical operator type!");
    }
}
```

---

## 9.3 列绑定解析

### 9.3.1 列绑定与列引用

在逻辑计划中，列通过 `ColumnBinding(table_index, column_index)` 标识。物理计划需要将其转换为基于位置的 `BoundReferenceExpression(index)`。

```
┌─────────────────────────────────────────────────────────────────┐
│ 列绑定解析示例                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  逻辑计划中:                                                    │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ SELECT a, b FROM t1, t2 WHERE t1.id = t2.id               ││
│  │                                                            ││
│  │ Projection [t1.a, t1.b]                                   ││
│  │ • t1.a → ColumnBinding(table=0, column=0)                 ││
│  │ • t1.b → ColumnBinding(table=0, column=1)                 ││
│  │                                                            ││
│  │ Join [t1.id = t2.id]                                      ││
│  │ • t1.id → ColumnBinding(table=0, column=2)                ││
│  │ • t2.id → ColumnBinding(table=1, column=0)                ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  列绑定解析后:                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ Join 输出 bindings: [(0,0), (0,1), (0,2), (1,0), (1,1)]   ││
│  │                     索引:   0      1      2      3      4  ││
│  │                                                            ││
│  │ Projection 表达式:                                         ││
│  │ • t1.a → BoundReference(index=0)                          ││
│  │ • t1.b → BoundReference(index=1)                          ││
│  │                                                            ││
│  │ Join 条件:                                                 ││
│  │ • t1.id → BoundReference(index=2)                         ││
│  │ • t2.id → BoundReference(index=3)                         ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3.2 ColumnBindingResolver 实现

```cpp
// src/execution/column_binding_resolver.cpp

class ColumnBindingResolver : public LogicalOperatorVisitor {
public:
    void VisitOperator(LogicalOperator &op) override;

    // 将 BoundColumnRefExpression 替换为 BoundReferenceExpression
    unique_ptr<Expression> VisitReplace(
        BoundColumnRefExpression &expr,
        unique_ptr<Expression> *expr_ptr) override;

private:
    // 当前可见的列绑定列表
    vector<ColumnBinding> bindings;
    // 对应的类型（用于验证）
    vector<LogicalType> types;
};

void ColumnBindingResolver::VisitOperator(LogicalOperator &op) {
    // 对于大多数算子：
    // 1. 先访问子节点
    // 2. 然后访问自身表达式
    // 3. 更新 bindings 为当前算子的输出

    // 特殊处理 Join（需要分别解析左右两侧）
    if (op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
        auto &comp_join = op.Cast<LogicalComparisonJoin>();

        // 先处理左侧
        VisitOperator(*comp_join.children[0]);
        for (auto &cond : comp_join.conditions) {
            VisitExpression(&cond.left);  // 解析左侧条件
        }

        // 再处理右侧
        VisitOperator(*comp_join.children[1]);
        for (auto &cond : comp_join.conditions) {
            VisitExpression(&cond.right);  // 解析右侧条件
        }

        // 更新 bindings 为 Join 输出
        bindings = op.GetColumnBindings();
        types = op.types;
        return;
    }

    // 通用处理
    VisitOperatorChildren(op);
    VisitOperatorExpressions(op);
    bindings = op.GetColumnBindings();
    types = op.types;
}

unique_ptr<Expression> ColumnBindingResolver::VisitReplace(
    BoundColumnRefExpression &expr,
    unique_ptr<Expression> *expr_ptr) {

    // 在 bindings 中查找匹配的列
    for (idx_t i = 0; i < bindings.size(); i++) {
        if (expr.binding == bindings[i]) {
            // 找到匹配，创建位置引用
            return make_uniq<BoundReferenceExpression>(
                expr.GetAlias(),
                expr.return_type,
                i);  // 使用位置索引
        }
    }

    // 未找到，报错
    throw InternalException("Failed to bind column reference");
}
```

---

## 9.4 Join 物理算子选择

### 9.4.1 Join 算法概述

DuckDB 支持多种 Join 算法，根据条件类型和数据量选择最优算法：

```
┌─────────────────────────────────────────────────────────────────┐
│ Join 算法选择决策树                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       ┌──────────────┐                         │
│                       │ Join 条件    │                         │
│                       └──────┬───────┘                         │
│                              │                                  │
│              ┌───────────────┼───────────────┐                 │
│              │               │               │                  │
│              ▼               ▼               ▼                  │
│       ┌──────────┐   ┌──────────┐   ┌──────────────┐          │
│       │ 无条件   │   │ 有等值   │   │ 只有范围条件 │          │
│       └────┬─────┘   └────┬─────┘   └──────┬───────┘          │
│            │              │                │                    │
│            ▼              ▼                ▼                    │
│     ┌──────────┐   ┌──────────────┐  ┌────────────────┐       │
│     │ Cross    │   │ 检查数据量  │  │ 检查范围条件数 │       │
│     │ Product  │   └──────┬───────┘  └───────┬────────┘       │
│     └──────────┘          │                  │                 │
│                           │                  │                 │
│              ┌────────────┼────────────┐     │                 │
│              │            │            │     │                 │
│              ▼            ▼            ▼     ▼                 │
│        ┌─────────┐  ┌──────────┐  ┌─────────────┐             │
│        │ 小数据量│  │ 大数据量 │  │ 2+范围条件  │             │
│        │ (< 5)   │  │ (≥ 5)    │  └──────┬──────┘             │
│        └────┬────┘  └────┬─────┘         │                    │
│             │            │               │                    │
│             ▼            ▼               ▼                    │
│       ┌──────────┐ ┌──────────┐    ┌──────────┐              │
│       │ Nested   │ │  Hash    │    │  IEJoin  │              │
│       │ Loop     │ │  Join    │    └──────────┘              │
│       │ Join     │ └──────────┘                               │
│       └──────────┘                                            │
│                                                                 │
│       ┌──────────────────────────────────────────────────────┐│
│       │ 只有1个范围条件                                       ││
│       │                                                       ││
│       │       ┌────────────────┐                             ││
│       │       │ 数据量检查     │                             ││
│       │       └───────┬────────┘                             ││
│       │               │                                       ││
│       │   ┌───────────┴───────────┐                          ││
│       │   │                       │                           ││
│       │   ▼                       ▼                           ││
│       │ ┌──────────┐        ┌───────────────┐                ││
│       │ │ 小数据量 │        │ 大数据量      │                ││
│       │ │          │        │               │                ││
│       │ │ Nested   │        │ Piecewise     │                ││
│       │ │ Loop     │        │ Merge Join    │                ││
│       │ └──────────┘        └───────────────┘                ││
│       └──────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.4.2 Join 物理计划生成

```cpp
// src/execution/physical_plan/plan_comparison_join.cpp

PhysicalOperator &PhysicalPlanGenerator::PlanComparisonJoin(LogicalComparisonJoin &op) {
    // 递归创建子计划
    auto &left = CreatePlan(*op.children[0]);
    auto &right = CreatePlan(*op.children[1]);

    // 无条件 Join → Cross Product
    if (op.conditions.empty()) {
        return Make<PhysicalCrossProduct>(op.types, left, right, op.estimated_cardinality);
    }

    // 分析 Join 条件
    idx_t has_range = 0;
    bool has_equality = op.HasEquality(has_range);
    bool can_merge = has_range > 0;
    bool can_iejoin = has_range >= 2 && recursive_cte_tables.empty();

    // 某些 Join 类型限制算法选择
    switch (op.join_type) {
    case JoinType::SEMI:
    case JoinType::ANTI:
    case JoinType::MARK:
        can_merge = can_merge && op.conditions.size() == 1;
        can_iejoin = false;
        break;
    default:
        break;
    }

    // 优先选择 Hash Join（有等值条件时）
    bool prefer_range_joins = DBConfig::GetSetting<PreferRangeJoinsSetting>(context);
    if (has_equality && !prefer_range_joins) {
        auto &join = Make<PhysicalHashJoin>(
            op, left, right,
            std::move(op.conditions),
            op.join_type,
            op.left_projection_map,
            op.right_projection_map,
            std::move(op.mark_types),
            op.estimated_cardinality,
            std::move(op.filter_pushdown));
        join.Cast<PhysicalHashJoin>().join_stats = std::move(op.join_stats);
        return join;
    }

    // 小数据量检查
    idx_t nested_loop_threshold = DBConfig::GetSetting<NestedLoopJoinThresholdSetting>(context);
    if (left.estimated_cardinality < nested_loop_threshold ||
        right.estimated_cardinality < nested_loop_threshold) {
        can_iejoin = false;
        can_merge = false;
    }

    // IEJoin：两个或更多范围条件
    if (can_iejoin) {
        return Make<PhysicalIEJoin>(
            op, left, right,
            std::move(op.conditions),
            op.join_type,
            op.estimated_cardinality,
            std::move(op.filter_pushdown));
    }

    // Piecewise Merge Join：单个范围条件
    if (can_merge) {
        return Make<PhysicalPiecewiseMergeJoin>(
            op, left, right,
            std::move(op.conditions),
            op.join_type,
            op.estimated_cardinality,
            std::move(op.filter_pushdown));
    }

    // Nested Loop Join
    if (PhysicalNestedLoopJoin::IsSupported(op.conditions, op.join_type)) {
        return Make<PhysicalNestedLoopJoin>(
            op, left, right,
            std::move(op.conditions),
            op.join_type,
            op.estimated_cardinality,
            std::move(op.filter_pushdown));
    }

    // Blockwise Nested Loop Join（最通用的后备方案）
    auto condition = JoinCondition::CreateExpression(std::move(op.conditions));
    return Make<PhysicalBlockwiseNLJoin>(
        op, left, right,
        std::move(condition),
        op.join_type,
        op.estimated_cardinality);
}
```

### 9.4.3 各种 Join 算法特点

| 算法 | 适用场景 | 时间复杂度 | 特点 |
|------|---------|-----------|------|
| Hash Join | 等值条件 | O(n + m) | 最常用，支持并行 |
| Nested Loop | 小数据量 | O(n × m) | 简单通用 |
| Piecewise Merge | 范围条件 | O(n log n + m log m) | 需要排序 |
| IEJoin | 2+范围条件 | O(n log n) | 区间优化 |
| Blockwise NL | 复杂条件 | O(n × m) | 最后备选 |

---

## 9.5 Aggregate 物理算子选择

### 9.5.1 聚合算法概述

```
┌─────────────────────────────────────────────────────────────────┐
│ Aggregate 算法选择                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌──────────────────┐                        │
│                    │   有分组列吗？   │                        │
│                    └────────┬─────────┘                        │
│                             │                                   │
│             ┌───────────────┴───────────────┐                  │
│             │                               │                   │
│             ▼                               ▼                   │
│     ┌──────────────┐               ┌──────────────┐            │
│     │     无       │               │     有       │            │
│     └──────┬───────┘               └──────┬───────┘            │
│            │                              │                     │
│            ▼                              │                     │
│  ┌─────────────────────┐                  │                     │
│  │ PhysicalUngrouped   │                  │                     │
│  │ Aggregate           │                  │                     │
│  │ (全表聚合)          │                  │                     │
│  └─────────────────────┘                  │                     │
│                                           │                     │
│                    ┌──────────────────────┘                    │
│                    │                                            │
│                    ▼                                            │
│          ┌─────────────────┐                                   │
│          │ 检查数据源分区  │                                   │
│          └────────┬────────┘                                   │
│                   │                                            │
│       ┌───────────┴───────────┐                               │
│       │                       │                                │
│       ▼                       ▼                                │
│  ┌──────────┐           ┌──────────┐                          │
│  │ 分区对齐 │           │ 不对齐   │                          │
│  └────┬─────┘           └────┬─────┘                          │
│       │                      │                                 │
│       ▼                      │                                 │
│  ┌────────────────┐          │                                 │
│  │ Partitioned    │          │                                 │
│  │ Aggregate      │          │                                 │
│  └────────────────┘          │                                 │
│                              │                                 │
│            ┌─────────────────┘                                 │
│            │                                                   │
│            ▼                                                   │
│   ┌─────────────────────┐                                     │
│   │ 检查 Perfect Hash   │                                     │
│   │ 条件                │                                     │
│   └──────────┬──────────┘                                     │
│              │                                                 │
│     ┌────────┴────────┐                                       │
│     │                 │                                        │
│     ▼                 ▼                                        │
│ ┌─────────┐     ┌─────────┐                                   │
│ │ 满足    │     │ 不满足  │                                   │
│ └────┬────┘     └────┬────┘                                   │
│      │               │                                         │
│      ▼               ▼                                         │
│ ┌───────────────┐ ┌───────────────┐                           │
│ │ PerfectHash   │ │ Hash          │                           │
│ │ Aggregate     │ │ Aggregate     │                           │
│ └───────────────┘ └───────────────┘                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.5.2 聚合物理计划生成

```cpp
// src/execution/physical_plan/plan_aggregate.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalAggregate &op) {
    // 创建子计划
    reference<PhysicalOperator> plan = CreatePlan(*op.children[0]);

    // 提取聚合表达式
    plan = ExtractAggregateExpressions(plan, op.expressions, op.groups, op.grouping_sets);

    // 检查是否可以使用简单聚合
    bool can_use_simple = true;
    for (auto &expression : op.expressions) {
        auto &aggregate = expression->Cast<BoundAggregateExpression>();
        if (!aggregate.function.HasStateSimpleUpdateCallback()) {
            can_use_simple = false;
            break;
        }
    }

    // 无分组：使用 UngroupedAggregate
    if (op.groups.empty() && op.grouping_sets.size() <= 1) {
        if (can_use_simple) {
            auto &group_by = Make<PhysicalUngroupedAggregate>(
                op.types,
                std::move(op.expressions),
                op.estimated_cardinality,
                op.distinct_validity);
            group_by.children.push_back(plan);
            return group_by;
        }
        auto &group_by = Make<PhysicalHashAggregate>(
            context, op.types,
            std::move(op.expressions),
            op.estimated_cardinality);
        group_by.children.push_back(plan);
        return group_by;
    }

    // 有分组：尝试各种优化
    vector<column_t> partition_columns;
    vector<idx_t> required_bits;

    // 尝试 Partitioned Aggregate
    if (can_use_simple && CanUsePartitionedAggregate(context, op, plan, partition_columns)) {
        auto &group_by = Make<PhysicalPartitionedAggregate>(
            context, op.types,
            std::move(op.expressions),
            std::move(op.groups),
            std::move(partition_columns),
            op.estimated_cardinality);
        group_by.children.push_back(plan);
        return group_by;
    }

    // 尝试 Perfect Hash Aggregate
    if (CanUsePerfectHashAggregate(context, op, required_bits)) {
        auto &group_by = Make<PhysicalPerfectHashAggregate>(
            context, op.types,
            std::move(op.expressions),
            std::move(op.groups),
            std::move(op.group_stats),
            std::move(required_bits),
            op.estimated_cardinality);
        group_by.children.push_back(plan);
        return group_by;
    }

    // 默认：Hash Aggregate
    auto &group_by = Make<PhysicalHashAggregate>(
        context, op.types,
        std::move(op.expressions),
        std::move(op.groups),
        std::move(op.grouping_sets),
        std::move(op.grouping_functions),
        op.estimated_cardinality,
        group_validity,
        op.distinct_validity);
    group_by.children.push_back(plan);
    return group_by;
}
```

### 9.5.3 Perfect Hash Aggregate 条件

Perfect Hash Aggregate 使用数组代替哈希表，条件更严格但性能更好：

```cpp
static bool CanUsePerfectHashAggregate(
    ClientContext &context,
    LogicalAggregate &op,
    vector<idx_t> &bits_per_group) {

    // 不支持多个 grouping sets
    if (op.grouping_sets.size() > 1 || !op.grouping_functions.empty()) {
        return false;
    }

    idx_t perfect_hash_bits = 0;
    for (idx_t group_idx = 0; group_idx < op.groups.size(); group_idx++) {
        auto &group = op.groups[group_idx];
        auto &stats = op.group_stats[group_idx];

        // 只支持整型
        switch (group->return_type.InternalType()) {
        case PhysicalType::INT8:
        case PhysicalType::INT16:
        case PhysicalType::INT32:
        case PhysicalType::INT64:
        case PhysicalType::UINT8:
        case PhysicalType::UINT16:
        case PhysicalType::UINT32:
        case PhysicalType::UINT64:
            break;
        default:
            return false;
        }

        // 需要有 min/max 统计信息
        if (!stats || !NumericStats::HasMinMax(*stats)) {
            // 小类型可以使用类型边界
            switch (group->return_type.InternalType()) {
            case PhysicalType::INT8:
            case PhysicalType::INT16:
            case PhysicalType::UINT8:
            case PhysicalType::UINT16:
                break;
            default:
                return false;  // 大类型没有统计信息无法使用
            }
        }

        // 计算值域范围
        auto range = max_value - min_value + 2;  // +2 for NULL and one-indexed

        // 范围不能超过 2^32
        if (range >= NumericLimits<int32_t>::Maximum()) {
            return false;
        }

        // 计算需要的位数
        idx_t required_bits = RequiredBitsForValue(range);
        bits_per_group.push_back(required_bits);
        perfect_hash_bits += required_bits;

        // 总位数不能超过阈值（默认 12 位 = 4096 个桶）
        if (perfect_hash_bits > DBConfig::GetSetting<PerfectHtThresholdSetting>(context)) {
            return false;
        }
    }

    // 检查聚合函数是否支持
    for (auto &expression : op.expressions) {
        auto &aggregate = expression->Cast<BoundAggregateExpression>();
        if (aggregate.IsDistinct() || !aggregate.function.HasStateCombineCallback()) {
            return false;
        }
    }

    return true;
}
```

### 9.5.4 聚合算法对比

| 算法 | 数据结构 | 适用场景 | 优势 |
|------|---------|---------|------|
| Ungrouped | 无 | 无分组 | 最简单，最快 |
| Perfect Hash | 数组 | 整型分组键，小范围 | 无哈希冲突 |
| Partitioned | 分区表 | 数据源已分区 | 利用分区 |
| Hash | 哈希表 | 通用 | 最灵活 |

---

## 9.6 Distinct 物理计划

### 9.6.1 Distinct 实现原理

DISTINCT 在 DuckDB 中通过 Hash Aggregate 实现，对于 `DISTINCT ON` 语法需要额外处理：

```
┌─────────────────────────────────────────────────────────────────┐
│ DISTINCT 实现                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SELECT DISTINCT a, b FROM t;                                  │
│                                                                 │
│  实现方式:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ HashAggregate                                              ││
│  │   groups: [a, b]                                           ││
│  │   aggregates: []                                           ││
│  │      ↓                                                     ││
│  │ TableScan(t)                                               ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  SELECT DISTINCT ON (a) a, b, c FROM t ORDER BY a, d;          │
│                                                                 │
│  实现方式:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ Projection [a, b, c]                                       ││
│  │      ↓                                                     ││
│  │ HashAggregate                                              ││
│  │   groups: [a]           -- DISTINCT ON 的列               ││
│  │   aggregates: [                                            ││
│  │     FIRST(b ORDER BY a, d),  -- 非分组列用 FIRST 聚合     ││
│  │     FIRST(c ORDER BY a, d)                                 ││
│  │   ]                                                        ││
│  │      ↓                                                     ││
│  │ TableScan(t)                                               ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.6.2 Distinct 物理计划生成

```cpp
// src/execution/physical_plan/plan_distinct.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalDistinct &op) {
    reference<PhysicalOperator> child = CreatePlan(*op.children[0]);

    auto &types = child.get().GetTypes();
    vector<unique_ptr<Expression>> groups, aggregates, projections;
    idx_t group_count = op.distinct_targets.size();
    unordered_map<idx_t, idx_t> group_by_references;
    vector<LogicalType> aggregate_types;

    // 为每个 distinct_target 创建分组
    for (idx_t i = 0; i < op.distinct_targets.size(); i++) {
        auto &target = op.distinct_targets[i];
        if (target->GetExpressionType() == ExpressionType::BOUND_REF) {
            auto &bound_ref = target->Cast<BoundReferenceExpression>();
            group_by_references[bound_ref.index] = i;
        }
        aggregate_types.push_back(target->return_type);
        groups.push_back(std::move(target));
    }

    bool requires_projection = false;
    if (types.size() != group_count) {
        requires_projection = true;
    }

    // 处理非分组列
    for (idx_t i = 0; i < types.size(); ++i) {
        auto logical_type = types[i];

        auto entry = group_by_references.find(i);
        if (entry != group_by_references.end()) {
            // 是分组列，直接引用
            auto group_index = entry->second;
            projections.push_back(make_uniq<BoundReferenceExpression>(
                logical_type, group_index));
            if (group_index != i) {
                requires_projection = true;
            }
        } else {
            // 非分组列，使用 FIRST 聚合
            auto bound = make_uniq<BoundReferenceExpression>(logical_type, i);
            vector<unique_ptr<Expression>> first_children;
            first_children.push_back(std::move(bound));

            FunctionBinder function_binder(context);
            auto first_aggregate = function_binder.BindAggregateFunction(
                FirstFunctionGetter::GetFunction(logical_type),
                std::move(first_children),
                nullptr,
                AggregateType::NON_DISTINCT);

            // 如果有 ORDER BY，传递给 FIRST
            first_aggregate->order_bys = op.order_by ? op.order_by->Copy() : nullptr;

            projections.push_back(make_uniq<BoundReferenceExpression>(
                logical_type, group_count + aggregates.size()));
            aggregate_types.push_back(logical_type);
            aggregates.push_back(std::move(first_aggregate));
            requires_projection = true;
        }
    }

    // 提取聚合表达式
    child = ExtractAggregateExpressions(child, aggregates, groups, nullptr);

    // 创建 Hash Aggregate
    auto &group_by = Make<PhysicalHashAggregate>(
        context, aggregate_types,
        std::move(aggregates),
        std::move(groups),
        child.get().estimated_cardinality);
    group_by.children.push_back(child);

    if (!requires_projection) {
        return group_by;
    }

    // 需要投影重新排列列顺序
    auto &proj = Make<PhysicalProjection>(
        types,
        std::move(projections),
        group_by.estimated_cardinality);
    proj.children.push_back(group_by);
    return proj;
}
```

---

## 9.7 其他算子的物理计划

### 9.7.1 TableScan 物理计划

```cpp
// src/execution/physical_plan/plan_get.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalGet &op) {
    auto column_ids = op.GetColumnIds();

    // 处理表函数（有子查询输入）
    if (!op.children.empty()) {
        reference<PhysicalOperator> child = ResolveAndPlan(std::move(op.children[0]));
        // 可能需要添加类型转换
        // ...
        auto &table_in_out = Make<PhysicalTableInOutFunction>(
            op.types, op.function, std::move(op.bind_data),
            column_ids, op.estimated_cardinality,
            std::move(op.projected_input));
        table_in_out.children.push_back(child);
        return table_in_out;
    }

    // 创建表过滤器
    unique_ptr<TableFilterSet> table_filters;
    if (!op.table_filters.filters.empty()) {
        table_filters = CreateTableFilterSet(op.table_filters, column_ids);
    }

    // 创建 TableScan
    if (!op.function.projection_pushdown) {
        // 不支持投影下推：返回所有列，然后添加投影
        auto &table_scan = Make<PhysicalTableScan>(
            op.returned_types, op.function, std::move(op.bind_data),
            op.returned_types, column_ids, vector<column_t>(),
            op.names, std::move(table_filters), op.estimated_cardinality,
            std::move(op.extra_info), std::move(op.parameters),
            std::move(op.virtual_columns));

        // 检查是否需要投影
        if (NeedsProjection(column_ids, op.returned_types.size())) {
            // 添加投影算子
            auto &proj = Make<PhysicalProjection>(/* ... */);
            proj.children.push_back(table_scan);
            return proj;
        }
        return table_scan;
    }

    // 支持投影下推
    auto &table_scan = Make<PhysicalTableScan>(
        op.types, op.function, std::move(op.bind_data),
        op.returned_types, column_ids, op.projection_ids,
        op.names, std::move(table_filters), op.estimated_cardinality,
        std::move(op.extra_info), std::move(op.parameters),
        std::move(op.virtual_columns));
    table_scan.Cast<PhysicalTableScan>().dynamic_filters = op.dynamic_filters;
    return table_scan;
}
```

### 9.7.2 Order 物理计划

```cpp
// src/execution/physical_plan/plan_order.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOrder &op) {
    auto &plan = CreatePlan(*op.children[0]);

    // 无排序条件
    if (op.orders.empty()) {
        return plan;
    }

    // 创建投影映射
    vector<idx_t> projection_map;
    if (op.HasProjectionMap()) {
        projection_map = std::move(op.projection_map);
    } else {
        for (idx_t i = 0; i < plan.types.size(); i++) {
            projection_map.push_back(i);
        }
    }

    // 创建排序算子
    auto &order = Make<PhysicalOrder>(
        op.types,
        std::move(op.orders),
        std::move(projection_map),
        op.estimated_cardinality);
    order.children.push_back(plan);
    return order;
}
```

---

## 9.8 物理计划验证

### 9.8.1 计划验证流程

物理计划生成后会进行验证，确保计划的正确性：

```cpp
// 验证物理计划
void PhysicalPlan::Verify() {
#ifdef DEBUG
    // 验证所有算子
    for (auto &op : ops) {
        VerifyOperator(op.get());
    }
#endif
}

void VerifyOperator(PhysicalOperator &op) {
    // 验证类型一致性
    for (idx_t i = 0; i < op.children.size(); i++) {
        auto &child = op.children[i];
        // 检查子算子输出类型与期望一致
    }

    // 验证表达式
    for (auto &expr : op.expressions) {
        VerifyExpression(*expr);
    }

    // 递归验证子算子
    for (auto &child : op.children) {
        VerifyOperator(child.get());
    }
}
```

---

## 9.9 逻辑到物理算子映射总结

```
┌─────────────────────────────────────────────────────────────────┐
│ 逻辑算子到物理算子映射                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LogicalGet                                                     │
│  ├─ PhysicalTableScan (表扫描)                                 │
│  ├─ PhysicalTableInOutFunction (表函数)                        │
│  └─ PhysicalProjection (投影，如需要)                          │
│                                                                 │
│  LogicalProjection → PhysicalProjection                        │
│  LogicalFilter → PhysicalFilter                                │
│                                                                 │
│  LogicalComparisonJoin                                         │
│  ├─ PhysicalHashJoin (等值条件)                                │
│  ├─ PhysicalPiecewiseMergeJoin (单范围条件)                    │
│  ├─ PhysicalIEJoin (多范围条件)                                │
│  ├─ PhysicalNestedLoopJoin (小数据量)                          │
│  └─ PhysicalBlockwiseNLJoin (复杂条件)                         │
│                                                                 │
│  LogicalAggregate                                              │
│  ├─ PhysicalUngroupedAggregate (无分组)                        │
│  ├─ PhysicalPerfectHashAggregate (整型小范围)                  │
│  ├─ PhysicalPartitionedAggregate (分区对齐)                    │
│  └─ PhysicalHashAggregate (通用)                               │
│                                                                 │
│  LogicalDistinct → PhysicalHashAggregate + PhysicalProjection  │
│  LogicalOrder → PhysicalOrder                                  │
│  LogicalTopN → PhysicalTopN                                    │
│  LogicalLimit → PhysicalLimit                                  │
│  LogicalCrossProduct → PhysicalCrossProduct                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.10 本章小结

本章介绍了 DuckDB 物理计划生成的核心内容：

1. **PhysicalPlanGenerator 架构**：使用 Arena 分配器管理物理算子内存，通过 CreatePlan 分发到具体算子处理

2. **列绑定解析**：将逻辑计划中的 `ColumnBinding` 转换为物理计划中基于位置的 `BoundReference`

3. **Join 算法选择**：根据条件类型和数据量选择 Hash Join、Merge Join、Nested Loop Join 或 IEJoin

4. **Aggregate 算法选择**：根据分组情况选择 Ungrouped、Perfect Hash、Partitioned 或 Hash Aggregate

5. **Distinct 实现**：通过 Hash Aggregate 实现，非分组列使用 FIRST 聚合

物理计划生成是查询优化的最后一环，它将优化器的智能决策转化为具体的执行算法，直接影响查询性能。

---

## 附录：核心源文件索引

| 组件 | 文件路径 |
|------|----------|
| PhysicalPlanGenerator | `src/execution/physical_plan_generator.cpp` |
| ColumnBindingResolver | `src/execution/column_binding_resolver.cpp` |
| Join 计划 | `src/execution/physical_plan/plan_comparison_join.cpp` |
| Aggregate 计划 | `src/execution/physical_plan/plan_aggregate.cpp` |
| Distinct 计划 | `src/execution/physical_plan/plan_distinct.cpp` |
| Get 计划 | `src/execution/physical_plan/plan_get.cpp` |
| Order 计划 | `src/execution/physical_plan/plan_order.cpp` |

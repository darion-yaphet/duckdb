# DuckDB 查询处理深度解析：第六章 - Physical Planner（物理计划生成）

## 概述

Physical Planner 是 DuckDB 查询处理流程的最后一个阶段，负责将优化后的逻辑计划（LogicalOperator 树）转换为可执行的物理计划（PhysicalOperator 树）。这个转换过程涉及：

1. **算法选择**：为每个逻辑算子选择最合适的物理实现
2. **资源配置**：根据数据特征配置算子参数
3. **Pipeline 构建**：将物理算子组织成可并行执行的 Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    Physical Planner 流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   LogicalOperator Tree (优化后)                              │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────────────────────────────┐               │
│   │     PhysicalPlanGenerator               │               │
│   │   ┌───────────────────────────────────┐ │               │
│   │   │ CreatePlan(LogicalOperator)       │ │               │
│   │   │   ├── 算法选择                     │ │               │
│   │   │   ├── 资源配置                     │ │               │
│   │   │   └── 递归处理子节点               │ │               │
│   │   └───────────────────────────────────┘ │               │
│   └─────────────────────────────────────────┘               │
│          │                                                  │
│          ▼                                                  │
│   PhysicalOperator Tree                                     │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────────────────────────────┐               │
│   │     Pipeline Construction               │               │
│   │   ┌───────────────────────────────────┐ │               │
│   │   │ BuildPipelines()                  │ │               │
│   │   │   ├── 识别阻塞点 (Sink)           │ │               │
│   │   │   ├── 分割 Pipeline               │ │               │
│   │   │   └── 建立依赖关系                │ │               │
│   │   └───────────────────────────────────┘ │               │
│   └─────────────────────────────────────────┘               │
│          │                                                  │
│          ▼                                                  │
│   可执行 Pipeline 集合                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.1 PhysicalPlanGenerator 架构

### 6.1.1 核心类设计

`PhysicalPlanGenerator` 是物理计划生成的核心类，位于 `src/execution/physical_plan_generator.hpp`：

```cpp
// src/include/duckdb/execution/physical_plan_generator.hpp

class PhysicalPlan {
public:
    explicit PhysicalPlan(Allocator &allocator) : arena(allocator) {};

    // 创建物理算子（使用 Arena 分配内存）
    template <class T, class... ARGS>
    PhysicalOperator &Make(ARGS &&... args) {
        static_assert(std::is_base_of<PhysicalOperator, T>::value,
                      "T must be a physical operator");
        auto ptr = arena.Make<T>(*this, std::forward<ARGS>(args)...);
        ops.push_back(*ptr);
        return *ptr;
    }

    PhysicalOperator &Root() { return *root; }
    void SetRoot(PhysicalOperator &op) { root = op; }

private:
    ArenaAllocator arena;                         // Arena 分配器
    vector<reference<PhysicalOperator>> ops;      // 所有算子引用
    optional_ptr<PhysicalOperator> root;          // 根算子
};

class PhysicalPlanGenerator {
public:
    explicit PhysicalPlanGenerator(ClientContext &context);

    // 主入口：生成物理计划
    unique_ptr<PhysicalPlan> Plan(unique_ptr<LogicalOperator> logical);

    // 为单个算子创建物理计划
    PhysicalOperator &CreatePlan(LogicalOperator &op);

    // CTE 相关状态
    unordered_map<idx_t, shared_ptr<ColumnDataCollection>> recursive_cte_tables;
    unordered_map<idx_t, shared_ptr<ColumnDataCollection>> recurring_cte_tables;
    unordered_map<idx_t, vector<const_reference<PhysicalOperator>>> materialized_ctes;

protected:
    // 各类算子的转换方法
    PhysicalOperator &CreatePlan(LogicalAggregate &op);
    PhysicalOperator &CreatePlan(LogicalComparisonJoin &op);
    PhysicalOperator &CreatePlan(LogicalGet &op);
    PhysicalOperator &CreatePlan(LogicalFilter &op);
    PhysicalOperator &CreatePlan(LogicalProjection &op);
    PhysicalOperator &CreatePlan(LogicalOrder &op);
    PhysicalOperator &CreatePlan(LogicalLimit &op);
    PhysicalOperator &CreatePlan(LogicalWindow &op);
    // ... 40+ 其他算子

private:
    ClientContext &context;
    unique_ptr<PhysicalPlan> physical_plan;
};
```

### 6.1.2 计划生成流程

```cpp
// 主入口
unique_ptr<PhysicalPlan> PhysicalPlanGenerator::Plan(unique_ptr<LogicalOperator> logical) {
    // 1. 创建物理计划容器
    physical_plan = make_uniq<PhysicalPlan>(Allocator::DefaultAllocator());

    // 2. 递归转换逻辑计划
    auto &root = CreatePlan(*logical);
    physical_plan->SetRoot(root);

    // 3. 验证物理计划
    root.Verify();

    return std::move(physical_plan);
}

// 通用 CreatePlan 入口（根据算子类型分发）
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOperator &op) {
    switch (op.type) {
    case LogicalOperatorType::LOGICAL_GET:
        return CreatePlan(op.Cast<LogicalGet>());
    case LogicalOperatorType::LOGICAL_FILTER:
        return CreatePlan(op.Cast<LogicalFilter>());
    case LogicalOperatorType::LOGICAL_PROJECTION:
        return CreatePlan(op.Cast<LogicalProjection>());
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        return CreatePlan(op.Cast<LogicalAggregate>());
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
        return CreatePlan(op.Cast<LogicalComparisonJoin>());
    case LogicalOperatorType::LOGICAL_ORDER_BY:
        return CreatePlan(op.Cast<LogicalOrder>());
    // ... 其他类型
    }
}
```

---

## 6.2 PhysicalOperator 基类设计

### 6.2.1 核心类层次

`PhysicalOperator` 是所有物理算子的基类，位于 `src/include/duckdb/execution/physical_operator.hpp`：

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class PhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::INVALID;

    PhysicalOperator(PhysicalPlan &physical_plan, PhysicalOperatorType type,
                     vector<LogicalType> types, idx_t estimated_cardinality);

    // 子算子列表（使用 Arena 链表）
    ArenaLinkedList<reference<PhysicalOperator>> children;

    // 算子类型
    PhysicalOperatorType type;

    // 输出类型
    vector<LogicalType> types;

    // 估计基数
    idx_t estimated_cardinality;

    // 全局状态
    unique_ptr<GlobalSinkState> sink_state;
    unique_ptr<GlobalOperatorState> op_state;

    //=== Operator 接口（中间算子）===
    virtual unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const;
    virtual OperatorResultType Execute(ExecutionContext &context,
                                       DataChunk &input, DataChunk &chunk,
                                       GlobalOperatorState &gstate,
                                       OperatorState &state) const;

    //=== Source 接口（数据源）===
    virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const;
    virtual unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                             GlobalSourceState &gstate) const;
    virtual SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                                     OperatorSourceInput &input) const;
    virtual bool IsSource() const { return false; }
    virtual bool ParallelSource() const { return false; }

    //=== Sink 接口（数据汇）===
    virtual SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                                OperatorSinkInput &input) const;
    virtual SinkCombineResultType Combine(ExecutionContext &context,
                                          OperatorSinkCombineInput &input) const;
    virtual SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                                      ClientContext &context,
                                      OperatorSinkFinalizeInput &input) const;
    virtual bool IsSink() const { return false; }
    virtual bool ParallelSink() const { return false; }

    //=== Pipeline 构建 ===
    virtual void BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline);
    virtual vector<const_reference<PhysicalOperator>> GetSources() const;
};
```

### 6.2.2 三种算子角色

物理算子在执行时扮演三种不同角色：

```
┌─────────────────────────────────────────────────────────────┐
│                   PhysicalOperator 角色                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Source（数据源）                                    │    │
│  │  • 产生数据的起点                                    │    │
│  │  • 实现 GetData() 方法                               │    │
│  │  • 例：TableScan, ChunkScan                         │    │
│  │  • IsSource() 返回 true                             │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Operator（中间算子）                                │    │
│  │  • 处理数据流的中间节点                              │    │
│  │  • 实现 Execute() 方法                               │    │
│  │  • 例：Filter, Projection                           │    │
│  │  • 非 Source 也非 Sink                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Sink（数据汇 / 阻塞算子）                           │    │
│  │  • 消费数据的终点，需要收集所有数据                   │    │
│  │  • 实现 Sink(), Combine(), Finalize() 方法           │    │
│  │  • 例：HashAggregate, HashJoin (build 侧)           │    │
│  │  • IsSink() 返回 true                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2.3 物理算子类型枚举

```cpp
// src/include/duckdb/common/enums/physical_operator_type.hpp

enum class PhysicalOperatorType : uint8_t {
    INVALID,

    // 基本算子
    ORDER_BY,
    LIMIT,
    STREAMING_LIMIT,
    LIMIT_PERCENT,
    TOP_N,
    WINDOW,
    UNNEST,
    FILTER,
    PROJECTION,
    PIVOT,

    // 聚合算子
    UNGROUPED_AGGREGATE,
    HASH_GROUP_BY,
    PERFECT_HASH_GROUP_BY,
    PARTITIONED_AGGREGATE,

    // 扫描算子
    TABLE_SCAN,
    DUMMY_SCAN,
    COLUMN_DATA_SCAN,
    CHUNK_SCAN,
    RECURSIVE_CTE_SCAN,
    CTE_SCAN,
    DELIM_SCAN,
    EXPRESSION_SCAN,
    POSITIONAL_SCAN,

    // Join 算子
    BLOCKWISE_NL_JOIN,
    NESTED_LOOP_JOIN,
    HASH_JOIN,
    CROSS_PRODUCT,
    PIECEWISE_MERGE_JOIN,
    IE_JOIN,
    LEFT_DELIM_JOIN,
    RIGHT_DELIM_JOIN,
    POSITIONAL_JOIN,
    ASOF_JOIN,

    // 集合算子
    UNION,
    RECURSIVE_CTE,
    CTE,

    // DML 算子
    INSERT,
    BATCH_INSERT,
    DELETE_OPERATOR,
    UPDATE,
    MERGE_INTO,

    // 其他
    CREATE_TABLE,
    CREATE_INDEX,
    EXPLAIN,
    EXPLAIN_ANALYZE,
    // ...
};
```

---

## 6.3 Join 算法选择

Join 是查询处理中最复杂的操作，DuckDB 提供了多种物理实现。`plan_comparison_join.cpp` 实现了 Join 算法的选择逻辑：

### 6.3.1 Join 算法选择流程

```cpp
// src/execution/physical_plan/plan_comparison_join.cpp

PhysicalOperator &PhysicalPlanGenerator::PlanComparisonJoin(LogicalComparisonJoin &op) {
    // 1. 递归处理子节点
    D_ASSERT(op.children.size() == 2);
    idx_t lhs_cardinality = op.children[0]->EstimateCardinality(context);
    idx_t rhs_cardinality = op.children[1]->EstimateCardinality(context);
    auto &left = CreatePlan(*op.children[0]);
    auto &right = CreatePlan(*op.children[1]);

    // 2. 无条件 Join → Cross Product
    if (op.conditions.empty()) {
        return Make<PhysicalCrossProduct>(op.types, left, right, op.estimated_cardinality);
    }

    // 3. 分析 Join 条件
    idx_t has_range = 0;
    bool has_equality = op.HasEquality(has_range);  // 检查是否有等值条件
    bool can_merge = has_range > 0;                  // 是否有范围条件
    bool can_iejoin = has_range >= 2 && recursive_cte_tables.empty();  // IE Join 需要 2+ 范围条件

    // 4. 根据 Join 类型调整策略
    switch (op.join_type) {
    case JoinType::SEMI:
    case JoinType::ANTI:
    case JoinType::RIGHT_ANTI:
    case JoinType::RIGHT_SEMI:
    case JoinType::MARK:
        can_merge = can_merge && op.conditions.size() == 1;
        can_iejoin = false;  // 这些类型不支持 IE Join
        break;
    default:
        break;
    }

    // 5. 选择最优算法
    bool prefer_range_joins = DBConfig::GetSetting<PreferRangeJoinsSetting>(context);

    // 5a. 等值 Join → Hash Join（优先）
    if (has_equality && !prefer_range_joins) {
        auto &join = Make<PhysicalHashJoin>(op, left, right, std::move(op.conditions),
                                            op.join_type, op.left_projection_map,
                                            op.right_projection_map, ...);
        return join;
    }

    // 5b. 小表 → Nested Loop Join
    idx_t nested_loop_join_threshold = DBConfig::GetSetting<NestedLoopJoinThresholdSetting>(context);
    if (left.estimated_cardinality < nested_loop_join_threshold ||
        right.estimated_cardinality < nested_loop_join_threshold) {
        can_iejoin = false;
        can_merge = false;
    }

    // 5c. 范围 Join with 2+ 条件 → IE Join
    if (can_iejoin) {
        return Make<PhysicalIEJoin>(op, left, right, std::move(op.conditions), ...);
    }

    // 5d. 范围 Join → Piecewise Merge Join
    if (can_merge) {
        return Make<PhysicalPiecewiseMergeJoin>(op, left, right, std::move(op.conditions), ...);
    }

    // 5e. 不等式 Join → Nested Loop Join
    if (PhysicalNestedLoopJoin::IsSupported(op.conditions, op.join_type)) {
        return Make<PhysicalNestedLoopJoin>(op, left, right, std::move(op.conditions), ...);
    }

    // 5f. 复杂条件 → Blockwise Nested Loop Join（fallback）
    auto condition = JoinCondition::CreateExpression(std::move(op.conditions));
    return Make<PhysicalBlockwiseNLJoin>(op, left, right, std::move(condition), ...);
}
```

### 6.3.2 Join 算法对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB Join 算法选择                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  算法               条件                    复杂度        适用场景           │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Hash Join          等值条件                O(n+m)        大表等值 Join      │
│                     has_equality=true                                       │
│                                                                             │
│  IE Join            2+ 范围条件             O(n log n)    范围 Join          │
│                     has_range >= 2                       (inequality)       │
│                                                                             │
│  Piecewise          1 范围条件              O(n log n)    简单范围 Join      │
│  Merge Join         has_range = 1                                           │
│                                                                             │
│  Nested Loop        不等式条件              O(n*m)        小表或复杂条件     │
│  Join               小表 (<threshold)                                       │
│                                                                             │
│  Blockwise NL       复杂任意条件            O(n*m)        fallback          │
│  Join               (fallback)                                              │
│                                                                             │
│  Cross Product      无条件                  O(n*m)        笛卡尔积           │
│                     conditions.empty()                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3.3 算法选择决策树

```
                    有 Join 条件？
                         │
            ┌────────────┴────────────┐
            │                         │
           Yes                       No
            │                         │
            ▼                         ▼
      有等值条件？             PhysicalCrossProduct
            │
    ┌───────┴───────┐
    │               │
   Yes             No
    │               │
    ▼               ▼
PhysicalHashJoin  有范围条件？
                      │
              ┌───────┴───────┐
              │               │
             Yes             No
              │               │
              ▼               ▼
        范围条件数>=2？   PhysicalNestedLoopJoin
              │              (或 BlockwiseNLJoin)
      ┌───────┴───────┐
      │               │
     Yes             No
      │               │
      ▼               ▼
  基数够大？     PhysicalPiecewiseMergeJoin
      │
  ┌───┴───┐
  │       │
 Yes     No
  │       │
  ▼       ▼
PhysicalIEJoin  PhysicalNestedLoopJoin
```

---

## 6.4 聚合算法选择

DuckDB 提供四种聚合物理实现，根据数据特征和查询模式选择最优算法：

### 6.4.1 聚合算法选择逻辑

```cpp
// src/execution/physical_plan/plan_aggregate.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalAggregate &op) {
    // 1. 递归处理子节点
    reference<PhysicalOperator> plan = CreatePlan(*op.children[0]);
    plan = ExtractAggregateExpressions(plan, op.expressions, op.groups, op.grouping_sets);

    // 2. 检查是否支持简单聚合
    bool can_use_simple_aggregation = true;
    for (auto &expression : op.expressions) {
        auto &aggregate = expression->Cast<BoundAggregateExpression>();
        if (!aggregate.function.HasStateSimpleUpdateCallback()) {
            can_use_simple_aggregation = false;
            break;
        }
    }

    // 3. 无分组聚合
    if (op.groups.empty() && op.grouping_sets.size() <= 1) {
        if (can_use_simple_aggregation) {
            // Ungrouped Aggregate：无需 Hash Table
            return Make<PhysicalUngroupedAggregate>(op.types, std::move(op.expressions), ...);
        }
        // 需要 Hash Table 的无分组聚合
        return Make<PhysicalHashAggregate>(context, op.types, std::move(op.expressions), ...);
    }

    // 4. 分组聚合 - 尝试各种优化
    vector<column_t> partition_columns;
    vector<idx_t> required_bits;

    // 4a. Partitioned Aggregate（数据源已按分组列分区）
    if (can_use_simple_aggregation &&
        CanUsePartitionedAggregate(context, op, plan, partition_columns)) {
        return Make<PhysicalPartitionedAggregate>(context, op.types, std::move(op.expressions),
                                                   std::move(op.groups),
                                                   std::move(partition_columns), ...);
    }

    // 4b. Perfect Hash Aggregate（分组键范围小）
    if (CanUsePerfectHashAggregate(context, op, required_bits)) {
        return Make<PhysicalPerfectHashAggregate>(context, op.types, std::move(op.expressions),
                                                   std::move(op.groups),
                                                   std::move(required_bits), ...);
    }

    // 4c. Hash Aggregate（通用方案）
    return Make<PhysicalHashAggregate>(context, op.types, std::move(op.expressions),
                                        std::move(op.groups), std::move(op.grouping_sets), ...);
}
```

### 6.4.2 聚合算法对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 聚合算法选择                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PhysicalUngroupedAggregate                                                 │
│  ──────────────────────────────                                             │
│  • 条件：无 GROUP BY 子句，聚合函数支持简单更新                              │
│  • 实现：直接累加，无需 Hash Table                                          │
│  • 优点：内存占用最小，速度最快                                              │
│  • 示例：SELECT COUNT(*), SUM(x) FROM table                                 │
│                                                                             │
│  PhysicalPartitionedAggregate                                               │
│  ─────────────────────────────────                                          │
│  • 条件：数据源已按分组列分区（如 Parquet 按某列分区）                        │
│  • 实现：每个分区独立聚合，无需全局 Hash Table                               │
│  • 优点：流式处理，内存效率高                                                │
│  • 示例：GROUP BY partition_key WHERE source is partitioned by partition_key │
│                                                                             │
│  PhysicalPerfectHashAggregate                                               │
│  ─────────────────────────────────                                          │
│  • 条件：分组键为整数类型，值域范围 < 2^threshold (默认 22 bits)             │
│  • 实现：直接数组索引，避免 Hash 冲突                                        │
│  • 优点：无冲突，访问 O(1)                                                   │
│  • 示例：GROUP BY status WHERE status IN (1, 2, 3, 4, 5)                     │
│                                                                             │
│  PhysicalHashAggregate                                                      │
│  ───────────────────────                                                    │
│  • 条件：通用方案（fallback）                                               │
│  • 实现：基于 Hash Table 的分组聚合                                         │
│  • 优点：适用于任意分组键类型和数量                                          │
│  • 示例：GROUP BY name, category                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4.3 Perfect Hash 适用性检查

```cpp
static bool CanUsePerfectHashAggregate(ClientContext &context, LogicalAggregate &op,
                                        vector<idx_t> &bits_per_group) {
    if (op.grouping_sets.size() > 1 || !op.grouping_functions.empty()) {
        return false;
    }

    idx_t perfect_hash_bits = 0;
    for (idx_t group_idx = 0; group_idx < op.groups.size(); group_idx++) {
        auto &group = op.groups[group_idx];
        auto &stats = op.group_stats[group_idx];

        // 只支持整数类型
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

        // 需要统计信息来计算值域范围
        if (!stats || !NumericStats::HasMinMax(*stats)) {
            // 小类型可以用默认范围
            if (/* 类型太大 */) return false;
        }

        // 计算值域范围
        hugeint_t range = NumericStats::Max(*stats) - NumericStats::Min(*stats) + 2;

        // 范围不能太大
        if (range >= NumericLimits<int32_t>::Maximum()) {
            return false;
        }

        // 累计所需 bits
        idx_t required_bits = RequiredBitsForValue(range);
        bits_per_group.push_back(required_bits);
        perfect_hash_bits += required_bits;

        // 检查是否超过阈值（默认 22 bits）
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

---

## 6.5 表扫描计划生成

### 6.5.1 TableScan 生成逻辑

```cpp
// src/execution/physical_plan/plan_get.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalGet &op) {
    auto column_ids = op.GetColumnIds();

    // 1. 处理表函数（有子查询输入）
    if (!op.children.empty()) {
        reference<PhysicalOperator> child = ResolveAndPlan(std::move(op.children[0]));

        // 添加类型转换投影（如需要）
        if (any_cast_required) {
            auto &proj = Make<PhysicalProjection>(...);
            proj.children.push_back(child);
            child = proj;
        }

        auto &table_in_out = Make<PhysicalTableInOutFunction>(op.types, op.function,
                                                               std::move(op.bind_data), ...);
        table_in_out.children.push_back(child);
        return table_in_out;
    }

    // 2. 创建 TableFilterSet
    unique_ptr<TableFilterSet> table_filters;
    if (!op.table_filters.filters.empty()) {
        table_filters = CreateTableFilterSet(op.table_filters, column_ids);
    }

    // 3. 处理不支持下推的过滤器
    optional_ptr<PhysicalOperator> filter;
    if (table_filters && op.function.supports_pushdown_type) {
        // 检查哪些过滤器不能下推
        for (auto &entry : table_filters->filters) {
            if (!op.function.supports_pushdown_type(*op.bind_data, column_id)) {
                // 创建 PhysicalFilter 处理这些条件
                filter = Make<PhysicalFilter>(...);
            }
        }
    }

    // 4. 创建 TableScan
    op.ResolveOperatorTypes();

    if (!op.function.projection_pushdown) {
        // 不支持投影下推 → 扫描所有列 + 投影
        auto &table_scan = Make<PhysicalTableScan>(op.returned_types, op.function, ...);

        // 如果需要，添加投影
        if (/* 需要投影 */) {
            auto &proj = Make<PhysicalProjection>(...);
            proj.children.push_back(table_scan);
            return proj;
        }
        return table_scan;
    }

    // 5. 支持投影下推 → 只扫描需要的列
    auto &table_scan = Make<PhysicalTableScan>(op.types, op.function, std::move(op.bind_data),
                                                op.returned_types, column_ids, op.projection_ids,
                                                op.names, std::move(table_filters), ...);

    // 设置动态过滤器（运行时下推）
    table_scan.Cast<PhysicalTableScan>().dynamic_filters = op.dynamic_filters;

    if (filter) {
        filter->children.push_back(table_scan);
        return *filter;
    }
    return table_scan;
}
```

### 6.5.2 TableScan 的投影下推

```
┌─────────────────────────────────────────────────────────────────┐
│                    TableScan 投影下推                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始查询: SELECT a, b FROM t WHERE c > 10                       │
│                                                                 │
│  不支持投影下推:                  支持投影下推:                    │
│  ─────────────────────           ─────────────────               │
│                                                                 │
│  PhysicalProjection [a, b]       PhysicalFilter [c > 10]        │
│         │                               │                       │
│         ▼                               ▼                       │
│  PhysicalFilter [c > 10]         PhysicalTableScan              │
│         │                         column_ids: [a, b, c]         │
│         ▼                         projection_ids: [0, 1]        │
│  PhysicalTableScan                (只返回 a, b)                  │
│   (扫描所有列)                                                   │
│                                                                 │
│  效果：减少 I/O 和内存使用                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.6 Pipeline 构建

### 6.6.1 Pipeline 概念

Pipeline 是 DuckDB 执行模型的核心概念。一个 Pipeline 是一条从 Source 到 Sink 的数据流路径，中间不包含阻塞点。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pipeline 结构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pipeline                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Source ──▶ Operator ──▶ Operator ──▶ ... ──▶ Sink      │    │
│  │                                                         │    │
│  │  例如:                                                   │    │
│  │  TableScan ──▶ Filter ──▶ Projection ──▶ HashAggregate  │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  • Source: 产生数据（IsSource() = true）                        │
│  • Operator: 处理数据（中间节点）                               │
│  • Sink: 消费数据，可能阻塞（IsSink() = true）                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.6.2 Pipeline 构建算法

```cpp
// src/include/duckdb/parallel/pipeline.hpp

class Pipeline : public enable_shared_from_this<Pipeline> {
    Executor &executor;

    // Pipeline 组成
    optional_ptr<PhysicalOperator> source;           // 数据源
    vector<reference<PhysicalOperator>> operators;   // 中间算子
    optional_ptr<PhysicalOperator> sink;             // 数据汇

    // 状态
    unique_ptr<GlobalSourceState> source_state;

    // 依赖关系
    vector<weak_ptr<Pipeline>> parents;       // 依赖此 Pipeline
    vector<weak_ptr<Pipeline>> dependencies;  // 此 Pipeline 依赖的
};

// src/execution/physical_operator.cpp

void PhysicalOperator::BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline) {
    op_state.reset();
    auto &state = meta_pipeline.GetState();

    // Case 1: Source（叶子节点，非 Sink）
    if (!IsSink() && children.empty()) {
        state.SetPipelineSource(current, *this);
        return;
    }

    // 目前只支持单子节点
    if (children.size() != 1) {
        throw InternalException("Operator not supported in BuildPipelines");
    }

    // Case 2: Sink（阻塞点）
    if (IsSink()) {
        sink_state.reset();

        // Sink 成为当前 Pipeline 的 Source（因为它产生输出）
        state.SetPipelineSource(current, *this);

        // 创建新的子 MetaPipeline 处理子节点
        auto &child_meta_pipeline = meta_pipeline.CreateChildMetaPipeline(current, *this);
        child_meta_pipeline.Build(children[0].get());
        return;
    }

    // Case 3: 中间算子
    state.AddPipelineOperator(current, *this);
    children[0].get().BuildPipelines(current, meta_pipeline);
}
```

### 6.6.3 MetaPipeline 概念

`MetaPipeline` 表示一组具有相同 Sink 的 Pipeline：

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp

enum class MetaPipelineType : uint8_t {
    REGULAR = 0,    // 普通 Pipeline
    JOIN_BUILD = 1  // Join Build 侧
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
    Executor &executor;
    PipelineBuildState &state;

    optional_ptr<Pipeline> parent;              // 父 Pipeline
    optional_ptr<PhysicalOperator> sink;        // 共享的 Sink
    MetaPipelineType type;

    vector<shared_ptr<Pipeline>> pipelines;              // 同一 Sink 的多个 Pipeline
    vector<shared_ptr<MetaPipeline>> children;           // 子 MetaPipeline
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;

public:
    void Build(PhysicalOperator &op);

    // 创建子 Pipeline
    Pipeline &CreatePipeline();
    Pipeline &CreateUnionPipeline(Pipeline &current, bool order_matters);
    MetaPipeline &CreateChildMetaPipeline(Pipeline &current, PhysicalOperator &op,
                                          MetaPipelineType type = MetaPipelineType::REGULAR);
};
```

### 6.6.4 Pipeline 构建示例

```sql
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
GROUP BY c.name
ORDER BY SUM(o.amount) DESC
LIMIT 10;
```

```
物理计划:
                                Pipeline 划分:
┌────────────────────┐
│    PhysicalTopN    │ ◄─────── Pipeline 3 Source (Sink 已完成后的输出)
│         │          │
│         ▼          │
│  PhysicalHashAgg   │ ◄─────── Pipeline 3 Sink (阻塞点)
│         │          │          Pipeline 2 Source
│         ▼          │
│   PhysicalFilter   │ ◄─────── Pipeline 2 Operator
│         │          │
│         ▼          │
│  PhysicalHashJoin  │ ◄─────── Pipeline 2 Sink (阻塞点)
│       /    \       │          Pipeline 1 Source (probe 侧)
│      /      \      │
│  Probe     Build   │
│    │         │     │
│    ▼         ▼     │
│ TableScan TableScan│ ◄─────── Pipeline 1: customers (probe)
│ customers  orders  │          Pipeline 0: orders (build)
└────────────────────┘

Pipeline 执行顺序:
1. Pipeline 0: TableScan(orders) → HashJoin(build)
2. Pipeline 1: TableScan(customers) → Filter → HashJoin(probe) → HashAggregate
3. Pipeline 2: HashAggregate(output) → TopN
4. Pipeline 3: TopN(output) → ResultCollector
```

---

## 6.7 EXPLAIN 输出

### 6.7.1 EXPLAIN 计划生成

```cpp
// src/execution/physical_plan/plan_explain.cpp

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalExplain &op) {
    D_ASSERT(op.children.size() == 1);

    // 获取优化后的逻辑计划字符串
    auto logical_plan_opt = op.children[0]->ToString(op.explain_format);

    // 生成物理计划
    auto &plan = CreatePlan(*op.children[0]);

    // EXPLAIN ANALYZE: 实际执行并收集统计
    if (op.explain_type == ExplainType::EXPLAIN_ANALYZE) {
        auto &explain = Make<PhysicalExplainAnalyze>(op.types, op.explain_format);
        explain.children.push_back(plan);
        return explain;
    }

    // EXPLAIN: 只显示计划
    op.physical_plan = plan.ToString(op.explain_format);

    // 根据配置决定输出内容
    vector<string> keys, values;
    switch (ClientConfig::GetConfig(context).explain_output_type) {
    case ExplainOutputType::OPTIMIZED_ONLY:
        keys = {"logical_opt"};
        values = {logical_plan_opt};
        break;
    case ExplainOutputType::PHYSICAL_ONLY:
        keys = {"physical_plan"};
        values = {op.physical_plan};
        break;
    default:
        keys = {"logical_plan", "logical_opt", "physical_plan"};
        values = {op.logical_plan_unopt, logical_plan_opt, op.physical_plan};
    }

    // 创建结果集
    auto collection = make_uniq<ColumnDataCollection>(...);
    // ... 填充数据

    return Make<PhysicalColumnDataScan>(op.types, PhysicalOperatorType::COLUMN_DATA_SCAN,
                                         op.estimated_cardinality, std::move(collection));
}
```

### 6.7.2 EXPLAIN 输出示例

```sql
EXPLAIN SELECT * FROM customers WHERE id = 1;
```

```
┌─────────────────────────────────────────────────────────────┐
│                      EXPLAIN Output                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  physical_plan                                              │
│  ─────────────                                              │
│  ┌───────────────────────────┐                              │
│  │       TABLE_SCAN          │                              │
│  │   ─────────────────────   │                              │
│  │   Table: customers        │                              │
│  │   Filters: id=1           │                              │
│  │   Columns: [id, name...]  │                              │
│  │   Est. Card: 1            │                              │
│  └───────────────────────────┘                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
EXPLAIN ANALYZE SELECT COUNT(*) FROM orders GROUP BY customer_id;
```

```
┌─────────────────────────────────────────────────────────────┐
│                   EXPLAIN ANALYZE Output                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ HASH_GROUP_BY                                        │   │
│  │   Actual Rows: 1000                                  │   │
│  │   Execution Time: 45ms                               │   │
│  │   Memory: 128KB                                      │   │
│  │                 │                                    │   │
│  │                 ▼                                    │   │
│  │ ┌────────────────────────────────────────────────┐   │   │
│  │ │ TABLE_SCAN                                     │   │   │
│  │ │   Table: orders                                │   │   │
│  │ │   Actual Rows: 100000                          │   │   │
│  │ │   Execution Time: 120ms                        │   │   │
│  │ │   I/O: 5MB read                                │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.8 其他算子转换

### 6.8.1 简单算子

```cpp
// Filter
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalFilter &op) {
    auto &plan = CreatePlan(*op.children[0]);
    auto &filter = Make<PhysicalFilter>(op.types, std::move(op.expressions),
                                         op.estimated_cardinality);
    filter.children.push_back(plan);
    return filter;
}

// Projection
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalProjection &op) {
    auto &plan = CreatePlan(*op.children[0]);
    auto &proj = Make<PhysicalProjection>(op.types, std::move(op.expressions),
                                           op.estimated_cardinality);
    proj.children.push_back(plan);
    return proj;
}

// Order By
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOrder &op) {
    auto &plan = CreatePlan(*op.children[0]);
    auto &order = Make<PhysicalOrder>(op.types, std::move(op.orders),
                                       std::move(op.projections), op.estimated_cardinality);
    order.children.push_back(plan);
    return order;
}

// Limit
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalLimit &op) {
    auto &plan = CreatePlan(*op.children[0]);
    auto &limit = Make<PhysicalLimit>(op.types, op.limit_val, op.offset_val,
                                       op.estimated_cardinality);
    limit.children.push_back(plan);
    return limit;
}
```

### 6.8.2 集合操作

```cpp
// UNION
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalSetOperation &op) {
    auto &left = CreatePlan(*op.children[0]);
    auto &right = CreatePlan(*op.children[1]);

    switch (op.type) {
    case LogicalOperatorType::LOGICAL_UNION:
        return Make<PhysicalUnion>(op.types, left, right, op.estimated_cardinality);
    case LogicalOperatorType::LOGICAL_INTERSECT:
        // 使用 Hash Join 实现
        return Make<PhysicalHashJoin>(/* SEMI JOIN 实现 INTERSECT */);
    case LogicalOperatorType::LOGICAL_EXCEPT:
        // 使用 Hash Join 实现
        return Make<PhysicalHashJoin>(/* ANTI JOIN 实现 EXCEPT */);
    }
}
```

---

## 6.9 CachingPhysicalOperator

DuckDB 提供了 `CachingPhysicalOperator` 基类来优化小 Chunk 处理：

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class CachingPhysicalOperator : public PhysicalOperator {
public:
    static constexpr const idx_t CACHE_THRESHOLD = 64;

    bool caching_supported;

    // 包装 Execute 方法，缓存小 Chunk
    OperatorResultType Execute(ExecutionContext &context, DataChunk &input, DataChunk &chunk,
                               GlobalOperatorState &gstate, OperatorState &state) const final {
        auto &caching_state = state.Cast<CachingOperatorState>();

        // 调用实际执行
        auto result = ExecuteInternal(context, input, chunk, gstate, caching_state);

        // 如果结果太小，缓存起来
        if (chunk.size() < CACHE_THRESHOLD && caching_state.can_cache_chunk) {
            if (!caching_state.cached_chunk) {
                caching_state.cached_chunk = make_uniq<DataChunk>();
                caching_state.cached_chunk->Initialize(chunk.GetTypes());
            }

            caching_state.cached_chunk->Append(chunk);

            // 缓存满了才返回
            if (caching_state.cached_chunk->size() >= (STANDARD_VECTOR_SIZE - CACHE_THRESHOLD)) {
                chunk.Move(*caching_state.cached_chunk);
                return result;
            }

            // 缓存未满，返回空
            chunk.Reset();
        }

        return result;
    }

protected:
    // 子类实现实际执行逻辑
    virtual OperatorResultType ExecuteInternal(...) const = 0;
};
```

---

## 6.10 完整示例

### 6.10.1 查询示例

```sql
SELECT
    c.name,
    COUNT(*) as order_count,
    SUM(o.amount) as total_amount
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
  AND o.date >= '2024-01-01'
GROUP BY c.name
HAVING SUM(o.amount) > 1000
ORDER BY total_amount DESC
LIMIT 10;
```

### 6.10.2 物理计划生成过程

```
输入: 优化后的逻辑计划

LogicalLimit (10)
    │
    ▼
LogicalOrder (total_amount DESC)
    │
    ▼
LogicalFilter (SUM(o.amount) > 1000)  -- HAVING
    │
    ▼
LogicalAggregate
  Groups: [c.name]
  Aggregates: [COUNT(*), SUM(o.amount)]
    │
    ▼
LogicalComparisonJoin (c.id = o.customer_id)
    │
    ├──▶ LogicalFilter (c.country = 'China')
    │        │
    │        ▼
    │    LogicalGet (customers)
    │
    └──▶ LogicalFilter (o.date >= '2024-01-01')
             │
             ▼
         LogicalGet (orders)

═══════════════════════════════════════════════════════

物理计划生成:

1. CreatePlan(LogicalLimit)
   → Make<PhysicalLimit>(10)

2. CreatePlan(LogicalOrder)
   → Make<PhysicalOrder>(total_amount DESC)

3. CreatePlan(LogicalFilter - HAVING)
   → Make<PhysicalFilter>(SUM > 1000)

4. CreatePlan(LogicalAggregate)
   → 检查 Perfect Hash: 不适用(name 是 VARCHAR)
   → 检查 Partitioned: 不适用
   → Make<PhysicalHashAggregate>

5. CreatePlan(LogicalComparisonJoin)
   → 有等值条件 c.id = o.customer_id
   → Make<PhysicalHashJoin>

6. CreatePlan(LogicalFilter - WHERE)
   → 已下推到 TableScan

7. CreatePlan(LogicalGet - customers)
   → Make<PhysicalTableScan> with filters

8. CreatePlan(LogicalGet - orders)
   → Make<PhysicalTableScan> with filters

═══════════════════════════════════════════════════════

输出: 物理计划

PhysicalLimit (10)
    │
    ▼
PhysicalOrder (total_amount DESC)
    │
    ▼
PhysicalFilter (SUM(o.amount) > 1000)
    │
    ▼
PhysicalHashAggregate
  Groups: [c.name]
  Aggregates: [COUNT(*), SUM(o.amount)]
    │
    ▼
PhysicalHashJoin (INNER, c.id = o.customer_id)
    │
    ├──▶ PhysicalTableScan (customers)
    │      Filters: country = 'China'
    │      Columns: [id, name]
    │
    └──▶ PhysicalTableScan (orders)
           Filters: date >= '2024-01-01'
           Columns: [customer_id, amount]

═══════════════════════════════════════════════════════

Pipeline 构建:

Pipeline 0 (Build Side):
  Source: PhysicalTableScan(orders)
  Sink: PhysicalHashJoin(build)

Pipeline 1 (Probe + Aggregate):
  Source: PhysicalTableScan(customers)
  Operators: [PhysicalHashJoin(probe)]
  Sink: PhysicalHashAggregate

Pipeline 2 (Output):
  Source: PhysicalHashAggregate
  Operators: [PhysicalFilter, PhysicalOrder]
  Sink: PhysicalLimit

Pipeline 3 (Result):
  Source: PhysicalLimit
  Sink: ResultCollector

依赖关系:
  Pipeline 1 depends on Pipeline 0
  Pipeline 2 depends on Pipeline 1
  Pipeline 3 depends on Pipeline 2
```

---

## 6.11 源文件索引

| 文件 | 功能 |
|------|------|
| `src/include/duckdb/execution/physical_plan_generator.hpp` | PhysicalPlanGenerator 类定义 |
| `src/include/duckdb/execution/physical_operator.hpp` | PhysicalOperator 基类 |
| `src/include/duckdb/common/enums/physical_operator_type.hpp` | 物理算子类型枚举 |
| `src/execution/physical_plan/plan_comparison_join.cpp` | Join 算法选择 |
| `src/execution/physical_plan/plan_aggregate.cpp` | 聚合算法选择 |
| `src/execution/physical_plan/plan_get.cpp` | TableScan 生成 |
| `src/execution/physical_plan/plan_explain.cpp` | EXPLAIN 处理 |
| `src/execution/physical_operator.cpp` | Pipeline 构建 |
| `src/include/duckdb/parallel/pipeline.hpp` | Pipeline 类 |
| `src/include/duckdb/parallel/meta_pipeline.hpp` | MetaPipeline 类 |

---

## 6.12 总结

Physical Planner 是将逻辑计划转换为可执行物理计划的关键阶段：

1. **算法选择智能化**：根据数据特征、统计信息、Join 条件类型等选择最优物理算法
2. **多种 Join 实现**：Hash Join、IE Join、Merge Join、Nested Loop Join 各有适用场景
3. **聚合优化**：Perfect Hash、Partitioned、Ungrouped 等专用实现
4. **Pipeline 模型**：将物理计划切分为可并行执行的 Pipeline，提高执行效率
5. **投影下推**：尽早裁剪不需要的列，减少数据传输

物理计划生成完成后，执行引擎将按照 Pipeline 依赖关系调度执行，这是查询执行的最后一步，将在执行引擎章节详细介绍。

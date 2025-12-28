# DuckDB 优化器深度解析：第三章 - Join Order 优化

## 3.1 章节概述

Join Order 优化是数据库查询优化中最关键的环节之一。对于一个包含 n 个表的 JOIN 查询，不同的 JOIN 顺序可能产生截然不同的执行代价，候选计划数量随表数呈指数级增长。DuckDB 实现了基于动态规划的 Join Order 优化器，使用 DPhyp (Dynamic Programming with Hypergraph) 算法高效地搜索最优 JOIN 顺序。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Join Order 优化系统架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入：逻辑计划树 (LogicalOperator)                                       │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 1. RelationManager：提取可重排序的关系                        │      │
│  │    - 识别基表扫描 (TableScan)                                 │      │
│  │    - 识别可重排序的 Join (Inner, Semi, Anti)                  │      │
│  │    - 处理非可重排序操作 (Left/Right/Outer Join)               │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 2. QueryGraphManager：构建查询图                              │      │
│  │    - 提取 Join 条件作为边                                     │      │
│  │    - 管理 JoinRelationSet                                     │      │
│  │    - 构建超图边 (HyperGraph Edges)                            │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 3. CostModel + CardinalityEstimator：代价估算                  │      │
│  │    - 基于统计信息估算基数                                      │      │
│  │    - 计算 Join 计划的代价                                     │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 4. PlanEnumerator：计划枚举 (DPhyp 算法)                      │      │
│  │    - 精确动态规划 (≤12 个关系)                                 │      │
│  │    - 贪心近似 (>12 个关系或超时)                               │      │
│  │    - Cross Product 处理                                       │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ 5. Reconstruct：重建逻辑计划                                   │      │
│  │    - 根据最优计划生成 Join 树                                  │      │
│  │    - 推送剩余过滤条件                                          │      │
│  └──────────────────────────────────────────────────────────────┘      │
│     │                                                                   │
│     ▼                                                                   │
│  输出：优化后的逻辑计划树                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3.2 JoinOrderOptimizer 入口

### 3.2.1 类设计

`JoinOrderOptimizer` 是 Join Order 优化的主入口类：

```cpp
// src/include/duckdb/optimizer/join_order/join_order_optimizer.hpp

class JoinOrderOptimizer {
public:
    explicit JoinOrderOptimizer(ClientContext &context);
    JoinOrderOptimizer CreateChildOptimizer();

    //! 执行 Join 重排序优化
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan,
                                          optional_ptr<RelationStats> stats = nullptr);

    //! CTE 统计信息管理
    void AddMaterializedCTEStats(idx_t index, RelationStats &&stats);
    RelationStats GetMaterializedCTEStats(idx_t index);

private:
    ClientContext &context;

    //! 管理查询图、关系和边
    QueryGraphManager query_graph_manager;

    //! 从查询图提取的过滤条件
    vector<unique_ptr<Expression>> filters;
    vector<unique_ptr<FilterInfo>> filter_infos;

    //! 等价表达式集合（用于生成隐含边）
    expression_map_t<vector<FilterInfo *>> equivalence_sets;

    //! 基数估算器
    CardinalityEstimator cardinality_estimator;

    //! 优化深度控制（防止栈溢出）
    idx_t depth;
};
```

### 3.2.2 优化主流程

```cpp
// src/optimizer/join_order/join_order_optimizer.cpp

unique_ptr<LogicalOperator> JoinOrderOptimizer::Optimize(unique_ptr<LogicalOperator> plan,
                                                         optional_ptr<RelationStats> stats) {
    // 深度检查，防止过深的计划消耗过多栈空间
    if (depth > query_graph_manager.context.config.max_expression_depth) {
        return plan;
    }

    LogicalOperator *op = plan.get();

    // 步骤 1: 提取可重排序的关系和 Join 条件
    bool reorderable = query_graph_manager.Build(*this, *op);

    // 获取关系统计信息
    auto relation_stats = query_graph_manager.relation_manager.GetRelationStats();
    unique_ptr<LogicalOperator> new_logical_plan = nullptr;

    if (reorderable) {
        // 步骤 2: 创建代价模型
        auto cost_model = CostModel(query_graph_manager);

        // 步骤 3: 初始化计划枚举器
        auto plan_enumerator = PlanEnumerator(query_graph_manager, cost_model,
                                              query_graph_manager.GetQueryGraphEdges());

        // 步骤 4: 初始化叶子节点计划
        plan_enumerator.InitLeafPlans();

        // 步骤 5: 求解最优 Join 顺序
        plan_enumerator.SolveJoinOrder();

        // 步骤 6: 重建逻辑计划
        query_graph_manager.plans = &plan_enumerator.GetPlans();
        new_logical_plan = query_graph_manager.Reconstruct(std::move(plan));
    } else {
        // 不可重排序，直接返回原计划
        new_logical_plan = std::move(plan);
    }

    // 步骤 7: 传播统计信息
    if (stats) {
        auto cardinality = new_logical_plan->EstimateCardinality(context);
        // ... 合并统计信息
    }

    return new_logical_plan;
}
```

### 3.2.3 子优化器创建

对于非可重排序的子查询（如 Left Join、Union 等），需要递归创建子优化器：

```cpp
JoinOrderOptimizer JoinOrderOptimizer::CreateChildOptimizer() {
    JoinOrderOptimizer child_optimizer(context);
    // 继承 CTE 统计信息
    child_optimizer.materialized_cte_stats = materialized_cte_stats;
    child_optimizer.delim_scan_stats = delim_scan_stats;
    child_optimizer.depth = depth + 1;  // 深度+1
    child_optimizer.recursive_cte_indexes = recursive_cte_indexes;
    return child_optimizer;
}
```

---

## 3.3 RelationManager：关系提取

### 3.3.1 核心结构

```cpp
// src/include/duckdb/optimizer/join_order/relation_manager.hpp

//! 表示单个关系及其元数据
struct SingleJoinRelation {
    LogicalOperator &op;            // 关系对应的算子
    optional_ptr<LogicalOperator> parent;  // 父算子
    RelationStats stats;            // 统计信息
};

class RelationManager {
public:
    //! 从逻辑计划提取 Join 关系
    bool ExtractJoinRelations(JoinOrderOptimizer &optimizer,
                              LogicalOperator &input_op,
                              vector<reference<LogicalOperator>> &filter_operators,
                              optional_ptr<LogicalOperator> parent = nullptr);

    //! 提取 Join 边（过滤条件）
    vector<unique_ptr<FilterInfo>> ExtractEdges(LogicalOperator &op,
                                                vector<reference<LogicalOperator>> &filter_operators,
                                                JoinRelationSetManager &set_manager);

    //! 从表达式提取绑定
    bool ExtractBindings(Expression &expression, unordered_set<idx_t> &bindings);

private:
    //! 所有关系集合
    vector<unique_ptr<SingleJoinRelation>> relations;

    //! 表索引 -> 关系编号的映射
    unordered_map<idx_t, idx_t> relation_mapping;
};
```

### 3.3.2 可重排序判断

并非所有 Join 都可以重新排序。以下是判断逻辑：

```cpp
static bool JoinIsReorderable(LogicalOperator &op) {
    if (op.type == LogicalOperatorType::LOGICAL_CROSS_PRODUCT) {
        return true;  // Cross Product 总是可重排序
    } else if (op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
        auto &join = op.Cast<LogicalComparisonJoin>();
        switch (join.join_type) {
        case JoinType::INNER:
        case JoinType::SEMI:
        case JoinType::ANTI:
            // 仅当条件包含列引用时才可重排序
            for (auto &cond : join.conditions) {
                if (ExpressionContainsColumnRef(*cond.left) &&
                    ExpressionContainsColumnRef(*cond.right)) {
                    return true;
                }
            }
            return false;
        default:
            return false;  // Left/Right/Outer Join 不可重排序
        }
    }
    return false;
}
```

**可重排序的 Join 类型：**
- `INNER JOIN`
- `SEMI JOIN`
- `ANTI JOIN`
- `CROSS PRODUCT`

**不可重排序的 Join 类型：**
- `LEFT JOIN`
- `RIGHT JOIN`
- `OUTER JOIN`
- `ASOF JOIN`
- `ANY JOIN`

### 3.3.3 关系提取流程

```cpp
bool RelationManager::ExtractJoinRelations(JoinOrderOptimizer &optimizer,
                                            LogicalOperator &input_op,
                                            vector<reference<LogicalOperator>> &filter_operators,
                                            optional_ptr<LogicalOperator> parent) {
    optional_ptr<LogicalOperator> op = &input_op;

    // 跳过单子节点算子（Filter, Limit 等）
    while (op->children.size() == 1 && !OperatorNeedsRelation(op->type)) {
        if (op->type == LogicalOperatorType::LOGICAL_FILTER) {
            filter_operators.push_back(*op);  // 收集过滤算子
        }
        op = op->children[0].get();
    }

    // 处理不同类型的算子
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_GET:
        // 基表扫描，提取统计信息并添加为关系
        auto stats = RelationStatisticsHelper::ExtractGetStats(get, context);
        AddRelation(input_op, parent, stats);
        return true;

    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
        if (JoinIsReorderable(*op)) {
            // 可重排序 Join：递归提取左右子树
            filter_operators.push_back(*op);
            bool can_reorder_left = ExtractJoinRelations(optimizer, *op->children[0],
                                                         filter_operators, op);
            bool can_reorder_right = ExtractJoinRelations(optimizer, *op->children[1],
                                                          filter_operators, op);
            return can_reorder_left && can_reorder_right;
        } else {
            // 不可重排序 Join：作为独立优化单元
            // 递归优化子树
            return true;
        }

    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
    case LogicalOperatorType::LOGICAL_WINDOW:
        // 阻塞算子：创建子优化器优化子树
        auto child_optimizer = optimizer.CreateChildOptimizer();
        op->children[0] = child_optimizer.Optimize(std::move(op->children[0]), &child_stats);
        AddAggregateOrWindowRelation(input_op, parent, operator_stats, op->type);
        return true;
    }
}
```

### 3.3.4 Semi/Anti Join 特殊处理

Semi 和 Anti Join 需要特殊处理，因为右侧列在 Join 后会消失：

```cpp
if (join.join_type == JoinType::SEMI || join.join_type == JoinType::ANTI) {
    RelationStats child_stats;
    auto child_optimizer = optimizer.CreateChildOptimizer();
    // 右侧作为独立关系，不参与重排序
    op->children[1] = child_optimizer.Optimize(std::move(op->children[1]), &child_stats);
    AddRelation(*op->children[1], op, child_stats);

    // 禁止与此关系形成 Cross Product
    no_cross_product_relations.insert(relations.size() - 1);
}
```

```
Semi/Anti Join 约束示意：

原始查询：(A ⨝ B) ⋉ C
其中：A.x = B.y AND B.z = C.w

如果错误重排为：(A ⋉ C) ⨝ B
则 B.z = C.w 条件无法执行（C 的列在 Semi Join 后消失）

正确做法：将 Semi Join 的右侧 C 作为不可移动的关系
```

---

## 3.4 JoinRelationSet：关系集合表示

### 3.4.1 数据结构

```cpp
// src/include/duckdb/optimizer/join_order/join_relation.hpp

//! 关系集合，用于 Join 图中表示节点
struct JoinRelationSet {
    unsafe_unique_array<idx_t> relations;  // 关系 ID 数组（有序）
    idx_t count;                           // 关系数量

    string ToString() const;
    static bool IsSubset(JoinRelationSet &super, JoinRelationSet &sub);
};

//! 管理所有 JoinRelationSet 对象，保证唯一性
class JoinRelationSetManager {
public:
    //! 获取单个关系的集合
    JoinRelationSet &GetJoinRelation(idx_t index);

    //! 从绑定集合获取关系集合
    JoinRelationSet &GetJoinRelation(const unordered_set<idx_t> &bindings);

    //! 合并两个关系集合
    JoinRelationSet &Union(JoinRelationSet &left, JoinRelationSet &right);

private:
    //! 使用 Trie 树结构存储，保证唯一性
    struct JoinRelationTreeNode {
        unique_ptr<JoinRelationSet> relation;
        unordered_map<idx_t, unique_ptr<JoinRelationTreeNode>> children;
    };

    JoinRelationTreeNode root;
};
```

### 3.4.2 Union 操作

合并两个关系集合（归并排序）：

```cpp
JoinRelationSet &JoinRelationSetManager::Union(JoinRelationSet &left, JoinRelationSet &right) {
    auto relations = make_unsafe_uniq_array<idx_t>(left.count + right.count);
    idx_t count = 0;
    idx_t i = 0, j = 0;

    while (true) {
        if (i == left.count) {
            // 左侧耗尽，添加右侧剩余
            for (; j < right.count; j++) {
                relations[count++] = right.relations[j];
            }
            break;
        } else if (j == right.count) {
            // 右侧耗尽，添加左侧剩余
            for (; i < left.count; i++) {
                relations[count++] = left.relations[i];
            }
            break;
        } else if (left.relations[i] < right.relations[j]) {
            relations[count++] = left.relations[i++];
        } else if (left.relations[i] > right.relations[j]) {
            relations[count++] = right.relations[j++];
        } else {
            // 相等，只添加一次
            relations[count++] = left.relations[i];
            i++; j++;
        }
    }
    return GetJoinRelation(std::move(relations), count);
}
```

```
关系集合示例：

表: A=0, B=1, C=2, D=3

集合 {A, B} 表示为: [0, 1]
集合 {B, C, D} 表示为: [1, 2, 3]

Union({A,B}, {C,D}) = [0, 1, 2, 3]
Union({A,B}, {B,C}) = [0, 1, 2]  // B 只出现一次
```

---

## 3.5 QueryGraphManager：查询图管理

### 3.5.1 类设计

```cpp
// src/include/duckdb/optimizer/join_order/query_graph_manager.hpp

//! 过滤信息，表示 Join 条件
class FilterInfo {
public:
    unique_ptr<Expression> filter;     // 过滤表达式
    reference<JoinRelationSet> set;    // 涉及的关系集合
    idx_t filter_index;                // 过滤条件索引
    JoinType join_type;                // Join 类型

    optional_ptr<JoinRelationSet> left_set;   // 左侧关系
    optional_ptr<JoinRelationSet> right_set;  // 右侧关系
    ColumnBinding left_binding;        // 左侧列绑定
    ColumnBinding right_binding;       // 右侧列绑定
};

class QueryGraphManager {
public:
    //! 从逻辑计划构建查询图
    bool Build(JoinOrderOptimizer &optimizer, LogicalOperator &op);

    //! 重建逻辑计划
    unique_ptr<LogicalOperator> Reconstruct(unique_ptr<LogicalOperator> plan);

    //! 创建 Cross Product 边
    void CreateQueryGraphCrossProduct(JoinRelationSet &left, JoinRelationSet &right);

    RelationManager relation_manager;      // 关系管理器
    JoinRelationSetManager set_manager;    // 集合管理器

    //! 最优计划（由 PlanEnumerator 填充）
    optional_ptr<const reference_map_t<JoinRelationSet, unique_ptr<DPJoinNode>>> plans;

private:
    //! 过滤算子引用
    vector<reference<LogicalOperator>> filter_operators;

    //! 过滤信息及绑定
    vector<unique_ptr<FilterInfo>> filters_and_bindings;

    //! 查询图边
    QueryGraphEdges query_graph;

    void CreateHyperGraphEdges();
};
```

### 3.5.2 构建查询图

```cpp
bool QueryGraphManager::Build(JoinOrderOptimizer &optimizer, LogicalOperator &op) {
    // 步骤 1: 提取关系
    auto can_reorder = relation_manager.ExtractJoinRelations(optimizer, op, filter_operators);
    auto num_relations = relation_manager.NumRelations();

    if (num_relations <= 1 || !can_reorder) {
        return false;  // 无需优化
    }

    // 步骤 2: 提取边（Join 条件）
    filters_and_bindings = relation_manager.ExtractEdges(op, filter_operators, set_manager);

    // 步骤 3: 创建超图边
    CreateHyperGraphEdges();

    return true;
}
```

### 3.5.3 创建超图边

```cpp
void QueryGraphManager::CreateHyperGraphEdges() {
    for (auto &filter_info : filters_and_bindings) {
        auto &filter = filter_info->filter;

        if (filter->GetExpressionClass() == ExpressionClass::BOUND_COMPARISON) {
            auto &comparison = filter->Cast<BoundComparisonExpression>();

            // 提取左右两侧的关系绑定
            unordered_set<idx_t> left_bindings, right_bindings;
            relation_manager.ExtractBindings(*comparison.left, left_bindings);
            relation_manager.ExtractBindings(*comparison.right, right_bindings);

            if (!left_bindings.empty() && !right_bindings.empty()) {
                // 创建关系集合
                filter_info->left_set = &set_manager.GetJoinRelation(left_bindings);
                filter_info->right_set = &set_manager.GetJoinRelation(right_bindings);

                // 检查两侧是否不相交
                if (Disjoint(left_bindings, right_bindings)) {
                    // 创建双向边
                    query_graph.CreateEdge(*filter_info->left_set,
                                          *filter_info->right_set, filter_info);
                    query_graph.CreateEdge(*filter_info->right_set,
                                          *filter_info->left_set, filter_info);
                }
            }
        }
    }
}
```

```
查询图示例：

SQL: SELECT * FROM A, B, C, D
     WHERE A.x = B.y
       AND B.z = C.w
       AND A.p = D.q

查询图：
        A ────── B
        │       │
        │       │
        D       C

边:
  [A] ↔ [B] : A.x = B.y
  [B] ↔ [C] : B.z = C.w
  [A] ↔ [D] : A.p = D.q
```

---

## 3.6 QueryGraphEdges：图边存储

### 3.6.1 数据结构

```cpp
// src/include/duckdb/optimizer/join_order/query_graph.hpp

struct NeighborInfo {
    optional_ptr<JoinRelationSet> neighbor;  // 邻居关系集合
    vector<optional_ptr<FilterInfo>> filters; // 连接的过滤条件
};

class QueryGraphEdges {
public:
    struct QueryEdge {
        vector<unique_ptr<NeighborInfo>> neighbors;  // 邻居列表
        unordered_map<idx_t, unique_ptr<QueryEdge>> children;  // 子节点（Trie 结构）
    };

    //! 创建边
    void CreateEdge(JoinRelationSet &left, JoinRelationSet &right,
                    optional_ptr<FilterInfo> info);

    //! 获取两个集合之间的连接
    const vector<reference<NeighborInfo>> GetConnections(JoinRelationSet &node,
                                                         JoinRelationSet &other) const;

    //! 获取邻居（排除指定集合）
    const vector<idx_t> GetNeighbors(JoinRelationSet &node,
                                     unordered_set<idx_t> &exclusion_set) const;

private:
    QueryEdge root;
};
```

### 3.6.2 边创建与查询

```cpp
void QueryGraphEdges::CreateEdge(JoinRelationSet &left, JoinRelationSet &right,
                                  optional_ptr<FilterInfo> filter_info) {
    // 找到或创建左侧集合对应的 QueryEdge
    auto info = GetQueryEdge(left);

    // 检查是否已存在到右侧的边
    for (idx_t i = 0; i < info->neighbors.size(); i++) {
        if (info->neighbors[i]->neighbor == &right) {
            // 已存在，添加过滤条件
            if (filter_info) {
                info->neighbors[i]->filters.push_back(filter_info);
            }
            return;
        }
    }

    // 创建新的邻居
    auto n = make_uniq<NeighborInfo>(&right);
    if (filter_info) {
        n->filters.push_back(filter_info);
    }
    info->neighbors.push_back(std::move(n));
}
```

---

## 3.7 PlanEnumerator：DPhyp 算法实现

### 3.7.1 算法概述

DuckDB 的 Join Order 优化基于 "Dynamic Programming Strikes Back" 论文实现的 DPhyp 算法。核心思想是：

1. **Connected Subgraph (CSG) 枚举**：枚举所有连通子图
2. **Complement (Cmp) 枚举**：对每个子图枚举其补集
3. **动态规划**：记录每个关系集合的最优计划

```cpp
// src/include/duckdb/optimizer/join_order/plan_enumerator.hpp

class PlanEnumerator {
public:
    static constexpr idx_t THRESHOLD_TO_SWAP_TO_APPROXIMATE = 12;  // 切换到近似算法的阈值

    void SolveJoinOrder();
    void InitLeafPlans();

private:
    //! 精确求解
    bool SolveJoinOrderExactly();

    //! 近似求解（贪心）
    void SolveJoinOrderApproximately();

    //! CSG 枚举
    bool EmitCSG(JoinRelationSet &node);
    bool EnumerateCSGRecursive(JoinRelationSet &node, unordered_set<idx_t> &exclusion_set);

    //! Cmp 枚举
    bool EnumerateCmpRecursive(JoinRelationSet &left, JoinRelationSet &right,
                               unordered_set<idx_t> &exclusion_set);

    //! 发射候选对
    DPJoinNode &EmitPair(JoinRelationSet &left, JoinRelationSet &right,
                         const vector<reference<NeighborInfo>> &info);

    //! DP 表：关系集合 -> 最优计划
    reference_map_t<JoinRelationSet, unique_ptr<DPJoinNode>> plans;

    //! 已检查的候选对数量
    idx_t pairs = 0;
};
```

### 3.7.2 初始化叶子计划

```cpp
void PlanEnumerator::InitLeafPlans() {
    auto relation_stats = query_graph_manager.relation_manager.GetRelationStats();

    // 初始化基数估算器的等价关系
    cost_model.cardinality_estimator.InitEquivalentRelations(
        query_graph_manager.GetFilterBindings());

    // 为每个单表创建叶子计划
    for (idx_t i = 0; i < relation_stats.size(); i++) {
        auto stats = relation_stats.at(i);
        auto &relation_set = query_graph_manager.set_manager.GetJoinRelation(i);

        // 创建叶子节点
        auto join_node = make_uniq<DPJoinNode>(relation_set);
        join_node->cost = 0;  // 叶子节点代价为 0
        join_node->cardinality = stats.cardinality;

        // 存入 DP 表
        plans[relation_set] = std::move(join_node);

        // 初始化基数估算器属性
        cost_model.cardinality_estimator.InitCardinalityEstimatorProps(&relation_set, stats);
    }
}
```

### 3.7.3 主求解流程

```cpp
void PlanEnumerator::SolveJoinOrder() {
    // 根据关系数量选择算法
    if (query_graph_manager.relation_manager.NumRelations() >= THRESHOLD_TO_SWAP_TO_APPROXIMATE) {
        // 超过 12 个关系，直接使用贪心算法
        SolveJoinOrderApproximately();
    } else if (!SolveJoinOrderExactly()) {
        // 精确算法超时，切换到贪心
        SolveJoinOrderApproximately();
    }

    // 检查是否找到完整计划
    unordered_set<idx_t> bindings;
    for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
        bindings.insert(i);
    }
    auto &total_relation = query_graph_manager.set_manager.GetJoinRelation(bindings);
    auto final_plan = plans.find(total_relation);

    if (final_plan == plans.end()) {
        // 未找到完整计划，需要添加 Cross Product
        GenerateCrossProducts();
        return SolveJoinOrder();  // 递归求解
    }
}
```

### 3.7.4 精确动态规划

```cpp
bool PlanEnumerator::SolveJoinOrderExactly() {
    // 从大到小遍历所有关系
    for (idx_t i = query_graph_manager.relation_manager.NumRelations(); i > 0; i--) {
        // 以每个节点作为起始节点
        auto &start_node = query_graph_manager.set_manager.GetJoinRelation(i - 1);

        // 发射 CSG
        if (!EmitCSG(start_node)) {
            return false;  // 超时
        }

        // 初始化排除集合（所有编号小于当前节点的关系）
        unordered_set<idx_t> exclusion_set;
        for (idx_t j = 0; j < i; j++) {
            exclusion_set.insert(j);
        }

        // 递归枚举 CSG
        if (!EnumerateCSGRecursive(start_node, exclusion_set)) {
            return false;
        }
    }
    return true;
}
```

### 3.7.5 CSG 枚举

```cpp
bool PlanEnumerator::EmitCSG(JoinRelationSet &node) {
    if (node.count == query_graph_manager.relation_manager.NumRelations()) {
        return true;  // 已包含所有关系
    }

    // 创建排除集合
    unordered_set<idx_t> exclusion_set;
    for (idx_t i = 0; i < node.relations[0]; i++) {
        exclusion_set.insert(i);
    }
    UpdateExclusionSet(&node, exclusion_set);

    // 获取邻居
    auto neighbors = query_graph.GetNeighbors(node, exclusion_set);
    if (neighbors.empty()) {
        return true;
    }

    // 按降序遍历邻居
    std::sort(neighbors.begin(), neighbors.end(), std::greater<idx_t>());

    for (auto neighbor : neighbors) {
        auto &neighbor_relation = query_graph_manager.set_manager.GetJoinRelation(neighbor);
        auto connections = query_graph.GetConnections(node, neighbor_relation);

        if (!connections.empty()) {
            // 尝试发射候选对
            if (!TryEmitPair(node, neighbor_relation, connections)) {
                return false;  // 超时
            }
        }

        // 递归枚举 Cmp
        if (!EnumerateCmpRecursive(node, neighbor_relation, new_exclusion_set)) {
            return false;
        }
    }
    return true;
}
```

### 3.7.6 候选对发射

```cpp
DPJoinNode &PlanEnumerator::EmitPair(JoinRelationSet &left, JoinRelationSet &right,
                                      const vector<reference<NeighborInfo>> &info) {
    // 获取左右计划
    auto left_plan = plans.find(left);
    auto right_plan = plans.find(right);

    // 合并关系集合
    auto &new_set = query_graph_manager.set_manager.Union(left, right);

    // 创建新的 Join 计划
    auto new_plan = CreateJoinTree(new_set, info, *left_plan->second, *right_plan->second);

    // 检查是否比现有计划更优
    auto entry = plans.find(new_set);
    if (entry == plans.end() || new_plan->cost < entry->second->cost) {
        // 更新 DP 表
        plans[new_set] = std::move(new_plan);
    }

    return *plans[new_set];
}

bool PlanEnumerator::TryEmitPair(JoinRelationSet &left, JoinRelationSet &right,
                                  const vector<reference<NeighborInfo>> &info) {
    pairs++;

    // 超过 10000 对时切换到贪心
    if (pairs >= 10000) {
        return false;
    }

    EmitPair(left, right, info);
    return true;
}
```

### 3.7.7 贪心近似算法

```cpp
void PlanEnumerator::SolveJoinOrderApproximately() {
    // 初始化：所有基表
    vector<reference<JoinRelationSet>> join_relations;
    for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
        join_relations.push_back(query_graph_manager.set_manager.GetJoinRelation(i));
    }

    // 贪心合并：每次选择代价最小的配对
    while (join_relations.size() > 1) {
        idx_t best_left = 0, best_right = 0;
        optional_ptr<DPJoinNode> best_connection;

        // O(n^2) 搜索最优配对
        for (idx_t i = 0; i < join_relations.size(); i++) {
            for (idx_t j = i + 1; j < join_relations.size(); j++) {
                auto connection = query_graph.GetConnections(join_relations[i],
                                                             join_relations[j]);
                if (!connection.empty()) {
                    auto node = EmitPair(join_relations[i], join_relations[j], connection);
                    if (!best_connection || node.cost < best_connection->cost) {
                        best_connection = &node;
                        best_left = i;
                        best_right = j;
                    }
                }
            }
        }

        if (!best_connection) {
            // 没有连接，需要添加 Cross Product
            // 选择基数最小的两个关系
            // ...
        }

        // 更新关系列表
        auto &new_set = query_graph_manager.set_manager.Union(
            join_relations[best_left], join_relations[best_right]);
        join_relations.erase(join_relations.begin() + best_right);
        join_relations.erase(join_relations.begin() + best_left);
        join_relations.push_back(new_set);
    }
}
```

```
贪心算法示例：

关系: A(100行), B(1000行), C(10000行), D(50行)
边: A-B, B-C, C-D

第1轮:
  - A⨝B 代价: 100*1000/... = X1
  - B⨝C 代价: 1000*10000/... = X2
  - C⨝D 代价: 10000*50/... = X3
  选择最小代价的配对

第2轮:
  继续合并直到只剩一个关系集合
```

---

## 3.8 DPJoinNode：计划节点

### 3.8.1 数据结构

```cpp
// src/include/duckdb/optimizer/join_order/join_node.hpp

class DPJoinNode {
public:
    //! 该节点覆盖的关系集合
    JoinRelationSet &set;

    //! 连接信息（过滤条件）
    optional_ptr<NeighborInfo> info;

    //! 是否为叶子节点
    bool is_leaf;

    //! 左右子计划的关系集合
    JoinRelationSet &left_set;
    JoinRelationSet &right_set;

    //! 代价（累计）
    double cost;

    //! 估计基数
    idx_t cardinality;

    //! 创建中间节点
    DPJoinNode(JoinRelationSet &set, optional_ptr<NeighborInfo> info,
               JoinRelationSet &left, JoinRelationSet &right, double cost);

    //! 创建叶子节点
    explicit DPJoinNode(JoinRelationSet &set);
};
```

### 3.8.2 计划树结构

```
DP 表存储示例：

plans = {
  [0]        -> DPJoinNode(leaf, A, cardinality=100)
  [1]        -> DPJoinNode(leaf, B, cardinality=1000)
  [2]        -> DPJoinNode(leaf, C, cardinality=10000)
  [0,1]      -> DPJoinNode(join, [0]⨝[1], cost=150)
  [1,2]      -> DPJoinNode(join, [1]⨝[2], cost=1500)
  [0,1,2]    -> DPJoinNode(join, [0,1]⨝[2], cost=1650)
}

对应的 Join 树：

         ⨝ [0,1,2]
        / \
       ⨝   C
      / \
     A   B
```

---

## 3.9 CostModel：代价模型

### 3.9.1 代价计算

```cpp
// src/optimizer/join_order/cost_model.cpp

double CostModel::ComputeCost(DPJoinNode &left, DPJoinNode &right) {
    // 合并关系集合
    auto &combination = query_graph_manager.set_manager.Union(left.set, right.set);

    // 估算 Join 基数
    auto join_card = cardinality_estimator.EstimateCardinalityWithSet<double>(combination);

    // 代价 = Join 结果基数 + 左子树代价 + 右子树代价
    auto join_cost = join_card;
    return join_cost + left.cost + right.cost;
}
```

### 3.9.2 代价模型分析

当前 DuckDB 使用简单的代价模型：

$$Cost(Plan) = Cardinality(Join) + Cost(Left) + Cost(Right)$$

这是一个简化模型，假设：
- 所有 Join 算法的代价与结果基数成正比
- 不考虑 Hash Join vs Merge Join vs Nested Loop 的差异
- 不考虑 Build/Probe 端选择

```
代价计算示例：

A(100) ⨝ B(1000) ⨝ C(10000)

计划1: (A ⨝ B) ⨝ C
  - A ⨝ B: 假设选择率 10%, 结果 = 10,000
  - (A⨝B) ⨝ C: 假设选择率 1%, 结果 = 1,000,000
  - 总代价 = 10,000 + 1,000,000 = 1,010,000

计划2: A ⨝ (B ⨝ C)
  - B ⨝ C: 假设选择率 10%, 结果 = 1,000,000
  - A ⨝ (B⨝C): 假设选择率 1%, 结果 = 1,000,000
  - 总代价 = 1,000,000 + 1,000,000 = 2,000,000

选择计划1
```

---

## 3.10 CardinalityEstimator：基数估算

### 3.10.1 估算原理

基数估算基于以下公式（来自 Tom Ebergen 的硕士论文）：

$$Cardinality = \frac{\prod_{R \in Relations} |R|}{\prod_{J \in JoinConditions} max(distinct(J.left), distinct(J.right))}$$

简单来说：
- **分子**：所有参与表的基数乘积
- **分母**：所有 Join 条件的最大 Distinct 值乘积

### 3.10.2 等价关系管理

```cpp
// src/optimizer/join_order/cardinality_estimator.cpp

struct RelationsSetToStats {
    //! 等价的列绑定集合
    //! 例如: A.x = B.y AND B.y = C.z => {A.x, B.y, C.z}
    column_binding_set_t equivalent_relations;

    //! 使用 HLL 估算的 Distinct Count
    idx_t distinct_count_hll;

    //! 不使用 HLL 的 Distinct Count
    idx_t distinct_count_no_hll;

    bool has_distinct_count_hll;

    //! 关联的过滤条件
    vector<optional_ptr<FilterInfo>> filters;
};

void CardinalityEstimator::InitEquivalentRelations(
    const vector<unique_ptr<FilterInfo>> &filter_infos) {

    for (auto &filter : filter_infos) {
        if (SingleColumnFilter(*filter)) {
            // 单列过滤，添加到统计
            AddRelationStats(*filter);
        } else if (EmptyFilter(*filter)) {
            continue;
        } else {
            // Join 条件，建立等价关系
            auto matching_sets = DetermineMatchingEquivalentSets(filter.get());
            AddToEquivalenceSets(filter.get(), matching_sets);
        }
    }
}
```

### 3.10.3 基数估算实现

```cpp
template <>
double CardinalityEstimator::EstimateCardinalityWithSet(JoinRelationSet &new_set) {
    // 缓存检查
    if (relation_set_2_cardinality.find(new_set.ToString()) != relation_set_2_cardinality.end()) {
        return relation_set_2_cardinality[new_set.ToString()].cardinality_before_filters;
    }

    // 计算分母
    auto denom = GetDenominator(new_set);

    // 计算分子（只包含 numerator_relations，排除 Semi/Anti Join 右侧）
    auto numerator = GetNumerator(denom.numerator_relations);

    // 基数 = 分子 / 分母
    double result = numerator / denom.denominator;

    // 缓存结果
    relation_set_2_cardinality[new_set.ToString()] = CardinalityHelper(result);

    return result;
}
```

### 3.10.4 分母计算

```cpp
DenomInfo CardinalityEstimator::GetDenominator(JoinRelationSet &set) {
    vector<Subgraph2Denominator> subgraphs;

    // 获取所有相关的边（按 distinct count 降序）
    auto edges = GetEdges(relation_set_stats, set);

    for (auto &edge : edges) {
        auto subgraph_connections = SubgraphsConnectedByEdge(edge, subgraphs);

        if (subgraph_connections.empty()) {
            // 创建新子图
            auto left_subgraph = Subgraph2Denominator();
            auto right_subgraph = Subgraph2Denominator();
            // ... 初始化并合并
        } else if (subgraph_connections.size() == 1) {
            // 扩展现有子图
            // ...
        } else if (subgraph_connections.size() == 2) {
            // 合并两个子图
            // ...
        }
    }

    return DenomInfo(*subgraphs[0].numerator_relations, 1, subgraphs[0].denom);
}
```

### 3.10.5 不同比较类型的处理

```cpp
double CardinalityEstimator::CalculateUpdatedDenom(Subgraph2Denominator left,
                                                   Subgraph2Denominator right,
                                                   FilterInfoWithTotalDomains &filter) {
    double new_denom = left.denom * right.denom;

    switch (filter.filter_info->join_type) {
    case JoinType::INNER: {
        ExpressionType comparison_type = /* 从表达式提取 */;

        double extra_ratio = 1;
        switch (comparison_type) {
        case ExpressionType::COMPARE_EQUAL:
            // 等值连接：使用 distinct count
            extra_ratio = filter.distinct_count;
            break;
        case ExpressionType::COMPARE_LESSTHAN:
        case ExpressionType::COMPARE_GREATERTHAN:
            // 范围连接：使用 distinct count 的 2/3 次方作为惩罚
            extra_ratio = pow(filter.distinct_count, 2.0 / 3.0);
            break;
        }
        new_denom *= extra_ratio;
        break;
    }
    case JoinType::SEMI:
    case JoinType::ANTI:
        // Semi/Anti Join：使用固定选择率 (1/5)
        new_denom = left.denom * DEFAULT_SEMI_ANTI_SELECTIVITY;
        break;
    }

    return new_denom;
}
```

```
基数估算示例：

表 A: 1000 行, A.x distinct = 100
表 B: 5000 行, B.y distinct = 200
表 C: 10000 行, C.z distinct = 50

查询: A JOIN B ON A.x = B.y JOIN C ON B.y = C.z

等价集合: {A.x, B.y, C.z}
合并后的 distinct count = max(100, 200, 50) = 200

估算基数:
  numerator = 1000 * 5000 * 10000 = 50,000,000,000
  denominator = 200 * 200 = 40,000  (两个 Join 条件)
  cardinality = 50,000,000,000 / 40,000 = 1,250,000
```

---

## 3.11 计划重建

### 3.11.1 重建流程

```cpp
unique_ptr<LogicalOperator> QueryGraphManager::Reconstruct(unique_ptr<LogicalOperator> plan) {
    // 收集所有关系
    unordered_set<idx_t> bindings;
    for (idx_t i = 0; i < relation_manager.NumRelations(); i++) {
        bindings.insert(i);
    }
    auto &total_relation = set_manager.GetJoinRelation(bindings);

    // 提取所有关系算子
    vector<unique_ptr<LogicalOperator>> extracted_relations;
    for (auto &relation : relation_manager.GetRelations()) {
        extracted_relations.push_back(ExtractJoinRelation(relation));
    }

    // 递归生成 Join 树
    auto join_tree = GenerateJoins(extracted_relations, total_relation);

    // 推送剩余过滤条件
    for (auto &filter : filters_and_bindings) {
        if (filter->filter) {
            join_tree.op = PushFilter(std::move(join_tree.op), std::move(filter->filter));
        }
    }

    return std::move(join_tree.op);
}
```

### 3.11.2 递归生成 Join

```cpp
GenerateJoinRelation QueryGraphManager::GenerateJoins(
    vector<unique_ptr<LogicalOperator>> &extracted_relations,
    JoinRelationSet &set) {

    auto dp_entry = plans->find(set);
    auto &node = dp_entry->second;

    if (!node->is_leaf) {
        // 递归生成左右子树
        auto left = GenerateJoins(extracted_relations, node->left_set);
        auto right = GenerateJoins(extracted_relations, node->right_set);

        if (node->info->filters.empty()) {
            // 无过滤条件，创建 Cross Product
            result_operator = LogicalCrossProduct::Create(std::move(left.op),
                                                          std::move(right.op));
        } else {
            // 有过滤条件，创建 Comparison Join
            auto join = make_uniq<LogicalComparisonJoin>(chosen_filter->join_type);
            join->children.push_back(std::move(left.op));
            join->children.push_back(std::move(right.op));

            // 设置 Join 条件
            for (auto &filter_ref : node->info->filters) {
                auto condition = std::move(filters_and_bindings[f->filter_index]->filter);
                // 可能需要反转条件
                bool invert = !JoinRelationSet::IsSubset(*left.set, *f->left_set);
                auto cond = MaybeInvertConditions(std::move(condition), invert);
                join->conditions.push_back(std::move(cond));
            }

            result_operator = std::move(join);
        }
    } else {
        // 叶子节点，直接返回关系算子
        result_operator = std::move(extracted_relations[node->set.relations[0]]);
    }

    // 设置估计基数
    result_operator->estimated_cardinality = node->cardinality;

    return GenerateJoinRelation(result_relation, std::move(result_operator));
}
```

---

## 3.12 Cross Product 处理

### 3.12.1 何时需要 Cross Product

当查询图不连通时，需要添加 Cross Product：

```sql
-- 不连通的查询图
SELECT * FROM A, B, C, D
WHERE A.x = B.y AND C.z = D.w
-- A-B 和 C-D 是两个独立的连通分量
```

### 3.12.2 生成 Cross Product 边

```cpp
void PlanEnumerator::GenerateCrossProducts() {
    // 在所有关系对之间创建 Cross Product 边
    for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
        auto &left = query_graph_manager.set_manager.GetJoinRelation(i);
        for (idx_t j = 0; j < query_graph_manager.relation_manager.NumRelations(); j++) {
            // 检查是否允许 Cross Product（Semi/Anti Join 右侧不允许）
            auto cross_product_allowed =
                query_graph_manager.relation_manager.CrossProductWithRelationAllowed(i) &&
                query_graph_manager.relation_manager.CrossProductWithRelationAllowed(j);

            if (i != j && cross_product_allowed) {
                auto &right = query_graph_manager.set_manager.GetJoinRelation(j);
                query_graph_manager.CreateQueryGraphCrossProduct(left, right);
            }
        }
    }
}
```

---

## 3.13 算法复杂度分析

### 3.13.1 时间复杂度

| 算法 | 时间复杂度 | 适用场景 |
|------|-----------|---------|
| DPhyp 精确 | O(3^n) | n ≤ 12 |
| DPhyp 近似 | O(n^3) | n > 12 或超时 |

### 3.13.2 空间复杂度

| 组件 | 空间复杂度 |
|------|-----------|
| DP 表 | O(2^n) |
| 查询图 | O(n^2) |
| 关系集合 | O(2^n) |

### 3.13.3 切换策略

```
关系数量与算法选择：

n ≤ 11:  精确 DP，保证最优
n = 12:  边界情况，精确 DP 但可能超时
n ≥ 13:  直接使用贪心近似

超时检测：pairs ≥ 10000 时切换到贪心
```

---

## 3.14 实际优化示例

### 3.14.1 TPC-H Query 5

```sql
SELECT n_name, SUM(l_extendedprice * (1 - l_discount)) as revenue
FROM customer, orders, lineitem, supplier, nation, region
WHERE c_custkey = o_custkey
  AND l_orderkey = o_orderkey
  AND l_suppkey = s_suppkey
  AND c_nationkey = s_nationkey
  AND s_nationkey = n_nationkey
  AND n_regionkey = r_regionkey
  AND r_name = 'ASIA'
GROUP BY n_name
ORDER BY revenue DESC;
```

**查询图：**
```
     region ── nation ── supplier
                │            │
                └── customer ─┘
                       │
                    orders
                       │
                   lineitem
```

**优化后的 Join 顺序（假设）：**
```
              ⨝
             / \
         region  ⨝
                / \
           nation  ⨝
                  / \
             supplier ⨝
                     / \
               customer ⨝
                       / \
                   orders lineitem
```

### 3.14.2 Star Schema 优化

星型模式查询：
```sql
SELECT d.year, p.category, SUM(f.amount)
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_store s ON f.store_key = s.store_key
WHERE d.year = 2023 AND s.region = 'EAST'
GROUP BY d.year, p.category;
```

**优化策略：**
1. 将选择性高的维度表（dim_date, dim_store）先 Join
2. 利用过滤条件减少中间结果
3. 最后 Join 大的事实表

---

## 3.15 源文件索引

| 组件 | 文件路径 |
|------|---------|
| JoinOrderOptimizer | `src/optimizer/join_order/join_order_optimizer.cpp` |
| QueryGraphManager | `src/optimizer/join_order/query_graph_manager.cpp` |
| RelationManager | `src/optimizer/join_order/relation_manager.cpp` |
| PlanEnumerator | `src/optimizer/join_order/plan_enumerator.cpp` |
| CostModel | `src/optimizer/join_order/cost_model.cpp` |
| CardinalityEstimator | `src/optimizer/join_order/cardinality_estimator.cpp` |
| QueryGraphEdges | `src/optimizer/join_order/query_graph.cpp` |
| JoinRelationSet | `src/optimizer/join_order/join_relation_set.cpp` |
| DPJoinNode | `src/optimizer/join_order/join_node.cpp` |
| RelationStatisticsHelper | `src/optimizer/join_order/relation_statistics_helper.cpp` |

---

## 3.16 本章小结

本章详细介绍了 DuckDB 的 Join Order 优化系统：

1. **系统架构**：JoinOrderOptimizer 作为入口，协调 RelationManager、QueryGraphManager、CostModel、PlanEnumerator 等组件

2. **关系提取**：RelationManager 从逻辑计划中提取可重排序的关系，识别 Inner/Semi/Anti Join 为可重排序，Left/Right/Outer Join 为不可重排序

3. **查询图构建**：QueryGraphManager 将 Join 条件转换为超图边，使用 JoinRelationSet 表示关系集合

4. **DPhyp 算法**：
   - 对于小规模查询（≤12 表），使用精确动态规划
   - 对于大规模查询或超时情况，使用 O(n^3) 贪心近似
   - 通过 CSG 和 Cmp 枚举高效搜索计划空间

5. **代价模型**：基于基数估算的简单代价模型，使用分子/分母公式估算 Join 基数

6. **计划重建**：根据最优计划递归生成 LogicalOperator 树，处理条件反转和 Cross Product

关键设计特点：
- **渐进式优化**：超时时自动切换到贪心算法
- **等价关系追踪**：通过传递闭包优化基数估算
- **Semi/Anti Join 特殊处理**：防止非法重排序
- **Cross Product 惰性添加**：仅在需要时添加

下一章将介绍聚合和 Distinct 优化。

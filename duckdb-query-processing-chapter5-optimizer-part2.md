# DuckDB 查询处理深度解析（五）：优化器（下）- Join 顺序优化与统计信息

## 前言

在上一章中，我们深入分析了 DuckDB 优化器的表达式重写和谓词优化机制。本章作为优化器下篇，将重点讲解更高级的优化技术：

1. **Join 顺序优化器**：JoinOrderOptimizer 架构和 DPhyp 算法
2. **查询图（Query Graph）**：关系和连接边的建模
3. **基数估计（Cardinality Estimation）**：使用 HLL 和等价集的估计方法
4. **代价模型（Cost Model）**：Join 代价计算
5. **计划枚举（Plan Enumerator）**：动态规划和贪婪算法
6. **统计信息传播**：StatisticsPropagator 的设计
7. **其他高级优化**：Join 消除、TopN 优化、Build 侧选择等

---

## 1. Join 顺序优化器架构

### 1.1 JoinOrderOptimizer 类设计

Join 顺序优化是查询优化中最复杂也最重要的部分。DuckDB 实现了基于动态规划的 Join 顺序优化器：

```cpp
// src/include/duckdb/optimizer/join_order/join_order_optimizer.hpp

class JoinOrderOptimizer {
public:
    explicit JoinOrderOptimizer(ClientContext &context);
    JoinOrderOptimizer CreateChildOptimizer();

    //! 执行 Join 重排序优化
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan,
                                          optional_ptr<RelationStats> stats = nullptr);

    //! 物化 CTE 统计信息管理
    void AddMaterializedCTEStats(idx_t index, RelationStats &&stats);
    RelationStats GetMaterializedCTEStats(idx_t index);

    //! DelimScan 统计信息管理
    void AddDelimScanStats(RelationStats &stats);
    RelationStats GetDelimScanStats();

private:
    ClientContext &context;

    //! 管理查询图、关系和边
    QueryGraphManager query_graph_manager;

    //! 从查询图提取的过滤器
    vector<unique_ptr<Expression>> filters;

    //! 过滤器信息
    vector<unique_ptr<FilterInfo>> filter_infos;

    //! 等价集映射（用于推导隐含的 Join 边）
    //! 例如: A=B AND B=C 中，B 的等价集是 {A, C}，可以推导出 A=C
    expression_map_t<vector<FilterInfo *>> equivalence_sets;

    //! 基数估计器
    CardinalityEstimator cardinality_estimator;

    //! 物化 CTE 统计信息映射
    unordered_map<idx_t, RelationStats> materialized_cte_stats;

    //! 当前正在优化的 DelimJoin 的 DelimScan 统计信息
    optional_ptr<RelationStats> delim_scan_stats;

    //! 递归深度
    idx_t depth;

public:
    //! 递归 CTE 索引
    unordered_set<idx_t> recursive_cte_indexes;
};
```

### 1.2 优化流程概览

```cpp
unique_ptr<LogicalOperator> JoinOrderOptimizer::Optimize(
    unique_ptr<LogicalOperator> plan,
    optional_ptr<RelationStats> stats) {

    // 防止过深的递归（栈空间限制）
    if (depth > query_graph_manager.context.config.max_expression_depth) {
        return plan;
    }

    LogicalOperator *op = plan.get();

    // 第一步：构建查询图
    // 提取可重排序的关系，对不可重排序的操作递归优化其子节点
    bool reorderable = query_graph_manager.Build(*this, *op);

    // 获取关系统计信息
    auto relation_stats = query_graph_manager.relation_manager.GetRelationStats();
    unique_ptr<LogicalOperator> new_logical_plan = nullptr;

    if (reorderable) {
        // 查询图现在包含了过滤器和关系

        // 第二步：初始化代价模型
        auto cost_model = CostModel(query_graph_manager);

        // 第三步：初始化计划枚举器
        auto plan_enumerator = PlanEnumerator(
            query_graph_manager, cost_model,
            query_graph_manager.GetQueryGraphEdges());

        // 第四步：初始化叶节点计划（单表计划）
        plan_enumerator.InitLeafPlans();

        // 第五步：求解最优 Join 顺序
        plan_enumerator.SolveJoinOrder();

        // 第六步：从查询图计划重建逻辑计划
        query_graph_manager.plans = &plan_enumerator.GetPlans();
        new_logical_plan = query_graph_manager.Reconstruct(std::move(plan));
    } else {
        new_logical_plan = std::move(plan);
        if (relation_stats.size() == 1) {
            new_logical_plan->estimated_cardinality = relation_stats.at(0).cardinality;
            new_logical_plan->has_estimated_cardinality = true;
        }
    }

    // 传播统计信息
    if (stats) {
        auto cardinality = new_logical_plan->EstimateCardinality(context);
        auto bindings = new_logical_plan->GetColumnBindings();
        auto new_stats = RelationStatisticsHelper::CombineStatsOfReorderableOperator(
            bindings, relation_stats);
        new_stats.cardinality = cardinality;
        RelationStatisticsHelper::CopyRelationStats(*stats, new_stats);
    } else {
        new_logical_plan->EstimateCardinality(context);
    }

    return new_logical_plan;
}
```

### 1.3 优化流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Join Order Optimization Pipeline                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入: LogicalOperator Tree                                             │
│         │                                                               │
│         ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │          Step 1: Build Query Graph                               │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │ • 提取可重排序的关系（表、子查询等）                      │    │   │
│  │  │ • 提取 Join 条件和过滤器                                  │    │   │
│  │  │ • 构建关系之间的边                                        │    │   │
│  │  │ • 推导等价类和隐含边                                      │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │          Step 2: Initialize Cost Model                          │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │ • 初始化基数估计器                                        │    │   │
│  │  │ • 计算各列的 distinct count（使用 HLL）                  │    │   │
│  │  │ • 建立等价集合（用于 Join 选择性估计）                    │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │          Step 3: Initialize Leaf Plans                          │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │ • 为每个单表创建 DPJoinNode                               │    │   │
│  │  │ • 设置叶节点代价为 0                                      │    │   │
│  │  │ • 记录基数信息                                            │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │          Step 4: Solve Join Order                               │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │ if (关系数 >= 12) {                                      │    │   │
│  │  │     SolveJoinOrderApproximately()  // 贪婪算法           │    │   │
│  │  │ } else if (!SolveJoinOrderExactly()) {                   │    │   │
│  │  │     // 动态规划超时                                       │    │   │
│  │  │     SolveJoinOrderApproximately()                        │    │   │
│  │  │ }                                                        │    │   │
│  │  │ // 处理不连通的图（需要笛卡尔积）                         │    │   │
│  │  │ if (!found_final_plan) GenerateCrossProducts()           │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │          Step 5: Reconstruct Logical Plan                       │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │ • 从 DP 表中获取最优计划                                  │    │   │
│  │  │ • 递归构建 LogicalJoin 树                                 │    │   │
│  │  │ • 设置估计的基数                                          │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  输出: Optimized LogicalOperator Tree with Optimal Join Order           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 查询图（Query Graph）

### 2.1 QueryGraphEdges 设计

查询图用于表示关系之间的连接关系：

```cpp
// src/include/duckdb/optimizer/join_order/query_graph.hpp

//! 邻居信息：表示一个关系的邻接关系
struct NeighborInfo {
    explicit NeighborInfo(optional_ptr<JoinRelationSet> neighbor)
        : neighbor(neighbor) {}

    optional_ptr<JoinRelationSet> neighbor;  // 邻居关系集
    vector<optional_ptr<FilterInfo>> filters;  // 连接这两个关系的过滤器
};

//! 查询图边：表示关系之间的连接
class QueryGraphEdges {
public:
    //! 查询边节点结构
    struct QueryEdge {
        vector<unique_ptr<NeighborInfo>> neighbors;  // 邻居列表
        unordered_map<idx_t, unique_ptr<QueryEdge>> children;  // 子边
    };

public:
    //! 获取连接两个关系集的所有边
    const vector<reference<NeighborInfo>> GetConnections(
        JoinRelationSet &node, JoinRelationSet &other) const;

    //! 获取一个节点的所有邻居（排除 exclusion_set 中的）
    const vector<idx_t> GetNeighbors(
        JoinRelationSet &node, unordered_set<idx_t> &exclusion_set) const;

    //! 遍历一个节点的所有邻居
    void EnumerateNeighbors(JoinRelationSet &node,
                            const std::function<bool(NeighborInfo &)> &callback) const;

    //! 创建边
    void CreateEdge(JoinRelationSet &left, JoinRelationSet &right,
                    optional_ptr<FilterInfo> info);

private:
    QueryEdge root;  // 根边
};
```

### 2.2 JoinRelationSet：关系集合表示

```cpp
// src/include/duckdb/optimizer/join_order/join_relation.hpp

//! JoinRelationSet 表示参与 Join 的关系集合
class JoinRelationSet {
public:
    JoinRelationSet(unsafe_unique_array<idx_t> relations, idx_t count)
        : relations(std::move(relations)), count(count) {}

    //! 关系 ID 数组
    unsafe_unique_array<idx_t> relations;
    //! 关系数量
    idx_t count;

    //! 检查一个集合是否是另一个的子集
    static bool IsSubset(JoinRelationSet &super, JoinRelationSet &sub);

    string ToString() const;
};

//! 管理 JoinRelationSet 的创建和缓存
class JoinRelationSetManager {
public:
    //! 获取单个关系的 JoinRelationSet
    JoinRelationSet &GetJoinRelation(idx_t index);

    //! 获取多个关系的 JoinRelationSet
    JoinRelationSet &GetJoinRelation(const unordered_set<idx_t> &bindings);

    //! 合并两个 JoinRelationSet
    JoinRelationSet &Union(JoinRelationSet &left, JoinRelationSet &right);
};
```

### 2.3 查询图构建示例

考虑以下查询：

```sql
SELECT *
FROM A, B, C, D
WHERE A.x = B.y
  AND B.y = C.z
  AND C.z = D.w
```

查询图表示：

```
        A ──── B ──── C ──── D
         (x=y)  (y=z)  (z=w)

Relations: {A, B, C, D}
Edges:
  - (A, B): filter A.x = B.y
  - (B, C): filter B.y = C.z
  - (C, D): filter C.z = D.w

Equivalence Sets:
  - {A.x, B.y, C.z, D.w} (通过传递性推导)
```

---

## 3. 基数估计（Cardinality Estimation）

### 3.1 CardinalityEstimator 设计

DuckDB 使用一种基于等价集和 distinct count 的基数估计方法：

```cpp
// src/include/duckdb/optimizer/join_order/cardinality_estimator.hpp

//! 等价关系的统计信息
struct RelationsSetToStats {
    //! 等价的列绑定集合
    //! 例如: A.x = B.y AND B.y = C.z → {A.x, B.y, C.z}
    column_binding_set_t equivalent_relations;

    //! 使用 HLL 估计的 distinct count
    idx_t distinct_count_hll;

    //! 不使用 HLL 的 distinct count 估计
    idx_t distinct_count_no_hll;

    //! 是否有 HLL 统计
    bool has_distinct_count_hll;

    //! 相关的过滤器
    vector<optional_ptr<FilterInfo>> filters;
};

class CardinalityEstimator {
public:
    static constexpr double DEFAULT_SEMI_ANTI_SELECTIVITY = 5;

    //! 初始化等价关系
    void InitEquivalentRelations(const vector<unique_ptr<FilterInfo>> &filter_infos);

    //! 初始化基数估计器属性
    void InitCardinalityEstimatorProps(optional_ptr<JoinRelationSet> set,
                                        RelationStats &stats);

    //! 估计 Join 结果的基数
    template <class T>
    T EstimateCardinalityWithSet(JoinRelationSet &new_set);

private:
    //! 等价集统计信息
    vector<RelationsSetToStats> relation_set_stats;

    //! 关系集到基数的映射（缓存）
    unordered_map<string, CardinalityHelper> relation_set_2_cardinality;

    //! 关系集管理器
    JoinRelationSetManager set_manager;

    //! 各关系的统计信息
    vector<RelationStats> relation_stats;

    //! 获取分子（各表基数的乘积）
    double GetNumerator(JoinRelationSet &set);

    //! 获取分母（distinct count 的积）
    DenomInfo GetDenominator(JoinRelationSet &set);

    //! 计算更新后的分母
    double CalculateUpdatedDenom(Subgraph2Denominator left,
                                  Subgraph2Denominator right,
                                  FilterInfoWithTotalDomains &filter);
};
```

### 3.2 基数估计公式

DuckDB 的基数估计基于以下论文中的方法：

> "Join Order Optimization with Almost No Statistics"
> - Tom Ebergen, MSc Thesis

**核心公式：**

对于两个表 A 和 B 通过 `A.x = B.y` 进行 Join：

```
Cardinality(A ⋈ B) = |A| × |B| / max(distinct(A.x), distinct(B.y))
```

**对于多表 Join：**

```
                   |A| × |B| × |C| × ...
Cardinality = ────────────────────────────────────
               distinct(col1) × distinct(col2) × ...
```

**实现代码：**

```cpp
template <>
double CardinalityEstimator::EstimateCardinalityWithSet(JoinRelationSet &new_set) {
    // 检查缓存
    if (relation_set_2_cardinality.find(new_set.ToString()) !=
        relation_set_2_cardinality.end()) {
        return relation_set_2_cardinality[new_set.ToString()].cardinality_before_filters;
    }

    // 计算分母
    auto denom = GetDenominator(new_set);

    // 计算分子（各表基数的乘积）
    // 对于 SEMI/ANTI Join，只包含左侧关系的基数
    auto numerator = GetNumerator(denom.numerator_relations);

    // 基数 = 分子 / 分母
    double result = numerator / denom.denominator;

    // 缓存结果
    auto new_entry = CardinalityHelper(result);
    relation_set_2_cardinality[new_set.ToString()] = new_entry;

    return result;
}
```

### 3.3 分母计算（GetDenominator）

分母计算是基数估计的核心，需要处理多种 Join 类型：

```cpp
DenomInfo CardinalityEstimator::GetDenominator(JoinRelationSet &set) {
    vector<Subgraph2Denominator> subgraphs;

    // 获取连接这些关系的所有边
    auto edges = GetEdges(relation_set_stats, set);

    for (auto &edge : edges) {
        // 检查当前子图是否已经连接了所有关系
        if (subgraphs.size() == 1 &&
            subgraphs.at(0).relations->ToString() == set.ToString()) {
            continue;  // 跳过多余的边
        }

        // 查找这条边连接的子图
        auto subgraph_connections = SubgraphsConnectedByEdge(edge, subgraphs);

        if (subgraph_connections.empty()) {
            // 创建新子图
            auto left_subgraph = Subgraph2Denominator();
            auto right_subgraph = Subgraph2Denominator();
            left_subgraph.relations = edge.filter_info->left_set;
            right_subgraph.relations = edge.filter_info->right_set;
            left_subgraph.denom = CalculateUpdatedDenom(left_subgraph, right_subgraph, edge);
            subgraphs.push_back(left_subgraph);
        }
        else if (subgraph_connections.size() == 1) {
            // 将新关系添加到现有子图
            auto left_subgraph = &subgraphs.at(subgraph_connections.at(0));
            // ... 更新子图
        }
        else if (subgraph_connections.size() == 2) {
            // 合并两个子图
            auto subgraph_to_merge_into = &subgraphs.at(subgraph_connections.at(0));
            auto subgraph_to_delete = &subgraphs.at(subgraph_connections.at(1));
            // ... 合并逻辑
        }
    }

    // 返回分母信息
    return DenomInfo(*subgraphs.at(0).numerator_relations,
                     1, subgraphs.at(0).denom);
}
```

### 3.4 不同 Join 类型的分母计算

```cpp
double CardinalityEstimator::CalculateUpdatedDenom(
    Subgraph2Denominator left,
    Subgraph2Denominator right,
    FilterInfoWithTotalDomains &filter) {

    double new_denom = left.denom * right.denom;

    switch (filter.filter_info->join_type) {
    case JoinType::INNER: {
        // 收集比较类型
        ExpressionType comparison_type = ExpressionType::INVALID;
        ExpressionIterator::EnumerateExpression(filter.filter_info->filter,
            [&](Expression &expr) {
                if (expr.GetExpressionClass() == ExpressionClass::BOUND_COMPARISON) {
                    comparison_type = expr.GetExpressionType();
                }
            });

        double extra_ratio = 1;
        switch (comparison_type) {
        case ExpressionType::COMPARE_EQUAL:
        case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
            // 等值 Join：使用 distinct count
            extra_ratio = filter.has_distinct_count_hll
                ? static_cast<double>(filter.distinct_count_hll)
                : static_cast<double>(filter.distinct_count_no_hll);
            break;

        case ExpressionType::COMPARE_LESSTHAN:
        case ExpressionType::COMPARE_GREATERTHAN:
        case ExpressionType::COMPARE_NOTEQUAL:
            // 非等值 Join：使用惩罚因子
            extra_ratio = filter.has_distinct_count_hll
                ? static_cast<double>(filter.distinct_count_hll)
                : static_cast<double>(filter.distinct_count_no_hll);
            extra_ratio = pow(extra_ratio, 2.0 / 3.0);  // 惩罚
            break;

        default:
            break;
        }
        new_denom *= extra_ratio;
        return new_denom;
    }

    case JoinType::SEMI:
    case JoinType::ANTI: {
        // SEMI/ANTI Join：使用默认选择性（5）
        // 只保留一侧的基数
        if (JoinRelationSet::IsSubset(*left.relations, *filter.filter_info->left_set)) {
            new_denom = left.denom * DEFAULT_SEMI_ANTI_SELECTIVITY;
        } else {
            new_denom = right.denom * DEFAULT_SEMI_ANTI_SELECTIVITY;
        }
        return new_denom;
    }

    default:
        // 笛卡尔积
        return new_denom;
    }
}
```

---

## 4. 代价模型（Cost Model）

### 4.1 CostModel 设计

DuckDB 使用简单但有效的代价模型：

```cpp
// src/include/duckdb/optimizer/join_order/cost_model.hpp

class CostModel {
public:
    explicit CostModel(QueryGraphManager &query_graph_manager);

    //! 初始化代价模型
    void InitCostModel();

    //! 计算 Join 的代价
    double ComputeCost(DPJoinNode &left, DPJoinNode &right);

    //! 基数估计器
    CardinalityEstimator cardinality_estimator;

private:
    QueryGraphManager &query_graph_manager;
};
```

### 4.2 代价计算

当前的代价模型非常简洁，主要基于基数：

```cpp
// src/optimizer/join_order/cost_model.cpp

double CostModel::ComputeCost(DPJoinNode &left, DPJoinNode &right) {
    // 获取合并后的关系集
    auto &combination = query_graph_manager.set_manager.Union(left.set, right.set);

    // 估计 Join 结果的基数
    auto join_card = cardinality_estimator.EstimateCardinalityWithSet<double>(combination);

    // Join 代价 = Join 结果基数
    auto join_cost = join_card;

    // 总代价 = 当前 Join 代价 + 左子树代价 + 右子树代价
    return join_cost + left.cost + right.cost;
}
```

**代价模型的特点：**

1. **简单有效**：代价等于 Join 结果的估计基数
2. **累积代价**：包含子树的代价，形成完整的计划代价
3. **可扩展**：可以加入 Join 算法、I/O 代价等因素

---

## 5. 计划枚举（Plan Enumerator）

### 5.1 PlanEnumerator 设计

DuckDB 实现了 DPhyp（Dynamic Programming Hyper-graph）算法：

```cpp
// src/include/duckdb/optimizer/join_order/plan_enumerator.hpp

class PlanEnumerator {
public:
    explicit PlanEnumerator(QueryGraphManager &query_graph_manager,
                            CostModel &cost_model,
                            const QueryGraphEdges &query_graph);

    //! 切换到近似算法的阈值
    static constexpr idx_t THRESHOLD_TO_SWAP_TO_APPROXIMATE = 12;

    //! 求解 Join 顺序
    void SolveJoinOrder();

    //! 初始化叶节点计划
    void InitLeafPlans();

    //! 获取计划表
    const reference_map_t<JoinRelationSet, unique_ptr<DPJoinNode>> &GetPlans() const;

private:
    //! 查询图边
    QueryGraphEdges const &query_graph;

    //! 已考虑的 Join 对数量
    idx_t pairs = 0;

    //! 查询图管理器
    QueryGraphManager &query_graph_manager;

    //! 代价模型
    CostModel &cost_model;

    //! DP 表：JoinRelationSet → 最优计划
    reference_map_t<JoinRelationSet, unique_ptr<DPJoinNode>> plans;

    //! 创建 Join 树节点
    unique_ptr<DPJoinNode> CreateJoinTree(JoinRelationSet &set,
                                           const vector<reference<NeighborInfo>> &connections,
                                           DPJoinNode &left,
                                           DPJoinNode &right);

    //! 发射一个 Join 对
    DPJoinNode &EmitPair(JoinRelationSet &left, JoinRelationSet &right,
                          const vector<reference<NeighborInfo>> &info);

    //! 尝试发射 Join 对（带超时检测）
    bool TryEmitPair(JoinRelationSet &left, JoinRelationSet &right,
                     const vector<reference<NeighborInfo>> &info);

    //! 精确求解（动态规划）
    bool SolveJoinOrderExactly();

    //! 近似求解（贪婪算法）
    void SolveJoinOrderApproximately();

    //! 生成笛卡尔积边
    void GenerateCrossProducts();

    //! CSG（Connected SubGraph）枚举
    bool EmitCSG(JoinRelationSet &node);
    bool EnumerateCSGRecursive(JoinRelationSet &node, unordered_set<idx_t> &exclusion_set);
    bool EnumerateCmpRecursive(JoinRelationSet &left, JoinRelationSet &right,
                               unordered_set<idx_t> &exclusion_set);
};
```

### 5.2 DPJoinNode：计划节点

```cpp
// src/include/duckdb/optimizer/join_order/join_node.hpp

class DPJoinNode {
public:
    //! 此节点代表的关系集
    JoinRelationSet &set;

    //! 左右子节点如何连接的信息
    optional_ptr<NeighborInfo> info;

    //! 是否是叶节点
    bool is_leaf;

    //! 左右子节点的关系集
    JoinRelationSet &left_set;
    JoinRelationSet &right_set;

    //! 计划代价
    double cost;

    //! 估计基数
    idx_t cardinality;

    //! 创建中间节点
    DPJoinNode(JoinRelationSet &set, optional_ptr<NeighborInfo> info,
               JoinRelationSet &left, JoinRelationSet &right, double cost);

    //! 创建叶节点（代价为 0）
    explicit DPJoinNode(JoinRelationSet &set);
};
```

### 5.3 求解 Join 顺序

```cpp
void PlanEnumerator::SolveJoinOrder() {
    bool force_no_cross_product =
        DBConfig::GetSetting<DebugForceNoCrossProductSetting>(query_graph_manager.context);

    // 决定使用精确算法还是近似算法
    if (query_graph_manager.relation_manager.NumRelations() >= THRESHOLD_TO_SWAP_TO_APPROXIMATE) {
        // 关系数 >= 12，直接使用贪婪算法
        SolveJoinOrderApproximately();
    } else if (!SolveJoinOrderExactly()) {
        // 动态规划超时，回退到贪婪算法
        SolveJoinOrderApproximately();
    }

    // 检查是否找到了完整的计划
    unordered_set<idx_t> bindings;
    for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
        bindings.insert(i);
    }
    auto &total_relation = query_graph_manager.set_manager.GetJoinRelation(bindings);
    auto final_plan = plans.find(total_relation);

    if (final_plan == plans.end()) {
        // 没有找到完整计划，说明图不连通，需要笛卡尔积
        if (force_no_cross_product) {
            throw InvalidInputException(
                "Query requires a cross-product, but 'force_no_cross_product' PRAGMA is enabled");
        }
        GenerateCrossProducts();
        return SolveJoinOrder();  // 递归求解
    }
}
```

### 5.4 精确求解：DPhyp 算法

```cpp
bool PlanEnumerator::SolveJoinOrderExactly() {
    // DPhyp 算法：从每个节点开始，枚举所有连通子图
    for (idx_t i = query_graph_manager.relation_manager.NumRelations(); i > 0; i--) {
        // 以每个节点为起点
        auto &start_node = query_graph_manager.set_manager.GetJoinRelation(i - 1);

        // 发射起始节点
        if (!EmitCSG(start_node)) {
            return false;  // 超时
        }

        // 初始化排除集（所有编号小于当前节点的关系）
        unordered_set<idx_t> exclusion_set;
        for (idx_t j = 0; j < i; j++) {
            exclusion_set.insert(j);
        }

        // 递归枚举邻居
        if (!EnumerateCSGRecursive(start_node, exclusion_set)) {
            return false;  // 超时
        }
    }
    return true;
}

bool PlanEnumerator::TryEmitPair(JoinRelationSet &left, JoinRelationSet &right,
                                  const vector<reference<NeighborInfo>> &info) {
    pairs++;

    // 超时检测：超过 10000 对时切换到贪婪算法
    if (pairs >= 10000) {
        return false;
    }

    EmitPair(left, right, info);
    return true;
}
```

### 5.5 近似求解：贪婪算法

```cpp
void PlanEnumerator::SolveJoinOrderApproximately() {
    // 贪婪算法：每次选择代价最小的 Join 对

    // 初始化：所有单表关系
    vector<reference<JoinRelationSet>> join_relations;
    for (idx_t i = 0; i < query_graph_manager.relation_manager.NumRelations(); i++) {
        join_relations.push_back(query_graph_manager.set_manager.GetJoinRelation(i));
    }

    while (join_relations.size() > 1) {
        // 寻找代价最小的 Join 对
        idx_t best_left = 0, best_right = 0;
        optional_ptr<DPJoinNode> best_connection;

        for (idx_t i = 0; i < join_relations.size(); i++) {
            auto left = join_relations[i];
            for (idx_t j = i + 1; j < join_relations.size(); j++) {
                auto right = join_relations[j];

                // 检查是否可以连接
                auto connection = query_graph.GetConnections(left, right);
                if (!connection.empty()) {
                    auto node = EmitPair(left, right, connection);

                    if (!best_connection || node.cost < best_connection->cost) {
                        best_connection = &EmitPair(left, right, connection);
                        best_left = i;
                        best_right = j;
                    }
                }
            }
        }

        if (!best_connection) {
            // 找不到连接，需要添加笛卡尔积
            // ... 选择两个最小的关系进行笛卡尔积
        }

        // 更新待处理关系：移除 left 和 right，添加合并后的关系
        auto &new_set = query_graph_manager.set_manager.Union(
            join_relations.at(best_left).get(),
            join_relations.at(best_right).get());
        join_relations.erase(join_relations.begin() + best_right);
        join_relations.erase(join_relations.begin() + best_left);
        join_relations.push_back(new_set);
    }
}
```

### 5.6 创建 Join 树

```cpp
unique_ptr<DPJoinNode> PlanEnumerator::CreateJoinTree(
    JoinRelationSet &set,
    const vector<reference<NeighborInfo>> &possible_connections,
    DPJoinNode &left,
    DPJoinNode &right) {

    // 选择最佳连接（优先非笛卡尔积）
    optional_ptr<NeighborInfo> best_connection = possible_connections.back().get();
    bool found_non_cross_product = false;

    for (auto &connection : possible_connections) {
        for (auto &filter : connection.get().filters) {
            if (filter->join_type != JoinType::INVALID) {
                best_connection = connection.get();
                found_non_cross_product = true;
                break;
            }
        }
        if (found_non_cross_product) break;
    }

    // 确定 Join 类型
    auto join_type = JoinType::INVALID;
    for (auto &filter_binding : best_connection->filters) {
        if (!filter_binding->left_set || !filter_binding->right_set) {
            continue;
        }
        join_type = filter_binding->join_type;
        // 优先 SEMI/ANTI Join（更高选择性）
        if (join_type == JoinType::SEMI || join_type == JoinType::ANTI) {
            break;
        }
    }

    // 计算代价
    auto cost = cost_model.ComputeCost(left, right);

    // 创建计划节点
    auto result = make_uniq<DPJoinNode>(set, best_connection, left.set, right.set, cost);
    result->cardinality = cost_model.cardinality_estimator.EstimateCardinalityWithSet<idx_t>(set);

    return result;
}
```

---

## 6. 统计信息传播（Statistics Propagation）

### 6.1 StatisticsPropagator 设计

统计信息传播器在优化后期遍历计划树，传播和利用统计信息：

```cpp
// src/include/duckdb/optimizer/statistics_propagator.hpp

class StatisticsPropagator {
public:
    StatisticsPropagator(Optimizer &optimizer, LogicalOperator &root);

    //! 传播统计信息
    unique_ptr<NodeStatistics> PropagateStatistics(unique_ptr<LogicalOperator> &node_ptr);

    //! 获取统计信息映射
    column_binding_map_t<unique_ptr<BaseStatistics>> GetStatisticsMap();

    //! 检查是否可以传播类型转换
    static bool CanPropagateCast(const LogicalType &source, const LogicalType &target);

private:
    //! 针对不同算子类型的传播方法
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalFilter &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalGet &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalJoin &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalProjection &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalAggregate &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalLimit &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalOrder &op, ...);
    unique_ptr<NodeStatistics> PropagateStatistics(LogicalWindow &op, ...);

    //! 传播表达式统计
    unique_ptr<BaseStatistics> PropagateExpression(unique_ptr<Expression> &expr);

    //! 处理过滤器
    FilterPropagateResult HandleFilter(unique_ptr<Expression> &condition);

    //! 更新过滤器统计
    void UpdateFilterStatistics(BaseStatistics &input,
                                 ExpressionType comparison_type,
                                 const Value &constant);

    //! 比较统计信息
    FilterPropagateResult PropagateComparison(BaseStatistics &left,
                                               BaseStatistics &right,
                                               ExpressionType comparison);

private:
    Optimizer &optimizer;
    ClientContext &context;
    optional_ptr<LogicalOperator> root;
    column_binding_map_t<unique_ptr<BaseStatistics>> statistics_map;
    unique_ptr<NodeStatistics> node_stats;
};
```

### 6.2 统计信息传播的作用

1. **过滤器优化**：根据统计信息判断过滤条件是否总是 true/false
2. **Join 优化**：根据统计信息创建动态过滤器
3. **聚合优化**：如果统计信息足够，可以直接计算聚合结果
4. **空结果检测**：检测必定返回空结果的情况

---

## 7. 其他高级优化

### 7.1 Join 消除（Join Elimination）

移除对查询结果没有影响的冗余 Join：

```cpp
// src/include/duckdb/optimizer/join_elimination.hpp

class JoinElimination : public LogicalOperatorVisitor {
public:
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> op);

private:
    //! 尝试消除 Join
    unique_ptr<LogicalOperator> TryEliminateJoin();

    //! 检查是否包含 distinct group
    bool ContainDistinctGroup(vector<ColumnBinding> &exprs);

    PipelineInfo pipe_info;
};
```

**Join 消除条件（以 LEFT JOIN 为例）：**

1. 输出只包含外表列
2. 内表列不用于过滤（WHERE、HAVING 等）
3. 每个外表行最多匹配一个内表行：
   - 内表 Join 列是唯一的（主键、唯一索引）
   - 或者 Join 列包含完整的 distinct group

**示例：**

```sql
-- 原始查询
SELECT a.id, a.name
FROM a
LEFT JOIN b ON a.id = b.a_id  -- b.a_id 是唯一的
WHERE a.active = true

-- 优化后（消除了无用的 LEFT JOIN）
SELECT a.id, a.name
FROM a
WHERE a.active = true
```

### 7.2 TopN 优化

将 ORDER BY + LIMIT 组合优化为 TopN 算子：

```cpp
// src/include/duckdb/optimizer/topn_optimizer.hpp

class TopN {
public:
    explicit TopN(ClientContext &context);

    //! 优化 ORDER BY + LIMIT 为 TopN
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> op);

    //! 检查是否可以优化
    static bool CanOptimize(LogicalOperator &op,
                            optional_ptr<ClientContext> context = nullptr);

private:
    //! 下推动态过滤器
    void PushdownDynamicFilters(LogicalTopN &op);

    ClientContext &context;
};
```

**优化示例：**

```sql
-- 原始查询
SELECT * FROM t ORDER BY x LIMIT 10

-- 优化后
LogicalTopN(10, ORDER BY x)
└── LogicalGet(t)

-- 优势：
-- 1. 不需要完整排序，只需维护大小为 10 的堆
-- 2. 可以提前终止扫描（如果使用索引）
```

### 7.3 Build/Probe 侧优化

选择 Hash Join 的 Build 侧和 Probe 侧：

```cpp
// src/include/duckdb/optimizer/build_probe_side_optimizer.hpp

class BuildProbeSideOptimizer : LogicalOperatorVisitor {
private:
    static constexpr idx_t COLUMN_COUNT_PENALTY = 2;
    static constexpr double PREFER_RIGHT_DEEP_PENALTY = 0.15;

public:
    explicit BuildProbeSideOptimizer(ClientContext &context, LogicalOperator &op);

    void VisitOperator(LogicalOperator &op) override;

private:
    //! 尝试交换 Join 子节点
    bool TryFlipJoinChildren(LogicalOperator &op) const;

    //! 检查子树是否包含 Join
    static idx_t ChildHasJoins(LogicalOperator &op);

    //! 计算 Build 侧大小
    static BuildSize GetBuildSizes(const LogicalOperator &op,
                                    idx_t lhs_cardinality,
                                    idx_t rhs_cardinality);

    static double GetBuildSize(vector<LogicalType> types, idx_t cardinality);

private:
    ClientContext &context;
    vector<ColumnBinding> preferred_on_probe_side;
};
```

**Build 侧选择原则：**

1. **较小的一侧作为 Build 侧**：减少 Hash 表大小
2. **考虑列数**：列多的一侧代价更高
3. **偏好右深树**：轻微偏好右侧作为 Build 侧

### 7.4 延迟物化（Late Materialization）

推迟读取不需要的列，直到真正需要时才读取：

```cpp
// 延迟物化的思想：
// 1. 先只读取 Join 和过滤器需要的列
// 2. 在完成 Join 和过滤后，再回表读取其他列
// 3. 减少中间结果的数据量

// 示例：
// SELECT a.*, b.name FROM a JOIN b ON a.id = b.a_id WHERE b.active = true

// 无延迟物化：
// TableScan(a, columns=[*]) → Join → Filter → Project

// 有延迟物化：
// TableScan(a, columns=[id, rowid]) → Join → Filter → LookupColumns(a, rowid) → Project
```

### 7.5 TopN 窗口消除

将 `ROW_NUMBER() + WHERE row_num <= N` 模式优化为 TopN：

```cpp
// 优化前：
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY x ORDER BY y) AS rn
    FROM t
) WHERE rn <= 10

// 优化后（理论上）：
// 对每个分区只保留前 10 行
LogicalTopNPartitioned(10, PARTITION BY x, ORDER BY y)
└── LogicalGet(t)
```

---

## 8. 优化示例

### 8.1 四表 Join 优化过程

```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
JOIN categories cat ON p.category_id = cat.id
WHERE c.country = 'China'
  AND cat.name = 'Electronics'
```

**Step 1: 构建查询图**

```
Relations: {o, c, p, cat}
Edges:
  - (o, c): o.customer_id = c.id
  - (o, p): o.product_id = p.id
  - (p, cat): p.category_id = cat.id
Filters:
  - c.country = 'China'
  - cat.name = 'Electronics'
```

**Step 2: 初始化统计信息**

```
Base cardinalities:
  - orders: 1,000,000
  - customers: 50,000 (with filter: ~10,000)
  - products: 10,000
  - categories: 100 (with filter: ~1)

Distinct counts:
  - o.customer_id: 40,000
  - c.id: 50,000
  - o.product_id: 8,000
  - p.id: 10,000
  - p.category_id: 100
  - cat.id: 100
```

**Step 3: 枚举 Join 顺序**

```
候选顺序及估计基数：

1. ((o ⋈ c) ⋈ p) ⋈ cat
   - o ⋈ c: 1M × 10K / 50K = 200K
   - (o ⋈ c) ⋈ p: 200K × 10K / 10K = 200K
   - ((o ⋈ c) ⋈ p) ⋈ cat: 200K × 1 / 100 = 2K
   - 总代价: 200K + 200K + 2K = 402K

2. (cat ⋈ p) ⋈ (c ⋈ o)
   - cat ⋈ p: 1 × 10K / 100 = 100
   - c ⋈ o: 10K × 1M / 50K = 200K
   - (cat ⋈ p) ⋈ (c ⋈ o): 100 × 200K / 10K = 2K
   - 总代价: 100 + 200K + 2K = 202K  ✓ 更优

3. ((cat ⋈ p) ⋈ o) ⋈ c
   - cat ⋈ p: 100
   - (cat ⋈ p) ⋈ o: 100 × 1M / 10K = 10K
   - ((cat ⋈ p) ⋈ o) ⋈ c: 10K × 10K / 50K = 2K
   - 总代价: 100 + 10K + 2K = 12K  ✓✓ 最优
```

**Step 4: 生成最优计划**

```
LogicalJoin(INNER, o.customer_id = c.id)
├── LogicalJoin(INNER, o.product_id = p.id)
│   ├── LogicalGet(orders)
│   └── LogicalJoin(INNER, p.category_id = cat.id)
│       ├── LogicalGet(products)
│       └── LogicalFilter(cat.name = 'Electronics')
│           └── LogicalGet(categories)
└── LogicalFilter(c.country = 'China')
    └── LogicalGet(customers)
```

---

## 9. 总结

### 9.1 核心技术总结

| 技术 | 作用 | 实现类 |
|------|------|--------|
| Query Graph | 表示关系间的连接关系 | `QueryGraphEdges` |
| DPhyp Algorithm | 精确枚举 Join 顺序 | `PlanEnumerator` |
| Greedy Algorithm | 近似求解大规模 Join | `PlanEnumerator` |
| Cardinality Estimation | 估计 Join 结果大小 | `CardinalityEstimator` |
| Cost Model | 计算计划代价 | `CostModel` |
| Statistics Propagation | 传播和利用统计信息 | `StatisticsPropagator` |
| Join Elimination | 消除冗余 Join | `JoinElimination` |
| TopN Optimization | ORDER BY + LIMIT → TopN | `TopN` |
| Build Side Selection | 选择 Hash Join Build 侧 | `BuildProbeSideOptimizer` |

### 9.2 基数估计公式

```
                    |R₁| × |R₂| × ... × |Rₙ|
Cardinality = ─────────────────────────────────────
               max(d(c₁)) × max(d(c₂)) × ... × max(d(cₘ))

其中：
- |Rᵢ| = 关系 Rᵢ 的基数
- d(cⱼ) = 列 cⱼ 的 distinct count
- cⱼ 来自等价集中的所有列
```

### 9.3 算法选择策略

```
if (关系数 >= 12) {
    使用贪婪算法 (O(n³))
} else {
    尝试动态规划 (DPhyp)
    if (超时 或 对数 > 10000) {
        回退到贪婪算法
    }
}
```

### 9.4 下一章预告

下一章将讲解物理计划生成（Physical Planner）：
- LogicalOperator → PhysicalOperator 转换
- Join 算法选择（Hash Join vs Merge Join vs Nested Loop）
- Pipeline 构建和并行执行
- 动态过滤器下推

---

## 附录：相关源文件索引

| 组件 | 主要文件 |
|------|----------|
| JoinOrderOptimizer | `src/optimizer/join_order/join_order_optimizer.cpp` |
| QueryGraph | `src/optimizer/join_order/query_graph.cpp` |
| PlanEnumerator | `src/optimizer/join_order/plan_enumerator.cpp` |
| CardinalityEstimator | `src/optimizer/join_order/cardinality_estimator.cpp` |
| CostModel | `src/optimizer/join_order/cost_model.cpp` |
| StatisticsPropagator | `src/optimizer/statistics_propagator.cpp` |
| JoinElimination | `src/optimizer/join_elimination.cpp` |
| TopNOptimizer | `src/optimizer/topn_optimizer.cpp` |
| BuildProbeSideOptimizer | `src/optimizer/build_probe_side_optimizer.cpp` |

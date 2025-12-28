# DuckDB 查询优化器深度解析：第八章 - 特殊优化技术

## 8.1 章节概述

本章介绍 DuckDB 优化器中的一系列特殊优化技术。这些优化针对特定的查询模式或计划结构，能够显著提升查询性能。

```
┌─────────────────────────────────────────────────────────────────┐
│                    特殊优化技术全景                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 表达式级优化                                             │   │
│  │ • Common Subexpression Elimination (CSE)               │   │
│  │   - 识别重复计算的表达式                                  │   │
│  │   - 通过投影算子共享计算结果                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Join 优化                                               │   │
│  │ • Build/Probe Side Selection                           │   │
│  │   - 基于代价选择 Hash Join 的构建端                      │   │
│  │ • Join Filter Pushdown                                 │   │
│  │   - 动态生成并下推 Join 过滤器                           │   │
│  │ • Join Elimination                                     │   │
│  │   - 消除不影响结果的冗余 Join                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CTE 优化                                                │   │
│  │ • CTE Inlining                                         │   │
│  │   - 内联只使用一次的 CTE                                 │   │
│  │   - 对有 LIMIT 的查询做特殊处理                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Limit 优化                                              │   │
│  │ • Limit Pushdown                                       │   │
│  │   - 将 LIMIT 下推到 Projection 之下                     │   │
│  │   - 减少不必要的表达式计算                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.2 公共子表达式消除 (CSE)

### 8.2.1 CSE 概述

公共子表达式消除 (Common Subexpression Elimination) 是一种经典的编译器优化技术，DuckDB 将其应用于 SQL 查询优化。当同一个表达式在查询中多次出现时，CSE 可以避免重复计算。

```
优化前:
SELECT a + b + c, (a + b) * 2, (a + b) - d FROM t;

┌─────────────────────────────────────────────────────────────────┐
│ 计算流程（未优化）                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  表达式1: a + b + c     →  计算 (a + b)，然后 + c              │
│  表达式2: (a + b) * 2   →  重新计算 (a + b)，然后 * 2          │
│  表达式3: (a + b) - d   →  再次计算 (a + b)，然后 - d          │
│                                                                 │
│  共计算 (a + b) 三次！                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

优化后:
┌─────────────────────────────────────────────────────────────────┐
│ 计算流程（CSE 优化）                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐                                           │
│  │ 新增 Projection │  计算 cse_0 = a + b                       │
│  └────────┬────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │ 原始 Projection │                                           │
│  │                 │                                           │
│  │  cse_0 + c      │  复用 cse_0                               │
│  │  cse_0 * 2      │  复用 cse_0                               │
│  │  cse_0 - d      │  复用 cse_0                               │
│  └─────────────────┘                                           │
│                                                                 │
│  (a + b) 只计算一次！                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2.2 CSE 实现架构

```cpp
// src/optimizer/cse_optimizer.cpp

// CSE 节点：记录表达式出现次数和位置
struct CSENode {
    idx_t count = 1;                    // 出现次数
    unique_ptr<Expression> *expr_ptr;   // 表达式位置指针
};

// CSE 替换状态
struct CSEReplacementState {
    // 表达式 → 出现信息的映射
    expression_map_t<CSENode> expression_count;
    // 待替换的表达式列表
    vector<unique_ptr<Expression> *> expressions_to_replace;
};
```

### 8.2.3 表达式计数

CSE 的第一步是遍历所有表达式，统计每个表达式的出现次数：

```cpp
void CommonSubExpressionOptimizer::CountExpressions(
    Expression &expr,
    CSEReplacementState &state) {

    // 检查表达式是否可以提取为 CSE
    if (!CanExtract(expr)) {
        // 某些表达式不能提取（如列引用、常量等）
        return;
    }

    // 查找表达式是否已经出现过
    auto node = state.expression_count.find(expr);
    if (node == state.expression_count.end()) {
        // 首次出现，添加到计数表
        state.expression_count[expr] = CSENode();
    } else {
        // 再次出现，增加计数
        node->second.count++;
    }

    // 递归处理子表达式
    ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) {
        CountExpressions(child, state);
    });
}
```

### 8.2.4 处理短路求值

对于 AND/OR 等短路求值表达式，需要特殊处理：

```cpp
void CommonSubExpressionOptimizer::CountExpressions(
    Expression &expr,
    CSEReplacementState &state) {

    // 短路求值表达式：AND, OR, CASE
    // 只有第一个分支保证执行，其他分支可能不执行
    switch (expr.type) {
    case ExpressionType::CONJUNCTION_AND:
    case ExpressionType::CONJUNCTION_OR: {
        // 只计算第一个子表达式
        auto &conj = expr.Cast<BoundConjunctionExpression>();
        if (!conj.children.empty()) {
            CountExpressions(*conj.children[0], state);
        }
        // 后续子表达式可能不执行，不计入 CSE 候选
        return;
    }
    case ExpressionType::CASE_EXPR: {
        auto &case_expr = expr.Cast<BoundCaseExpression>();
        // CASE 的条件部分可能不全部执行
        // 只处理 ELSE 分支（保证执行）
        if (case_expr.else_expr) {
            CountExpressions(*case_expr.else_expr, state);
        }
        return;
    }
    default:
        break;
    }

    // 正常处理...
}
```

### 8.2.5 表达式替换

当表达式出现次数超过阈值时，执行替换：

```cpp
void CommonSubExpressionOptimizer::PerformCSEReplacement(
    unique_ptr<Expression> &expr_ptr,
    CSEReplacementState &state) {

    Expression &expr = *expr_ptr;

    // 查找是否是 CSE 候选
    auto node = state.expression_count.find(expr);
    if (node != state.expression_count.end()) {
        if (node->second.count > 1) {
            // 出现超过一次，需要替换
            if (!node->second.expr_ptr) {
                // 首次遇到，记录位置用于后续提取
                node->second.expr_ptr = &expr_ptr;
                state.expressions_to_replace.push_back(&expr_ptr);
            }
        }
    }

    // 递归处理子表达式
    ExpressionIterator::EnumerateChildren(expr, [&](unique_ptr<Expression> &child) {
        PerformCSEReplacement(child, state);
    });
}
```

### 8.2.6 投影插入

将提取的公共子表达式作为新的投影列：

```cpp
unique_ptr<LogicalOperator> CommonSubExpressionOptimizer::ExtractCommonSubexpressions(
    unique_ptr<LogicalOperator> child,
    vector<unique_ptr<Expression>> &expressions,
    vector<unique_ptr<Expression> *> &expressions_to_replace) {

    if (expressions_to_replace.empty()) {
        return child;
    }

    // 创建新的投影算子
    auto projection = make_uniq<LogicalProjection>(
        GenerateTableIndex(), vector<unique_ptr<Expression>>());

    // 首先保留原有列
    for (idx_t i = 0; i < child->types.size(); i++) {
        projection->expressions.push_back(
            make_uniq<BoundColumnRefExpression>(
                child->types[i],
                ColumnBinding(child->GetTableIndex(), i)));
    }

    // 添加 CSE 表达式作为新列
    for (auto expr_ptr : expressions_to_replace) {
        idx_t cse_column_index = projection->expressions.size();

        // 将原表达式移动到投影中
        projection->expressions.push_back(std::move(**expr_ptr));

        // 原位置替换为对新列的引用
        *expr_ptr = make_uniq<BoundColumnRefExpression>(
            projection->expressions.back()->return_type,
            ColumnBinding(projection->GetTableIndex(), cse_column_index));
    }

    // 更新所有引用相同 CSE 的位置
    UpdateCSEReferences(expressions, expressions_to_replace, projection->GetTableIndex());

    projection->children.push_back(std::move(child));
    return std::move(projection);
}
```

### 8.2.7 CSE 完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│ CSE 优化流程                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  输入计划:                                                      │
│  ┌──────────────────────────────┐                              │
│  │ Projection                   │                              │
│  │   (a+b)+c, (a+b)*2, (a+b)-d │                              │
│  └──────────────┬───────────────┘                              │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────────────────┐                              │
│  │ TableScan                    │                              │
│  │   columns: a, b, c, d        │                              │
│  └──────────────────────────────┘                              │
│                                                                 │
│  Step 1: CountExpressions                                      │
│  ┌──────────────────────────────────────┐                      │
│  │ expression_count:                    │                      │
│  │   (a+b)   → count: 3                │                      │
│  │   (a+b)+c → count: 1                │                      │
│  │   (a+b)*2 → count: 1                │                      │
│  │   (a+b)-d → count: 1                │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
│  Step 2: PerformCSEReplacement                                 │
│  ┌──────────────────────────────────────┐                      │
│  │ expressions_to_replace:              │                      │
│  │   (a+b) at positions [0,1,2]        │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
│  Step 3: ExtractCommonSubexpressions                           │
│                                                                 │
│  输出计划:                                                      │
│  ┌──────────────────────────────┐                              │
│  │ Projection (原始)            │                              │
│  │   #0+c, #0*2, #0-d          │  引用 CSE 列                  │
│  └──────────────┬───────────────┘                              │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────────────────┐                              │
│  │ Projection (新增)            │                              │
│  │   a, b, c, d, (a+b) as #0   │  计算 CSE                     │
│  └──────────────┬───────────────┘                              │
│                 │                                               │
│                 ▼                                               │
│  ┌──────────────────────────────┐                              │
│  │ TableScan                    │                              │
│  └──────────────────────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.3 Build/Probe 端选择优化

### 8.3.1 Hash Join 背景

Hash Join 是数据库中最常用的 Join 算法之一。它分为两个阶段：
1. **Build 阶段**: 读取一侧数据构建哈希表
2. **Probe 阶段**: 读取另一侧数据，在哈希表中查找匹配

选择哪一侧作为 Build 端对性能影响巨大：

```
┌─────────────────────────────────────────────────────────────────┐
│ Build/Probe 选择的影响                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景: 小表 (1K 行) JOIN 大表 (1M 行)                           │
│                                                                 │
│  方案 A: 小表做 Build                                           │
│  ┌────────────────────────────────────────────────────┐        │
│  │ Build: 1K 行 → 小哈希表，内存友好                   │        │
│  │ Probe: 1M 行 → 顺序扫描，每行查找一次               │        │
│  │ 总代价: 1K (build) + 1M (probe) ≈ 1M               │        │
│  └────────────────────────────────────────────────────┘        │
│                                                                 │
│  方案 B: 大表做 Build                                           │
│  ┌────────────────────────────────────────────────────┐        │
│  │ Build: 1M 行 → 大哈希表，可能溢出内存              │        │
│  │ Probe: 1K 行 → 只需查找 1K 次                      │        │
│  │ 总代价: 1M (build) + 1K (probe) ≈ 1M               │        │
│  │ 但内存代价: 1M 行哈希表可能导致 spilling！          │        │
│  └────────────────────────────────────────────────────┘        │
│                                                                 │
│  结论: 应该选择小表做 Build 端                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 代价计算模型

DuckDB 使用基于哈希表大小的代价模型：

```cpp
// src/optimizer/build_probe_side_optimizer.cpp

double BuildProbeSideOptimizer::GetBuildSize(
    vector<LogicalType> types,
    const idx_t cardinality) {

    // 计算行宽度
    idx_t row_width = 0;
    for (auto &type : types) {
        row_width += GetTypeIdSize(type.InternalType());
    }

    // Hash Table 额外开销:
    // - HASH 列
    // - ht_entry 指针
    // - 链表指针
    row_width += 3 * sizeof(ht_entry_t);

    // 总大小 = 行宽 × 基数
    return static_cast<double>(row_width * cardinality);
}
```

### 8.3.3 优化决策逻辑

```cpp
void BuildProbeSideOptimizer::TryFlipJoinChildren(
    LogicalOperator &op,
    idx_t cardinality_ratio) {

    D_ASSERT(op.type == LogicalOperatorType::LOGICAL_COMPARISON_JOIN);
    auto &join = op.Cast<LogicalComparisonJoin>();

    // 获取两侧的基数
    auto lhs_cardinality = join.children[0]->EstimatedCardinality();
    auto rhs_cardinality = join.children[1]->EstimatedCardinality();

    // 如果基数差异不大，不值得交换
    if (lhs_cardinality < cardinality_ratio * rhs_cardinality) {
        return;
    }

    // 计算两侧做 Build 端的代价
    auto lhs_build_size = GetBuildSize(
        join.children[0]->types, lhs_cardinality);
    auto rhs_build_size = GetBuildSize(
        join.children[1]->types, rhs_cardinality);

    // 如果左侧做 Build 更大，则交换
    if (lhs_build_size > rhs_build_size) {
        FlipChildren(join);
    }
}
```

### 8.3.4 Join 类型约束

并非所有 Join 类型都可以交换两侧：

```cpp
void BuildProbeSideOptimizer::FlipChildren(LogicalComparisonJoin &join) {
    // 检查 Join 类型是否允许交换
    switch (join.join_type) {
    case JoinType::INNER:
        // Inner Join: 可以交换，类型不变
        break;
    case JoinType::LEFT:
        // Left Join → Right Join
        join.join_type = JoinType::RIGHT;
        break;
    case JoinType::RIGHT:
        // Right Join → Left Join
        join.join_type = JoinType::LEFT;
        break;
    case JoinType::SEMI:
        // Semi Join → Right Semi
        join.join_type = JoinType::RIGHT_SEMI;
        break;
    case JoinType::ANTI:
        // Anti Join → Right Anti
        join.join_type = JoinType::RIGHT_ANTI;
        break;
    default:
        // FULL OUTER, MARK, SINGLE 等不能交换
        return;
    }

    // 交换子节点
    std::swap(join.children[0], join.children[1]);

    // 交换 Join 条件中的左右表达式
    for (auto &condition : join.conditions) {
        std::swap(condition.left, condition.right);
        // 调整比较方向
        condition.comparison = FlipComparisonExpression(condition.comparison);
    }
}
```

### 8.3.5 优化流程图

```
┌─────────────────────────────────────────────────────────────────┐
│ Build/Probe Side 优化流程                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  输入: Hash Join                                                │
│  ┌────────────────────────────────────────┐                    │
│  │         Hash Join (INNER)              │                    │
│  │           /          \                 │                    │
│  │     Left (100K)    Right (1K)          │                    │
│  │     Build 端       Probe 端             │                    │
│  └────────────────────────────────────────┘                    │
│                                                                 │
│  Step 1: 检查基数差异                                           │
│  ┌────────────────────────────────────────┐                    │
│  │ lhs_cardinality = 100K                 │                    │
│  │ rhs_cardinality = 1K                   │                    │
│  │ ratio = 100 > cardinality_ratio (5)    │                    │
│  │ → 继续检查                              │                    │
│  └────────────────────────────────────────┘                    │
│                                                                 │
│  Step 2: 计算 Build 代价                                        │
│  ┌────────────────────────────────────────┐                    │
│  │ lhs_build_size = 100K × row_width      │                    │
│  │ rhs_build_size = 1K × row_width        │                    │
│  │ lhs_build_size > rhs_build_size        │                    │
│  │ → 应该交换                              │                    │
│  └────────────────────────────────────────┘                    │
│                                                                 │
│  Step 3: 执行交换                                               │
│  ┌────────────────────────────────────────┐                    │
│  │         Hash Join (INNER)              │                    │
│  │           /          \                 │                    │
│  │     Right (1K)    Left (100K)          │                    │
│  │     Build 端       Probe 端             │                    │
│  └────────────────────────────────────────┘                    │
│                                                                 │
│  优化效果: 哈希表大小从 100K 降低到 1K                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.4 动态 Join Filter 下推

### 8.4.1 动态过滤概述

动态 Join Filter（也称为 Bloom Filter Pushdown 或 Runtime Filter）是一种在 Join 执行时动态生成过滤器并下推到扫描端的优化技术。

```
┌─────────────────────────────────────────────────────────────────┐
│ 动态 Join Filter 原理                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景:                                                          │
│  SELECT * FROM orders o                                        │
│  JOIN customers c ON o.customer_id = c.id                      │
│  WHERE c.country = 'China';                                    │
│                                                                 │
│  传统执行:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                   Hash Join                                ││
│  │                  /         \                               ││
│  │     TableScan(orders)    Filter + TableScan(customers)    ││
│  │     扫描全部 10M 行       只有 100K 中国客户                ││
│  │                                                            ││
│  │     实际匹配的订单可能只有 500K                            ││
│  │     但需要扫描 10M 行订单！                                ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  动态过滤执行:                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │   Step 1: Build 阶段收集 customer_id 的范围                ││
│  │           min_id = 1000, max_id = 5000                     ││
│  │                                                            ││
│  │   Step 2: 生成动态过滤器                                   ││
│  │           o.customer_id >= 1000 AND o.customer_id <= 5000  ││
│  │                                                            ││
│  │   Step 3: 下推到 orders 扫描                               ││
│  │           利用 Zonemap 跳过不相关的数据段                   ││
│  │           只扫描可能匹配的 800K 行                          ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  性能提升: 10M → 800K (减少 92% 的扫描)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4.2 优化器实现

```cpp
// src/optimizer/join_filter_pushdown_optimizer.cpp

void JoinFilterPushdownOptimizer::GenerateJoinFilters(
    LogicalComparisonJoin &join,
    vector<JoinFilterInfo> &filters) {

    // 遍历 Join 条件
    for (auto &condition : join.conditions) {
        // 只处理等值条件
        if (condition.comparison != ExpressionType::COMPARE_EQUAL) {
            continue;
        }

        // 只处理整型列（可以计算范围）
        if (!condition.left->return_type.IsIntegral() ||
            !condition.right->return_type.IsIntegral()) {
            continue;
        }

        // 检查是否是列引用
        if (condition.left->type != ExpressionType::BOUND_COLUMN_REF ||
            condition.right->type != ExpressionType::BOUND_COLUMN_REF) {
            continue;
        }

        // 生成动态过滤信息
        JoinFilterInfo filter;
        filter.probe_column = condition.right->Copy();  // Probe 端的列
        filter.build_column = condition.left->Copy();   // Build 端的列

        // 将在执行时计算 min/max 并生成过滤器
        filters.push_back(std::move(filter));
    }
}
```

### 8.4.3 物理计划中的实现

动态过滤在物理执行时才真正生效：

```cpp
// 在 Hash Join Build 阶段收集统计信息
void PhysicalHashJoin::BuildHashTable(ExecutionContext &context) {
    // 构建哈希表...

    // 同时收集 Build 端键列的 min/max
    for (auto &filter : join_filter_infos) {
        filter.min_value = GetMinValue(filter.build_column);
        filter.max_value = GetMaxValue(filter.build_column);
    }

    // 生成动态过滤器
    GenerateDynamicFilters();
}

// 生成的动态过滤器会被下推到 TableScan
// TableScan 利用 Zonemap 快速跳过不相关的数据段
```

### 8.4.4 聚合函数生成过滤信息

优化器通过插入聚合算子来收集 Build 端的统计信息：

```cpp
void JoinFilterPushdownOptimizer::CreateFilterAggregate(
    LogicalComparisonJoin &join,
    vector<JoinFilterInfo> &filters) {

    // 创建聚合表达式收集 min/max
    vector<unique_ptr<Expression>> aggregates;

    for (auto &filter : filters) {
        // MIN(build_column)
        auto min_agg = make_uniq<BoundAggregateExpression>(
            AggregateFunction::GetFunction("min"),
            {filter.build_column->Copy()},
            nullptr, nullptr, AggregateType::NON_DISTINCT);
        aggregates.push_back(std::move(min_agg));

        // MAX(build_column)
        auto max_agg = make_uniq<BoundAggregateExpression>(
            AggregateFunction::GetFunction("max"),
            {filter.build_column->Copy()},
            nullptr, nullptr, AggregateType::NON_DISTINCT);
        aggregates.push_back(std::move(max_agg));
    }

    // 聚合结果用于生成运行时过滤器
}
```

### 8.4.5 动态过滤流程图

```
┌─────────────────────────────────────────────────────────────────┐
│ 动态 Join Filter 执行流程                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  逻辑计划:                                                      │
│  ┌─────────────────────────────────────────────────┐           │
│  │              Hash Join                          │           │
│  │           (o.id = c.customer_id)               │           │
│  │              /          \                       │           │
│  │    TableScan(orders)  Filter(country='China')  │           │
│  │                       TableScan(customers)     │           │
│  └─────────────────────────────────────────────────┘           │
│                                                                 │
│  优化后计划:                                                    │
│  ┌─────────────────────────────────────────────────┐           │
│  │              Hash Join                          │           │
│  │           (o.id = c.customer_id)               │           │
│  │           [dynamic_filter: o.id]               │           │
│  │              /          \                       │           │
│  │    TableScan(orders)  Aggregate                │           │
│  │    [待接收动态过滤器]   (MIN/MAX c.customer_id) │           │
│  │                             |                   │           │
│  │                       Filter + Scan             │           │
│  └─────────────────────────────────────────────────┘           │
│                                                                 │
│  执行时:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 1. Build 阶段执行 Aggregate                                ││
│  │    结果: MIN(customer_id)=1000, MAX(customer_id)=5000      ││
│  │                                                            ││
│  │ 2. 生成动态过滤器                                          ││
│  │    o.id >= 1000 AND o.id <= 5000                          ││
│  │                                                            ││
│  │ 3. 下推到 TableScan(orders)                                ││
│  │    利用 Zonemap: 只扫描 id 范围与 [1000,5000] 重叠的段     ││
│  │                                                            ││
│  │ 4. Probe 阶段大大减少                                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.5 Join 消除优化

### 8.5.1 Join 消除原理

当 Join 的一侧对结果没有实际贡献时，可以完全消除该 Join。常见场景：

```
┌─────────────────────────────────────────────────────────────────┐
│ Join 消除场景                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景 1: 主键/唯一键 Join 且只使用另一侧列                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ SELECT o.* FROM orders o                                   ││
│  │ JOIN customers c ON o.customer_id = c.id;  -- c.id 是主键 ││
│  │                                                            ││
│  │ 如果:                                                      ││
│  │   1. c.id 是唯一的（主键或唯一约束）                       ││
│  │   2. 查询只使用 o 的列，不使用 c 的任何列                  ││
│  │   3. o.customer_id 有外键约束保证匹配存在                  ││
│  │                                                            ││
│  │ 则可以简化为:                                              ││
│  │ SELECT * FROM orders;                                      ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  场景 2: LEFT JOIN 右侧列未使用                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ SELECT o.* FROM orders o                                   ││
│  │ LEFT JOIN shipping s ON o.id = s.order_id;                ││
│  │                                                            ││
│  │ LEFT JOIN 不会减少左侧行数                                 ││
│  │ 如果右侧列未使用，可以消除 Join                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.5.2 实现核心逻辑

```cpp
// src/optimizer/join_elimination.cpp

unique_ptr<LogicalOperator> JoinEliminationOptimizer::Optimize(
    unique_ptr<LogicalOperator> op) {

    if (op->type != LogicalOperatorType::LOGICAL_COMPARISON_JOIN) {
        // 递归处理子节点
        for (auto &child : op->children) {
            child = Optimize(std::move(child));
        }
        return op;
    }

    auto &join = op->Cast<LogicalComparisonJoin>();

    // 检查是否可以消除
    if (CanEliminateJoin(join)) {
        // 返回保留的那一侧
        return std::move(join.children[GetPreservedSide(join)]);
    }

    // 递归处理子节点
    for (auto &child : op->children) {
        child = Optimize(std::move(child));
    }
    return op;
}
```

### 8.5.3 唯一性检测

判断 Join 是否可以消除的关键是检测一侧是否有唯一性保证：

```cpp
bool JoinEliminationOptimizer::CanEliminateJoin(LogicalComparisonJoin &join) {
    // 只处理 Inner Join 和 Left Join
    if (join.join_type != JoinType::INNER &&
        join.join_type != JoinType::LEFT) {
        return false;
    }

    // 检查右侧列是否被使用
    // 通过检查父算子的列引用来判断
    if (!RightSideColumnsUnused(join)) {
        return false;
    }

    // 检查 Join 键是否唯一
    // 通过 distinct_groups 统计信息判断
    for (auto &condition : join.conditions) {
        if (condition.comparison != ExpressionType::COMPARE_EQUAL) {
            return false;  // 必须是等值 Join
        }

        // 检查右侧键列是否唯一
        auto &right_col = condition.right;
        if (!IsColumnUnique(join.children[1].get(), right_col)) {
            return false;
        }
    }

    return true;
}
```

### 8.5.4 基于统计信息的唯一性判断

```cpp
bool JoinEliminationOptimizer::IsColumnUnique(
    LogicalOperator *op,
    const unique_ptr<Expression> &column) {

    // 获取列的统计信息
    auto stats = GetColumnStatistics(op, column);
    if (!stats) {
        return false;
    }

    // 检查 distinct_count 是否等于行数
    auto cardinality = op->EstimatedCardinality();
    auto distinct_count = stats->GetDistinctCount();

    // 如果每个值都不同，则该列是唯一的
    if (distinct_count.has_value() &&
        distinct_count.value() >= cardinality) {
        return true;
    }

    // 检查是否有唯一约束
    // (通过 catalog 获取约束信息)
    return HasUniqueConstraint(op, column);
}
```

### 8.5.5 Join 消除示例

```
┌─────────────────────────────────────────────────────────────────┐
│ Join 消除示例                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始查询:                                                      │
│  SELECT o.order_id, o.amount                                   │
│  FROM orders o                                                 │
│  INNER JOIN customers c ON o.customer_id = c.id;               │
│                                                                 │
│  条件检查:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ 1. c.id 是主键 (唯一)                              ✓      ││
│  │ 2. 只使用 o 的列，不使用 c 的列                    ✓      ││
│  │ 3. INNER JOIN 类型                                 ✓      ││
│  │ 4. Join 条件是等值比较                             ✓      ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  优化后:                                                        │
│  SELECT order_id, amount FROM orders;                          │
│                                                                 │
│  性能提升:                                                      │
│  - 避免读取 customers 表                                       │
│  - 避免 Hash Join 计算                                         │
│  - 减少内存使用                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.6 CTE 内联优化

### 8.6.1 CTE 概述

CTE (Common Table Expression) 是 SQL 中定义临时命名结果集的语法。DuckDB 需要决定是物化 CTE 还是内联展开。

```sql
-- CTE 示例
WITH regional_sales AS (
    SELECT region, SUM(amount) as total
    FROM orders
    GROUP BY region
)
SELECT * FROM regional_sales
WHERE total > 1000000;
```

### 8.6.2 内联决策因素

```
┌─────────────────────────────────────────────────────────────────┐
│ CTE 内联决策                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  物化 (Materialize):                                           │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ • CTE 被引用多次                                           ││
│  │ • CTE 计算代价高（大量聚合、复杂 Join）                    ││
│  │ • 结果集较小，物化后复用收益大                             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  内联 (Inline):                                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ • CTE 只被引用一次                                         ││
│  │ • CTE 计算简单                                             ││
│  │ • 主查询有 LIMIT，可以提前终止                             ││
│  │ • 内联后可以进行更多优化（谓词下推等）                     ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.6.3 内联实现

```cpp
// src/optimizer/cte_inlining.cpp

class CTEInliningOptimizer : public LogicalOperatorVisitor {
public:
    void VisitOperator(LogicalOperator &op) override {
        if (op.type == LogicalOperatorType::LOGICAL_CTE_REF) {
            auto &cte_ref = op.Cast<LogicalCTERef>();

            // 检查是否应该内联
            if (ShouldInline(cte_ref)) {
                // 用 CTE 定义替换引用
                InlineCTE(cte_ref);
            }
        }

        // 递归处理子节点
        VisitOperatorChildren(op);
    }

private:
    bool ShouldInline(LogicalCTERef &cte_ref) {
        // 获取 CTE 的引用计数
        auto ref_count = GetCTEReferenceCount(cte_ref.cte_index);

        // 只引用一次，一定内联
        if (ref_count == 1) {
            return true;
        }

        // 检查主查询是否有 LIMIT
        if (HasLimitAbove(cte_ref)) {
            // 有 LIMIT 时，内联可以提前终止
            return true;
        }

        // 多次引用，保持物化
        return false;
    }
};
```

### 8.6.4 LIMIT 场景的特殊处理

```cpp
bool CTEInliningOptimizer::HasLimitAbove(LogicalCTERef &cte_ref) {
    // 遍历祖先节点，查找 LIMIT
    LogicalOperator *current = &cte_ref;
    while (current->parent) {
        current = current->parent;

        if (current->type == LogicalOperatorType::LOGICAL_LIMIT) {
            auto &limit = current->Cast<LogicalLimit>();
            // 小 LIMIT 值时内联收益大
            if (limit.limit_val.has_value() &&
                limit.limit_val.value() < INLINE_THRESHOLD) {
                return true;
            }
        }
    }
    return false;
}
```

### 8.6.5 CTE 内联示例

```
┌─────────────────────────────────────────────────────────────────┐
│ CTE 内联示例                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始查询:                                                      │
│  WITH expensive_calc AS (                                      │
│      SELECT customer_id, SUM(amount) as total                  │
│      FROM orders                                               │
│      GROUP BY customer_id                                      │
│  )                                                             │
│  SELECT * FROM expensive_calc                                  │
│  ORDER BY total DESC                                           │
│  LIMIT 10;                                                     │
│                                                                 │
│  物化执行:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ 1. 执行 CTE，聚合全部 10M 订单                             ││
│  │ 2. 物化结果 (100K 客户)                                    ││
│  │ 3. 排序物化结果                                            ││
│  │ 4. 取前 10 条                                              ││
│  │                                                            ││
│  │ 代价: 全量聚合 + 全量排序                                  ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  内联执行:                                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ 计划:                                                      ││
│  │   TopN(10)                                                 ││
│  │      ↓                                                     ││
│  │   HashAggregate (customer_id, SUM(amount))                 ││
│  │      ↓                                                     ││
│  │   TableScan(orders)                                        ││
│  │                                                            ││
│  │ 优化后可以:                                                ││
│  │ - 使用 TopN 算子代替全量排序                               ││
│  │ - 只维护 10 个最大值的堆                                   ││
│  │                                                            ││
│  │ 代价: 全量聚合 + O(n log 10) TopN                          ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.7 LIMIT 下推优化

### 8.7.1 LIMIT 下推原理

将 LIMIT 算子下推到投影 (Projection) 之下，可以减少不必要的表达式计算：

```
┌─────────────────────────────────────────────────────────────────┐
│ LIMIT 下推原理                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始计划:                                                      │
│  SELECT complex_calculation(a, b, c) FROM t LIMIT 10;          │
│                                                                 │
│  ┌─────────────────┐                                           │
│  │    LIMIT 10     │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│  ┌────────┴────────┐                                           │
│  │   Projection    │  计算 complex_calculation                 │
│  │   (复杂表达式)   │  对全部 1M 行都计算！                     │
│  └────────┬────────┘                                           │
│           │                                                     │
│  ┌────────┴────────┐                                           │
│  │   TableScan     │                                           │
│  │   (1M rows)     │                                           │
│  └─────────────────┘                                           │
│                                                                 │
│  优化后:                                                        │
│  ┌─────────────────┐                                           │
│  │   Projection    │  只计算 10 行的表达式                      │
│  └────────┬────────┘                                           │
│           │                                                     │
│  ┌────────┴────────┐                                           │
│  │    LIMIT 10     │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│  ┌────────┴────────┐                                           │
│  │   TableScan     │  只扫描 10 行                              │
│  │   (提前终止)     │                                           │
│  └─────────────────┘                                           │
│                                                                 │
│  性能提升: 表达式计算从 1M 次降到 10 次                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.7.2 实现逻辑

```cpp
// src/optimizer/limit_pushdown.cpp

unique_ptr<LogicalOperator> LimitPushdownOptimizer::Optimize(
    unique_ptr<LogicalOperator> op) {

    if (op->type != LogicalOperatorType::LOGICAL_LIMIT) {
        // 递归处理子节点
        for (auto &child : op->children) {
            child = Optimize(std::move(child));
        }
        return op;
    }

    auto &limit = op->Cast<LogicalLimit>();

    // 检查是否可以下推
    if (!CanPushdownLimit(limit)) {
        return op;
    }

    // 获取子节点
    auto &child = limit.children[0];

    // 只对 Projection 下推
    if (child->type == LogicalOperatorType::LOGICAL_PROJECTION) {
        return PushdownThroughProjection(std::move(op));
    }

    return op;
}
```

### 8.7.3 下推条件检查

```cpp
bool LimitPushdownOptimizer::CanPushdownLimit(LogicalLimit &limit) {
    // 必须有具体的 LIMIT 值
    if (!limit.limit_val.has_value()) {
        return false;
    }

    // LIMIT 值必须足够小（默认阈值 8192）
    // 大 LIMIT 下推收益不大
    if (limit.limit_val.value() > LIMIT_PUSHDOWN_THRESHOLD) {
        return false;
    }

    // 不能有 OFFSET（OFFSET 需要先跳过行）
    if (limit.offset_val.has_value() && limit.offset_val.value() > 0) {
        return false;
    }

    return true;
}
```

### 8.7.4 交换 LIMIT 和 Projection

```cpp
unique_ptr<LogicalOperator> LimitPushdownOptimizer::PushdownThroughProjection(
    unique_ptr<LogicalOperator> op) {

    auto &limit = op->Cast<LogicalLimit>();
    auto &projection = limit.children[0]->Cast<LogicalProjection>();

    // 检查 Projection 的表达式是否安全
    // 不能下推过有副作用的表达式
    for (auto &expr : projection.expressions) {
        if (ExpressionHasSideEffects(*expr)) {
            return op;
        }
    }

    // 交换 LIMIT 和 Projection
    //
    // 原: LIMIT -> Projection -> Child
    // 新: Projection -> LIMIT -> Child

    auto child = std::move(projection.children[0]);
    projection.children[0] = std::move(op);  // LIMIT 变成 Projection 的子节点
    limit.children[0] = std::move(child);    // 原来的 Child 变成 LIMIT 的子节点

    return std::move(limit.children[0]->parent);  // 返回新的根 (Projection)
}
```

### 8.7.5 副作用检查

某些表达式有副作用，不能改变其执行次数：

```cpp
bool LimitPushdownOptimizer::ExpressionHasSideEffects(Expression &expr) {
    // 递归检查表达式树
    bool has_side_effects = false;

    ExpressionIterator::EnumerateChildren(expr, [&](Expression &child) {
        if (has_side_effects) return;

        switch (child.type) {
        case ExpressionType::BOUND_FUNCTION:
            // 检查函数是否有副作用
            auto &func = child.Cast<BoundFunctionExpression>();
            if (func.function.side_effects == FunctionSideEffects::HAS_SIDE_EFFECTS) {
                has_side_effects = true;
            }
            break;
        // RANDOM(), NOW() 等函数每次调用结果可能不同
        // 不能改变调用次数
        }
    });

    return has_side_effects;
}
```

---

## 8.8 其他特殊优化

### 8.8.1 TopN 优化

当 ORDER BY 后紧跟 LIMIT 时，可以使用更高效的 TopN 算法：

```
┌─────────────────────────────────────────────────────────────────┐
│ TopN 优化                                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始: ORDER BY + LIMIT                                        │
│  ┌────────────────────────────────────────────────────────────┐│
│  │    LIMIT 10                                                ││
│  │       ↓                                                    ││
│  │    ORDER BY amount DESC                                    ││
│  │       ↓                                                    ││
│  │    TableScan (1M rows)                                     ││
│  │                                                            ││
│  │    复杂度: O(n log n) 全量排序                             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  优化后: TopN                                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │    TopN(10, ORDER BY amount DESC)                          ││
│  │       ↓                                                    ││
│  │    TableScan (1M rows)                                     ││
│  │                                                            ││
│  │    算法: 维护大小为 10 的最小堆                            ││
│  │    复杂度: O(n log k) where k=10                           ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  性能对比 (1M 行，取前 10):                                    │
│  - 全量排序: O(1M × log(1M)) ≈ 20M 次比较                     │
│  - TopN:     O(1M × log(10)) ≈ 3.3M 次比较                    │
│  - 内存:     排序需要 O(n)，TopN 只需要 O(k)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.8.2 Distinct 消除

当列已知唯一时，DISTINCT 可以消除：

```cpp
// 检查是否可以消除 DISTINCT
bool CanEliminateDistinct(LogicalOperator &distinct) {
    // 获取 DISTINCT 列
    auto &columns = distinct.GetColumnBindings();

    // 检查是否已经唯一
    // 例如：主键列、已经 GROUP BY 的列
    for (auto &col : columns) {
        auto stats = GetColumnStatistics(col);
        if (stats && stats->IsUnique()) {
            continue;  // 该列唯一
        }
        return false;  // 无法确认唯一
    }

    // 所有列都唯一，可以消除 DISTINCT
    return true;
}
```

### 8.8.3 重复表达式规范化

确保相同表达式有一致的内部表示：

```cpp
// 规范化比较表达式
// 保证列引用在左边，常量在右边
void NormalizeComparisonExpression(BoundComparisonExpression &expr) {
    if (expr.left->type == ExpressionType::VALUE_CONSTANT &&
        expr.right->type == ExpressionType::BOUND_COLUMN_REF) {
        // 交换左右
        std::swap(expr.left, expr.right);
        // 翻转比较符号
        expr.type = FlipComparisonExpression(expr.type);
    }
}

// 规范化算术表达式
// a + 5 和 5 + a 统一为 a + 5
void NormalizeArithmeticExpression(BoundFunctionExpression &expr) {
    if (expr.function.name == "+" || expr.function.name == "*") {
        // 可交换操作，按字典序排列参数
        SortChildren(expr.children);
    }
}
```

---

## 8.9 优化器执行顺序

这些特殊优化在优化器管道中的执行位置：

```
┌─────────────────────────────────────────────────────────────────┐
│ 优化器执行顺序                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  早期阶段:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. Expression Rewriter (表达式简化)                     │   │
│  │ 2. Filter Pushdown (谓词下推)                           │   │
│  │ 3. CTE Inlining (CTE 内联决策)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  中期阶段:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. Join Order Optimization (Join 顺序优化)             │   │
│  │ 5. Build/Probe Side Optimizer (Build 端选择)           │   │
│  │ 6. Join Elimination (Join 消除)                         │   │
│  │ 7. Join Filter Pushdown (动态过滤准备)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  后期阶段:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 8. TopN Optimizer (TopN 融合)                           │   │
│  │ 9. Limit Pushdown (LIMIT 下推)                          │   │
│  │ 10. CSE Optimizer (公共子表达式消除)                    │   │
│  │ 11. Column Lifetime Analysis (列生命周期分析)          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  顺序设计原则:                                                  │
│  - 逻辑等价变换在前 (Filter Pushdown, Join Reorder)           │
│  - 代价相关优化在中 (Build Side, Join Filter)                  │
│  - 执行优化在后 (TopN, CSE, LIMIT)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8.10 本章小结

本章介绍了 DuckDB 优化器中的特殊优化技术：

| 优化技术 | 核心目标 | 关键条件 |
|---------|---------|---------|
| CSE | 避免重复计算 | 相同表达式出现多次 |
| Build/Probe | 减小哈希表 | Hash Join 两侧大小差异 |
| Join Filter | 减少扫描量 | 等值 Join + 整型键 |
| Join Elimination | 消除冗余 Join | 唯一键 + 列未使用 |
| CTE Inlining | 启用更多优化 | 单次引用或有 LIMIT |
| LIMIT Pushdown | 减少计算量 | 小 LIMIT + 无副作用 |

这些优化虽然各自独立，但在实际执行中会相互配合，共同提升查询性能。理解这些优化的原理和触发条件，有助于编写更高效的 SQL 查询。

---

## 附录：核心源文件索引

| 组件 | 文件路径 |
|------|----------|
| CSE Optimizer | `src/optimizer/cse_optimizer.cpp` |
| Build/Probe Optimizer | `src/optimizer/build_probe_side_optimizer.cpp` |
| Join Filter Pushdown | `src/optimizer/join_filter_pushdown_optimizer.cpp` |
| Join Elimination | `src/optimizer/join_elimination.cpp` |
| CTE Inlining | `src/optimizer/cte_inlining.cpp` |
| Limit Pushdown | `src/optimizer/limit_pushdown.cpp` |
| TopN Optimizer | `src/optimizer/topn_optimizer.cpp` |

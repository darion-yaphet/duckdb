# DuckDB 查询优化器深度解析：第五章 - 子查询优化

## 5.1 子查询优化概述

子查询是 SQL 中强大但复杂的特性。DuckDB 对子查询的处理分为两个主要阶段：

1. **规划阶段 (Planner)**: 在 Binder 中识别和分类子查询，生成初始逻辑计划
2. **优化阶段 (Optimizer)**: 通过去关联化 (Decorrelation) 和特殊优化提升执行效率

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        子查询处理流程                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SQL 子查询                                                              │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────┐                              │
│  │ 子查询分类                             │                              │
│  │ • Scalar Subquery (标量子查询)         │                              │
│  │ • EXISTS Subquery (存在性子查询)       │                              │
│  │ • ANY/IN Subquery (集合子查询)         │                              │
│  └────────────────┬─────────────────────┘                              │
│                   │                                                     │
│                   ▼                                                     │
│  ┌──────────────────────────────────────┐                              │
│  │ 关联性判断                             │                              │
│  │ • Uncorrelated: 独立执行               │                              │
│  │ • Correlated: 引用外部列               │                              │
│  └────────────────┬─────────────────────┘                              │
│                   │                                                     │
│       ┌───────────┴───────────┐                                        │
│       │                       │                                        │
│       ▼                       ▼                                        │
│  ┌─────────────┐      ┌─────────────────────┐                          │
│  │ Uncorrelated │      │ Correlated          │                          │
│  │ 添加 Cross   │      │ 创建 DependentJoin   │                          │
│  │ Product     │      │ 后续去关联化         │                          │
│  └─────────────┘      └──────────┬──────────┘                          │
│                                  │                                      │
│                                  ▼                                      │
│                       ┌─────────────────────┐                          │
│                       │ FlattenDependentJoin │                          │
│                       │ 去关联化优化          │                          │
│                       └──────────┬──────────┘                          │
│                                  │                                      │
│                                  ▼                                      │
│                       ┌─────────────────────┐                          │
│                       │ Deliminator         │                          │
│                       │ DelimJoin 优化       │                          │
│                       └─────────────────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.1.1 子查询类型

DuckDB 支持三种主要的子查询类型：

```cpp
// src/include/duckdb/planner/expression/bound_subquery_expression.hpp
enum class SubqueryType : uint8_t {
    INVALID = 0,
    SCALAR = 1,     // 标量子查询: (SELECT max(x) FROM t)
    EXISTS = 2,     // 存在性子查询: EXISTS (SELECT * FROM t WHERE ...)
    ANY = 3,        // ANY/IN 子查询: x IN (SELECT y FROM t)
    NOT_EXISTS = 4  // NOT EXISTS
};
```

**示例：**

```sql
-- 标量子查询 (SCALAR)
SELECT name, (SELECT MAX(price) FROM products) AS max_price
FROM products;

-- 存在性子查询 (EXISTS)
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- IN 子查询 (ANY)
SELECT * FROM products
WHERE category_id IN (SELECT id FROM categories WHERE active = true);
```

---

## 5.2 子查询规划 (PlanSubquery)

子查询的规划入口位于 `src/planner/binder/query_node/plan_subquery.cpp`。

### 5.2.1 主函数结构

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:328
unique_ptr<Expression> Binder::PlanSubquery(BoundSubqueryExpression &expr,
                                            unique_ptr<LogicalOperator> &root) {
    // 为子查询创建新的 Binder，共享根 Binder
    auto &subquery = expr.subquery;
    auto subquery_binder = Binder::CreateBinder(context, this);
    subquery_binder->plan_subquery = false;

    // 创建子查询的逻辑计划
    auto plan = subquery_binder->CreatePlan(*subquery);

    unique_ptr<Expression> result_expression;
    if (!expr.IsCorrelated()) {
        // 非关联子查询：直接添加 Cross Product
        result_expression = PlanUncorrelatedSubquery(*this, expr, root, std::move(plan));
    } else {
        // 关联子查询：创建 DependentJoin，后续去关联化
        result_expression = PlanCorrelatedSubquery(*this, expr, root, std::move(plan));
    }

    return result_expression;
}
```

### 5.2.2 非关联子查询处理

非关联子查询不引用外部查询的列，可以独立执行一次，然后将结果与外部查询进行连接。

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:38
static unique_ptr<Expression> PlanUncorrelatedSubquery(
    Binder &binder, BoundSubqueryExpression &expr,
    unique_ptr<LogicalOperator> &root, unique_ptr<LogicalOperator> plan) {

    D_ASSERT(!expr.IsCorrelated());

    switch (expr.subquery_type) {
    case SubqueryType::SCALAR: {
        // 标量子查询：添加 LIMIT 1 确保只返回一行
        // 然后用 Cross Product 连接
        auto limit = make_uniq<LogicalLimit>(BoundLimitNode::ConstantValue(1),
                                              BoundLimitNode());
        limit->AddChild(std::move(plan));
        plan = std::move(limit);

        // 创建 CROSS PRODUCT
        auto cross_product = LogicalCrossProduct::Create(std::move(root), std::move(plan));
        root = std::move(cross_product);

        // 返回引用子查询结果列的表达式
        return make_uniq<BoundColumnRefExpression>(
            expr.return_type,
            ColumnBinding(plan_table_index, 0));
    }

    case SubqueryType::EXISTS: {
        // EXISTS：使用 MARK Join
        auto mark_join = make_uniq<LogicalComparisonJoin>(JoinType::MARK);
        mark_join->AddChild(std::move(root));
        mark_join->AddChild(std::move(plan));
        mark_join->mark_index = mark_table_index;
        root = std::move(mark_join);

        return make_uniq<BoundColumnRefExpression>(
            LogicalType::BOOLEAN,
            ColumnBinding(mark_table_index, 0));
    }

    case SubqueryType::ANY: {
        // ANY/IN：使用 MARK Join 或 SINGLE Join
        // 根据比较类型决定 Join 策略
        // ...
    }
    }
}
```

**非关联标量子查询转换示例：**

```sql
-- 原始查询
SELECT name, (SELECT MAX(price) FROM products) AS max_price
FROM customers;

-- 转换后的逻辑计划
┌─────────────────────┐
│ Projection          │
│ [name, $scalar]     │
├─────────────────────┤
│ Cross Product       │
├─────────────────────┤
│ ├── Scan customers  │
│ └── Limit 1         │
│     └── Aggregate   │
│         MAX(price)  │
│         └── Scan    │
│             products│
└─────────────────────┘
```

### 5.2.3 关联子查询处理

关联子查询引用外部查询的列，需要特殊处理。DuckDB 创建 `LogicalDependentJoin` 表示这种依赖关系。

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:177
static unique_ptr<Expression> PlanCorrelatedSubquery(
    Binder &binder, BoundSubqueryExpression &expr,
    unique_ptr<LogicalOperator> &root, unique_ptr<LogicalOperator> plan) {

    auto &correlated_columns = expr.binder->correlated_columns;

    // 为每个关联列添加投影
    D_ASSERT(!expr.IsCorrelated() || !correlated_columns.empty());

    switch (expr.subquery_type) {
    case SubqueryType::SCALAR: {
        // 创建 DependentJoin (LEFT OUTER)
        auto dependent_join = make_uniq<LogicalDependentJoin>(JoinType::LEFT);
        dependent_join->AddChild(std::move(root));
        dependent_join->AddChild(std::move(plan));

        // 记录关联列
        for (auto &corr : correlated_columns) {
            dependent_join->correlated_columns.push_back(corr);
        }

        root = std::move(dependent_join);

        return make_uniq<BoundColumnRefExpression>(
            expr.return_type,
            ColumnBinding(plan_table_index, 0));
    }

    case SubqueryType::EXISTS: {
        // EXISTS 使用 MARK 类型的 DependentJoin
        auto dependent_join = make_uniq<LogicalDependentJoin>(JoinType::MARK);
        // ...
    }

    case SubqueryType::ANY: {
        // IN/ANY 使用 MARK 或 SINGLE 类型
        // ...
    }
    }
}
```

**关联列结构：**

```cpp
// src/include/duckdb/planner/binder.hpp
struct CorrelatedColumnInfo {
    ColumnBinding binding;    // 外部列的绑定
    LogicalType type;         // 列类型
    string name;              // 列名
    idx_t depth;              // 嵌套深度
};
```

---

## 5.3 去关联化 (Flatten Dependent Join)

去关联化是子查询优化的核心，将嵌套循环式的关联子查询转换为高效的 Join 操作。

### 5.3.1 FlattenDependentJoins 类

```cpp
// src/include/duckdb/planner/subquery/flatten_dependent_join.hpp
class FlattenDependentJoins : public LogicalOperatorVisitor {
public:
    // 执行去关联化
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> op);

private:
    Binder &binder;

    // 替换关联表达式中的外部列引用
    bool ReplaceCorrelatedExpressions(unique_ptr<LogicalOperator> &op);

    // 替换关联子查询的关联列
    void RewriteCorrelatedExpressions(vector<CorrelatedColumnInfo> &correlated_columns,
                                      unique_ptr<LogicalOperator> &op);

    // 递归地扁平化 DependentJoin
    unique_ptr<LogicalOperator> PushDownDependentJoin(unique_ptr<LogicalOperator> op);
};
```

### 5.3.2 去关联化算法

去关联化的核心思想是：

1. 识别子查询中引用的外部列
2. 将外部列的值通过 Delim Join 传递给子查询
3. 将原来的嵌套循环执行转换为一次性的 Join 操作

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:40
unique_ptr<LogicalOperator> FlattenDependentJoins::Optimize(
    unique_ptr<LogicalOperator> op) {

    // 1. 首先处理子树中的 DependentJoin
    for (auto &child : op->children) {
        child = Optimize(std::move(child));
    }

    // 2. 如果当前节点是 DependentJoin，进行去关联化
    if (op->type == LogicalOperatorType::LOGICAL_DEPENDENT_JOIN) {
        auto &dependent_join = op->Cast<LogicalDependentJoin>();

        // 收集关联列
        vector<CorrelatedColumnInfo> correlated_columns;
        for (auto &corr : dependent_join.correlated_columns) {
            correlated_columns.push_back(corr);
        }

        // 在子计划中替换关联列引用
        ReplaceCorrelatedExpressions(op->children[1], correlated_columns);

        // 创建 Delim Join
        return CreateDelimJoin(std::move(op), correlated_columns);
    }

    return op;
}
```

### 5.3.3 Delim Join 创建

`LogicalDelimJoin` 是 DuckDB 用于处理去关联化子查询的特殊 Join 类型。它的工作原理是：

1. **Build 端**: 扫描外部查询，提取关联列的不同值
2. **Probe 端**: 将这些值传递给子查询执行

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:200
unique_ptr<LogicalOperator> FlattenDependentJoins::CreateDelimJoin(
    unique_ptr<LogicalOperator> op,
    vector<CorrelatedColumnInfo> &correlated_columns) {

    auto &dependent_join = op->Cast<LogicalDependentJoin>();

    // 创建 DelimJoin
    auto delim_join = make_uniq<LogicalDelimJoin>(dependent_join.join_type);

    // 左侧是外部查询
    delim_join->AddChild(std::move(dependent_join.children[0]));

    // 右侧是子查询
    delim_join->AddChild(std::move(dependent_join.children[1]));

    // 设置关联列 - 这些列将被 "delimited" (划界)
    for (auto &corr : correlated_columns) {
        delim_join->delim_bindings.push_back(corr.binding);
    }

    // 添加 DelimGet 算子来获取划界的值
    auto delim_get = make_uniq<LogicalDelimGet>(delim_scan_idx, delim_types);

    // 将子查询中的关联列引用替换为 DelimGet 的输出
    // ...

    return delim_join;
}
```

### 5.3.4 去关联化转换示例

```sql
-- 原始关联子查询
SELECT c.name,
       (SELECT SUM(o.amount)
        FROM orders o
        WHERE o.customer_id = c.id) AS total
FROM customers c;

-- 原始执行方式 (概念上的嵌套循环)
for each customer c:
    execute: SELECT SUM(o.amount) FROM orders o WHERE o.customer_id = c.id
    output: c.name, result

-- 去关联化后
SELECT c.name, agg.total
FROM customers c
LEFT JOIN (
    SELECT o.customer_id, SUM(o.amount) AS total
    FROM orders o
    GROUP BY o.customer_id
) agg ON agg.customer_id = c.id;
```

**去关联化后的逻辑计划：**

```
┌─────────────────────────────────────┐
│ Projection [c.name, agg.total]      │
├─────────────────────────────────────┤
│ DelimJoin (LEFT)                    │
│ delim_bindings: [c.id]              │
├─────────────────────────────────────┤
│ ├── Scan customers                  │
│ └── Aggregate [customer_id]         │
│     SUM(amount)                     │
│     ├── Filter: customer_id = $delim│
│     └── Scan orders                 │
│                                     │
│ (子查询中的 c.id 被替换为 DelimGet) │
└─────────────────────────────────────┘
```

### 5.3.5 关联表达式替换

去关联化的关键步骤是将子查询中对外部列的引用替换为 `DelimGet` 的输出：

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:340
void FlattenDependentJoins::RewriteCorrelatedExpressions(
    vector<CorrelatedColumnInfo> &correlated_columns,
    unique_ptr<LogicalOperator> &op) {

    // 创建表达式重写器
    RewriteCorrelatedExpressionRewriter rewriter(correlated_columns, delim_get_bindings);

    // 遍历所有表达式并替换
    for (auto &expr : op->expressions) {
        rewriter.VisitExpression(&expr);
    }

    // 递归处理子节点
    for (auto &child : op->children) {
        RewriteCorrelatedExpressions(correlated_columns, child);
    }
}

// 重写器将 ColumnRef(外部列) 替换为 ColumnRef(DelimGet输出)
class RewriteCorrelatedExpressionRewriter : public ExpressionVisitor {
    void VisitReplace(BoundColumnRefExpression &expr,
                      unique_ptr<Expression> *expr_ptr) override {
        // 检查是否是关联列
        for (idx_t i = 0; i < correlated_columns.size(); i++) {
            if (expr.binding == correlated_columns[i].binding) {
                // 替换为 DelimGet 的输出列
                *expr_ptr = make_uniq<BoundColumnRefExpression>(
                    expr.return_type,
                    delim_get_bindings[i]);
                return;
            }
        }
    }
};
```

### 5.3.6 递归去关联化

对于嵌套的关联子查询，需要递归处理：

```cpp
// src/planner/subquery/flatten_dependent_join.cpp:100
unique_ptr<LogicalOperator> FlattenDependentJoins::PushDownDependentJoin(
    unique_ptr<LogicalOperator> op) {

    // 处理不同类型的算子
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY: {
        // 聚合算子：将分组列添加到 DelimGet
        auto &aggregate = op->Cast<LogicalAggregate>();

        // 关联列需要被添加到分组键中
        for (auto &corr : correlated_columns) {
            aggregate.groups.push_back(
                make_uniq<BoundColumnRefExpression>(corr.type, corr.binding));
        }

        // 递归处理子节点
        op->children[0] = PushDownDependentJoin(std::move(op->children[0]));
        return op;
    }

    case LogicalOperatorType::LOGICAL_FILTER: {
        // 过滤算子：直接向下传递
        op->children[0] = PushDownDependentJoin(std::move(op->children[0]));
        return op;
    }

    case LogicalOperatorType::LOGICAL_GET: {
        // 扫描算子：添加 Cross Product 与 DelimGet
        auto delim_get = make_uniq<LogicalDelimGet>(delim_scan_idx, delim_types);
        auto cross_product = LogicalCrossProduct::Create(
            std::move(delim_get), std::move(op));
        return cross_product;
    }

    // 其他算子类型...
    }
}
```

---

## 5.4 Deliminator 优化

`Deliminator` 是针对 `DelimJoin` 的专门优化器，目标是消除不必要的 DelimJoin 或将其转换为更高效的形式。

### 5.4.1 Deliminator 类

```cpp
// src/include/duckdb/optimizer/deliminator.hpp
class Deliminator {
public:
    // 入口函数
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> op);

private:
    // 查找并优化 DelimJoin
    bool HasDelimJoin(LogicalOperator &op);

    // 尝试消除 DelimJoin
    bool TryEliminateDelimJoin(LogicalOperator &op);

    // 将 DelimJoin 转换为普通 Join
    unique_ptr<LogicalOperator> ConvertDelimJoinToRegularJoin(
        unique_ptr<LogicalOperator> op);
};
```

### 5.4.2 优化策略

Deliminator 执行多种优化：

**1. 移除冗余的 DelimGet + Join 组合**

当 DelimJoin 的右侧只是简单地将 DelimGet 与表进行 Join 时，可以直接转换为普通 Join：

```cpp
// src/optimizer/deliminator.cpp:50
bool Deliminator::TryEliminateDelimJoin(LogicalOperator &op) {
    if (op.type != LogicalOperatorType::LOGICAL_DELIM_JOIN) {
        return false;
    }

    auto &delim_join = op.Cast<LogicalDelimJoin>();

    // 检查右侧是否可以消除 DelimGet
    auto &right = delim_join.children[1];

    // 模式匹配：寻找 DelimGet + Join 的组合
    if (CanEliminateDelimGet(right)) {
        // 将 DelimJoin 转换为普通 Join
        // DelimGet 的作用被 Join 条件替代
        ConvertToRegularJoin(delim_join);
        return true;
    }

    return false;
}
```

**2. Join 条件提取**

从 DelimJoin 中提取等值条件，用于后续的 Join 优化：

```cpp
// src/optimizer/deliminator.cpp:120
void Deliminator::ExtractJoinConditions(LogicalDelimJoin &delim_join) {
    // DelimJoin 的 delim_bindings 定义了等值关系
    for (idx_t i = 0; i < delim_join.delim_bindings.size(); i++) {
        auto &delim_binding = delim_join.delim_bindings[i];

        // 在右侧找到对应的 DelimGet 引用
        auto delim_get_binding = FindDelimGetBinding(delim_join.children[1], i);

        if (delim_get_binding.IsValid()) {
            // 创建等值条件: outer.col = inner.col
            auto condition = make_uniq<BoundComparisonExpression>(
                ExpressionType::COMPARE_EQUAL,
                make_uniq<BoundColumnRefExpression>(delim_binding),
                make_uniq<BoundColumnRefExpression>(delim_get_binding));

            delim_join.conditions.push_back(std::move(condition));
        }
    }
}
```

### 5.4.3 转换示例

```
-- 优化前 (DelimJoin)
┌─────────────────────────────────────┐
│ DelimJoin (LEFT)                    │
│ delim: [c.id]                       │
├─────────────────────────────────────┤
│ ├── Scan customers (c)              │
│ └── Hash Join                       │
│     ├── DelimGet [id]               │
│     └── Aggregate [customer_id]     │
│         └── Scan orders             │
└─────────────────────────────────────┘

-- 优化后 (普通 Left Join)
┌─────────────────────────────────────┐
│ Left Hash Join                      │
│ c.id = agg.customer_id              │
├─────────────────────────────────────┤
│ ├── Scan customers (c)              │
│ └── Aggregate [customer_id]         │
│     └── Scan orders                 │
└─────────────────────────────────────┘
```

---

## 5.5 IN 子句重写

`InClauseRewriter` 针对大型 IN 列表进行优化，将其转换为 MARK Join。

### 5.5.1 为什么需要 IN 重写？

```sql
-- 大型 IN 列表效率低下
SELECT * FROM orders
WHERE customer_id IN (1, 2, 3, ..., 1000);

-- 问题:
-- 1. 每行需要进行 1000 次比较
-- 2. IN 表达式无法利用索引
-- 3. 计划复杂度高
```

### 5.5.2 InClauseRewriter 实现

```cpp
// src/optimizer/in_clause_rewriter.cpp:15
class InClauseRewriter : public LogicalOperatorVisitor {
public:
    // 阈值：超过此数量的常量将被重写为 Join
    static constexpr idx_t IN_CLAUSE_REWRITE_THRESHOLD = 5;

    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op);

private:
    // 收集需要重写的 IN 表达式
    void ExtractInClauses(LogicalOperator &op);

    // 创建常量表和 MARK Join
    unique_ptr<LogicalOperator> CreateMarkJoin(
        unique_ptr<LogicalOperator> op,
        BoundOperatorExpression &in_expr);
};
```

### 5.5.3 重写算法

```cpp
// src/optimizer/in_clause_rewriter.cpp:50
unique_ptr<LogicalOperator> InClauseRewriter::CreateMarkJoin(
    unique_ptr<LogicalOperator> op,
    BoundOperatorExpression &in_expr) {

    // 1. 提取 IN 列表中的常量值
    vector<Value> values;
    for (idx_t i = 1; i < in_expr.children.size(); i++) {
        auto &child = in_expr.children[i];
        if (child->type == ExpressionType::VALUE_CONSTANT) {
            values.push_back(child->Cast<BoundConstantExpression>().value);
        }
    }

    // 2. 创建 ColumnDataCollection 存储常量
    auto collection = make_uniq<ColumnDataCollection>(context, types);
    DataChunk chunk;
    chunk.Initialize(context, types);

    for (auto &value : values) {
        chunk.SetValue(0, chunk.size(), value);
        chunk.SetCardinality(chunk.size() + 1);

        if (chunk.size() >= STANDARD_VECTOR_SIZE) {
            collection->Append(chunk);
            chunk.Reset();
        }
    }
    if (chunk.size() > 0) {
        collection->Append(chunk);
    }

    // 3. 创建 LogicalColumnDataGet 算子
    auto column_data_get = make_uniq<LogicalColumnDataGet>(
        table_index, types, std::move(collection));

    // 4. 创建 MARK Join
    auto mark_join = make_uniq<LogicalComparisonJoin>(JoinType::MARK);
    mark_join->AddChild(std::move(op));
    mark_join->AddChild(std::move(column_data_get));

    // 5. 设置 Join 条件
    auto comparison = make_uniq<BoundComparisonExpression>(
        ExpressionType::COMPARE_EQUAL,
        std::move(in_expr.children[0]),  // 被比较的列
        make_uniq<BoundColumnRefExpression>(  // IN 列表的值
            types[0], ColumnBinding(table_index, 0)));

    mark_join->conditions.push_back(JoinCondition::CreateFromExpression(
        std::move(comparison)));

    mark_join->mark_index = mark_index;

    return mark_join;
}
```

### 5.5.4 重写示例

```sql
-- 原始查询
SELECT * FROM orders
WHERE customer_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

-- 重写后的逻辑计划
┌─────────────────────────────────────┐
│ Filter: mark_column = true          │
├─────────────────────────────────────┤
│ MARK Join                           │
│ customer_id = $const_col            │
├─────────────────────────────────────┤
│ ├── Scan orders                     │
│ └── ColumnDataGet                   │
│     values: [1,2,3,4,5,6,7,8,9,10] │
└─────────────────────────────────────┘
```

### 5.5.5 MARK Join 的优势

MARK Join 相比原始 IN 表达式有以下优势：

1. **Hash Table 加速**: 将 O(n*m) 的比较降为 O(n) 的哈希查找
2. **内存效率**: 常量存储在 ColumnDataCollection 中，可以 spill 到磁盘
3. **NULL 处理**: MARK Join 正确处理 NULL 值语义
4. **可并行**: Hash Join 可以并行执行

---

## 5.6 特殊子查询优化

### 5.6.1 ANY 子查询与比较操作

ANY 子查询支持多种比较操作符：

```sql
-- 等值 (转换为 SEMI Join 或 MARK Join)
SELECT * FROM t1 WHERE x = ANY(SELECT y FROM t2);

-- 不等 (需要特殊处理)
SELECT * FROM t1 WHERE x > ANY(SELECT y FROM t2);
-- 等价于: x > MIN(SELECT y FROM t2)
```

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:240
static unique_ptr<Expression> PlanAnySubquery(
    Binder &binder, BoundSubqueryExpression &expr,
    unique_ptr<LogicalOperator> &root, unique_ptr<LogicalOperator> plan) {

    auto comparison_type = expr.comparison_type;

    if (comparison_type == ExpressionType::COMPARE_EQUAL) {
        // 等值比较：使用 MARK Join
        auto mark_join = make_uniq<LogicalComparisonJoin>(JoinType::MARK);
        // ...
    } else {
        // 非等值比较：转换为聚合 + 比较
        // x > ANY(subquery) 等价于 x > (SELECT MIN(y) FROM subquery)
        // x < ANY(subquery) 等价于 x < (SELECT MAX(y) FROM subquery)

        if (comparison_type == ExpressionType::COMPARE_GREATERTHAN ||
            comparison_type == ExpressionType::COMPARE_GREATERTHANOREQUALTO) {
            // 使用 MIN 聚合
            auto min_aggregate = CreateMinAggregate(plan);
            // ...
        } else if (comparison_type == ExpressionType::COMPARE_LESSTHAN ||
                   comparison_type == ExpressionType::COMPARE_LESSTHANOREQUALTO) {
            // 使用 MAX 聚合
            auto max_aggregate = CreateMaxAggregate(plan);
            // ...
        }
    }
}
```

### 5.6.2 NOT EXISTS 优化

NOT EXISTS 子查询转换为 ANTI Join：

```sql
-- 原始查询
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- 转换为 ANTI Join
SELECT c.*
FROM customers c
ANTI JOIN orders o ON o.customer_id = c.id;
```

```cpp
// src/planner/binder/query_node/plan_subquery.cpp:300
if (expr.subquery_type == SubqueryType::NOT_EXISTS) {
    // NOT EXISTS 使用 ANTI Join
    if (expr.IsCorrelated()) {
        auto dependent_join = make_uniq<LogicalDependentJoin>(JoinType::ANTI);
        // ...
    } else {
        auto anti_join = make_uniq<LogicalComparisonJoin>(JoinType::ANTI);
        // ...
    }
}
```

### 5.6.3 标量子查询的 SINGLE Join

对于需要确保子查询返回单个值的场景，DuckDB 使用 SINGLE Join：

```cpp
// SINGLE Join 确保右侧最多返回一行
// 如果返回多行，则抛出错误

enum class JoinType : uint8_t {
    // ...
    SINGLE = 7,  // 标量子查询使用
    // ...
};
```

---

## 5.7 完整优化流程示例

让我们通过一个复杂的例子来展示完整的子查询优化流程：

```sql
-- 原始查询：找出购买金额超过平均值的客户
SELECT c.name
FROM customers c
WHERE (SELECT SUM(o.amount) FROM orders o WHERE o.customer_id = c.id)
      > (SELECT AVG(amount) FROM orders);
```

### 阶段 1: 解析和绑定

```
两个子查询被识别:
1. 关联标量子查询: SELECT SUM(o.amount) FROM orders o WHERE o.customer_id = c.id
   - 关联列: c.id

2. 非关联标量子查询: SELECT AVG(amount) FROM orders
   - 无关联列
```

### 阶段 2: 初始逻辑计划

```
┌────────────────────────────────────────────────┐
│ Projection [c.name]                            │
├────────────────────────────────────────────────┤
│ Filter: $subquery1 > $subquery2                │
├────────────────────────────────────────────────┤
│ Cross Product                                  │
├────────────────────────────────────────────────┤
│ ├── DependentJoin (LEFT)                       │
│ │   correlated: [c.id]                         │
│ │   ├── Scan customers                         │
│ │   └── Aggregate SUM(amount)                  │
│ │       └── Filter: customer_id = c.id         │
│ │           └── Scan orders                    │
│ │                                              │
│ └── Limit 1                                    │
│     └── Aggregate AVG(amount)                  │
│         └── Scan orders                        │
└────────────────────────────────────────────────┘
```

### 阶段 3: 去关联化 (FlattenDependentJoin)

```
┌────────────────────────────────────────────────┐
│ Projection [c.name]                            │
├────────────────────────────────────────────────┤
│ Filter: sum_amount > avg_amount                │
├────────────────────────────────────────────────┤
│ Cross Product                                  │
├────────────────────────────────────────────────┤
│ ├── DelimJoin (LEFT)                           │
│ │   delim: [c.id]                              │
│ │   ├── Scan customers                         │
│ │   └── Aggregate [customer_id] SUM(amount)    │
│ │       └── Join (customer_id = $delim)        │
│ │           ├── DelimGet [id]                  │
│ │           └── Scan orders                    │
│ │                                              │
│ └── Aggregate AVG(amount)                      │
│     └── Scan orders                            │
└────────────────────────────────────────────────┘
```

### 阶段 4: Deliminator 优化

```
┌────────────────────────────────────────────────┐
│ Projection [c.name]                            │
├────────────────────────────────────────────────┤
│ Filter: sum_amount > avg_amount                │
├────────────────────────────────────────────────┤
│ Cross Product                                  │
├────────────────────────────────────────────────┤
│ ├── Left Hash Join (c.id = customer_id)        │
│ │   ├── Scan customers                         │
│ │   └── Aggregate [customer_id] SUM(amount)    │
│ │       └── Scan orders                        │
│ │                                              │
│ └── Aggregate AVG(amount)                      │
│     └── Scan orders                            │
└────────────────────────────────────────────────┘
```

### 阶段 5: 进一步优化

经过 Filter Pushdown 和 Common Subexpression Elimination 后：

```
┌────────────────────────────────────────────────┐
│ Projection [c.name]                            │
├────────────────────────────────────────────────┤
│ Filter: sum_amount > avg_amount                │
├────────────────────────────────────────────────┤
│ Cross Product                                  │
├────────────────────────────────────────────────┤
│ ├── Left Hash Join (c.id = customer_id)        │
│ │   ├── Scan customers                         │
│ │   └── Aggregate [customer_id] SUM(amount)    │  -- 复用扫描
│ │       └─┐                                    │
│ │         │                                    │
│ └── Aggregate AVG(amount)                      │
│     └─────┴── Scan orders (共享)               │
└────────────────────────────────────────────────┘
```

---

## 5.8 源码索引

| 组件 | 文件路径 | 主要功能 |
|------|----------|----------|
| 子查询规划 | `src/planner/binder/query_node/plan_subquery.cpp` | 子查询分类和初始计划生成 |
| 去关联化 | `src/planner/subquery/flatten_dependent_join.cpp` | DependentJoin 转换为 DelimJoin |
| Deliminator | `src/optimizer/deliminator.cpp` | DelimJoin 优化和消除 |
| IN 重写 | `src/optimizer/in_clause_rewriter.cpp` | 大型 IN 列表转换为 MARK Join |
| DependentJoin | `src/planner/operator/logical_dependent_join.cpp` | 关联 Join 算子定义 |
| DelimJoin | `src/planner/operator/logical_delim_join.cpp` | Delim Join 算子定义 |
| DelimGet | `src/planner/operator/logical_delim_get.cpp` | Delim 值获取算子 |

---

## 5.9 设计要点总结

### 5.9.1 子查询处理原则

1. **尽早识别关联性**: 在绑定阶段就确定子查询是否关联
2. **统一表示**: 所有关联子查询先转换为 DependentJoin
3. **延迟去关联化**: 在优化阶段统一执行去关联化
4. **保持正确性**: 严格处理 NULL 语义和空集情况

### 5.9.2 去关联化策略

1. **DelimJoin 作为中间表示**: 提供统一的去关联化框架
2. **自底向上处理**: 先处理最内层的子查询
3. **条件传播**: 将外部列值通过 DelimGet 传递给内部
4. **最终消除**: 通过 Deliminator 将 DelimJoin 转换为普通 Join

### 5.9.3 性能考量

1. **避免嵌套循环**: 去关联化将 O(n*m) 变为 O(n+m)
2. **利用 Hash Join**: 转换后可以使用高效的 Hash Join
3. **复用扫描**: 相同表的多次扫描可以合并
4. **IN 列表优化**: 大型 IN 列表通过 MARK Join 处理

### 5.9.4 与其他优化的交互

- **谓词下推**: 去关联化后，条件可以进一步下推
- **Join 重排序**: 转换后的 Join 可以参与 Join Order 优化
- **聚合优化**: 子查询中的聚合可以被其他聚合优化处理

---

## 5.10 本章小结

本章详细介绍了 DuckDB 的子查询优化机制：

1. **子查询分类**: Scalar、EXISTS、ANY/IN 三种类型，以及关联/非关联的区分
2. **规划阶段**: 非关联子查询直接添加 Cross Product，关联子查询创建 DependentJoin
3. **去关联化**: FlattenDependentJoins 将 DependentJoin 转换为 DelimJoin
4. **Deliminator**: 进一步优化 DelimJoin，消除不必要的 DelimGet
5. **IN 重写**: 大型 IN 列表转换为高效的 MARK Join

子查询优化是查询优化器中最复杂的部分之一，DuckDB 通过精心设计的分层处理架构，将复杂的子查询转换为高效的 Join 操作，显著提升了查询性能。

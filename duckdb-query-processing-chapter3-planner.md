# DuckDB 查询处理深度解析：第三章 - Planner（逻辑计划生成）

## 本章概述

Planner 是 DuckDB 查询处理流程中的关键组件，负责将 Binder 生成的 BoundStatement 转换为 LogicalOperator 树。这个过程包括构建算子树、处理子查询、规划 JOIN 等复杂操作。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Planner 在查询处理中的位置                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   BoundStatement (Binder Output)                                            │
│        │                                                                    │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                         Planner                                  │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  1. 构建 LogicalOperator 树                                   │ │       │
│   │  │     - LogicalGet (表扫描)                                     │ │       │
│   │  │     - LogicalFilter (过滤)                                    │ │       │
│   │  │     - LogicalJoin (连接)                                      │ │       │
│   │  │     - LogicalAggregate (聚合)                                 │ │       │
│   │  │     - LogicalProjection (投影)                                │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  2. 处理子查询 (PlanSubqueries)                               │ │       │
│   │  │     - 非相关子查询 → CrossProduct                             │ │       │
│   │  │     - 相关子查询 → DependentJoin                              │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  3. 处理修饰符 (Modifiers)                                    │ │       │
│   │  │     - ORDER BY → LogicalOrder                                 │ │       │
│   │  │     - LIMIT → LogicalLimit                                    │ │       │
│   │  │     - DISTINCT → LogicalDistinct                              │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │                                                                    │
│        ▼                                                                    │
│   LogicalOperator Tree (Planner Output)                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.1 Planner 类设计

### 3.1.1 Planner 类

```cpp
// src/include/duckdb/planner/planner.hpp

class Planner {
    friend class Binder;

public:
    explicit Planner(ClientContext &context);

public:
    //! 最终的逻辑计划
    unique_ptr<LogicalOperator> plan;
    //! 结果列名
    vector<string> names;
    //! 结果类型
    vector<LogicalType> types;
    //! 参数数据映射
    case_insensitive_map_t<BoundParameterData> parameter_data;

    //! Binder 实例
    shared_ptr<Binder> binder;
    //! 客户端上下文
    ClientContext &context;

    //! 语句属性
    StatementProperties properties;
    //! 参数值映射
    bound_parameter_map_t value_map;

public:
    //! 创建逻辑计划
    void CreatePlan(unique_ptr<SQLStatement> statement);
    //! 验证计划正确性
    static void VerifyPlan(ClientContext &context, unique_ptr<LogicalOperator> &op,
                           optional_ptr<bound_parameter_map_t> map = nullptr);

private:
    void CreatePlan(SQLStatement &statement);
    shared_ptr<PreparedStatementData> PrepareSQLStatement(unique_ptr<SQLStatement> statement);
};
```

### 3.1.2 计划生成流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Planner 工作流程                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SQLStatement                                                              │
│        │                                                                    │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  Planner::CreatePlan(SQLStatement)                              │       │
│   │     │                                                           │       │
│   │     ▼                                                           │       │
│   │  binder = Binder::CreateBinder(context)                         │       │
│   │     │                                                           │       │
│   │     ▼                                                           │       │
│   │  binder->Bind(statement)                                        │       │
│   │     - 符号解析                                                   │       │
│   │     - 类型推导                                                   │       │
│   │     - 语义检查                                                   │       │
│   │     - 逻辑计划构建 (在 Binder 内部完成)                           │       │
│   │     │                                                           │       │
│   │     ▼                                                           │       │
│   │  BoundStatement { plan, types, names }                          │       │
│   │     │                                                           │       │
│   │     ▼                                                           │       │
│   │  plan = std::move(bound.plan)                                   │       │
│   │  types = std::move(bound.types)                                 │       │
│   │  names = std::move(bound.names)                                 │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │                                                                    │
│        ▼                                                                    │
│   LogicalOperator Tree                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.2 LogicalOperator - 逻辑算子基类

### 3.2.1 LogicalOperator 类设计

```cpp
// src/include/duckdb/planner/logical_operator.hpp

class LogicalOperator {
public:
    explicit LogicalOperator(LogicalOperatorType type);
    LogicalOperator(LogicalOperatorType type, vector<unique_ptr<Expression>> expressions);
    virtual ~LogicalOperator();

    //! 算子类型
    LogicalOperatorType type;
    //! 子算子列表
    vector<unique_ptr<LogicalOperator>> children;
    //! 算子包含的表达式
    vector<unique_ptr<Expression>> expressions;
    //! 返回类型（由 ResolveTypes() 设置）
    vector<LogicalType> types;
    //! 估计基数
    idx_t estimated_cardinality;
    bool has_estimated_cardinality;

public:
    //! 获取列绑定信息
    virtual vector<ColumnBinding> GetColumnBindings();
    //! 获取根索引
    virtual idx_t GetRootIndex();
    //! 生成列绑定
    static vector<ColumnBinding> GenerateColumnBindings(idx_t table_idx, idx_t column_count);

    //! 解析算子及子算子的类型
    void ResolveOperatorTypes();

    //! 获取算子名称
    virtual string GetName() const;
    //! 转换为字符串表示
    virtual string ToString(ExplainFormat format = ExplainFormat::DEFAULT) const;

    //! 添加子算子
    void AddChild(unique_ptr<LogicalOperator> child);
    //! 估计基数
    virtual idx_t EstimateCardinality(ClientContext &context);

    //! 序列化/反序列化
    virtual void Serialize(Serializer &serializer) const;
    static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);

    //! 复制算子
    virtual unique_ptr<LogicalOperator> Copy(ClientContext &context) const;

    //! 获取算子关联的表索引
    virtual vector<idx_t> GetTableIndex() const;

protected:
    //! 解析此算子的类型（子类实现）
    virtual void ResolveTypes() = 0;
};
```

### 3.2.2 LogicalOperatorType 枚举

```cpp
// src/include/duckdb/common/enums/logical_operator_type.hpp

enum class LogicalOperatorType : uint8_t {
    LOGICAL_INVALID = 0,

    // 基础算子
    LOGICAL_PROJECTION = 1,          // 投影
    LOGICAL_FILTER = 2,              // 过滤
    LOGICAL_AGGREGATE_AND_GROUP_BY = 3,  // 聚合
    LOGICAL_WINDOW = 4,              // 窗口函数
    LOGICAL_UNNEST = 5,              // 展开数组
    LOGICAL_LIMIT = 6,               // LIMIT
    LOGICAL_ORDER_BY = 7,            // ORDER BY
    LOGICAL_TOP_N = 8,               // TopN（ORDER BY + LIMIT 优化）
    LOGICAL_DISTINCT = 11,           // DISTINCT
    LOGICAL_SAMPLE = 12,             // SAMPLE

    // 数据源
    LOGICAL_GET = 25,                // 表扫描
    LOGICAL_CHUNK_GET = 26,          // 数据块扫描
    LOGICAL_DELIM_GET = 27,          // 分隔符扫描
    LOGICAL_EXPRESSION_GET = 28,     // 表达式扫描
    LOGICAL_DUMMY_SCAN = 29,         // 虚拟扫描
    LOGICAL_EMPTY_RESULT = 30,       // 空结果
    LOGICAL_CTE_REF = 31,            // CTE 引用

    // 连接
    LOGICAL_JOIN = 50,               // 基础连接
    LOGICAL_DELIM_JOIN = 51,         // 分隔符连接
    LOGICAL_COMPARISON_JOIN = 52,    // 比较连接
    LOGICAL_ANY_JOIN = 53,           // 任意条件连接
    LOGICAL_CROSS_PRODUCT = 54,      // 笛卡尔积
    LOGICAL_POSITIONAL_JOIN = 55,    // 位置连接
    LOGICAL_ASOF_JOIN = 56,          // ASOF 连接
    LOGICAL_DEPENDENT_JOIN = 57,     // 相关子查询连接

    // 集合操作
    LOGICAL_UNION = 75,              // UNION
    LOGICAL_EXCEPT = 76,             // EXCEPT
    LOGICAL_INTERSECT = 77,          // INTERSECT
    LOGICAL_RECURSIVE_CTE = 78,      // 递归 CTE
    LOGICAL_MATERIALIZED_CTE = 79,   // 物化 CTE

    // 数据修改
    LOGICAL_INSERT = 100,            // INSERT
    LOGICAL_DELETE = 101,            // DELETE
    LOGICAL_UPDATE = 102,            // UPDATE

    // DDL
    LOGICAL_CREATE_TABLE = 126,
    LOGICAL_CREATE_INDEX = 127,
    // ... 其他 DDL 操作
};
```

---

## 3.3 核心 LogicalOperator 详解

### 3.3.1 LogicalGet - 表扫描

```cpp
// src/include/duckdb/planner/operator/logical_get.hpp

class LogicalGet : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_GET;

    LogicalGet(idx_t table_index, TableFunction function, unique_ptr<FunctionData> bind_data,
               vector<LogicalType> returned_types, vector<string> returned_names,
               virtual_column_map_t virtual_columns = virtual_column_map_t());

    //! 表在绑定上下文中的索引
    idx_t table_index;
    //! 扫描函数
    TableFunction function;
    //! 函数绑定数据
    unique_ptr<FunctionData> bind_data;
    //! 所有可返回的列类型
    vector<LogicalType> returned_types;
    //! 所有可返回的列名
    vector<string> names;
    //! 虚拟列映射
    virtual_column_map_t virtual_columns;
    //! 外部使用的列索引（投影下推）
    vector<idx_t> projection_ids;
    //! 下推的表过滤器
    TableFilterSet table_filters;
    //! 动态过滤器（来自上游 JOIN）
    shared_ptr<DynamicTableFilterSet> dynamic_filters;

private:
    //! 实际需要扫描的列 ID
    vector<ColumnIndex> column_ids;
};
```

**LogicalGet 工作原理：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LogicalGet 示例                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT name, age FROM users WHERE age > 18                                 │
│                                                                             │
│  LogicalGet {                                                               │
│      table_index: 0                                                         │
│      function: seq_scan("users")                                            │
│      returned_types: [INTEGER, VARCHAR, INTEGER, ...]  // 全部列            │
│      names: ["id", "name", "age", ...]                 // 全部列名          │
│      column_ids: [1, 2]                                // 只扫描 name, age  │
│      table_filters: { age > 18 }                       // 下推的过滤条件    │
│  }                                                                          │
│                                                                             │
│  输出列绑定:                                                                  │
│    ColumnBinding(0, 0) → name (VARCHAR)                                     │
│    ColumnBinding(0, 1) → age (INTEGER)                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.2 LogicalProjection - 投影

```cpp
// src/include/duckdb/planner/operator/logical_projection.hpp

class LogicalProjection : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_PROJECTION;

    LogicalProjection(idx_t table_index, vector<unique_ptr<Expression>> select_list);

    //! 投影算子的表索引
    idx_t table_index;
    // expressions 继承自 LogicalOperator，存储 SELECT 列表
};
```

**LogicalProjection 工作原理：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LogicalProjection 示例                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT name, age * 2 AS double_age FROM users                              │
│                                                                             │
│  LogicalProjection {                                                        │
│      table_index: 1                                                         │
│      expressions: [                                                         │
│          BoundColumnRefExpression(ColumnBinding(0, 0)),  // name            │
│          BoundFunctionExpression(                        // age * 2         │
│              function: *(INTEGER, INTEGER),                                 │
│              children: [                                                    │
│                  BoundColumnRefExpression(ColumnBinding(0, 1)),             │
│                  BoundConstantExpression(2)                                 │
│              ]                                                              │
│          )                                                                  │
│      ]                                                                      │
│  }                                                                          │
│                                                                             │
│  输出列绑定:                                                                  │
│    ColumnBinding(1, 0) → name                                               │
│    ColumnBinding(1, 1) → double_age                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.3 LogicalFilter - 过滤

```cpp
// src/include/duckdb/planner/operator/logical_filter.hpp

class LogicalFilter : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_FILTER;

    explicit LogicalFilter(unique_ptr<Expression> expression);
    LogicalFilter();

    //! 投影映射（优化用）
    vector<idx_t> projection_map;

    //! 分割 AND 连接的谓词
    bool SplitPredicates() {
        return SplitPredicates(expressions);
    }

    static bool SplitPredicates(vector<unique_ptr<Expression>> &expressions);
};
```

**LogicalFilter 工作原理：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       LogicalFilter 示例                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WHERE age > 18 AND country = 'China'                                       │
│                                                                             │
│  LogicalFilter {                                                            │
│      expressions: [                                                         │
│          BoundComparisonExpression(                                         │
│              GREATER_THAN,                                                  │
│              BoundColumnRefExpression(age),                                 │
│              BoundConstantExpression(18)                                    │
│          ),                                                                 │
│          BoundComparisonExpression(                                         │
│              EQUAL,                                                         │
│              BoundColumnRefExpression(country),                             │
│              BoundConstantExpression('China')                               │
│          )                                                                  │
│      ]                                                                      │
│      // AND 条件被分割成多个独立表达式                                        │
│  }                                                                          │
│                                                                             │
│  注意: expressions 中的多个条件隐式为 AND 关系                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.4 LogicalAggregate - 聚合

```cpp
// src/include/duckdb/planner/operator/logical_aggregate.hpp

class LogicalAggregate : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY;

    LogicalAggregate(idx_t group_index, idx_t aggregate_index, vector<unique_ptr<Expression>> select_list);

    //! GROUP BY 的表索引
    idx_t group_index;
    //! 聚合结果的表索引
    idx_t aggregate_index;
    //! GROUPING 函数的表索引
    idx_t groupings_index;
    //! GROUP BY 表达式列表
    vector<unique_ptr<Expression>> groups;
    //! GROUPING SETS
    vector<GroupingSet> grouping_sets;
    //! GROUPING 函数调用列表
    vector<unsafe_vector<idx_t>> grouping_functions;
    //! 分组列统计信息
    vector<unique_ptr<BaseStatistics>> group_stats;
};
```

**LogicalAggregate 工作原理：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LogicalAggregate 示例                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT country, SUM(amount), COUNT(*) FROM orders GROUP BY country         │
│                                                                             │
│  LogicalAggregate {                                                         │
│      group_index: 2                                                         │
│      aggregate_index: 3                                                     │
│      groups: [                                                              │
│          BoundColumnRefExpression(country)                                  │
│      ]                                                                      │
│      expressions: [        // 聚合函数列表                                   │
│          BoundAggregateExpression(SUM, [amount]),                           │
│          BoundAggregateExpression(COUNT_STAR, [])                           │
│      ]                                                                      │
│  }                                                                          │
│                                                                             │
│  输出列绑定:                                                                  │
│    ColumnBinding(2, 0) → country (来自 groups)                              │
│    ColumnBinding(3, 0) → SUM(amount) (来自 aggregates)                      │
│    ColumnBinding(3, 1) → COUNT(*) (来自 aggregates)                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.5 LogicalJoin 层次结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      LogicalJoin 类层次结构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LogicalJoin (基类)                                                          │
│       │  - join_type: JoinType (INNER, LEFT, RIGHT, OUTER, SEMI, ANTI, MARK)│
│       │  - mark_index: idx_t                                                │
│       │  - left_projection_map, right_projection_map                        │
│       │                                                                     │
│       ├── LogicalComparisonJoin                                             │
│       │       │  - conditions: vector<JoinCondition>                        │
│       │       │  - duplicate_eliminated_columns                             │
│       │       │                                                             │
│       │       ├── LogicalASofJoin                                           │
│       │       │       ASOF 连接（时间序列连接）                              │
│       │       │                                                             │
│       │       └── LogicalDelimJoin                                          │
│       │               分隔符连接（用于相关子查询）                           │
│       │                                                                     │
│       ├── LogicalAnyJoin                                                    │
│       │       - condition: unique_ptr<Expression>                           │
│       │       任意条件连接（不能优化为比较连接）                             │
│       │                                                                     │
│       ├── LogicalCrossProduct                                               │
│       │       笛卡尔积（无条件连接）                                         │
│       │                                                                     │
│       ├── LogicalPositionalJoin                                             │
│       │       位置连接（按行位置匹配）                                       │
│       │                                                                     │
│       └── LogicalDependentJoin                                              │
│               相关子查询连接（待去相关化）                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.6 LogicalComparisonJoin

```cpp
// src/include/duckdb/planner/operator/logical_comparison_join.hpp

class LogicalComparisonJoin : public LogicalJoin {
public:
    explicit LogicalComparisonJoin(JoinType type,
                                   LogicalOperatorType logical_type = LogicalOperatorType::LOGICAL_COMPARISON_JOIN);

    //! 连接条件列表
    vector<JoinCondition> conditions;
    //! MARK 连接的类型
    vector<LogicalType> mark_types;
    //! 去重消除的列
    vector<unique_ptr<Expression>> duplicate_eliminated_columns;
    //! 是否翻转 DelimJoin
    bool delim_flipped = false;
    //! 是否可将 MARK JOIN 转换为 SEMI JOIN
    bool convert_mark_to_semi = true;
    //! 动态过滤器下推信息
    unique_ptr<JoinFilterPushdownInfo> filter_pushdown;
    //! 额外的非等值谓词
    unique_ptr<Expression> predicate;

public:
    //! 创建连接算子
    static unique_ptr<LogicalOperator> CreateJoin(ClientContext &context, JoinType type, JoinRefType ref_type,
                                                  unique_ptr<LogicalOperator> left_child,
                                                  unique_ptr<LogicalOperator> right_child,
                                                  unique_ptr<Expression> condition);

    //! 提取连接条件
    static void ExtractJoinConditions(ClientContext &context, JoinType type, JoinRefType ref_type,
                                      unique_ptr<LogicalOperator> &left_child,
                                      unique_ptr<LogicalOperator> &right_child,
                                      unique_ptr<Expression> condition,
                                      vector<JoinCondition> &conditions,
                                      vector<unique_ptr<Expression>> &arbitrary_expressions);
};
```

**JoinCondition 结构：**

```cpp
struct JoinCondition {
    unique_ptr<Expression> left;     // 左侧表达式
    unique_ptr<Expression> right;    // 右侧表达式
    ExpressionType comparison;       // 比较类型 (=, <, >, <=, >=, <>)
};
```

**LogicalComparisonJoin 示例：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LogicalComparisonJoin 示例                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id            │
│                                                                             │
│  LogicalComparisonJoin {                                                    │
│      type: LOGICAL_COMPARISON_JOIN                                          │
│      join_type: INNER                                                       │
│      conditions: [                                                          │
│          JoinCondition {                                                    │
│              left: BoundColumnRefExpression(ColumnBinding(0, 1)),  // o.customer_id   │
│              right: BoundColumnRefExpression(ColumnBinding(1, 0)), // c.id          │
│              comparison: COMPARE_EQUAL                                      │
│          }                                                                  │
│      ]                                                                      │
│      children: [                                                            │
│          LogicalGet(orders),     // 左子树                                  │
│          LogicalGet(customers)   // 右子树                                  │
│      ]                                                                      │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.7 LogicalOrder - 排序

```cpp
// src/include/duckdb/planner/operator/logical_order.hpp

class LogicalOrder : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_ORDER_BY;

    explicit LogicalOrder(vector<BoundOrderByNode> orders);

    //! 排序规范列表
    vector<BoundOrderByNode> orders;
    //! 投影映射
    vector<idx_t> projection_map;
};

struct BoundOrderByNode {
    OrderType type;                    // ASC 或 DESC
    OrderByNullType null_order;        // NULLS FIRST 或 NULLS LAST
    unique_ptr<Expression> expression; // 排序表达式
};
```

### 3.3.8 LogicalLimit - 限制

```cpp
// src/include/duckdb/planner/operator/logical_limit.hpp

class LogicalLimit : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_LIMIT;

    LogicalLimit(BoundLimitNode limit_val, BoundLimitNode offset_val);

    //! LIMIT 值
    BoundLimitNode limit_val;
    //! OFFSET 值
    BoundLimitNode offset_val;
};

struct BoundLimitNode {
    LimitNodeType type;   // CONSTANT_VALUE, CONSTANT_PERCENTAGE, EXPRESSION_VALUE, EXPRESSION_PERCENTAGE, UNSET
    int64_t constant_integer;
    double constant_percentage;
    unique_ptr<Expression> expression;

    static BoundLimitNode ConstantValue(int64_t value);
};
```

### 3.3.9 LogicalWindow - 窗口函数

```cpp
// src/include/duckdb/planner/operator/logical_window.hpp

class LogicalWindow : public LogicalOperator {
public:
    static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_WINDOW;

    explicit LogicalWindow(idx_t window_index)
        : LogicalOperator(LogicalOperatorType::LOGICAL_WINDOW), window_index(window_index) {}

    //! 窗口函数的表索引
    idx_t window_index;
    // expressions 继承自 LogicalOperator，存储窗口函数表达式
};
```

### 3.3.10 LogicalSetOperation - 集合操作

```cpp
// src/include/duckdb/planner/operator/logical_set_operation.hpp

class LogicalSetOperation : public LogicalOperator {
public:
    LogicalSetOperation(idx_t table_index, idx_t column_count,
                        unique_ptr<LogicalOperator> top,
                        unique_ptr<LogicalOperator> bottom,
                        LogicalOperatorType type, bool setop_all,
                        bool allow_out_of_order = true);

    //! 结果表索引
    idx_t table_index;
    //! 结果列数
    idx_t column_count;
    //! 是否保留重复行 (ALL vs DISTINCT)
    bool setop_all;
    //! 是否允许乱序执行
    bool allow_out_of_order;
};
```

---

## 3.4 SELECT 语句计划生成

### 3.4.1 CreatePlan(BoundSelectNode)

```cpp
// src/planner/binder/query_node/plan_select_node.cpp

unique_ptr<LogicalOperator> Binder::CreatePlan(BoundSelectNode &statement) {
    D_ASSERT(statement.from_table.plan);
    // 1. 从 FROM 子句获取初始计划
    auto root = std::move(statement.from_table.plan);

    // 2. 处理 SAMPLE 子句
    if (statement.sample_options) {
        root = make_uniq<LogicalSample>(std::move(statement.sample_options), std::move(root));
    }

    // 3. 处理 WHERE 子句
    if (statement.where_clause) {
        root = PlanFilter(std::move(statement.where_clause), std::move(root));
    }

    // 4. 处理聚合（GROUP BY + 聚合函数）
    if (!statement.aggregates.empty() || !statement.groups.group_expressions.empty() || statement.having) {
        // 处理 GROUP BY 中的子查询
        for (auto &group : statement.groups.group_expressions) {
            PlanSubqueries(group, root);
        }
        // 处理聚合函数中的子查询
        for (auto &expr : statement.aggregates) {
            PlanSubqueries(expr, root);
        }
        // 创建 LogicalAggregate
        auto aggregate = make_uniq<LogicalAggregate>(
            statement.group_index, statement.aggregate_index,
            std::move(statement.aggregates));
        aggregate->groups = std::move(statement.groups.group_expressions);
        aggregate->groupings_index = statement.groupings_index;
        aggregate->grouping_sets = std::move(statement.groups.grouping_sets);
        aggregate->grouping_functions = std::move(statement.grouping_functions);
        aggregate->AddChild(std::move(root));
        root = std::move(aggregate);
    }

    // 5. 处理 HAVING 子句
    if (statement.having) {
        PlanSubqueries(statement.having, root);
        auto having = make_uniq<LogicalFilter>(std::move(statement.having));
        having->AddChild(std::move(root));
        root = std::move(having);
    }

    // 6. 处理窗口函数
    if (!statement.windows.empty()) {
        auto win = make_uniq<LogicalWindow>(statement.window_index);
        win->expressions = std::move(statement.windows);
        for (auto &expr : win->expressions) {
            PlanSubqueries(expr, root);
        }
        win->AddChild(std::move(root));
        root = std::move(win);
    }

    // 7. 处理 QUALIFY 子句
    if (statement.qualify) {
        PlanSubqueries(statement.qualify, root);
        auto qualify = make_uniq<LogicalFilter>(std::move(statement.qualify));
        qualify->AddChild(std::move(root));
        root = std::move(qualify);
    }

    // 8. 处理 UNNEST
    for (idx_t i = statement.unnests.size(); i > 0; i--) {
        auto &unnest_node = statement.unnests[i - 1];
        auto unnest = make_uniq<LogicalUnnest>(unnest_node.index);
        unnest->expressions = std::move(unnest_node.expressions);
        unnest->AddChild(std::move(root));
        root = std::move(unnest);
    }

    // 9. 处理 SELECT 列表中的子查询
    for (auto &expr : statement.select_list) {
        PlanSubqueries(expr, root);
    }

    // 10. 创建投影
    auto proj = make_uniq<LogicalProjection>(statement.projection_index, std::move(statement.select_list));
    proj->AddChild(std::move(root));
    root = std::move(proj);

    // 11. 处理 DISTINCT, ORDER BY, LIMIT 等修饰符
    root = VisitQueryNode(statement, std::move(root));

    // 12. 处理列裁剪
    if (statement.need_prune) {
        // 创建一个投影来裁剪多余的列
        vector<unique_ptr<Expression>> prune_expressions;
        for (idx_t i = 0; i < statement.column_count; i++) {
            prune_expressions.push_back(make_uniq<BoundColumnRefExpression>(
                proj->expressions[i]->return_type,
                ColumnBinding(statement.projection_index, i)));
        }
        auto prune = make_uniq<LogicalProjection>(statement.prune_index, std::move(prune_expressions));
        prune->AddChild(std::move(root));
        root = std::move(prune);
    }

    return root;
}
```

### 3.4.2 VisitQueryNode - 处理修饰符

```cpp
// src/planner/binder/query_node/plan_query_node.cpp

unique_ptr<LogicalOperator> Binder::VisitQueryNode(BoundQueryNode &node, unique_ptr<LogicalOperator> root) {
    for (auto &mod : node.modifiers) {
        switch (mod->type) {
        case ResultModifierType::DISTINCT_MODIFIER: {
            auto &bound = mod->Cast<BoundDistinctModifier>();
            if (bound.target_distincts.empty()) {
                break;
            }
            auto distinct = make_uniq<LogicalDistinct>(
                std::move(bound.target_distincts), bound.distinct_type);
            distinct->AddChild(std::move(root));
            root = std::move(distinct);
            break;
        }
        case ResultModifierType::ORDER_MODIFIER: {
            auto &bound = mod->Cast<BoundOrderModifier>();
            // 处理 DISTINCT ON + ORDER BY
            if (root->type == LogicalOperatorType::LOGICAL_DISTINCT) {
                auto &distinct = root->Cast<LogicalDistinct>();
                if (distinct.distinct_type == DistinctType::DISTINCT_ON) {
                    distinct.order_by = std::move(bound.orders);
                }
            }
            auto order = make_uniq<LogicalOrder>(std::move(bound.orders));
            order->AddChild(std::move(root));
            root = std::move(order);
            break;
        }
        case ResultModifierType::LIMIT_MODIFIER: {
            auto &bound = mod->Cast<BoundLimitModifier>();
            auto limit = make_uniq<LogicalLimit>(
                std::move(bound.limit_val), std::move(bound.offset_val));
            limit->AddChild(std::move(root));
            root = std::move(limit);
            break;
        }
        default:
            throw BinderException("Unimplemented modifier type!");
        }
    }
    return root;
}
```

---

## 3.5 JOIN 计划生成

### 3.5.1 CreatePlan(BoundJoinRef)

```cpp
// src/planner/binder/tableref/plan_joinref.cpp

unique_ptr<LogicalOperator> Binder::CreatePlan(BoundJoinRef &ref) {
    auto left = std::move(ref.left.plan);
    auto right = std::move(ref.right.plan);

    // 1. 优化：将 RIGHT JOIN 转换为 LEFT JOIN
    if (ref.type == JoinType::RIGHT && ref.ref_type != JoinRefType::ASOF) {
        ref.type = JoinType::LEFT;
        std::swap(left, right);
    }

    // 2. 处理 LATERAL JOIN
    if (ref.lateral) {
        return PlanLateralJoin(std::move(left), std::move(right),
                               ref.correlated_columns, ref.type,
                               std::move(ref.condition));
    }

    // 3. 处理 CROSS JOIN
    if (ref.ref_type == JoinRefType::CROSS) {
        return LogicalCrossProduct::Create(std::move(left), std::move(right));
    }

    // 4. 处理 POSITIONAL JOIN
    if (ref.ref_type == JoinRefType::POSITIONAL) {
        return LogicalPositionalJoin::Create(std::move(left), std::move(right));
    }

    // 5. 处理含子查询的 INNER JOIN
    if (ref.type == JoinType::INNER && ref.condition->HasSubquery()) {
        // 生成 CrossProduct + Filter，后续由优化器处理
        auto root = LogicalCrossProduct::Create(std::move(left), std::move(right));
        auto filter = make_uniq<LogicalFilter>(std::move(ref.condition));
        for (auto &expr : filter->expressions) {
            PlanSubqueries(expr, root);
        }
        filter->AddChild(std::move(root));
        return std::move(filter);
    }

    // 6. 创建比较连接
    auto result = LogicalComparisonJoin::CreateJoin(
        context, ref.type, ref.ref_type,
        std::move(left), std::move(right),
        std::move(ref.condition));

    // 7. 处理 MARK JOIN
    if (ref.type == JoinType::MARK) {
        result->Cast<LogicalJoin>().mark_index = ref.mark_index;
    }

    return result;
}
```

### 3.5.2 连接条件提取

```cpp
void LogicalComparisonJoin::ExtractJoinConditions(
    ClientContext &context, JoinType type, JoinRefType ref_type,
    unique_ptr<LogicalOperator> &left_child,
    unique_ptr<LogicalOperator> &right_child,
    const unordered_set<idx_t> &left_bindings,
    const unordered_set<idx_t> &right_bindings,
    vector<unique_ptr<Expression>> &expressions,
    vector<JoinCondition> &conditions,
    vector<unique_ptr<Expression>> &arbitrary_expressions) {

    for (auto &expr : expressions) {
        auto side = JoinSide::GetJoinSide(*expr, left_bindings, right_bindings);

        // 处理常量条件
        if (side == JoinSide::NONE) {
            if (CanEliminate(context, type, expr)) {
                continue;  // TRUE 条件可以消除
            }
        }
        // 只涉及左表的条件下推到左子树
        else if (side == JoinSide::LEFT) {
            if (CanPushToLeftChild(type)) {
                PushFilterToChild(left_child, expr);
                continue;
            }
        }
        // 只涉及右表的条件下推到右子树
        else if (side == JoinSide::RIGHT) {
            if (CanPushToRightChild(type)) {
                PushFilterToChild(right_child, expr);
                continue;
            }
        }
        // 涉及两边的比较条件转换为 JoinCondition
        else if (side == JoinSide::BOTH) {
            if (IsComparisonExpression(*expr) &&
                CreateJoinCondition(*expr, left_bindings, right_bindings, conditions)) {
                continue;
            }
        }

        // 其他条件作为任意表达式
        arbitrary_expressions.push_back(std::move(expr));
    }
}
```

### 3.5.3 连接类型选择

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       连接算子选择逻辑                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  输入: JoinCondition[], arbitrary_expressions[]                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │ 条件检查                                                          │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ├── ASOF JOIN? ─────────────────────────→ LogicalAsofJoin             │
│       │                                                                     │
│       ├── conditions 为空? ───────────────────→ LogicalAnyJoin              │
│       │       (任意条件连接，不能使用 Hash Join)                              │
│       │                                                                     │
│       ├── 有 arbitrary_expressions?                                         │
│       │       │                                                             │
│       │       ├── INNER JOIN? ────────────────→ LogicalComparisonJoin       │
│       │       │                                   + LogicalFilter           │
│       │       │                                                             │
│       │       └── 其他 JOIN? ─────────────────→ LogicalAnyJoin              │
│       │               (条件合并到 AnyJoin.condition)                         │
│       │                                                                     │
│       └── 只有 conditions ────────────────────→ LogicalComparisonJoin        │
│               (可以使用 Hash Join)                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.6 子查询计划生成

### 3.6.1 PlanSubqueries

```cpp
// src/planner/binder/query_node/plan_subquery.cpp

void Binder::PlanSubqueries(unique_ptr<Expression> &expr_ptr, unique_ptr<LogicalOperator> &root) {
    if (!expr_ptr) {
        return;
    }
    auto &expr = *expr_ptr;

    // 递归处理子表达式
    ExpressionIterator::EnumerateChildren(expr, [&](unique_ptr<Expression> &expr) {
        PlanSubqueries(expr, root);
    });

    // 处理子查询表达式
    if (expr.GetExpressionClass() == ExpressionClass::BOUND_SUBQUERY) {
        auto &subquery = expr.Cast<BoundSubqueryExpression>();
        expr_ptr = PlanSubquery(subquery, root);
    }
}
```

### 3.6.2 非相关子查询规划

```cpp
static unique_ptr<Expression> PlanUncorrelatedSubquery(
    Binder &binder, BoundSubqueryExpression &expr,
    unique_ptr<LogicalOperator> &root,
    unique_ptr<LogicalOperator> plan) {

    switch (expr.subquery_type) {
    case SubqueryType::EXISTS: {
        // EXISTS 子查询
        // 1. 添加 LIMIT 1（只关心是否存在）
        auto limit = make_uniq<LogicalLimit>(BoundLimitNode::ConstantValue(1), BoundLimitNode());
        limit->AddChild(std::move(plan));
        plan = std::move(limit);

        // 2. 添加 COUNT(*) 聚合（结果为 0 或 1）
        auto count_star = ...;
        auto aggregate = make_uniq<LogicalAggregate>(...);
        aggregate->AddChild(std::move(plan));
        plan = std::move(aggregate);

        // 3. 添加投影（COUNT(*) = 1）
        auto comparison = make_uniq<BoundComparisonExpression>(
            ExpressionType::COMPARE_EQUAL,
            count_star_ref, constant_1);
        auto projection = make_uniq<LogicalProjection>(...);
        projection->AddChild(std::move(plan));
        plan = std::move(projection);

        // 4. CrossProduct 添加到主查询
        root = LogicalCrossProduct::Create(std::move(root), std::move(plan));

        // 5. 返回引用投影结果的列引用
        return make_uniq<BoundColumnRefExpression>(...);
    }

    case SubqueryType::SCALAR: {
        // 标量子查询
        // 使用 FIRST 聚合获取第一行
        auto first_agg = ...;
        auto aggregate = make_uniq<LogicalAggregate>(...);
        aggregate->AddChild(std::move(plan));

        // CrossProduct 添加到主查询
        root = LogicalCrossProduct::Create(std::move(root), std::move(aggregate));

        return make_uniq<BoundColumnRefExpression>(...);
    }

    case SubqueryType::ANY: {
        // IN/ANY 子查询
        // 使用 MARK JOIN
        auto join = make_uniq<LogicalComparisonJoin>(JoinType::MARK);
        join->mark_index = mark_index;
        join->AddChild(std::move(root));
        join->AddChild(std::move(plan));
        // 添加连接条件
        join->conditions.push_back(...);

        root = std::move(join);
        return make_uniq<BoundColumnRefExpression>(..., ColumnBinding(mark_index, 0));
    }
    }
}
```

### 3.6.3 相关子查询规划

```cpp
static unique_ptr<Expression> PlanCorrelatedSubquery(
    Binder &binder, BoundSubqueryExpression &expr,
    unique_ptr<LogicalOperator> &root,
    unique_ptr<LogicalOperator> plan) {

    auto &correlated_columns = expr.binder->correlated_columns;

    // 创建 DependentJoin（待后续去相关化）
    auto delim_join = CreateDuplicateEliminatedJoin(
        correlated_columns, join_type, std::move(root), perform_delim);

    switch (expr.subquery_type) {
    case SubqueryType::SCALAR:
        // 相关标量子查询使用 SINGLE JOIN
        delim_join->subquery_type = SubqueryType::SCALAR;
        delim_join->AddChild(std::move(plan));
        root = std::move(delim_join);
        return make_uniq<BoundColumnRefExpression>(...);

    case SubqueryType::EXISTS:
        // 相关 EXISTS 使用 MARK JOIN
        delim_join->subquery_type = SubqueryType::EXISTS;
        delim_join->mark_index = mark_index;
        delim_join->AddChild(std::move(plan));
        root = std::move(delim_join);
        return make_uniq<BoundColumnRefExpression>(..., ColumnBinding(mark_index, 0));

    case SubqueryType::ANY:
        // 相关 IN/ANY 使用带条件的 MARK JOIN
        delim_join->subquery_type = SubqueryType::ANY;
        delim_join->mark_index = mark_index;
        // 添加比较条件
        delim_join->AddChild(std::move(plan));
        root = std::move(delim_join);
        return make_uniq<BoundColumnRefExpression>(..., ColumnBinding(mark_index, 0));
    }
}
```

### 3.6.4 相关子查询处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      相关子查询处理流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  示例查询:                                                                   │
│  SELECT * FROM orders o                                                     │
│  WHERE amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.id)  │
│                                                                             │
│  阶段 1: Binder 识别相关列                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  CorrelatedColumnInfo:                                          │       │
│  │    binding: (0, 0)  // o.id                                     │       │
│  │    depth: 1                                                     │       │
│  │    type: INTEGER                                                │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  阶段 2: Planner 创建 LogicalDependentJoin                                   │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  LogicalDependentJoin {                                         │       │
│  │      join_type: SINGLE                                          │       │
│  │      correlated_columns: [(0,0)]                                │       │
│  │      perform_delim: true                                        │       │
│  │      children: [                                                │       │
│  │          LogicalGet(orders),        // 外部查询                  │       │
│  │          LogicalAggregate(          // 子查询                    │       │
│  │              LogicalFilter(                                     │       │
│  │                  LogicalGet(orders)                             │       │
│  │              )                                                  │       │
│  │          )                                                      │       │
│  │      ]                                                          │       │
│  │  }                                                              │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  阶段 3: Optimizer 进行子查询去相关化 (Decorrelation)                         │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  转换为:                                                         │       │
│  │  LogicalComparisonJoin(INNER) {                                 │       │
│  │      conditions: [o.id = subq.customer_id]                      │       │
│  │      children: [                                                │       │
│  │          LogicalGet(orders),                                    │       │
│  │          LogicalAggregate(                                      │       │
│  │              groups: [customer_id],                             │       │
│  │              aggregates: [AVG(amount)],                         │       │
│  │              child: LogicalGet(orders)                          │       │
│  │          )                                                      │       │
│  │      ]                                                          │       │
│  │  }                                                              │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.7 完整计划生成示例

```sql
SELECT c.name, SUM(o.amount) as total
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China' AND o.date >= '2024-01-01'
GROUP BY c.name
HAVING SUM(o.amount) > 1000
ORDER BY total DESC
LIMIT 10;
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         逻辑计划树                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                           LogicalLimit                                       │
│                           (limit: 10)                                        │
│                               │                                              │
│                               ▼                                              │
│                           LogicalOrder                                       │
│                    (orders: [total DESC])                                    │
│                               │                                              │
│                               ▼                                              │
│                         LogicalProjection                                    │
│                (table_index: 5, prune to [name, total])                     │
│                               │                                              │
│                               ▼                                              │
│                         LogicalProjection                                    │
│         (table_index: 4, expressions: [c.name, SUM(o.amount)])              │
│                               │                                              │
│                               ▼                                              │
│                           LogicalFilter                                      │
│                    (SUM(o.amount) > 1000) [HAVING]                          │
│                               │                                              │
│                               ▼                                              │
│                         LogicalAggregate                                     │
│           (group_index: 2, aggregate_index: 3)                              │
│           groups: [c.name]                                                  │
│           aggregates: [SUM(o.amount)]                                       │
│                               │                                              │
│                               ▼                                              │
│                      LogicalComparisonJoin                                   │
│                         (INNER JOIN)                                         │
│           conditions: [c.id = o.customer_id]                                │
│                      ┌────────┴────────┐                                     │
│                      ▼                 ▼                                     │
│              LogicalFilter      LogicalFilter                                │
│        (c.country = 'China')   (o.date >= '2024-01-01')                     │
│                      │                 │                                     │
│                      ▼                 ▼                                     │
│              LogicalGet          LogicalGet                                  │
│         (customers, idx: 0)   (orders, idx: 1)                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

注意:
1. WHERE 条件在 JOIN 条件提取时被下推到各自的表扫描上方
2. HAVING 子句转换为聚合后的 LogicalFilter
3. SELECT 列表生成 LogicalProjection
4. 需要列裁剪时会添加额外的 Prune Projection
```

---

## 3.8 列绑定传播

### 3.8.1 GetColumnBindings

每个 LogicalOperator 都实现 `GetColumnBindings()` 方法，返回该算子输出的列绑定：

```cpp
// LogicalGet
vector<ColumnBinding> LogicalGet::GetColumnBindings() {
    if (!projection_ids.empty()) {
        return GenerateColumnBindings(table_index, projection_ids.size());
    }
    return GenerateColumnBindings(table_index, column_ids.size());
}

// LogicalProjection
vector<ColumnBinding> LogicalProjection::GetColumnBindings() {
    return GenerateColumnBindings(table_index, expressions.size());
}

// LogicalFilter - 透传子算子的列绑定
vector<ColumnBinding> LogicalFilter::GetColumnBindings() {
    return children[0]->GetColumnBindings();
}

// LogicalAggregate
vector<ColumnBinding> LogicalAggregate::GetColumnBindings() {
    vector<ColumnBinding> result;
    // 先添加 GROUP BY 列
    for (idx_t i = 0; i < groups.size(); i++) {
        result.push_back(ColumnBinding(group_index, i));
    }
    // 再添加聚合列
    for (idx_t i = 0; i < expressions.size(); i++) {
        result.push_back(ColumnBinding(aggregate_index, i));
    }
    return result;
}

// LogicalJoin
vector<ColumnBinding> LogicalJoin::GetColumnBindings() {
    auto left_bindings = children[0]->GetColumnBindings();
    auto right_bindings = children[1]->GetColumnBindings();
    left_bindings.insert(left_bindings.end(), right_bindings.begin(), right_bindings.end());
    return left_bindings;
}
```

### 3.8.2 ResolveTypes

类型解析在计划生成后通过 `ResolveOperatorTypes()` 递归完成：

```cpp
void LogicalOperator::ResolveOperatorTypes() {
    // 先解析所有子算子的类型
    for (auto &child : children) {
        child->ResolveOperatorTypes();
    }
    // 再解析当前算子的类型
    ResolveTypes();
}

// LogicalProjection 的类型解析
void LogicalProjection::ResolveTypes() {
    types.clear();
    for (auto &expr : expressions) {
        types.push_back(expr->return_type);
    }
}

// LogicalFilter 透传子算子的类型
void LogicalFilter::ResolveTypes() {
    types = children[0]->types;
}
```

---

## 3.9 源文件索引

| 组件 | 文件路径 |
|------|----------|
| Planner 类 | `src/include/duckdb/planner/planner.hpp` |
| LogicalOperator 基类 | `src/include/duckdb/planner/logical_operator.hpp` |
| LogicalOperator 类型 | `src/include/duckdb/common/enums/logical_operator_type.hpp` |
| 算子定义 | `src/include/duckdb/planner/operator/` |
| SELECT 计划生成 | `src/planner/binder/query_node/plan_select_node.cpp` |
| 修饰符处理 | `src/planner/binder/query_node/plan_query_node.cpp` |
| JOIN 计划生成 | `src/planner/binder/tableref/plan_joinref.cpp` |
| 子查询处理 | `src/planner/binder/query_node/plan_subquery.cpp` |

---

## 3.10 本章小结

本章详细剖析了 DuckDB 的 Planner（逻辑计划生成器）组件：

1. **Planner 架构**：Planner 协调 Binder 完成绑定和计划生成，最终输出 LogicalOperator 树

2. **LogicalOperator 基类**：定义了算子的通用接口，包括类型、子算子、表达式、列绑定等

3. **核心算子类型**：
   - `LogicalGet`: 表扫描，支持列投影和过滤器下推
   - `LogicalProjection`: 投影，计算 SELECT 列表表达式
   - `LogicalFilter`: 过滤，支持谓词分割
   - `LogicalAggregate`: 聚合，处理 GROUP BY 和聚合函数
   - `LogicalJoin`: 连接家族，包括 ComparisonJoin、AnyJoin、CrossProduct 等
   - `LogicalOrder/LogicalLimit`: 处理 ORDER BY 和 LIMIT

4. **SELECT 计划生成流程**：
   FROM → SAMPLE → WHERE → GROUP BY/聚合 → HAVING → 窗口 → QUALIFY → SELECT → 修饰符

5. **JOIN 计划生成**：提取连接条件，选择合适的连接算子类型

6. **子查询处理**：
   - 非相关子查询通过 CrossProduct 或 MARK JOIN 处理
   - 相关子查询创建 DependentJoin，待优化器去相关化

7. **列绑定传播**：每个算子通过 GetColumnBindings() 暴露输出列，用于上层算子引用

下一章我们将进入 Optimizer，了解如何对逻辑计划进行优化。

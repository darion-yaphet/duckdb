# DuckDB 查询处理深度解析：第二章 - Binder（语义绑定器）

## 本章概述

Binder 是 DuckDB 查询处理流程中承上启下的关键组件。它接收 Parser 生成的 AST（ParsedStatement），将其中的符号（表名、列名、函数名等）解析到实际的 Catalog 对象，进行类型推导和语义检查，最终生成可供 Planner 使用的 BoundStatement。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Binder 在查询处理中的位置                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   SQLStatement (Parser Output)                                              │
│        │                                                                    │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                         Binder                                   │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  1. 符号解析 (Symbol Resolution)                             │ │       │
│   │  │     - 表名 → TableCatalogEntry                              │ │       │
│   │  │     - 列名 → ColumnBinding (table_index, column_index)      │ │       │
│   │  │     - 函数名 → FunctionCatalogEntry                          │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  2. 类型推导 (Type Inference)                                │ │       │
│   │  │     - 表达式类型计算                                          │ │       │
│   │  │     - 隐式类型转换                                            │ │       │
│   │  │     - 函数重载解析                                            │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   │  ┌─────────────────────────────────────────────────────────────┐ │       │
│   │  │  3. 语义检查 (Semantic Validation)                           │ │       │
│   │  │     - 聚合函数正确使用                                        │ │       │
│   │  │     - GROUP BY 语义验证                                       │ │       │
│   │  │     - 子查询相关性检测                                        │ │       │
│   │  └─────────────────────────────────────────────────────────────┘ │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │                                                                    │
│        ▼                                                                    │
│   BoundStatement + LogicalOperator Tree (Binder Output)                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.1 Binder 整体架构

### 2.1.1 Binder 类设计

`Binder` 类是语义绑定的核心，负责将解析树转换为绑定树。

```cpp
// src/include/duckdb/planner/binder.hpp

class Binder : public enable_shared_from_this<Binder> {
public:
    //! 创建 Binder 实例
    static shared_ptr<Binder> CreateBinder(ClientContext &context,
                                           optional_ptr<Binder> parent = nullptr,
                                           BinderType binder_type = BinderType::REGULAR_BINDER);

    //! 绑定 SQL 语句
    BoundStatement Bind(SQLStatement &statement);
    BoundStatement Bind(QueryNode &node);

    //! 客户端上下文
    ClientContext &context;
    //! 绑定上下文 - 管理当前作用域内的表和列绑定
    BindContext bind_context;
    //! 相关列集合 - 用于跟踪子查询中引用的外部列
    CorrelatedColumns correlated_columns;
    //! 当前子查询别名
    string alias;
    //! 宏参数绑定
    optional_ptr<DummyBinding> macro_binding;
    //! Lambda 参数绑定
    optional_ptr<vector<DummyBinding>> lambda_bindings;

private:
    //! 父 Binder（用于子查询绑定）
    shared_ptr<Binder> parent;
    //! Binder 类型
    BinderType binder_type;
    //! 全局 Binder 状态（跨子查询共享）
    shared_ptr<GlobalBinderState> global_binder_state;
    //! 查询级别 Binder 状态
    shared_ptr<QueryBinderState> query_binder_state;
    //! Binder 深度（用于相关子查询处理）
    idx_t depth;
};
```

### 2.1.2 Binder 类型

```cpp
enum class BinderType : uint8_t {
    REGULAR_BINDER,   // 常规绑定器
    VIEW_BINDER       // 视图绑定器（用于绑定视图定义）
};

enum class BindingMode : uint8_t {
    STANDARD_BINDING,           // 标准绑定模式
    EXTRACT_NAMES,              // 只提取名称（用于 DESCRIBE）
    EXTRACT_REPLACEMENT_SCANS,  // 提取替换扫描
    EXTRACT_QUALIFIED_NAMES     // 提取限定名称
};
```

### 2.1.3 GlobalBinderState - 全局共享状态

```cpp
struct GlobalBinderState {
    //! 已绑定的表计数器
    idx_t bound_tables = 0;
    //! 语句属性
    StatementProperties prop;
    //! 绑定模式
    BindingMode mode = BindingMode::STANDARD_BINDING;
    //! 提取的表名集合
    unordered_set<string> table_names;
    //! 替换扫描映射
    case_insensitive_map_t<unique_ptr<TableRef>> replacement_scans;
    //! USING 列集合
    vector<unique_ptr<UsingColumnSet>> using_column_sets;
    //! 参数映射
    optional_ptr<BoundParameterMap> parameters;
};
```

---

## 2.2 BindContext - 绑定上下文

`BindContext` 管理当前作用域内所有可见的表和列绑定，是符号解析的核心数据结构。

### 2.2.1 BindContext 类设计

```cpp
// src/include/duckdb/planner/bind_context.hpp

class BindContext {
public:
    //! 根据列名查找匹配的表绑定
    optional_ptr<Binding> GetMatchingBinding(const string &column_name);

    //! 绑定列引用表达式
    BindResult BindColumn(ColumnRefExpression &colref, idx_t depth);

    //! 添加基表绑定
    void AddBaseTable(idx_t index, const string &alias,
                      const vector<string> &names,
                      const vector<LogicalType> &types,
                      vector<ColumnIndex> &bound_column_ids,
                      TableCatalogEntry &entry);

    //! 添加子查询绑定
    void AddSubquery(idx_t index, const string &alias,
                     SubqueryRef &ref, BoundStatement &subquery);

    //! 添加 CTE 绑定
    void AddCTEBinding(idx_t index, BindingAlias alias,
                       const vector<string> &names,
                       const vector<LogicalType> &types,
                       CTEType cte_type);

    //! 获取所有绑定列表
    const vector<unique_ptr<Binding>> &GetBindingsList();

    //! 处理 USING 子句绑定
    void AddUsingBinding(const string &column_name, UsingColumnSet &set);

private:
    Binder &binder;
    //! 绑定列表（按插入顺序）
    vector<unique_ptr<Binding>> bindings_list;
    //! USING 列映射
    case_insensitive_map_t<reference_set_t<UsingColumnSet>> using_columns;
    //! CTE 绑定列表
    vector<unique_ptr<CTEBinding>> cte_bindings;
};
```

### 2.2.2 绑定查找流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           列名解析流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ColumnRefExpression: "column_name" 或 "table.column"                       │
│        │                                                                    │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  1. 检查 USING 绑定                                              │       │
│   │     - 如果是 USING 子句中的列，返回 COALESCE 表达式               │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │ (未找到)                                                           │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  2. 检查 Lambda 参数                                             │       │
│   │     - 如果是 Lambda 表达式的参数，返回 LambdaRefExpression        │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │ (未找到)                                                           │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  3. 在 bindings_list 中查找匹配的表                              │       │
│   │     - 遍历所有绑定，查找包含该列名的表                            │       │
│   │     - 如果多个表包含同名列，抛出歧义错误                          │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │ (未找到)                                                           │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  4. 检查宏参数绑定                                               │       │
│   │     - 如果在宏展开中，检查是否是宏参数                            │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│        │ (未找到)                                                           │
│        ▼                                                                    │
│   返回错误：Column "xxx" not found                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Binding 层次结构

`Binding` 表示一个表或类表对象在当前作用域中的绑定。

### 2.3.1 Binding 基类

```cpp
// src/include/duckdb/planner/table_binding.hpp

enum class BindingType {
    BASE,           // 基础绑定
    TABLE,          // 表绑定（带列追踪）
    DUMMY,          // 虚拟绑定（宏参数、Lambda）
    CATALOG_ENTRY,  // Catalog 条目绑定
    CTE             // CTE 绑定
};

struct Binding {
    Binding(BindingType binding_type, BindingAlias alias,
            vector<LogicalType> types, vector<string> names, idx_t index);

    //! 尝试获取列索引
    bool TryGetBindingIndex(const string &column_name, column_t &column_index);

    //! 绑定列引用
    virtual BindResult Bind(ColumnRefExpression &colref, idx_t depth);

protected:
    BindingType binding_type;      // 绑定类型
    BindingAlias alias;            // 别名
    idx_t index;                   // 表索引
    vector<LogicalType> types;     // 列类型
    vector<string> names;          // 列名
    case_insensitive_map_t<column_t> name_map;  // 名称到索引映射
};
```

### 2.3.2 TableBinding - 表绑定

```cpp
struct TableBinding : public Binding {
    static constexpr const BindingType TYPE = BindingType::TABLE;

    TableBinding(const string &alias, vector<LogicalType> types,
                 vector<string> names, vector<ColumnIndex> &bound_column_ids,
                 optional_ptr<StandardEntry> entry, idx_t index,
                 virtual_column_map_t virtual_columns);

    //! 已绑定的列 ID 引用（用于投影下推）
    vector<ColumnIndex> &bound_column_ids;
    //! 底层 Catalog 条目
    optional_ptr<StandardEntry> entry;
    //! 虚拟列映射
    virtual_column_map_t virtual_columns;

    //! 展开生成列
    unique_ptr<ParsedExpression> ExpandGeneratedColumn(const string &column_name);

    //! 绑定列引用
    BindResult Bind(ColumnRefExpression &colref, idx_t depth) override;
};
```

### 2.3.3 CTEBinding - CTE 绑定

```cpp
enum class CTEType {
    CAN_BE_REFERENCED,     // 可被引用（普通 CTE）
    CANNOT_BE_REFERENCED   // 不可被引用（递归 CTE 的锚点）
};

struct CTEBinding : public Binding {
    static constexpr const BindingType TYPE = BindingType::CTE;

    CTEBinding(BindingAlias alias, vector<LogicalType> types,
               vector<string> names, idx_t index, CTEType type);

    bool CanBeReferenced() const;
    bool IsReferenced() const;
    void Reference();

private:
    CTEType cte_type;
    idx_t reference_count;
    shared_ptr<CTEBindState> bind_state;
};
```

### 2.3.4 DummyBinding - 宏和 Lambda 绑定

```cpp
struct DummyBinding : public Binding {
    static constexpr const BindingType TYPE = BindingType::DUMMY;
    static constexpr const char *DUMMY_NAME = "0_macro_parameters";

    DummyBinding(vector<LogicalType> types, vector<string> names, string dummy_name);

    //! 宏参数（指向实际参数列表）
    vector<unique_ptr<ParsedExpression>> *arguments;
    //! 虚拟绑定名称
    string dummy_name;

    //! 绑定宏参数
    BindResult Bind(ColumnRefExpression &col_ref, idx_t depth) override;
    //! 绑定 Lambda 参数
    BindResult Bind(LambdaRefExpression &lambda_ref, idx_t depth);
};
```

---

## 2.4 ColumnBinding - 列绑定

`ColumnBinding` 是 Binder 的核心输出之一，用于唯一标识一个列。

```cpp
// src/include/duckdb/planner/column_binding.hpp

struct ColumnBinding {
    idx_t table_index;   // 表索引（对应 LogicalOperator 的 table_index）
    idx_t column_index;  // 列索引（在表内的位置）

    ColumnBinding()
        : table_index(DConstants::INVALID_INDEX),
          column_index(DConstants::INVALID_INDEX) {}

    ColumnBinding(idx_t table, idx_t column)
        : table_index(table), column_index(column) {}

    string ToString() const {
        return "#[" + to_string(table_index) + "." + to_string(column_index) + "]";
    }

    bool operator==(const ColumnBinding &rhs) const {
        return table_index == rhs.table_index && column_index == rhs.column_index;
    }
};
```

**ColumnBinding 工作原理：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ColumnBinding 示例                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT customers.name, orders.amount                                        │
│  FROM customers, orders                                                      │
│  WHERE customers.id = orders.customer_id                                     │
│                                                                             │
│  绑定结果：                                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │  customers 表 (table_index = 0)                                  │       │
│  │    id    → ColumnBinding(0, 0)                                   │       │
│  │    name  → ColumnBinding(0, 1)                                   │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │  orders 表 (table_index = 1)                                     │       │
│  │    id          → ColumnBinding(1, 0)                             │       │
│  │    customer_id → ColumnBinding(1, 1)                             │       │
│  │    amount      → ColumnBinding(1, 2)                             │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  表达式绑定结果：                                                             │
│    customers.name → BoundColumnRefExpression(ColumnBinding(0, 1))           │
│    orders.amount  → BoundColumnRefExpression(ColumnBinding(1, 2))           │
│    customers.id = orders.customer_id                                        │
│      → BoundComparisonExpression(                                           │
│            BoundColumnRefExpression(ColumnBinding(0, 0)),                   │
│            BoundColumnRefExpression(ColumnBinding(1, 1)))                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.5 ExpressionBinder - 表达式绑定器

`ExpressionBinder` 负责将 `ParsedExpression` 转换为 `Expression`（绑定表达式）。

### 2.5.1 ExpressionBinder 基类

```cpp
// src/include/duckdb/planner/expression_binder.hpp

class ExpressionBinder {
public:
    ExpressionBinder(Binder &binder, ClientContext &context,
                     bool replace_binder = false);

    //! 目标类型（用于类型转换）
    LogicalType target_type;

    //! 绑定表达式
    unique_ptr<Expression> Bind(unique_ptr<ParsedExpression> &expr,
                                optional_ptr<LogicalType> result_type = nullptr,
                                bool root_expression = true);

    //! 限定列名（添加表名前缀）
    unique_ptr<ParsedExpression> QualifyColumnName(ColumnRefExpression &col_ref,
                                                   ErrorData &error);

    //! 静态方法：限定所有列名
    static void QualifyColumnNames(Binder &binder,
                                   unique_ptr<ParsedExpression> &expr);

protected:
    //! 绑定各类表达式的方法
    BindResult BindExpression(BetweenExpression &expr, idx_t depth);
    BindResult BindExpression(CaseExpression &expr, idx_t depth);
    BindResult BindExpression(ColumnRefExpression &expr, idx_t depth,
                              bool root_expression,
                              unique_ptr<ParsedExpression> &expr_ptr);
    BindResult BindExpression(FunctionExpression &expr, idx_t depth,
                              unique_ptr<ParsedExpression> &expr_ptr);
    BindResult BindExpression(SubqueryExpression &expr, idx_t depth);
    // ... 更多表达式类型

    //! 绑定函数
    virtual BindResult BindFunction(FunctionExpression &expr,
                                    ScalarFunctionCatalogEntry &function,
                                    idx_t depth);

    //! 绑定聚合函数
    virtual BindResult BindAggregate(FunctionExpression &expr,
                                     AggregateFunctionCatalogEntry &function,
                                     idx_t depth);

    Binder &binder;
    ClientContext &context;
    vector<BoundColumnReferenceInfo> bound_columns;
};
```

### 2.5.2 ExpressionBinder 层次结构

DuckDB 为不同的 SQL 子句提供了专门的 ExpressionBinder：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ExpressionBinder 层次结构                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ExpressionBinder (基类)                                                     │
│       │                                                                     │
│       ├── BaseSelectBinder                                                  │
│       │       │   支持聚合函数和窗口函数                                      │
│       │       │                                                             │
│       │       ├── SelectBinder       (SELECT 列表)                           │
│       │       │                                                             │
│       │       ├── HavingBinder       (HAVING 子句)                           │
│       │       │                                                             │
│       │       └── QualifyBinder      (QUALIFY 子句)                          │
│       │                                                                     │
│       ├── WhereBinder                (WHERE 子句)                            │
│       │       不支持聚合函数                                                  │
│       │                                                                     │
│       ├── GroupBinder                (GROUP BY 子句)                         │
│       │       处理别名引用和位置引用                                          │
│       │                                                                     │
│       ├── OrderBinder                (ORDER BY 子句)                         │
│       │       支持别名和位置引用                                              │
│       │                                                                     │
│       ├── LateralBinder              (LATERAL 子查询)                        │
│       │       支持对外部表的引用                                              │
│       │                                                                     │
│       ├── InsertBinder               (INSERT 语句)                           │
│       │                                                                     │
│       ├── UpdateBinder               (UPDATE SET 子句)                       │
│       │                                                                     │
│       ├── ConstantBinder             (常量表达式)                             │
│       │       只允许常量                                                      │
│       │                                                                     │
│       ├── CheckBinder                (CHECK 约束)                            │
│       │                                                                     │
│       ├── IndexBinder                (索引表达式)                             │
│       │                                                                     │
│       └── TableFunctionBinder        (表函数参数)                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.5.3 WhereBinder 示例

```cpp
// src/include/duckdb/planner/expression_binder/where_binder.hpp

class WhereBinder : public ExpressionBinder {
public:
    WhereBinder(Binder &binder, ClientContext &context,
                optional_ptr<ColumnAliasBinder> column_alias_binder = nullptr);

protected:
    BindResult BindExpression(unique_ptr<ParsedExpression> &expr_ptr,
                              idx_t depth,
                              bool root_expression = false) override;

    //! WHERE 子句不支持聚合函数
    string UnsupportedAggregateMessage() override;

    //! 尝试解析列别名引用
    bool TryResolveAliasReference(ColumnRefExpression &colref, idx_t depth,
                                  bool root_expression, BindResult &result,
                                  unique_ptr<ParsedExpression> &expr_ptr) override;

private:
    optional_ptr<ColumnAliasBinder> column_alias_binder;
};
```

### 2.5.4 BaseSelectBinder - SELECT 绑定器基类

```cpp
// src/include/duckdb/planner/expression_binder/base_select_binder.hpp

struct BoundGroupInformation {
    parsed_expression_map_t<idx_t> map;           // 表达式 → 组索引
    case_insensitive_map_t<idx_t> alias_map;      // 别名 → 组索引
    unordered_map<idx_t, idx_t> collated_groups;  // 需要排序的组
};

class BaseSelectBinder : public ExpressionBinder {
public:
    BaseSelectBinder(Binder &binder, ClientContext &context,
                     BoundSelectNode &node, BoundGroupInformation &info);

    bool BoundAggregates() { return bound_aggregate; }

protected:
    BindResult BindExpression(unique_ptr<ParsedExpression> &expr_ptr,
                              idx_t depth,
                              bool root_expression = false) override;

    //! 绑定聚合函数
    BindResult BindAggregate(FunctionExpression &expr,
                             AggregateFunctionCatalogEntry &function,
                             idx_t depth) override;

    //! 绑定窗口函数
    virtual BindResult BindWindow(WindowExpression &expr, idx_t depth);

    //! 尝试绑定到 GROUP BY 中的表达式
    idx_t TryBindGroup(ParsedExpression &expr);

    bool inside_window;
    bool bound_aggregate = false;
    BoundSelectNode &node;
    BoundGroupInformation &info;
};
```

---

## 2.6 Expression - 绑定表达式层次结构

绑定后的表达式继承自 `Expression` 基类，与 `ParsedExpression` 形成对应关系。

### 2.6.1 Expression 基类

```cpp
// src/include/duckdb/planner/expression.hpp

class Expression : public BaseExpression {
public:
    Expression(ExpressionType type, ExpressionClass expression_class,
               LogicalType return_type);

    //! 返回类型（在绑定阶段确定）
    LogicalType return_type;

    //! 表达式属性查询
    bool IsAggregate() const override;
    bool IsWindow() const override;
    bool HasSubquery() const override;
    bool IsScalar() const override;
    bool HasParameter() const override;
    virtual bool IsVolatile() const;
    virtual bool IsFoldable() const;

    //! 复制表达式
    virtual unique_ptr<Expression> Copy() const = 0;
};
```

### 2.6.2 绑定表达式类型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ParsedExpression → Expression 转换                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ParsedExpression (Parser 输出)    →    Expression (Binder 输出)            │
│                                                                             │
│  ColumnRefExpression               →    BoundColumnRefExpression            │
│    column_names: ["t", "a"]              binding: ColumnBinding(0, 1)       │
│                                          return_type: INTEGER               │
│                                                                             │
│  ConstantExpression                →    BoundConstantExpression             │
│    value: Value(42)                      value: Value(42)                   │
│                                          return_type: INTEGER               │
│                                                                             │
│  FunctionExpression                →    BoundFunctionExpression             │
│    function_name: "abs"                  function: ScalarFunction           │
│    children: [...]                       children: [Expression...]          │
│                                          return_type: 由函数确定            │
│                                                                             │
│  ComparisonExpression              →    BoundComparisonExpression           │
│    type: COMPARE_EQUAL                   type: COMPARE_EQUAL                │
│    left, right: ParsedExpr              left, right: Expression             │
│                                          return_type: BOOLEAN               │
│                                                                             │
│  SubqueryExpression                →    BoundSubqueryExpression             │
│    subquery: SelectStatement            binder: Binder                      │
│    subquery_type: SCALAR/ANY/IN         subquery: BoundStatement            │
│                                          return_type: 由子查询确定          │
│                                                                             │
│  (FunctionExpression)              →    BoundAggregateExpression            │
│    function_name: "sum"                  function: AggregateFunction        │
│                                          filter: optional Expression        │
│                                                                             │
│  WindowExpression                  →    BoundWindowExpression               │
│    (解析时已识别为窗口)                   function: WindowFunction           │
│                                          partitions, orders: [...]          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.6.3 BoundColumnRefExpression

```cpp
// src/include/duckdb/planner/expression/bound_columnref_expression.hpp

class BoundColumnRefExpression : public Expression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::BOUND_COLUMN_REF;

    BoundColumnRefExpression(LogicalType type, ColumnBinding binding,
                             idx_t depth = 0);

    //! 列绑定（table_index, column_index）
    ColumnBinding binding;

    //! 子查询深度
    //! depth=0 表示当前查询
    //! depth=1 表示父查询（相关子查询引用）
    //! depth=2 表示祖父查询...
    idx_t depth;

    bool IsScalar() const override { return false; }
    bool IsFoldable() const override { return false; }
};
```

### 2.6.4 BoundFunctionExpression

```cpp
// src/include/duckdb/planner/expression/bound_function_expression.hpp

class BoundFunctionExpression : public Expression {
public:
    BoundFunctionExpression(LogicalType return_type, ScalarFunction bound_function,
                            vector<unique_ptr<Expression>> arguments,
                            unique_ptr<FunctionData> bind_info,
                            bool is_operator = false);

    //! 绑定的标量函数
    ScalarFunction function;
    //! 子表达式
    vector<unique_ptr<Expression>> children;
    //! 函数绑定信息（函数特定数据）
    unique_ptr<FunctionData> bind_info;
    //! 是否是运算符
    bool is_operator;
};
```

---

## 2.7 语句绑定 - BindSelectNode

### 2.7.1 SELECT 语句绑定流程

```cpp
// src/planner/binder/query_node/bind_select_node.cpp

BoundStatement Binder::BindSelectNode(SelectNode &statement, BoundStatement from_table) {
    auto result = make_uniq<BoundSelectNode>();

    // 1. 生成各类索引
    result->projection_index = GenerateTableIndex();
    result->group_index = GenerateTableIndex();
    result->aggregate_index = GenerateTableIndex();
    result->window_index = GenerateTableIndex();

    // 2. 设置 FROM 子句结果
    result->from_table = std::move(from_table);

    // 3. 展开 SELECT 列表中的 * 表达式
    vector<unique_ptr<ParsedExpression>> new_select_list;
    ExpandStarExpressions(statement.select_list, new_select_list);
    statement.select_list = std::move(new_select_list);

    // 4. 绑定 WHERE 子句
    if (statement.where_clause) {
        ColumnAliasBinder alias_binder(bind_state);
        WhereBinder where_binder(*this, context, &alias_binder);
        result->where_clause = where_binder.Bind(statement.where_clause);
    }

    // 5. 绑定 ORDER BY / DISTINCT 修饰符
    OrderBinder order_binder({*this}, statement, bind_state);
    PrepareModifiers(order_binder, statement, *result);

    // 6. 绑定 GROUP BY 子句
    if (!statement.groups.group_expressions.empty()) {
        GroupBinder group_binder(*this, context, statement,
                                 result->group_index, bind_state, info.alias_map);
        for (idx_t i = 0; i < statement.groups.group_expressions.size(); i++) {
            auto bound_expr = group_binder.Bind(statement.groups.group_expressions[i]);
            result->groups.group_expressions.push_back(std::move(bound_expr));
        }
    }

    // 7. 绑定 HAVING 子句
    if (statement.having) {
        HavingBinder having_binder(*this, context, *result, info,
                                   statement.aggregate_handling);
        result->having = having_binder.Bind(statement.having);
    }

    // 8. 绑定 SELECT 列表
    SelectBinder select_binder(*this, context, *result, info);
    for (idx_t i = 0; i < statement.select_list.size(); i++) {
        auto expr = select_binder.Bind(statement.select_list[i]);
        result->select_list.push_back(std::move(expr));
        result->types.push_back(result_type);
    }

    // 9. 验证聚合语义
    //    如果有聚合或 GROUP BY，非聚合列必须在 GROUP BY 中
    if (!result->groups.group_expressions.empty() || !result->aggregates.empty()) {
        if (select_binder.HasBoundColumns()) {
            // 检查是否有未在 GROUP BY 中的列
            throw BinderException("column must appear in GROUP BY clause...");
        }
    }

    // 10. 创建逻辑计划
    BoundStatement bound_statement;
    bound_statement.plan = CreatePlan(*result);
    bound_statement.types = result->types;
    bound_statement.names = result->names;
    return bound_statement;
}
```

### 2.7.2 BoundSelectNode 结构

```cpp
// src/include/duckdb/planner/query_node/bound_select_node.hpp

class BoundSelectNode : public BoundQueryNode {
public:
    //! 绑定状态信息
    SelectBindState bind_state;

    //! 投影列表（绑定后的 SELECT 表达式）
    vector<unique_ptr<Expression>> select_list;

    //! FROM 子句的绑定结果
    BoundStatement from_table;

    //! WHERE 子句
    unique_ptr<Expression> where_clause;

    //! GROUP BY 子句
    BoundGroupByNode groups;

    //! HAVING 子句
    unique_ptr<Expression> having;

    //! QUALIFY 子句
    unique_ptr<Expression> qualify;

    //! SAMPLE 选项
    unique_ptr<SampleOptions> sample_options;

    //! 结果列数
    idx_t column_count;

    //! 各类索引
    idx_t projection_index;   // 投影算子索引
    idx_t group_index;        // GROUP BY 算子索引
    idx_t aggregate_index;    // 聚合算子索引
    idx_t groupings_index;    // GROUPING 函数索引
    idx_t window_index;       // 窗口算子索引
    idx_t prune_index;        // 裁剪算子索引

    //! 聚合函数列表
    vector<unique_ptr<Expression>> aggregates;

    //! 窗口函数列表
    vector<unique_ptr<Expression>> windows;

    //! UNNEST 表达式
    unordered_map<idx_t, BoundUnnestNode> unnests;
};
```

---

## 2.8 函数绑定

### 2.8.1 函数解析流程

```cpp
// src/planner/binder/expression/bind_function_expression.cpp

BindResult ExpressionBinder::BindExpression(FunctionExpression &function,
                                            idx_t depth,
                                            unique_ptr<ParsedExpression> &expr_ptr) {
    // 1. 查找函数
    auto func = BindAndQualifyFunction(function, true);

    // 2. 根据函数类型分发
    switch (func->type) {
    case CatalogType::SCALAR_FUNCTION_ENTRY: {
        // 检查是否是 Lambda 函数
        auto child = function.IsLambdaFunction();
        if (child) {
            return BindLambdaFunction(function,
                                      func->Cast<ScalarFunctionCatalogEntry>(),
                                      depth);
        }
        return BindFunction(function,
                            func->Cast<ScalarFunctionCatalogEntry>(),
                            depth);
    }
    case CatalogType::MACRO_ENTRY:
        // 宏函数展开
        return BindMacro(function,
                         func->Cast<ScalarMacroCatalogEntry>(),
                         depth, expr_ptr);
    default:
        // 聚合函数
        return BindAggregate(function,
                             func->Cast<AggregateFunctionCatalogEntry>(),
                             depth);
    }
}
```

### 2.8.2 标量函数绑定

```cpp
BindResult ExpressionBinder::BindFunction(FunctionExpression &function,
                                          ScalarFunctionCatalogEntry &func,
                                          idx_t depth) {
    // 1. 绑定所有子表达式
    ErrorData error;
    for (idx_t i = 0; i < function.children.size(); i++) {
        BindChild(function.children[i], depth, error);
    }
    if (error.HasError()) {
        return BindResult(std::move(error));
    }

    // 2. 提取子表达式
    vector<unique_ptr<Expression>> children;
    for (idx_t i = 0; i < function.children.size(); i++) {
        auto &child = BoundExpression::GetExpression(*function.children[i]);
        children.push_back(std::move(child));
    }

    // 3. 使用 FunctionBinder 解析重载
    FunctionBinder function_binder(binder);
    auto result = function_binder.BindScalarFunction(func, std::move(children),
                                                      error, function.is_operator);

    return BindResult(std::move(result));
}
```

### 2.8.3 函数重载解析

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        函数重载解析流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  函数调用：abs(-5)                                                           │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  1. 查找函数集 (ScalarFunctionCatalogEntry)                       │       │
│  │     abs: [abs(TINYINT), abs(SMALLINT), abs(INTEGER),             │       │
│  │           abs(BIGINT), abs(FLOAT), abs(DOUBLE), ...]             │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  2. 获取参数类型                                                  │       │
│  │     参数 -5 的类型: INTEGER                                       │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  3. 匹配最佳重载                                                  │       │
│  │     完全匹配 > 隐式转换匹配 > 需要类型转换                          │       │
│  │     选中: abs(INTEGER) -> INTEGER                                │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  4. 插入必要的类型转换                                            │       │
│  │     如果需要，用 BoundCastExpression 包装参数                      │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  返回 BoundFunctionExpression                                               │
│    function: abs(INTEGER) -> INTEGER                                        │
│    children: [BoundConstantExpression(-5)]                                  │
│    return_type: INTEGER                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.9 子查询绑定

### 2.9.1 子查询类型

```cpp
enum class SubqueryType : uint8_t {
    SCALAR,     // 标量子查询: (SELECT max(x) FROM t)
    EXISTS,     // EXISTS 子查询: EXISTS (SELECT ...)
    ANY,        // ANY/IN 子查询: x IN (SELECT ...)
    NOT_EXISTS  // NOT EXISTS
};
```

### 2.9.2 子查询绑定流程

```cpp
// src/planner/binder/expression/bind_subquery_expression.cpp

BindResult ExpressionBinder::BindExpression(SubqueryExpression &expr, idx_t depth) {
    // 1. 创建新的子 Binder
    auto subquery_binder = Binder::CreateBinder(context, &binder);
    subquery_binder->can_contain_nulls = true;

    // 2. 绑定子查询
    auto bound_node = subquery_binder->BindNode(*expr.subquery->node);

    // 3. 处理相关列（depth > 1 的列引用外部查询）
    for (idx_t i = 0; i < subquery_binder->correlated_columns.size(); i++) {
        CorrelatedColumnInfo corr = subquery_binder->correlated_columns[i];
        if (corr.depth > 1) {
            // 引用更外层查询的列
            corr.depth -= 1;
            binder.AddCorrelatedColumn(corr);
        }
    }

    // 4. 如果有子表达式（如 x IN (...)），绑定它
    if (expr.child) {
        auto error = Bind(expr.child, depth);
        if (error.HasError()) {
            return BindResult(std::move(error));
        }
    }

    // 5. 验证子查询返回列数
    if (expr.subquery_type != SubqueryType::EXISTS) {
        if (bound_node.types.size() != expected_columns) {
            throw BinderException("Subquery returns %zu columns - expected %d",
                                  bound_node.types.size(), expected_columns);
        }
    }

    // 6. 创建 BoundSubqueryExpression
    auto result = make_uniq<BoundSubqueryExpression>(return_type);
    result->binder = std::move(subquery_binder);
    result->subquery = std::move(bound_node);
    result->subquery_type = expr.subquery_type;
    result->comparison_type = expr.comparison_type;

    return BindResult(std::move(result));
}
```

### 2.9.3 相关子查询处理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          相关子查询示例                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SELECT * FROM orders o                                                      │
│  WHERE amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.id)   │
│                                                                             │
│  外部查询 (depth=0):                                                         │
│    orders 表绑定: table_index=0                                              │
│    o.id → ColumnBinding(0, 0)                                               │
│    o.amount → ColumnBinding(0, 1)                                           │
│                                                                             │
│  子查询 (depth=1):                                                           │
│    orders 表绑定: table_index=1                                              │
│    orders.customer_id → ColumnBinding(1, 2)                                 │
│    orders.amount → ColumnBinding(1, 1)                                      │
│                                                                             │
│  相关列处理:                                                                  │
│    o.id 在子查询中引用 → BoundColumnRefExpression(ColumnBinding(0, 0), depth=1) │
│    标记为相关列: CorrelatedColumnInfo(binding=(0,0), depth=1)                 │
│                                                                             │
│  执行时处理:                                                                  │
│    子查询变为依赖外部行的"参数化查询"                                          │
│    对于外部每一行，使用对应的 o.id 值执行子查询                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.10 CTE 绑定

### 2.10.1 普通 CTE 绑定

```cpp
// CTE 绑定示例
// WITH cte AS (SELECT * FROM t) SELECT * FROM cte, cte

optional_ptr<CTEBinding> Binder::GetCTEBinding(const BindingAlias &name) {
    // 查找 CTE 定义
    auto binding = bind_context.GetCTEBinding(name);
    if (binding) {
        return binding;
    }
    // 在父 Binder 中查找
    if (parent) {
        return parent->GetCTEBinding(name);
    }
    return nullptr;
}
```

### 2.10.2 递归 CTE 绑定

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          递归 CTE 绑定流程                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WITH RECURSIVE cte(n) AS (                                                  │
│      SELECT 1                    -- 锚点 (anchor)                            │
│      UNION ALL                                                               │
│      SELECT n + 1 FROM cte       -- 递归部分 (recursive)                     │
│      WHERE n < 10                                                            │
│  )                                                                           │
│  SELECT * FROM cte;                                                          │
│                                                                             │
│  绑定流程:                                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  1. 识别 RECURSIVE 关键字，设置递归 CTE 处理模式                   │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  2. 绑定锚点部分: SELECT 1                                        │       │
│  │     - 推导 CTE 列的初始类型                                        │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  3. 将 CTE 添加到绑定上下文                                        │       │
│  │     CTEType::CAN_BE_REFERENCED                                   │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  4. 绑定递归部分: SELECT n + 1 FROM cte WHERE n < 10             │       │
│  │     - cte 引用解析到前一步添加的绑定                               │       │
│  │     - 验证列类型兼容                                               │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │  5. 创建 RecursiveCTENode 包含:                                   │       │
│  │     - left: 锚点计划                                              │       │
│  │     - right: 递归计划                                             │       │
│  │     - union_all: true/false                                      │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.11 类型系统

### 2.11.1 类型推导

DuckDB 的类型推导遵循以下原则：

1. **列引用**: 类型来自 Catalog 中的表定义
2. **常量**: 类型由字面值决定（如 `42` 是 `INTEGER`，`'hello'` 是 `VARCHAR`）
3. **运算符**: 根据操作数类型推导结果类型
4. **函数**: 根据重载解析确定返回类型

```cpp
// 类型解析示例

// 获取最大兼容类型
LogicalType LogicalType::TryGetMaxLogicalType(ClientContext &context,
                                               const LogicalType &left,
                                               const LogicalType &right,
                                               LogicalType &result) {
    // 数值类型提升
    // TINYINT + INTEGER → INTEGER
    // INTEGER + DOUBLE → DOUBLE

    // 字符串连接
    // VARCHAR + VARCHAR → VARCHAR

    // 日期时间
    // DATE + INTERVAL → TIMESTAMP
}
```

### 2.11.2 隐式类型转换

```cpp
// src/planner/expression/bound_cast_expression.hpp

class BoundCastExpression : public Expression {
public:
    //! 添加类型转换
    static unique_ptr<Expression> AddCastToType(ClientContext &context,
                                                 unique_ptr<Expression> expr,
                                                 const LogicalType &target_type,
                                                 bool try_cast = false);

    unique_ptr<Expression> child;
    LogicalType source_type;
    BoundCastInfo bound_cast;  // 转换函数
    bool try_cast;             // 是否是 TRY_CAST
};
```

**隐式转换规则示例：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          隐式类型转换示例                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  表达式: age + 0.5                                                           │
│  age 类型: INTEGER                                                           │
│  0.5 类型: DOUBLE                                                            │
│                                                                             │
│  绑定结果:                                                                    │
│  BoundFunctionExpression {                                                  │
│      function: +(DOUBLE, DOUBLE) -> DOUBLE                                  │
│      children: [                                                            │
│          BoundCastExpression {                                              │
│              child: BoundColumnRefExpression(age, INTEGER)                  │
│              source_type: INTEGER                                           │
│              target_type: DOUBLE                                            │
│          },                                                                 │
│          BoundConstantExpression(0.5, DOUBLE)                               │
│      ]                                                                      │
│      return_type: DOUBLE                                                    │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.12 TableRef 绑定

### 2.12.1 BaseTableRef 绑定

```cpp
// src/planner/binder/tableref/bind_basetableref.cpp

BoundStatement Binder::Bind(BaseTableRef &ref) {
    // 1. 在 Catalog 中查找表
    auto entry = GetCatalogEntry(ref.catalog_name, ref.schema_name,
                                 EntryLookupInfo(CatalogType::TABLE_ENTRY, ref.table_name),
                                 OnEntryNotFound::THROW_EXCEPTION);

    // 2. 检查是否是 CTE
    auto cte_binding = GetCTEBinding(BindingAlias(ref.table_name));
    if (cte_binding) {
        return BindCTE(ref.table_name, *cte_binding);
    }

    // 3. 创建表扫描计划
    auto &table = entry->Cast<TableCatalogEntry>();
    auto table_index = GenerateTableIndex();

    // 4. 获取列信息
    auto &columns = table.GetColumns();
    vector<LogicalType> types;
    vector<string> names;
    for (auto &column : columns.Physical()) {
        types.push_back(column.Type());
        names.push_back(column.Name());
    }

    // 5. 添加到绑定上下文
    bind_context.AddBaseTable(table_index, ref.alias.empty() ? ref.table_name : ref.alias,
                              names, types, bound_column_ids, table);

    // 6. 创建 LogicalGet
    auto get = make_uniq<LogicalGet>(table_index, function,
                                      std::move(bind_data), types, names);

    BoundStatement result;
    result.plan = std::move(get);
    result.types = std::move(types);
    result.names = std::move(names);
    return result;
}
```

### 2.12.2 JoinRef 绑定

```cpp
// src/planner/binder/tableref/bind_joinref.cpp

BoundStatement Binder::Bind(JoinRef &ref) {
    // 1. 绑定左表
    auto left = Bind(*ref.left);

    // 2. 绑定右表
    auto right = Bind(*ref.right);

    // 3. 处理 USING 子句
    if (ref.using_columns.size() > 0) {
        // 将 USING 转换为等值连接条件
        for (auto &using_col : ref.using_columns) {
            // 创建 left.col = right.col 条件
        }
    }

    // 4. 绑定连接条件
    if (ref.condition) {
        // 创建新的绑定器来绑定 ON 条件
        // 两边的列都可见
    }

    // 5. 合并绑定上下文
    bind_context.AddContext(left_context);
    bind_context.AddContext(right_context);

    // 6. 对于 SEMI/ANTI JOIN，移除右表的绑定
    if (ref.join_type == JoinType::SEMI || ref.join_type == JoinType::ANTI) {
        bind_context.RemoveContext(right_aliases);
    }

    // 7. 创建 BoundJoinRef
    auto bound_join = make_uniq<BoundJoinRef>(ref.ref_type);
    bound_join->type = ref.join_type;
    bound_join->left = std::move(left);
    bound_join->right = std::move(right);
    bound_join->condition = std::move(condition);

    return CreatePlan(*bound_join);
}
```

---

## 2.13 BoundStatement - 绑定输出

### 2.13.1 BoundStatement 结构

```cpp
// src/include/duckdb/planner/bound_statement.hpp

struct BoundStatement {
    //! 初始逻辑计划（由 Binder 创建）
    unique_ptr<LogicalOperator> plan;
    //! 结果类型列表
    vector<LogicalType> types;
    //! 结果列名列表
    vector<string> names;
    //! 额外信息
    ExtraBoundInfo extra_info;
};

struct ExtraBoundInfo {
    //! 集合操作类型（UNION/INTERSECT/EXCEPT）
    SetOperationType setop_type = SetOperationType::NONE;
    //! 子 Binder（用于集合操作）
    vector<shared_ptr<Binder>> child_binders;
    //! 子语句绑定结果
    vector<BoundStatement> bound_children;
    //! 原始表达式（用于错误报告）
    vector<unique_ptr<ParsedExpression>> original_expressions;
};
```

---

## 2.14 完整绑定示例

让我们通过一个完整示例来理解整个绑定流程：

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
│                          完整绑定流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 绑定 FROM 子句                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  customers c (table_index=0):                                        │   │
│  │    id      → ColumnBinding(0, 0), INTEGER                           │   │
│  │    name    → ColumnBinding(0, 1), VARCHAR                           │   │
│  │    country → ColumnBinding(0, 2), VARCHAR                           │   │
│  │                                                                      │   │
│  │  orders o (table_index=1):                                          │   │
│  │    id          → ColumnBinding(1, 0), INTEGER                       │   │
│  │    customer_id → ColumnBinding(1, 1), INTEGER                       │   │
│  │    amount      → ColumnBinding(1, 2), DECIMAL                       │   │
│  │    date        → ColumnBinding(1, 3), DATE                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. 绑定 JOIN 条件                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  c.id = o.customer_id →                                             │   │
│  │  BoundComparisonExpression(                                         │   │
│  │      COMPARE_EQUAL,                                                 │   │
│  │      BoundColumnRefExpression(ColumnBinding(0, 0), INTEGER),        │   │
│  │      BoundColumnRefExpression(ColumnBinding(1, 1), INTEGER)         │   │
│  │  )                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. 绑定 WHERE 子句（使用 WhereBinder）                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  c.country = 'China' AND o.date >= '2024-01-01' →                   │   │
│  │  BoundConjunctionExpression(AND,                                    │   │
│  │      BoundComparisonExpression(EQUAL,                               │   │
│  │          BoundColumnRefExpression(ColumnBinding(0, 2), VARCHAR),    │   │
│  │          BoundConstantExpression('China', VARCHAR)),                │   │
│  │      BoundComparisonExpression(GREATER_THAN_OR_EQUAL,               │   │
│  │          BoundColumnRefExpression(ColumnBinding(1, 3), DATE),       │   │
│  │          BoundConstantExpression('2024-01-01', DATE))               │   │
│  │  )                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  4. 绑定 GROUP BY 子句（使用 GroupBinder）                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  c.name →                                                           │   │
│  │  BoundColumnRefExpression(ColumnBinding(0, 1), VARCHAR)             │   │
│  │  (添加到 group_expressions[0])                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  5. 绑定 SELECT 列表（使用 SelectBinder）                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  c.name →                                                           │   │
│  │    绑定到 GROUP BY 中的表达式（索引引用）                              │   │
│  │                                                                      │   │
│  │  SUM(o.amount) as total →                                           │   │
│  │  BoundAggregateExpression(                                          │   │
│  │      function: sum(DECIMAL) -> DECIMAL,                             │   │
│  │      children: [BoundColumnRefExpression(ColumnBinding(1, 2))]      │   │
│  │  )                                                                  │   │
│  │  (添加到 aggregates 列表)                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  6. 绑定 HAVING 子句（使用 HavingBinder）                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SUM(o.amount) > 1000 →                                             │   │
│  │  BoundComparisonExpression(GREATER_THAN,                            │   │
│  │      (引用已绑定的聚合表达式),                                         │   │
│  │      BoundConstantExpression(1000, INTEGER)                         │   │
│  │  )                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  7. 绑定 ORDER BY（使用 OrderBinder）                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  total DESC →                                                       │   │
│  │  别名 "total" 解析到 SELECT 列表第 2 列                              │   │
│  │  BoundOrderModifier(column_index=1, DESC, NULLS_LAST)               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  8. 绑定 LIMIT                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LIMIT 10 →                                                         │   │
│  │  BoundLimitModifier(limit_val: 10)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  输出 BoundSelectNode:                                                       │
│    types: [VARCHAR, DECIMAL]                                                │
│    names: ["name", "total"]                                                 │
│    groups.group_expressions: [c.name]                                       │
│    aggregates: [SUM(o.amount)]                                              │
│    where_clause: c.country='China' AND o.date>='2024-01-01'                 │
│    having: SUM(o.amount) > 1000                                             │
│    modifiers: [ORDER BY total DESC, LIMIT 10]                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2.15 源文件索引

| 组件 | 文件路径 |
|------|----------|
| Binder 类 | `src/include/duckdb/planner/binder.hpp` |
| BindContext | `src/include/duckdb/planner/bind_context.hpp` |
| ExpressionBinder | `src/include/duckdb/planner/expression_binder.hpp` |
| 表绑定 | `src/include/duckdb/planner/table_binding.hpp` |
| 列绑定 | `src/include/duckdb/planner/column_binding.hpp` |
| 绑定表达式 | `src/include/duckdb/planner/expression/` |
| SELECT 绑定 | `src/planner/binder/query_node/bind_select_node.cpp` |
| 表达式绑定 | `src/planner/binder/expression/` |
| 表引用绑定 | `src/planner/binder/tableref/` |
| 语句绑定 | `src/planner/binder/statement/` |

---

## 2.16 本章小结

本章详细剖析了 DuckDB 的 Binder（语义绑定器）组件：

1. **Binder 架构**：分层设计支持子查询和 CTE，GlobalBinderState 管理跨查询共享状态

2. **BindContext**：管理当前作用域内的表和列绑定，支持列名解析和歧义检测

3. **Binding 层次结构**：包括 TableBinding、CTEBinding、DummyBinding 等，统一管理不同类型的绑定

4. **ColumnBinding**：(table_index, column_index) 唯一标识列，是列引用解析的核心输出

5. **ExpressionBinder**：不同 SQL 子句使用专门的绑定器（WhereBinder、SelectBinder、GroupBinder 等）

6. **类型系统**：自动进行类型推导和隐式转换，函数重载解析

7. **子查询处理**：支持标量子查询、EXISTS、IN/ANY，正确处理相关列

8. **CTE 支持**：支持普通 CTE 和递归 CTE 的绑定

下一章我们将进入 Planner，了解如何将 BoundStatement 转换为 LogicalOperator 树。

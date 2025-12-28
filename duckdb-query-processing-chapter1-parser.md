# DuckDB 查询处理深度解析（一）：Parser - SQL 解析器

## 引言

Parser（解析器）是查询处理流程的第一个阶段，负责将 SQL 文本转换为结构化的抽象语法树（AST）。DuckDB 基于 PostgreSQL 的解析器 libpg_query，并通过 Transformer 将其转换为 DuckDB 内部的 AST 表示。

本章将深入分析 DuckDB 解析器的设计与实现，包括 Parser 类、Transformer、以及各种 AST 节点类型。

## 1. Parser 整体架构

### 1.1 解析流程概览

```
┌─────────────────────────────────────────────────────────────┐
│                    SQL 解析流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SQL 文本                                                    │
│  "SELECT a, b FROM t WHERE a > 10"                          │
│                     │                                       │
│                     ▼                                       │
│  ┌─────────────────────────────────────────────┐            │
│  │           libpg_query (PostgreSQL)          │            │
│  │                                              │            │
│  │   词法分析 → 语法分析 → PostgreSQL AST      │            │
│  └────────────────────┬────────────────────────┘            │
│                       │ PGNode 树                           │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │              Transformer                    │            │
│  │                                              │            │
│  │   PostgreSQL AST → DuckDB AST              │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │           DuckDB AST                        │            │
│  │                                              │            │
│  │   SQLStatement                              │            │
│  │   ├── ParsedExpression                      │            │
│  │   ├── TableRef                              │            │
│  │   └── QueryNode                             │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Parser 类

```cpp
// src/include/duckdb/parser/parser.hpp

class Parser {
public:
    explicit Parser(ParserOptions options = ParserOptions());

    //! 解析后的 SQL 语句列表
    vector<unique_ptr<SQLStatement>> statements;

public:
    //! 解析 SQL 查询，将结果存入 statements
    void ParseQuery(const string &query);

    //! 词法分析，返回 Token 列表
    static vector<SimplifiedToken> Tokenize(const string &query);

    //! 检查是否为关键字
    static KeywordCategory IsKeyword(const string &text);

    //! 返回所有关键字列表
    static vector<ParserKeyword> KeywordList();

    //! 辅助解析方法
    static vector<unique_ptr<ParsedExpression>> ParseExpressionList(
        const string &select_list, ParserOptions options = ParserOptions());

    static GroupByNode ParseGroupByList(
        const string &group_by, ParserOptions options = ParserOptions());

    static vector<OrderByNode> ParseOrderList(
        const string &select_list, ParserOptions options = ParserOptions());

    static ColumnList ParseColumnList(
        const string &column_list, ParserOptions options = ParserOptions());

private:
    ParserOptions options;
};
```

### 1.3 ParserOptions

```cpp
// src/include/duckdb/parser/parser_options.hpp

struct ParserOptions {
    //! 是否保留 SQL 注释
    bool preserve_sql_comments = false;
    //! 是否允许 prepared statement 参数
    bool allow_prepared_params = true;
    //! 参数类型（? 或 $1）
    PreparedParamType param_type = PreparedParamType::AUTO_DETECT;
    //! 扩展列表
    case_insensitive_map_t<ParserExtension> extensions;
    //! 最大表达式深度限制
    idx_t max_expression_depth = 500;
};
```

## 2. libpg_query 集成

### 2.1 PostgreSQL 解析器

DuckDB 使用 libpg_query 库，这是从 PostgreSQL 源码中提取的独立解析器：

```cpp
// 解析过程调用 libpg_query
namespace duckdb_libpgquery {
    struct PGNode;      // PostgreSQL AST 节点基类
    struct PGList;      // PostgreSQL 链表
    struct PGSelectStmt; // SELECT 语句
    // ... 其他 PostgreSQL 类型
}
```

### 2.2 解析器调用

```cpp
void Parser::ParseQuery(const string &query) {
    // 1. 预处理：移除 Unicode 空格
    string clean_query;
    if (StripUnicodeSpaces(query, clean_query)) {
        // 使用清理后的查询
    }

    // 2. 调用 libpg_query 解析
    duckdb_libpgquery::PGList *parse_tree = ...;

    // 3. 转换为 DuckDB AST
    Transformer transformer(options);
    transformer.TransformParseTree(parse_tree, statements);
}
```

## 3. Transformer：AST 转换器

### 3.1 Transformer 类设计

```cpp
// src/include/duckdb/parser/transformer.hpp

class Transformer {
public:
    explicit Transformer(ParserOptions &options);
    Transformer(Transformer &parent);  // 子查询使用

    //! 转换解析树为 SQL 语句列表
    bool TransformParseTree(duckdb_libpgquery::PGList *tree,
                            vector<unique_ptr<SQLStatement>> &statements);

    //! 获取参数数量
    idx_t ParamCount() const;

private:
    optional_ptr<Transformer> parent;
    ParserOptions &options;

    //! Prepared statement 参数索引
    idx_t prepared_statement_parameter_index = 0;
    //! 命名参数映射
    case_insensitive_map_t<idx_t> named_param_map;

    //! 窗口定义（按名称存储）
    case_insensitive_map_t<duckdb_libpgquery::PGWindowDef *> window_clauses;
    //! CTE 存储
    vector<reference<CommonTableExpressionMap>> stored_cte_map;
};
```

### 3.2 语句转换方法

```cpp
// 主转换入口
unique_ptr<SQLStatement> Transformer::TransformStatement(
    duckdb_libpgquery::PGNode &stmt);

// 各类语句转换
unique_ptr<SelectStatement> TransformSelectStmt(PGSelectStmt &select);
unique_ptr<InsertStatement> TransformInsert(PGInsertStmt &stmt);
unique_ptr<UpdateStatement> TransformUpdate(PGUpdateStmt &stmt);
unique_ptr<DeleteStatement> TransformDelete(PGDeleteStmt &stmt);
unique_ptr<CreateStatement> TransformCreateTable(PGCreateStmt &node);
unique_ptr<CreateStatement> TransformCreateView(PGViewStmt &node);
unique_ptr<CreateStatement> TransformCreateIndex(PGIndexStmt &stmt);
unique_ptr<AlterStatement> TransformAlter(PGAlterTableStmt &stmt);
unique_ptr<SQLStatement> TransformDrop(PGDropStmt &stmt);
unique_ptr<CopyStatement> TransformCopy(PGCopyStmt &stmt);
unique_ptr<ExplainStatement> TransformExplain(PGExplainStmt &stmt);
// ... 更多语句类型
```

### 3.3 表达式转换方法

```cpp
// 主表达式转换
unique_ptr<ParsedExpression> TransformExpression(PGNode &node);

// 各类表达式转换
unique_ptr<ParsedExpression> TransformBoolExpr(PGBoolExpr &root);
unique_ptr<ParsedExpression> TransformCase(PGCaseExpr &root);
unique_ptr<ParsedExpression> TransformTypeCast(PGTypeCast &root);
unique_ptr<ParsedExpression> TransformColumnRef(PGColumnRef &root);
unique_ptr<ConstantExpression> TransformValue(PGValue val);
unique_ptr<ParsedExpression> TransformAExpr(PGAExpr &root);
unique_ptr<ParsedExpression> TransformFuncCall(PGFuncCall &root);
unique_ptr<ParsedExpression> TransformSubquery(PGSubLink &root);
unique_ptr<ParsedExpression> TransformInterval(PGIntervalConstant &root);
unique_ptr<ParsedExpression> TransformLambda(PGLambdaFunction &node);
// ... 更多表达式类型
```

### 3.4 表引用转换方法

```cpp
// 主表引用转换
unique_ptr<TableRef> TransformTableRefNode(PGNode &n);
unique_ptr<TableRef> TransformFrom(PGList *root);

// 各类表引用转换
unique_ptr<TableRef> TransformRangeVar(PGRangeVar &root);
unique_ptr<TableRef> TransformRangeFunction(PGRangeFunction &root);
unique_ptr<TableRef> TransformJoin(PGJoinExpr &root);
unique_ptr<TableRef> TransformRangeSubselect(PGRangeSubselect &root);
unique_ptr<TableRef> TransformValuesList(PGList *list);
unique_ptr<TableRef> TransformPivot(PGPivotExpr &root);
```

## 4. SQLStatement 层次结构

### 4.1 基类设计

```cpp
// src/include/duckdb/parser/sql_statement.hpp

class SQLStatement {
public:
    static constexpr const StatementType TYPE = StatementType::INVALID_STATEMENT;

    explicit SQLStatement(StatementType type) : type(type) {}
    virtual ~SQLStatement() {}

    //! 语句类型
    StatementType type;
    //! 在查询字符串中的位置
    idx_t stmt_location = 0;
    //! 语句长度
    idx_t stmt_length = 0;
    //! 命名参数映射
    case_insensitive_map_t<idx_t> named_param_map;
    //! 对应的 SQL 文本
    string query;

    virtual string ToString() const = 0;
    virtual unique_ptr<SQLStatement> Copy() const = 0;

    template <class TARGET>
    TARGET &Cast() {
        if (type != TARGET::TYPE && TARGET::TYPE != StatementType::INVALID_STATEMENT) {
            throw InternalException("Statement type mismatch");
        }
        return reinterpret_cast<TARGET &>(*this);
    }
};
```

### 4.2 语句类型枚举

```cpp
// src/include/duckdb/common/enums/statement_type.hpp

enum class StatementType : uint8_t {
    INVALID_STATEMENT,
    SELECT_STATEMENT,       // SELECT
    INSERT_STATEMENT,       // INSERT
    UPDATE_STATEMENT,       // UPDATE
    DELETE_STATEMENT,       // DELETE
    CREATE_STATEMENT,       // CREATE TABLE/VIEW/INDEX/...
    DROP_STATEMENT,         // DROP
    ALTER_STATEMENT,        // ALTER
    PREPARE_STATEMENT,      // PREPARE
    EXECUTE_STATEMENT,      // EXECUTE
    TRANSACTION_STATEMENT,  // BEGIN/COMMIT/ROLLBACK
    COPY_STATEMENT,         // COPY
    EXPLAIN_STATEMENT,      // EXPLAIN
    PRAGMA_STATEMENT,       // PRAGMA
    SET_STATEMENT,          // SET
    LOAD_STATEMENT,         // LOAD
    CALL_STATEMENT,         // CALL
    EXPORT_STATEMENT,       // EXPORT DATABASE
    ATTACH_STATEMENT,       // ATTACH
    DETACH_STATEMENT,       // DETACH
    VACUUM_STATEMENT,       // VACUUM
    MERGE_INTO_STATEMENT,   // MERGE INTO
    // ...
};
```

### 4.3 主要语句类型

```
┌─────────────────────────────────────────────────────────────┐
│                   SQLStatement 层次结构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SQLStatement (基类)                                         │
│  ├── SelectStatement                                        │
│  │   └── node: QueryNode (SELECT 查询结构)                  │
│  │                                                          │
│  ├── InsertStatement                                        │
│  │   ├── table: string                                      │
│  │   ├── columns: vector<string>                            │
│  │   ├── select_statement: SelectStatement                  │
│  │   └── on_conflict_info: OnConflictInfo                   │
│  │                                                          │
│  ├── UpdateStatement                                        │
│  │   ├── table: TableRef                                    │
│  │   ├── set_info: UpdateSetInfo                            │
│  │   ├── condition: ParsedExpression                        │
│  │   └── from_table: TableRef                               │
│  │                                                          │
│  ├── DeleteStatement                                        │
│  │   ├── table: TableRef                                    │
│  │   ├── condition: ParsedExpression                        │
│  │   └── using_clauses: vector<TableRef>                    │
│  │                                                          │
│  ├── CreateStatement                                        │
│  │   └── info: CreateInfo (多态)                            │
│  │       ├── CreateTableInfo                                │
│  │       ├── CreateViewInfo                                 │
│  │       ├── CreateIndexInfo                                │
│  │       ├── CreateSchemaInfo                               │
│  │       └── ...                                            │
│  │                                                          │
│  ├── DropStatement                                          │
│  │   └── info: DropInfo                                     │
│  │                                                          │
│  ├── AlterStatement                                         │
│  │   └── info: AlterInfo (多态)                             │
│  │                                                          │
│  ├── CopyStatement                                          │
│  │   └── info: CopyInfo                                     │
│  │                                                          │
│  ├── ExplainStatement                                       │
│  │   └── stmt: SQLStatement                                 │
│  │                                                          │
│  └── ... (其他语句类型)                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 5. QueryNode 结构

### 5.1 QueryNode 基类

```cpp
// src/include/duckdb/parser/query_node.hpp

enum class QueryNodeType : uint8_t {
    SELECT_NODE = 1,         // 普通 SELECT
    SET_OPERATION_NODE = 2,  // UNION/INTERSECT/EXCEPT
    BOUND_SUBQUERY_NODE = 3, // 绑定后的子查询
    RECURSIVE_CTE_NODE = 4,  // 递归 CTE
    CTE_NODE = 5,            // 普通 CTE
    STATEMENT_NODE = 6       // 语句节点
};

class QueryNode {
public:
    explicit QueryNode(QueryNodeType type) : type(type) {}
    virtual ~QueryNode() {}

    //! 节点类型
    QueryNodeType type;
    //! 结果修饰符（ORDER BY, LIMIT 等）
    vector<unique_ptr<ResultModifier>> modifiers;
    //! CTE 定义
    CommonTableExpressionMap cte_map;

    virtual string ToString() const = 0;
    virtual unique_ptr<QueryNode> Copy() const = 0;
};
```

### 5.2 SelectNode

```cpp
// src/include/duckdb/parser/query_node/select_node.hpp

class SelectNode : public QueryNode {
public:
    static constexpr const QueryNodeType TYPE = QueryNodeType::SELECT_NODE;

    SelectNode();

    //! SELECT 列表
    vector<unique_ptr<ParsedExpression>> select_list;

    //! FROM 子句
    unique_ptr<TableRef> from_table;

    //! WHERE 子句
    unique_ptr<ParsedExpression> where_clause;

    //! GROUP BY 子句
    GroupByNode groups;

    //! HAVING 子句
    unique_ptr<ParsedExpression> having;

    //! QUALIFY 子句（窗口函数过滤）
    unique_ptr<ParsedExpression> qualify;

    //! 聚合处理模式
    AggregateHandling aggregate_handling;

    //! SAMPLE 子句
    unique_ptr<SampleOptions> sample;
};
```

### 5.3 SetOperationNode

```cpp
// src/include/duckdb/parser/query_node/set_operation_node.hpp

enum class SetOperationType : uint8_t {
    NONE = 0,
    UNION = 1,
    EXCEPT = 2,
    INTERSECT = 3,
    UNION_BY_NAME = 4
};

class SetOperationNode : public QueryNode {
public:
    static constexpr const QueryNodeType TYPE = QueryNodeType::SET_OPERATION_NODE;

    //! 左侧查询
    unique_ptr<QueryNode> left;

    //! 右侧查询
    unique_ptr<QueryNode> right;

    //! 操作类型
    SetOperationType setop_type = SetOperationType::NONE;

    //! 是否 UNION ALL
    bool setop_all = false;
};
```

### 5.4 RecursiveCTENode

```cpp
// src/include/duckdb/parser/query_node/recursive_cte_node.hpp

class RecursiveCTENode : public QueryNode {
public:
    static constexpr const QueryNodeType TYPE = QueryNodeType::RECURSIVE_CTE_NODE;

    //! CTE 名称
    string ctename;

    //! 是否 UNION ALL
    bool union_all;

    //! 锚点查询（非递归部分）
    unique_ptr<QueryNode> left;

    //! 递归查询
    unique_ptr<QueryNode> right;

    //! 别名
    vector<string> aliases;

    //! 物化策略
    CTEMaterialize cte_materialize = CTEMaterialize::CTE_MATERIALIZE_DEFAULT;
};
```

## 6. ParsedExpression 层次结构

### 6.1 基类设计

```cpp
// src/include/duckdb/parser/parsed_expression.hpp

class ParsedExpression : public BaseExpression {
public:
    ParsedExpression(ExpressionType type, ExpressionClass expression_class)
        : BaseExpression(type, expression_class) {}

    //! 是否包含聚合函数
    bool IsAggregate() const override;
    //! 是否包含窗口函数
    bool IsWindow() const override;
    //! 是否包含子查询
    bool HasSubquery() const override;
    //! 是否为标量表达式
    bool IsScalar() const override;
    //! 是否包含参数
    bool HasParameter() const override;

    virtual unique_ptr<ParsedExpression> Copy() const = 0;
};
```

### 6.2 主要表达式类型

```
┌─────────────────────────────────────────────────────────────┐
│               ParsedExpression 层次结构                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ParsedExpression (基类)                                     │
│  │                                                          │
│  ├── ColumnRefExpression                                    │
│  │   ├── column_names: vector<string>                       │
│  │   └── 示例: "a", "t.a", "schema.t.a"                     │
│  │                                                          │
│  ├── ConstantExpression                                     │
│  │   └── value: Value                                       │
│  │                                                          │
│  ├── FunctionExpression                                     │
│  │   ├── function_name: string                              │
│  │   ├── schema: string                                     │
│  │   ├── children: vector<ParsedExpression>                 │
│  │   ├── filter: ParsedExpression (聚合过滤)                │
│  │   ├── order_bys: OrderModifier                           │
│  │   ├── distinct: bool                                     │
│  │   └── is_operator: bool                                  │
│  │                                                          │
│  ├── ComparisonExpression                                   │
│  │   ├── left: ParsedExpression                             │
│  │   └── right: ParsedExpression                            │
│  │                                                          │
│  ├── ConjunctionExpression                                  │
│  │   └── children: vector<ParsedExpression>                 │
│  │                                                          │
│  ├── OperatorExpression                                     │
│  │   └── children: vector<ParsedExpression>                 │
│  │                                                          │
│  ├── CastExpression                                         │
│  │   ├── child: ParsedExpression                            │
│  │   ├── cast_type: LogicalType                             │
│  │   └── try_cast: bool                                     │
│  │                                                          │
│  ├── CaseExpression                                         │
│  │   ├── case_checks: vector<CaseCheck>                     │
│  │   └── else_expr: ParsedExpression                        │
│  │                                                          │
│  ├── SubqueryExpression                                     │
│  │   ├── subquery: SelectStatement                          │
│  │   ├── subquery_type: SubqueryType                        │
│  │   │   (SCALAR, EXISTS, NOT_EXISTS, ANY, ALL)             │
│  │   └── comparison_type: ExpressionType                    │
│  │                                                          │
│  ├── WindowExpression                                       │
│  │   ├── function_name: string                              │
│  │   ├── children: vector<ParsedExpression>                 │
│  │   ├── partitions: vector<ParsedExpression>               │
│  │   ├── orders: vector<OrderByNode>                        │
│  │   ├── start/end: WindowBoundary                          │
│  │   └── filter_expr: ParsedExpression                      │
│  │                                                          │
│  ├── StarExpression                                         │
│  │   ├── relation_name: string                              │
│  │   ├── exclude_list: vector<string>                       │
│  │   ├── replace_list: vector<pair<string, expr>>           │
│  │   └── columns: bool (COLUMNS(*))                         │
│  │                                                          │
│  ├── ParameterExpression                                    │
│  │   ├── parameter_nr: idx_t                                │
│  │   └── identifier: string                                 │
│  │                                                          │
│  ├── LambdaExpression                                       │
│  │   ├── lhs: ParsedExpression (参数)                       │
│  │   └── expr: ParsedExpression (主体)                      │
│  │                                                          │
│  ├── CollateExpression                                      │
│  │   ├── child: ParsedExpression                            │
│  │   └── collation: string                                  │
│  │                                                          │
│  ├── BetweenExpression                                      │
│  │   ├── input: ParsedExpression                            │
│  │   ├── lower: ParsedExpression                            │
│  │   └── upper: ParsedExpression                            │
│  │                                                          │
│  ├── PositionalReferenceExpression                          │
│  │   └── index: idx_t (如 #1, #2)                           │
│  │                                                          │
│  └── DefaultExpression                                      │
│      └── (DEFAULT 关键字)                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 ColumnRefExpression 示例

```cpp
// src/include/duckdb/parser/expression/columnref_expression.hpp

class ColumnRefExpression : public ParsedExpression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::COLUMN_REF;

    ColumnRefExpression();
    ColumnRefExpression(string column_name);
    ColumnRefExpression(string column_name, string table_name);

    //! 列名组成部分（从右到左：列名、表名、schema 名）
    vector<string> column_names;

    //! 判断是否为限定名（如 t.a）
    bool IsQualified() const { return column_names.size() > 1; }

    //! 获取列名（最右边的部分）
    const string &GetColumnName() const;

    //! 获取表名（倒数第二个部分）
    const string &GetTableName() const;
};
```

### 6.4 FunctionExpression 示例

```cpp
// src/include/duckdb/parser/expression/function_expression.hpp

class FunctionExpression : public ParsedExpression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::FUNCTION;

    //! 函数名
    string function_name;
    //! Schema 名
    string schema;
    //! Catalog 名
    string catalog;
    //! 参数列表
    vector<unique_ptr<ParsedExpression>> children;
    //! 聚合过滤 (FILTER WHERE)
    unique_ptr<ParsedExpression> filter;
    //! 排序 (ORDER BY for ordered aggregates)
    unique_ptr<OrderModifier> order_bys;
    //! DISTINCT 标志
    bool distinct;
    //! 是否为运算符形式
    bool is_operator;
    //! 导出状态
    bool export_state;

    //! 判断是否为聚合函数
    bool IsAggregate() const override;
};
```

## 7. TableRef 层次结构

### 7.1 基类设计

```cpp
// src/include/duckdb/parser/tableref.hpp

enum class TableReferenceType : uint8_t {
    INVALID,
    BASE_TABLE,         // 基本表
    SUBQUERY,           // 子查询
    JOIN,               // JOIN
    TABLE_FUNCTION,     // 表函数
    EXPRESSION_LIST,    // VALUES 列表
    COLUMN_DATA,        // 列数据
    PIVOT,              // PIVOT
    SHOW_REF,           // SHOW
    EMPTY               // 空（无 FROM）
};

class TableRef {
public:
    explicit TableRef(TableReferenceType type) : type(type) {}
    virtual ~TableRef() {}

    TableReferenceType type;
    //! 别名
    string alias;
    //! SAMPLE 选项
    unique_ptr<SampleOptions> sample;
    //! 查询位置
    optional_idx query_location;
    //! 列别名
    vector<string> column_name_alias;

    virtual string ToString() const = 0;
    virtual unique_ptr<TableRef> Copy() = 0;
};
```

### 7.2 主要表引用类型

```
┌─────────────────────────────────────────────────────────────┐
│                    TableRef 层次结构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TableRef (基类)                                             │
│  │                                                          │
│  ├── BaseTableRef                                           │
│  │   ├── schema_name: string                                │
│  │   ├── table_name: string                                 │
│  │   ├── catalog_name: string                               │
│  │   └── 示例: FROM my_table, FROM schema.table             │
│  │                                                          │
│  ├── SubqueryRef                                            │
│  │   └── subquery: SelectStatement                          │
│  │   └── 示例: FROM (SELECT ...) AS sub                     │
│  │                                                          │
│  ├── JoinRef                                                │
│  │   ├── left: TableRef                                     │
│  │   ├── right: TableRef                                    │
│  │   ├── condition: ParsedExpression                        │
│  │   ├── type: JoinType                                     │
│  │   │   (INNER, LEFT, RIGHT, FULL, CROSS, SEMI, ANTI,     │
│  │   │    MARK, SINGLE, POSITION, ASOF)                     │
│  │   ├── ref_type: JoinRefType                              │
│  │   │   (REGULAR, NATURAL, CROSS, POSITIONAL, ASOF)        │
│  │   └── using_columns: vector<string>                      │
│  │                                                          │
│  ├── TableFunctionRef                                       │
│  │   ├── function: FunctionExpression                       │
│  │   └── 示例: FROM read_csv('file.csv')                    │
│  │                                                          │
│  ├── ExpressionListRef                                      │
│  │   └── values: vector<vector<ParsedExpression>>           │
│  │   └── 示例: VALUES (1, 'a'), (2, 'b')                    │
│  │                                                          │
│  ├── PivotRef                                               │
│  │   ├── source: TableRef                                   │
│  │   ├── aggregates: vector<PivotColumn>                    │
│  │   ├── pivots: vector<PivotColumn>                        │
│  │   ├── groups: vector<string>                             │
│  │   └── include_nulls: bool                                │
│  │                                                          │
│  └── EmptyTableRef                                          │
│      └── 示例: SELECT 1 + 1 (无 FROM)                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 JoinRef 示例

```cpp
// src/include/duckdb/parser/tableref/joinref.hpp

enum class JoinType : uint8_t {
    INVALID,
    LEFT,       // LEFT [OUTER] JOIN
    RIGHT,      // RIGHT [OUTER] JOIN
    INNER,      // INNER JOIN
    OUTER,      // FULL OUTER JOIN
    CROSS,      // CROSS JOIN (笛卡尔积)
    SEMI,       // LEFT SEMI JOIN
    ANTI,       // LEFT ANTI JOIN
    MARK,       // MARK JOIN (用于 EXISTS)
    SINGLE,     // SINGLE JOIN (最多一条匹配)
    POSITION,   // POSITIONAL JOIN
    ASOF        // ASOF JOIN (时序数据)
};

enum class JoinRefType : uint8_t {
    REGULAR,    // 普通 JOIN (ON 条件)
    NATURAL,    // NATURAL JOIN
    CROSS,      // CROSS JOIN
    POSITIONAL, // POSITIONAL JOIN
    ASOF,       // ASOF JOIN
    DEPENDENT   // DEPENDENT JOIN
};

class JoinRef : public TableRef {
public:
    static constexpr const TableReferenceType TYPE = TableReferenceType::JOIN;

    //! 左表
    unique_ptr<TableRef> left;
    //! 右表
    unique_ptr<TableRef> right;
    //! JOIN 条件
    unique_ptr<ParsedExpression> condition;
    //! JOIN 类型
    JoinType type;
    //! JOIN 引用类型
    JoinRefType ref_type;
    //! USING 列
    vector<string> using_columns;
};
```

## 8. ResultModifier 系统

### 8.1 ResultModifier 类型

```cpp
// src/include/duckdb/parser/result_modifier.hpp

enum class ResultModifierType : uint8_t {
    LIMIT_MODIFIER = 1,
    ORDER_MODIFIER = 2,
    DISTINCT_MODIFIER = 3,
    LIMIT_PERCENT_MODIFIER = 4
};

class ResultModifier {
public:
    explicit ResultModifier(ResultModifierType type) : type(type) {}
    ResultModifierType type;
};

// LIMIT 修饰符
class LimitModifier : public ResultModifier {
    unique_ptr<ParsedExpression> limit;   // LIMIT 值
    unique_ptr<ParsedExpression> offset;  // OFFSET 值
};

// ORDER BY 修饰符
class OrderModifier : public ResultModifier {
    vector<OrderByNode> orders;
};

// DISTINCT 修饰符
class DistinctModifier : public ResultModifier {
    vector<unique_ptr<ParsedExpression>> distinct_on_targets;
};
```

### 8.2 OrderByNode

```cpp
struct OrderByNode {
    OrderType type;              // ASC/DESC
    OrderByNullType null_order;  // NULLS FIRST/LAST
    unique_ptr<ParsedExpression> expression;
};
```

## 9. CTE（公共表表达式）

### 9.1 CommonTableExpressionMap

```cpp
// src/include/duckdb/parser/query_node.hpp

class CommonTableExpressionMap {
public:
    InsertionOrderPreservingMap<unique_ptr<CommonTableExpressionInfo>> map;

    string ToString() const;
    CommonTableExpressionMap Copy() const;
};
```

### 9.2 CommonTableExpressionInfo

```cpp
// src/include/duckdb/parser/common_table_expression_info.hpp

struct CommonTableExpressionInfo {
    //! CTE 别名
    vector<string> aliases;
    //! CTE 查询
    unique_ptr<SelectStatement> query;
    //! 物化策略
    CTEMaterialize materialized = CTEMaterialize::CTE_MATERIALIZE_DEFAULT;
};
```

## 10. 解析示例

### 10.1 简单 SELECT

```sql
SELECT a, b FROM t WHERE a > 10 ORDER BY b LIMIT 100;
```

解析结果：

```
SelectStatement
└── node: SelectNode
    ├── select_list:
    │   ├── ColumnRefExpression("a")
    │   └── ColumnRefExpression("b")
    ├── from_table: BaseTableRef("t")
    ├── where_clause:
    │   └── ComparisonExpression(GREATER_THAN)
    │       ├── left: ColumnRefExpression("a")
    │       └── right: ConstantExpression(10)
    └── modifiers:
        ├── OrderModifier
        │   └── orders: [OrderByNode(ColumnRef("b"), ASC)]
        └── LimitModifier
            └── limit: ConstantExpression(100)
```

### 10.2 带 JOIN 的查询

```sql
SELECT a.id, b.name
FROM table_a a
LEFT JOIN table_b b ON a.id = b.a_id
WHERE b.active = true;
```

解析结果：

```
SelectStatement
└── node: SelectNode
    ├── select_list:
    │   ├── ColumnRefExpression("a", "id")
    │   └── ColumnRefExpression("b", "name")
    ├── from_table: JoinRef
    │   ├── left: BaseTableRef("table_a", alias="a")
    │   ├── right: BaseTableRef("table_b", alias="b")
    │   ├── type: LEFT
    │   └── condition: ComparisonExpression(EQUAL)
    │       ├── left: ColumnRefExpression("a", "id")
    │       └── right: ColumnRefExpression("b", "a_id")
    └── where_clause:
        └── ComparisonExpression(EQUAL)
            ├── left: ColumnRefExpression("b", "active")
            └── right: ConstantExpression(true)
```

### 10.3 带 CTE 和聚合的查询

```sql
WITH monthly_sales AS (
    SELECT month, SUM(amount) as total
    FROM sales
    GROUP BY month
)
SELECT month, total
FROM monthly_sales
WHERE total > 1000
ORDER BY total DESC;
```

解析结果：

```
SelectStatement
└── node: SelectNode
    ├── cte_map:
    │   └── "monthly_sales": CommonTableExpressionInfo
    │       └── query: SelectStatement
    │           └── node: SelectNode
    │               ├── select_list:
    │               │   ├── ColumnRefExpression("month")
    │               │   └── FunctionExpression("sum")
    │               │       └── children: [ColumnRef("amount")]
    │               │       └── alias: "total"
    │               ├── from_table: BaseTableRef("sales")
    │               └── groups: GroupByNode
    │                   └── expressions: [ColumnRef("month")]
    ├── select_list:
    │   ├── ColumnRefExpression("month")
    │   └── ColumnRefExpression("total")
    ├── from_table: BaseTableRef("monthly_sales")
    ├── where_clause: ComparisonExpression(GREATER_THAN)
    │   ├── left: ColumnRefExpression("total")
    │   └── right: ConstantExpression(1000)
    └── modifiers:
        └── OrderModifier
            └── orders: [OrderByNode(ColumnRef("total"), DESC)]
```

## 11. Parser 扩展机制

### 11.1 ParserExtension

```cpp
// src/include/duckdb/parser/parser_extension.hpp

struct ParserExtension {
    //! 解析函数
    parser_extension_parse_function_t parse_function;
    //! 用户数据
    shared_ptr<ParserExtensionData> data;
};

struct ParserExtensionParseResult {
    enum class ParserExtensionResultType { PARSE_SUCCESSFUL, PARSE_ERROR };

    ParserExtensionResultType type;
    string error_message;
    unique_ptr<SQLStatement> statement;
};
```

### 11.2 扩展使用

扩展可以注册自定义解析函数来处理特殊语法：

```cpp
// 注册扩展
ParserOptions options;
options.extensions["my_extension"] = ParserExtension{
    my_parse_function,
    make_shared<MyExtensionData>()
};

// 解析器会在标准解析失败时尝试扩展
```

## 12. 总结

### 12.1 核心组件

| 组件 | 功能 |
|------|------|
| Parser | 解析入口，调用 libpg_query |
| Transformer | PostgreSQL AST → DuckDB AST |
| SQLStatement | 语句抽象基类 |
| QueryNode | 查询节点（SELECT, UNION 等） |
| ParsedExpression | 表达式抽象基类 |
| TableRef | 表引用抽象基类 |
| ResultModifier | 结果修饰符（ORDER, LIMIT 等） |

### 12.2 设计优势

1. **复用成熟解析器**：基于 PostgreSQL 解析器，语法兼容性好
2. **清晰的 AST 层次**：Statement → QueryNode → Expression/TableRef
3. **类型安全转换**：使用 Cast<T>() 模板方法
4. **可扩展**：支持 ParserExtension 自定义语法
5. **位置追踪**：保存查询位置信息，便于错误报告

### 12.3 下一步

Parser 完成后，生成的 AST 将传递给 Binder（绑定器）进行语义分析，包括：
- 符号解析（表名、列名）
- 类型推导和检查
- 函数重载解析
- 子查询处理

在下一章中，我们将深入分析 Binder 的设计与实现。

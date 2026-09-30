# 第 4 周：Parser、Binder 与 Catalog

[上一周：类型与向量](week-03-types-and-vectors.md) · [返回总计划](learning-plan.md) · [下一周：优化器](week-05-optimizer.md)

本周回答一个核心问题：SQL 语法正确之后，DuckDB 如何确定表、列、函数和类型的具体含义。你会用一组刻意失败的查询区分 Parser、Binder 与 Catalog 的职责。

## 本周目标

完成本周后，你应当能够：

- 从 PEG grammar 追到 `SelectNode` 的 transformer。
- 解释 Parser 为什么无法独自判断表或列是否存在。
- 描述 Binder 如何先绑定 FROM，再建立名称作用域并绑定表达式。
- 说明 Catalog 在表名解析中的作用。
- 根据错误输入预测失败阶段，并用实际错误核对。

## 前置条件与本周范围

先完成第 2、3 周，能区分语法对象与绑定后的逻辑计划，并理解逻辑类型。本周不修改 grammar，不运行代码生成，不深入优化器。

所有命令都从仓库根目录执行：

```bash
cd /Users/darion.yaphet/source/duckdb
```

下面每条失败命令都启动一个全新的 `:memory:` 数据库。命令返回非零是预期现象；逐条执行，记录错误前缀与定位，不要用一个长命令掩盖中间失败。

## 第 1 次学习：用失败查询划分阶段（1.5 小时）

### 语法错误

```bash
build/reldebug/duckdb :memory: -c "SELEC 42;"
```

可核对：错误以 `Parser Error:` 开头，并将位置指向无法继续匹配的 token 附近。

### 表不存在

```bash
build/reldebug/duckdb :memory: -c "SELECT * FROM missing_table;"
```

可核对：当前版本报告 `Catalog Error: Table with name missing_table does not exist!`。它发生在绑定表引用时的目录查找阶段；“Catalog Error” 是错误类别，不表示 Parser 负责查表。

### 列不存在

```bash
build/reldebug/duckdb :memory: -c "
CREATE TABLE t(a INTEGER);
SELECT b FROM t;
"
```

可核对：`Binder Error: Referenced column "b" not found in FROM clause!`，候选绑定包含 `a`。

### 列名歧义

```bash
build/reldebug/duckdb :memory: -c "
CREATE TABLE l(id INTEGER);
CREATE TABLE r(id INTEGER);
SELECT id FROM l JOIN r ON l.id = r.id;
"
```

可核对：`Binder Error: Ambiguous reference to column name "id"`，提示使用 `l.id` 或 `r.id`。

### 函数签名或类型不匹配

```bash
build/reldebug/duckdb :memory: -c "
SELECT DATE '2024-01-01' + DATE '2024-01-02';
"
```

可核对：`Binder Error: No function matches ... '+(DATE, DATE)'`，随后列出候选重载。

为五个例子分别记录：Parser 是否能构造语法树、是否需要查 Catalog、是否需要名称作用域、是否需要函数重载解析。

## 第 2 次学习：从 PEG grammar 到 SelectNode（2 小时）

在 [select.gram](../src/parser/peg/grammar/statements/select.gram) 中按规则名搜索，不要从头背完整文件：

```text
SelectStatement
  -> SelectStatementInternal
  -> SelectStatementBody
  -> SimpleSelect
  -> SelectFrom + WhereClause? + GroupByClause? + ...
  -> SelectClause + FromClause?
```

用第 1 周主线查询标注：

- `SELECT customer_id, SUM(amount) ...` 对应 `SelectClause` 和 `TargetList`。
- `FROM orders` 对应 `FromClause` 与 `BaseTableRef`。
- `WHERE ...` 对应 `WhereClause`。
- `GROUP BY customer_id` 对应 `GroupByClause`。
- `ORDER BY`、`LIMIT` 属于 `ResultModifiers`。

接着读 [transform_select.cpp](../src/parser/peg/transformer/transform_select.cpp)：

- `TransformSelectClause` 创建 `SelectNode` 并填充 `select_list`。
- `TransformSimpleSelect` 把 WHERE、GROUP BY、HAVING、QUALIFY 和 SAMPLE 放进 node。
- `TransformSelectStatementInternalRule` 处理 CTE 与结果修饰器。

此阶段只知道文本结构。比如 `customer_id` 是一个列引用形式，但还不知道它属于哪张表、类型是什么。

## 第 3 次学习：Binder 如何赋予 SQL 含义（2 小时）

先读 `Binder::BindNode(SelectNode &)`。当前实现明确先取出并绑定 `from_table`，再调用 `BindSelectNode`。这一步为后续列解析准备作用域。

再读 `Binder::Bind(BaseTableRef &)` 的主线：

1. 检查名称是否引用 CTE。
2. 构造 `EntryLookupInfo`。
3. 通过 `BindTableName` 解析 catalog/schema/table 名称。
4. 通过 `entry_retriever.GetEntry(...)` 查找表或视图。
5. 对真实表生成 table index、扫描信息和 `LogicalGet`。
6. 通过 `bind_context.AddBaseTable(...)` 加入列名与类型绑定。

然后读 `ExpressionBinder::BindExpression(ColumnRefExpression &...)`：

1. `QualifyColumnName` 尝试把列名限定到某个 binding。
2. 找不到时尝试别名、SQL value function 和外层作用域。
3. 成功限定后由 `bind_context.BindColumn` 生成绑定表达式。
4. 最终记录绑定列信息，后续计划使用稳定的列引用。

把 `ColumnBinding` 理解为计划内部对列位置的标识。SQL 名称用于用户书写和绑定；进入计划后，仅靠反复查字符串既低效又容易在投影、连接后产生歧义。

## 第 4 次学习：成功对照、测试与失败矩阵（1.5 小时）

先用 `USING` 做一个成功对照：

```bash
build/reldebug/duckdb :memory: -csv -c "
CREATE TABLE left_t(id INTEGER, a INTEGER);
CREATE TABLE right_t(id INTEGER, b INTEGER);
INSERT INTO left_t VALUES (1, 10);
INSERT INTO right_t VALUES (1, 20);
SELECT id FROM left_t JOIN right_t USING(id);
"
```

可核对结果：

```text
id
1
```

解释为什么前面的 `JOIN ... ON` 产生两个可见的 `id` 候选，而 `USING(id)` 的输出可直接引用合并后的连接键。不要把这条行为推广到所有限定名规则，结论只覆盖这个实验。

运行与别名绑定相关的现有测试：

```bash
build/reldebug/test/unittest test/sql/filter/test_alias_filter.test
```

确认输出包含实际执行的测试数和 `All tests passed`。最后完成失败矩阵，并为每一行写出源码证据。

## 源码阅读顺序

| 顺序 | 文件与符号 | 关注产物 | 必须回答的问题 |
| ---: | --- | --- | --- |
| 1 | [select.gram](../src/parser/peg/grammar/statements/select.gram) `SimpleSelect`、`SelectClause`、`FromClause` | PEG 匹配结构 | WHERE、GROUP BY、ORDER BY 分别挂在哪层？ |
| 2 | [transform_select.cpp](../src/parser/peg/transformer/transform_select.cpp) `TransformSelectClause` | `SelectNode::select_list` | SELECT 项怎样进入语法对象？ |
| 3 | 同文件 `TransformSimpleSelect` | WHERE、GROUP、HAVING 等字段 | transformer 是否会检查列真实存在？ |
| 4 | [bind_select_node.cpp](../src/planner/binder/query_node/bind_select_node.cpp) `BindNode`、`BindSelectNode` | 绑定后的 SELECT | 为什么先绑定 FROM？ |
| 5 | [bind_basetableref.cpp](../src/planner/binder/tableref/bind_basetableref.cpp) `Binder::Bind(BaseTableRef &)` | 表 entry、binding、`LogicalGet` | CTE、表和替换扫描怎样分流？ |
| 6 | [bind_columnref_expression.cpp](../src/planner/binder/expression/bind_columnref_expression.cpp) `BindExpression(ColumnRefExpression...)` | `BoundColumnRefExpression` | 未限定列名怎样找到唯一来源？ |
| 7 | [expression_binder.cpp](../src/planner/expression_binder.cpp) 通用 `BindExpression` 分派 | 按表达式类别分派 | 函数、常量、列引用为何走不同分支？ |
| 8 | [catalog.hpp](../src/include/duckdb/catalog/catalog.hpp) `Catalog::GetEntry` | 元数据 entry | Catalog 如何成为表/函数/类型名称的权威来源？ |
| 9 | [test_alias_filter.test](../test/sql/filter/test_alias_filter.test) | 行为契约 | SELECT 别名在过滤中的当前语义由哪些用例固定？ |

## 本周产出模板

```markdown
# Parser / Binder / Catalog 失败矩阵

| SQL 摘要 | Parser 能否完成 | 失败阶段 | 错误类别 | 原因 | 关键处理函数 |
| --- | --- | --- | --- | --- | --- |
| SELEC 42 | | | | | |
| missing_table | | | | | |
| 不存在列 b | | | | | |
| 歧义列 id | | | | | |
| DATE + DATE | | | | | |

## 成功查询追踪
- grammar 规则：
- transformer 产物：
- 表 entry：
- bind context：
- 列绑定：
- 逻辑算子：
```

## 验收 checklist

- [ ] 能沿 `SimpleSelect` 规则拆解一条主线 SELECT。
- [ ] 能指出 `TransformSelectClause` 与 `TransformSimpleSelect` 的不同职责。
- [ ] 能解释 Parser 为什么不能判断 `missing_table` 是否存在。
- [ ] 能说明 Binder 为什么先处理 FROM。
- [ ] 五个失败例子的阶段预测与实际错误一致。
- [ ] 能解释未限定 `id` 为什么可能歧义。
- [ ] 知道 Catalog entry、SQL 名称和 ColumnBinding 不是同一对象。
- [ ] `test_alias_filter.test` 实际执行且通过，或已准确记录未运行原因。

## 自测题与参考答案

1. `SELECT b FROM t` 语法正确，为什么仍会失败？
   - Parser 只确认结构合法；Binder 在 FROM 作用域中找不到列 `b`。
2. 表不存在为什么当前显示 `Catalog Error`，却仍说失败发生在绑定阶段？
   - Binder 在绑定 `BaseTableRef` 时调用 Catalog 查 entry；查找失败产生 Catalog 类别的错误。
3. 为什么 `SELECT id FROM l JOIN r ON ...` 会歧义？
   - `l` 和 `r` 都向绑定上下文提供名为 `id` 的列，未限定名称没有唯一来源。
4. `DATE + DATE` 为什么不是 Parser Error？
   - `+` 表达式语法合法，但 Binder 找不到参数类型为 `(DATE, DATE)` 的匹配函数重载。
5. `ColumnBinding` 解决什么问题？
   - 它用计划内部的位置标识列，让后续算子在名称解析完成后稳定引用列。
6. transformer 是否应该访问某个用户数据库来确认表存在？
   - 不应该；transformer 将匹配结果转成解析对象，目录解析属于 Binder/Catalog 阶段。

## 常见卡点

- 用错误前缀机械等同阶段：错误类别反映抛出位置/类别；还要看当时由哪个阶段发起查找。
- 在巨大 grammar 中顺序阅读：围绕一条具体 SELECT 搜规则名更有效。
- 把 `SelectNode` 当成绑定完成对象：其中仍包含 `ParsedExpression` 和未解析表引用。
- 忽略 FROM 先绑定：没有表作用域，就无法可靠解析 SELECT、WHERE 等位置的列。
- 看到别名可在某处使用就推广到所有子句：以测试和各专用 binder 的规则为准。
- 把 SQL 列名一直带到执行：后续计划会使用绑定后的列引用和位置。

## 可选拓展

- 将歧义查询改成 `SELECT l.id ...`，解释限定名如何使绑定唯一。
- 阅读 `BindSelectNode` 中 WHERE、GROUP BY、HAVING 与 SELECT list 的 binder 创建顺序。
- 在 [test_alias_filter.test](../test/sql/filter/test_alias_filter.test) 中选一个边界用例，写出名称查找候选集合。

## 本页示例的验证状态

已在提交 `d6b403d382` 的现有 `build/reldebug/duckdb` 上验证五类失败查询、`JOIN ... USING(id)` 成功查询，以及 `test_alias_filter.test`；测试实际执行 1 个测试用例、15 个断言并通过。PEG grammar、transformer、表绑定和列绑定符号已按当前源码核对。本文未修改 grammar、未运行 `make generate-files`、未启动调试器，也未执行全量测试。

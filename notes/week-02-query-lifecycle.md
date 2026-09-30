# 第 2 周：追踪一条 SQL 的生命周期

[上一周：环境与测试](week-01-environment-and-tests.md) · [返回总计划](learning-plan.md) · [下一周：类型与向量](week-03-types-and-vectors.md)

本周把“SQL 文本变成结果”映射到当前代码中的真实入口。目标不是背调用栈，而是分清各阶段的输入、输出、职责和边界。

## 本周目标

完成本周后，你应当能够：

- 画出解析、绑定与逻辑计划、优化、物理计划、执行、取结果的阶段图。
- 为每个阶段指出一个当前源码入口。
- 区分 `SQLStatement`、绑定后的表达式、逻辑计划与物理计划。
- 解释“阶段图”为什么不是一条逐函数、无分支的调用栈。
- 用简单查询和参数化查询核对准备与执行的边界。

## 前置条件与本周范围

先完成第 1 周，能够运行 `build/reldebug/duckdb`。本周不修改源码，不要求进入优化规则、pipeline 或算子算法内部。

所有命令从仓库根目录执行：

```bash
cd /Users/darion.yaphet/source/duckdb
```

当前实现有一个容易误读的版本细节：普通 `Connection::Query` 进入 `ClientContext::Query(const string &, ...)` 后，直接创建 `StatementIterator(ParseIterator(...))`；`ParseIterator` 再调用 `Parser::ParseTopLevelStatement`。`ParseStatementsInternal` 是供 `Prepare` 等 eager 调用方把迭代器排空为 vector 的对照入口。两条路径概念上都属于 PEG 解析阶段，但不能把 `Parser::ParseQuery` 写成每条普通查询都必经的断点。

## 第 1 次学习：先从行为建立阶段假设（1.5 小时）

运行一个全新内存会话：

```bash
build/reldebug/duckdb :memory: -csv -c "
SELECT 42 AS answer, typeof(42) AS answer_type;
PREPARE add_one AS SELECT ?::INTEGER + 1 AS result;
EXECUTE add_one(41);
"
```

可核对结果：

```text
answer,answer_type
42,INTEGER
result
42
```

执行前先写下自己的阶段假设：

1. 文本在何处切分成三条语句？
2. `?` 在解析时是否已经有具体值？
3. `PREPARE` 时哪些工作可以完成，`EXECUTE` 时还要补什么？
4. `typeof(42)` 的返回类型由哪个阶段确定？

再执行：

```bash
build/reldebug/duckdb :memory: -c "
SET explain_output = 'all';
EXPLAIN SELECT 42 AS answer;
"
```

把逻辑计划、优化后逻辑计划和物理计划分开记录。显示格式可能变化，重点是对象层次，不要逐字符复制输出。

## 第 2 次学习：解析、绑定和逻辑计划（2 小时）

按下面顺序阅读，只回答表中问题，不追所有辅助函数：

1. `Connection::Query` 如何转入 `ClientContext::Query(const string &, ...)`。
2. `ClientContext::Query` 如何直接创建并消费 `StatementIterator(ParseIterator(...))`。
3. `ParseIterator::EnsureTokenized` 为什么只对完整输入分词一次。
4. `ParseIterator::Peek` 如何逐条调用 `Parser::ParseTopLevelStatement`。
5. `ParseStatementsInternal` 如何为 `Prepare` 等 eager 调用方排空相同的迭代器。
6. `Planner::CreatePlan` 如何调用 `binder->Bind(statement)` 并保存名称、类型和逻辑计划。

用这组术语整理笔记：

- SQL 文本：用户输入的字符序列。
- `SQLStatement`：解析得到的语法级语句对象，名称尚未全部解析到目录对象。
- `BoundStatement`：包含输出名称、类型和逻辑计划。
- `LogicalOperator` 树：描述“做什么”，还不是具体执行算法和 pipeline。

此时回到第 1 次的四个问题，修改答案并标出证据文件。

## 第 3 次学习：优化、物理计划和执行（2 小时）

阅读 `ClientContext::CreatePreparedStatementInternal`，把函数分成三段：

1. `Planner::CreatePlan` 生成逻辑计划。
2. `Optimizer::Optimize` 改写逻辑计划。
3. `PhysicalPlanGenerator::Plan` 解析类型和列绑定，并创建物理计划。

继续阅读 `ClientContext::PendingPreparedStatementInternal`：它创建 `Executor`，取得结果收集器，然后调用 `executor.Initialize(...)`。执行任务由 `ClientContext::ExecuteTaskInternal` 向 executor 推进；结果准备好后由 `FetchResultInternal` 取得。

画阶段图时使用下面的主线，但在图下注明它是职责图：

```text
SQL 文本
  -> ParseIterator / PEG Parser
  -> SQLStatement
  -> Binder + Planner
  -> LogicalOperator 树
  -> Optimizer
  -> 优化后的 LogicalOperator 树
  -> PhysicalPlanGenerator
  -> PhysicalPlan
  -> Executor / Result Collector
  -> QueryResult
```

回答两个关键问题：

- 为什么 `CreatePreparedStatementInternal` 已经创建物理计划，但还没有返回查询结果？
- 为什么结果收集器也作为物理执行计划的一部分传给 `Executor::Initialize`？

## 第 4 次学习：形成可检查的生命周期图（1.5 小时）

用 `SELECT 42` 与第 1 周 orders 查询各填一次本周产出模板。简单查询用来减少噪声，orders 查询用来确认多个阶段仍适用。

可选的 LLDB 练习如下；它依赖本机调试器权限和符号，本文没有替你运行：

```bash
lldb -- build/reldebug/duckdb :memory: -c "SELECT 42"
```

在 LLDB 中可尝试：

```text
breakpoint set -n 'duckdb::ParseIterator::Peek'
breakpoint set -n 'duckdb::Planner::CreatePlan'
breakpoint set -n 'duckdb::Optimizer::Optimize'
breakpoint set -n 'duckdb::PhysicalPlanGenerator::Plan'
breakpoint set -n 'duckdb::Executor::Initialize'
run
thread backtrace
continue
```

优化构建中函数可能内联、局部变量可能不可见；断点未命中时先用 `image lookup -rn` 查符号，再以源码阅读和 `EXPLAIN` 作为主证据。不要为了命中断点修改业务源码。

## 源码阅读顺序

| 顺序 | 文件与符号 | 关注输入与输出 | 必须回答的问题 |
| ---: | --- | --- | --- |
| 1 | [connection.cpp](../src/main/connection.cpp) `Connection::Query(const string &)` | API 调用 → context 调用 | CLI 主查询怎样进入 `ClientContext`？ |
| 2 | [client_context.cpp](../src/main/client_context.cpp) `ClientContext::Query(const string &, ...)` | 字符串 → lazy statement iterator | 普通查询为何绕过 `ParseStatementsInternal`？ |
| 3 | 同文件 `ParseStatementsInternal`、`Prepare` | 字符串 → eager `SQLStatement` vector | 哪些调用方需要一次取得全部语句？ |
| 4 | [parse_iterator.cpp](../src/main/parse_iterator.cpp) `EnsureTokenized`、`Peek` | 字符串 → token → 单条语句 | 普通查询为什么不必经过 `Parser::ParseQuery`？ |
| 5 | [parser.cpp](../src/parser/parser.cpp) `NormalizeSQLString`、`ParseQuery` | PEG 与扩展回退路径 | `ParseQuery` 与迭代解析共享哪些职责？ |
| 6 | [planner.cpp](../src/planner/planner.cpp) `Planner::CreatePlan` | `SQLStatement` → `BoundStatement` 内容 | 名称、类型和逻辑计划在哪里产生？ |
| 7 | [client_context.cpp](../src/main/client_context.cpp) `CreatePreparedStatementInternal` | 逻辑计划 → 物理计划 | 优化与物理规划在准备阶段怎样串联？ |
| 8 | [optimizer.cpp](../src/optimizer/optimizer.cpp) `Optimizer::Optimize` | 逻辑计划 → 等价逻辑计划 | 优化器改变语义还是实现选择？ |
| 9 | [physical_plan_generator.cpp](../src/execution/physical_plan_generator.cpp) `Plan`、`ResolveAndPlan` | 逻辑计划 → `PhysicalPlan` | 类型与列引用为何在创建算子前解析？ |
| 10 | [client_context.cpp](../src/main/client_context.cpp) `PendingPreparedStatementInternal`、`ExecuteTaskInternal`、`FetchResultInternal` | 物理计划 → 任务 → 结果 | 准备、执行和取结果的边界在哪里？ |
| 11 | [executor.cpp](../src/parallel/executor.cpp) `Executor::Initialize` | 物理根节点 → 可执行状态 | 初始化与真正执行是不是同一件事？ |

## 本周产出模板

```markdown
# 查询生命周期图

查询：
提交：

| 阶段 | 输入 | 输出 | 入口符号 | 本查询中的具体对象/现象 |
| --- | --- | --- | --- | --- |
| 解析 | | | | |
| 绑定与逻辑规划 | | | | |
| 优化 | | | | |
| 物理规划 | | | | |
| 执行 | | | | |
| 取结果 | | | | |

## 阶段图与实际调用栈的差异
- 迭代解析：
- 准备与执行边界：
- 结果收集：
- 调试证据（如有）：
```

## 验收 checklist

- [ ] 能用自己的话描述六个阶段，不只背函数名。
- [ ] 能指出普通查询当前使用 `ParseIterator` 和 PEG parser。
- [ ] 能区分 `SQLStatement`、绑定后的逻辑计划和物理计划。
- [ ] 能解释准备阶段为什么不等于执行阶段。
- [ ] 生命周期图的每个节点都有输入、输出和源码入口。
- [ ] 图下注明它是职责图，不冒充完整调用栈。
- [ ] `SELECT 42` 与参数化示例得到预期结果。

## 自测题与参考答案

1. Parser 能否确定 `orders.amount` 的最终类型？
   - 不能。Parser 构造语法对象；Binder 根据 Catalog 和作用域解析名称与类型。
2. 优化器的输入和输出为什么都可以是逻辑计划？
   - 它执行语义等价改写，仍描述逻辑操作，还未选择完整物理执行实现。
3. 物理计划已生成，为什么还需要 Executor？
   - 物理计划描述算子；Executor 建立执行状态、pipeline 和任务并推进计算。
4. `PREPARE add_one` 时是否已经知道参数值是 41？
   - 不知道。具体值由 `EXECUTE add_one(41)` 提供；显式 `::INTEGER` 让参数类型可绑定。
5. 为什么不能把 `Parser::ParseQuery` 当作普通 CLI 查询的必经断点？
   - 当前普通 `ClientContext::Query` 直接使用 `StatementIterator(ParseIterator(...))`，其 `Peek` 逐条调用 `ParseTopLevelStatement`；它绕过 `ParseStatementsInternal`。

## 常见卡点

- 一路追进模板和异常处理后迷路：回到“该函数输入和输出是什么”，超出问题就停止。
- 把 Planner 等同于 Optimizer：Planner 绑定并生成逻辑计划，Optimizer 对逻辑计划做等价改写。
- 认为 `EXPLAIN` 展示 C++ 调用栈：它展示计划，不展示运行时函数栈。
- 断点未命中就判断源码没走：优化、内联、符号名或当前迭代解析路径都可能影响断点。
- 看到 `PreparedStatement` 就认为只与显式 `PREPARE` 有关：普通查询也会走准备可执行计划的内部阶段。

## 可选拓展

- 阅读 `ParseIterator::HasMore`，解释它如何在不推进正式 token 游标的情况下探测后续语句。
- 比较 `EXPLAIN SELECT 42` 与 orders 查询的计划规模，列出共同阶段和新增算子。
- 在 LLDB 中仅对 `Planner::CreatePlan` 保存一次回溯，与职责图逐项比较。

## 本页示例的验证状态

已在提交 `d6b403d382` 的现有 `build/reldebug/duckdb` 上验证 `SELECT 42`、`typeof(42)`、`PREPARE add_one` 和 `EXECUTE add_one(41)`。源码符号已按当前实现核对。本文未启动 LLDB、未验证断点命中情况，也未重新构建项目；调试练习由学习者在本机执行并记录实际证据。

# 第 6 周：物理算子与 DataChunk 数据流

导航：[总计划](learning-plan.md) · [上一周：优化器与计划变化](week-05-optimizer.md) · [下一周：聚合与连接](week-07-aggregation-and-joins.md)

本周用 7 小时把逻辑计划连接到执行阶段。重点不是背诵算子目录，而是理解 Source、普通 Operator、Sink 三种职责，以及一个 `DataChunk` 怎样被取出、过滤和继续传递。

## 本周目标

完成后你应该能够：

- 从 `LogicalFilter` 找到 `PhysicalFilter` 的创建位置。
- 区分 Source 的 `GetData`、普通 Operator 的 `Execute`、Sink 的 `Sink`。
- 解释 `PhysicalFilter::ExecuteInternal` 的全通过、部分通过和零行三条路径。
- 说明 `Reference` 与 `Slice` 怎样避免逐行复制。
- 读懂 `OperatorResultType::NEED_MORE_INPUT` 的含义，不把它误解为“没有输出”。

## 前置与范围

前置：完成第 3 周的 Vector/DataChunk 和第 5 周的过滤下推。你应知道选择向量描述“逻辑行到物理位置”的映射。

本周范围：

- 必读：物理算子公共接口、Filter、Table Scan、逻辑 Filter 到物理 Filter 的映射。
- 只建立概念：pipeline 如何驱动 source/operator/sink；第 8 周再深入调度。
- 不展开：异步 I/O、缓存算子完整状态机、所有 `OperatorResultType` 边界情况。

从仓库根目录开始：

```bash
cd /Users/darion.yaphet/source/duckdb
```

## 第 1 次：确认真实计划和三类职责（1.5 小时）

1. 运行“实验 A”，确认计划中确实存在独立 `Filter`。实验同时禁用过滤下推和统计传播，避免全真/全假谓词被统计规则提前消除。
2. 从计划底部向上标出：Table Scan 是 Source，Filter 是普通 Operator，查询结果收集端是 Sink。
3. 阅读 `operator_result_type.hpp` 中 Source、Operator、Sink 的返回枚举说明。
4. 写下每类接口的输入、输出和状态对象。
5. 注意：同一个物理算子可以在不同阶段承担多种职责，例如聚合常同时有 Sink 和 Source 能力；三类是接口职责，不是互斥类层次。

## 第 2 次：精读 PhysicalFilter（2 小时）

按源码表前六行阅读，并画出以下路径：

```text
LogicalFilter
  -> PhysicalPlanGenerator::CreatePlan(LogicalFilter &)
  -> PhysicalFilter
  -> FilterState(ExpressionExecutor + SelectionVector)
  -> PhysicalFilter::ExecuteInternal
```

对 `ExecuteInternal` 做三列表：

| `result_count` | 输出动作 | 含义 |
| --- | --- | --- |
| 等于输入行数 | `chunk.Reference(input)` | 全通过，引用整个输入 |
| 大于 0 且小于输入 | `chunk.Slice(input, sel, result_count)` | 部分通过，用选择向量切片 |
| 等于 0 | 不写输出 chunk | 当前输入没有通过行 |

三条路径最后都返回 `NEED_MORE_INPUT`，表示当前输入已处理完，可以接收下一块；它不说明当前调用是否产生了非空输出。

## 第 3 次：用三种选择率核对数据流（2 小时）

1. 执行“实验 B”的全通过、部分通过、零行查询。
2. 先记录固定结果，再分别运行 `EXPLAIN ANALYZE` 观察实际行数。
3. 把输入 chunk、SelectionVector 和输出 chunk 画成三张小图。
4. 若使用 LLDB，在 `duckdb::PhysicalFilter::ExecuteInternal` 设置断点，观察 `input.size()`、`result_count` 和 `chunk.size()`。
5. LLDB 是可选项；若没有执行，笔记明确写“未做断点验证”，不要把源码推断写成现场观察。

## 第 4 次：串起扫描、过滤与下游（1.5 小时）

1. 阅读 `PhysicalTableScan::GetDataInternal`，只追“表函数如何填充输出 chunk”。
2. 对照 `PhysicalOperator::GetData`、`Execute`、`Sink` 的基类入口。
3. 写一段 150 字以内的说明：谁申请状态、谁循环取块、Filter 如何交出结果。
4. 完成产出模板、checklist 和自测题。
5. 恢复优化器设置。

```sql
RESET disabled_optimizers;
```

## 源码阅读顺序

| 顺序 | 文件与符号 | 阅读时回答的问题 |
| --- | --- | --- |
| 1 | [physical_operator.hpp](../src/include/duckdb/execution/physical_operator.hpp) `PhysicalOperator` | 一个算子如何声明自己是否为 source、operator、sink？ |
| 2 | [operator_result_type.hpp](../src/include/duckdb/common/enums/operator_result_type.hpp) | 三类接口的返回值分别控制什么？ |
| 3 | [plan_filter.cpp](../src/execution/physical_plan/plan_filter.cpp) `CreatePlan(LogicalFilter &)` | 何时创建 `PhysicalFilter`，何时再加 Projection？ |
| 4 | [physical_filter.hpp](../src/include/duckdb/execution/operator/filter/physical_filter.hpp) `PhysicalFilter` | Filter 保存什么表达式，是否支持并行 operator？ |
| 5 | [physical_filter.cpp](../src/execution/operator/filter/physical_filter.cpp) `FilterState` | 为什么表达式执行器和 SelectionVector 属于每个 operator state？ |
| 6 | [physical_filter.cpp](../src/execution/operator/filter/physical_filter.cpp) `ExecuteInternal` | 全通过、部分通过、零行分别怎样构造输出？ |
| 7 | [physical_table_scan.cpp](../src/execution/operator/scan/physical_table_scan.cpp) `GetDataInternal` | 扫描如何调用 table function 产生 chunk？ |
| 8 | [physical_operator.cpp](../src/execution/physical_operator.cpp) `GetData`、`Execute`、`Sink` | 基类如何分隔三种数据流接口？ |
| 9（预览） | [pipeline_executor.cpp](../src/parallel/pipeline_executor.cpp) `FetchFromSource`、`ExecutePushInternal` | 谁把 source 输出依次推过 operators？ |

阅读 `physical_operator.hpp` 时，重点看 `GlobalSourceState`、`LocalSourceState`、`GlobalOperatorState`、`OperatorState`、`GlobalSinkState` 和 `LocalSinkState` 的用途，不需要记住全部成员。

## 实验 A：强制保留独立 PhysicalFilter

过滤通常会被推入扫描，统计传播还可能根据已知列范围删除恒真 Filter 或改写为 Empty Result。为了受控观察 `PhysicalFilter` 的三个执行分支，在新的内存会话中临时禁用这两条规则：

```bash
build/reldebug/duckdb
```

```sql
CREATE TABLE filter_input(i INTEGER, payload VARCHAR);
INSERT INTO filter_input VALUES
    (0, 'zero'),
    (1, 'one'),
    (2, 'two'),
    (3, 'three');

SELECT name
FROM duckdb_optimizers()
WHERE name IN ('filter_pushdown', 'statistics_propagation')
ORDER BY name;

SET disabled_optimizers = 'filter_pushdown,statistics_propagation';
SET explain_output = 'physical_only';

EXPLAIN
SELECT i, payload
FROM filter_input
WHERE i >= 2
ORDER BY i;
```

列表查询应返回两个规则名。再检查计划：当前构建应显示独立 `FILTER`/`Filter` 位于扫描之上，而不是只在 Table Scan 的 `Filters` 字段中看到条件。计划展示格式可能因版本变化；如果没有独立 Filter，不要继续按假设设断点，应先检查设置和实际计划。

运行查询，固定结果为：

| i | payload |
| ---: | --- |
| 2 | two |
| 3 | three |

## 实验 B：全通过、部分通过、零行

继续使用实验 A 的同一会话和同一张表，保持 `filter_pushdown` 与 `statistics_propagation` 禁用。先为三个查询分别执行 `EXPLAIN`，确认它们都有独立 Filter：

```sql
-- 全通过：4 行
EXPLAIN SELECT i FROM filter_input WHERE i >= 0 ORDER BY i;
SELECT i FROM filter_input WHERE i >= 0 ORDER BY i;

-- 部分通过：2 行
EXPLAIN SELECT i FROM filter_input WHERE i >= 2 ORDER BY i;
SELECT i FROM filter_input WHERE i >= 2 ORDER BY i;

-- 零行：空结果
EXPLAIN SELECT i FROM filter_input WHERE i > 100 ORDER BY i;
SELECT i FROM filter_input WHERE i > 100 ORDER BY i;

-- 用聚合明确核对行数
SELECT
    COUNT(*) FILTER (WHERE i >= 0) AS all_pass,
    COUNT(*) FILTER (WHERE i >= 2) AS partial_pass,
    COUNT(*) FILTER (WHERE i > 100) AS no_pass
FROM filter_input;
```

最后一条查询的固定结果是 `(4, 2, 0)`。在已经用 EXPLAIN 确认三个计划均保留独立 Filter 的前提下，前三条分别对应 Filter 中的 `Reference`、`Slice` 和不写非空输出的路径；实际执行可能按一个或多个 DataChunk 处理，这取决于输入规模和执行环境，本例只用于建立最小行为模型。

如需观察实际行数：

```sql
EXPLAIN ANALYZE SELECT i FROM filter_input WHERE i >= 2 ORDER BY i;
```

`EXPLAIN ANALYZE` 会执行查询；计时和显示细节不是固定结果。

## 可选 LLDB 观察

退出 CLI 后，从仓库根目录启动：

```bash
lldb -- build/reldebug/duckdb -c "SET disabled_optimizers='filter_pushdown,statistics_propagation'; CREATE TABLE t AS SELECT * FROM range(4) r(i); SELECT * FROM t WHERE i >= 2;"
```

在 LLDB 中：

```text
breakpoint set -n 'duckdb::PhysicalFilter::ExecuteInternal'
run
frame variable input
next
frame variable result_count
```

优化构建下局部变量可能不可见。断点未命中时，先用 `EXPLAIN` 确认有独立 Filter，再检查符号名；不要通过关闭更多优化器来制造与学习目标无关的复杂计划。

## 本周产出模板

```markdown
# 第 6 周学习记录

提交：<git rev-parse --short HEAD>

## 计划与角色
- Source：
- Operators：
- Sink：

## PhysicalFilter 三条路径
| 输入行数 | 通过行数 | 动作 | 输出行数 |
| --- | --- | --- | --- |
| | | | |

## 状态对象
- 全局状态解决：
- 局部状态解决：
- FilterState 包含：

## 断点记录
- 是否执行：
- input.size()：
- result_count：
- chunk.size()：

## 未解决问题
- ...
```

## 验收 checklist

- [ ] 计划中确认存在独立 Filter，而不是把扫描内过滤当成 PhysicalFilter。
- [ ] 能从 `LogicalFilter` 找到 `PhysicalFilter` 创建代码。
- [ ] 能区分 `GetData`、`Execute`、`Sink`。
- [ ] 能解释 `Reference` 与 `Slice` 的触发条件。
- [ ] 能解释零行输出为何仍可返回 `NEED_MORE_INPUT`。
- [ ] 三种选择率的固定结果均已核对。
- [ ] 已恢复 `disabled_optimizers`，没有让两条禁用设置污染后续实验。

## 自测题与参考答案

1. Source 和普通 Operator 的核心差别是什么？  
   参考：Source 不依赖上游输入，通过 `GetData` 产生 chunk；普通 Operator 接收一个输入 chunk，通过 `Execute` 变换或扩展输出。

2. 为什么全通过时使用 `Reference`？  
   参考：输出可直接引用输入向量，避免构造选择向量和复制数据。

3. 部分通过时 `Slice` 是否复制每个值？  
   参考：其核心是让输出通过 SelectionVector 引用选中的逻辑行；具体向量表示可能共享底层数据。

4. `NEED_MORE_INPUT` 是否表示输出 chunk 为空？  
   参考：不是。它表示算子已消费当前输入；本次可能产生全量、部分或零行输出。

5. 为什么要区分 local 与 global state？  
   参考：并行任务需要独立的线程/任务局部状态，同时某些算子还需查询范围内共享的全局状态。

## 常见卡点

- Filter 断点不命中：过滤可能已推入扫描；先检查 `EXPLAIN`。
- 把计划显示名当成 C++ 类名：显示格式会变化，用 `PhysicalOperatorType` 和创建代码确认。
- 认为零行会终止整条 pipeline：零行可能只代表当前 chunk 没有匹配，source 仍可能有后续输入。
- 把 Slice 理解为物化复制：回顾第 3 周的选择向量与 Dictionary 表示。
- 一开始追 `PipelineExecutor` 全部状态机：本周只看它如何连接接口，第 8 周再深入。

## 可选扩展

- 把表扩展到 5000 行，观察一次查询是否多次命中 Filter 断点。
- 阅读 [expression_executor.cpp](../src/execution/expression_executor.cpp) 中选择表达式入口，追踪 `SelectExpression`。
- 对比启用过滤下推后的 Table Scan，记录独立 Filter 消失后职责移到了哪里。
- 查看 Projection 的 `Execute`，比较只变换列与过滤行的差别。

## 验证范围

文档中的建表、查询和结果已用本地现有 `build/reldebug/duckdb` 验证；同时禁用 `filter_pushdown` 与 `statistics_propagation` 后，当前构建对 `i >= 0`、`i >= 2`、`i > 100` 三个查询都保留独立 Filter，固定计数为 `(4, 2, 0)`。仅禁用过滤下推不足以保证恒真/恒假 Filter 留在计划中。LLDB 命令仅按当前符号名核对，未在本轮实际执行断点会话；局部变量可见性取决于构建和调试器。未运行全量测试。

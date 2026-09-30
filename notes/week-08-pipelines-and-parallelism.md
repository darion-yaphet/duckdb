# 第 8 周：Pipeline、任务调度与并行执行

[总计划](learning-plan.md) · [上一周：聚合与连接](week-07-aggregation-and-joins.md) · [下一周：存储与缓冲](week-09-storage-and-buffers.md)

本周用约 7 小时，把物理算子树转换成“哪些工作能同时做、哪些必须等待”的执行视角。所有命令从仓库根目录执行。

## 本周目标与前置条件

开始前应理解 Source、Operator、Sink，以及哈希聚合和连接需要保存中间状态的原因。完成后能画出一份 pipeline 依赖草图，解释任务与线程的区别，并完成一次条件受控的线程数比较。

本周精读一个简单聚合查询的调度路径。异步 I/O、复杂 exchange、多种外部输入模式和线程池所有实现细节列为后续专题。

重要区别：物理算子树描述计算关系，pipeline 描述可连续推进的数据流，任务是调度执行的工作单位，线程是执行任务的资源。它们不是一一对应关系。

## 四次学习安排

| 次数 | 时间 | 任务 | 当次完成标准 |
| --- | --- | --- | --- |
| 1 | 1.5 小时 | 读计划并预测聚合的依赖边界 | 画出输入积累与结果输出两个阶段 |
| 2 | 2 小时 | 精读 pipeline 构建和执行入口 | 找到状态建立、调度、执行的关键符号 |
| 3 | 2 小时 | 运行受控线程数实验 | 所有结果一致，有预热和重复测量记录 |
| 4 | 1.5 小时 | 解释观察、补依赖图、自测 | 区分事实、推断与未验证机制 |

## 源码阅读顺序

| 顺序 | 入口 | 要回答的问题 |
| --- | --- | --- |
| 1 | [physical_operator.hpp](../src/include/duckdb/execution/physical_operator.hpp)：`BuildPipelines` | 一个物理算子如何参与构建数据流？ |
| 2 | [executor.cpp](../src/parallel/executor.cpp)：`InitializeInternal`、`ScheduleEvents` | 物理计划什么时候变成可调度的执行结构？ |
| 3 | [pipeline.hpp](../src/include/duckdb/parallel/pipeline.hpp)、[pipeline.cpp](../src/parallel/pipeline.cpp)：`Pipeline::Ready`、`Schedule`、`ScheduleParallel` | Source、算子列表、Sink 与共享状态放在哪里？ |
| 4 | [pipeline_executor.cpp](../src/parallel/pipeline_executor.cpp)：`Execute`、`ExecutePushInternal` | 单个执行实例如何循环获取、处理并输出数据块？ |
| 5 | [task_scheduler.cpp](../src/parallel/task_scheduler.cpp)：`ScheduleTask`、`ExecuteTasks` | 一个任务怎样进入调度队列并被执行？ |
| 6 | [meta_pipeline.hpp](../src/include/duckdb/parallel/meta_pipeline.hpp)、[event.hpp](../src/include/duckdb/parallel/event.hpp) | 更高层依赖与完成事件如何表示？ |

精读 2–4 的一个入口，其他文件用于确认对象边界。不要求把整份调用图画到线程池每个分支。

```bash
rg -n 'BuildPipelines|ScheduleEvents|InitializeInternal' src/parallel/executor.cpp src/execution
rg -n 'Pipeline::Ready|Pipeline::Schedule' src/parallel/pipeline.cpp
rg -n 'PipelineExecutor::Execute|ExecutePushInternal' src/parallel/pipeline_executor.cpp
```

## 实验一：从聚合计划寻找依赖边界

启动全新内存会话：

```bash
build/reldebug/duckdb :memory:
```

```sql
SET threads = 1;
CREATE TABLE events AS
SELECT i AS event_id, (i % 100)::INTEGER AS customer_id
FROM range(1000000) AS t(i);

EXPLAIN
SELECT customer_id, COUNT(*) AS n, SUM(event_id) AS total
FROM events GROUP BY customer_id;

SELECT COUNT(*) AS group_count, SUM(n) AS rows_seen, SUM(total) AS checksum
FROM (
    SELECT customer_id, COUNT(*) AS n, SUM(event_id) AS total
    FROM events GROUP BY customer_id
) AS grouped;
```

校验结果应为 `100,1000000,499999500000`。分组数由取模范围决定，总行数是一百万，总和可用等差数列公式独立算出。

先画概念草图：扫描向聚合积累状态；输入处理与必要的合并完成后，聚合才能输出最终分组结果。再沿当前算子的 `BuildPipelines` 与执行事件核对真实边界。实际实现可能有更多阶段，草图不等于完整运行时依赖图。

回答：哪部分可以让多个任务处理不同输入范围？哪些状态必须汇总？在输入尚未完成时，某个分组的最终计数是否已经确定？

## 实验二：同一进程比较 1 与 4 个线程

下面脚本只使用 Python 标准库与当前 CLI，不需要安装 Python DuckDB 包。它在**同一个 CLI 进程**建立表，分别为两种线程设置预热一次、测量五次。建表和进程启动不计入每次查询计时。

退出之前的交互 CLI，从仓库根目录执行：

```bash
python3 - <<'PY'
import subprocess

query = """
SELECT COUNT(*) AS group_count, SUM(n) AS rows_seen, SUM(total) AS checksum
FROM (
    SELECT customer_id, COUNT(*) AS n, SUM(event_id) AS total
    FROM events GROUP BY customer_id
) AS grouped;
"""
commands = [
    "SET max_execution_time=10000;",
    "SET threads=1;",
    "CREATE TABLE events AS SELECT i AS event_id, (i % 100)::INTEGER AS customer_id "
    "FROM range(1000000) AS t(i);",
]
for thread_count in (1, 4):
    commands.extend([".timer off", f"SET threads={thread_count};", query])
    commands.append(".timer on")
    commands.extend([query] * 5)
commands.append(".timer off")
result = subprocess.run(
    ["build/reldebug/duckdb", ":memory:", "-csv"],
    input="\n".join(commands) + "\n", text=True,
    capture_output=True, timeout=60,
)
print(result.stdout)
print(result.stderr)
if result.returncode != 0:
    raise SystemExit(result.returncode)
expected = "100,1000000,499999500000"
assert result.stdout.splitlines().count(expected) == 12, "查询结果或执行次数不符合预期"
PY
```

两次预热加十次计时查询，总共应出现 12 行相同校验结果。前五个查询计时属于 1 线程，后五个属于 4 线程；预热时 timer 关闭。计时格式以当前 CLI 输出为准。

把每组五次耗时排序，取中间值，不挑最快一次。此数据和分组规模可能过小，4 线程不保证更快。查询结果的汇总校验也不是对所有分组逐项相等的完整证明；它是本实验可独立计算的校验条件。

## 实验三：解释结果，而不是只抄耗时

| 需要控制或记录的因素 | 原因 |
| --- | --- |
| 同一二进制、提交、构建配置 | Debug、reldebug、release 的性能不能直接混比 |
| 同一数据、SQL 和统计信息 | 避免把输入或计划变化误记为线程收益 |
| 预热与重复次数 | 降低初始化和偶发调度噪声影响 |
| 机器负载与可用核数 | 其他任务会干扰测量 |
| 线程设置与实际工作分配 | 允许 4 个线程不等于每阶段始终有 4 个活跃线程 |

重新启动内存 CLI 并执行实验一的建表 SQL，再分别设置 1 和 4 个线程，对原分组查询运行 `EXPLAIN ANALYZE`；实验二脚本退出后不会保留表。观察当前输出提供的实际行数与算子时间；不要将算子时间简单相加当作总墙钟时间，尤其在并行执行时。

若差异很小，结论可以是“此输入下没有观察到稳定收益”。选做交换测量顺序为 `(4, 1)`，或仅增加行数后重新完成两组测量。一次只改变一个因素，不运行巨大笛卡尔积。

## 本周产出模板

```markdown
# 并行执行实验
- 提交、构建、机器与负载：
- SQL、行数、分组数：
- 物理算子树：
- pipeline 依赖草图及源码依据：
- 局部状态与全局合并：
- 1线程：五次耗时、中位数
- 4线程：五次耗时、中位数
- 12次查询的校验结果：
- 已观察的现象：
- 解释与尚未证实的假设：
```

## 验收清单

- [ ] 能区分算子、pipeline、任务、线程，而非画成一一对应关系。
- [ ] 从源码找到 pipeline 构建、调度和执行三个入口。
- [ ] 依赖图明确标出至少一个必须等待的阶段。
- [ ] 同一进程内完成两组预热与测量，校验结果全部一致。
- [ ] 使用中位数总结，没有只挑最快一次，也没有保证固定加速比。
- [ ] 明确标注图中哪些来自源码、哪些需要运行时调试进一步确认。

## 自测与参考答案

1. **一个物理算子是否等于一个线程？** 不是；多个任务可以执行同类算子，线程也会执行不同任务。
2. **聚合为什么可能打断连续的数据流？** 最终结果依赖输入积累和必要的状态汇总，下游不能把尚未完成的状态当作最终结果。
3. **每个线程是否应随意修改共享哈希表？** 不能如此推断；要查局部状态、共享状态与同步或合并设计。
4. **设置 threads=4 是否证明四个核全程忙碌？** 不证明；任务数量、输入大小、依赖和执行路径都有限制。
5. **单次测量快一倍是否足够？** 不够；需要条件一致、重复结果、正确性校验与适当的统计汇总。

## 常见卡点、拓展与验证范围

Python 超时或查询被中断时，先确认机器负载与已有构建，降低数据量后完整重跑；修改数据量时同步更新独立计算的期望值。CLI 计时输出很小或接近零时，不据此得出无限加速比。

可选调试在 `Pipeline::Schedule` 或 `PipelineExecutor::Execute` 处观察调用栈和状态。多线程断点会改变时序，调试过程中测得的耗时不用于性能结论。

本文的 SQL、Python 脚本和固定校验结果已用现有 `reldebug` 验证。未重新构建，未执行多线程断点会话，也未完成系统性性能基准；真实 pipeline 运行时图和本机性能分析由学习者补充。

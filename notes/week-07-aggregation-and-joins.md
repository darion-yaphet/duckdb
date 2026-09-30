# 第 7 周：哈希聚合与哈希连接

导航：[总计划](learning-plan.md) · [上一周：物理算子与数据流](week-06-physical-operators.md) · [下一周：Pipeline、任务调度与并行](week-08-pipelines-and-parallelism.md)

本周用 7 小时学习两个分析查询核心算子：聚合把多行压缩成分组状态，连接把两侧满足条件的行组合起来。重点是状态生命周期、build/probe 两阶段和 NULL/重复键边界。

## 本周目标

完成后你应该能够：

- 区分无分组聚合、Hash Group By 和 Perfect Hash Group By。
- 用 `Sink -> Combine -> Finalize -> GetData` 描述哈希聚合状态生命周期。
- 用 build/probe 描述哈希连接，并从实际计划确认哪一侧承担哪种角色。
- 正确预测 `COUNT(*)`、`COUNT(column)`、`SUM` 在 NULL 和空输入下的结果。
- 用乘法关系解释重复连接键为什么扩大结果行数。

## 前置与范围

前置：完成第 6 周，理解 Source、Operator、Sink 以及 local/global state。

本周范围：

- 必读：`PhysicalHashAggregate` 与 `PhysicalHashJoin` 的主要接口。
- 实验：普通 Hash Group By、等值 Hash Join、INNER/LEFT、重复键、NULL、空输入。
- 只了解名称：Perfect Hash 聚合、外部哈希连接、radix partition。
- 不展开：哈希表内存布局、溢写算法、复杂连接谓词和所有 join type。

所有命令从仓库根目录执行：

```bash
cd /Users/darion.yaphet/source/duckdb
```

## 第 1 次：先用 SQL 固定边界语义（1.5 小时）

1. 执行“实验 A”，在看结果前先手算每个分组。
2. 回答 `COUNT(*)` 与 `COUNT(amount)` 为什么不同。
3. 对空输入分别执行有 GROUP BY 和无 GROUP BY 的聚合。
4. 使用 `EXPLAIN` 确认实验 A 当前选择 `Hash Group By`，再进入对应源码。
5. 若你的版本选择其他算子，记录实际名称并转读相应实现；不要用 SQL 名称推断物理算法。

## 第 2 次：追踪哈希聚合状态（2 小时）

按源码表第 1–5 行阅读，围绕一组 `grp = 1` 的两行输入画状态变化：

```text
输入 chunk
  -> Sink：定位/创建 group，更新 COUNT 与 SUM 中间状态
  -> Combine：合并不同任务的局部状态
  -> Finalize：完成哈希表与聚合终态
  -> GetDataInternal：按 chunk 输出分组结果
```

阅读时只回答：分组键在哪里、聚合参数在哪里、局部状态何时合并、结果何时可读。不要追进所有模板和哈希表内部实现。

## 第 3 次：连接的 build/probe 与边界输入（2 小时）

1. 执行“实验 B”，先预测 INNER 和 LEFT 的行数。
2. 为键 10 画笛卡尔配对：左侧 2 行 × 右侧 2 行 = 4 行。
3. 用 `EXPLAIN` 确认当前使用 Hash Join，记录优化器是否交换输入。
4. 阅读 `PhysicalHashJoin::Sink`：它对实际 build-side chunk 计算 key 并构建哈希表。
5. 再定位 probe 路径；理解 probe 可能一次产生多批输出，因此不能假设一块输入只对应一块输出。

“右表是 build side”不是 SQL 语义保证。优化器可能交换两侧；以物理计划及 pipeline 构建结果为准。

## 第 4 次：整理两张状态图并自测（1.5 小时）

1. 画“哈希聚合状态生命周期图”。
2. 画“哈希连接 build 完成后，probe 才能开始”的两阶段图。
3. 在图上标出 local state、global state 和 pipeline 边界。
4. 填写产出模板，完成 checklist 和自测题。
5. 运行指定聚合测试文件，记录通过、失败或未执行，不将“文件存在”写成“测试通过”。

```bash
build/reldebug/test/unittest test/sql/aggregate/group/test_group_by.test
```

## 源码阅读顺序

| 顺序 | 文件与符号 | 阅读时回答的问题 |
| --- | --- | --- |
| 1 | [plan_aggregate.cpp](../src/execution/physical_plan/plan_aggregate.cpp) `CreatePlan(LogicalAggregate &)` | 何时选择无分组、Perfect Hash 或普通 Hash 聚合？ |
| 2 | [physical_hash_aggregate.hpp](../src/include/duckdb/execution/operator/aggregate/physical_hash_aggregate.hpp) `PhysicalHashAggregate` | 该算子暴露哪些 sink/source 接口？ |
| 3 | [physical_hash_aggregate.cpp](../src/execution/operator/aggregate/physical_hash_aggregate.cpp) `Sink` | 聚合参数怎样引用输入列并进入分组表？ |
| 4 | 同文件 `Combine`、`Finalize` | 线程局部表何时汇总，何时准备输出？ |
| 5 | 同文件 `GetDataInternal` | 完成的分组状态怎样变为结果 chunk？ |
| 6 | [plan_comparison_join.cpp](../src/execution/physical_plan/plan_comparison_join.cpp) `CreatePlan(LogicalComparisonJoin &)` | 哪些条件使计划选择 Hash Join？ |
| 7 | [physical_hash_join.hpp](../src/include/duckdb/execution/operator/join/physical_hash_join.hpp) `PhysicalHashJoin` | 连接条件、join type 和 payload 怎样保存在算子中？ |
| 8 | [physical_hash_join.cpp](../src/execution/operator/join/physical_hash_join.cpp) `Sink`、`Combine` | build 侧 key 和 payload 如何进入局部/全局哈希表？ |
| 9 | 同文件 `Finalize`、`GetDataInternal` | build 完成后怎样准备 probe，怎样输出未匹配行？ |

按需查阅 [join_hashtable.hpp](../src/include/duckdb/execution/join_hashtable.hpp) 的公开接口。遇到 tuple layout、radix partition 或 external join 分支时先停下，只记录它们解决的问题。

## 实验 A：聚合、NULL 与空输入

新开内存会话，按顺序执行：

```bash
build/reldebug/duckdb
```

```sql
CREATE TABLE agg_input(grp BIGINT, amount INTEGER);
INSERT INTO agg_input VALUES
    (1, 10),
    (1, NULL),
    (1000000000000, 20),
    (1000000000000, 30),
    (NULL, 40);

SET explain_output = 'physical_only';

EXPLAIN
SELECT grp, COUNT(*) AS rows, COUNT(amount) AS non_null, SUM(amount) AS total
FROM agg_input
GROUP BY grp;

SELECT grp, COUNT(*) AS rows, COUNT(amount) AS non_null, SUM(amount) AS total
FROM agg_input
GROUP BY grp
ORDER BY grp NULLS LAST;

SELECT COUNT(*) AS rows, SUM(amount) AS total
FROM agg_input
WHERE FALSE;

SELECT grp, COUNT(*) AS rows
FROM agg_input
WHERE FALSE
GROUP BY grp;
```

本地当前构建对第一条计划选择普通 `Hash Group By`。使用跨度很大的 BIGINT 键是为了降低 Perfect Hash 被选择的可能，但选择仍属于版本和统计信息相关行为，执行前必须看实际计划。

固定结果：

| grp | rows | non_null | total |
| ---: | ---: | ---: | ---: |
| 1 | 2 | 1 | 10 |
| 1000000000000 | 2 | 2 | 50 |
| NULL | 1 | 1 | 40 |

无 GROUP BY 的空输入仍返回一行：`rows = 0, total = NULL`。带 GROUP BY 的空输入返回零行。

## 实验 B：重复键、NULL 键与无匹配行

使用新的内存会话，完整按顺序执行：

```sql
CREATE TABLE left_rows(id INTEGER, k INTEGER, amount INTEGER);
INSERT INTO left_rows VALUES
    (1, 10, 120),
    (2, 10,  50),
    (3, 20,  80),
    (4, NULL, 100),
    (5, 30, NULL);

CREATE TABLE right_rows(k INTEGER, label VARCHAR);
INSERT INTO right_rows VALUES
    (10, 'A'),
    (10, 'B'),
    (20, 'C'),
    (NULL, 'N'),
    (40, 'D');

EXPLAIN
SELECT l.id, r.label
FROM left_rows l
INNER JOIN right_rows r ON l.k = r.k;

SELECT l.id, l.k, r.label
FROM left_rows l
INNER JOIN right_rows r ON l.k = r.k
ORDER BY l.id, r.label;

SELECT l.id, l.k, r.label
FROM left_rows l
LEFT JOIN right_rows r ON l.k = r.k
ORDER BY l.id, r.label;

SELECT
    COUNT(*) AS output_rows,
    COUNT(r.label) AS matched_rows
FROM left_rows l
LEFT JOIN right_rows r ON l.k = r.k;
```

INNER JOIN 固定返回 5 行：键 10 产生 4 行，键 20 产生 1 行；NULL 键不与 NULL 键相等。LEFT JOIN 固定返回 7 行，其中 `id = 4` 和 `id = 5` 各产生一行补 NULL 的结果。最后一个查询固定为 `(7, 5)`。

用下面的核对式解释重复键，而不是记住行数：

```text
某个等值键的输出数 = 左侧该键行数 × 右侧该键行数
键 10：2 × 2 = 4
```

## 本周产出模板

```markdown
# 第 7 周学习记录

提交：<git rev-parse --short HEAD>

## 实际计划
- 聚合算子：
- 连接算子：
- 是否发生输入交换：

## 聚合状态
- Sink 输入：
- Local state：
- Combine：
- Finalize：
- GetData：

## 连接状态
- Build side：
- Probe side：
- 重复键输出：
- NULL/无匹配处理：

## 固定结果核对
- 聚合：
- INNER JOIN：
- LEFT JOIN：

## 未解决问题
- ...
```

## 验收 checklist

- [ ] 先用 EXPLAIN 辨认实际聚合算法，再阅读对应实现。
- [ ] 能解释 `COUNT(*)` 与 `COUNT(column)` 的 NULL 差异。
- [ ] 能解释无分组空输入与分组空输入的结果差异。
- [ ] 能按 Sink、Combine、Finalize、GetData 描述聚合。
- [ ] 能从实际计划识别 Hash Join 及两侧输入。
- [ ] INNER/LEFT、重复键、NULL 键结果均已手算并执行核对。
- [ ] 指定 group-by 测试已记录真实运行状态。

## 自测题与参考答案

1. 聚合为什么需要中间状态？  
   参考：输入按 chunk 到达，SUM/COUNT 等必须跨 chunk 累积；并行时还要合并不同任务的局部状态。

2. `COUNT(amount)` 为什么忽略 NULL，而 `COUNT(*)` 不忽略？  
   参考：前者计数非 NULL 表达式值，后者计数输入行。

3. 为什么键 10 的连接输出是 4 行？  
   参考：等值连接输出所有配对，左侧 2 行与右侧 2 行形成 2×2 个组合。

4. 普通 `=` 连接中两个 NULL 会匹配吗？  
   参考：不会；`NULL = NULL` 的结果是 UNKNOWN，不满足 join 条件。

5. 为什么 build 完成是 pipeline 边界？  
   参考：probe 必须查询可用的哈希表，因此依赖 build 侧 Sink/Finalize 完成。

## 常见卡点

- 看到 GROUP BY 就直接读普通 Hash Aggregate：先看 EXPLAIN，可能是 Perfect Hash。
- 默认 SQL 右表就是 build side：优化器可交换输入，以实际物理计划为准。
- 把估算行数当成重复键实际输出：执行查询并用每键乘法核对。
- 把 NULL 当成普通键：GROUP BY 会把 NULL 放入一个分组，但普通等值 JOIN 不让 NULL 与 NULL 匹配。
- 深陷哈希表模板：先守住状态生命周期和固定 SQL 结果，再做内部专题。

## 可选扩展

- 将分组键改为连续小整数，观察是否选择 Perfect Hash Group By，并转读对应实现。
- 把连接条件改为不等式，观察物理连接算法是否变化。
- 增加右表键 20 的重复行，先用乘法预测新结果。
- 阅读 [test_group_by.test](../test/sql/aggregate/group/test_group_by.test) 中 NULL 和错误输入案例。

## 验证范围

两组 SQL 已用本地现有 `build/reldebug/duckdb` 验证。当前构建中，大跨度 BIGINT 分组选择 Hash Group By，等值连接选择 Hash Join；这些算子选择不作跨版本保证。固定结果已核对：聚合三组、空输入 `(0, NULL)`、INNER JOIN 5 行、LEFT JOIN `(7, 5)`。指定 `test_group_by.test` 已通过（1 个测试用例、119 个断言）；未运行完整测试集。

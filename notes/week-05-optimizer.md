# 第 5 周：优化器与计划变化

导航：[总计划](learning-plan.md) · [上一周：Parser、Binder 与 Catalog](week-04-parser-binder-catalog.md) · [下一周：物理算子与数据流](week-06-physical-operators.md)

本周用 7 小时回答一个核心问题：DuckDB 怎样在不改变查询语义的前提下，重写逻辑计划并减少后续工作量？主线只精读过滤下推；Top N 和连接相关变换用于建立边界感，不追求读完整个优化器。

## 本周目标

完成后你应该能够：

- 在 `Optimizer::Optimize` 中定位内置优化器的执行入口和大致次序。
- 解释 `LogicalFilter` 如何被收集、拆分，并进入 `LogicalGet::table_filters`。
- 用同一输入比较启用和禁用 `filter_pushdown` 时的计划与结果。
- 说明“结果相同”与“成本更低”是两项不同的论证。
- 对 `LEFT JOIN` 和 NULL 保持谨慎，知道谓词不能机械地跨连接移动。

## 前置与学习范围

开始前应完成第 4 周，能区分 Parser、Binder、逻辑计划和 Catalog。还需要会使用 `EXPLAIN`，知道计划从下向上读取。

本周范围：

- 必读：优化器总流程、过滤下推到扫描、LEFT JOIN 的限制。
- 选读：`ORDER BY ... LIMIT` 到 Top N 的重写。
- 不展开：代价模型全貌、连接顺序搜索、统计信息推导、所有优化规则。

所有命令均从仓库根目录执行：

```bash
cd /Users/darion.yaphet/source/duckdb
```

## 第 1 次：从行为建立问题清单（1.5 小时）

1. 运行 `SELECT name FROM duckdb_optimizers() ORDER BY name;`，确认当前版本公开的优化器名称。
2. 回顾主线查询，先预测过滤条件会出现于独立 `FILTER`，还是扫描节点内部。
3. 执行下面的“实验 A”，保存启用和禁用过滤下推的两份计划。
4. 对照两份结果，写下三个观察：过滤位置、扫描估算行数、最终结果。
5. 不把估算行数当作实际行数；需要实际行数时使用 `EXPLAIN ANALYZE`。

本次结束时，至少能提出这些问题：

- 优化规则由谁按顺序调用？
- `WHERE a AND b` 何时拆成两个谓词？
- 扫描接口怎样表示可下推过滤？
- 为什么同一个 SQL 在不同数据、统计信息或提交上可能出现不同计划？

## 第 2 次：精读过滤下推主路径（2 小时）

按源码阅读表的前六行依次阅读。每个函数只记录“输入对象、做出的决定、输出对象”，不要逐行抄代码。

重点追踪：

```text
Optimizer::Optimize
  -> Optimizer::RunBuiltInOptimizers
  -> RunOptimizer(FILTER_PUSHDOWN, ...)
  -> FilterPushdown::Rewrite
  -> FilterPushdown::PushdownFilter
  -> FilterPushdown::PushdownGet
```

读完后，用自己的话解释：`PushdownFilter` 为什么可以移除当前 `LogicalFilter`，以及 `PushdownGet` 在何种情况下会重新把过滤放回扫描上方。

## 第 3 次：研究语义边界（2 小时）

1. 阅读 `FilterPushdown::PushdownLeftJoin` 中对左右两侧绑定的判断。
2. 运行“实验 B”，先预测再核对三个查询的结果。
3. 对比 `WHERE r.label = 'A'` 与 `ON ... AND r.label = 'A'`：它们对无匹配左行的处理不同。
4. 解释 `FilterRemovesNull` 为什么会影响 LEFT JOIN 是否可转成 INNER JOIN。
5. 可选阅读 Top N：找到 `TopNOptimizer` 接收逻辑计划并决定是否重写的位置，再用 `EXPLAIN SELECT ... ORDER BY ... LIMIT ...` 观察当前版本计划。

注意：观察到 Top N、连接类型变化或过滤位置变化，只能说明当前查询与当前版本的选择；不要写成永久保证。

## 第 4 次：整理证据并自测（1.5 小时）

1. 画一张过滤下推前后逻辑计划图。
2. 为每条边标出对应源码函数，而不是只写“优化器”。
3. 填写本周产出模板。
4. 完成验收 checklist 和自测题。
5. 将 `disabled_optimizers` 恢复，避免影响后续会话。

```sql
RESET disabled_optimizers;
```

## 源码阅读顺序

| 顺序 | 文件与符号 | 阅读时回答的问题 |
| --- | --- | --- |
| 1 | [optimizer.cpp](../src/optimizer/optimizer.cpp) `Optimizer::Optimize` | 优化前后怎样校验计划？何时跳过优化？ |
| 2 | [optimizer.cpp](../src/optimizer/optimizer.cpp) `Optimizer::RunBuiltInOptimizers` | 表达式重写、filter pullup、filter pushdown 的相对顺序是什么？ |
| 3 | [optimizer.cpp](../src/optimizer/optimizer.cpp) `Optimizer::RunOptimizer` | 禁用规则、计时和计划校验在哪里发生？ |
| 4 | [filter_pushdown.cpp](../src/optimizer/filter_pushdown.cpp) `FilterPushdown::Rewrite` | 不同逻辑算子如何分派到不同下推分支？ |
| 5 | [pushdown_filter.cpp](../src/optimizer/pushdown/pushdown_filter.cpp) `PushdownFilter` | AND 谓词如何加入 `FilterCombiner`？静态 false 如何处理？ |
| 6 | [pushdown_get.cpp](../src/optimizer/pushdown/pushdown_get.cpp) `PushdownGet` | 什么条件允许生成 table filters？未完全下推的谓词去哪里？ |
| 7 | [pushdown_left_join.cpp](../src/optimizer/pushdown/pushdown_left_join.cpp) `PushdownLeftJoin` | 哪些谓词可进入左侧？何时可能转为 INNER JOIN？ |
| 8（选读） | [topn_optimizer.cpp](../src/optimizer/topn_optimizer.cpp) `TopNOptimizer` | ORDER BY 与 LIMIT 满足什么形状时才可能合并？ |

建议同时打开 [logical_filter.hpp](../src/include/duckdb/planner/operator/logical_filter.hpp) 与 [logical_get.hpp](../src/include/duckdb/planner/operator/logical_get.hpp)，确认优化器操作的是逻辑算子，而不是直接操作物理 `PhysicalFilter`。

## 实验 A：确认过滤下推改变计划但不改变结果

新开一个 CLI 会话，完整按顺序粘贴以下 SQL。表建在内存中，退出即消失。

```bash
build/reldebug/duckdb
```

```sql
CREATE TABLE fp(id INTEGER, amount INTEGER, status VARCHAR);
INSERT INTO fp VALUES
    (1, 10, 'paid'),
    (2, 60, 'paid'),
    (3, 80, 'pending'),
    (4, NULL, 'paid');

SET explain_output = 'optimized_only';

EXPLAIN
SELECT id, amount
FROM fp
WHERE status = 'paid' AND amount >= 50
ORDER BY id;

SELECT id, amount
FROM fp
WHERE status = 'paid' AND amount >= 50
ORDER BY id;

SET disabled_optimizers = 'filter_pushdown';

EXPLAIN
SELECT id, amount
FROM fp
WHERE status = 'paid' AND amount >= 50
ORDER BY id;

SELECT id, amount
FROM fp
WHERE status = 'paid' AND amount >= 50
ORDER BY id;

RESET disabled_optimizers;
```

本地当前构建中，启用规则时两个条件出现在 `Seq Scan` 的 `Filters` 中；禁用后出现独立 `Filter`，扫描本身没有这两个过滤。可核对的查询结果始终只有一行：`id = 2, amount = 60`。

如果你的计划不同，先运行以下查询确认规则名存在，再记录提交号和完整计划：

```sql
SELECT name FROM duckdb_optimizers() WHERE name = 'filter_pushdown';
```

计划形状和估算基数不作为固定结果；最终数据结果才是固定断言。

## 实验 B：LEFT JOIN、NULL 与谓词位置

使用新的内存会话，按顺序执行：

```sql
CREATE TABLE l(id INTEGER, k INTEGER);
INSERT INTO l VALUES (1, 10), (2, 20), (3, NULL), (4, 40);

CREATE TABLE r(k INTEGER, label VARCHAR);
INSERT INTO r VALUES (10, 'A'), (20, 'B'), (NULL, 'N');

SELECT l.id, r.label
FROM l LEFT JOIN r ON l.k = r.k
ORDER BY l.id;

SELECT l.id, r.label
FROM l LEFT JOIN r ON l.k = r.k
WHERE r.label = 'A'
ORDER BY l.id;

SELECT l.id, r.label
FROM l LEFT JOIN r ON l.k = r.k AND r.label = 'A'
ORDER BY l.id;
```

可核对结果：第一个查询 4 行；第二个查询只有 `(1, 'A')`；第三个查询仍有 4 行，未命中的 `label` 为 NULL。`NULL = NULL` 不为 true，所以 `id = 3` 不会匹配右表的 NULL 键。

为后两个查询执行 `EXPLAIN`，记录当前计划是否改变连接类型或过滤位置，但不要把该选择当作语义本身。语义由结果定义，计划只是实现。

## 本周产出模板

```markdown
# 第 5 周学习记录

提交：<git rev-parse --short HEAD>

## 优化前后计划
- SQL：
- 启用 filter_pushdown：
- 禁用 filter_pushdown：
- 固定结果：
- 不稳定信息（估算/算子选择）：

## 调用关系
- 入口：
- 谓词收集：
- 扫描下推：
- 无法下推时：

## 语义边界
- LEFT JOIN 例子：
- NULL 影响：
- 不能直接下推的原因：

## 未解决问题
- ...
```

## 验收 checklist

- [ ] 能从 `Optimizer::Optimize` 找到 FILTER_PUSHDOWN 的调用位置。
- [ ] 能解释 `PushdownFilter` 与 `PushdownGet` 的职责差异。
- [ ] 保存了启用/禁用规则的两份计划。
- [ ] 验证两种设置下最终结果相同。
- [ ] 能解释为什么 LEFT JOIN 右侧谓词的位置会改变结果。
- [ ] 已执行 `RESET disabled_optimizers`。
- [ ] 笔记区分固定结果、计划观察和性能推测。

## 自测题与参考答案

1. 为什么“计划变短”不能单独证明优化正确？  
   参考：正确性要求所有相关输入上的语义等价；计划长度只描述形状，不能证明结果相同，也不能证明成本更低。

2. `PushdownFilter` 删除逻辑 Filter 后，条件一定会进入扫描吗？  
   参考：不一定。条件可能继续穿过其他算子、成为连接条件，或在无法安全下推时由 `PushFinalFilters` 留在更高位置。

3. 为什么 `WHERE r.x = 1` 通常不能原样下推到 LEFT JOIN 的右侧并保留外连接？  
   参考：WHERE 会过滤连接后补出的 NULL 行；只过滤右侧输入仍会保留无匹配左行，两者语义不同。

4. `EXPLAIN` 中的 `~2 rows` 是什么？  
   参考：优化阶段的基数估算，不是查询实际处理或返回的确定行数。

5. 禁用单个优化器有什么学习价值？  
   参考：提供受控对照，帮助把计划差异关联到某条规则；它仍可能受其他优化器影响。

## 常见卡点

- 计划没有变化：确认规则名、执行顺序和会话设置；简化 SQL，并检查过滤是否本来就无法下推。
- 把逻辑 Filter 与物理 Filter 混淆：本周关注 `LogicalFilter`；物理算子留到第 6 周。
- 只比较耗时：小表耗时噪声很大，本周先证明语义与计划变化，不做性能结论。
- 忘记 NULL：比较表达式遇到 NULL 通常不是 true，外连接的补 NULL 行尤其关键。
- 试图读完 `optimizer.cpp`：只追 FILTER_PUSHDOWN 相关路径，其余规则记录名称即可。

## 可选扩展

- 比较 `ORDER BY amount DESC LIMIT 2` 与不带 LIMIT 的计划，定位 Top N 重写。
- 阅读 [test/sql/table_function/duckdb_optimizers.test](../test/sql/table_function/duckdb_optimizers.test)，了解 optimizer 列表的回归测试。
- 为同一查询加入 `EXPLAIN ANALYZE`，区分估算行数和实际行数。
- 选择一个带表达式的谓词，观察 expression rewriter 是否先将其简化。

## 验证范围

本周文档中的 SQL 已用本地现有 `build/reldebug/duckdb` 验证。过滤下推对照确认：当前构建启用时过滤位于扫描内，禁用时存在独立 Filter；固定结果为 `(2, 60)`。LEFT JOIN 结果按 SQL 语义给出。没有运行全量优化器测试，也没有对计划形状、估算基数或性能做跨版本保证。

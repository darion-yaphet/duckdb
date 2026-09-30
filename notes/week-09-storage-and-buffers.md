# 第 9 周：列式存储、行组与缓冲管理

[总计划](learning-plan.md) · [上一周：Pipeline 与并行](week-08-pipelines-and-parallelism.md) · [下一周：事务与恢复](week-10-transactions-and-recovery.md)

本周用约 7 小时，把执行器中的一次表扫描追到存储层，并用独立练习数据库观察数据持久化和列段信息。所有 shell 命令从仓库根目录执行。

## 本周目标与前置条件

完成后应能解释：扫描需要哪些列；表、行组、列数据和存储块如何关联；缓冲管理为什么需要 pin/unpin；数据正常重启后仍存在的证据是什么。

开始前应能读懂一个物理扫描计划，知道 DataChunk 与 Vector 的关系，并能运行 `build/reldebug/duckdb`。如果第 8 周尚未掌握并行调度，本周先固定单线程，仍可学习扫描与存储主线。

本周只读扫描、块生命周期和普通持久化。具体压缩编码、外部排序和故障恢复算法列为选学，不要求读完整个 `src/storage/`。

## 四次学习安排

| 次数 | 时间 | 任务 | 当次完成标准 |
| --- | --- | --- | --- |
| 1 | 1.5 小时 | 建立专用数据库，记录 SQL 结果和存储信息 | 能重开数据库并复现三个校验值 |
| 2 | 2 小时 | 沿 TableScan → DataTable → RowGroup 阅读 | 标出初始化、状态推进、数据返回的位置 |
| 3 | 2 小时 | 比较列裁剪，阅读 ColumnData 与缓冲接口 | 能区分 SQL 列、执行 Vector 和存储块 |
| 4 | 1.5 小时 | 重跑实验、整理图表、完成自测 | 每个关键结论有源码或运行证据 |

第一次先记录现象，第二、三次再解释。看不懂的压缩分支只记录“输入、输出和调用者”，不要为此停下整周进度。

## 源码阅读顺序

| 顺序 | 文件与符号 | 带着什么问题读 |
| --- | --- | --- |
| 1 | [table_scan.cpp](../src/function/table/table_scan.cpp)：`TableScanFunction::GetFunction`、`TableScanFunc` | 扫描回调在哪里注册？输入状态与输出 DataChunk 如何传递？ |
| 2 | [data_table.cpp](../src/storage/data_table.cpp)：`DataTable::InitializeScan`、`DataTable::Scan` | 列选择和过滤条件传到哪里？如何区分持久数据和事务局部数据？ |
| 3 | [row_group.cpp](../src/storage/table/row_group.cpp)：`RowGroup::InitializeScan`、`RowGroup::Scan` | 一次扫描如何前进？哪些条件可以跳过数据？ |
| 4 | [column_data.cpp](../src/storage/table/column_data.cpp)：`ColumnData::Scan`、`ScanVector` | 一列怎样解码或引用到结果向量？ |
| 5 | [standard_buffer_manager.cpp](../src/storage/standard_buffer_manager.cpp)：`Allocate`、`Pin`、`Unpin` | 谁保证正在使用的块不会被驱逐？谁释放使用权？ |
| 6 | [pragma_storage_info.cpp](../src/function/table/system/pragma_storage_info.cpp) | SQL 中看到的存储信息来自哪层对象？ |

精读前 3 项的一条扫描路径，后 3 项按问题查阅。先看参数和返回值，再看主干分支；暂不展开所有扫描变体。

```bash
rg -n 'DataTable::InitializeScan|DataTable::Scan' src/storage/data_table.cpp
rg -n 'RowGroup::InitializeScan|RowGroup::Scan' src/storage/table/row_group.cpp
rg -n 'StandardBufferManager::Pin|StandardBufferManager::Unpin' src/storage/standard_buffer_manager.cpp
```

## 实验一：创建并重新打开专用数据库

在同一个 shell 中执行以下命令。变量保存本次实验的唯一目录；重复整个初始化步骤会创建新目录，无需覆盖已有数据库。

```bash
study_storage_dir=$(mktemp -d "${TMPDIR:-/tmp}/duckdb-week09.XXXXXX")
build/reldebug/duckdb "$study_storage_dir/storage.duckdb" -csv <<'SQL'
SET threads = 1;
CREATE TABLE orders AS
SELECT i AS order_id,
       (i % 100)::INTEGER AS customer_id,
       ((i % 10) * 10)::INTEGER AS amount,
       CASE WHEN i % 2 = 0 THEN 'paid' ELSE 'pending' END AS status
FROM range(10000) AS t(i);

SELECT COUNT(*) AS n, SUM(order_id) AS id_sum, SUM(amount) AS amount_sum
FROM orders;
CHECKPOINT;
SQL
```

三个结果应为 `10000,49995000,450000`。先用等差数列和每 10 行的金额分布手算一遍，再接受数据库输出。

上述进程正常结束后，启动新进程读取同一文件：

```bash
build/reldebug/duckdb "$study_storage_dir/storage.duckdb" -csv -c \
  "SELECT COUNT(*) AS n, SUM(order_id) AS id_sum, SUM(amount) AS amount_sum FROM orders"
```

结果仍应为 `10000,49995000,450000`。在笔记里记录目录变量的实际值、提交号和两次结果。换了 shell 后变量不会自动保留，需要从记录恢复路径。

思考：这证明了正常退出后的持久化，能否据此证明突然断电时所有事务都能恢复？答案是否定的；本实验没有注入故障，也没有检查崩溃恢复路径。

## 实验二：观察列段与压缩信息

继续使用刚才的文件：

```bash
build/reldebug/duckdb "$study_storage_dir/storage.duckdb" -csv <<'SQL'
SELECT column_name, segment_type, count, compression, persistent
FROM pragma_storage_info('orders')
ORDER BY column_name, segment_type;

SELECT COUNT(*) > 0 AS has_segments,
       bool_and(persistent) AS all_persistent
FROM pragma_storage_info('orders');
SQL
```

本数据经过 checkpoint 后，两个布尔值应为 `true,true`。第一份输出用于观察，不把具体压缩算法和段数量写成跨版本固定断言。

依次回答：

1. 同一列为什么可能出现值段与 `VALIDITY` 段？
2. `count` 表示这个段覆盖的数据量，为什么不能把所有输出行的 `count` 相加当作表行数？
3. 字符串列与整数列的压缩方法是否相同？当前观察能否推广到任意数据分布？
4. `persistent=true` 表示存储段已持久化，是否意味着每次读它都必然发生磁盘 I/O？

第四题要考虑缓冲和操作系统缓存。元数据与实际 I/O 不是同一类证据。

## 实验三：查询需要的列与实际扫描

进入练习文件的交互会话：

```bash
build/reldebug/duckdb "$study_storage_dir/storage.duckdb"
```

```sql
SET threads = 1;
EXPLAIN SELECT amount FROM orders WHERE order_id BETWEEN 10 AND 14;
EXPLAIN SELECT order_id, customer_id, amount, status
FROM orders WHERE order_id BETWEEN 10 AND 14;

SELECT amount FROM orders
WHERE order_id BETWEEN 10 AND 14 ORDER BY order_id;
```

最后一个结果依次为 `0,10,20,30,40`。比较两个计划的投影列与过滤条件：第一条只输出 amount，但过滤还需要 order_id；区分“需要参与扫描的列”与“最终输出的列”。

将 `BETWEEN 10 AND 14` 换成覆盖更多数据的范围，观察计划与结果。这里只有一万行，不据此断言列裁剪带来了多少性能收益。

## 源码练习：画出一批数据的所有权

以实验三的第一条查询为例画图：表 → 行组 → 列数据 → 扫描状态 → 输出 DataChunk。再把 BlockHandle、BufferHandle 与缓冲管理器放在适当位置。

每条连线注明一种含义：拥有对象、临时访问、传递扫描状态或返回数据。不要把这些关系都画成没有说明的箭头。

阅读 `Pin` 与 `Unpin` 时记录：已有驻留缓冲和需要加载缓冲的路径有什么不同；引用句柄的生命周期在哪里结束；是否能从静态源码直接得知当前实验发生了多少驱逐。

可选调试：在 `DataTable::Scan` 或 `ColumnData::Scan` 设断点，对照调用栈。优化构建可能内联部分函数；断点未命中时先检查实际扫描路径与符号，不从未命中推断函数无用。

## 本周产出模板

```markdown
# 存储扫描实验
- 版本与构建：
- 专用数据库路径：
- 初始化/重启后三个校验值：
- 观察到的列段与压缩：
- 第一条查询的输出列、过滤列、计划投影：
- 扫描路径：符号 → 符号 → 符号
- 所有权图：对象与句柄的生命周期
- 尚无证据的结论：磁盘读取次数、崩溃恢复等
```

## 验收清单

- [ ] 能在新进程中得到一致的行数、order_id 求和与金额求和。
- [ ] 能解释表、行组、列段、Vector 和 DataChunk 的层次差别。
- [ ] 找到扫描状态初始化和推进位置，各记录一个真实符号。
- [ ] 能区分输出列与过滤需要的列。
- [ ] 能解释 pin/unpin 的用途，不把 unpin 等同于立即释放或落盘。
- [ ] 没有将普通重启实验写成完整恢复验证，也没有由计划直接推断精确 I/O。

## 自测与参考答案

1. **一个 DataChunk 是否就是一个磁盘块？** 不是。前者是执行层的批量数据容器，后者是存储管理单位，边界与生命周期不必相同。
2. **只 SELECT amount，就只需要读取 amount 吗？** 不一定；过滤、连接和其他表达式可能需要额外列，再由计划确定处理方式。
3. **unpin 后块是否立即消失？** 不必然；它可能仍驻留在缓存中，后续是否驱逐由缓冲管理决定。
4. **同一表的段数和压缩方式是否恒定？** 否。数据分布、写入、checkpoint、版本和配置均可能影响。
5. **重启后总行数一致足够吗？** 只是一个检查。求和等额外不变量能发现更多问题，但仍不构成全量数据或故障恢复证明。

## 卡点、拓展与验证范围

数据库已存在且建表失败时，重新从 `mktemp` 初始化整组实验；不要删除不确定归属的文件。输出中出现未预期压缩方法时，以元数据和当前源码为准，不为了匹配示例强行改变设置。

有余力时把数据增至几十万行，比较行组、列段和扫描范围；再进入具体压缩实现。内存限制和磁盘溢写另设专题，避免与本周基础模型混在一起。

文档编写时已用现有 `reldebug` 执行建表、checkpoint、元数据查询、重开数据库校验和两种列选择查询。未重新构建，未执行调试器或故障注入；图表、断点追踪和扩展数据实验由学习者完成。

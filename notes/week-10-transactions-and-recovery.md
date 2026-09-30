# 第 10 周：事务、MVCC、WAL 与 Checkpoint

[总计划](learning-plan.md) · [上一周：存储与缓冲](week-09-storage-and-buffers.md) · [下一周：函数与扩展](week-11-functions-and-extensions.md)

本周用约 7 小时研究两个问题：并发连接为什么能看到不同的数据，以及提交的数据如何保留下来。所有命令从仓库根目录执行。

## 本周目标与前置条件

完成后能预测双连接事务的查询结果，画出提交与回滚的状态变化，并解释 MVCC、undo、WAL、checkpoint 的不同职责。

前置知识：能阅读 sqllogictest，理解连接与数据库的区别，完成过第 9 周的正常持久化实验。先回顾 `statement ok con1`、`query I con2`：名称表示同一测试数据库上的不同连接。

本周以最小事务时间线和提交路径为主，不要求理解所有并发控制细节，不把普通 `restart` 当作崩溃注入。

## 四次学习安排

| 次数 | 时间 | 任务 | 当次完成标准 |
| --- | --- | --- | --- |
| 1 | 1.5 小时 | 预测并执行双连接读写实验 | 每一次 SELECT 都有预测值与实际值 |
| 2 | 2 小时 | 阅读事务开始、提交、回滚与 undo | 标出三个生命周期入口和版本保留原因 |
| 3 | 2 小时 | 阅读 WAL/checkpoint，执行专用文件实验 | 能区分提交、checkpoint 和重启验证 |
| 4 | 1.5 小时 | 整理时间线、重跑测试、自测 | 不依赖术语也能解释现象 |

## 源码阅读顺序

| 顺序 | 入口 | 要回答的问题 |
| --- | --- | --- |
| 1 | [test_basic_transactions.test](../test/sql/transactions/test_basic_transactions.test) | 两个连接中的 DDL 为什么在不同时间可见？ |
| 2 | [duck_transaction_manager.cpp](../src/transaction/duck_transaction_manager.cpp)：`StartTransaction`、`CommitTransaction`、`RollbackTransaction` | 事务何时登记、提交、撤销？哪些状态要同步？ |
| 3 | [undo_buffer.cpp](../src/transaction/undo_buffer.cpp) | 写入过程保留什么信息，回滚与版本读取怎样使用它？ |
| 4 | [write_ahead_log.cpp](../src/storage/write_ahead_log.cpp)：`WriteAheadLog::Flush` | 日志写入与持久化动作在哪里区分？ |
| 5 | [wal_replay.cpp](../src/storage/wal_replay.cpp)：`WriteAheadLog::Replay` | 启动时日志如何被读取并恢复？ |
| 6 | [checkpoint_manager.cpp](../src/storage/checkpoint_manager.cpp)：`SingleFileCheckpointWriter::CreateCheckpoint` | checkpoint 如何组织元数据和持久化状态？ |

本周精读前 3 项与后 3 项中的关键入口。`DuckTransactionManager::Checkpoint` 是事务管理到 checkpoint 协调的另一个入口。

## 实验一：双连接读、提交与回滚

下面命令创建独立临时测试文件并从标准输入执行。无需修改项目测试目录；当前 runner 支持 `--stdin`，因此文件也能放在 notes 下。不要把任意 notes 路径直接当成默认测试发现范围中的名称。

```bash
study_txn_dir=$(mktemp -d "${TMPDIR:-/tmp}/duckdb-week10.XXXXXX")
cat > "$study_txn_dir/visibility.test" <<'TEST'
# name: week10_visibility.test
# description: Learn snapshot visibility and rollback with two connections
# group: [learning]

statement ok con1
CREATE TABLE balance(id INTEGER, amount INTEGER);

statement ok con1
INSERT INTO balance VALUES (1, 100);

statement ok con1
BEGIN;

query I con1
SELECT amount FROM balance WHERE id = 1;
----
100

statement ok con2
BEGIN;

statement ok con2
UPDATE balance SET amount = 150 WHERE id = 1;

query I con2
SELECT amount FROM balance WHERE id = 1;
----
150

query I con1
SELECT amount FROM balance WHERE id = 1;
----
100

statement ok con2
COMMIT;

query I con1
SELECT amount FROM balance WHERE id = 1;
----
100

statement ok con1
COMMIT;

query I con1
SELECT amount FROM balance WHERE id = 1;
----
150

statement ok con2
BEGIN;

statement ok con2
UPDATE balance SET amount = 999 WHERE id = 1;

query I con2
SELECT amount FROM balance WHERE id = 1;
----
999

statement ok con2
ROLLBACK;

query I con1
SELECT amount FROM balance WHERE id = 1;
----
150
TEST
build/reldebug/test/unittest --stdin < "$study_txn_dir/visibility.test"
```

先遮住期望结果，按以下时间线填表，再运行：

| 时刻 | con1 | con2 | con1 应读到 |
| --- | --- | --- | --- |
| A | BEGIN 后读 100 | 尚未更新 | 100 |
| B | 保持事务 | 更新为 150，未提交 | 100 |
| C | 保持事务 | COMMIT | 100 |
| D | COMMIT 后再次查询 | 无活动事务 | 150 |
| E | 新查询 | 更新为 999 后 ROLLBACK | 150 |

解释重点：读自己的未提交写入、其他连接不能读到该写入、旧事务继续看到旧版本、新查询看到已提交版本。不要只写“因为 ACID”，需要指出具体时刻和版本。

## 实验二：对照仓库中的目录可见性测试

```bash
build/reldebug/test/unittest test/sql/transactions/test_basic_transactions.test
```

读原测试时把“更新一行”换成“创建一张表”来预测。记录表在创建连接、另一个已有事务、另一个新事务中的可见性。

原文件末尾有一条注释与实际连接名不一致：判断执行行为时以 `statement ... con1/con2` 为准。不要把同一连接重复建表误读成已经验证了两连接写冲突。

选做：另写一个同一行的双写冲突用例。先运行 SQL 获得当前错误类别，再使用稳定的错误片段写断言；不预设完整错误文本跨版本不变。

## 实验三：提交、回滚、checkpoint 后重新打开

继续在同一 shell 使用实验一创建的临时目录；没有做实验一时，先执行其 `mktemp` 赋值。以下文件仅用于本次练习，初始化执行一次：

```bash
build/reldebug/duckdb "$study_txn_dir/durability.duckdb" -csv <<'SQL'
CREATE TABLE ledger(id INTEGER, amount INTEGER);
BEGIN;
INSERT INTO ledger VALUES (1, 100);
COMMIT;
BEGIN;
INSERT INTO ledger VALUES (2, 999);
ROLLBACK;
CHECKPOINT;
SELECT COUNT(*) AS n, SUM(amount) AS total FROM ledger;
SQL
build/reldebug/duckdb "$study_txn_dir/durability.duckdb" -csv -c \
  "SELECT COUNT(*) AS n, SUM(amount) AS total FROM ledger"
```

两次结果都应为 `1,100`。记录三条不同结论：提交的记录存在，回滚的记录不存在，正常关闭重开后结果一致。实验没有证明每种崩溃时序下的正确性，也不能仅凭磁盘上是否出现 `.wal` 文件判断提交是否成功。

阅读 `Flush`、`Replay` 和 `CreateCheckpoint`，分别圈出日志持久化、日志重放、checkpoint 写入的入口。暂不运行 `kill -9` 或篡改数据库文件；完整恢复测试需要独立故障场景与判定标准。

## 概念对照与产出模板

| 机制 | 本周需要掌握的作用 | 不应混淆为 |
| --- | --- | --- |
| MVCC | 让事务依据可见性规则读取合适版本 | 所有连接随时看到最新写入 |
| undo | 支持撤销及旧版本相关处理，具体路径查实现 | 用于启动恢复的 WAL 本身 |
| WAL | 记录恢复所需变化，并在规定时机持久化 | 每次提交都完整重写数据库文件 |
| checkpoint | 将数据库状态组织为持久化检查点 | 代替所有事务提交与日志协议 |

```markdown
# 事务可见性与持久化
- 提交号、构建与测试命令：
- 双连接时间线：操作、事务边界、预测值、实际值
- Start / Commit / Rollback 的源码入口：
- 旧版本什么时候仍然需要保留：
- WAL Flush / Replay / Checkpoint 的职责：
- 已验证：
- 尚未验证的故障时序：
```

## 验收清单与自测

- [ ] 自定义双连接测试实际执行且通过，不只是过滤到零个测试。
- [ ] 能解释 con2 提交后，con1 的旧事务为什么仍读到 100。
- [ ] 能区分数据行可见性和表等目录对象的可见性。
- [ ] 找到事务生命周期与 WAL/checkpoint 的真实源码符号。
- [ ] 重开练习文件后仍得到 `1,100`，没有把结果夸大为崩溃恢复验证。

1. **另一个连接提交后，我的查询总能立即看到吗？** 不一定；实验中已有事务继续读取其可见版本，结束事务后的新查询才看到 150。
2. **为什么保留旧版本？** 活跃事务可能仍需要读取，不能只因有了新值就立即丢弃旧值。
3. **回滚与恢复有什么不同？** 回滚撤销一个事务的未完成工作；启动恢复依据持久化状态和日志恢复数据库，两者触发时机与输入不同。
4. **checkpoint 后是否再也不需要 WAL？** 不对；后续写入与故障恢复仍需按日志和提交协议工作。
5. **两个独立 CLI 进程能直接替代 con1/con2 吗？** 本练习不能这样替代；测试框架在同一数据库实例中管理连接，避免混入进程间文件访问限制。

## 常见卡点与验证范围

`--stdin` 读取的是 sqllogictest 文本，不是普通 SQL。出现解析失败时先检查空行、`----` 和连接名；不要把整段测试直接贴进 DuckDB CLI。

文档中的双连接测试、原有基本事务测试及持久化 SQL 在当前已有 `reldebug` 上验证；未重新构建、未运行崩溃注入和完整恢复测试集。可选冲突练习由学习者设计和执行。

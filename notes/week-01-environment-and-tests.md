# 第 1 周：建立使用与测试闭环

[返回总计划](learning-plan.md) · [下一周：查询生命周期](week-02-query-lifecycle.md)

本周用 7 小时建立最重要的学习闭环：运行 SQL、预测结果、查看计划、阅读测试、执行测试、记录证据。后续各周都重复这个闭环。

## 本周目标

完成本周后，你应当能够：

- 从仓库根目录启动一个全新的内存数据库会话。
- 独立运行单个 sqllogictest 文件，并判断它是真的通过还是被跳过。
- 解释 `statement ok`、`statement error`、`query I`、`query II` 和 `----`。
- 区分 `EXPLAIN` 与 `EXPLAIN ANALYZE`。
- 保存一份包含版本、命令、预期、实际结果和结论的实验记录。

## 前置条件与本周范围

前置条件：会使用终端，能读懂基础 SQL，知道 Git 提交号用于标识代码版本。

本周只建立工具和测试反馈，不要求理解所有物理算子。不要修改源码、生成文件或真实用户数据库。所有命令都从仓库根目录执行：

```bash
cd /Users/darion.yaphet/source/duckdb
```

文中的 `:memory:` 明确要求 CLI 使用内存数据库。每次重新执行 CLI 命令都会得到一个全新的数据库；同一条 `-c` 字符串中的语句则按顺序共享同一会话。

## 第 1 次学习：确认环境与反馈入口（1.5 小时）

### 1. 记录版本

运行：

```bash
git rev-parse --short HEAD
cmake --version | head -n 1
c++ --version | head -n 1
build/reldebug/duckdb --version
```

把输出写入本周产出。版本不同不一定是错误，但后续计划和结果都要以自己的提交为准。

### 2. 做最小查询

```bash
build/reldebug/duckdb :memory: -csv -c "SELECT 42 AS answer;"
```

可核对结果：

```text
answer
42
```

### 3. 认识测试入口

先阅读 [test/README.md](../test/README.md) 的 “Typical entrypoints” 和 “Test Contract”。然后确认测试二进制存在：

```bash
test -x build/reldebug/test/unittest && echo "unittest ready"
```

如果不存在，记录“缺少 reldebug 构建”，再按总计划执行 `make reldebug`。构建耗时不计入 7 小时学习时间。

## 第 2 次学习：建立 orders 基线（2 小时）

以下数据集供后续多周复用。为了让本周实验可独立重复，这里给出完整 SQL。

```bash
build/reldebug/duckdb :memory:
```

在这个全新交互会话中按顺序执行：

```sql
SET threads = 1;

CREATE TABLE orders (
    order_id INTEGER,
    customer_id INTEGER,
    amount INTEGER,
    status VARCHAR
);

INSERT INTO orders VALUES
    (1, 10, 120, 'paid'),
    (2, 10,  50, 'paid'),
    (3, 20,  80, 'pending'),
    (4, 20, 100, 'paid'),
    (5, 30, NULL, 'paid'),
    (6, 30,  20, 'paid');

SELECT customer_id, SUM(amount) AS total, COUNT(*) AS n
FROM orders
WHERE status = 'paid' AND amount >= 50
GROUP BY customer_id
ORDER BY total DESC, customer_id
LIMIT 5;
```

可核对结果：

| customer_id | total | n |
| ---: | ---: | ---: |
| 10 | 170 | 2 |
| 20 | 100 | 1 |

逐步拆掉查询，再重新加回：

1. 只执行 `FROM`、`WHERE` 和 `SELECT`，写出保留的 `order_id`。
2. 加入 `GROUP BY` 和聚合，预测每位客户的结果。
3. 加入 `ORDER BY` 与 `LIMIT`，解释最终顺序。
4. 说明 `order_id = 5` 为什么没有进入聚合：`NULL >= 50` 不为真。

不要把这个内存会话关闭后再直接运行最后的 `SELECT`；关闭后表会消失。后续周需要该数据时，应重新执行上面的建表和插入语句。

## 第 3 次学习：读执行计划与测试格式（2 小时）

在仍包含 `orders` 的同一会话中执行：

```sql
SET explain_output = 'physical_only';

EXPLAIN
SELECT customer_id, SUM(amount) AS total, COUNT(*) AS n
FROM orders
WHERE status = 'paid' AND amount >= 50
GROUP BY customer_id
ORDER BY total DESC, customer_id
LIMIT 5;
```

当前版本可核对的结构是：计划底部有顺序扫描和两个过滤条件，中部有分组聚合，顶部有 Top N。估算行数与具体显示名称可能随版本或统计信息改变，不把完整文本当作固定答案。

再执行：

```sql
EXPLAIN ANALYZE
SELECT customer_id, SUM(amount) AS total, COUNT(*) AS n
FROM orders
WHERE status = 'paid' AND amount >= 50
GROUP BY customer_id
ORDER BY total DESC, customer_id
LIMIT 5;
```

回答：

- `EXPLAIN` 是否真正执行了被解释的查询？
- `EXPLAIN ANALYZE` 为什么会出现实际行数和耗时？
- 为什么应从计划底部向顶部读数据流？

然后阅读 [test_limit.test](../test/sql/order/test_limit.test) 前 60 行和 [test_explain.test](../test/sql/explain/test_explain.test) 前 45 行。至少标出：

- `query I` 表示一列整数结果，`query II` 表示两列结果。
- `----` 将 SQL 与期望结果分开。
- `statement ok` 只要求语句成功。
- `statement error` 后面的内容约束预期错误。
- `<REGEX>:` 允许错误或计划文本匹配正则表达式。

## 第 4 次学习：运行测试并形成记录（1.5 小时）

从仓库根目录运行：

```bash
build/reldebug/test/unittest test/sql/order/test_limit.test
build/reldebug/test/unittest test/sql/explain/test_explain.test
```

看到 `All tests passed` 且测试数量非零才算通过。若看到 skip，记录跳过原因，不能写成“测试通过”。

任选 `test_limit.test` 中一个 `query`，按下面顺序讲清楚：输入数据是什么、SQL 做什么、结果类型是什么、预期结果为何成立、失败时由谁报告差异。

## 源码与文档阅读顺序

| 顺序 | 文件或符号 | 阅读边界 | 必须回答的问题 |
| ---: | --- | --- | --- |
| 1 | [AGENTS.md](../AGENTS.md) 的 Testing、Test File Format | 只读项目约定 | 推荐构建与单测命令是什么？哪些测试不应新增？ |
| 2 | [test/README.md](../test/README.md) | 入口与 Test Contract | 测试临时目录和数据目录由谁提供？ |
| 3 | [test_limit.test](../test/sql/order/test_limit.test) | 前 60 行 | 成功、错误、单列和双列结果分别怎么写？ |
| 4 | [test_explain.test](../test/sql/explain/test_explain.test) | 前 45 行 | 哪些断言只验证成功，哪些检查计划内容？ |
| 5 | `build/reldebug/test/unittest` | 只运行，不反查测试框架实现 | 如何证明指定文件确实执行且通过？ |

## 本周产出模板

复制下面模板到自己的学习笔记中：

```markdown
# 第 1 周实验记录

- 提交：
- DuckDB 版本：
- CMake / 编译器：
- 使用的构建目录：

## orders 基线
- 执行方式：全新 `:memory:` 会话 / 其他
- 查询前预测：
- 实际结果：
- NULL 行被排除的位置：

## 计划观察
- 扫描与过滤：
- 聚合：
- 排序与 LIMIT：
- EXPLAIN 与 EXPLAIN ANALYZE 的区别：

## 测试证据
- 命令：
- 执行测试数：
- 断言数：
- 结果或跳过原因：
```

## 验收 checklist

- [ ] 能从仓库根目录启动全新内存会话。
- [ ] orders 基线查询得到 `10/170/2` 和 `20/100/1`。
- [ ] 能解释 NULL 行为什么未通过过滤。
- [ ] 能从底向上指出扫描、聚合、Top N。
- [ ] 能说清 `EXPLAIN` 与 `EXPLAIN ANALYZE` 的差别。
- [ ] 能读懂本周出现的五种 sqllogictest 标记。
- [ ] 两个指定测试实际执行且通过，或已准确记录未运行原因。

## 自测题与参考答案

1. `statement ok` 能否证明结果值正确？
   - 不能。它只证明语句成功；结果值应使用 `query` 和期望结果验证。
2. 为什么同一条 `-c` 中先建表再查询可行，而分成两次 CLI 命令通常不可行？
   - 两次 `:memory:` 启动是两个数据库；同一 `-c` 内的语句共享一个会话。
3. `EXPLAIN ANALYZE` 与普通查询的共同点是什么？
   - 都会执行查询；前者返回带实际运行信息的计划。
4. `amount` 为 NULL 的行为什么不能通过 `amount >= 50`？
   - 比较结果是 UNKNOWN，`WHERE` 只保留结果为 TRUE 的行。
5. 测试输出显示零个测试但退出码为零，能否记为通过？
   - 不能。必须确认目标测试实际被发现并执行。

## 常见卡点

- `build/reldebug/duckdb` 不存在：先确认是否在仓库根目录，再执行总计划中的 `make reldebug`。
- 关闭 CLI 后找不到 `orders`：这是内存数据库的预期行为，重新执行完整基线 SQL。
- 计划名称与本文不同：先核对提交和 `explain_output`，关注数据流结构，不机械比较整段文字。
- 测试被跳过：查看是否有 `require-env` 或扩展条件，并把 skip 与 pass 分开记录。
- 一开始就追测试框架所有 C++：本周只学习测试文件契约，框架实现留到有具体问题时再追。

## 可选拓展

- 将 `amount >= 50` 改成 `amount IS NULL`，先预测结果再查看计划。
- 将 `LIMIT 5` 改成 `LIMIT 1`，观察结果与 Top N 参数。
- 阅读 [test_limit.test](../test/sql/order/test_limit.test) 的其余边界案例，选两个错误用例解释失败原因。

## 本页示例的验证状态

已在提交 `d6b403d382` 的现有 `build/reldebug` 上验证：版本查询、`SELECT 42`、orders 完整基线、物理计划、`test_limit.test` 和 `test_explain.test`。本页没有重新执行完整构建，也没有运行全量单元测试；`make reldebug` 是留给缺少构建产物的学习者执行的恢复步骤。

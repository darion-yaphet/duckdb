# 第 11 周：函数注册、类型绑定与扩展

[总计划](learning-plan.md) · [上一周：事务与恢复](week-10-transactions-and-recovery.md) · [下一周：综合练习](week-12-capstone.md)

本周用约 7 小时，沿一个函数读通 SQL 调用到批量执行的过程。所有命令从仓库根目录执行；普通 SQL 在全新内存会话中运行。

## 本周目标与前置条件

完成后能解释函数签名、重载、绑定、执行回调和 NULL 传播，并能区分核心函数实现与扩展加载。

前置：了解 Binder、Vector 与 ExpressionExecutor，能阅读一个模板函数和一份 SQL 测试。无需完成所有存储细节，但应能说明输入向量与结果向量由谁传递。

本周准备两条路径：`json_valid` 实现短，适合追踪扩展注册；`length`/`strlen` 可在现有核心构建上直接实验。当前本地 JSON 扩展未安装，先完成字符串实验，JSON 运行部分在构建该扩展后进行。

## 四次学习安排

| 次数 | 时间 | 任务 | 当次完成标准 |
| --- | --- | --- | --- |
| 1 | 1.5 小时 | 预测字符串例子，列函数行为矩阵 | 区分字节、Unicode 码点与 NULL |
| 2 | 2 小时 | 从注册与签名读到执行回调 | 找到注册、类型选择、执行器三处源码证据 |
| 3 | 2 小时 | 跑测试并阅读 JSON 实现，按环境选择实验 | 明确通过、跳过、待构建的区别 |
| 4 | 1.5 小时 | 写一份函数规格并自测 | 能为第 12 周定义完整边界行为 |

## 源码阅读顺序

| 顺序 | 入口 | 需要回答的问题 |
| --- | --- | --- |
| 1 | [test_length.test](../test/sql/function/string/test_length.test) | 数值结果、NULL、不同字符集分别怎样测试？ |
| 2 | [length.cpp](../src/function/scalar/string/length.cpp)：`StringLengthOperator`、`StrLenOperator`、`LengthFun::GetFunctions` | 实际计算与签名构造如何分离？ |
| 3 | [scalar_function.hpp](../src/include/duckdb/function/scalar_function.hpp)：`ScalarFunction::UnaryFunction` | 回调怎样把 DataChunk 的一列交给 UnaryExecutor？ |
| 4 | [json_valid.cpp](../extension/json/json_functions/json_valid.cpp)：`GetValidFunction`、`GetValidFunctionInternal`、`ValidFunction` | 同一实现怎样提供 VARCHAR/JSON 两种重载？ |
| 5 | [json_functions.cpp](../extension/json/json_functions.cpp)：`GetValidFunction` 的注册调用 | 函数集合在哪里加入扩展？ |
| 6 | [test_json_valid.test](../test/sql/json/scalar/test_json_valid.test) | 文本有效性、特殊语法与 NULL 的预期是什么？ |

精读一个核心函数回调和整个短小的 `json_valid.cpp`。绑定细节从调用关系继续查阅，不要求一次掌握所有函数类别。

```bash
rg -n 'LengthFun::GetFunctions|StringLengthOperator|StrLenOperator' src/function/scalar/string/length.cpp
rg -n 'GetValidFunction|ValidFunction' extension/json/json_functions.cpp extension/json/json_functions/json_valid.cpp
rg -n 'UnaryFunction|UnaryExecutor::Execute' src/include/duckdb/function/scalar_function.hpp
```

## 实验一：先确定函数语义

```bash
build/reldebug/duckdb -csv
```

```sql
WITH inputs(label, s) AS (
    VALUES ('ascii', 'duck'),
           ('han', '中文'),
           ('empty', ''),
           ('null', NULL),
           ('combining', 'e' || chr(769))
)
SELECT label, length(s) AS codepoints, strlen(s) AS bytes
FROM inputs ORDER BY label;

SELECT typeof(length('duck')) AS result_type;
```

预期：

| label | codepoints | bytes |
| --- | --- | --- |
| ascii | 4 | 4 |
| combining | 2 | 3 |
| empty | 0 | 0 |
| han | 2 | 6 |
| null | NULL | NULL |

返回类型应为 `BIGINT`。`chr(769)` 是组合重音字符：它和 e 组合显示时可能看起来像一个字符，但这里有两个 Unicode 码点。不要把 `length` 简单解释成“屏幕上看见的字符数”。

回答三个问题：为何空字符串是 0 而 NULL 仍是 NULL？为何中文的两种长度不同？执行回调接收到的类型与 SQL 中函数的返回类型分别在哪里指定？

## 实验二：从常量到表输入

在同一内存会话执行：

```sql
CREATE TEMP TABLE texts(id INTEGER, s VARCHAR);
INSERT INTO texts VALUES (1, 'duck'), (2, '中文'), (3, NULL), (4, '');
SELECT id, length(s), strlen(s) FROM texts ORDER BY id;
EXPLAIN SELECT length('duck');
EXPLAIN SELECT length(s) FROM texts;
```

表查询的结果依次为 `(1,4,4)`、`(2,2,6)`、`(3,NULL,NULL)`、`(4,0,0)`。比较两个计划，判断常量调用是否被提前计算。

如果想在执行回调设断点，优先使用表输入，避免常量折叠让实际执行位置改变。SQL 写了函数名并不保证在最终执行阶段逐行调用该函数。

阅读 `LengthPropagateStats` 时只关注一个问题：为什么统计信息能够影响具体函数回调？不要把统计优化与用户可见的函数语义混为一谈。

运行已有测试，确认本周依赖的基本行为：

```bash
build/reldebug/test/unittest test/sql/function/string/test_length.test
```

## 实验三：检查扩展是否真的可用

在全新 CLI 会话中执行以下只读检查，关闭自动安装和自动加载，避免把网络下载混入学习实验：

```sql
SET autoinstall_known_extensions = false;
SET autoload_known_extensions = false;
SELECT extension_name, installed, loaded
FROM duckdb_extensions()
WHERE extension_name = 'json';
```

本次编写时得到 `json,false,false`，JSON 测试因 `require json` 被跳过。跳过不能算 JSON 行为已经通过验证。

有时间构建扩展时使用项目已有入口，构建耗时不算本周精读时间：

```bash
DUCKDB_EXTENSIONS='json' make reldebug
build/reldebug/test/unittest test/sql/json/scalar/test_json_valid.test
```

构建后重新检查扩展状态。在 JSON 可加载的会话里执行：

```sql
SET autoinstall_known_extensions = false;
SET autoload_known_extensions = false;
LOAD json;
SELECT json_valid('{"a":1}') AS valid_object,
       json_valid('{bad}') AS invalid_object,
       json_valid('{"a":1,}') AS trailing_comma,
       json_valid('') AS empty_text,
       json_valid(NULL) AS null_input;
```

依据当前测试与实现，预期为 `true,false,true,false,NULL`；这些 JSON 值在本文编写时未做运行验证。若 `LOAD json` 报扩展文件不存在，先检查构建产物与运行的 CLI 是否属于同一构建，记录阻塞原因，继续源码阅读即可。

尾逗号用例用于强调：函数语义以当前实现及测试为准，不能仅凭对标准 JSON 的印象猜答案。

## 源码练习：读通一个标量函数

将 `json_valid` 拆成四层并各记录一个符号：

1. **注册层：** 函数集合怎样被加入扩展提供的函数列表。
2. **签名层：** 参数类型与返回类型在哪里定义，两个重载是否共用回调。
3. **状态层：** `JSONFunctionLocalState::ResetAndGet` 提供什么本地状态，分配器从哪里来。
4. **执行层：** `UnaryExecutor::Execute<string_t, bool>` 如何连接输入向量与结果。

继续追踪执行器的 NULL 处理，说明 `ValidFunction` 的 lambda 为什么没有自己写 NULL 判断。不要把这一点推广成“所有标量函数都自动忽略 NULL”；函数可能有不同的 NULL 处理配置。

再把 `length` 的调用图画在旁边，标出共用框架与各自特有的计算部分。

## 本周产出模板

```markdown
# 函数行为与实现
- 名称、参数类型、返回类型：
- 空输入、NULL、边界值与错误输入的行为：
- 重载集合与注册入口：
- 执行回调、执行器与局部状态：
- 常量输入与表输入的计划差异：
- 运行证据：命令、输出、实际执行/跳过
- 下一周准备练习的行为及测试矩阵：
```

## 验收清单

- [ ] 字符串五类输入的结果与预期一致。
- [ ] 能解释码点、字节与显示字符的区别。
- [ ] 找到至少一个函数的签名、注册与执行入口。
- [ ] 能解释常量折叠为什么会影响调试断点。
- [ ] 知道何时复用 UnaryExecutor，何时才需要手写类型化向量读写。
- [ ] 明确记录 JSON 扩展的实际状态，没有把跳过记成通过。
- [ ] 写出一份完整行为规格，为第 12 周的小改动做准备。

## 自测与参考答案

1. **函数注册完成就一定能接收所有类型吗？** 不能；绑定需要匹配签名、重载与转换规则。
2. **为什么 `strlen('中文')` 等于 6？** UTF-8 编码中这两个汉字各占三个字节；此函数计字节。
3. **为什么函数内不一定看到 NULL 分支？** 通用执行器可能按配置处理 NULL；要沿配置和执行器确认。
4. **常量调用能验证整批向量处理吗？** 不能充分验证；还应使用表输入、NULL 和足够多的行。
5. **JSON 测试返回成功退出码就算通过吗？** 不够；必须检查测试确实执行，且没有因缺失扩展而跳过。

## 卡点、拓展与验证范围

模板太深时停在 `UnaryFunction → UnaryExecutor` 的接口层，先解释输入、输出和 NULL，再追一个具体模板实例。函数注册由生成流程参与时，阅读 [core_functions README](../extension/core_functions/README.md)，区分源定义与生成产物。

拓展可以选择 LIST/STRUCT 重载、函数统计信息或局部状态管理之一，每次只增加一个问题。

本次已验证字符串查询、表输入与常量输入的计划、返回类型及扩展状态；`test_length.test` 通过 11 个断言。JSON 运行实验与扩展构建未执行。源码符号按当前仓库核对，LLDB 断点练习和调用图由学习者完成。

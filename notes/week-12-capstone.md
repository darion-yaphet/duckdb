# 第 12 周：完成一个可验证的小改动

[总计划](learning-plan.md) · [上一周：函数与扩展](week-11-functions-and-extensions.md)

本周用约 7 小时，把前十一周的阅读和实验转化成一次独立的小型工程练习。目标是交付范围清楚、行为明确、测试可重复的本地改动。所有命令从仓库根目录执行。

## 本周目标与前置条件

开始前至少能做到：从 SQL 定位相关模块；阅读并运行目标测试；解释待修改行为；区分已验证事实和猜测。若这些条件尚未满足，选择下面的基础题即可。

本周不要求修改复杂优化规则、设计新存储格式或提升大型基准性能。新依赖、跨模块重构和多个独立特性会使范围失控，应留到后续专题。

## 选择一个题目

| 档位 | 题目 | 适合情况 | 合格交付 |
| --- | --- | --- | --- |
| 基础 | 为熟悉的函数补一组缺失的行为测试 | 已掌握 SQL 和测试，C++ 阅读仍慢 | 说明现有覆盖缺口；新增测试有清晰预期且真实执行 |
| 标准 | 修复一个能稳定复现的小问题 | 能解释根因和最小影响范围 | 修复前失败、修复后通过，相关行为不回归 |
| 进阶 | 实现练习函数 `study_is_even(BIGINT)` | 熟悉注册、签名和执行器 | 一份精确定义的规格、最小实现及边界测试 |

标准题需要先找到实际问题，不把不熟悉的行为直接定为 bug。基础题需要先检查覆盖，不为了增加测试数重复已有用例。进阶题仅作为本地教学练习，不代表建议项目增加这个函数。

## 四次学习安排

| 次数 | 时间 | 任务 | 当次完成标准 |
| --- | --- | --- | --- |
| 1 | 1.5 小时 | 选题、查现有行为和测试、写一页规格 | 范围与验收条件可以明确判断 |
| 2 | 2 小时 | 写最小测试，定位实现与注册入口 | 修复题有失败证据，其他题有确定预期 |
| 3 | 2 小时 | 做最小改动，重建并运行目标测试 | 目标行为通过，能解释每处源码变化 |
| 4 | 1.5 小时 | 相关回归、格式与 diff 检查、复盘 | 交付说明足够让别人独立复验 |

编译和广泛测试可能额外占用机器时间。第三次结束仍没理解根因时，把交付收窄为可复现问题与诊断记录，不堆叠试探性改动或宣称修复完成。

## 开始前记录工作区

```bash
git status --short
git rev-parse --short HEAD
git diff --stat
```

把已有变更记下来，保留其他人的工作。可在本地使用唯一命名的练习分支；已有学习分支也可继续，不需要为了这份练习强制切换或重置工作区。

确认构建方式与测试入口，阅读 [AGENTS.md](../AGENTS.md)、[CONTRIBUTING.md](../CONTRIBUTING.md) 和 [test/README.md](../test/README.md)。本周交付以本地文件和验证记录为准，对外提交另遵循仓库贡献要求。

## 写一页可验收规格

实现前填写下面的模板。最多选一个行为，不用“完善”“优化体验”这类无法判定完成的描述。

```markdown
# 练习规格
- 选择题目：基础 / 标准 / 进阶
- 当前提交与构建：
- 用户可见行为：给出最小 SQL 和当前输出
- 目标行为：给出期望输出或错误类别
- 证据：现有测试、源码入口、相关规则
- 修改范围：预计涉及的文件与函数
- 不包含：此次明确不处理的类型、功能或场景
- 边界：NULL、空输入、极值、错误类型、重复执行
- 验收命令：
- 成功标准：
```

标准题额外写一句根因假设和一条能证伪它的实验。基础题说明为什么现有测试没有覆盖所选组合。进阶题写清楚每一种输入的行为。

## 源码与测试定位

| 练习方向 | 阅读入口 | 要回答的问题 |
| --- | --- | --- |
| 字符串测试 | [test_length.test](../test/sql/function/string/test_length.test)、[length.cpp](../src/function/scalar/string/length.cpp) | 缺失的是新边界，还是已有用例的改写？ |
| 通用标量执行 | [scalar_function.hpp](../src/include/duckdb/function/scalar_function.hpp)：`ScalarFunction::UnaryFunction` | 现有执行器能否满足类型和 NULL 约定？ |
| 数值函数实现 | [numeric.cpp](../extension/core_functions/scalar/math/numeric.cpp) | 同类函数怎样声明签名、实现与注册？ |
| 函数源定义 | [functions.json](../extension/core_functions/scalar/math/functions.json)、[core_functions README](../extension/core_functions/README.md) | 哪些文件是源定义，哪些由生成流程产生？ |
| 构建清单 | [CMakeLists.txt](../extension/core_functions/scalar/math/CMakeLists.txt) | 新源文件是否需要加入目标？复用已有文件是否更合适？ |
| 测试框架 | [test/unittest.cpp](../test/unittest.cpp) | `--stdin` 如何执行练习测试，正式测试怎样被发现？ |

```bash
rg -n 'length\(|strlen\(' test/sql/function/string/test_length.test
rg -n 'UnaryFunction|UnaryExecutor' src/include/duckdb/function/scalar_function.hpp
rg -n 'GetFunction|GetFunctions' extension/core_functions/scalar/math/numeric.cpp
```

## 基础题示例：先练会编写与运行测试

以下是框架练习，**不声称它补足了当前项目的覆盖缺口**。先跑通，再搜索现有测试，选择一个确实有价值的组合。

```bash
study_capstone_dir=$(mktemp -d "${TMPDIR:-/tmp}/duckdb-week12.XXXXXX")
cat > "$study_capstone_dir/string_lengths.test" <<'TEST'
# name: week12_string_lengths.test
# description: Practice string length assertions
# group: [learning]

query II
SELECT length('中文'), strlen('中文');
----
2	6

query II
SELECT length('e' || chr(769)), strlen('e' || chr(769));
----
2	3

query II
SELECT length(NULL::VARCHAR), strlen(NULL::VARCHAR);
----
NULL	NULL
TEST
build/reldebug/test/unittest --stdin < "$study_capstone_dir/string_lengths.test"
```

将输出记录为“测试框架练习已通过”。若把练习转成正式补充测试，按项目目录和命名约定放置，再用正常测试文件入口执行，确认实际发现了用例。

## 进阶题规格：study_is_even

这个名字表示尚未实现的练习函数。当前数据库中没有它；不要把下面的目标结果当作现有功能。

- 输入类型：`BIGINT`；输出类型：`BOOLEAN`。
- 非 NULL 输入返回是否为偶数；负数也遵守同一规则。
- NULL 传播为 NULL；不使用“NULL 当作 0”的隐含转换。
- 极值必须有定义，避免通过 `abs(x)` 等对最小负数可能溢出的中间操作判断。
- 只增加 BIGINT 签名；其他 SQL 类型能否调用取决于现有绑定与转换规则，不笼统声明全部拒绝。

| 测试输入 | 目标结果 | 覆盖原因 |
| --- | --- | --- |
| `0`、`2` | true | 零与正常偶数 |
| `1`、`-3` | false | 正负奇数 |
| `-2` | true | 负偶数 |
| `-9223372036854775808` | true | BIGINT 最小值 |
| `9223372036854775807` | false | BIGINT 最大值 |
| `NULL::BIGINT` | NULL | NULL 传播 |

先用已有 SQL 表达式验证数学预期，避免把错误答案写进测试：

```sql
WITH cases(x) AS (
    VALUES ('-9223372036854775808'::BIGINT), (-3::BIGINT),
           (-2::BIGINT), (0::BIGINT), (1::BIGINT), (2::BIGINT),
           ('9223372036854775807'::BIGINT), (NULL::BIGINT)
)
SELECT x, x % 2 = 0 AS expected FROM cases ORDER BY x NULLS LAST;
```

实现步骤：先写引用 `study_is_even` 的测试，确认失败原因是函数尚未存在；再沿已读懂的注册模式添加源定义与最小实现；使用现有逐元素执行器；最后重建并重复相同测试。这里不给整段 C++ 答案，要求自己解释签名、回调和 NULL 的处理位置。

## 测试矩阵与实现约束

| 层面 | 至少覆盖 | 注意事项 |
| --- | --- | --- |
| 语义 | 正常值、空值、边界值 | 预期不能由待测实现直接生成 |
| 输入来源 | 常量与表列 | 常量折叠可能绕开部分运行时路径 |
| 数据量 | 小数据与跨批次输入 | 如表中插入 10000 行，验证结果而非只确认不崩溃 |
| 类型 | 正确类型、一个不兼容类型或显式转换场景 | 先确认绑定规则再写错误断言 |
| 重复性 | 同一测试重跑两次 | 不依赖上次遗留表、文件或设置 |
| 回归 | 目标文件与相关已有测试 | 新功能通过不代表旧行为无变化 |

不要为此次练习引入依赖。一般向量读写遵循新 API，逐元素标量函数使用现有执行器；不要从旧代码复制手动 selection/validity 循环。新增慢测试用 `.test_slow`，新测试不添加 `PRAGMA enable_verification`。

## 验证顺序

1. 先读差异，确认变更只覆盖规格中列出的行为。
2. 修改了生成源定义时运行项目要求的 `make generate-files`；普通测试改动不需要无条件生成文件。
3. 执行 `make reldebug`，再运行新测试和相关原测试。
4. 执行格式与差异检查；格式工具可能修改其他文件，复查每处变化。
5. 按影响范围扩大到快速测试集；准备完整提交时按仓库要求运行 `make allunit`，涉及向量或配置时增加相应检查。

```bash
make format-fix
git diff --check
git diff --stat
git status --short
```

本页只是安排工作，不表示这些构建、格式化或全量测试已经执行。测试仅用新增文件时，`git diff` 不会展示未跟踪文件的内容，要结合 `git status` 与实际文件检查。

## 最终交付模板

```markdown
# 第 12 周练习结果
- 问题与目标行为：
- 修改文件及每处修改理由：
- 实现依据：关键符号、注册路径、类型与NULL约定
- 修复前证据（修复题必填）：
- 验证命令与实际结果：
- 已有行为的回归覆盖：
- 未执行或跳过的检查及原因：
- 限制、后续问题：
- 五分钟讲解：SQL → 绑定 → 执行 → 结果
```

## 验收清单、自测与复盘

- [ ] 一个明确题目与完整规格，没有中途混入无关改动。
- [ ] 每个结果有独立的预期依据，失败原因与目标问题相关。
- [ ] 能解释修改文件、注册与生成流程，没有只改生成产物。
- [ ] 新测试实际执行；修复题保留修复前失败和修复后通过的证据。
- [ ] 相关回归通过，格式与差异已检查；未执行项明确记录。
- [ ] 能在五分钟内解释整个变化，并留下别人可重复执行的命令。

1. **测试全绿是否足够？** 不足；还要确认测试覆盖目标、没有跳过或零匹配，且运行的是相应构建。
2. **新测试原本就通过，是不是没价值？** 不一定，可能补足真实覆盖；但不能把它当作某个修复已生效的证据。
3. **性能变快能证明语义正确吗？** 不能；正确性与性能需要不同证据，先验证结果和边界行为。
4. **实现超过预算怎么办？** 缩小题目或保留诊断记录，并清楚标明未完成；不通过删掉重要用例制造完成状态。

完成后回看第 2 周的查询图，把本次改动标在图上；再选优化、执行、存储或函数中的一个方向继续四到六周。

## 本页验证范围

本次验证了字符串 sqllogictest 示例与已有取模表达式的极值结果；`study_is_even` 仅为拟实现规格，没有新增该函数或执行源码修改。构建、格式化、生成文件及全量回归是学员实施改动后的任务。

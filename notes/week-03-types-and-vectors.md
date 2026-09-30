# 第 3 周：掌握类型、Vector 与 DataChunk

[上一周：查询生命周期](week-02-query-lifecycle.md) · [返回总计划](learning-plan.md) · [下一周：Parser、Binder 与 Catalog](week-04-parser-binder-catalog.md)

本周学习执行器处理数据的基本单位。重点是建立“逻辑行、底层位置、NULL 有效性和列向量表示”之间的关系，并认识当前项目推荐的类型化读写 API。

## 本周目标

完成本周后，你应当能够：

- 解释 `LogicalType`、`Vector`、`DataChunk`、`SelectionVector` 和 `ValidityMask` 的职责。
- 区分 FLAT、CONSTANT、DICTIONARY 和 SEQUENCE 等向量表示。
- 解释逻辑第 `i` 行为什么不一定存储在底层第 `i` 个位置。
- 知道新代码何时使用 `Vector::Values<T>()`、`ValidValues<T>()` 和 `FlatVector::Writer<T>()`。
- 说明嵌套 LIST/STRUCT 为什么不能简化成普通标量数组。

## 前置条件与本周范围

先完成第 2 周，知道物理算子之间传递的是批量数据。本周只阅读向量接口和做 SQL 行为实验，不修改 C++，不深入哈希表、排序或存储压缩。

从仓库根目录执行所有命令：

```bash
cd /Users/darion.yaphet/source/duckdb
```

本周所有 SQL 都放在一条 `-c` 中，因此共享一个全新的内存会话，不依赖第 1 周留下的表。

## 第 1 次学习：从 SQL 类型观察逻辑模型（1.5 小时）

先预测四个 `typeof` 的结果，再运行：

```bash
build/reldebug/duckdb :memory: -csv -c "
SELECT
    typeof(42::INTEGER) AS integer_type,
    typeof('duck'::VARCHAR) AS string_type,
    typeof([1, NULL, 3]) AS list_type,
    typeof(struct_pack(id := 42, name := 'duck')) AS struct_type;
"
```

可核对结果：

```text
integer_type,string_type,list_type,struct_type
INTEGER,VARCHAR,INTEGER[],"STRUCT(id INTEGER, ""name"" VARCHAR)"
```

把输出中的 CSV 引号与 SQL 类型本身分开理解。`""name""` 是 CSV 对双引号的转义，逻辑类型仍是包含字段 `id` 和 `name` 的 STRUCT。

阅读 `LogicalType` 在 [types.hpp](../src/include/duckdb/common/types.hpp) 中的声明和常用静态类型。只回答：

- 类型对象描述的是一整列还是某个具体值？
- LIST 的子类型和 STRUCT 的字段类型为什么也必须保存在类型信息中？
- SQL `NULL` 为什么仍然需要所属类型？

## 第 2 次学习：Vector 表示与逻辑行（2 小时）

阅读 `VectorType` 枚举和 `Vector` 的构造、`Reference`、`Slice`、`Dictionary`、`Flatten`、`GetVectorType`。

为四种常见表示写一句话：

- FLAT：每个位置通常对应一个物理值槽位。
- CONSTANT：多个逻辑行共享一个值。
- DICTIONARY：通过 selection 映射到另一个向量。
- SEQUENCE：用起点和步长表示一段数列。

手画下面的过滤例子，不需要写 C++：

```text
底层值位置:  0   1   2   3   4
底层值:     10  20  30  40  50
selection:   4   1   3
逻辑行:      0   1   2
逻辑值:     50  20  40
```

回答：逻辑第 0 行应访问哪个底层位置？如果底层位置 1 为 NULL，应检查逻辑索引还是映射后的物理索引？

然后阅读 [vector_iterator.hpp](../src/include/duckdb/common/vector/vector_iterator.hpp) 中 `VectorIterator<T>`。确认它在构造时取得统一格式，并由 `ValueEntry` 封装 selection 与 validity 的映射。调用者通过 `entry.IsValid()` 和 `entry.GetValue()` 读取，不应在普通类型化循环中自己重复索引逻辑。

## 第 3 次学习：DataChunk、NULL 与嵌套值（2 小时）

运行一个自包含实验：

```bash
build/reldebug/duckdb :memory: -csv -c "
SELECT
    i,
    CASE WHEN i % 2 = 0 THEN i * 10 END AS maybe_value,
    [i, NULL] AS pair,
    struct_pack(id := i, even := i % 2 = 0) AS item
FROM range(4) AS t(i)
ORDER BY i;
"
```

可核对结果：

```text
i,maybe_value,pair,item
0,0,"[0, NULL]","{'id': 0, 'even': true}"
1,NULL,"[1, NULL]","{'id': 1, 'even': false}"
2,20,"[2, NULL]","{'id': 2, 'even': true}"
3,NULL,"[3, NULL]","{'id': 3, 'even': false}"
```

对这个结果画一个 `DataChunk`：4 行、4 列，每列标出 `LogicalType`，并在 `maybe_value` 上标出无效行。LIST 列需要表达每行列表边界和子元素，STRUCT 列需要表达两个子向量。

阅读 [data_chunk.hpp](../src/include/duckdb/common/types/data_chunk.hpp)：

- `data` 保存列向量。
- `size()` 返回逻辑行数，`ColumnCount()` 返回列数。
- `Reference` 可让 chunk 引用另一个 chunk 的数据。
- `SetCardinality` 已弃用；当前注释要求优先使用 `CheckCardinality` 或 `SetChildCardinality`。

注意：Vector 现在携带自己的 size，不能从旧代码推断“所有行数只由 DataChunk 外部传入”。

## 第 4 次学习：当前读写 API 与对象关系表（1.5 小时）

按顺序阅读：

1. `Vector::Values<T>()`：适用于任意向量表示的类型化读取。
2. `Vector::ValidValues<T>()`：迭代时跳过 NULL 行。
3. `Vector::Validity()`：只遍历有效性。
4. `FlatVector::Writer<T>()`：按顺序写值或 NULL，writer 自己推进位置。
5. `VectorStructType` 与 `VectorListType` 的 iterator/writer 特化。

形成以下使用规则：

- 一般读取：在循环外构造一次 `auto values = vec.Values<T>()`。
- 读取值前：先调用 `IsValid()`；只有确定有效时再 `GetValue()`。
- 一般顺序写入：创建 writer 后严格写满声明的 `count`；故意少写要 `Truncate()`。
- 逐元素标量函数：优先使用 `UnaryExecutor`、`BinaryExecutor` 或 `GenericExecutor`。
- 类型擦除的底层系统仍可能直接使用 `ToUnifiedFormat`。
- `VectorListType<T>` 不接受 DICTIONARY_VECTOR 来源；可能是字典时先按项目约定 flatten。

不要把 `Values<T>()` 理解为“底层一定已经是连续数组”。它内部统一了表示差异，逻辑迭代仍会尊重 selection 与 validity。

## 源码阅读顺序

| 顺序 | 文件与符号 | 关注点 | 必须回答的问题 |
| ---: | --- | --- | --- |
| 1 | [vector_type.hpp](../src/include/duckdb/common/enums/vector_type.hpp) `VectorType` | 六种表示枚举 | 表示方式与 SQL 逻辑类型有何不同？ |
| 2 | [vector.hpp](../src/include/duckdb/common/types/vector.hpp) `Vector` | type、size、buffer、auxiliary data | Vector 自己保存哪些元数据？ |
| 3 | 同文件 `Reference`、`Slice`、`Dictionary`、`Flatten` | 引用、选择与物化 | 哪些操作可以避免复制？ |
| 4 | [data_chunk.hpp](../src/include/duckdb/common/types/data_chunk.hpp) `DataChunk` | 多列与统一基数 | chunk 如何表达 N 行 M 列？ |
| 5 | [vector_iterator.hpp](../src/include/duckdb/common/vector/vector_iterator.hpp) `VectorIterator<T>`、`ValueEntry` | selection 与 NULL | `GetIndex()`、selection index 和值位置有什么区别？ |
| 6 | 同文件 LIST/STRUCT 特化 | 嵌套读取 | 子元素和子字段如何暴露给调用者？ |
| 7 | [vector_writer.hpp](../src/include/duckdb/common/vector/vector_writer.hpp) `VectorWriter<T>` | 顺序写入和析构断言 | 为什么 writer 必须写满 count？ |
| 8 | [flat_vector.hpp](../src/include/duckdb/common/vector/flat_vector.hpp) `FlatVector::Writer` | writer 工厂入口 | 为什么 writer 明确针对 flat result？ |

## 本周产出模板

```markdown
# 核心对象关系表

| 对象 | 表示什么 | 保存/引用什么 | 行数或索引从哪里来 | NULL 如何表达 |
| --- | --- | --- | --- | --- |
| LogicalType | | | | |
| Vector | | | | |
| DataChunk | | | | |
| SelectionVector | | | | |
| ValidityMask | | | | |
| VectorIterator | | | | |
| VectorWriter | | | | |

## 过滤映射示例
- 底层值：
- selection：
- 逻辑行到物理位置：
- NULL 检查过程：
- 是否复制数据，依据是什么：
```

## 验收 checklist

- [ ] 能解释逻辑类型与向量表示是两个维度。
- [ ] 能画出 selection `[4, 1, 3]` 的逻辑行到物理位置映射。
- [ ] 能解释 NULL 检查为什么要考虑映射后的索引。
- [ ] 能说出 `DataChunk::size()` 与 `ColumnCount()` 的含义。
- [ ] 能解释 `Reference`/`Slice` 如何支持避免复制。
- [ ] 知道一般新代码使用 `Values<T>()` 和 `FlatVector::Writer<T>()`。
- [ ] SQL 类型与嵌套值实验得到预期结果。

## 自测题与参考答案

1. CONSTANT_VECTOR 有 100 个逻辑行时，是否必须存 100 份值？
   - 不必；它可让逻辑行共享一个常量值。
2. DICTIONARY_VECTOR 的逻辑第 `i` 行一定对应底层第 `i` 项吗？
   - 不一定；应通过 selection 找到底层位置。
3. `ValidValues<T>()` 与 `Values<T>()` 的主要差别是什么？
   - 前者跳过 NULL，后者保留所有逻辑行并让调用者检查 `IsValid()`。
4. 为什么 `GetValue()` 前要检查 `IsValid()`？
   - `GetValue()` 断言值有效；NULL 位置的底层数据不应当作有效值读取。
5. 向量化是否等同于 SIMD？
   - 不等同。向量化指批量处理；SIMD 是可用于批量处理的一种 CPU 指令层优化。
6. 为什么 LIST 需要专门 iterator？
   - 每行有长度/偏移和子向量，不能只用一个普通标量指针描述。

## 常见卡点

- 把 VectorType 当成 SQL 类型：前者是物理表示，后者是值的逻辑类型。
- 看到 `ToUnifiedFormat` 就照抄旧式手动循环：普通新代码应使用 iterator；底层类型擦除代码才有例外。
- 为读取而无条件 `Flatten()`：`Values<T>()` 可读多种向量表示，通常不需要先物化。
- 把 DataChunk 当成行对象数组：它拥有多个列向量和一个共同逻辑基数。
- 忽略 writer 析构断言：声明写 `count` 行后必须写满，或显式 `Truncate()`。
- 对 LIST 的 dictionary 限制一概而论：限制针对 `VectorListType<T>` iterator，先核对实际来源表示。

## 可选拓展

- 阅读 `VectorIterator<VectorStructType<...>>` 的 `ForEach`，说明编译期字段类型如何保留。
- 阅读 `VectorIterator<VectorListType<T>>` 的 `GetChildValues()`，画出一行 LIST 的子范围。
- 在后续调试器练习中检查一个 `DataChunk` 的 `size()`、`ColumnCount()` 和各列 `GetVectorType()`。

## 本页示例的验证状态

已在提交 `d6b403d382` 的现有 `build/reldebug/duckdb` 上验证类型查询及 4 行嵌套值查询，输出与本页一致。当前 Vector、DataChunk、iterator 和 writer 的符号与弃用说明已按源码核对。本文未编译任何新的 C++ 示例、未修改源码，也未在调试器中检查运行时 DataChunk；这些观察练习由学习者执行。

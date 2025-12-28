# DuckDB 执行引擎篇：详细目录

执行引擎是 DuckDB 高性能的核心所在。本篇将深入分析 DuckDB 的向量化执行模型、Pipeline 并行框架、物理算子实现等关键技术。

---

## 篇章结构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    执行引擎篇 (6章)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  第1章  执行模型概述                                         │
│         ├── Push vs Pull 模型对比                           │
│         ├── 向量化执行原理                                   │
│         └── Pipeline 执行框架                               │
│                                                              │
│  第2章  向量化数据结构                                       │
│         ├── Vector 核心结构                                 │
│         ├── DataChunk 批量数据                              │
│         └── SelectionVector 选择向量                        │
│                                                              │
│  第3章  表达式执行器                                         │
│         ├── ExpressionExecutor 架构                         │
│         ├── 表达式类型与执行                                 │
│         └── 向量化表达式求值                                 │
│                                                              │
│  第4章  物理算子实现                                         │
│         ├── PhysicalOperator 基类                           │
│         ├── 扫描算子 (Scan)                                 │
│         ├── 过滤与投影 (Filter/Projection)                  │
│         ├── 连接算子 (Join)                                 │
│         ├── 聚合算子 (Aggregate)                            │
│         └── 排序与TopN (Order/Limit)                        │
│                                                              │
│  第5章  Pipeline 并行执行                                    │
│         ├── Pipeline 结构与生命周期                         │
│         ├── MetaPipeline 协调                               │
│         ├── 任务调度与线程池                                 │
│         └── 事件驱动执行                                     │
│                                                              │
│  第6章  Hash Table 与内存管理                                │
│         ├── JoinHashTable                                   │
│         ├── AggregateHashTable                              │
│         ├── 内存预算与溢出                                   │
│         └── Morsel-Driven 并行                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 第1章：执行模型概述

### 章节目标
理解 DuckDB 执行引擎的设计理念和整体架构

### 详细大纲

```
1.1 数据库执行模型演进
    1.1.1 火山模型 (Volcano/Iterator Model)
          • 逐行处理 (Tuple-at-a-time)
          • next() 接口
          • 优点：简单、内存友好
          • 缺点：函数调用开销大、缓存不友好

    1.1.2 向量化模型 (Vectorized Model)
          • 批量处理 (Vector-at-a-time)
          • 2048 行为一批
          • SIMD 友好
          • DuckDB 的选择

    1.1.3 编译执行模型 (Compiled Model)
          • 代码生成 (Code Generation)
          • HyPer/Umbra 的选择
          • 与向量化的对比

1.2 Push vs Pull 执行
    1.2.1 Pull 模型 (传统火山模型)
          • 消费者驱动
          • 递归调用栈
          • 流程控制在顶层

    1.2.2 Push 模型 (DuckDB 选择)
          • 生产者驱动
          • 数据主动推送
          • 更好的并行支持
          • Pipeline 天然适配

1.3 DuckDB 执行架构
    1.3.1 查询执行流程
          LogicalPlan → PhysicalPlan → Pipeline → Execution

    1.3.2 核心组件关系
          ┌─────────────────────────────────────┐
          │ Executor                            │
          │   └── PipelineExecutor              │
          │         └── PhysicalOperator        │
          │               └── ExpressionExecutor│
          └─────────────────────────────────────┘

    1.3.3 执行上下文
          • ClientContext
          • ExecutionContext
          • ThreadContext

1.4 向量化执行原理
    1.4.1 为什么是 2048?
          • CPU 缓存考量
          • SIMD 对齐
          • 内存带宽平衡

    1.4.2 向量化的收益
          • 减少解释开销
          • 提高缓存命中率
          • 利用 SIMD 指令
          • 编译器优化友好

    1.4.3 向量化的挑战
          • 控制流处理 (SelectionVector)
          • 字符串处理
          • 复杂表达式
```

### 源码参考
- `src/include/duckdb/execution/executor.hpp`
- `src/include/duckdb/execution/physical_operator.hpp`
- `src/include/duckdb/execution/execution_context.hpp`
- `src/parallel/executor.cpp`

---

## 第2章：向量化数据结构

### 章节目标
深入理解 Vector、DataChunk 等核心数据结构的设计

### 详细大纲

```
2.1 Vector：向量化的基石
    2.1.1 Vector 结构
          class Vector {
              VectorType vector_type;      // 向量类型
              LogicalType type;            // 数据类型
              data_ptr_t data;             // 数据指针
              ValidityMask validity;       // NULL 标记
              unique_ptr<VectorBuffer> buffer;  // 缓冲区
              unique_ptr<VectorAuxiliaryData> auxiliary;  // 辅助数据
          };

    2.1.2 VectorType 类型
          • FLAT_VECTOR      - 扁平向量 (最常见)
          • CONSTANT_VECTOR  - 常量向量 (所有值相同)
          • DICTIONARY_VECTOR - 字典向量 (压缩)
          • SEQUENCE_VECTOR  - 序列向量 (如 row_id)
          • FSST_VECTOR      - FSST 压缩字符串

    2.1.3 ValidityMask
          • 位图表示 NULL
          • 64 位为一组
          • AllValid 优化

2.2 DataChunk：批量数据容器
    2.2.1 结构设计
          class DataChunk {
              vector<Vector> data;     // 列向量数组
              idx_t count;             // 有效行数 (≤2048)
              idx_t capacity;          // 容量
          };

    2.2.2 常用操作
          • Initialize() - 初始化
          • SetCardinality() - 设置行数
          • Append() - 追加数据
          • Flatten() - 扁平化
          • Slice() - 切片
          • Reference() - 引用

2.3 SelectionVector：选择向量
    2.3.1 作用
          • 表示"哪些行有效"
          • 避免数据拷贝
          • 实现向量化过滤

    2.3.2 结构
          class SelectionVector {
              sel_t *sel_vector;  // 选择的行索引数组
          };

    2.3.3 使用场景
          • Filter 算子
          • NULL 处理
          • Join 匹配

2.4 字符串处理
    2.4.1 string_t 结构
          • 内联短字符串 (≤12字节)
          • 指针+长度 (长字符串)

    2.4.2 StringHeap
          • 统一字符串存储
          • 生命周期管理

    2.4.3 Dictionary 编码
          • 字典向量
          • 压缩与解压

2.5 复杂类型向量
    2.5.1 LIST 向量
          • child_vector + offset_vector

    2.5.2 STRUCT 向量
          • child_vectors 数组

    2.5.3 MAP 向量
          • 基于 LIST 实现
```

### 源码参考
- `src/include/duckdb/common/types/vector.hpp`
- `src/common/types/vector.cpp` (107KB，核心实现)
- `src/include/duckdb/common/types/data_chunk.hpp`
- `src/common/types/data_chunk.cpp`
- `src/include/duckdb/common/types/selection_vector.hpp`
- `src/include/duckdb/common/types/validity_mask.hpp`

---

## 第3章：表达式执行器

### 章节目标
理解 SQL 表达式如何被向量化执行

### 详细大纲

```
3.1 ExpressionExecutor 架构
    3.1.1 核心结构
          class ExpressionExecutor {
              vector<unique_ptr<ExpressionExecutorState>> states;
              ClientContext &context;
          };

    3.1.2 执行状态
          • ExpressionExecutorState
          • ExpressionState (每个表达式)

3.2 表达式类型
    3.2.1 BoundExpression 层次
          • BoundConstantExpression   - 常量
          • BoundColumnRefExpression  - 列引用
          • BoundFunctionExpression   - 函数调用
          • BoundComparisonExpression - 比较
          • BoundConjunctionExpression - AND/OR
          • BoundCaseExpression       - CASE WHEN
          • BoundCastExpression       - 类型转换

    3.2.2 表达式求值顺序
          • 深度优先
          • 自底向上

3.3 向量化表达式执行
    3.3.1 常量表达式
          • 直接设置 CONSTANT_VECTOR

    3.3.2 列引用
          • Reference 到输入 DataChunk

    3.3.3 函数执行
          ScalarFunction {
              function_t function;  // 向量化函数指针
          };

          // 函数签名
          void Function(DataChunk &args,
                        ExpressionState &state,
                        Vector &result);

    3.3.4 比较与逻辑
          • VectorOperations 工具函数
          • 短路求值优化

3.4 特殊表达式处理
    3.4.1 CASE WHEN
          • 分支向量化
          • SelectionVector 分割

    3.4.2 NULL 处理
          • ValidityMask 传播
          • Three-valued logic

    3.4.3 IN 表达式
          • 小列表：展开比较
          • 大列表：构建 HashSet
```

### 源码参考
- `src/include/duckdb/execution/expression_executor.hpp`
- `src/execution/expression_executor.cpp`
- `src/execution/expression_executor/` (各类表达式执行器)
- `src/common/vector_operations/` (向量操作)

---

## 第4章：物理算子实现

### 章节目标
掌握各类物理算子的向量化实现

### 详细大纲

```
4.1 PhysicalOperator 基类
    4.1.1 核心接口
          class PhysicalOperator {
              // Source 接口 (Pipeline 起点)
              virtual SourceResultType GetData(
                  ExecutionContext &context,
                  DataChunk &chunk,
                  OperatorSourceInput &input);

              // Sink 接口 (Pipeline 终点)
              virtual SinkResultType Sink(
                  ExecutionContext &context,
                  DataChunk &chunk,
                  OperatorSinkInput &input);

              // Operator 接口 (中间算子)
              virtual OperatorResultType Execute(
                  ExecutionContext &context,
                  DataChunk &input,
                  DataChunk &chunk,
                  GlobalOperatorState &gstate,
                  OperatorState &state);
          };

    4.1.2 算子状态
          • GlobalOperatorState - 全局共享
          • LocalSourceState   - 线程本地 Source
          • LocalSinkState     - 线程本地 Sink
          • OperatorState      - 算子状态

    4.1.3 结果类型
          • HAVE_MORE_OUTPUT
          • FINISHED
          • BLOCKED

4.2 扫描算子 (Scan)
    4.2.1 PhysicalTableScan
          • 表扫描实现
          • 并行扫描分片
          • 投影下推
          • 过滤下推

    4.2.2 PhysicalColumnDataScan
          • 内存数据扫描

    4.2.3 并行扫描
          • 行组级别并行
          • Morsel 划分

4.3 过滤与投影
    4.3.1 PhysicalFilter
          • 向量化条件求值
          • SelectionVector 生成
          • 过滤后 Slice

    4.3.2 PhysicalProjection
          • 表达式计算
          • 结果向量生成

4.4 连接算子 (Join)
    4.4.1 PhysicalHashJoin
          • Build 阶段 (Sink)
          • Probe 阶段 (Source)
          • JoinHashTable 使用

    4.4.2 PhysicalNestedLoopJoin
          • 小表适用
          • 向量化笛卡尔积

    4.4.3 PhysicalMergeJoin
          • 有序数据
          • 范围连接

    4.4.4 其他 Join
          • PhysicalIEJoin (不等连接)
          • PhysicalAsOfJoin (时序连接)
          • PhysicalPositionalJoin (位置连接)

4.5 聚合算子 (Aggregate)
    4.5.1 PhysicalHashAggregate
          • AggregateHashTable
          • GROUP BY 处理
          • 聚合函数状态

    4.5.2 PhysicalPerfectHashAggregate
          • 小基数优化
          • 数组直接寻址

    4.5.3 PhysicalStreamingAggregate
          • 有序数据聚合
          • 流式处理

    4.5.4 PhysicalUngroupedAggregate
          • 无 GROUP BY
          • 简单聚合

4.6 排序与 TopN
    4.6.1 PhysicalOrder
          • RadixSort
          • 外部排序
          • 并行归并

    4.6.2 PhysicalTopN
          • 堆排序
          • TopN 优化

    4.6.3 PhysicalLimit
          • 提前终止
          • OFFSET 处理
```

### 源码参考
- `src/include/duckdb/execution/physical_operator.hpp`
- `src/execution/physical_operator.cpp`
- `src/execution/operator/` (各类算子)
- `src/execution/operator/scan/`
- `src/execution/operator/join/`
- `src/execution/operator/aggregate/`
- `src/execution/operator/order/`

---

## 第5章：Pipeline 并行执行

### 章节目标
理解 DuckDB 的 Pipeline 并行执行框架

### 详细大纲

```
5.1 Pipeline 概念
    5.1.1 什么是 Pipeline
          • 算子链 (Source → Operators → Sink)
          • 无需物化中间结果
          • 数据流式处理

    5.1.2 Pipeline Breaker
          • 需要物化的算子
          • Hash Join Build
          • Hash Aggregate
          • Sort

    5.1.3 Pipeline 示例
          SELECT a, SUM(b) FROM t WHERE c > 10 GROUP BY a

          Pipeline 1: Scan → Filter → Aggregate(Build)
          Pipeline 2: Aggregate(Scan) → Result

5.2 Pipeline 结构
    5.2.1 Pipeline 类
          class Pipeline {
              weak_ptr<Executor> executor;
              PhysicalOperator &source;           // 数据源
              PhysicalOperator *sink;             // 数据汇
              vector<reference<PhysicalOperator>> operators;  // 中间算子
              vector<shared_ptr<Pipeline>> children;  // 依赖
              vector<shared_ptr<Pipeline>> parents;   // 被依赖
              bool ready;                         // 是否就绪
              atomic<idx_t> total_tasks;          // 任务数
          };

    5.2.2 Pipeline 状态
          • 初始化
          • 等待依赖
          • 就绪
          • 执行中
          • 完成

5.3 MetaPipeline
    5.3.1 作用
          • 管理相关 Pipeline 组
          • 处理依赖关系
          • Union/Intersect 等

    5.3.2 构建过程
          • 从 Physical Plan 生成
          • 识别 Pipeline Breaker
          • 建立依赖图

5.4 PipelineExecutor
    5.4.1 执行循环
          while (!finished) {
              // 1. 从 Source 获取数据
              source.GetData(chunk);

              // 2. 经过中间算子
              for (op : operators) {
                  op.Execute(chunk, result);
                  swap(chunk, result);
              }

              // 3. 推送到 Sink
              sink.Sink(chunk);
          }

    5.4.2 状态管理
          • 算子状态初始化
          • 中间结果缓存
          • 完成处理

5.5 任务调度
    5.5.1 TaskScheduler
          • 全局任务队列
          • 工作线程池
          • 任务窃取

    5.5.2 Task 类型
          • PipelineTask - 执行 Pipeline
          • PipelineFinishTask - 完成处理
          • ExecutorTask - 通用任务

    5.5.3 线程模型
          • 主线程提交
          • 工作线程执行
          • 回调通知

5.6 事件驱动
    5.6.1 Event 机制
          • PipelineEvent
          • PipelineCompleteEvent
          • PipelineFinishEvent

    5.6.2 异步执行
          • Interrupt 机制
          • 异步 IO
          • 结果流式返回
```

### 源码参考
- `src/include/duckdb/parallel/pipeline.hpp`
- `src/parallel/pipeline.cpp`
- `src/include/duckdb/parallel/pipeline_executor.hpp`
- `src/parallel/pipeline_executor.cpp`
- `src/include/duckdb/parallel/meta_pipeline.hpp`
- `src/parallel/meta_pipeline.cpp`
- `src/include/duckdb/parallel/task_scheduler.hpp`
- `src/parallel/task_scheduler.cpp`
- `src/parallel/executor.cpp`

---

## 第6章：Hash Table 与内存管理

### 章节目标
理解核心数据结构和内存管理策略

### 详细大纲

```
6.1 JoinHashTable
    6.1.1 结构设计
          class JoinHashTable {
              // Hash 表
              unique_ptr<HashTableEntry[]> hash_map;
              idx_t capacity;

              // 数据存储
              TupleDataCollection data_collection;

              // 构建状态
              vector<idx_t> entry_sizes;
              bool finalized;
          };

    6.1.2 构建过程 (Build)
          • 分配 Hash 表
          • 插入数据
          • 处理冲突 (链表)

    6.1.3 探测过程 (Probe)
          • 计算 Hash
          • 查找匹配
          • 输出结果

    6.1.4 并行构建
          • 分区构建
          • 合并 Hash 表

6.2 AggregateHashTable
    6.2.1 结构设计
          class AggregateHashTable {
              TupleDataCollection data_collection;
              unique_ptr<AggregateHTEntry[]> hash_map;
              vector<AggregateObject> aggregates;
          };

    6.2.2 聚合状态
          • 状态初始化
          • 状态更新
          • 状态合并
          • 结果输出

    6.2.3 分组处理
          • GROUP BY 列 Hash
          • 相等比较
          • 新组创建

6.3 RadixPartitionedHashTable
    6.3.1 分区策略
          • Radix 分区
          • 减少缓存冲突
          • 并行友好

    6.3.2 溢出处理
          • 内存超限检测
          • 分区写入磁盘
          • 分区读回处理

6.4 内存管理
    6.4.1 内存预算
          • 查询级别预算
          • 算子级别预算
          • BufferManager 集成

    6.4.2 溢出到磁盘
          • 临时文件管理
          • 外部排序
          • 外部聚合

    6.4.3 内存复用
          • DataChunk 池化
          • Vector 复用
          • 状态复用

6.5 Morsel-Driven 并行
    6.5.1 概念
          • Morsel = 一批行 (约 100K)
          • 动态任务分配
          • 负载均衡

    6.5.2 实现
          • RowGroup 对应 Morsel
          • 原子任务获取
          • 局部聚合 + 全局合并

    6.5.3 优势
          • 自适应并行度
          • 减少线程同步
          • 缓存友好
```

### 源码参考
- `src/include/duckdb/execution/join_hashtable.hpp`
- `src/execution/join_hashtable.cpp` (70KB)
- `src/include/duckdb/execution/aggregate_hashtable.hpp`
- `src/execution/aggregate_hashtable.cpp` (38KB)
- `src/include/duckdb/execution/radix_partitioned_hashtable.hpp`
- `src/execution/radix_partitioned_hashtable.cpp` (40KB)

---

## 预估篇幅

| 章节 | 主题 | 预估大小 |
|------|------|----------|
| 第1章 | 执行模型概述 | ~20KB |
| 第2章 | 向量化数据结构 | ~25KB |
| 第3章 | 表达式执行器 | ~20KB |
| 第4章 | 物理算子实现 | ~35KB |
| 第5章 | Pipeline 并行执行 | ~25KB |
| 第6章 | Hash Table 与内存管理 | ~20KB |

**总计：约 145KB**

---

## 关键技术点标注

```
必读亮点:
├── ⭐⭐⭐ Vector 结构设计 (第2章)
├── ⭐⭐⭐ 向量化函数执行 (第3章)
├── ⭐⭐⭐ Hash Join 实现 (第4章)
├── ⭐⭐⭐ Pipeline 模型 (第5章)
└── ⭐⭐⭐ Morsel-Driven 并行 (第6章)

核心创新:
├── Push-based 执行模型
├── 2048 行向量化批处理
├── SelectionVector 过滤
├── 多种 VectorType 优化
└── 事件驱动的并行调度
```

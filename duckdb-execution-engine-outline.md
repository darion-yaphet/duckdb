# DuckDB 执行引擎深度解析：系列目录

## 系列概述

本系列深入剖析 DuckDB 的执行引擎，从物理计划到最终结果输出的完整执行过程。涵盖向量化执行、Pipeline 调度、算子实现、并行执行和内存管理等核心组件。

```
┌─────────────────────────────────────────────────────────────────┐
│                    DuckDB 执行引擎架构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  物理计划 (PhysicalOperator Tree)                               │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第一章：执行模型概述                              │           │
│  │ Push-based Model、Pipeline、向量化执行           │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第二章：Pipeline 与调度                           │           │
│  │ MetaPipeline、PipelineExecutor、Event System    │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第三章：向量化执行基础                            │           │
│  │ Vector、DataChunk、SelectionVector             │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第四章：表达式执行器                              │           │
│  │ ExpressionExecutor、ExpressionState            │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第五章：扫描与过滤算子                            │           │
│  │ TableScan、Filter、Projection                  │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第六章：Join 算子实现                             │           │
│  │ HashJoin、MergeJoin、NestedLoopJoin            │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第七章：聚合与分组算子                            │           │
│  │ HashAggregate、PerfectHashAggregate、Window    │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第八章：排序与 TopN 算子                          │           │
│  │ RadixSort、MergeSort、TopN                     │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第九章：并行执行                                  │           │
│  │ TaskScheduler、Thread Pool、Parallelism        │           │
│  └────────────────────┬────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────┐           │
│  │ 第十章：执行时内存管理                            │           │
│  │ OperatorState、Spilling、ProgressBar           │           │
│  └─────────────────────────────────────────────────┘           │
│                       │                                         │
│                       ▼                                         │
│  查询结果                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第一章：执行模型概述

### 1.1 Push-based vs Pull-based 执行模型
- 传统 Volcano 模型 (Pull)
- DuckDB 的 Push-based 模型
- 模型对比与优劣分析

### 1.2 向量化执行
- 为什么选择向量化
- SIMD 优化
- 批量处理 vs 逐行处理

### 1.3 Pipeline 概念
- Pipeline 定义
- Pipeline Breaker (阻塞算子)
- Source、Operator、Sink 角色

### 1.4 执行入口
- `Executor` 类设计
- `ExecuteTask()` 方法
- 结果收集 (`QueryResult`)

### 1.5 PhysicalOperator 接口
- `GetData()` - Source 接口
- `Execute()` - Operator 接口
- `Sink()` / `Combine()` / `Finalize()` - Sink 接口

---

## 第二章：Pipeline 与调度

### 2.1 Pipeline 结构
- `Pipeline` 类设计
- Source → Operators → Sink 链
- Pipeline 依赖关系

### 2.2 MetaPipeline
- `MetaPipeline` 设计
- Pipeline 分组与依赖
- 递归 Pipeline 构建

### 2.3 PipelineExecutor
- 单 Pipeline 执行逻辑
- Operator State 管理
- 中间结果处理

### 2.4 Event System
- `PipelineEvent` 基类
- `PipelineInitializeEvent`
- `PipelineFinishEvent`
- 事件触发与回调

### 2.5 任务调度
- `TaskScheduler` 设计
- 工作窃取 (Work Stealing)
- 任务优先级

### 2.6 进度追踪
- `ProgressBar` 集成
- 行数估计
- 进度百分比计算

---

## 第三章：向量化执行基础

### 3.1 Vector 类
- `Vector` 设计
- `VectorType` (FLAT, CONSTANT, DICTIONARY, SEQUENCE)
- 数据存储 (`data`, `validity`)

### 3.2 DataChunk
- `DataChunk` 设计
- 列式存储
- Chunk 大小 (STANDARD_VECTOR_SIZE = 2048)

### 3.3 SelectionVector
- 选择向量设计
- 稀疏表示
- Filter 优化

### 3.4 ValidityMask
- NULL 值处理
- 位图压缩
- 批量操作

### 3.5 UnifiedVectorFormat
- 统一向量格式
- 类型转换
- 高效访问

### 3.6 Vector Operations
- 批量操作函数
- SIMD 实现
- 模板化设计

---

## 第四章：表达式执行器

### 4.1 ExpressionExecutor 架构
- 表达式执行流程
- 递归执行 vs 扁平化执行
- 表达式编译

### 4.2 ExpressionState
- 状态管理
- 中间结果缓存
- 子表达式状态

### 4.3 标量函数执行
- `ScalarFunction` 执行
- 参数处理
- 返回值处理

### 4.4 聚合函数执行
- `AggregateFunction` 接口
- State 初始化与合并
- Finalize 处理

### 4.5 条件表达式
- CASE WHEN 执行
- 短路求值
- NULL 处理

### 4.6 Cast 表达式
- 类型转换执行
- 隐式转换
- 转换失败处理

---

## 第五章：扫描与过滤算子

### 5.1 PhysicalTableScan
- TableScan 实现
- 列裁剪
- 分区扫描

### 5.2 ColumnDataScan
- 临时数据扫描
- 物化中间结果
- 并行扫描

### 5.3 PhysicalFilter
- 过滤执行
- 选择向量更新
- 谓词优化

### 5.4 PhysicalProjection
- 投影执行
- 表达式计算
- 列重排

### 5.5 Zonemap 过滤
- 统计信息利用
- Segment 跳过
- 动态过滤

### 5.6 索引扫描
- IndexScan 实现
- ART 索引集成
- 范围扫描优化

---

## 第六章：Join 算子实现

### 6.1 Join 算子概述
- Join 类型 (Inner, Left, Right, Full, Semi, Anti)
- 物理实现选择
- 算子状态管理

### 6.2 PhysicalHashJoin
- Build 阶段 (Hash Table 构建)
- Probe 阶段 (匹配查找)
- 并行 Build/Probe
- Mark Join 支持

### 6.3 HashJoinHashTable
- Hash Table 设计
- 冲突处理 (链式)
- 内存布局优化

### 6.4 PhysicalMergeJoin (Piecewise Merge Join)
- Sort-Merge 原理
- 有序数据优化
- 范围条件处理

### 6.5 PhysicalNestedLoopJoin
- 嵌套循环实现
- 小表优化
- Cross Product 处理

### 6.6 动态 Join Filter
- Bloom Filter 生成
- Filter 下推
- 运行时过滤

---

## 第七章：聚合与分组算子

### 7.1 聚合算子概述
- 分组聚合 vs 标量聚合
- 聚合函数状态
- 多阶段聚合

### 7.2 PhysicalHashAggregate
- Hash Table 分组
- State 管理
- 并行聚合与合并

### 7.3 PhysicalPerfectHashAggregate
- 完美哈希条件
- 密集数组实现
- 性能优势

### 7.4 PhysicalStreamingAggregate
- 有序数据聚合
- 流式处理
- 内存效率

### 7.5 PhysicalUngroupedAggregate
- 无分组聚合
- 单一结果
- 早期终止

### 7.6 PhysicalWindow
- 窗口函数执行
- Frame 计算
- 分区与排序

### 7.7 Distinct 聚合
- DISTINCT 处理
- Hash Set 实现
- 内存优化

---

## 第八章：排序与 TopN 算子

### 8.1 PhysicalOrder
- 排序算法选择
- Radix Sort 实现
- 外部排序 (Spilling)

### 8.2 RowLayout
- 行格式设计
- 比较键编码
- Payload 管理

### 8.3 并行排序
- 分区排序
- 归并合并
- 负载均衡

### 8.4 PhysicalTopN
- TopN 优化
- 堆实现
- ORDER BY + LIMIT 融合

### 8.5 外部排序
- 内存不足处理
- 临时文件管理
- 多路归并

---

## 第九章：并行执行

### 9.1 TaskScheduler
- 任务队列管理
- 工作线程池
- 任务提交与执行

### 9.2 并行度控制
- `max_threads` 设置
- 动态并行度
- 资源竞争处理

### 9.3 Pipeline 并行
- Source 并行 (数据分区)
- Operator 并行 (任务拆分)
- Sink 并行与合并

### 9.4 Morsel-Driven Parallelism
- Morsel 概念
- 动态任务分配
- 负载均衡

### 9.5 同步与协调
- 原子操作
- 锁策略
- Pipeline 依赖等待

### 9.6 中断与取消
- 查询取消机制
- 超时处理
- 资源清理

---

## 第十章：执行时内存管理

### 10.1 OperatorState
- 算子状态设计
- 生命周期管理
- 状态共享

### 10.2 LocalState vs GlobalState
- 局部状态 (每线程)
- 全局状态 (共享)
- 状态合并

### 10.3 内存预算
- 内存限制配置
- 算子内存分配
- 内存超限处理

### 10.4 Spilling 机制
- 何时 Spill
- 临时文件管理
- 数据恢复

### 10.5 ColumnDataCollection
- 中间结果存储
- 分区管理
- 迭代访问

### 10.6 进度与监控
- `QueryProfiler` 集成
- 执行统计收集
- EXPLAIN ANALYZE 输出

---

## 附录

### A. 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| Executor | `src/execution/executor.cpp` |
| Pipeline | `src/execution/pipeline*.cpp` |
| Vector | `src/common/vector_operations/` |
| DataChunk | `src/common/types/data_chunk.cpp` |
| ExpressionExecutor | `src/execution/expression_executor/` |
| Physical Operators | `src/execution/operator/` |
| TaskScheduler | `src/parallel/task_scheduler.cpp` |

### B. PhysicalOperator 类型速查

| 类别 | 算子 | Source/Sink |
|------|------|-------------|
| 扫描 | PhysicalTableScan | Source |
| 过滤 | PhysicalFilter | Operator |
| 投影 | PhysicalProjection | Operator |
| 连接 | PhysicalHashJoin | Sink + Source |
| 聚合 | PhysicalHashAggregate | Sink + Source |
| 排序 | PhysicalOrder | Sink + Source |
| 限制 | PhysicalLimit | Operator |
| 结果 | PhysicalResult | Sink |

### C. 执行流程示例

```sql
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
GROUP BY c.name
ORDER BY SUM(o.amount) DESC
LIMIT 10;
```

Pipeline 分解：
```
Pipeline 1: TableScan(customers) → Filter → [HashJoin Build]
Pipeline 2: TableScan(orders) → Filter → HashJoin Probe → [HashAggregate]
Pipeline 3: HashAggregate Scan → [Order]
Pipeline 4: Order Scan → TopN → Result
```

执行顺序：
1. Pipeline 1 执行完成（Build 端准备就绪）
2. Pipeline 2 执行完成（Probe + 聚合）
3. Pipeline 3 执行完成（排序）
4. Pipeline 4 执行完成（输出结果）

---

## 写作计划

| 章节 | 预计篇幅 | 核心内容 |
|------|----------|----------|
| 第一章 | ~30KB | 执行模型、Pipeline 概念、向量化基础 |
| 第二章 | ~35KB | Pipeline 调度、事件系统、任务管理 |
| 第三章 | ~40KB | Vector、DataChunk、SelectionVector |
| 第四章 | ~35KB | ExpressionExecutor、函数执行 |
| 第五章 | ~30KB | TableScan、Filter、Projection |
| 第六章 | ~45KB | HashJoin、MergeJoin、动态过滤 |
| 第七章 | ~40KB | HashAggregate、Window 函数 |
| 第八章 | ~35KB | 排序算法、外部排序、TopN |
| 第九章 | ~35KB | 并行执行、TaskScheduler |
| 第十章 | ~30KB | 内存管理、Spilling、监控 |

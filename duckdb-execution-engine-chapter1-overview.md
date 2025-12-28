# DuckDB 执行引擎深度解析：第一章 执行模型概述

## 1.1 执行模型演进

### 1.1.1 传统 Volcano 模型 (Pull-based)

传统数据库系统采用 **Volcano 迭代器模型**（也称 Pull 模型）：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Volcano 模型 (Pull-based)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  应用层                                                          │
│     │                                                           │
│     │ next()                                                    │
│     ▼                                                           │
│  ┌─────────────────┐                                            │
│  │    Projection   │  ←── next() 调用子节点                      │
│  └────────┬────────┘                                            │
│           │ next()                                              │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │     Filter      │  ←── next() 调用子节点                      │
│  └────────┬────────┘                                            │
│           │ next()                                              │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │   Table Scan    │  ←── 从存储读取一行数据                      │
│  └─────────────────┘                                            │
│                                                                 │
│  特点：                                                          │
│  - 每次 next() 返回一行数据                                       │
│  - 函数调用开销大（每行一次虚函数调用）                            │
│  - 代码简洁，易于实现                                             │
│  - 无法有效利用 CPU 缓存和 SIMD                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1.2 DuckDB 的 Push-based 向量化执行

DuckDB 采用 **Push-based 向量化执行模型**：

```
┌─────────────────────────────────────────────────────────────────┐
│                DuckDB Push-based 向量化执行模型                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐                                            │
│  │   Table Scan    │  ─── Source: 产生 DataChunk (2048 行)      │
│  └────────┬────────┘                                            │
│           │ DataChunk (push)                                    │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │     Filter      │  ─── Operator: 处理 DataChunk              │
│  └────────┬────────┘                                            │
│           │ DataChunk (push)                                    │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │   Projection    │  ─── Operator: 处理 DataChunk              │
│  └────────┬────────┘                                            │
│           │ DataChunk (push)                                    │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  Result Sink    │  ─── Sink: 收集最终结果                     │
│  └─────────────────┘                                            │
│                                                                 │
│  特点：                                                          │
│  - 批量处理 (STANDARD_VECTOR_SIZE = 2048)                        │
│  - 数据主动推送到下游算子                                         │
│  - 利用 SIMD 向量化指令                                          │
│  - 更好的 CPU 缓存利用率                                          │
│  - 减少虚函数调用开销                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1.3 模型对比

| 特性 | Volcano (Pull) | DuckDB (Push) |
|------|----------------|---------------|
| 数据流向 | 自顶向下拉取 | 自底向上推送 |
| 处理单位 | 单行 | 向量 (2048 行) |
| 函数调用 | 每行一次 | 每批一次 |
| SIMD 优化 | 困难 | 自然支持 |
| 缓存效率 | 低 | 高 |
| 实现复杂度 | 简单 | 中等 |
| 阻塞处理 | 简单 | 需要 Pipeline 切分 |

---

## 1.2 Pipeline 概念

### 1.2.1 什么是 Pipeline

Pipeline 是 DuckDB 执行引擎的核心抽象。一个 Pipeline 代表一段**连续执行的算子链**，数据在其中无阻塞地流动：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Pipeline 结构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐    ┌───────────┐    ┌───────────┐    ┌─────────┐  │
│  │ Source  │───▶│ Operator  │───▶│ Operator  │───▶│  Sink   │  │
│  │(GetData)│    │ (Execute) │    │ (Execute) │    │ (Sink)  │  │
│  └─────────┘    └───────────┘    └───────────┘    └─────────┘  │
│                                                                 │
│  Source:    数据源，产生 DataChunk                               │
│  Operator:  中间算子，处理并传递 DataChunk                        │
│  Sink:      数据汇，消费 DataChunk 并可能阻塞                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2.2 Pipeline Breaker (阻塞算子)

某些算子需要收集所有输入后才能产生输出，称为 **Pipeline Breaker**：

```cpp
// 常见的 Pipeline Breaker:
// - HashJoin (Build 端)
// - HashAggregate
// - Order
// - Window
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  Pipeline Breaker 示例                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SELECT * FROM t1 JOIN t2 ON t1.id = t2.id ORDER BY t1.name     │
│                                                                 │
│  Pipeline 1: TableScan(t2) ──────────────▶ [HashJoin Build]     │
│                                                  │              │
│                                                  ▼              │
│  Pipeline 2: TableScan(t1) ──▶ HashJoin Probe ──▶ [Order]       │
│                                                       │         │
│                                                       ▼         │
│  Pipeline 3: Order Scan ─────────────────────────▶ Result       │
│                                                                 │
│  [方括号] 表示 Pipeline Breaker / Sink                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2.3 Pipeline 数据结构

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class Pipeline : public enable_shared_from_this<Pipeline> {
private:
    //! 数据源算子
    optional_ptr<PhysicalOperator> source;
    //! 中间算子链
    vector<reference<PhysicalOperator>> operators;
    //! 数据汇算子（Sink / Pipeline Breaker）
    optional_ptr<PhysicalOperator> sink;

    //! 全局 Source 状态
    unique_ptr<GlobalSourceState> source_state;

    //! 父 Pipeline（依赖当前 Pipeline 完成）
    vector<weak_ptr<Pipeline>> parents;
    //! 依赖的 Pipeline
    vector<weak_ptr<Pipeline>> dependencies;

    //! 批次索引管理
    idx_t base_batch_index = 0;
    mutex batch_lock;
    multiset<idx_t> batch_indexes;

public:
    void Ready();           // 准备执行
    void Reset();           // 重置状态
    void Schedule(shared_ptr<Event> &event);  // 调度执行
};
```

---

## 1.3 PhysicalOperator 接口

### 1.3.1 PhysicalOperator 基类

`PhysicalOperator` 是所有物理算子的基类：

```cpp
// src/include/duckdb/execution/physical_operator.hpp
class PhysicalOperator {
public:
    //! 子算子列表
    ArenaLinkedList<reference<PhysicalOperator>> children;
    //! 算子类型
    PhysicalOperatorType type;
    //! 输出类型
    vector<LogicalType> types;
    //! 估计基数
    idx_t estimated_cardinality;

    //! 全局 Sink 状态
    unique_ptr<GlobalSinkState> sink_state;
    //! 全局 Operator 状态
    unique_ptr<GlobalOperatorState> op_state;

    // ========== 三类接口 ==========

    // 1. Operator 接口 (中间算子)
    virtual OperatorResultType Execute(ExecutionContext &context,
                                        DataChunk &input,
                                        DataChunk &chunk,
                                        GlobalOperatorState &gstate,
                                        OperatorState &state) const;

    // 2. Source 接口 (数据源)
    virtual SourceResultType GetData(ExecutionContext &context,
                                      DataChunk &chunk,
                                      OperatorSourceInput &input) const;

    // 3. Sink 接口 (数据汇)
    virtual SinkResultType Sink(ExecutionContext &context,
                                 DataChunk &chunk,
                                 OperatorSinkInput &input) const;
    virtual SinkCombineResultType Combine(ExecutionContext &context,
                                           OperatorSinkCombineInput &input) const;
    virtual SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                                       ClientContext &context,
                                       OperatorSinkFinalizeInput &input) const;
};
```

### 1.3.2 三类算子角色

```
┌─────────────────────────────────────────────────────────────────┐
│                    PhysicalOperator 角色分类                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      Source                              │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  - 产生数据的算子（Pipeline 起点）                        │   │
│  │  - 实现 GetData() 方法                                   │   │
│  │  - 例如: TableScan, ColumnDataScan                       │   │
│  │  - IsSource() 返回 true                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Operator                             │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  - 处理数据的中间算子                                     │   │
│  │  - 实现 Execute() 方法                                   │   │
│  │  - 例如: Filter, Projection                              │   │
│  │  - 既不是 Source 也不是 Sink                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                       Sink                               │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  - 消费数据的算子（Pipeline 终点 / Breaker）              │   │
│  │  - 实现 Sink() / Combine() / Finalize() 方法            │   │
│  │  - 例如: HashJoin (Build), HashAggregate, Order         │   │
│  │  - IsSink() 返回 true                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3.3 算子结果类型

```cpp
// src/include/duckdb/common/enums/operator_result_type.hpp

// Operator (Execute) 返回值
enum class OperatorResultType : uint8_t {
    NEED_MORE_INPUT,    // 需要更多输入
    HAVE_MORE_OUTPUT,   // 还有更多输出（用相同输入再次调用）
    FINISHED,           // Pipeline 完成
    BLOCKED             // 阻塞（异步 I/O）
};

// Source (GetData) 返回值
enum class SourceResultType : uint8_t {
    HAVE_MORE_OUTPUT,   // 还有更多数据
    FINISHED,           // 数据源耗尽
    BLOCKED             // 阻塞
};

// Sink (Sink) 返回值
enum class SinkResultType : uint8_t {
    NEED_MORE_INPUT,    // 需要更多输入
    FINISHED,           // Sink 完成
    BLOCKED             // 阻塞
};

// Sink (Finalize) 返回值
enum class SinkFinalizeType : uint8_t {
    READY,              // 准备就绪，可以作为 Source
    NO_OUTPUT_POSSIBLE, // 无输出（可跳过后续 Pipeline）
    BLOCKED             // 阻塞
};
```

---

## 1.4 Executor 执行入口

### 1.4.1 Executor 类设计

```cpp
// src/include/duckdb/execution/executor.hpp
class Executor {
public:
    ClientContext &context;

    //! 初始化执行器
    void Initialize(PhysicalOperator &physical_plan);

    //! 执行一个任务
    PendingExecutionResult ExecuteTask(bool dry_run = false);

    //! 获取查询结果
    unique_ptr<QueryResult> GetResult();

    //! 返回所有 Pipeline 的进度
    idx_t GetPipelinesProgress(ProgressData &progress);

private:
    //! 物理计划根节点
    optional_ptr<PhysicalOperator> physical_plan;

    //! 所有 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! 根 Pipeline
    vector<shared_ptr<Pipeline>> root_pipelines;

    //! 根 Pipeline 执行器
    unique_ptr<PipelineExecutor> root_executor;

    //! 事件列表
    vector<shared_ptr<Event>> events;

    //! 已完成的 Pipeline 数量
    atomic<idx_t> completed_pipelines;
    //! 总 Pipeline 数量
    idx_t total_pipelines;
};
```

### 1.4.2 执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Executor 执行流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Initialize(physical_plan)                                   │
│     │                                                           │
│     ├─▶ 构建 MetaPipeline 树                                    │
│     ├─▶ 收集所有 Pipeline                                       │
│     └─▶ 创建 Pipeline 事件和依赖关系                             │
│                                                                 │
│  2. ScheduleEvents()                                            │
│     │                                                           │
│     ├─▶ 创建 PipelineInitializeEvent                           │
│     ├─▶ 创建 PipelineEvent                                     │
│     ├─▶ 创建 PipelinePrepareFinishEvent                        │
│     ├─▶ 创建 PipelineFinishEvent                               │
│     └─▶ 创建 PipelineCompleteEvent                             │
│                                                                 │
│  3. ExecuteTask() [循环]                                        │
│     │                                                           │
│     ├─▶ 从 TaskScheduler 获取任务                               │
│     ├─▶ 执行 PipelineTask                                      │
│     └─▶ 处理事件完成和依赖触发                                   │
│                                                                 │
│  4. GetResult()                                                 │
│     │                                                           │
│     └─▶ 从 ResultCollector 获取最终结果                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4.3 Pipeline 事件链

```cpp
// src/parallel/executor.cpp
struct PipelineEventStack {
    Event &pipeline_initialize_event;   // 初始化事件
    Event &pipeline_event;              // 执行事件
    Event &pipeline_prepare_finish_event; // 准备完成事件
    Event &pipeline_finish_event;       // 完成事件
    Event &pipeline_complete_event;     // 完全完成事件
};

// 依赖关系:
// initialize -> event -> prepare_finish -> finish -> complete
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pipeline 事件生命周期                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PipelineInitializeEvent                                        │
│  ├── 初始化 Source 状态                                         │
│  └── 初始化 Sink 状态                                           │
│           │                                                     │
│           ▼                                                     │
│  PipelineEvent                                                  │
│  ├── 创建 PipelineTask                                          │
│  └── 调度到 TaskScheduler                                       │
│           │                                                     │
│           ▼                                                     │
│  PipelinePrepareFinishEvent                                     │
│  └── 调用 Sink::PrepareFinalize()                               │
│           │                                                     │
│           ▼                                                     │
│  PipelineFinishEvent                                            │
│  └── 调用 Sink::Finalize()                                      │
│           │                                                     │
│           ▼                                                     │
│  PipelineCompleteEvent                                          │
│  └── 触发依赖的 Pipeline                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.5 PipelineExecutor 执行逻辑

### 1.5.1 PipelineExecutor 结构

```cpp
// src/include/duckdb/parallel/pipeline_executor.hpp
class PipelineExecutor {
private:
    Pipeline &pipeline;
    ThreadContext thread;
    ExecutionContext context;

    //! 中间 Chunk（每个 Operator 一个）
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    //! 中间状态（每个 Operator 一个）
    vector<unique_ptr<OperatorState>> intermediate_states;

    //! 本地 Source 状态
    unique_ptr<LocalSourceState> local_source_state;
    //! 本地 Sink 状态
    unique_ptr<LocalSinkState> local_sink_state;

    //! 最终 Chunk（送入 Sink）
    DataChunk final_chunk;

    //! 尚未处理完的 Operator 栈
    stack<idx_t> in_process_operators;

public:
    //! 执行 Pipeline
    PipelineExecuteResult Execute();
    PipelineExecuteResult Execute(idx_t max_chunks);

    //! 完成处理
    PipelineExecuteResult PushFinalize();
};
```

### 1.5.2 Execute 核心逻辑

```cpp
// src/parallel/pipeline_executor.cpp
PipelineExecuteResult PipelineExecutor::Execute(idx_t max_chunks) {
    auto &source_chunk = pipeline.operators.empty() ? final_chunk
                                                     : *intermediate_chunks[0];
    ExecutionBudget chunk_budget(max_chunks);

    do {
        if (context.client.interrupted) {
            throw InterruptException();
        }

        if (exhausted_pipeline && done_flushing) {
            break;  // 执行完成
        }

        if (remaining_sink_chunk) {
            // Sink 被中断，重试
            result = ExecutePushInternal(final_chunk, chunk_budget);
            remaining_sink_chunk = false;
        }
        else if (!in_process_operators.empty() && !started_flushing) {
            // Operator 返回 HAVE_MORE_OUTPUT，继续处理
            result = ExecutePushInternal(source_chunk, chunk_budget);
        }
        else if (exhausted_pipeline && !done_flushing) {
            // Source 耗尽，刷新缓存
            TryFlushCachingOperators(chunk_budget);
        }
        else {
            // 正常路径：从 Source 获取数据
            source_chunk.Reset();
            auto source_result = FetchFromSource(source_chunk);

            if (source_result == SourceResultType::BLOCKED) {
                return PipelineExecuteResult::INTERRUPTED;
            }
            if (source_result == SourceResultType::FINISHED) {
                exhausted_pipeline = true;
            }

            // 推送数据通过 Pipeline
            result = ExecutePushInternal(source_chunk, chunk_budget);
        }
    } while (!chunk_budget.IsDepleted());

    return PipelineExecuteResult::NOT_FINISHED;
}
```

### 1.5.3 数据流动图

```
┌─────────────────────────────────────────────────────────────────┐
│                 PipelineExecutor 数据流动                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FetchFromSource()                                           │
│     │                                                           │
│     │  source_chunk                                             │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Source                               │   │
│  │               source->GetData()                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│     │                                                           │
│     │  intermediate_chunks[0]                                   │
│     ▼                                                           │
│  2. ExecutePushInternal(chunk, budget, idx=0)                   │
│     │                                                           │
│     │  for each operator:                                       │
│     │    input = intermediate_chunks[i]                         │
│     │    output = intermediate_chunks[i+1]                      │
│     │    operator->Execute(input, output)                       │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Operator 1: Execute()                       │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │              Operator 2: Execute()                       │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │              Operator N: Execute()                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│     │                                                           │
│     │  final_chunk                                              │
│     ▼                                                           │
│  3. Sink(final_chunk)                                           │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                       Sink                               │   │
│  │                 sink->Sink(chunk)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.6 MetaPipeline 层次结构

### 1.6.1 MetaPipeline 概念

`MetaPipeline` 表示一组共享相同 Sink 的 Pipeline：

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp
enum class MetaPipelineType : uint8_t {
    REGULAR = 0,    // 常规 MetaPipeline
    JOIN_BUILD = 1  // Join Build 端
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
private:
    Executor &executor;
    PipelineBuildState &state;

    //! 共享的 Sink 算子
    optional_ptr<PhysicalOperator> sink;
    //! MetaPipeline 类型
    MetaPipelineType type;

    //! 所有共享该 Sink 的 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! Pipeline 依赖关系
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
    //! 子 MetaPipeline
    vector<shared_ptr<MetaPipeline>> children;

public:
    //! 构建 Pipeline 结构
    void Build(PhysicalOperator &op);

    //! 创建 Pipeline
    Pipeline &CreatePipeline();
    Pipeline &CreateUnionPipeline(Pipeline &current, bool order_matters);

    //! 创建子 MetaPipeline（用于 Join Build 等）
    MetaPipeline &CreateChildMetaPipeline(Pipeline &current, PhysicalOperator &op,
                                           MetaPipelineType type);
};
```

### 1.6.2 Pipeline 构建规则

```
┌─────────────────────────────────────────────────────────────────┐
│                   MetaPipeline 构建规则                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  规则 1: Join 的 Build 端优先构建                                │
│  ─────────────────────────────────                              │
│  - 创建子 MetaPipeline 处理 Build 端                            │
│  - 当前 Pipeline 依赖于 Build MetaPipeline                      │
│  - Union Pipeline 自动继承该依赖                                 │
│                                                                 │
│  规则 2: 子 Pipeline 最后构建                                    │
│  ─────────────────────────────────                              │
│  - 例如: FULL OUTER JOIN 的 HT 扫描                             │
│  - 子 Pipeline 依赖于：                                         │
│    * 当前 streaming Pipeline                                    │
│    * 之后添加到 MetaPipeline 的所有 Pipeline                    │
│                                                                 │
│  规则 3: Union 产生多个 Pipeline                                 │
│  ─────────────────────────────────                              │
│  - 每个 Union 子树创建独立 Pipeline                             │
│  - 共享相同的 Sink                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.6.3 HashJoin Pipeline 构建示例

```sql
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.x > 10
```

```
┌─────────────────────────────────────────────────────────────────┐
│                HashJoin Pipeline 构建                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  物理计划:                                                       │
│                                                                 │
│       HashJoin                                                  │
│       /      \                                                  │
│   Filter    TableScan(t2)                                       │
│     |            ↓                                              │
│  TableScan(t1)  Build Side                                      │
│     ↓                                                           │
│  Probe Side                                                     │
│                                                                 │
│  Pipeline 构建:                                                  │
│                                                                 │
│  MetaPipeline (sink = Result)                                   │
│  │                                                              │
│  ├── Pipeline 1: TableScan(t1) → Filter → HashJoin Probe → Result│
│  │                                                              │
│  └── Child MetaPipeline (sink = HashJoin Build, type = JOIN_BUILD)│
│      │                                                          │
│      └── Pipeline 2: TableScan(t2) → HashJoin Build             │
│                                                                 │
│  依赖关系: Pipeline 1 depends on Pipeline 2                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.7 算子状态管理

### 1.7.1 状态层次结构

```cpp
// src/include/duckdb/execution/physical_operator_states.hpp

// 1. Operator 状态（中间算子）
class OperatorState {
public:
    virtual void Finalize(const PhysicalOperator &op,
                          ExecutionContext &context);
};

class GlobalOperatorState {
public:
    virtual idx_t MaxThreads(idx_t source_max_threads);
};

// 2. Source 状态
class GlobalSourceState : public StateWithBlockableTasks {
public:
    virtual idx_t MaxThreads();  // 最大并行度
};

class LocalSourceState {
    // 线程本地状态
};

// 3. Sink 状态
class GlobalSinkState : public StateWithBlockableTasks {
public:
    SinkFinalizeType state;
    virtual idx_t MaxThreads(idx_t source_max_threads);
};

class LocalSinkState {
    SourcePartitionInfo partition_info;  // 分区信息
};
```

### 1.7.2 状态生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                     状态生命周期                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GlobalState（全局状态）                                         │
│  ─────────────────────────                                      │
│  - Pipeline 级别共享                                            │
│  - 由第一个 PipelineExecutor 创建                               │
│  - 所有线程共享访问                                              │
│  - 在 Finalize 时清理                                           │
│                                                                 │
│  LocalState（本地状态）                                          │
│  ─────────────────────────                                      │
│  - PipelineExecutor / 线程级别                                  │
│  - 每个 PipelineExecutor 独立创建                               │
│  - 无需同步访问                                                  │
│  - 在 Combine 时合并到 GlobalState                              │
│                                                                 │
│  生命周期:                                                       │
│                                                                 │
│  GetGlobalState() ──▶ GlobalState 创建                          │
│        │                                                        │
│        ▼                                                        │
│  GetLocalState()  ──▶ LocalState 创建 (每线程)                  │
│        │                                                        │
│        ▼                                                        │
│  Execute/Sink()   ──▶ 使用 Local + Global 状态                  │
│        │                                                        │
│        ▼                                                        │
│  Combine()        ──▶ LocalState → GlobalState                  │
│        │                                                        │
│        ▼                                                        │
│  Finalize()       ──▶ GlobalState 完成处理                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.8 PhysicalOperator 类型

### 1.8.1 完整类型列表

```cpp
// src/include/duckdb/common/enums/physical_operator_type.hpp
enum class PhysicalOperatorType : uint8_t {
    INVALID,

    // 排序和限制
    ORDER_BY, LIMIT, STREAMING_LIMIT, LIMIT_PERCENT, TOP_N,

    // 窗口
    WINDOW, STREAMING_WINDOW,

    // 聚合
    UNGROUPED_AGGREGATE, HASH_GROUP_BY, PERFECT_HASH_GROUP_BY,
    PARTITIONED_AGGREGATE,

    // 基础算子
    FILTER, PROJECTION, UNNEST, PIVOT,

    // 扫描
    TABLE_SCAN, DUMMY_SCAN, COLUMN_DATA_SCAN, CHUNK_SCAN,
    RECURSIVE_CTE_SCAN, CTE_SCAN, DELIM_SCAN, EXPRESSION_SCAN,
    POSITIONAL_SCAN,

    // Join
    BLOCKWISE_NL_JOIN, NESTED_LOOP_JOIN, HASH_JOIN, CROSS_PRODUCT,
    PIECEWISE_MERGE_JOIN, IE_JOIN, LEFT_DELIM_JOIN, RIGHT_DELIM_JOIN,
    POSITIONAL_JOIN, ASOF_JOIN,

    // 集合操作
    UNION, RECURSIVE_CTE, CTE,

    // DML
    INSERT, BATCH_INSERT, DELETE_OPERATOR, UPDATE, MERGE_INTO,

    // DDL
    CREATE_TABLE, CREATE_TABLE_AS, CREATE_INDEX, ALTER,
    CREATE_SEQUENCE, CREATE_VIEW, CREATE_SCHEMA, CREATE_MACRO,
    DROP, CREATE_TYPE, ATTACH, DETACH,

    // 辅助
    EXPLAIN, EXPLAIN_ANALYZE, EMPTY_RESULT, EXECUTE, PREPARE,
    VACUUM, EXPORT, SET, LOAD, RESULT_COLLECTOR, ...
};
```

### 1.8.2 算子角色分类

| 类别 | 算子 | Source | Operator | Sink |
|------|------|--------|----------|------|
| 扫描 | TABLE_SCAN | ✓ | | |
| 扫描 | COLUMN_DATA_SCAN | ✓ | | |
| 过滤 | FILTER | | ✓ | |
| 投影 | PROJECTION | | ✓ | |
| 限制 | LIMIT | | ✓ | |
| Join | HASH_JOIN | ✓(Probe) | | ✓(Build) |
| 聚合 | HASH_GROUP_BY | ✓ | | ✓ |
| 排序 | ORDER_BY | ✓ | | ✓ |
| 窗口 | WINDOW | ✓ | | ✓ |
| TopN | TOP_N | | | ✓ |
| 结果 | RESULT_COLLECTOR | | | ✓ |

---

## 1.9 向量化执行基础

### 1.9.1 STANDARD_VECTOR_SIZE

DuckDB 使用固定的向量大小进行批处理：

```cpp
// 默认向量大小: 2048 行
#define STANDARD_VECTOR_SIZE 2048
```

选择 2048 的原因：
- **L1 缓存优化**：2048 × 8 bytes = 16KB，适合 L1 数据缓存
- **SIMD 对齐**：2048 是常见 SIMD 宽度的整数倍
- **平衡粒度**：足够大以摊销函数调用开销，又不会太大导致延迟

### 1.9.2 向量化执行优势

```
┌─────────────────────────────────────────────────────────────────┐
│                    向量化执行优势                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 减少函数调用开销                                             │
│     ─────────────────────                                       │
│     Row-at-a-time:  N 行 = N 次函数调用                         │
│     Vectorized:     N 行 = 1 次函数调用 (处理 2048 行)          │
│                                                                 │
│  2. SIMD 并行                                                   │
│     ─────────────────────                                       │
│     for (i = 0; i < N; i += 8) {                               │
│         __m256i a = _mm256_loadu_si256(input + i);             │
│         __m256i b = _mm256_loadu_si256(other + i);             │
│         __m256i c = _mm256_add_epi32(a, b);                    │
│         _mm256_storeu_si256(output + i, c);                    │
│     }                                                           │
│                                                                 │
│  3. 缓存效率                                                    │
│     ─────────────────────                                       │
│     - 数据在 L1/L2 缓存中保持热                                 │
│     - 减少内存访问延迟                                          │
│     - 预取更有效                                                │
│                                                                 │
│  4. 分支预测                                                    │
│     ─────────────────────                                       │
│     - 批量处理减少条件分支                                      │
│     - 使用 SelectionVector 避免分支                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.10 源文件索引

| 组件 | 文件路径 |
|------|----------|
| Executor | `src/include/duckdb/execution/executor.hpp` |
| Executor 实现 | `src/parallel/executor.cpp` |
| PhysicalOperator | `src/include/duckdb/execution/physical_operator.hpp` |
| 算子状态 | `src/include/duckdb/execution/physical_operator_states.hpp` |
| 算子类型枚举 | `src/include/duckdb/common/enums/physical_operator_type.hpp` |
| 结果类型枚举 | `src/include/duckdb/common/enums/operator_result_type.hpp` |
| Pipeline | `src/include/duckdb/parallel/pipeline.hpp` |
| PipelineExecutor | `src/include/duckdb/parallel/pipeline_executor.hpp` |
| PipelineExecutor 实现 | `src/parallel/pipeline_executor.cpp` |
| MetaPipeline | `src/include/duckdb/parallel/meta_pipeline.hpp` |
| Pipeline 事件 | `src/include/duckdb/parallel/pipeline_event.hpp` |

---

## 1.11 总结

DuckDB 执行引擎的核心特点：

```
┌─────────────────────────────────────────────────────────────────┐
│                   执行引擎核心特点                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Push-based 执行模型                                         │
│     - 数据从 Source 推送到 Sink                                 │
│     - 相比 Pull 模型更适合向量化                                 │
│                                                                 │
│  2. 向量化处理                                                   │
│     - 批量处理 2048 行                                          │
│     - 利用 SIMD 指令                                            │
│     - 优化 CPU 缓存                                             │
│                                                                 │
│  3. Pipeline 抽象                                               │
│     - Source → Operators → Sink                                │
│     - Pipeline Breaker 切分阻塞点                               │
│     - MetaPipeline 管理依赖                                     │
│                                                                 │
│  4. 三类算子角色                                                 │
│     - Source: GetData() 产生数据                                │
│     - Operator: Execute() 处理数据                              │
│     - Sink: Sink()/Combine()/Finalize() 消费数据               │
│                                                                 │
│  5. 事件驱动调度                                                 │
│     - Pipeline 事件链                                           │
│     - 依赖关系管理                                              │
│     - 支持异步 I/O                                              │
│                                                                 │
│  6. 状态分离                                                    │
│     - GlobalState: Pipeline 共享                                │
│     - LocalState: 线程本地                                      │
│     - Combine 合并本地状态                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

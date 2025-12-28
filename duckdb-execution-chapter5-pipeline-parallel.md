# DuckDB 执行引擎深度解析（五）：Pipeline 并行执行

## 引言

在前四章中，我们分析了 DuckDB 的执行模型、向量化数据结构、表达式执行器和物理算子。本章将深入探讨 **Pipeline 并行执行** 的设计与实现。

DuckDB 采用基于 Pipeline 的执行模型，结合任务调度系统，实现了高效的多核并行查询处理。本章将详细解析 Pipeline、PipelineExecutor、TaskScheduler 和 Event 系统的协作机制。

## 1. Pipeline 架构概览

### 1.1 Pipeline 的定义

Pipeline 是一组从 Source 到 Sink 的算子链，在物化点（如 Hash Join Build、Aggregation）处断开：

```cpp
// src/include/duckdb/parallel/pipeline.hpp

class Pipeline : public enable_shared_from_this<Pipeline> {
    friend class Executor;
    friend class PipelineExecutor;
    friend class MetaPipeline;

public:
    explicit Pipeline(Executor &execution_context);

    Executor &executor;

private:
    //! Pipeline 是否已就绪
    bool ready;
    //! Pipeline 是否已初始化
    atomic<bool> initialized;

    //! 数据源算子
    optional_ptr<PhysicalOperator> source;
    //! 中间处理算子链
    vector<reference<PhysicalOperator>> operators;
    //! 数据汇聚算子（Sink）
    optional_ptr<PhysicalOperator> sink;

    //! Source 的全局状态
    unique_ptr<GlobalSourceState> source_state;

    //! 父 Pipeline（依赖此 Pipeline 的 Pipeline）
    vector<weak_ptr<Pipeline>> parents;
    //! 依赖的 Pipeline
    vector<weak_ptr<Pipeline>> dependencies;

    //! 批次索引基数（用于多 Pipeline 共享 Source）
    idx_t base_batch_index = 0;
    //! 批次索引锁
    mutex batch_lock;
    //! 活跃的批次索引集合
    multiset<idx_t> batch_indexes;
};
```

### 1.2 Pipeline 结构示意

```
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline 结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐                                           │
│   │   Source    │ ← GetData() 产生数据                      │
│   │  (TableScan)│                                           │
│   └──────┬──────┘                                           │
│          │ DataChunk                                        │
│          ▼                                                  │
│   ┌─────────────┐                                           │
│   │  Operator 1 │ ← Execute() 转换数据                      │
│   │   (Filter)  │                                           │
│   └──────┬──────┘                                           │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────┐                                           │
│   │  Operator 2 │                                           │
│   │ (Projection)│                                           │
│   └──────┬──────┘                                           │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────┐                                           │
│   │    Sink     │ ← Sink() 消费数据                         │
│   │ (HashBuild) │                                           │
│   └─────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 Pipeline 依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                Pipeline 依赖示例                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  查询: SELECT * FROM A JOIN B ON a.id = b.id WHERE ...      │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │ Pipeline 1: Build Hash Table                │            │
│  │                                              │            │
│  │  TableScan(B) → HashJoin.Sink()             │            │
│  └─────────────────────┬───────────────────────┘            │
│                        │ depends                            │
│                        ▼                                    │
│  ┌─────────────────────────────────────────────┐            │
│  │ Pipeline 2: Probe Hash Table                │            │
│  │                                              │            │
│  │  TableScan(A) → Filter → HashJoin.Execute() │            │
│  │                          → Result           │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Pipeline 2 在 Pipeline 1 完成后才能开始                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. PipelineExecutor 执行引擎

### 2.1 类设计

`PipelineExecutor` 负责在单个线程上执行一个 Pipeline：

```cpp
// src/include/duckdb/parallel/pipeline_executor.hpp

enum class PipelineExecuteResult {
    FINISHED,       // 执行完成，Source 已耗尽
    NOT_FINISHED,   // 未完成，可立即再次调用
    INTERRUPTED     // 被中断，需等待回调后再调用
};

class PipelineExecutor {
public:
    PipelineExecutor(ClientContext &context, Pipeline &pipeline);

    //! 完整执行 Pipeline 直到 Source 耗尽
    PipelineExecuteResult Execute();
    //! 执行 Pipeline，最多处理 max_chunks 个数据块
    PipelineExecuteResult Execute(idx_t max_chunks);
    //! Source 耗尽后的最终处理
    PipelineExecuteResult PushFinalize();

private:
    //! 被执行的 Pipeline
    Pipeline &pipeline;
    //! 线程上下文
    ThreadContext thread;
    //! 执行上下文
    ExecutionContext context;

    //! 中间数据块（每个 Operator 一个）
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    //! 中间算子状态
    vector<unique_ptr<OperatorState>> intermediate_states;

    //! Source 本地状态
    unique_ptr<LocalSourceState> local_source_state;
    //! Sink 本地状态
    unique_ptr<LocalSinkState> local_sink_state;
    //! 中断状态
    InterruptState interrupt_state;

    //! 最终输出块
    DataChunk final_chunk;

    //! 还有输出的算子栈
    stack<idx_t> in_process_operators;

    //! Pipeline 是否已完成
    bool exhausted_pipeline = false;
    //! 是否开始刷新缓存
    bool started_flushing = false;
    //! 是否有残余的 Sink 数据块
    bool remaining_sink_chunk = false;
};
```

### 2.2 执行循环

```cpp
// src/parallel/pipeline_executor.cpp

PipelineExecuteResult PipelineExecutor::Execute(idx_t max_chunks) {
    D_ASSERT(pipeline.sink);
    auto &source_chunk = pipeline.operators.empty() ? final_chunk : *intermediate_chunks[0];
    ExecutionBudget chunk_budget(max_chunks);

    do {
        if (context.client.interrupted) {
            throw InterruptException();
        }

        OperatorResultType result;

        if (exhausted_pipeline && done_flushing && !remaining_sink_chunk &&
            !next_batch_blocked && in_process_operators.empty()) {
            // Pipeline 完全完成
            break;
        } else if (remaining_sink_chunk) {
            // Sink 被阻塞，重试 Sink
            result = ExecutePushInternal(final_chunk, chunk_budget);
            remaining_sink_chunk = false;
        } else if (!in_process_operators.empty() && !started_flushing) {
            // 有算子还有更多输出
            result = ExecutePushInternal(source_chunk, chunk_budget);
        } else if (exhausted_pipeline && !next_batch_blocked && !done_flushing) {
            // Source 耗尽，刷新缓存算子
            auto flush_completed = TryFlushCachingOperators(chunk_budget);
            if (flush_completed) {
                done_flushing = true;
                break;
            }
            // ...处理阻塞情况
        } else if (!exhausted_pipeline || next_batch_blocked) {
            // 正常路径：从 Source 获取数据
            source_chunk.Reset();
            SourceResultType source_result = FetchFromSource(source_chunk);

            if (source_result == SourceResultType::BLOCKED) {
                return PipelineExecuteResult::INTERRUPTED;
            }
            if (source_result == SourceResultType::FINISHED) {
                exhausted_pipeline = true;
            }

            // 处理分区批次（如需要）
            if (required_partition_info.AnyRequired()) {
                auto next_batch_result = NextBatch(source_chunk, ...);
                if (next_batch_result == SinkNextBatchType::BLOCKED) {
                    return PipelineExecuteResult::INTERRUPTED;
                }
            }

            // 推送数据通过 Pipeline
            result = ExecutePushInternal(source_chunk, chunk_budget);
        }

        // 处理 Sink 中断
        if (result == OperatorResultType::BLOCKED) {
            remaining_sink_chunk = true;
            return PipelineExecuteResult::INTERRUPTED;
        }

        if (result == OperatorResultType::FINISHED) {
            exhausted_pipeline = true;
            break;
        }
    } while (chunk_budget.Next());

    // 检查是否需要继续执行
    if ((!exhausted_pipeline || !done_flushing) && !IsFinished()) {
        return PipelineExecuteResult::NOT_FINISHED;
    }

    return PushFinalize();
}
```

### 2.3 数据推送流程

```cpp
// 将数据块推送通过整个 Pipeline
OperatorResultType PipelineExecutor::ExecutePushInternal(
    DataChunk &input, ExecutionBudget &chunk_budget, idx_t initial_idx) {

    D_ASSERT(pipeline.sink);
    if (input.size() == 0) {
        return OperatorResultType::NEED_MORE_INPUT;
    }

    OperatorResultType result = OperatorResultType::HAVE_MORE_OUTPUT;
    do {
        if (&input != &final_chunk) {
            final_chunk.Reset();
            // 通过所有中间算子
            result = Execute(input, final_chunk, initial_idx);
            if (result == OperatorResultType::FINISHED) {
                return OperatorResultType::FINISHED;
            }
        } else {
            result = OperatorResultType::NEED_MORE_INPUT;
        }

        // Sink 处理
        auto &sink_chunk = final_chunk;
        if (sink_chunk.size() > 0) {
            StartOperator(*pipeline.sink);
            OperatorSinkInput sink_input {
                *pipeline.sink->sink_state,
                *local_sink_state,
                interrupt_state
            };

            auto sink_result = Sink(sink_chunk, sink_input);
            EndOperator(*pipeline.sink, nullptr);

            if (sink_result == SinkResultType::BLOCKED) {
                return OperatorResultType::BLOCKED;
            } else if (sink_result == SinkResultType::FINISHED) {
                FinishProcessing();
                return OperatorResultType::FINISHED;
            }
        }

        if (result == OperatorResultType::NEED_MORE_INPUT) {
            return OperatorResultType::NEED_MORE_INPUT;
        }
    } while (chunk_budget.Next());

    return result;
}
```

### 2.4 执行流程图

```
┌─────────────────────────────────────────────────────────────┐
│              PipelineExecutor 执行流程                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │            Execute() 主循环                  │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│       ┌───────────────┼───────────────┐                     │
│       ▼               ▼               ▼                     │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐               │
│  │ 从Source│    │处理残余  │    │刷新缓存  │               │
│  │ 获取数据│    │Sink块    │    │算子      │               │
│  └────┬────┘    └────┬─────┘    └────┬─────┘               │
│       │              │               │                     │
│       └──────────────┼───────────────┘                     │
│                      ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │        ExecutePushInternal()                │            │
│  │                                              │            │
│  │  for each operator in pipeline:             │            │
│  │      chunk = operator.Execute(prev_chunk)   │            │
│  │                                              │            │
│  │  sink.Sink(final_chunk)                     │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│       ┌───────────────┼───────────────┐                     │
│       ▼               ▼               ▼                     │
│  NEED_MORE_INPUT  HAVE_MORE_OUTPUT  BLOCKED                │
│       │               │               │                     │
│       ▼               ▼               ▼                     │
│   继续获取数据    保存状态后      设置中断标志              │
│                  继续处理       返回INTERRUPTED             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 任务调度系统

### 3.1 TaskScheduler

`TaskScheduler` 管理工作线程和任务队列：

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp

class TaskScheduler {
    constexpr static int64_t TASK_TIMEOUT_USECS = 5000;  // 5ms 超时

public:
    explicit TaskScheduler(DatabaseInstance &db);

    static TaskScheduler &GetScheduler(ClientContext &context);

    //! 创建生产者令牌
    unique_ptr<ProducerToken> CreateProducer();

    //! 调度任务
    void ScheduleTask(ProducerToken &producer, shared_ptr<Task> task);
    void ScheduleTasks(ProducerToken &producer, vector<shared_ptr<Task>> &tasks);

    //! 从生产者获取任务
    bool GetTaskFromProducer(ProducerToken &token, shared_ptr<Task> &task);

    //! 持续执行任务直到标记为 false
    void ExecuteForever(atomic<bool> *marker);

    //! 执行指定数量的任务
    idx_t ExecuteTasks(atomic<bool> *marker, idx_t max_tasks);

    //! 设置线程数量
    void SetThreads(idx_t total_threads, idx_t external_threads);

    //! 获取线程数量
    int32_t NumberOfThreads();

    //! 唤醒 n 个等待的线程
    void Signal(idx_t n);

    //! 线程让步
    static void YieldThread();

private:
    DatabaseInstance &db;
    //! 并发任务队列
    unique_ptr<ConcurrentQueue> queue;
    //! 线程修改锁
    mutex thread_lock;
    //! 后台工作线程
    vector<unique_ptr<SchedulerThread>> threads;
    //! 线程停止标记
    vector<unique_ptr<atomic<bool>>> markers;
    //! 请求的线程数
    atomic<int32_t> requested_thread_count;
    //! 当前运行的线程数
    atomic<int32_t> current_thread_count;
};
```

### 3.2 Task 抽象

```cpp
// src/include/duckdb/parallel/task.hpp

enum class TaskExecutionMode : uint8_t {
    PROCESS_ALL,     // 完整处理
    PROCESS_PARTIAL  // 部分处理（可中断）
};

enum class TaskExecutionResult : uint8_t {
    TASK_FINISHED,      // 任务完成
    TASK_NOT_FINISHED,  // 任务未完成，需再次执行
    TASK_ERROR,         // 任务出错
    TASK_BLOCKED        // 任务被阻塞
};

class Task : public enable_shared_from_this<Task> {
public:
    virtual ~Task() {}

    //! 执行任务
    virtual TaskExecutionResult Execute(TaskExecutionMode mode) = 0;

    //! 取消调度（保持可重新调度）
    virtual void Deschedule();

    //! 重新调度
    virtual void Reschedule();

    //! 是否在结果上阻塞
    virtual bool TaskBlockedOnResult() const { return false; }

    optional_ptr<ProducerToken> token;
};
```

### 3.3 PipelineTask

`PipelineTask` 将 Pipeline 执行封装为可调度的任务：

```cpp
// src/include/duckdb/parallel/pipeline.hpp

class PipelineTask : public ExecutorTask {
    static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

public:
    explicit PipelineTask(Pipeline &pipeline_p, shared_ptr<Event> event_p);

    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

public:
    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
        if (!pipeline_executor) {
            // 延迟创建 PipelineExecutor
            pipeline_executor = make_uniq<PipelineExecutor>(
                pipeline.GetClientContext(), pipeline);
        }

        pipeline_executor->SetTaskForInterrupts(shared_from_this());

        if (mode == TaskExecutionMode::PROCESS_PARTIAL) {
            // 部分执行模式：最多处理 50 个数据块
            auto res = pipeline_executor->Execute(PARTIAL_CHUNK_COUNT);

            switch (res) {
            case PipelineExecuteResult::NOT_FINISHED:
                return TaskExecutionResult::TASK_NOT_FINISHED;
            case PipelineExecuteResult::INTERRUPTED:
                return TaskExecutionResult::TASK_BLOCKED;
            case PipelineExecuteResult::FINISHED:
                break;
            }
        } else {
            // 完整执行模式
            auto res = pipeline_executor->Execute();
            switch (res) {
            case PipelineExecuteResult::INTERRUPTED:
                return TaskExecutionResult::TASK_BLOCKED;
            case PipelineExecuteResult::FINISHED:
                break;
            default:
                throw InternalException("Unexpected result");
            }
        }

        event->FinishTask();
        pipeline_executor.reset();
        return TaskExecutionResult::TASK_FINISHED;
    }
};
```

### 3.4 任务调度流程

```
┌─────────────────────────────────────────────────────────────┐
│                   任务调度系统架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │              TaskScheduler                  │            │
│  │                                              │            │
│  │  ┌───────────────────────────────────────┐  │            │
│  │  │         ConcurrentQueue              │  │            │
│  │  │                                       │  │            │
│  │  │  Task1  Task2  Task3  Task4  ...     │  │            │
│  │  └───────────────────────────────────────┘  │            │
│  └──────────────────┬──────────────────────────┘            │
│                     │                                       │
│       ┌─────────────┼─────────────┐                         │
│       ▼             ▼             ▼                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                   │
│  │ Worker  │   │ Worker  │   │ Worker  │                   │
│  │ Thread 1│   │ Thread 2│   │ Thread N│                   │
│  └────┬────┘   └────┬────┘   └────┬────┘                   │
│       │             │             │                         │
│       ▼             ▼             ▼                         │
│  ┌─────────────────────────────────────────────┐            │
│  │           ExecuteForever()                  │            │
│  │                                              │            │
│  │  while (marker):                            │            │
│  │      task = queue.Pop()                     │            │
│  │      result = task.Execute(PROCESS_PARTIAL) │            │
│  │      if result == NOT_FINISHED:             │            │
│  │          queue.Push(task)                   │            │
│  │      elif result == BLOCKED:                │            │
│  │          task.Deschedule()                  │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. Event 系统

### 4.1 Event 基类

`Event` 表示一个可调度的事件，管理一组任务的执行：

```cpp
// src/include/duckdb/parallel/event.hpp

class Event : public enable_shared_from_this<Event> {
public:
    explicit Event(Executor &executor);
    virtual ~Event() = default;

    //! 调度事件的任务
    virtual void Schedule() = 0;

    //! 事件完成后立即调用
    virtual void FinishEvent() {}

    //! 事件完全完成后调用
    virtual void FinalizeFinish() {}

    //! 单个任务完成
    void FinishTask();

    //! 整个事件完成
    void Finish();

    //! 添加依赖
    void AddDependency(Event &event);

    //! 依赖完成通知
    void CompleteDependency();

    //! 设置任务列表
    void SetTasks(vector<shared_ptr<Task>> tasks);

    //! 插入新事件（作为依赖）
    void InsertEvent(shared_ptr<Event> replacement_event);

    bool IsFinished() const { return finished; }

protected:
    Executor &executor;
    //! 已完成的任务数
    atomic<idx_t> finished_tasks;
    //! 总任务数
    atomic<idx_t> total_tasks;

    //! 已完成的依赖数
    atomic<idx_t> finished_dependencies;
    //! 总依赖数
    idx_t total_dependencies;

    //! 父事件（依赖此事件的事件）
    vector<weak_ptr<Event>> parents;

    //! 是否已完成
    atomic<bool> finished;
};
```

### 4.2 Event 生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                    Event 生命周期                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  创建 Event                                                  │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 等待依赖                                     │            │
│  │                                              │            │
│  │ if (total_dependencies > 0):                │            │
│  │     wait for CompleteDependency() calls     │            │
│  └────────────────────┬────────────────────────┘            │
│                       │ 依赖完成                            │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ Schedule()                                  │            │
│  │                                              │            │
│  │ 创建并调度任务:                              │            │
│  │   tasks = CreateTasks()                     │            │
│  │   SetTasks(tasks)                           │            │
│  │   scheduler.ScheduleTasks(tasks)            │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│       ┌───────────────┼───────────────┐                     │
│       ▼               ▼               ▼                     │
│   Task 1          Task 2          Task N                   │
│       │               │               │                     │
│       │  Execute()    │  Execute()    │  Execute()          │
│       │               │               │                     │
│       ▼               ▼               ▼                     │
│   FinishTask()    FinishTask()    FinishTask()             │
│       │               │               │                     │
│       └───────────────┼───────────────┘                     │
│                       │ 所有任务完成                        │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ Finish()                                    │            │
│  │                                              │            │
│  │ 1. FinishEvent() - 事件特定的清理           │            │
│  │ 2. 通知父事件: parent.CompleteDependency()  │            │
│  │ 3. FinalizeFinish() - 最终清理              │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 PipelineEvent

`PipelineEvent` 是执行 Pipeline 的具体事件实现：

```cpp
// Pipeline 调度时创建 PipelineEvent

void Pipeline::Schedule(shared_ptr<Event> &event) {
    D_ASSERT(ready);
    D_ASSERT(sink);

    Reset();  // 初始化状态

    if (!ScheduleParallel(event)) {
        // 无法并行化，创建单个任务
        ScheduleSequentialTask(event);
    }
}

bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // 检查所有算子是否支持并行
    if (!sink->ParallelSink()) {
        return false;
    }
    if (!source->ParallelSource()) {
        return false;
    }

    auto max_threads = source_state->MaxThreads();

    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (!op.ParallelOperator()) {
            return false;
        }
        max_threads = MinValue<idx_t>(max_threads, op.op_state->MaxThreads(max_threads));
    }

    // 获取实际可用线程数
    auto &scheduler = TaskScheduler::GetScheduler(executor.context);
    auto active_threads = NumericCast<idx_t>(scheduler.NumberOfThreads());
    max_threads = MinValue(max_threads, active_threads);

    // 考虑 Sink 的线程限制
    if (sink && sink->sink_state) {
        max_threads = sink->sink_state->MaxThreads(max_threads);
    }

    return LaunchScanTasks(event, max_threads);
}

bool Pipeline::LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads) {
    if (max_threads <= 1) {
        return false;  // 太小，不值得并行化
    }

    // 为每个线程创建一个任务
    vector<shared_ptr<Task>> tasks;
    for (idx_t i = 0; i < max_threads; i++) {
        tasks.push_back(make_uniq<PipelineTask>(*this, event));
    }
    event->SetTasks(std::move(tasks));
    return true;
}
```

## 5. MetaPipeline 组织

### 5.1 MetaPipeline 概念

`MetaPipeline` 表示一组共享相同 Sink 的 Pipeline：

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp

enum class MetaPipelineType : uint8_t {
    REGULAR = 0,    // 常规 Sink
    JOIN_BUILD = 1  // Join Build Sink
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
public:
    MetaPipeline(Executor &executor, PipelineBuildState &state,
                 optional_ptr<PhysicalOperator> sink,
                 MetaPipelineType type = MetaPipelineType::REGULAR);

    //! 获取基础 Pipeline
    shared_ptr<Pipeline> &GetBasePipeline();

    //! 获取所有 Pipeline
    void GetPipelines(vector<shared_ptr<Pipeline>> &result, bool recursive);

    //! 创建新的 Pipeline
    Pipeline &CreatePipeline();

    //! 创建 Union Pipeline（当前 Pipeline 的克隆）
    Pipeline &CreateUnionPipeline(Pipeline &current, bool order_matters);

    //! 创建子 Pipeline
    void CreateChildPipeline(Pipeline &current, PhysicalOperator &op,
                             Pipeline &last_pipeline);

    //! 创建子 MetaPipeline
    MetaPipeline &CreateChildMetaPipeline(Pipeline &current, PhysicalOperator &op,
                                          MetaPipelineType type);

private:
    Executor &executor;
    PipelineBuildState &state;
    optional_ptr<Pipeline> parent;
    optional_ptr<PhysicalOperator> sink;
    MetaPipelineType type;

    //! 所有共享相同 Sink 的 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! Pipeline 间的依赖关系
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
    //! 子 MetaPipeline
    vector<shared_ptr<MetaPipeline>> children;
};
```

### 5.2 MetaPipeline 构建规则

```
┌─────────────────────────────────────────────────────────────┐
│                MetaPipeline 构建规则                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  规则 1: 对于 Join，先构建阻塞侧（Build）再构建探测侧       │
│                                                             │
│          HashJoin                                           │
│         /        \                                          │
│    Build侧     Probe侧                                      │
│       ↓           ↓                                         │
│   MetaPipeline1  MetaPipeline2（依赖 1）                    │
│                                                             │
│  ──────────────────────────────────────────────────────    │
│                                                             │
│  规则 2: 子 Pipeline 最后构建                                │
│                                                             │
│          HashJoin (Full Outer)                              │
│              │                                              │
│              ↓                                              │
│   Pipeline1: Probe → Output                                │
│              ↓                                              │
│   Pipeline2: ScanHT → Output (子 Pipeline)                 │
│                                                             │
│   Pipeline2 依赖 Pipeline1 及其后续 Pipeline                │
│                                                             │
│  ──────────────────────────────────────────────────────    │
│                                                             │
│  规则 3: Union 创建多个共享 Sink 的 Pipeline                 │
│                                                             │
│          UNION ALL                                          │
│         /        \                                          │
│    Query1      Query2                                       │
│       ↓           ↓                                         │
│   Pipeline1    Pipeline2                                    │
│       \         /                                           │
│        → Sink ←                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 6. 中断和异步处理

### 6.1 InterruptState

`InterruptState` 用于处理算子的异步阻塞：

```cpp
// src/include/duckdb/parallel/interrupt.hpp

enum class InterruptMode : uint8_t {
    NO_INTERRUPTS,  // 不支持中断
    TASK,           // 任务模式（推荐）
    BLOCKING        // 阻塞模式
};

class InterruptState {
public:
    //! 默认构造，不支持中断
    InterruptState();
    //! 任务模式
    explicit InterruptState(weak_ptr<Task> task);
    //! 阻塞模式
    explicit InterruptState(weak_ptr<InterruptDoneSignalState> done_signal);

    //! 中断完成回调
    void Callback() const;

protected:
    InterruptMode mode;
    weak_ptr<Task> current_task;
    weak_ptr<InterruptDoneSignalState> signal_state;
};
```

### 6.2 StateWithBlockableTasks

```cpp
// 用于 GlobalSourceState 和 GlobalSinkState 的阻塞支持

class StateWithBlockableTasks {
public:
    unique_lock<mutex> Lock() {
        return unique_lock<mutex>(lock);
    }

    //! 阻止后续阻塞
    void PreventBlocking(const unique_lock<mutex> &guard) {
        can_block = false;
    }

    //! 添加阻塞任务
    bool BlockTask(const unique_lock<mutex> &guard,
                   const InterruptState &interrupt_state) {
        if (can_block) {
            blocked_tasks.push_back(interrupt_state);
            return true;
        }
        return false;
    }

    //! 解除所有阻塞任务
    bool UnblockTasks(const unique_lock<mutex> &guard) {
        if (blocked_tasks.empty()) {
            return false;
        }
        for (auto &entry : blocked_tasks) {
            entry.Callback();  // 触发回调，重新调度任务
        }
        blocked_tasks.clear();
        return true;
    }

    //! Sink 阻塞辅助
    SinkResultType BlockSink(const unique_lock<mutex> &guard,
                             const InterruptState &interrupt_state) {
        return BlockTask(guard, interrupt_state)
            ? SinkResultType::BLOCKED
            : SinkResultType::FINISHED;
    }

    //! Source 阻塞辅助
    SourceResultType BlockSource(const unique_lock<mutex> &guard,
                                 const InterruptState &interrupt_state) {
        return BlockTask(guard, interrupt_state)
            ? SourceResultType::BLOCKED
            : SourceResultType::FINISHED;
    }

private:
    atomic<bool> can_block{true};
    mutable mutex lock;
    mutable vector<InterruptState> blocked_tasks;
};
```

### 6.3 异步执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                    异步执行流程                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 正常执行                                                 │
│  ┌─────────────────────────────────────────────┐            │
│  │ Task.Execute()                              │            │
│  │   └─ PipelineExecutor.Execute()             │            │
│  │       └─ Source.GetData() 或 Sink.Sink()   │            │
│  │           └─ 返回 BLOCKED                   │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│  2. 阻塞处理                                                 │
│  ┌────────────────────┴────────────────────────┐            │
│  │ 算子将任务添加到 blocked_tasks              │            │
│  │ 返回 BLOCKED/INTERRUPTED                    │            │
│  │ Task 返回 TASK_BLOCKED                      │            │
│  │ Task.Deschedule() - 从队列移除              │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│  3. 异步操作进行中                                           │
│  ┌────────────────────┴────────────────────────┐            │
│  │ 后台线程/IO 完成等                           │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│  4. 操作完成，恢复执行                                       │
│  ┌────────────────────┴────────────────────────┐            │
│  │ 算子调用 interrupt_state.Callback()         │            │
│  │   └─ Task.Reschedule() - 重新加入队列       │            │
│  │ Task.Execute() 继续执行                     │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 7. 并行度控制

### 7.1 线程数量决定

```cpp
// Pipeline 并行调度时确定线程数

bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // 1. 检查所有算子是否支持并行
    if (!sink->ParallelSink() || !source->ParallelSource()) {
        return false;
    }

    // 2. 从 Source 获取最大并行度
    auto max_threads = source_state->MaxThreads();

    // 3. 受中间算子限制
    for (auto &op_ref : operators) {
        if (!op.ParallelOperator()) {
            return false;
        }
        max_threads = MinValue<idx_t>(max_threads,
                                      op.op_state->MaxThreads(max_threads));
    }

    // 4. 受系统线程数限制
    auto &scheduler = TaskScheduler::GetScheduler(executor.context);
    auto active_threads = scheduler.NumberOfThreads();
    max_threads = MinValue(max_threads, active_threads);

    // 5. 受 Sink 限制
    if (sink && sink->sink_state) {
        max_threads = sink->sink_state->MaxThreads(max_threads);
    }

    return LaunchScanTasks(event, max_threads);
}
```

### 7.2 MaxThreads 实现示例

```cpp
// Source: TableScan 根据数据量决定并行度
class TableScanGlobalState : public GlobalSourceState {
    idx_t MaxThreads() override {
        // 基于扫描的行数估算
        return (total_rows + STANDARD_VECTOR_SIZE - 1) / STANDARD_VECTOR_SIZE;
    }
};

// Sink: HashJoin Build 根据内存限制决定
class HashJoinGlobalSinkState : public GlobalSinkState {
    idx_t MaxThreads(idx_t source_max_threads) override {
        // 受内存预算限制
        auto memory_limit = temporary_memory_state->GetReservation();
        auto threads_by_memory = memory_limit / estimated_memory_per_thread;
        return MinValue(source_max_threads, threads_by_memory);
    }
};

// Operator: 默认透传
class GlobalOperatorState {
    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }
};
```

## 8. 执行上下文

### 8.1 ThreadContext

`ThreadContext` 保存线程本地信息：

```cpp
// src/include/duckdb/parallel/thread_context.hpp

class ThreadContext {
public:
    explicit ThreadContext(ClientContext &context);
    ~ThreadContext();

    //! 算子性能分析器
    OperatorProfiler profiler;
    //! 日志记录器
    unique_ptr<Logger> logger;
};
```

### 8.2 ExecutionContext

`ExecutionContext` 统一访问执行相关上下文：

```cpp
// src/include/duckdb/execution/execution_context.hpp

class ExecutionContext {
public:
    ExecutionContext(ClientContext &client_p, ThreadContext &thread_p,
                     optional_ptr<Pipeline> pipeline_p = nullptr);

    ClientContext &client;
    ThreadContext &thread;
    optional_ptr<Pipeline> pipeline;
};
```

## 9. 批次索引和顺序保持

### 9.1 批次索引机制

某些算子（如 ORDER BY 输出）需要保持特定顺序：

```cpp
// 批次索引注册和更新
idx_t Pipeline::RegisterNewBatchIndex() {
    lock_guard<mutex> l(batch_lock);
    idx_t minimum = batch_indexes.empty()
        ? base_batch_index
        : *batch_indexes.begin();
    batch_indexes.insert(minimum);
    return minimum;
}

idx_t Pipeline::UpdateBatchIndex(idx_t old_index, idx_t new_index) {
    lock_guard<mutex> l(batch_lock);
    if (new_index < *batch_indexes.begin()) {
        throw InternalException("Invalid batch index");
    }
    auto entry = batch_indexes.find(old_index);
    batch_indexes.erase(entry);
    batch_indexes.insert(new_index);
    return *batch_indexes.begin();  // 返回最小批次索引
}
```

### 9.2 顺序依赖检查

```cpp
bool Pipeline::IsOrderDependent() const {
    // 检查 Source 顺序
    if (source) {
        auto source_order = source->SourceOrder();
        if (source_order == OrderPreservationType::FIXED_ORDER) {
            return true;
        }
        if (source_order == OrderPreservationType::NO_ORDER) {
            return false;
        }
    }

    // 检查中间算子顺序
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (op.OperatorOrder() == OrderPreservationType::NO_ORDER) {
            return false;
        }
        if (op.OperatorOrder() == OrderPreservationType::FIXED_ORDER) {
            return true;
        }
    }

    // 检查配置和 Sink
    if (!DBConfig::GetSetting<PreserveInsertionOrderSetting>(executor.context)) {
        return false;
    }
    if (sink && sink->SinkOrderDependent()) {
        return true;
    }
    return false;
}
```

## 10. 完整执行流程示例

```
┌─────────────────────────────────────────────────────────────┐
│              查询并行执行完整流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  查询: SELECT * FROM A JOIN B ON a.id=b.id GROUP BY ...     │
│                                                             │
│  ═══════════════════════════════════════════════════════   │
│                                                             │
│  1. 构建 MetaPipeline 树                                    │
│  ┌─────────────────────────────────────────────┐            │
│  │ MetaPipeline 1 (Aggregate Sink)             │            │
│  │  └─ Pipeline A: Probe → Aggregate.Sink()    │            │
│  │                                              │            │
│  │ MetaPipeline 2 (HashJoin Build, 依赖 1)     │            │
│  │  └─ Pipeline B: Scan(B) → HashJoin.Sink()   │            │
│  │                                              │            │
│  │ MetaPipeline 3 (Result, 依赖 Aggregate)     │            │
│  │  └─ Pipeline C: Aggregate.Source() → Result │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  ═══════════════════════════════════════════════════════   │
│                                                             │
│  2. 调度执行                                                 │
│  ┌─────────────────────────────────────────────┐            │
│  │ Phase 1: 执行 Pipeline B (Build Hash Table) │            │
│  │                                              │            │
│  │  Event: PipelineEvent                       │            │
│  │    └─ Tasks: [PipelineTask×N]               │            │
│  │                                              │            │
│  │  并行执行:                                   │            │
│  │    Thread 1: Scan(B) → HashJoin.Sink()      │            │
│  │    Thread 2: Scan(B) → HashJoin.Sink()      │            │
│  │    ...                                       │            │
│  │                                              │            │
│  │  完成后: HashJoin.Finalize()                 │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ Phase 2: 执行 Pipeline A (Probe + Aggregate)│            │
│  │                                              │            │
│  │  并行执行:                                   │            │
│  │    Thread 1: Scan(A) → HashJoin.Execute()   │            │
│  │              → Aggregate.Sink()             │            │
│  │    Thread 2: ...                            │            │
│  │                                              │            │
│  │  完成后: Aggregate.Finalize()               │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ Phase 3: 执行 Pipeline C (输出结果)          │            │
│  │                                              │            │
│  │  Aggregate.Source() → Result                │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 11. 总结

### 11.1 核心组件关系

```
┌────────────────────────────────────────────────────────────────┐
│                     并行执行组件关系                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  MetaPipeline                                                  │
│      │                                                         │
│      ├── Pipeline 1 ──┐                                        │
│      ├── Pipeline 2 ──┼── PipelineEvent                        │
│      └── Pipeline N ──┘         │                              │
│                                 ├── PipelineTask 1             │
│                                 ├── PipelineTask 2             │
│                                 └── PipelineTask N             │
│                                         │                      │
│                                         ▼                      │
│                                 PipelineExecutor               │
│                                         │                      │
│                                         ▼                      │
│                                 TaskScheduler                  │
│                                         │                      │
│                            ┌────────────┼────────────┐         │
│                            ▼            ▼            ▼         │
│                       Worker 1     Worker 2     Worker N       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 11.2 设计优势

| 特性 | 优势 |
|------|------|
| Pipeline 并行 | 同一 Pipeline 可多线程执行，充分利用多核 |
| 任务调度 | 工作窃取式调度，负载均衡 |
| Event 依赖 | 自动处理 Pipeline 间依赖，无需手动同步 |
| 中断支持 | 算子可异步阻塞，支持外部 IO 等操作 |
| 灵活并行度 | 算子可动态限制并行度，适应资源约束 |

### 11.3 关键设计决策

1. **Push-based 模型**：数据主动推送，减少调用开销
2. **细粒度任务**：PARTIAL 模式处理固定数量数据块，支持公平调度
3. **本地状态优先**：减少线程间同步，提高缓存效率
4. **延迟创建**：PipelineExecutor 在任务执行时才创建，减少内存使用
5. **异步友好**：InterruptState 机制支持非阻塞 IO 操作

在下一章中，我们将深入探讨 **哈希表和内存管理**，了解 DuckDB 如何高效管理大规模数据处理中的内存资源。

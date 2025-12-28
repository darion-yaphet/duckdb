# DuckDB 执行引擎深度解析：第九章 - 并行执行

## 9.1 章节概述

并行执行是现代数据库系统获得高性能的关键。本章深入分析 DuckDB 的并行执行框架，包括 TaskScheduler、Pipeline 并行、事件系统、Morsel-Driven 并行模型以及中断与同步机制。

```
┌────────────────────────────────────────────────────────────────────┐
│                     DuckDB 并行执行架构                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                       Executor                               │ │
│   │  • 管理查询执行生命周期                                      │ │
│   │  • 协调多个 Pipeline                                         │ │
│   │  • 错误处理和取消                                            │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                     MetaPipeline                             │ │
│   │  • Pipeline 分组                                             │ │
│   │  • 依赖关系管理                                              │ │
│   │  • 事件调度                                                  │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│          ┌─────────────────────┼─────────────────────┐            │
│          ▼                     ▼                     ▼            │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐    │
│   │  Pipeline 1 │       │  Pipeline 2 │       │  Pipeline 3 │    │
│   └──────┬──────┘       └──────┬──────┘       └──────┬──────┘    │
│          │                     │                     │            │
│          ▼                     ▼                     ▼            │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                    TaskScheduler                             │ │
│   │  • ConcurrentQueue (任务队列)                                │ │
│   │  • Worker Threads (工作线程)                                 │ │
│   │  • Semaphore (信号量)                                        │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                   Thread Pool                                │ │
│   │   ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐         │ │
│   │   │ T1   │  │ T2   │  │ T3   │  │ T4   │  │ ...  │         │ │
│   │   └──────┘  └──────┘  └──────┘  └──────┘  └──────┘         │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9.2 TaskScheduler：任务调度器

### 9.2.1 TaskScheduler 类设计

`TaskScheduler` 是 DuckDB 并行执行的核心组件，负责管理任务队列和工作线程。

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp

class TaskScheduler {
    // 信号量等待超时时间（微秒）
    constexpr static int64_t TASK_TIMEOUT_USECS = 5000;

public:
    explicit TaskScheduler(DatabaseInstance &db);
    ~TaskScheduler();

    static TaskScheduler &GetScheduler(ClientContext &context);
    static TaskScheduler &GetScheduler(DatabaseInstance &db);

    //! 创建生产者令牌（用于提交任务）
    unique_ptr<ProducerToken> CreateProducer();

    //! 调度单个任务
    void ScheduleTask(ProducerToken &producer, shared_ptr<Task> task);
    //! 批量调度任务
    void ScheduleTasks(ProducerToken &producer, vector<shared_ptr<Task>> &tasks);

    //! 从指定生产者获取任务
    bool GetTaskFromProducer(ProducerToken &token, shared_ptr<Task> &task);

    //! 工作线程主循环
    void ExecuteForever(atomic<bool> *marker);
    //! 执行指定数量的任务
    idx_t ExecuteTasks(atomic<bool> *marker, idx_t max_tasks);
    void ExecuteTasks(idx_t max_tasks);

    //! 设置线程数
    void SetThreads(idx_t total_threads, idx_t external_threads);
    void RelaunchThreads();

    //! 获取线程数
    DUCKDB_API int32_t NumberOfThreads();

    //! 唤醒 n 个等待的线程
    void Signal(idx_t n);

    //! 让出 CPU
    static void YieldThread();

    //! 获取当前 CPU ID（用于亲和性）
    static idx_t GetEstimatedCPUId();

private:
    DatabaseInstance &db;
    //! 任务队列
    unique_ptr<ConcurrentQueue> queue;
    //! 线程锁
    mutex thread_lock;
    //! 工作线程
    vector<unique_ptr<SchedulerThread>> threads;
    //! 线程标记（控制线程终止）
    vector<unique_ptr<atomic<bool>>> markers;
    //! 分配器刷新阈值
    atomic<idx_t> allocator_flush_threshold;
    //! 是否启用后台分配器线程
    atomic<bool> allocator_background_threads;
    //! 请求的线程数
    atomic<int32_t> requested_thread_count;
    //! 当前运行的线程数
    atomic<int32_t> current_thread_count;
};
```

### 9.2.2 ConcurrentQueue：并发任务队列

DuckDB 使用 moodycamel 的无锁并发队列实现高效的任务分发：

```cpp
// src/parallel/task_scheduler.cpp

#ifndef DUCKDB_NO_THREADS
typedef duckdb_moodycamel::ConcurrentQueue<shared_ptr<Task>> concurrent_queue_t;
typedef duckdb_moodycamel::LightweightSemaphore lightweight_semaphore_t;

struct ConcurrentQueue {
    ConcurrentQueue() : tasks_in_queue(0) {}

    lightweight_semaphore_t semaphore;

    //! 入队（单个任务）
    void Enqueue(ProducerToken &token, shared_ptr<Task> task);
    //! 批量入队
    void EnqueueBulk(ProducerToken &token, vector<shared_ptr<Task>> &tasks);
    //! 从指定生产者出队
    bool DequeueFromProducer(ProducerToken &token, shared_ptr<Task> &task);
    //! 从任意生产者出队
    bool Dequeue(shared_ptr<Task> &task);

    idx_t GetTasksInQueue() const;

private:
    concurrent_queue_t q;
    atomic<idx_t> tasks_in_queue;
};

void ConcurrentQueue::Enqueue(ProducerToken &token, shared_ptr<Task> task) {
    lock_guard<mutex> producer_lock(token.producer_lock);
    task->token = token;
    if (q.enqueue(token.token->queue_token, std::move(task))) {
        ++tasks_in_queue;
        semaphore.signal();  // 唤醒等待的工作线程
    } else {
        throw InternalException("Could not schedule task!");
    }
}

bool ConcurrentQueue::Dequeue(shared_ptr<Task> &task) {
    if (!q.try_dequeue(task)) {
        return false;
    }
    --tasks_in_queue;
    return true;
}
#endif
```

### 9.2.3 ProducerToken：生产者令牌

每个查询执行器（Executor）持有一个 ProducerToken，用于向队列提交任务：

```cpp
struct ProducerToken {
    ProducerToken(TaskScheduler &scheduler, unique_ptr<QueueProducerToken> token);
    ~ProducerToken();

    TaskScheduler &scheduler;
    unique_ptr<QueueProducerToken> token;
    mutex producer_lock;  // 保护生产者状态
};

struct QueueProducerToken {
    explicit QueueProducerToken(ConcurrentQueue &queue)
        : queue_token(queue.GetQueue()) {}

    duckdb_moodycamel::ProducerToken queue_token;
};
```

### 9.2.4 工作线程执行循环

```cpp
void TaskScheduler::ExecuteForever(atomic<bool> *marker) {
    static constexpr const int64_t INITIAL_FLUSH_WAIT = 500000; // 0.5秒

    const auto &block_allocator = BlockAllocator::Get(db);
    const auto &config = DBConfig::GetConfig(db);

    shared_ptr<Task> task;

    // 主循环
    while (*marker) {
        // 等待信号量（带超时，用于内存刷新）
        if (!block_allocator.SupportsFlush()) {
            queue->semaphore.wait();
        } else if (!queue->semaphore.wait(INITIAL_FLUSH_WAIT)) {
            // 空闲 0.5 秒后刷新内存分配
            block_allocator.ThreadFlush(allocator_background_threads,
                                         allocator_flush_threshold,
                                         requested_thread_count.load());
            // 继续等待或标记为空闲
            auto decay_delay = Allocator::DecayDelay();
            if (!decay_delay.IsValid()) {
                queue->semaphore.wait();
            } else {
                if (!queue->semaphore.wait(decay_delay.GetIndex() * 1000000 - INITIAL_FLUSH_WAIT)) {
                    Allocator::ThreadIdle();
                    queue->semaphore.wait();
                }
            }
        }

        // 尝试获取任务
        if (queue->Dequeue(task)) {
            auto process_mode = config.options.scheduler_process_partial
                                ? TaskExecutionMode::PROCESS_PARTIAL
                                : TaskExecutionMode::PROCESS_ALL;
            auto execute_result = task->Execute(process_mode);

            switch (execute_result) {
            case TaskExecutionResult::TASK_FINISHED:
            case TaskExecutionResult::TASK_ERROR:
                task.reset();
                break;
            case TaskExecutionResult::TASK_NOT_FINISHED:
                // 任务未完成，重新入队
                auto &token = *task->token;
                queue->Enqueue(token, std::move(task));
                break;
            case TaskExecutionResult::TASK_BLOCKED:
                // 任务阻塞，取消调度
                task->Deschedule();
                task.reset();
                break;
            }
        }
    }

    // 线程退出时刷新分配
    if (block_allocator.SupportsFlush()) {
        block_allocator.ThreadFlush(allocator_background_threads, 0, requested_thread_count.load());
        Allocator::ThreadIdle();
    }
}
```

```
┌────────────────────────────────────────────────────────────────────┐
│                    工作线程状态机                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│         ┌──────────────┐                                          │
│         │    WAITING   │ ◄────────────────────────────────────┐   │
│         │  (semaphore) │                                      │   │
│         └──────┬───────┘                                      │   │
│                │ signal                                       │   │
│                ▼                                              │   │
│         ┌──────────────┐      获取失败                        │   │
│         │   DEQUEUE    │ ─────────────────────────────────────┘   │
│         │    (task)    │                                          │
│         └──────┬───────┘                                          │
│                │ 获取成功                                          │
│                ▼                                                   │
│         ┌──────────────┐                                          │
│         │   EXECUTE    │                                          │
│         │    (task)    │                                          │
│         └──────┬───────┘                                          │
│                │                                                   │
│       ┌────────┼────────┬────────────────┐                        │
│       ▼        ▼        ▼                ▼                        │
│   FINISHED  ERROR   NOT_FINISHED     BLOCKED                      │
│       │        │        │                │                        │
│       │        │        │                │                        │
│       ▼        ▼        ▼                ▼                        │
│    reset()  reset()  re-enqueue()  deschedule()                   │
│       │        │        │                │                        │
│       └────────┴────────┴────────────────┘                        │
│                │                                                   │
│                └─────────────────────────────────────────────────►│
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.2.5 线程管理

```cpp
void TaskScheduler::SetThreads(idx_t total_threads, idx_t external_threads) {
    if (total_threads == 0) {
        throw SyntaxException("Number of threads must be positive!");
    }
    if (total_threads < external_threads) {
        throw SyntaxException("Number of threads can't be smaller than external threads!");
    }
    requested_thread_count = total_threads - external_threads;
}

void TaskScheduler::RelaunchThreadsInternal(int32_t n) {
    auto &config = DBConfig::GetConfig(db);
    auto new_thread_count = NumericCast<idx_t>(n);

    if (threads.size() == new_thread_count) {
        current_thread_count = threads.size() + config.options.external_threads;
        return;
    }

    // 减少线程数
    if (threads.size() > new_thread_count) {
        // 设置终止标记
        for (idx_t i = 0; i < threads.size(); i++) {
            *markers[i] = false;
        }
        Signal(threads.size());  // 唤醒所有线程

        // 等待线程结束
        for (idx_t i = 0; i < threads.size(); i++) {
            threads[i]->internal_thread->join();
        }
        threads.clear();
        markers.clear();
    }

    // 增加线程数
    if (threads.size() < new_thread_count) {
        idx_t create_new_threads = new_thread_count - threads.size();

        // 是否固定线程到 CPU
        static constexpr idx_t THREAD_PIN_THRESHOLD = 64;
        const auto pin_threads = db.config.options.pin_threads == ThreadPinMode::ON ||
                                 (db.config.options.pin_threads == ThreadPinMode::AUTO &&
                                  std::thread::hardware_concurrency() > THREAD_PIN_THRESHOLD);

        for (idx_t i = 0; i < create_new_threads; i++) {
            auto marker = unique_ptr<atomic<bool>>(new atomic<bool>(true));
            unique_ptr<thread> worker_thread;
            try {
                worker_thread = make_uniq<thread>(ThreadExecuteTasks, this, marker.get());
                if (pin_threads) {
                    SetThreadAffinity(*worker_thread, threads.size());
                }
            } catch (std::exception &ex) {
                // 线程创建失败，停止创建更多线程
                break;
            }
            threads.push_back(make_uniq<SchedulerThread>(std::move(worker_thread)));
            markers.push_back(std::move(marker));
        }
    }

    current_thread_count = threads.size() + config.options.external_threads;
    BlockAllocator::Get(db).FlushAll();
}
```

---

## 9.3 Task：任务抽象

### 9.3.1 Task 基类

```cpp
// src/include/duckdb/parallel/task.hpp

enum class TaskExecutionMode : uint8_t {
    PROCESS_ALL,     // 必须完全处理完成
    PROCESS_PARTIAL  // 可以部分处理后返回
};

enum class TaskExecutionResult : uint8_t {
    TASK_FINISHED,     // 任务完成
    TASK_NOT_FINISHED, // 任务未完成（需要重新调度）
    TASK_ERROR,        // 任务出错
    TASK_BLOCKED       // 任务阻塞（等待外部事件）
};

class Task : public enable_shared_from_this<Task> {
public:
    virtual ~Task() {}

    //! 执行任务
    virtual TaskExecutionResult Execute(TaskExecutionMode mode) = 0;

    //! 取消调度（任务阻塞时调用）
    virtual void Deschedule() {
        throw InternalException("Cannot deschedule task of base Task class");
    }

    //! 重新调度
    virtual void Reschedule() {
        throw InternalException("Cannot reschedule task of base Task class");
    }

    virtual bool TaskBlockedOnResult() const {
        return false;
    }

    virtual string TaskType() const {
        return "UnnamedTask";
    }

public:
    optional_ptr<ProducerToken> token;  // 关联的生产者
};
```

### 9.3.2 PipelineTask：Pipeline 执行任务

```cpp
// src/include/duckdb/parallel/pipeline.hpp

class PipelineTask : public ExecutorTask {
    static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

public:
    explicit PipelineTask(Pipeline &pipeline_p, shared_ptr<Event> event_p);

    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

    string TaskType() const override {
        return "PipelineTask";
    }

public:
    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override;
};
```

---

## 9.4 Pipeline 并行执行

### 9.4.1 Pipeline 类

```cpp
// src/include/duckdb/parallel/pipeline.hpp

class Pipeline : public enable_shared_from_this<Pipeline> {
public:
    explicit Pipeline(Executor &execution_context);

    Executor &executor;

public:
    void AddDependency(shared_ptr<Pipeline> &pipeline);

    //! 准备执行
    void Ready();
    void Reset();
    void ResetSink();
    void ResetSource(bool force);

    //! 调度执行
    void Schedule(shared_ptr<Event> &event);
    void PrepareFinalize();

    //! 获取进度
    bool GetProgress(ProgressData &progress_data);

    //! 获取所有算子
    vector<reference<PhysicalOperator>> GetOperators();

    optional_ptr<PhysicalOperator> GetSink() { return sink; }
    optional_ptr<PhysicalOperator> GetSource() { return source; }

    //! 检查是否有序依赖
    bool IsOrderDependent() const;

    //! 批次索引管理（用于有序输出）
    idx_t RegisterNewBatchIndex();
    idx_t UpdateBatchIndex(idx_t old_index, idx_t new_index);

private:
    bool ready;
    atomic<bool> initialized;

    //! Source 算子
    optional_ptr<PhysicalOperator> source;
    //! 中间算子链
    vector<reference<PhysicalOperator>> operators;
    //! Sink 算子
    optional_ptr<PhysicalOperator> sink;

    //! 全局 Source 状态
    unique_ptr<GlobalSourceState> source_state;

    //! 父 Pipeline（依赖于本 Pipeline）
    vector<weak_ptr<Pipeline>> parents;
    //! 本 Pipeline 的依赖
    vector<weak_ptr<Pipeline>> dependencies;

    //! 批次索引管理
    idx_t base_batch_index = 0;
    mutex batch_lock;
    multiset<idx_t> batch_indexes;

private:
    void ScheduleSequentialTask(shared_ptr<Event> &event);
    bool LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads);
    bool ScheduleParallel(shared_ptr<Event> &event);
};
```

### 9.4.2 Pipeline 调度

```cpp
void Pipeline::Schedule(shared_ptr<Event> &event) {
    D_ASSERT(ready);
    D_ASSERT(sink);

    // 检查是否可以并行
    if (!ScheduleParallel(event)) {
        // 不能并行，串行执行
        ScheduleSequentialTask(event);
    }
}

bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // 获取最大并行度
    auto max_threads = source_state->MaxThreads();

    if (IsOrderDependent() || max_threads == 1) {
        return false;  // 需要串行执行
    }

    return LaunchScanTasks(event, max_threads);
}

bool Pipeline::LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads) {
    // 获取可用线程数
    auto &scheduler = TaskScheduler::GetScheduler(executor.context);
    auto available_threads = scheduler.NumberOfThreads();

    // 限制到可用资源
    idx_t num_tasks = MinValue<idx_t>(max_threads, available_threads);

    if (num_tasks <= 1) {
        return false;
    }

    // 创建并调度任务
    vector<shared_ptr<Task>> tasks;
    tasks.reserve(num_tasks);
    for (idx_t i = 0; i < num_tasks; i++) {
        tasks.push_back(make_shared<PipelineTask>(*this, event));
    }
    event->SetTasks(std::move(tasks));

    return true;
}
```

### 9.4.3 PipelineExecutor：Pipeline 执行器

```cpp
// src/include/duckdb/parallel/pipeline_executor.hpp

enum class PipelineExecuteResult {
    FINISHED,     // 完全执行完成
    NOT_FINISHED, // 未完成，可立即再次调用
    INTERRUPTED   // 被中断，需要等待回调
};

class PipelineExecutor {
public:
    PipelineExecutor(ClientContext &context, Pipeline &pipeline);

    //! 完整执行 Pipeline
    PipelineExecuteResult Execute();
    //! 执行指定数量的 chunk
    PipelineExecuteResult Execute(idx_t max_chunks);
    //! 完成执行（Source 耗尽后调用）
    PipelineExecuteResult PushFinalize();

    //! 设置中断任务
    void SetTaskForInterrupts(weak_ptr<Task> current_task);

private:
    Pipeline &pipeline;
    ThreadContext thread;
    ExecutionContext context;

    //! 中间 chunk（每个算子一个）
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    //! 中间状态（每个算子一个）
    vector<unique_ptr<OperatorState>> intermediate_states;

    //! 本地 Source 状态
    unique_ptr<LocalSourceState> local_source_state;
    //! 本地 Sink 状态
    unique_ptr<LocalSinkState> local_sink_state;
    //! 中断状态
    InterruptState interrupt_state;

    //! 最终输出 chunk
    DataChunk final_chunk;

    //! 待处理算子栈
    stack<idx_t> in_process_operators;

    bool finalized = false;
    int32_t finished_processing_idx = -1;
    bool exhausted_pipeline = false;
    bool remaining_sink_chunk = false;

private:
    SourceResultType FetchFromSource(DataChunk &result);
    OperatorResultType ExecutePushInternal(DataChunk &input, ExecutionBudget &budget, idx_t initial_idx = 0);
    SinkNextBatchType NextBatch(DataChunk &source_chunk, const bool have_more_output);
};
```

```
┌────────────────────────────────────────────────────────────────────┐
│                 PipelineExecutor 执行流程                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                    Execute() 循环                            │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  1. FetchFromSource                                          │ │
│   │     source->GetData(chunk)                                   │ │
│   │     返回: HAVE_MORE_OUTPUT / FINISHED / BLOCKED              │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  2. ExecutePushInternal (遍历中间算子)                       │ │
│   │     for each operator:                                       │ │
│   │       operator->Execute(input, output)                       │ │
│   │       返回: HAVE_MORE_OUTPUT / NEED_MORE_INPUT / FINISHED   │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  3. Sink                                                     │ │
│   │     sink->Sink(chunk)                                        │ │
│   │     返回: NEED_MORE_INPUT / FINISHED / BLOCKED               │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  4. 检查是否需要继续                                         │ │
│   │     - Source 未耗尽 → 继续循环                               │ │
│   │     - Source 耗尽 → PushFinalize()                          │ │
│   │     - 被中断 → 返回 INTERRUPTED                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9.5 Event 系统

### 9.5.1 Event 基类

```cpp
// src/include/duckdb/parallel/event.hpp

class Event : public enable_shared_from_this<Event> {
public:
    explicit Event(Executor &executor);
    virtual ~Event() = default;

    //! 调度事件（创建并提交任务）
    virtual void Schedule() = 0;
    //! 事件完成后回调
    virtual void FinishEvent() {}
    //! 最终完成回调
    virtual void FinalizeFinish() {}

    //! 完成一个任务
    void FinishTask();
    //! 完成整个事件
    void Finish();

    //! 添加依赖
    void AddDependency(Event &event);
    bool HasDependencies() const {
        return total_dependencies != 0;
    }

    //! 完成一个依赖
    void CompleteDependency();

    //! 设置任务列表
    void SetTasks(vector<shared_ptr<Task>> tasks);

    //! 插入中间事件
    void InsertEvent(shared_ptr<Event> replacement_event);

    bool IsFinished() const {
        return finished;
    }

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

    //! 依赖于本事件的事件列表
    vector<weak_ptr<Event>> parents;

    //! 是否完成
    atomic<bool> finished;
};
```

### 9.5.2 事件依赖与触发

```cpp
void Event::AddDependency(Event &event) {
    event.parents.push_back(weak_from_this());
    total_dependencies++;
}

void Event::CompleteDependency() {
    idx_t current_finished = ++finished_dependencies;
    D_ASSERT(current_finished <= total_dependencies);
    if (current_finished == total_dependencies) {
        // 所有依赖完成，调度本事件
        Schedule();
    }
}

void Event::FinishTask() {
    idx_t current_finished = ++finished_tasks;
    D_ASSERT(current_finished <= total_tasks);
    if (current_finished == total_tasks) {
        // 所有任务完成
        Finish();
    }
}

void Event::Finish() {
    D_ASSERT(!finished);
    FinishEvent();
    finished = true;

    // 通知所有父事件
    for (auto &parent_ptr : parents) {
        auto parent = parent_ptr.lock();
        if (parent) {
            parent->CompleteDependency();
        }
    }

    FinalizeFinish();
}
```

```
┌────────────────────────────────────────────────────────────────────┐
│                       事件依赖图示例                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   SELECT * FROM t1 JOIN t2 ON ... ORDER BY ...                    │
│                                                                    │
│   ┌──────────────────────┐     ┌──────────────────────┐           │
│   │ PipelineEvent        │     │ PipelineEvent        │           │
│   │ (Scan t1 → Build HT) │     │ (Scan t2 → Probe)    │           │
│   └──────────┬───────────┘     └──────────┬───────────┘           │
│              │                            │                        │
│              │ dependencies               │ dependencies           │
│              ▼                            ▼                        │
│   ┌──────────────────────┐     ┌──────────────────────┐           │
│   │ PipelineFinishEvent  │     │ PipelineFinishEvent  │           │
│   │ (Finalize HT)        │─────│ (等待 Build 完成)    │           │
│   └──────────────────────┘     └──────────┬───────────┘           │
│                                           │                        │
│                                           │ dependencies           │
│                                           ▼                        │
│                                ┌──────────────────────┐           │
│                                │ PipelineEvent        │           │
│                                │ (HashAgg → Sort)     │           │
│                                └──────────┬───────────┘           │
│                                           │                        │
│                                           │ dependencies           │
│                                           ▼                        │
│                                ┌──────────────────────┐           │
│                                │ PipelineFinishEvent  │           │
│                                │ (输出结果)            │           │
│                                └──────────────────────┘           │
│                                                                    │
│   执行顺序:                                                        │
│   1. Scan t1 → Build HT (并行)                                    │
│   2. Finalize HT (等待 Build 完成)                                │
│   3. Scan t2 → Probe → HashAgg (并行，等待 HT Ready)              │
│   4. Sort (等待聚合完成)                                          │
│   5. 输出结果                                                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9.6 MetaPipeline：Pipeline 分组

### 9.6.1 MetaPipeline 设计

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp

enum class MetaPipelineType : uint8_t {
    REGULAR = 0,   // 普通 Pipeline
    JOIN_BUILD = 1 // Join 构建端
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
public:
    MetaPipeline(Executor &executor, PipelineBuildState &state,
                 optional_ptr<PhysicalOperator> sink,
                 MetaPipelineType type = MetaPipelineType::REGULAR);

public:
    Executor &GetExecutor() const;
    PipelineBuildState &GetState() const;
    optional_ptr<PhysicalOperator> GetSink() const;

    //! 获取基础 Pipeline
    shared_ptr<Pipeline> &GetBasePipeline();
    //! 获取所有 Pipeline
    void GetPipelines(vector<shared_ptr<Pipeline>> &result, bool recursive);
    //! 获取子 MetaPipeline
    void GetMetaPipelines(vector<shared_ptr<MetaPipeline>> &result, bool recursive, bool skip);

    //! 构建 MetaPipeline
    void Build(PhysicalOperator &op);
    //! 准备所有 Pipeline
    void Ready() const;

    //! 创建空 Pipeline
    Pipeline &CreatePipeline();
    //! 创建 Union Pipeline
    Pipeline &CreateUnionPipeline(Pipeline &current, bool order_matters);
    //! 创建子 Pipeline
    void CreateChildPipeline(Pipeline &current, PhysicalOperator &op, Pipeline &last_pipeline);
    //! 创建子 MetaPipeline
    MetaPipeline &CreateChildMetaPipeline(Pipeline &current, PhysicalOperator &op,
                                           MetaPipelineType type = MetaPipelineType::REGULAR);

private:
    Executor &executor;
    PipelineBuildState &state;
    optional_ptr<Pipeline> parent;
    optional_ptr<PhysicalOperator> sink;
    MetaPipelineType type;
    bool recursive_cte;

    //! 同一 Sink 的所有 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! Pipeline 间依赖关系
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
    //! 子 MetaPipeline
    vector<shared_ptr<MetaPipeline>> children;

    idx_t next_batch_index;
};
```

### 9.6.2 MetaPipeline 构建规则

```
┌────────────────────────────────────────────────────────────────────┐
│                    MetaPipeline 构建规则                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  规则 1: 阻塞算子分割 Pipeline                                     │
│  ─────────────────────────────                                     │
│                                                                    │
│    Pipeline 1                   Pipeline 2                         │
│    ┌──────────────┐            ┌──────────────┐                   │
│    │  TableScan   │            │  (Scan HT)   │                   │
│    │      ↓       │            │      ↓       │                   │
│    │   Filter     │   ────►    │  Projection  │                   │
│    │      ↓       │   BUILD    │      ↓       │                   │
│    │  HashJoin    │            │    Result    │                   │
│    │   (Build)    │            │              │                   │
│    └──────────────┘            └──────────────┘                   │
│                                                                    │
│  规则 2: Join Build 端创建子 MetaPipeline                          │
│  ──────────────────────────────────────────                        │
│                                                                    │
│    MetaPipeline (Probe)         MetaPipeline (Build)              │
│    ┌──────────────┐            ┌──────────────┐                   │
│    │ Pipeline:    │  depends   │ Pipeline:    │                   │
│    │ Scan → Probe │ ◄───────── │ Scan → Build │                   │
│    └──────────────┘            └──────────────┘                   │
│                                                                    │
│  规则 3: 子 Pipeline 依赖于当前及后续 Pipeline                     │
│  ─────────────────────────────────────────────                     │
│                                                                    │
│    当算子有子 Pipeline（如 FULL OUTER JOIN 的 Scan 阶段）：        │
│    - 子 Pipeline 自动依赖当前 Pipeline                            │
│    - 子 Pipeline 自动依赖所有后添加的 Pipeline                    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9.7 Executor：执行协调器

### 9.7.1 Executor 类

```cpp
// src/include/duckdb/execution/executor.hpp

class Executor {
public:
    static constexpr idx_t WAIT_TIME = 20;

public:
    explicit Executor(ClientContext &context);
    ~Executor();

    ClientContext &context;

public:
    static Executor &Get(ClientContext &context);

    //! 初始化执行计划
    void Initialize(PhysicalOperator &physical_plan);

    //! 取消执行
    void CancelTasks();
    //! 执行一个任务
    PendingExecutionResult ExecuteTask(bool dry_run = false);
    //! 等待任务
    void WaitForTask();

    //! 重置执行器
    void Reset();

    //! 获取输出类型
    vector<LogicalType> GetTypes();

    //! 错误处理
    void PushError(ErrorData exception);
    ErrorData GetError();
    bool HasError();
    void ThrowException();

    //! 工作线程辅助
    void WorkOnTasks();

    //! 重新调度被阻塞的任务
    void RescheduleTask(shared_ptr<Task> &task);
    void AddToBeRescheduled(shared_ptr<Task> &task);

    //! 进度查询
    idx_t GetPipelinesProgress(ProgressData &progress);

    //! Pipeline 完成通知
    void CompletePipeline() { completed_pipelines++; }

    //! 获取生产者令牌
    ProducerToken &GetToken() { return *producer; }

    //! 添加事件
    void AddEvent(shared_ptr<Event> event);

    //! 获取查询结果
    bool HasResultCollector();
    unique_ptr<QueryResult> GetResult();

    //! 检查是否完成
    bool ExecutionIsFinished();

private:
    optional_ptr<PhysicalOperator> physical_plan;

    mutex executor_lock;
    //! 所有 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! 根 Pipeline
    vector<shared_ptr<Pipeline>> root_pipelines;
    //! 递归 CTE
    vector<reference<PhysicalOperator>> recursive_ctes;
    //! 根 Pipeline 执行器
    unique_ptr<PipelineExecutor> root_executor;

    idx_t root_pipeline_idx;
    unique_ptr<ProducerToken> producer;
    vector<shared_ptr<Event>> events;
    shared_ptr<QueryProfiler> profiler;
    TaskErrorManager error_manager;

    atomic<idx_t> completed_pipelines;
    idx_t total_pipelines;
    bool cancelled;

    PendingExecutionResult execution_result;
    shared_ptr<Task> task;

    unordered_map<Task *, shared_ptr<Task>> to_be_rescheduled_tasks;
    std::condition_variable task_reschedule;
    atomic<idx_t> executor_tasks;
    atomic<idx_t> blocked_thread_time;
};
```

### 9.7.2 执行流程

```cpp
void Executor::Initialize(PhysicalOperator &physical_plan) {
    // 构建 Pipeline 树
    auto meta_pipeline = make_shared<MetaPipeline>(*this, state, nullptr);
    meta_pipeline->Build(physical_plan);

    // 收集所有 Pipeline
    vector<shared_ptr<MetaPipeline>> all_meta_pipelines;
    meta_pipeline->GetMetaPipelines(all_meta_pipelines, true, false);

    // 调度事件
    ScheduleEvents(all_meta_pipelines);
}

PendingExecutionResult Executor::ExecuteTask(bool dry_run) {
    if (cancelled) {
        return PendingExecutionResult::EXECUTION_ERROR;
    }

    // 处理待重新调度的任务
    lock_guard<mutex> guard(executor_lock);
    if (!to_be_rescheduled_tasks.empty()) {
        for (auto &entry : to_be_rescheduled_tasks) {
            RescheduleTask(entry.second);
        }
        to_be_rescheduled_tasks.clear();
    }

    // 执行当前任务
    if (task) {
        auto result = task->Execute(TaskExecutionMode::PROCESS_ALL);
        switch (result) {
        case TaskExecutionResult::TASK_FINISHED:
            task.reset();
            break;
        case TaskExecutionResult::TASK_NOT_FINISHED:
            // 继续执行
            break;
        case TaskExecutionResult::TASK_BLOCKED:
            task->Deschedule();
            return PendingExecutionResult::BLOCKED_ON_RESULT;
        case TaskExecutionResult::TASK_ERROR:
            return PendingExecutionResult::EXECUTION_ERROR;
        }
    }

    // 获取新任务
    if (!task) {
        if (!TaskScheduler::GetScheduler(context).GetTaskFromProducer(*producer, task)) {
            if (ExecutionIsFinished()) {
                return PendingExecutionResult::EXECUTION_FINISHED;
            }
            return PendingExecutionResult::NO_TASKS_AVAILABLE;
        }
    }

    return PendingExecutionResult::RESULT_READY;
}
```

---

## 9.8 中断与同步

### 9.8.1 InterruptState：中断状态

```cpp
// src/include/duckdb/parallel/interrupt.hpp

enum class InterruptMode : uint8_t {
    NO_INTERRUPTS, // 不支持中断
    TASK,          // 通过 Task 回调
    BLOCKING       // 阻塞等待
};

class InterruptState {
public:
    InterruptState();
    explicit InterruptState(weak_ptr<Task> task);
    explicit InterruptState(weak_ptr<InterruptDoneSignalState> done_signal);

    //! 执行回调（中断完成时调用）
    DUCKDB_API void Callback() const;

protected:
    InterruptMode mode;
    weak_ptr<Task> current_task;
    weak_ptr<InterruptDoneSignalState> signal_state;
};
```

### 9.8.2 StateWithBlockableTasks

```cpp
class StateWithBlockableTasks {
public:
    unique_lock<mutex> Lock() {
        return unique_lock<mutex>(lock);
    }

    //! 阻止后续任务阻塞
    void PreventBlocking(const unique_lock<mutex> &guard) {
        can_block = false;
    }

    //! 阻塞任务
    bool BlockTask(const unique_lock<mutex> &guard, const InterruptState &interrupt_state) {
        if (can_block) {
            blocked_tasks.push_back(interrupt_state);
            return true;
        }
        return false;
    }

    //! 解除所有阻塞
    bool UnblockTasks(const unique_lock<mutex> &guard) {
        if (blocked_tasks.empty()) {
            return false;
        }
        for (auto &entry : blocked_tasks) {
            entry.Callback();
        }
        blocked_tasks.clear();
        return true;
    }

    SinkResultType BlockSink(const unique_lock<mutex> &guard, const InterruptState &interrupt_state) {
        return BlockTask(guard, interrupt_state) ? SinkResultType::BLOCKED : SinkResultType::FINISHED;
    }

    SourceResultType BlockSource(const unique_lock<mutex> &guard, const InterruptState &interrupt_state) {
        return BlockTask(guard, interrupt_state) ? SourceResultType::BLOCKED : SourceResultType::FINISHED;
    }

private:
    atomic<bool> can_block {true};
    mutable mutex lock;
    mutable vector<InterruptState> blocked_tasks;
};
```

```
┌────────────────────────────────────────────────────────────────────┐
│                     中断与恢复流程                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   场景: 异步 I/O 操作                                              │
│                                                                    │
│   ┌────────────┐                     ┌────────────────┐           │
│   │   Task     │                     │ External Event │           │
│   │ (Pipeline) │                     │ (I/O Complete) │           │
│   └─────┬──────┘                     └───────┬────────┘           │
│         │                                    │                     │
│         │ 1. Execute()                       │                     │
│         ▼                                    │                     │
│   ┌────────────┐                             │                     │
│   │  Source::  │                             │                     │
│   │  GetData() │                             │                     │
│   └─────┬──────┘                             │                     │
│         │                                    │                     │
│         │ 2. 需要等待 I/O                    │                     │
│         │    返回 BLOCKED                    │                     │
│         ▼                                    │                     │
│   ┌────────────┐                             │                     │
│   │ BlockTask  │                             │                     │
│   │ (保存状态) │                             │                     │
│   └─────┬──────┘                             │                     │
│         │                                    │                     │
│         │ 3. Deschedule()                    │                     │
│         │    (从队列移除)                    │                     │
│         ▼                                    │                     │
│   ┌────────────┐                             │                     │
│   │   IDLE     │                             │                     │
│   │ (等待回调) │◄────────────────────────────┤                     │
│   └─────┬──────┘   4. I/O 完成              │                     │
│         │             Callback()             │                     │
│         │                                    │                     │
│         │ 5. Reschedule()                    │                     │
│         │    (重新入队)                      │                     │
│         ▼                                    │                     │
│   ┌────────────┐                             │                     │
│   │  QUEUED    │                             │                     │
│   │ (等待执行) │                             │                     │
│   └─────┬──────┘                             │                     │
│         │                                    │                     │
│         │ 6. Execute() 继续                  │                     │
│         ▼                                    │                     │
│   ┌────────────┐                             │                     │
│   │ RUNNING    │                             │                     │
│   └────────────┘                             │                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 9.9 Morsel-Driven 并行

### 9.9.1 概念

DuckDB 采用 Morsel-Driven 并行模型，将数据划分为固定大小的 Morsel（通常为 VECTOR_SIZE = 2048 行），动态分配给工作线程。

```
┌────────────────────────────────────────────────────────────────────┐
│                    Morsel-Driven 并行模型                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                       数据源                                 │ │
│   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │ │
│   │  │ M1   │ │ M2   │ │ M3   │ │ M4   │ │ M5   │ │ M6   │    │ │
│   │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘    │ │
│   └─────┼────────┼────────┼────────┼────────┼────────┼──────────┘ │
│         │        │        │        │        │        │            │
│         ▼        ▼        ▼        ▼        ▼        ▼            │
│   ┌───────────────────────────────────────────────────────────┐   │
│   │                GlobalSourceState                           │   │
│   │  • 维护 next_morsel 指针                                   │   │
│   │  • 线程安全的 morsel 分配                                  │   │
│   └───────────────────────────────────────────────────────────┘   │
│         │        │        │        │                              │
│         ▼        ▼        ▼        ▼                              │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                │
│   │Thread 1 │ │Thread 2 │ │Thread 3 │ │Thread 4 │                │
│   │  M1→M5  │ │  M2→M6  │ │   M3    │ │   M4    │                │
│   └─────────┘ └─────────┘ └─────────┘ └─────────┘                │
│                                                                    │
│   优点:                                                           │
│   • 动态负载均衡                                                  │
│   • 避免数据倾斜问题                                              │
│   • 线程不会空等                                                  │
│   • 简单的同步机制（原子操作）                                    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.9.2 实现示例

```cpp
// 典型的 GlobalSourceState 实现
class TableScanGlobalSourceState : public GlobalSourceState {
public:
    atomic<idx_t> current_row_group;
    idx_t total_row_groups;

    idx_t MaxThreads() override {
        return total_row_groups;
    }
};

// 获取下一个 Morsel
bool GetNextMorsel(TableScanGlobalSourceState &gstate, idx_t &row_group_idx) {
    row_group_idx = gstate.current_row_group.fetch_add(1);
    return row_group_idx < gstate.total_row_groups;
}
```

---

## 9.10 并行度控制

### 9.10.1 并行度决定因素

```
┌────────────────────────────────────────────────────────────────────┐
│                      并行度决定因素                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   1. 系统配置                                                      │
│      • max_threads 设置                                            │
│      • 可用 CPU 核心数                                             │
│                                                                    │
│   2. 算子特性                                                      │
│      • Source::MaxThreads() 返回值                                 │
│      • Sink::MaxThreads() 返回值                                   │
│      • 是否支持并行                                                │
│                                                                    │
│   3. 数据特性                                                      │
│      • 数据量（行数）                                              │
│      • 分区/分段数量                                               │
│      • Row Group 数量                                              │
│                                                                    │
│   4. 运行时限制                                                    │
│      • 内存压力                                                    │
│      • 其他查询竞争                                                │
│                                                                    │
│   最终并行度 = min(                                                │
│       system_max_threads,                                          │
│       source_max_threads,                                          │
│       available_threads                                            │
│   )                                                                │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 9.10.2 有序输出保证

某些算子需要保证输出顺序（如带 ORDER BY 的窗口函数）。DuckDB 使用批次索引（Batch Index）机制：

```cpp
class Pipeline {
    idx_t base_batch_index = 0;
    mutex batch_lock;
    multiset<idx_t> batch_indexes;

public:
    idx_t RegisterNewBatchIndex() {
        lock_guard<mutex> guard(batch_lock);
        // 获取当前最小批次索引
        idx_t result = base_batch_index;
        if (!batch_indexes.empty()) {
            result = *batch_indexes.begin();
        }
        // 注册新的批次索引
        batch_indexes.insert(result);
        return result;
    }

    idx_t UpdateBatchIndex(idx_t old_index, idx_t new_index) {
        lock_guard<mutex> guard(batch_lock);
        batch_indexes.erase(batch_indexes.find(old_index));
        batch_indexes.insert(new_index);
        return *batch_indexes.begin();
    }
};
```

---

## 9.11 源文件索引

| 组件 | 文件路径 |
|------|----------|
| TaskScheduler | `src/parallel/task_scheduler.cpp` |
| Task | `src/include/duckdb/parallel/task.hpp` |
| Pipeline | `src/include/duckdb/parallel/pipeline.hpp` |
| PipelineExecutor | `src/include/duckdb/parallel/pipeline_executor.hpp` |
| MetaPipeline | `src/include/duckdb/parallel/meta_pipeline.hpp` |
| Executor | `src/include/duckdb/execution/executor.hpp` |
| Event | `src/include/duckdb/parallel/event.hpp` |
| InterruptState | `src/include/duckdb/parallel/interrupt.hpp` |
| ExecutorTask | `src/include/duckdb/parallel/executor_task.hpp` |

---

## 9.12 本章小结

本章深入分析了 DuckDB 的并行执行框架：

1. **TaskScheduler** 是并行执行的核心，管理任务队列和工作线程
   - 使用无锁并发队列（moodycamel）
   - 信号量机制唤醒空闲线程
   - 支持任务的阻塞、取消调度和重新调度

2. **Pipeline** 是执行的基本单元
   - 由 Source → Operators → Sink 组成
   - 支持并行扫描（多任务）
   - 通过批次索引保证有序输出

3. **MetaPipeline** 管理 Pipeline 间的依赖关系
   - 处理阻塞算子（Join Build、Aggregate 等）
   - 协调多个 Pipeline 的执行顺序

4. **Event 系统** 实现事件驱动的执行调度
   - 支持依赖关系
   - 任务完成时自动触发后续事件

5. **Executor** 协调整个查询的执行
   - 构建 Pipeline 树
   - 处理错误和取消
   - 收集查询结果

6. **中断机制** 支持异步 I/O
   - InterruptState 管理中断状态
   - StateWithBlockableTasks 支持任务阻塞和恢复

7. **Morsel-Driven** 并行模型实现动态负载均衡
   - 数据按 Morsel 划分
   - 线程动态获取工作单元
   - 避免数据倾斜问题

DuckDB 的并行执行框架设计精良，通过 Pipeline 模型、事件系统和 Morsel-Driven 并行实现了高效的多核利用，同时保持了良好的扩展性和可维护性。

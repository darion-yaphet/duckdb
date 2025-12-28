# DuckDB 执行引擎深度解析：第二章 Pipeline 与调度

## 2.1 Pipeline 详解

### 2.1.1 Pipeline 完整结构

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class Pipeline : public enable_shared_from_this<Pipeline> {
private:
    //! Pipeline 就绪状态
    bool ready;
    //! Pipeline 初始化状态
    atomic<bool> initialized;

    //! 数据源算子
    optional_ptr<PhysicalOperator> source;
    //! 中间算子链（执行顺序）
    vector<reference<PhysicalOperator>> operators;
    //! 数据汇/Sink 算子
    optional_ptr<PhysicalOperator> sink;

    //! 全局 Source 状态
    unique_ptr<GlobalSourceState> source_state;

    //! 父 Pipeline（依赖当前 Pipeline）
    vector<weak_ptr<Pipeline>> parents;
    //! 依赖的 Pipeline
    vector<weak_ptr<Pipeline>> dependencies;

    //! 批次索引基准值
    idx_t base_batch_index = 0;
    //! 批次索引锁
    mutex batch_lock;
    //! 活跃的批次索引集合
    multiset<idx_t> batch_indexes;

public:
    Executor &executor;

    void Ready();     // 准备就绪
    void Reset();     // 重置状态
    void Schedule(shared_ptr<Event> &event);  // 调度执行
    bool GetProgress(ProgressData &progress); // 获取进度
};
```

### 2.1.2 Pipeline 生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pipeline 生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 构建阶段 (BuildPipelines)                                   │
│     │                                                           │
│     ├─▶ 创建 Pipeline 对象                                      │
│     ├─▶ 设置 Source、Operators、Sink                           │
│     └─▶ 建立依赖关系                                            │
│                                                                 │
│  2. Ready 阶段                                                  │
│     │                                                           │
│     ├─▶ 反转 operators 列表（从 Sink 向 Source 顺序 → 执行顺序） │
│     └─▶ 设置 ready = true                                      │
│                                                                 │
│  3. Reset 阶段                                                  │
│     │                                                           │
│     ├─▶ 初始化 Sink 全局状态 (sink_state)                       │
│     ├─▶ 初始化各 Operator 全局状态 (op_state)                   │
│     ├─▶ 初始化 Source 全局状态 (source_state)                   │
│     └─▶ 设置 initialized = true                                │
│                                                                 │
│  4. Schedule 阶段                                               │
│     │                                                           │
│     ├─▶ 尝试并行调度 (ScheduleParallel)                         │
│     │   └─▶ 检查所有算子是否支持并行                             │
│     └─▶ 否则串行调度 (ScheduleSequentialTask)                   │
│                                                                 │
│  5. Execute 阶段                                                │
│     │                                                           │
│     └─▶ PipelineExecutor 执行 Pipeline                         │
│                                                                 │
│  6. Finalize 阶段                                               │
│     │                                                           │
│     ├─▶ PrepareFinalize()                                      │
│     └─▶ Sink::Finalize()                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1.3 Pipeline 调度决策

```cpp
// src/parallel/pipeline.cpp
void Pipeline::Schedule(shared_ptr<Event> &event) {
    D_ASSERT(ready);
    D_ASSERT(sink);
    Reset();  // 初始化所有状态

    if (!ScheduleParallel(event)) {
        // 无法并行化：创建串行任务
        ScheduleSequentialTask(event);
    }
}

bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // 检查 Sink 是否支持并行
    if (!sink->ParallelSink()) {
        return false;
    }
    // 检查 Source 是否支持并行
    if (!source->ParallelSource()) {
        return false;
    }

    // 计算最大线程数
    auto max_threads = source_state->MaxThreads();

    // 检查每个中间算子
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (!op.ParallelOperator()) {
            return false;
        }
        max_threads = MinValue(max_threads, op.op_state->MaxThreads(max_threads));
    }

    // 限制为可用线程数
    auto &scheduler = TaskScheduler::GetScheduler(executor.context);
    auto active_threads = scheduler.NumberOfThreads();
    max_threads = MinValue(max_threads, active_threads);

    // Sink 状态可能进一步限制线程数
    if (sink && sink->sink_state) {
        max_threads = sink->sink_state->MaxThreads(max_threads);
    }

    return LaunchScanTasks(event, max_threads);
}
```

### 2.1.4 并行任务创建

```cpp
bool Pipeline::LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads) {
    if (max_threads <= 1) {
        return false;  // 单线程不值得并行化
    }

    // 为每个线程创建一个 PipelineTask
    vector<shared_ptr<Task>> tasks;
    for (idx_t i = 0; i < max_threads; i++) {
        tasks.push_back(make_uniq<PipelineTask>(*this, event));
    }
    event->SetTasks(std::move(tasks));
    return true;
}
```

---

## 2.2 PipelineTask 执行

### 2.2.1 PipelineTask 结构

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class PipelineTask : public ExecutorTask {
    static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

public:
    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override;
};
```

### 2.2.2 任务执行逻辑

```cpp
// src/parallel/pipeline.cpp
TaskExecutionResult PipelineTask::ExecuteTask(TaskExecutionMode mode) {
    // 延迟创建 PipelineExecutor（每个任务独立）
    if (!pipeline_executor) {
        pipeline_executor = make_uniq<PipelineExecutor>(
            pipeline.GetClientContext(), pipeline);
    }

    // 设置中断回调
    pipeline_executor->SetTaskForInterrupts(shared_from_this());

    if (mode == TaskExecutionMode::PROCESS_PARTIAL) {
        // 部分执行模式：处理固定数量的 Chunk
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

    // 完成任务
    event->FinishTask();
    pipeline_executor.reset();
    return TaskExecutionResult::TASK_FINISHED;
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  PipelineTask 执行模式                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROCESS_ALL (完整执行)                                          │
│  ─────────────────────                                          │
│  - 执行直到 Pipeline 完成                                        │
│  - 用于非交互式查询                                              │
│  - 单次调用完成所有工作                                          │
│                                                                 │
│  PROCESS_PARTIAL (部分执行)                                      │
│  ─────────────────────                                          │
│  - 每次执行固定数量 Chunk (PARTIAL_CHUNK_COUNT = 50)            │
│  - 用于交互式查询和流式结果                                      │
│  - 支持进度报告和取消                                            │
│  - 返回 NOT_FINISHED 表示需要继续                               │
│                                                                 │
│  BLOCKED (阻塞)                                                  │
│  ─────────────────────                                          │
│  - 异步 I/O 等待                                                 │
│  - 任务被挂起，等待回调唤醒                                      │
│  - 支持非阻塞执行                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Event 事件系统

### 2.3.1 Event 基类

```cpp
// src/include/duckdb/parallel/event.hpp
class Event : public enable_shared_from_this<Event> {
protected:
    Executor &executor;

    //! 已完成任务数
    atomic<idx_t> finished_tasks;
    //! 总任务数
    atomic<idx_t> total_tasks;

    //! 已完成的依赖数
    atomic<idx_t> finished_dependencies;
    //! 总依赖数
    idx_t total_dependencies;

    //! 依赖当前事件的父事件
    vector<weak_ptr<Event>> parents;

    //! 事件是否已完成
    atomic<bool> finished;

public:
    //! 调度事件（创建和提交任务）
    virtual void Schedule() = 0;

    //! 事件完成后立即调用
    virtual void FinishEvent() {}

    //! 事件完全结束后调用
    virtual void FinalizeFinish() {}

    //! 任务完成回调
    void FinishTask();

    //! 添加依赖
    void AddDependency(Event &event);

    //! 依赖完成回调
    void CompleteDependency();

    //! 设置任务列表
    void SetTasks(vector<shared_ptr<Task>> tasks);
};
```

### 2.3.2 Pipeline 事件类型

DuckDB 为每个 Pipeline 创建五种事件：

```
┌─────────────────────────────────────────────────────────────────┐
│                   Pipeline 事件链                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │     PipelineInitializeEvent         │                       │
│  │     ─────────────────────────       │                       │
│  │  - 初始化 Source 和 Sink 状态        │                       │
│  │  - 准备 Pipeline 执行               │                       │
│  └────────────────┬────────────────────┘                       │
│                   │ depends on                                 │
│                   ▼                                            │
│  ┌─────────────────────────────────────┐                       │
│  │         PipelineEvent               │                       │
│  │     ─────────────────────────       │                       │
│  │  - 创建 PipelineTask                │                       │
│  │  - 调度到 TaskScheduler             │                       │
│  │  - 执行 Pipeline 主逻辑             │                       │
│  └────────────────┬────────────────────┘                       │
│                   │ depends on                                 │
│                   ▼                                            │
│  ┌─────────────────────────────────────┐                       │
│  │     PipelinePrepareFinishEvent      │                       │
│  │     ─────────────────────────       │                       │
│  │  - 调用 Sink::PrepareFinalize()     │                       │
│  │  - 准备内存清理等                   │                       │
│  └────────────────┬────────────────────┘                       │
│                   │ depends on                                 │
│                   ▼                                            │
│  ┌─────────────────────────────────────┐                       │
│  │       PipelineFinishEvent           │                       │
│  │     ─────────────────────────       │                       │
│  │  - 调用 Sink::Finalize()            │                       │
│  │  - 完成 Sink 端处理                 │                       │
│  └────────────────┬────────────────────┘                       │
│                   │ depends on                                 │
│                   ▼                                            │
│  ┌─────────────────────────────────────┐                       │
│  │      PipelineCompleteEvent          │                       │
│  │     ─────────────────────────       │                       │
│  │  - 标记 Pipeline 完成               │                       │
│  │  - 触发依赖的 Pipeline              │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3.3 事件实现

```cpp
// PipelineInitializeEvent
class PipelineInitializeEvent : public BasePipelineEvent {
public:
    void Schedule() override {
        // 在主线程初始化
        pipeline->Reset();
    }

    void FinishEvent() override {
        // 初始化完成
    }
};

// PipelineEvent
class PipelineEvent : public BasePipelineEvent {
public:
    void Schedule() override {
        // 调度 Pipeline 执行
        pipeline->Schedule(shared_from_this());
    }

    void FinishEvent() override {
        // 所有任务完成
    }
};

// PipelineFinishEvent
class PipelineFinishEvent : public BasePipelineEvent {
public:
    void Schedule() override {
        // 无需调度任务
    }

    void FinishEvent() override {
        // 调用 Finalize
        auto &sink = *pipeline->GetSink();
        sink.Finalize(*pipeline, *this, context, input);
    }
};
```

### 2.3.4 事件依赖管理

```cpp
// src/parallel/event.cpp
void Event::AddDependency(Event &event) {
    total_dependencies++;
    event.parents.push_back(weak_from_this());
}

void Event::CompleteDependency() {
    idx_t current = ++finished_dependencies;
    D_ASSERT(googletypename_finished_dependencies <= total_dependencies);
    if (current == total_dependencies) {
        // 所有依赖完成，开始调度
        executor.AddEvent(shared_from_this());
        Schedule();
    }
}

void Event::FinishTask() {
    idx_t current = ++finished_tasks;
    if (current == total_tasks) {
        // 所有任务完成
        Finish();
    }
}

void Event::Finish() {
    finished = true;
    FinishEvent();

    // 通知所有父事件
    for (auto &parent_ref : parents) {
        auto parent = parent_ref.lock();
        if (parent) {
            parent->CompleteDependency();
        }
    }

    FinalizeFinish();
}
```

---

## 2.4 MetaPipeline 管理

### 2.4.1 MetaPipeline 结构

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp
enum class MetaPipelineType : uint8_t {
    REGULAR = 0,    // 普通 MetaPipeline
    JOIN_BUILD = 1  // Join Build 端
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
private:
    Executor &executor;
    PipelineBuildState &state;

    //! 父 Pipeline（可选）
    optional_ptr<Pipeline> parent;
    //! 共享的 Sink 算子
    optional_ptr<PhysicalOperator> sink;
    //! MetaPipeline 类型
    MetaPipelineType type;
    //! 是否是递归 CTE
    bool recursive_cte;

    //! 所有共享该 Sink 的 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! Pipeline 依赖关系
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
    //! 子 MetaPipeline
    vector<shared_ptr<MetaPipeline>> children;
    //! 下一个批次索引
    idx_t next_batch_index;

public:
    //! 构建 MetaPipeline
    void Build(PhysicalOperator &op);

    //! 创建 Pipeline
    Pipeline &CreatePipeline();
    Pipeline &CreateUnionPipeline(Pipeline &current, bool order_matters);

    //! 创建子 MetaPipeline
    MetaPipeline &CreateChildMetaPipeline(Pipeline &current, PhysicalOperator &op,
                                           MetaPipelineType type);
};
```

### 2.4.2 Pipeline 构建过程

```cpp
void MetaPipeline::Build(PhysicalOperator &op) {
    // 从 Sink 向 Source 遍历物理计划
    op.BuildPipelines(*pipelines[0], *this);
}

// 各算子实现 BuildPipelines
void PhysicalOperator::BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline) {
    // 默认实现：添加到当前 Pipeline 并递归处理子节点
    auto &state = meta_pipeline.GetState();
    state.AddPipelineOperator(current, *this);

    for (auto &child : children) {
        child.get().BuildPipelines(current, meta_pipeline);
    }
}

// HashJoin 的 BuildPipelines
void PhysicalHashJoin::BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline) {
    // 1. 创建 Build 端子 MetaPipeline
    auto &child_meta = meta_pipeline.CreateChildMetaPipeline(
        current, *this, MetaPipelineType::JOIN_BUILD);

    // 2. 构建 Build 端
    children[1].get().BuildPipelines(*child_meta.GetBasePipeline(), child_meta);

    // 3. 继续 Probe 端
    meta_pipeline.GetState().AddPipelineOperator(current, *this);
    children[0].get().BuildPipelines(current, meta_pipeline);
}
```

### 2.4.3 构建示例

```sql
SELECT * FROM t1 JOIN t2 ON ... JOIN t3 ON ... ORDER BY x
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    多表 Join Pipeline 构建                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  物理计划：                                                      │
│                                                                 │
│           Order                                                 │
│             │                                                   │
│          HashJoin2                                              │
│          /      \                                               │
│     HashJoin1   Scan(t3)                                       │
│     /      \                                                    │
│  Scan(t1)  Scan(t2)                                            │
│                                                                 │
│  MetaPipeline 结构：                                             │
│                                                                 │
│  MetaPipeline (sink = Order)                                    │
│  │                                                              │
│  ├── Pipeline 1: Scan(t1) → HashJoin1 Probe → HashJoin2 Probe   │
│  │                                     → [Order]                │
│  │                                                              │
│  ├── Child MetaPipeline (sink = HashJoin1 Build)                │
│  │   └── Pipeline 2: Scan(t2) → [HashJoin1 Build]               │
│  │                                                              │
│  └── Child MetaPipeline (sink = HashJoin2 Build)                │
│      └── Pipeline 3: Scan(t3) → [HashJoin2 Build]               │
│                                                                 │
│  依赖关系：                                                      │
│  Pipeline 1 depends on Pipeline 2 and Pipeline 3               │
│                                                                 │
│  最终 Pipeline：                                                 │
│  MetaPipeline (sink = Result)                                   │
│  └── Pipeline 4: Order Scan → [Result]                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 TaskScheduler 任务调度

### 2.5.1 TaskScheduler 结构

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp
class TaskScheduler {
    constexpr static int64_t TASK_TIMEOUT_USECS = 5000;  // 5ms

private:
    DatabaseInstance &db;

    //! 并发任务队列
    unique_ptr<ConcurrentQueue> queue;

    //! 线程锁
    mutex thread_lock;
    //! 后台工作线程
    vector<unique_ptr<SchedulerThread>> threads;
    //! 线程运行标记
    vector<unique_ptr<atomic<bool>>> markers;

    //! 请求的线程数
    atomic<int32_t> requested_thread_count;
    //! 当前运行的线程数
    atomic<int32_t> current_thread_count;

public:
    //! 创建生产者令牌
    unique_ptr<ProducerToken> CreateProducer();

    //! 调度单个任务
    void ScheduleTask(ProducerToken &producer, shared_ptr<Task> task);
    //! 批量调度任务
    void ScheduleTasks(ProducerToken &producer, vector<shared_ptr<Task>> &tasks);

    //! 从特定生产者获取任务
    bool GetTaskFromProducer(ProducerToken &token, shared_ptr<Task> &task);

    //! 执行任务直到标记为 false
    void ExecuteForever(atomic<bool> *marker);
    //! 执行指定数量的任务
    void ExecuteTasks(idx_t max_tasks);

    //! 设置线程数
    void SetThreads(idx_t total_threads, idx_t external_threads);

    //! 获取线程数
    int32_t NumberOfThreads();
};
```

### 2.5.2 ConcurrentQueue 实现

DuckDB 使用 moodycamel 的 ConcurrentQueue 实现无锁并发队列：

```cpp
// src/parallel/task_scheduler.cpp
struct ConcurrentQueue {
    lightweight_semaphore_t semaphore;   // 轻量级信号量
    concurrent_queue_t q;                 // 并发队列
    atomic<idx_t> tasks_in_queue;        // 队列任务数

    void Enqueue(ProducerToken &token, shared_ptr<Task> task) {
        lock_guard<mutex> producer_lock(token.producer_lock);
        task->token = token;
        if (q.enqueue(token.token->queue_token, std::move(task))) {
            ++tasks_in_queue;
            semaphore.signal();  // 唤醒等待的消费者
        }
    }

    void EnqueueBulk(ProducerToken &token, vector<shared_ptr<Task>> &tasks) {
        lock_guard<mutex> producer_lock(token.producer_lock);
        for (auto &task : tasks) {
            task->token = token;
        }
        if (q.enqueue_bulk(token.token->queue_token,
                           std::make_move_iterator(tasks.begin()),
                           tasks.size())) {
            tasks_in_queue += tasks.size();
            semaphore.signal(tasks.size());
        }
    }

    bool DequeueFromProducer(ProducerToken &token, shared_ptr<Task> &task) {
        lock_guard<mutex> producer_lock(token.producer_lock);
        if (!q.try_dequeue_from_producer(token.token->queue_token, task)) {
            return false;
        }
        --tasks_in_queue;
        return true;
    }

    bool Dequeue(shared_ptr<Task> &task) {
        if (!q.try_dequeue(task)) {
            return false;
        }
        --tasks_in_queue;
        return true;
    }
};
```

### 2.5.3 工作线程执行

```cpp
void TaskScheduler::ExecuteForever(atomic<bool> *marker) {
    while (*marker) {
        shared_ptr<Task> task;
        // 等待信号量（带超时）
        queue->semaphore.wait(TASK_TIMEOUT_USECS);

        // 尝试获取任务
        if (!queue->Dequeue(task)) {
            continue;
        }

        // 执行任务
        auto result = task->Execute(TaskExecutionMode::PROCESS_ALL);

        if (result == TaskExecutionResult::TASK_BLOCKED) {
            // 任务阻塞，等待回调重新调度
            task->Deschedule();
        }
    }
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  TaskScheduler 工作流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer (Executor)           TaskScheduler                    │
│       │                             │                           │
│       │  ScheduleTask(task)         │                           │
│       │ ──────────────────────────▶ │                           │
│       │                             │                           │
│       │           ┌─────────────────┼──────────────────┐        │
│       │           │    ConcurrentQueue                 │        │
│       │           │  ┌──────────────────────────────┐  │        │
│       │           │  │  Task │ Task │ Task │ ...   │  │        │
│       │           │  └──────────────────────────────┘  │        │
│       │           └─────────────────┼──────────────────┘        │
│       │                             │                           │
│       │                             │  signal()                 │
│       │                             ▼                           │
│       │                     ┌───────────────┐                   │
│       │                     │   Semaphore   │                   │
│       │                     └───────────────┘                   │
│       │                             │                           │
│       │                             │  wake up                  │
│       │                             ▼                           │
│       │           ┌─────────────────────────────────────┐       │
│       │           │  Worker Thread 1  │  Worker Thread N│       │
│       │           │     Dequeue()     │     Dequeue()   │       │
│       │           │     Execute()     │     Execute()   │       │
│       │           └─────────────────────────────────────┘       │
│       │                             │                           │
│       │                             │  FinishTask()             │
│       │                             ▼                           │
│       │                     Event::Finish()                     │
│       │                             │                           │
│       │                             ▼                           │
│       │                  CompleteDependency()                   │
│       │                     (trigger next)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Executor 调度流程

### 2.6.1 初始化和调度

```cpp
// src/parallel/executor.cpp
void Executor::Initialize(PhysicalOperator &plan) {
    physical_plan = &plan;

    // 构建 MetaPipeline
    PipelineBuildState state;
    auto root_meta = make_shared<MetaPipeline>(*this, state, nullptr);
    root_meta->Build(plan);

    // 收集所有 Pipeline
    root_meta->GetPipelines(pipelines, true);
    root_meta->GetMetaPipelines(meta_pipelines, true, false);

    // Ready 所有 Pipeline
    root_meta->Ready();

    total_pipelines = pipelines.size();
    completed_pipelines = 0;

    // 调度事件
    ScheduleEvents(meta_pipelines);
}

void Executor::ScheduleEvents(const vector<shared_ptr<MetaPipeline>> &meta_pipelines) {
    ScheduleEventData event_data(meta_pipelines, events, true);
    ScheduleEventsInternal(event_data);

    // 验证事件依赖图无环
    VerifyScheduledEvents(event_data);
}
```

### 2.6.2 SchedulePipeline 事件创建

```cpp
void Executor::SchedulePipeline(const shared_ptr<MetaPipeline> &meta_pipeline,
                                 ScheduleEventData &event_data) {
    auto base_pipeline = meta_pipeline->GetBasePipeline();

    // 为基础 Pipeline 创建完整事件链
    auto base_initialize_event = make_shared<PipelineInitializeEvent>(base_pipeline);
    auto base_event = make_shared<PipelineEvent>(base_pipeline);
    auto base_prepare_finish_event = make_shared<PipelinePrepareFinishEvent>(base_pipeline);
    auto base_finish_event = make_shared<PipelineFinishEvent>(base_pipeline);
    auto base_complete_event = make_shared<PipelineCompleteEvent>(executor, initial_schedule);

    // 建立事件依赖链
    // initialize -> event -> prepare_finish -> finish -> complete
    base_event->AddDependency(*base_initialize_event);
    base_prepare_finish_event->AddDependency(*base_event);
    base_finish_event->AddDependency(*base_prepare_finish_event);
    base_complete_event->AddDependency(*base_finish_event);

    events.push_back(std::move(base_initialize_event));
    events.push_back(std::move(base_event));
    events.push_back(std::move(base_prepare_finish_event));
    events.push_back(std::move(base_finish_event));
    events.push_back(std::move(base_complete_event));

    // 为 MetaPipeline 中的其他 Pipeline 创建事件
    vector<shared_ptr<Pipeline>> pipelines;
    meta_pipeline->GetPipelines(pipelines, false);

    for (idx_t i = 1; i < pipelines.size(); i++) {
        auto &pipeline = pipelines[i];
        auto pipeline_event = make_shared<PipelineEvent>(pipeline);

        // 共享初始化和完成事件
        pipeline_event->AddDependency(*base_initialize_event);
        base_prepare_finish_event->AddDependency(*pipeline_event);

        events.push_back(std::move(pipeline_event));
    }
}
```

### 2.6.3 ExecuteTask 主循环

```cpp
PendingExecutionResult Executor::ExecuteTask(bool dry_run) {
    // 检查错误
    if (HasError()) {
        ThrowException();
    }

    // 获取待执行任务
    shared_ptr<Task> task;
    bool has_task = GetTaskFromProducer(producer, task);

    if (!has_task) {
        // 检查是否有被阻塞的任务需要重新调度
        if (!to_be_rescheduled_tasks.empty()) {
            for (auto &entry : to_be_rescheduled_tasks) {
                RescheduleTask(entry.second);
            }
            to_be_rescheduled_tasks.clear();
        }

        // 检查是否所有 Pipeline 完成
        if (ExecutionIsFinished()) {
            return PendingExecutionResult::RESULT_READY;
        }

        return PendingExecutionResult::NO_TASKS_AVAILABLE;
    }

    // 执行任务
    auto result = task->Execute(TaskExecutionMode::PROCESS_PARTIAL);

    switch (result) {
    case TaskExecutionResult::TASK_FINISHED:
        // 任务完成
        break;
    case TaskExecutionResult::TASK_NOT_FINISHED:
        // 任务未完成，重新调度
        ScheduleTask(producer, task);
        break;
    case TaskExecutionResult::TASK_BLOCKED:
        // 任务阻塞
        AddToBeRescheduled(task);
        break;
    }

    return PendingExecutionResult::RESULT_NOT_READY;
}
```

---

## 2.7 批次索引管理

### 2.7.1 BatchIndex 用途

批次索引用于保证有序输出和分区处理：

```cpp
// Pipeline 批次管理
class Pipeline {
    idx_t base_batch_index = 0;
    mutex batch_lock;
    multiset<idx_t> batch_indexes;  // 活跃批次索引

    idx_t RegisterNewBatchIndex() {
        lock_guard<mutex> l(batch_lock);
        auto min_index = batch_indexes.empty() ? base_batch_index
                                                : *batch_indexes.begin();
        batch_indexes.insert(min_index);
        return min_index;
    }

    idx_t UpdateBatchIndex(idx_t old_index, idx_t new_index) {
        lock_guard<mutex> l(batch_lock);
        batch_indexes.erase(batch_indexes.find(old_index));
        batch_indexes.insert(new_index);
        return *batch_indexes.begin();  // 返回新的最小索引
    }
};
```

### 2.7.2 批次索引使用场景

```
┌─────────────────────────────────────────────────────────────────┐
│                    批次索引使用场景                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 有序输出保证                                                 │
│     ─────────────────                                           │
│     某些 Sink 需要按顺序接收数据（如 ORDER BY + 部分聚合）        │
│     批次索引确保数据按正确顺序处理                                │
│                                                                 │
│  2. 分区写入                                                    │
│     ─────────────────                                           │
│     COPY TO 分区文件时，需要知道当前处理的分区                    │
│     批次索引对应文件分区                                         │
│                                                                 │
│  3. 增量处理                                                    │
│     ─────────────────                                           │
│     NextBatch() 在批次切换时刷新之前的数据                       │
│     确保数据完整性                                               │
│                                                                 │
│  示例：                                                          │
│                                                                 │
│  Thread 1: batch_index = 0 → 1 → 2                             │
│  Thread 2: batch_index = 3 → 4 → 5                             │
│  Thread 3: batch_index = 6 → 7 → 8                             │
│                                                                 │
│  min_batch_index 追踪所有线程中最小的活跃批次                    │
│  当 min_batch_index 增加时，可以安全刷新之前的批次                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.8 进度追踪

### 2.8.1 ProgressData 结构

```cpp
struct ProgressData {
    double done = 0;    // 已完成数量
    double total = 0;   // 总数量

    bool IsValid() const {
        return total >= 0 && done >= 0;
    }

    double GetPercentage() const {
        if (total <= 0) return 0;
        return 100.0 * done / total;
    }

    void Normalize(double cardinality) {
        if (total > 0 && cardinality > 0) {
            done = done * cardinality / total;
            total = cardinality;
        }
    }
};
```

### 2.8.2 进度计算

```cpp
bool Pipeline::GetProgress(ProgressData &progress) {
    // 使用 Source 估计基数
    idx_t source_cardinality = MinValue(source->estimated_cardinality, 1ULL << 48ULL);
    if (source_cardinality < 1) {
        source_cardinality = 1;
    }

    if (!initialized) {
        progress.done = 0;
        progress.total = double(source_cardinality);
        return true;
    }

    // 从 Source 获取进度
    progress = source->GetProgress(context, *source_state);
    progress.Normalize(double(source_cardinality));

    // Sink 可能调整进度
    progress = sink->GetSinkProgress(context, *sink->sink_state, progress);

    return progress.IsValid();
}

idx_t Executor::GetPipelinesProgress(ProgressData &progress) {
    lock_guard<mutex> l(executor_lock);

    progress.done = completed_pipelines.load();
    progress.total = total_pipelines;

    // 累加所有活跃 Pipeline 的进度
    for (auto &pipeline : pipelines) {
        ProgressData pipeline_progress;
        if (pipeline->GetProgress(pipeline_progress)) {
            progress.done += pipeline_progress.GetPercentage() / 100.0;
        }
    }

    return completed_pipelines.load();
}
```

---

## 2.9 源文件索引

| 组件 | 文件路径 |
|------|----------|
| Pipeline | `src/include/duckdb/parallel/pipeline.hpp` |
| Pipeline 实现 | `src/parallel/pipeline.cpp` |
| PipelineExecutor | `src/include/duckdb/parallel/pipeline_executor.hpp` |
| PipelineExecutor 实现 | `src/parallel/pipeline_executor.cpp` |
| MetaPipeline | `src/include/duckdb/parallel/meta_pipeline.hpp` |
| Event 基类 | `src/include/duckdb/parallel/event.hpp` |
| BasePipelineEvent | `src/include/duckdb/parallel/base_pipeline_event.hpp` |
| PipelineEvent | `src/include/duckdb/parallel/pipeline_event.hpp` |
| PipelineInitializeEvent | `src/include/duckdb/parallel/pipeline_initialize_event.hpp` |
| PipelineFinishEvent | `src/include/duckdb/parallel/pipeline_finish_event.hpp` |
| PipelineCompleteEvent | `src/include/duckdb/parallel/pipeline_complete_event.hpp` |
| TaskScheduler | `src/include/duckdb/parallel/task_scheduler.hpp` |
| TaskScheduler 实现 | `src/parallel/task_scheduler.cpp` |
| Executor | `src/include/duckdb/execution/executor.hpp` |
| Executor 实现 | `src/parallel/executor.cpp` |

---

## 2.10 总结

```
┌─────────────────────────────────────────────────────────────────┐
│                Pipeline 调度系统核心特点                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Pipeline 抽象                                               │
│     - Source → Operators → Sink 数据流                         │
│     - 支持并行和串行执行                                         │
│     - 依赖关系管理                                              │
│                                                                 │
│  2. MetaPipeline 层次                                           │
│     - 管理共享 Sink 的多个 Pipeline                             │
│     - 处理 Join Build 等阻塞点                                  │
│     - 递归构建 Pipeline 结构                                    │
│                                                                 │
│  3. 事件驱动调度                                                 │
│     - 五阶段事件链                                              │
│     - 依赖完成触发下游                                          │
│     - 支持异步执行                                              │
│                                                                 │
│  4. TaskScheduler                                               │
│     - 无锁并发队列                                              │
│     - 信号量唤醒机制                                            │
│     - 工作窃取模式                                              │
│                                                                 │
│  5. 批次索引                                                    │
│     - 有序输出保证                                              │
│     - 分区处理支持                                              │
│     - 增量刷新机制                                              │
│                                                                 │
│  6. 进度追踪                                                    │
│     - Pipeline 级别进度                                        │
│     - Source/Sink 进度贡献                                     │
│     - 基数估计归一化                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

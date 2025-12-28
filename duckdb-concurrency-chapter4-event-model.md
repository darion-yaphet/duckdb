# 第四章：Event 事件驱动模型

## 概述

DuckDB 的并行执行采用事件驱动模型来协调任务之间的依赖关系。Event 是任务同步的核心抽象，通过依赖计数机制实现事件的有序触发。Pipeline 的各个阶段（初始化、执行、完成）都通过事件来协调。

本章深入分析 Event 基类的设计、依赖管理机制、Pipeline 事件类型以及事件调度流程。

## Event 事件基类

### 核心结构

```cpp
// src/include/duckdb/parallel/event.hpp
class Event : public enable_shared_from_this<Event> {
public:
    explicit Event(Executor &executor);
    virtual ~Event() = default;

    //! 调度事件（创建任务）
    virtual void Schedule() = 0;
    //! 事件完成后立即调用
    virtual void FinishEvent() {}
    //! 事件彻底完成后调用
    virtual void FinalizeFinish() {}

    void FinishTask();
    void Finish();
    void AddDependency(Event &event);
    void CompleteDependency();
    void SetTasks(vector<shared_ptr<Task>> tasks);
    void InsertEvent(shared_ptr<Event> replacement_event);

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

    //! 依赖此事件的父事件列表
    vector<weak_ptr<Event>> parents;

    //! 事件是否已完成
    atomic<bool> finished;
};
```

### 事件生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Event 生命周期                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐                                                │
│  │     创建        │                                                │
│  │ (构造函数)      │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐     ┌─────────────────────┐                    │
│  │  添加依赖       │────►│ total_dependencies++ │                    │
│  │ (AddDependency) │     └─────────────────────┘                    │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │   等待依赖      │◄─── finished_dependencies < total_dependencies │
│  └────────┬────────┘                                                │
│           │                                                         │
│           │ CompleteDependency() 被依赖事件调用                      │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │   Schedule()    │◄─── finished_dependencies == total_dependencies│
│  │   创建任务      │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │   执行任务      │◄─── 任务被 TaskScheduler 调度执行               │
│  └────────┬────────┘                                                │
│           │                                                         │
│           │ FinishTask() 每个任务完成时调用                          │
│           │                                                         │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │   Finish()      │◄─── finished_tasks == total_tasks              │
│  │   事件完成      │                                                │
│  └────────┬────────┘                                                │
│           │                                                         │
│           ├──► FinishEvent()    事件完成回调                         │
│           ├──► 通知父事件 CompleteDependency()                       │
│           └──► FinalizeFinish() 最终清理                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 依赖管理机制

### 添加依赖

```cpp
// src/parallel/event.cpp
void Event::AddDependency(Event &event) {
    total_dependencies++;
    event.parents.push_back(weak_ptr<Event>(shared_from_this()));
#ifdef DEBUG
    event.parents_raw.push_back(*this);
#endif
}
```

依赖关系是反向建立的：
- A 依赖 B：调用 `A.AddDependency(B)`
- 结果：B 的 parents 列表包含 A

```
┌─────────────────────────────────────────────────────────────┐
│                    依赖关系示意                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  A.AddDependency(B)  意味着：                                │
│                                                             │
│      ┌───────┐                   ┌───────┐                  │
│      │ Event │      依赖于       │ Event │                  │
│      │   A   │ ◄───────────────  │   B   │                  │
│      └───────┘                   └───────┘                  │
│          ▲                           │                      │
│          │                           │                      │
│          │    B.parents.push(A)      │                      │
│          │                           │                      │
│          └───────────────────────────┘                      │
│                                                             │
│  B 完成后会通知 A：A.CompleteDependency()                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 完成依赖

```cpp
void Event::CompleteDependency() {
    idx_t current_finished = ++finished_dependencies;
    D_ASSERT(current_finished <= total_dependencies);
    if (current_finished == total_dependencies) {
        // 所有依赖已完成：调度此事件
        D_ASSERT(total_tasks == 0);
        Schedule();
        if (total_tasks == 0) {
            Finish();
        }
    }
}
```

关键逻辑：
1. 原子递增 `finished_dependencies`
2. 当所有依赖完成时，调用 `Schedule()`
3. 如果 Schedule 没有创建任务，直接完成事件

### 事件完成

```cpp
void Event::Finish() {
    D_ASSERT(!finished);
    FinishEvent();               // 1. 事件完成回调
    finished = true;
    // 通知所有父事件
    for (auto &parent_entry : parents) {
        auto parent = parent_entry.lock();
        if (!parent) {
            continue;
        }
        parent->CompleteDependency();  // 2. 通知父事件
    }
    FinalizeFinish();            // 3. 最终清理
}
```

事件完成触发链式反应：
- 调用 FinishEvent 回调
- 通知所有依赖此事件的父事件
- 调用 FinalizeFinish 最终清理

### 任务完成

```cpp
void Event::FinishTask() {
    D_ASSERT(finished_tasks.load() < total_tasks.load());
    idx_t current_tasks = total_tasks;
    idx_t current_finished = ++finished_tasks;
    D_ASSERT(current_finished <= current_tasks);
    if (current_finished == current_tasks) {
        Finish();
    }
}
```

每个任务完成时调用 FinishTask，当所有任务完成时触发事件完成。

## 任务调度

### SetTasks

```cpp
void Event::SetTasks(vector<shared_ptr<Task>> tasks) {
    auto &ts = TaskScheduler::GetScheduler(executor.context);
    D_ASSERT(total_tasks == 0);
    D_ASSERT(!tasks.empty());
    this->total_tasks = tasks.size();
    ts.ScheduleTasks(executor.GetToken(), tasks);
}
```

SetTasks 将任务提交到 TaskScheduler：
1. 设置总任务数
2. 批量提交任务

### 动态插入事件

```cpp
void Event::InsertEvent(shared_ptr<Event> replacement_event) {
    replacement_event->parents = std::move(parents);
#ifdef DEBUG
    replacement_event->parents_raw = std::move(parents_raw);
#endif
    replacement_event->AddDependency(*this);
    executor.AddEvent(std::move(replacement_event));
}
```

InsertEvent 允许在事件完成后动态插入新事件：

```
┌─────────────────────────────────────────────────────────────┐
│                 InsertEvent 示意                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  调用前:                                                    │
│                                                             │
│      ┌───────┐           ┌───────┐                          │
│      │ Parent│◄──────────│Current│                          │
│      └───────┘           └───────┘                          │
│                                                             │
│  current.InsertEvent(new) 后:                               │
│                                                             │
│      ┌───────┐           ┌───────┐           ┌───────┐      │
│      │ Parent│◄──────────│  New  │◄──────────│Current│      │
│      └───────┘           └───────┘           └───────┘      │
│                                                             │
│  New 事件接管 Current 的父事件列表                           │
│  New 依赖 Current                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Pipeline 事件类型

### BasePipelineEvent

```cpp
// src/include/duckdb/parallel/base_pipeline_event.hpp
class BasePipelineEvent : public Event {
public:
    explicit BasePipelineEvent(shared_ptr<Pipeline> pipeline);
    explicit BasePipelineEvent(Pipeline &pipeline);

    void PrintPipeline() override {
        pipeline->Print();
    }

    //! 所属管道
    shared_ptr<Pipeline> pipeline;
};
```

BasePipelineEvent 是所有管道相关事件的基类，持有对 Pipeline 的引用。

### PipelineInitializeEvent - 初始化事件

```cpp
// src/parallel/pipeline_initialize_event.cpp
class PipelineInitializeTask : public ExecutorTask {
public:
    explicit PipelineInitializeTask(Pipeline &pipeline_p, shared_ptr<Event> event_p)
        : ExecutorTask(pipeline_p.executor, std::move(event_p)), pipeline(pipeline_p) {
    }

    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
        pipeline.ResetSink();
        event->FinishTask();
        return TaskExecutionResult::TASK_FINISHED;
    }
};

void PipelineInitializeEvent::Schedule() {
    vector<shared_ptr<Task>> tasks;
    tasks.push_back(make_uniq<PipelineInitializeTask>(*pipeline, shared_from_this()));
    SetTasks(std::move(tasks));
}
```

初始化事件：
- 创建单个 PipelineInitializeTask
- 任务执行 `pipeline.ResetSink()` 重置 Sink 状态
- 为后续执行做准备

### PipelineEvent - 执行事件

```cpp
// src/parallel/pipeline_event.cpp
void PipelineEvent::Schedule() {
    auto event = shared_from_this();
    auto &executor = pipeline->executor;
    try {
        pipeline->Schedule(event);
        D_ASSERT(total_tasks > 0);
    } catch (std::exception &ex) {
        executor.PushError(ErrorData(ex));
    } catch (...) {
        executor.PushError(ErrorData("Unknown exception while calling pipeline->Schedule(event)!"));
    }
}

void PipelineEvent::FinishEvent() {
    // 空实现
}
```

执行事件：
- 调用 `pipeline->Schedule(event)` 创建并行任务
- 任务数量取决于并行度设置
- 每个任务处理一部分数据

### PipelineFinishEvent - 完成事件

```cpp
// src/parallel/pipeline_finish_event.cpp
class PipelineFinishTask : public ExecutorTask {
    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
        auto sink = pipeline.GetSink();
        InterruptState interrupt_state(shared_from_this());

        // 对中间算子调用 Finalize
        for (; operator_idx < operators.size(); operator_idx++) {
            auto &op = operators[operator_idx].get();
            if (!op.RequiresOperatorFinalize()) {
                continue;
            }
            OperatorFinalizeInput op_finalize_input {*op.op_state, interrupt_state};
            auto op_state = op.OperatorFinalize(pipeline, *event, executor.context, op_finalize_input);
            if (op_state == OperatorFinalResultType::BLOCKED) {
                return TaskExecutionResult::TASK_BLOCKED;
            }
        }

        // 对 Sink 调用 Finalize
        OperatorSinkFinalizeInput finalize_input {*sink->sink_state, interrupt_state};
        auto sink_state = sink->Finalize(pipeline, *event, executor.context, finalize_input);

        if (sink_state == SinkFinalizeType::BLOCKED) {
            return TaskExecutionResult::TASK_BLOCKED;
        }

        sink->sink_state->state = sink_state;
        event->FinishTask();
        return TaskExecutionResult::TASK_FINISHED;
    }
};

void PipelineFinishEvent::Schedule() {
    vector<shared_ptr<Task>> tasks;
    tasks.push_back(make_uniq<PipelineFinishTask>(*pipeline, shared_from_this()));
    SetTasks(std::move(tasks));
}
```

完成事件：
- 创建单个 PipelineFinishTask
- 对所有中间算子调用 Finalize
- 对 Sink 调用 Finalize
- 支持异步阻塞（返回 TASK_BLOCKED）

## 事件调度流程

### 事件链构建

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pipeline 事件链                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐                                                │
│  │ InitializeEvent │◄─── 无依赖，首先执行                            │
│  └────────┬────────┘                                                │
│           │                                                         │
│           │ 完成后触发                                               │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │  PipelineEvent  │◄─── 依赖 InitializeEvent                       │
│  │  (并行执行)      │     创建多个并行任务                           │
│  └────────┬────────┘                                                │
│           │                                                         │
│           │ 所有任务完成后触发                                        │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │  FinishEvent    │◄─── 依赖 PipelineEvent                         │
│  │  (Finalize)     │     单任务执行                                  │
│  └────────┬────────┘                                                │
│           │                                                         │
│           │ 完成后触发后续管道                                        │
│           ▼                                                         │
│  ┌─────────────────┐                                                │
│  │ 下一个 Pipeline │                                                │
│  └─────────────────┘                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ScheduleEvents 实现

```cpp
// src/parallel/executor.cpp
void Executor::ScheduleEventsInternal(ScheduleEventData &event_data) {
    auto &events = event_data.events;

    // 为每个 MetaPipeline 创建事件
    for (auto &meta_pipeline : event_data.meta_pipelines) {
        // 创建 Initialize、Execute、Finish 事件
        // 建立依赖关系
        // ...
    }
}

void Executor::ScheduleEvents(const vector<shared_ptr<MetaPipeline>> &meta_pipelines) {
    ScheduleEventData event_data(meta_pipelines, events, true);
    ScheduleEventsInternal(event_data);
}
```

### 添加事件

```cpp
void Executor::AddEvent(shared_ptr<Event> event) {
    lock_guard<mutex> elock(executor_lock);
    if (cancelled) {
        return;
    }
    events.push_back(std::move(event));
}
```

事件列表由 Executor 持有并管理生命周期。

## 并发安全

### 原子操作

Event 使用原子变量保证线程安全：

```cpp
atomic<idx_t> finished_tasks;      // 已完成任务数
atomic<idx_t> total_tasks;         // 总任务数
atomic<idx_t> finished_dependencies; // 已完成依赖数
atomic<bool> finished;             // 事件是否完成
```

### 弱引用

使用 `weak_ptr` 持有父事件引用：

```cpp
vector<weak_ptr<Event>> parents;
```

原因：
- 避免循环引用导致内存泄漏
- 父事件可能提前销毁

在通知父事件时需要检查：

```cpp
for (auto &parent_entry : parents) {
    auto parent = parent_entry.lock();
    if (!parent) {
        continue;  // 父事件已销毁
    }
    parent->CompleteDependency();
}
```

## 事件类型一览

| 事件类型 | 任务数 | 功能 | 阻塞支持 |
|---------|--------|------|---------|
| PipelineInitializeEvent | 1 | 重置 Sink 状态 | 否 |
| PipelineEvent | N | 并行数据处理 | 是 |
| PipelineFinishEvent | 1 | Finalize 算子 | 是 |
| PipelineCompleteEvent | 1 | 收尾工作 | 是 |

## 依赖图示例

考虑一个有两个阶段的查询：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    复杂查询事件依赖图                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│         Pipeline 1                      Pipeline 2                  │
│  ┌─────────────────────┐         ┌─────────────────────┐            │
│  │                     │         │                     │            │
│  │  ┌──────────────┐   │         │  ┌──────────────┐   │            │
│  │  │ Initialize 1 │   │         │  │ Initialize 2 │   │            │
│  │  └──────┬───────┘   │         │  └──────┬───────┘   │            │
│  │         │           │         │         │           │            │
│  │         ▼           │         │         ▼           │            │
│  │  ┌──────────────┐   │         │  ┌──────────────┐   │            │
│  │  │  Execute 1   │   │         │  │  Execute 2   │   │            │
│  │  └──────┬───────┘   │         │  └──────┬───────┘   │            │
│  │         │           │         │         │           │            │
│  │         ▼           │         │         ▼           │            │
│  │  ┌──────────────┐   │         │  ┌──────────────┐   │            │
│  │  │   Finish 1   │───┼────────►│  │   Finish 2   │   │            │
│  │  └──────────────┘   │ 依赖    │  └──────────────┘   │            │
│  │                     │         │                     │            │
│  └─────────────────────┘         └─────────────────────┘            │
│                                                                     │
│  Pipeline 2 的 Execute 依赖 Pipeline 1 的 Finish                    │
│  因为 Pipeline 2 需要消费 Pipeline 1 的输出                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 设计特点

### 优势

1. **依赖驱动**：事件自动按依赖顺序执行
2. **高并发**：支持多任务并行执行
3. **灵活性**：支持动态插入事件
4. **异步支持**：任务可以阻塞和恢复

### 与传统模型对比

| 特性 | 事件驱动模型 | 传统线程池 |
|-----|-------------|-----------|
| 依赖管理 | 内建支持 | 手动同步 |
| 任务调度 | 自动触发 | 显式提交 |
| 并发控制 | 原子计数 | 锁/信号量 |
| 错误传播 | 通过 Executor | 回调/Future |

## 小结

Event 事件驱动模型是 DuckDB 并行执行的协调中心，其设计体现了几个关键原则：

1. **依赖计数**：通过原子计数器实现精确的依赖管理
2. **自动触发**：依赖完成自动触发后续事件
3. **任务聚合**：每个事件可以包含多个并行任务
4. **生命周期管理**：使用弱引用避免循环引用
5. **灵活扩展**：支持动态插入新事件

在下一章中，我们将探讨 InterruptState 异步中断机制，了解任务如何实现阻塞和恢复。

# 第五章：InterruptState 异步中断机制

## 概述

DuckDB 的算子可能需要执行异步操作，如网络 I/O、磁盘读写等。InterruptState 提供了一种统一的机制，允许任务在等待异步操作完成时"阻塞"，并在操作完成后被"唤醒"继续执行。

本章深入分析 InterruptMode 中断模式、InterruptState 状态管理、InterruptDoneSignalState 同步原语以及 StateWithBlockableTasks 可阻塞任务状态。

## InterruptMode 中断模式

### 模式定义

```cpp
// src/include/duckdb/parallel/interrupt.hpp
enum class InterruptMode : uint8_t {
    NO_INTERRUPTS,  // 无中断模式
    TASK,           // 任务级中断
    BLOCKING        // 阻塞式中断
};
```

### 模式说明

| 模式 | 使用场景 | 回调行为 |
|-----|---------|---------|
| NO_INTERRUPTS | 已知不会阻塞的操作 | 抛出异常 |
| TASK | 正常查询执行 | 重新调度任务 |
| BLOCKING | 无任务上下文的代码路径 | 发送信号唤醒 |

```
┌─────────────────────────────────────────────────────────────────────┐
│                     InterruptMode 选择                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌──────────────────────┐                         │
│                    │   算子需要阻塞？      │                         │
│                    └──────────┬───────────┘                         │
│                               │                                     │
│               ┌───────────────┼───────────────┐                     │
│               │ 否            │ 是            │                     │
│               ▼               │               │                     │
│     ┌─────────────────┐       │               │                     │
│     │  NO_INTERRUPTS  │       │               │                     │
│     │  (直接执行)      │       │               │                     │
│     └─────────────────┘       │               │                     │
│                               ▼               │                     │
│                    ┌──────────────────┐       │                     │
│                    │ 有 Task 上下文？  │       │                     │
│                    └──────────┬───────┘       │                     │
│                               │               │                     │
│               ┌───────────────┴───────────────┐                     │
│               │ 是                            │ 否                  │
│               ▼                               ▼                     │
│     ┌─────────────────┐             ┌─────────────────┐             │
│     │      TASK       │             │    BLOCKING     │             │
│     │  (推荐方式)      │             │ (条件变量等待)   │             │
│     └─────────────────┘             └─────────────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## InterruptState 状态管理

### 类定义

```cpp
// src/include/duckdb/parallel/interrupt.hpp
class InterruptState {
public:
    //! 默认构造：NO_INTERRUPTS 模式
    InterruptState();
    //! TASK 模式：注册任务弱引用
    explicit InterruptState(weak_ptr<Task> task);
    //! BLOCKING 模式：注册信号状态弱引用
    explicit InterruptState(weak_ptr<InterruptDoneSignalState> done_signal);

    //! 执行回调，表示中断结束
    DUCKDB_API void Callback() const;

protected:
    InterruptMode mode;
    weak_ptr<Task> current_task;
    weak_ptr<InterruptDoneSignalState> signal_state;
};
```

### 构造函数

```cpp
// src/parallel/interrupt.cpp
InterruptState::InterruptState() : mode(InterruptMode::NO_INTERRUPTS) {
}

InterruptState::InterruptState(weak_ptr<Task> task)
    : mode(InterruptMode::TASK), current_task(std::move(task)) {
}

InterruptState::InterruptState(weak_ptr<InterruptDoneSignalState> signal_state_p)
    : mode(InterruptMode::BLOCKING), signal_state(std::move(signal_state_p)) {
}
```

### Callback 回调

```cpp
void InterruptState::Callback() const {
    if (mode == InterruptMode::TASK) {
        auto task = current_task.lock();
        if (!task) {
            return;  // 任务已销毁，NOP
        }
        task->Reschedule();  // 重新调度任务
    } else if (mode == InterruptMode::BLOCKING) {
        auto signal_state_l = signal_state.lock();
        if (!signal_state_l) {
            return;  // 信号状态已销毁
        }
        signal_state_l->Signal();  // 发送信号
    } else {
        throw InternalException("Callback made on InterruptState "
                                "without valid interrupt mode specified");
    }
}
```

回调流程图：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Callback 执行流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────┐                                              │
│  │  InterruptState   │                                              │
│  │     Callback()    │                                              │
│  └─────────┬─────────┘                                              │
│            │                                                        │
│            ▼                                                        │
│  ┌───────────────────┐                                              │
│  │   检查 mode       │                                              │
│  └─────────┬─────────┘                                              │
│            │                                                        │
│     ┌──────┼──────────────────┐                                     │
│     │      │                  │                                     │
│     ▼      ▼                  ▼                                     │
│  ┌──────┐ ┌──────────┐    ┌──────────────┐                          │
│  │ TASK │ │ BLOCKING │    │NO_INTERRUPTS │                          │
│  └──┬───┘ └────┬─────┘    └──────┬───────┘                          │
│     │          │                 │                                  │
│     ▼          ▼                 ▼                                  │
│  ┌─────────┐ ┌──────────────┐ ┌────────────────────┐                │
│  │lock()   │ │lock()        │ │ throw Exception    │                │
│  │weak_ptr │ │signal_state  │ └────────────────────┘                │
│  └────┬────┘ └──────┬───────┘                                       │
│       │             │                                               │
│   ┌───┴───┐     ┌───┴───┐                                           │
│   │expired│     │expired│                                           │
│   │  ?    │     │  ?    │                                           │
│   └───┬───┘     └───┬───┘                                           │
│       │ No          │ No                                            │
│       ▼             ▼                                               │
│  ┌──────────────┐ ┌──────────────┐                                  │
│  │ Reschedule() │ │   Signal()   │                                  │
│  │   任务重调度  │ │  条件变量通知 │                                  │
│  └──────────────┘ └──────────────┘                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## InterruptDoneSignalState 同步原语

### 定义

```cpp
struct InterruptDoneSignalState {
    void Signal();
    void Await();

protected:
    mutex lock;
    std::condition_variable cv;
    bool done = false;
};
```

这是一个简单的一次性信号原语，用于 BLOCKING 模式下的同步。

### 实现

```cpp
// src/parallel/interrupt.cpp
void InterruptDoneSignalState::Signal() {
    {
        unique_lock<mutex> lck {lock};
        done = true;
    }
    cv.notify_all();
}

void InterruptDoneSignalState::Await() {
    std::unique_lock<std::mutex> lck(lock);
    cv.wait(lck, [&]() { return done; });
    // 收到信号后重置
    done = false;
}
```

### 使用模式

```
┌─────────────────────────────────────────────────────────────────────┐
│                InterruptDoneSignalState 使用模式                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   调用者线程                         异步操作线程                    │
│                                                                     │
│   ┌────────────────────┐                                            │
│   │ 创建 SignalState   │                                            │
│   │ 创建 InterruptState│                                            │
│   └─────────┬──────────┘                                            │
│             │                                                       │
│             ▼                                                       │
│   ┌────────────────────┐                                            │
│   │ 发起异步操作       │─────────────────────────┐                   │
│   │ (传入 InterruptState)                      │                   │
│   └─────────┬──────────┘                        │                   │
│             │                                   ▼                   │
│             ▼                         ┌────────────────────┐        │
│   ┌────────────────────┐              │   执行异步操作     │        │
│   │  Await()           │              │   (网络/磁盘 I/O)  │        │
│   │  (条件变量等待)    │              └─────────┬──────────┘        │
│   │       ⏸️            │                        │                   │
│   │                    │                        ▼                   │
│   │                    │              ┌────────────────────┐        │
│   │                    │              │  操作完成          │        │
│   │                    │              │  Callback()        │        │
│   │                    │◄─────────────│  → Signal()       │        │
│   │       ▼            │              └────────────────────┘        │
│   │  被唤醒，继续执行   │                                            │
│   └────────────────────┘                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Task 阻塞与恢复

### Deschedule - 移出调度

当任务返回 `TASK_BLOCKED` 时，TaskScheduler 调用 `Deschedule`：

```cpp
// src/parallel/task_scheduler.cpp
case TaskExecutionResult::TASK_BLOCKED:
    task->Deschedule();
    task.reset();
    break;
```

ExecutorTask 的 Deschedule 实现：

```cpp
// src/parallel/executor_task.cpp
void ExecutorTask::Deschedule() {
    auto this_ptr = shared_from_this();
    executor.AddToBeRescheduled(this_ptr);
}
```

任务被添加到 Executor 的待重调度列表，但不再被 TaskScheduler 调度执行。

### Reschedule - 重新调度

当异步操作完成，回调触发 `Reschedule`：

```cpp
void ExecutorTask::Reschedule() {
    auto this_ptr = shared_from_this();
    executor.RescheduleTask(this_ptr);
}
```

任务被重新提交到 TaskScheduler 的队列中。

### 阻塞-恢复流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    任务阻塞-恢复流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  TaskScheduler                Task                   异步操作       │
│       │                         │                        │          │
│       │ Execute()               │                        │          │
│       ├────────────────────────►│                        │          │
│       │                         │                        │          │
│       │                         │ 发起异步操作            │          │
│       │                         │  (保存 InterruptState)  │          │
│       │                         ├───────────────────────►│          │
│       │                         │                        │          │
│       │                         │ return TASK_BLOCKED    │          │
│       │◄────────────────────────┤                        │          │
│       │                         │                        │          │
│       │ Deschedule()            │                        │          │
│       ├────────────────────────►│                        │          │
│       │                         │                        │          │
│       │                         │ AddToBeRescheduled()   │          │
│       │                         ├─────────┐              │          │
│       │                         │         │              │          │
│       │                         │◄────────┘              │          │
│       │                         │                        │          │
│       │     (任务暂停)           │                        │ 异步完成 │
│       │                         │                        │          │
│       │                         │         Callback()     │          │
│       │                         │◄───────────────────────┤          │
│       │                         │                        │          │
│       │                         │ Reschedule()           │          │
│       │                         ├─────────┐              │          │
│       │                         │         │              │          │
│       │◄────────────────────────┤         │              │          │
│       │                         │         │              │          │
│       │ RescheduleTask()        │         │              │          │
│       │ (重新入队)               │         │              │          │
│       │                         │         │              │          │
│       │ Execute() (继续)        │         │              │          │
│       ├────────────────────────►│         │              │          │
│       │                         │         │              │          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## StateWithBlockableTasks 可阻塞任务状态

### 类定义

```cpp
class StateWithBlockableTasks {
public:
    unique_lock<mutex> Lock() {
        return unique_lock<mutex>(lock);
    }

    void PreventBlocking(const unique_lock<mutex> &guard) {
        VerifyLock(guard);
        can_block = false;
    }

    //! 返回 BLOCKED 前将任务添加到阻塞列表
    bool BlockTask(const unique_lock<mutex> &guard, const InterruptState &interrupt_state) {
        VerifyLock(guard);
        if (can_block) {
            blocked_tasks.push_back(interrupt_state);
            return true;
        }
        return false;
    }

    //! 解除所有阻塞任务
    bool UnblockTasks(const unique_lock<mutex> &guard) {
        VerifyLock(guard);
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

### 设计目的

StateWithBlockableTasks 提供了管理多个阻塞任务的能力：

1. **统一管理**：集中管理所有等待某个条件的任务
2. **批量唤醒**：条件满足时一次性唤醒所有等待者
3. **阻塞控制**：可以禁止新任务阻塞（PreventBlocking）

### 使用模式

```
┌─────────────────────────────────────────────────────────────────────┐
│              StateWithBlockableTasks 使用模式                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  任务 A                        StateWithBlockableTasks              │
│    │                                     │                          │
│    │ Lock()                              │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │                          │
│    │ 条件不满足，需要阻塞                  │                          │
│    │                                     │                          │
│    │ BlockSink(interrupt_state)          │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │ blocked_tasks.push(A)    │
│    │◄────────────────────────────────────┤                          │
│    │ return BLOCKED                      │                          │
│    │                                     │                          │
│  任务 B                                  │                          │
│    │                                     │                          │
│    │ Lock()                              │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │                          │
│    │ BlockSink(interrupt_state)          │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │ blocked_tasks.push(B)    │
│    │◄────────────────────────────────────┤                          │
│    │ return BLOCKED                      │                          │
│    │                                     │                          │
│  条件满足                                 │                          │
│    │                                     │                          │
│    │ Lock()                              │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │                          │
│    │ UnblockTasks()                      │                          │
│    ├────────────────────────────────────►│                          │
│    │                                     │ for task : blocked_tasks │
│    │                                     │   task.Callback()        │
│    │                                     │                          │
│    │                            任务 A 和 B 被重新调度                │
│    │                                     │                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 锁验证

```cpp
void VerifyLock(const unique_lock<mutex> &guard) const {
#ifdef DEBUG
    D_ASSERT(guard.mutex() && RefersToSameObject(*guard.mutex(), lock));
#endif
}
```

在 DEBUG 模式下验证调用者确实持有正确的锁。

## 实际使用示例

### PipelineFinishTask

```cpp
// src/parallel/pipeline_finish_event.cpp
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
        auto op_state = op.OperatorFinalize(..., op_finalize_input);
        if (op_state == OperatorFinalResultType::BLOCKED) {
            return TaskExecutionResult::TASK_BLOCKED;
        }
    }

    // 对 Sink 调用 Finalize
    OperatorSinkFinalizeInput finalize_input {*sink->sink_state, interrupt_state};
    auto sink_state = sink->Finalize(..., finalize_input);

    if (sink_state == SinkFinalizeType::BLOCKED) {
        return TaskExecutionResult::TASK_BLOCKED;
    }

    event->FinishTask();
    return TaskExecutionResult::TASK_FINISHED;
}
```

关键点：
1. 创建 `InterruptState(shared_from_this())` 关联当前任务
2. 将 InterruptState 传递给可能阻塞的操作
3. 如果操作返回 BLOCKED，任务也返回 TASK_BLOCKED
4. 异步操作完成时调用 `interrupt_state.Callback()`
5. 任务被重新调度，从中断点继续执行

### 状态保存

注意 `operator_idx` 成员变量：

```cpp
private:
    idx_t operator_idx = 0;
```

任务被重新调度后，从 `operator_idx` 记录的位置继续执行，而不是从头开始。

## 弱引用的重要性

InterruptState 使用 `weak_ptr`：

```cpp
weak_ptr<Task> current_task;
weak_ptr<InterruptDoneSignalState> signal_state;
```

原因：
1. **避免循环引用**：Task 可能间接持有 InterruptState
2. **安全处理取消**：如果任务被取消，回调变成 NOP
3. **内存安全**：不延长对象生命周期

```cpp
void InterruptState::Callback() const {
    if (mode == InterruptMode::TASK) {
        auto task = current_task.lock();
        if (!task) {
            return;  // 安全处理：任务已销毁
        }
        task->Reschedule();
    }
    // ...
}
```

## 设计优势

| 特性 | 实现方式 | 优势 |
|-----|---------|------|
| 模式灵活 | InterruptMode 枚举 | 适应不同场景 |
| 任务级阻塞 | weak_ptr<Task> | 避免内存泄漏 |
| 批量管理 | StateWithBlockableTasks | 高效唤醒 |
| 同步阻塞 | InterruptDoneSignalState | 无任务时可用 |

## 小结

InterruptState 异步中断机制是 DuckDB 处理异步操作的核心，其设计体现了几个关键原则：

1. **模式分离**：三种模式适应不同使用场景
2. **弱引用安全**：使用 weak_ptr 避免生命周期问题
3. **回调驱动**：异步完成通过 Callback 通知
4. **状态保存**：任务可以从中断点恢复执行
5. **批量管理**：StateWithBlockableTasks 支持多任务阻塞

在下一章中，我们将探讨事务并发控制，了解 DuckDB 如何管理事务的并发访问。

# 第三章：TaskScheduler 任务调度器

## 概述

TaskScheduler 是 DuckDB 并行执行的核心组件，负责管理工作线程池和任务队列。它采用生产者-消费者模式，通过无锁并发队列实现高效的任务分发，是 DuckDB 实现多核并行计算的基础设施。

本章深入分析 TaskScheduler 的架构设计、任务队列实现、生产者令牌机制以及工作线程的管理策略。

## TaskScheduler 架构设计

### 核心组件

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp
class TaskScheduler {
    // 超时时间：5毫秒
    constexpr static int64_t TASK_TIMEOUT_USECS = 5000;

private:
    DatabaseInstance &db;
    //! 任务队列
    unique_ptr<ConcurrentQueue> queue;
    //! 线程数修改锁
    mutex thread_lock;
    //! 后台工作线程
    vector<unique_ptr<SchedulerThread>> threads;
    //! 线程标记（false 时线程退出）
    vector<unique_ptr<atomic<bool>>> markers;
    //! 分配器刷新阈值
    atomic<idx_t> allocator_flush_threshold;
    //! 是否启用分配器后台线程
    atomic<bool> allocator_background_threads;
    //! 请求的线程数
    atomic<int32_t> requested_thread_count;
    //! 当前运行的线程数
    atomic<int32_t> current_thread_count;
};
```

### 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TaskScheduler                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐     ┌─────────────────────────────────────────┐   │
│  │   Executor   │     │           ConcurrentQueue               │   │
│  │  (生产者)    │────►│  ┌─────┬─────┬─────┬─────┬─────┐        │   │
│  └──────────────┘     │  │Task │Task │Task │Task │ ... │        │   │
│                       │  └─────┴─────┴─────┴─────┴─────┘        │   │
│  ┌──────────────┐     │                                         │   │
│  │   Executor   │────►│  LightweightSemaphore (唤醒信号)          │   │
│  │  (生产者)    │     └────────────────┬────────────────────────┘   │
│  └──────────────┘                      │                            │
│                                        │ Dequeue                    │
│                      ┌─────────────────┼─────────────────┐          │
│                      ▼                 ▼                 ▼          │
│               ┌───────────┐     ┌───────────┐     ┌───────────┐     │
│               │  Worker   │     │  Worker   │     │  Worker   │     │
│               │  Thread 0 │     │  Thread 1 │     │  Thread N │     │
│               └───────────┘     └───────────┘     └───────────┘     │
│                    │                 │                 │            │
│                    │                 │                 │            │
│               ┌────▼────┐       ┌────▼────┐       ┌────▼────┐       │
│               │ marker  │       │ marker  │       │ marker  │       │
│               │ [true]  │       │ [true]  │       │ [true]  │       │
│               └─────────┘       └─────────┘       └─────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## ConcurrentQueue 无锁队列

### moodycamel::ConcurrentQueue

DuckDB 使用 moodycamel 库的无锁并发队列：

```cpp
// src/parallel/task_scheduler.cpp
#ifndef DUCKDB_NO_THREADS
#include "concurrentqueue.h"
#include "lightweightsemaphore.h"

typedef duckdb_moodycamel::ConcurrentQueue<shared_ptr<Task>> concurrent_queue_t;
typedef duckdb_moodycamel::LightweightSemaphore lightweight_semaphore_t;
#endif
```

moodycamel::ConcurrentQueue 的特点：
- **无锁设计**：使用 CAS 操作实现线程安全
- **多生产者多消费者**：支持 MPMC 模式
- **批量操作**：支持批量入队/出队
- **生产者令牌**：提供局部性优化

### ConcurrentQueue 封装

```cpp
struct ConcurrentQueue {
    ConcurrentQueue() : tasks_in_queue(0) {
    }

    lightweight_semaphore_t semaphore;   // 轻量级信号量
    concurrent_queue_t q;                 // 底层无锁队列
    atomic<idx_t> tasks_in_queue;        // 任务计数
};
```

### 入队操作

```cpp
void ConcurrentQueue::Enqueue(ProducerToken &token, shared_ptr<Task> task) {
    lock_guard<mutex> producer_lock(token.producer_lock);
    task->token = token;                  // 记录任务所属生产者
    if (q.enqueue(token.token->queue_token, std::move(task))) {
        ++tasks_in_queue;
        semaphore.signal();               // 唤醒一个等待的工作线程
    } else {
        throw InternalException("Could not schedule task!");
    }
}
```

入队流程：
1. 获取生产者锁
2. 将任务关联到生产者令牌
3. 调用底层队列的 enqueue
4. 增加任务计数
5. 发送信号唤醒工作线程

### 批量入队

```cpp
void ConcurrentQueue::EnqueueBulk(ProducerToken &token, vector<shared_ptr<Task>> &tasks) {
    typedef std::make_signed<std::size_t>::type ssize_t;
    lock_guard<mutex> producer_lock(token.producer_lock);
    for (auto &task : tasks) {
        task->token = token;
    }
    if (q.enqueue_bulk(token.token->queue_token,
                       std::make_move_iterator(tasks.begin()),
                       tasks.size())) {
        tasks_in_queue += tasks.size();
        semaphore.signal(NumericCast<ssize_t>(tasks.size()));  // 唤醒多个线程
    } else {
        throw InternalException("Could not schedule tasks!");
    }
}
```

批量入队优化：
- 一次性入队多个任务，减少锁竞争
- 按任务数量发送信号，唤醒相应数量的线程

### 出队操作

```cpp
bool ConcurrentQueue::DequeueFromProducer(ProducerToken &token, shared_ptr<Task> &task) {
    lock_guard<mutex> producer_lock(token.producer_lock);
    if (!q.try_dequeue_from_producer(token.token->queue_token, task)) {
        return false;
    }
    --tasks_in_queue;
    return true;
}

bool ConcurrentQueue::Dequeue(shared_ptr<Task> &task) {
    if (!q.try_dequeue(task)) {
        return false;
    }
    --tasks_in_queue;
    return true;
}
```

两种出队方式：
- **DequeueFromProducer**：从特定生产者获取任务（工作窃取优化）
- **Dequeue**：从任意位置获取任务（全局调度）

## ProducerToken 生产者令牌

### 设计目的

生产者令牌为每个任务生产者（通常是 Executor）提供独立的入队通道：

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp
struct ProducerToken {
    ProducerToken(TaskScheduler &scheduler, unique_ptr<QueueProducerToken> token);
    ~ProducerToken();

    TaskScheduler &scheduler;
    unique_ptr<QueueProducerToken> token;  // moodycamel 生产者令牌
    mutex producer_lock;                    // 保护生产者操作
};
```

### 创建生产者

```cpp
unique_ptr<ProducerToken> TaskScheduler::CreateProducer() {
    auto token = make_uniq<QueueProducerToken>(*queue);
    return make_uniq<ProducerToken>(*this, std::move(token));
}
```

每个 Executor 在初始化时创建自己的 ProducerToken：

```cpp
// src/parallel/executor.cpp
this->producer = scheduler.CreateProducer();
```

### 局部性优化

生产者令牌的优势：
1. **减少竞争**：每个生产者有自己的入队区域
2. **缓存友好**：相关任务存储在相邻位置
3. **优先消费**：可以优先处理自己生产的任务

```
┌─────────────────────────────────────────────────────────────┐
│                    ConcurrentQueue 内部结构                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Producer 0 区域        Producer 1 区域       Producer N 区域│
│  ┌─────────────────┐   ┌─────────────────┐   ┌────────────┐ │
│  │ Task │ Task │...│   │ Task │ Task │...│   │ Task │ ... │ │
│  └─────────────────┘   └─────────────────┘   └────────────┘ │
│         ▲                     ▲                    ▲        │
│         │                     │                    │        │
│    ProducerToken 0       ProducerToken 1      ProducerToken N│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 工作线程管理

### SchedulerThread 封装

```cpp
struct SchedulerThread {
#ifndef DUCKDB_NO_THREADS
    explicit SchedulerThread(unique_ptr<thread> thread_p)
        : internal_thread(std::move(thread_p)) {
    }

    unique_ptr<thread> internal_thread;
#endif
};
```

### 设置线程数

```cpp
void TaskScheduler::SetThreads(idx_t total_threads, idx_t external_threads) {
    if (total_threads == 0) {
        throw SyntaxException("Number of threads must be positive!");
    }
#ifndef DUCKDB_NO_THREADS
    if (total_threads < external_threads) {
        throw SyntaxException("Number of threads can't be smaller than "
                              "number of external threads!");
    }
#endif
    requested_thread_count = NumericCast<int32_t>(total_threads - external_threads);
}
```

参数说明：
- **total_threads**：总线程数（包括主线程）
- **external_threads**：外部线程数（如主线程）
- **后台线程数** = total_threads - external_threads

### 线程启动

```cpp
void TaskScheduler::RelaunchThreadsInternal(int32_t n) {
#ifndef DUCKDB_NO_THREADS
    auto new_thread_count = NumericCast<idx_t>(n);

    // 减少线程：先停止再清理
    if (threads.size() > new_thread_count) {
        for (idx_t i = 0; i < threads.size(); i++) {
            *markers[i] = false;          // 设置退出标记
        }
        Signal(threads.size());           // 唤醒所有线程
        for (idx_t i = 0; i < threads.size(); i++) {
            threads[i]->internal_thread->join();  // 等待退出
        }
        threads.clear();
        markers.clear();
    }

    // 增加线程：启动新线程
    if (threads.size() < new_thread_count) {
        idx_t create_new_threads = new_thread_count - threads.size();

        // 是否绑定线程到 CPU 核心
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
                    SetThreadAffinity(*worker_thread, NumericCast<int>(threads.size()));
                }
            } catch (std::exception &ex) {
                break;  // 线程创建失败，停止创建更多
            }
            auto thread_wrapper = make_uniq<SchedulerThread>(std::move(worker_thread));
            threads.push_back(std::move(thread_wrapper));
            markers.push_back(std::move(marker));
        }
    }
    current_thread_count = NumericCast<int32_t>(threads.size() + config.options.external_threads);
#endif
}
```

### 线程亲和性

当 CPU 核心数超过阈值（64）时，启用线程亲和性：

```cpp
static void SetThreadAffinity(thread &thread, const int &cpu_id) {
#if defined(__GLIBC__)
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(thread.native_handle(), sizeof(cpu_set_t), &cpuset);
#endif
}
```

线程亲和性的好处：
- 减少线程在 CPU 核心间迁移
- 提高缓存命中率
- 在 NUMA 架构下减少跨节点内存访问

## 任务执行循环

### ExecuteForever - 后台线程主循环

```cpp
void TaskScheduler::ExecuteForever(atomic<bool> *marker) {
#ifndef DUCKDB_NO_THREADS
    static constexpr const int64_t INITIAL_FLUSH_WAIT = 500000; // 0.5秒

    const auto &block_allocator = BlockAllocator::Get(db);
    const auto &config = DBConfig::GetConfig(db);

    shared_ptr<Task> task;
    while (*marker) {
        // 等待任务或超时
        if (!block_allocator.SupportsFlush()) {
            queue->semaphore.wait();  // 无限等待
        } else if (!queue->semaphore.wait(INITIAL_FLUSH_WAIT)) {
            // 等待超时 - 刷新分配器
            block_allocator.ThreadFlush(...);
            // 继续等待或标记线程空闲
            ...
        }

        // 尝试获取并执行任务
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
                // 任务未完成 - 重新调度
                queue->Enqueue(*task->token, std::move(task));
                break;
            case TaskExecutionResult::TASK_BLOCKED:
                // 任务阻塞 - 移出调度
                task->Deschedule();
                task.reset();
                break;
            }
        } else if (queue->GetTasksInQueue() > 0) {
            // 出队失败但仍有任务 - 重试
            queue->semaphore.signal(1);
        }
    }
    // 线程退出前刷新分配器
    if (block_allocator.SupportsFlush()) {
        block_allocator.ThreadFlush(...);
        Allocator::ThreadIdle();
    }
#endif
}
```

### 执行状态处理

```
┌─────────────────────────────────────────────────────────────┐
│                     任务执行结果处理                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐                                            │
│  │ task.Execute│                                            │
│  └──────┬──────┘                                            │
│         │                                                   │
│         ▼                                                   │
│  ┌────────────────────────────────────────────────┐         │
│  │              TaskExecutionResult               │         │
│  └────────────────────────────────────────────────┘         │
│         │                                                   │
│    ┌────┼────┬────────────┬────────────┐                    │
│    │    │    │            │            │                    │
│    ▼    ▼    ▼            ▼            ▼                    │
│ ┌──────┐ ┌──────┐  ┌───────────┐  ┌─────────┐              │
│ │FINISH│ │ERROR │  │NOT_FINISH │  │ BLOCKED │              │
│ └──┬───┘ └──┬───┘  └─────┬─────┘  └────┬────┘              │
│    │        │            │             │                    │
│    ▼        ▼            ▼             ▼                    │
│ ┌──────────────┐   ┌───────────┐  ┌───────────┐            │
│ │ task.reset() │   │ Reschedule│  │ Deschedule│            │
│ │  释放任务     │   │  重新入队  │  │  移出调度  │            │
│ └──────────────┘   └───────────┘  └───────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 限制执行

```cpp
idx_t TaskScheduler::ExecuteTasks(atomic<bool> *marker, idx_t max_tasks) {
    idx_t completed_tasks = 0;
    while (*marker && completed_tasks < max_tasks) {
        shared_ptr<Task> task;
        if (!queue->Dequeue(task)) {
            return completed_tasks;
        }
        auto execute_result = task->Execute(TaskExecutionMode::PROCESS_ALL);

        switch (execute_result) {
        case TaskExecutionResult::TASK_FINISHED:
        case TaskExecutionResult::TASK_ERROR:
            task.reset();
            completed_tasks++;
            break;
        case TaskExecutionResult::TASK_NOT_FINISHED:
            throw InternalException("Task should not return TASK_NOT_FINISHED "
                                    "in PROCESS_ALL mode");
        case TaskExecutionResult::TASK_BLOCKED:
            task->Deschedule();
            task.reset();
            break;
        }
    }
    return completed_tasks;
}
```

此版本用于主线程参与任务执行，设置最大执行任务数以避免阻塞。

## 信号与唤醒机制

### 轻量级信号量

moodycamel::LightweightSemaphore 提供高效的线程唤醒：

```cpp
void TaskScheduler::Signal(idx_t n) {
#ifndef DUCKDB_NO_THREADS
    typedef std::make_signed<std::size_t>::type ssize_t;
    queue->semaphore.signal(NumericCast<ssize_t>(n));
#endif
}
```

信号量操作：
- **signal(n)**：增加 n 个许可，唤醒最多 n 个等待线程
- **wait()**：等待一个许可，无限阻塞
- **wait(timeout)**：带超时等待

### 唤醒策略

```
┌─────────────────────────────────────────────────────────────┐
│                       唤醒策略                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  生产者入队                                                  │
│  ┌────────────────────┐                                     │
│  │ Enqueue(1 task)    │ ───► signal(1)  唤醒 1 个线程        │
│  └────────────────────┘                                     │
│                                                             │
│  批量入队                                                    │
│  ┌────────────────────┐                                     │
│  │ EnqueueBulk(n tasks│ ───► signal(n)  唤醒 n 个线程        │
│  └────────────────────┘                                     │
│                                                             │
│  线程退出                                                    │
│  ┌────────────────────┐                                     │
│  │ 设置 markers=false │ ───► Signal(all) 唤醒所有线程        │
│  └────────────────────┘                                     │
│                                                             │
│  出队失败但有任务                                             │
│  ┌────────────────────┐                                     │
│  │ Dequeue 返回 false │ ───► signal(1)  重试信号             │
│  │ 但 GetTasksInQueue>0│                                    │
│  └────────────────────┘                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## CPU 核心检测

### GetEstimatedCPUId

获取当前线程运行的 CPU 核心：

```cpp
idx_t TaskScheduler::GetEstimatedCPUId() {
#if defined(_WIN32)
    return (idx_t)GetCurrentProcessorNumber();
#elif defined(_GNU_SOURCE)
    auto cpu = sched_getcpu();
    if (cpu < 0) {
        // 回退到线程 ID 哈希
        return (idx_t)std::hash<std::thread::id>()(std::this_thread::get_id());
    }
    return (idx_t)cpu;
#elif defined(__aarch64__) && defined(__APPLE__)
    // Apple Silicon 特殊处理
    uintptr_t c;
    asm volatile("mrs %x0, tpidrro_el0" : "=r"(c)::"memory");
    return (idx_t)(c & ((1 << 3) - 1));
#else
    return (idx_t)std::hash<std::thread::id>()(std::this_thread::get_id());
#endif
}
```

用途：
- 实现 CPU 亲和性的任务调度
- 优化 NUMA 架构下的内存访问
- 负载均衡决策

## 无线程模式

当编译时定义 `DUCKDB_NO_THREADS`，使用简化的队列实现：

```cpp
#else
struct ConcurrentQueue {
    reference_map_t<QueueProducerToken, std::queue<shared_ptr<Task>>> q;
    mutable mutex qlock;

    void Enqueue(ProducerToken &token, shared_ptr<Task> task) {
        lock_guard<mutex> lock(qlock);
        task->token = token;
        q[std::ref(*token.token)].push(std::move(task));
    }

    bool DequeueFromProducer(ProducerToken &token, shared_ptr<Task> &task) {
        lock_guard<mutex> lock(qlock);
        const auto it = q.find(std::ref(*token.token));
        if (it == q.end() || it->second.empty()) {
            return false;
        }
        task = std::move(it->second.front());
        it->second.pop();
        return true;
    }
};
#endif
```

特点：
- 使用 `std::queue` 替代无锁队列
- 使用互斥锁保护所有操作
- 不支持全局 Dequeue（单线程无需）

## 与 Executor 的集成

### ExecutorTask

```cpp
// src/include/duckdb/parallel/executor_task.hpp
class ExecutorTask : public Task {
public:
    ExecutorTask(Executor &executor, shared_ptr<Event> event);

    Executor &executor;
    shared_ptr<Event> event;
    unique_ptr<ThreadContext> thread_context;

    virtual TaskExecutionResult ExecuteTask(TaskExecutionMode mode) = 0;
    TaskExecutionResult Execute(TaskExecutionMode mode) override;
};
```

ExecutorTask 是查询执行任务的基类，关联：
- **Executor**：查询执行器
- **Event**：所属事件（用于依赖管理）
- **ThreadContext**：线程上下文

### 任务调度示例

```cpp
// src/parallel/event.cpp
void Event::SetTasks(vector<shared_ptr<Task>> tasks) {
    auto &ts = TaskScheduler::GetScheduler(executor.context);
    this->total_tasks = tasks.size();
    ts.ScheduleTasks(executor.GetToken(), tasks);
}
```

## 内存管理集成

### 分配器刷新

后台线程空闲时刷新内存分配器：

```cpp
if (!queue->semaphore.wait(INITIAL_FLUSH_WAIT)) {
    // 0.5秒内没有任务，刷新分配器
    block_allocator.ThreadFlush(allocator_background_threads,
                                 allocator_flush_threshold,
                                 NumericCast<idx_t>(requested_thread_count.load()));

    // 等待 decay delay 后标记线程空闲
    auto decay_delay = Allocator::DecayDelay();
    if (!queue->semaphore.wait(decay_delay * 1000000 - INITIAL_FLUSH_WAIT)) {
        Allocator::ThreadIdle();
        queue->semaphore.wait();  // 无限等待
    }
}
```

这个机制确保：
- 空闲线程及时释放内存
- 减少内存占用
- 优化内存分配器性能

## 设计特点总结

| 特性 | 实现方式 | 优势 |
|-----|---------|------|
| 无锁队列 | moodycamel::ConcurrentQueue | 高并发吞吐 |
| 生产者令牌 | ProducerToken | 减少竞争，局部性优化 |
| 轻量级信号量 | LightweightSemaphore | 高效线程唤醒 |
| 线程亲和性 | pthread_setaffinity_np | NUMA 优化 |
| 动态线程数 | RelaunchThreadsInternal | 资源弹性 |
| 内存管理 | ThreadFlush/ThreadIdle | 减少内存占用 |

## 小结

TaskScheduler 是 DuckDB 并行执行的核心调度器，其设计体现了几个关键原则：

1. **无锁优先**：使用 moodycamel 无锁队列，最大化并发性能
2. **生产者隔离**：通过 ProducerToken 减少竞争，提高局部性
3. **高效唤醒**：轻量级信号量实现精确的线程唤醒
4. **资源感知**：线程亲和性和内存管理优化
5. **灵活调度**：支持阻塞/非阻塞、限量执行等多种模式

在下一章中，我们将探讨 Event 事件驱动模型，了解任务之间如何通过依赖关系协调执行。

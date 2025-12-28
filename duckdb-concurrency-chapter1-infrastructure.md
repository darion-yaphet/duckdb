# 第一章：并发基础设施

## 1.1 概述

DuckDB 作为一个嵌入式分析型数据库，在并发控制方面采用了独特的设计理念。不同于传统的多用户 OLTP 数据库，DuckDB 侧重于单进程内的并行查询执行，同时保证数据一致性和线程安全。本章介绍 DuckDB 并发基础设施的核心组件。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DuckDB 并发层次结构                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    应用层 (Application Layer)                    │   │
│  │   ClientContext  │  Connection  │  Query Execution               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    调度层 (Scheduling Layer)                      │   │
│  │   TaskScheduler  │  Event System  │  Pipeline Executor           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    同步层 (Synchronization Layer)                 │   │
│  │   StorageLock  │  Transaction Lock  │  Catalog Lock              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    原语层 (Primitive Layer)                       │   │
│  │   mutex  │  atomic  │  condition_variable  │  ConcurrentQueue    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1.2 基础同步原语

### 1.2.1 互斥锁封装

DuckDB 对标准库的同步原语进行了简单封装，保持接口一致性：

```cpp
// src/include/duckdb/common/mutex.hpp

#pragma once

#ifdef __MVS__
#include <time.h>
#endif
#include <mutex>

namespace duckdb {
using std::lock_guard;
using std::mutex;
using std::unique_lock;
} // namespace duckdb
```

这种封装策略有几个优点：
- 统一命名空间，避免 `std::` 前缀污染
- 方便未来替换为自定义实现
- 支持平台特定的适配（如 z/OS 的 `__MVS__` 宏）

### 1.2.2 原子操作

```cpp
// src/include/duckdb/common/atomic.hpp

#pragma once

#include <atomic>

namespace duckdb {

using std::atomic;

//! NOTE: When repeatedly trying to atomically set a value in a loop, you can use:
//! * std::atomic_compare_exchange_weak
//! * std::atomic::compare_exchange_weak
//! If not used as a loop condition, use:
//! * std::atomic_compare_exchange_strong
//! * std::atomic::compare_exchange_strong
//! Performance may be optimized using std::memory_order, but NOT at the cost of correctness.

} // namespace duckdb
```

**关键注释解读：**
- `compare_exchange_weak`：适用于循环中的 CAS 操作，可能产生伪失败（spurious failure）
- `compare_exchange_strong`：单次操作使用，保证不会伪失败
- 内存序优化需谨慎，正确性优先于性能

### 1.2.3 原子指针

`atomic_ptr` 提供了线程安全的指针操作：

```cpp
// src/include/duckdb/common/atomic_ptr.hpp

template <class T, bool SAFE = true>
class atomic_ptr {
public:
    atomic_ptr() noexcept : ptr(nullptr) {}
    atomic_ptr(T *ptr_p) : ptr(ptr_p) {}
    atomic_ptr(T &ref) : ptr(&ref) {}
    atomic_ptr(const unique_ptr<T> &ptr_p) : ptr(ptr_p.get()) {}
    atomic_ptr(const shared_ptr<T> &ptr_p) : ptr(ptr_p.get()) {}

    T *GetPointer() {
        auto res = ptr.load();
        CheckValid(res);
        return res;
    }

    void set(T &ref) {
        ptr = &ref;
    }

    void reset() {
        ptr = nullptr;
    }

private:
    atomic<T *> ptr;
};

template <typename T>
using unsafe_atomic_ptr = atomic_ptr<T, false>;
```

**设计特点：**
- 模板参数 `SAFE` 控制是否进行空指针检查
- 支持从 `unique_ptr`、`shared_ptr` 隐式构造
- 提供类指针操作符（`*`、`->`）
- `unsafe_atomic_ptr` 别名用于性能敏感场景

## 1.3 DuckDB 并发设计哲学

### 1.3.1 单写多读模型

DuckDB 采用单写多读（Single Writer Multiple Reader, SWMR）模型：

```
┌─────────────────────────────────────────────────────────────────┐
│                    单写多读模型                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   写操作（排他）                    读操作（并行）               │
│   ┌─────────┐                      ┌─────────┐                  │
│   │ Writer  │                      │ Reader1 │                  │
│   │         │                      ├─────────┤                  │
│   │ INSERT  │       ×              │ Reader2 │  ✓ 并行读取      │
│   │ UPDATE  │  ─────────────▶      ├─────────┤                  │
│   │ DELETE  │                      │ Reader3 │                  │
│   └─────────┘                      └─────────┘                  │
│       │                                 │                       │
│       ▼                                 ▼                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    数据存储                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   特点：                                                        │
│   • 写操作串行执行，避免复杂的写冲突处理                         │
│   • 读操作完全并行，最大化分析查询吞吐                           │
│   • MVCC 保证读写不阻塞                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3.2 乐观并发控制

DuckDB 使用乐观并发控制（OCC）处理事务：

1. **读阶段**：事务在本地工作区执行，不加锁读取
2. **验证阶段**：提交时检查冲突
3. **写阶段**：验证通过后持久化

```cpp
// 事务提交简化流程
ErrorData CommitTransaction(Transaction &transaction) {
    // 1. 获取提交锁
    lock_guard<mutex> lock(transaction_lock);

    // 2. 分配提交时间戳
    transaction_t commit_id = GetCommitTimestamp();

    // 3. 验证并提交
    auto error = transaction.Commit(commit_id);
    if (error.HasError()) {
        // 冲突，回滚
        transaction.Rollback();
        return error;
    }

    // 4. 更新元数据
    last_commit = commit_id;
    return ErrorData();
}
```

### 1.3.3 无锁数据结构使用场景

DuckDB 在特定场景使用无锁数据结构：

| 场景 | 数据结构 | 原因 |
|------|----------|------|
| 任务队列 | ConcurrentQueue | 高频入队/出队，避免锁争用 |
| 引用计数 | atomic<idx_t> | 简单计数，无需复杂同步 |
| 状态标志 | atomic<bool> | 单一标志位，原子读写足够 |
| 事件计数 | atomic<idx_t> | 依赖完成计数 |

## 1.4 线程模型

### 1.4.1 主线程与工作线程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 线程模型                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────┐                                               │
│  │     主线程          │                                               │
│  │  (Client Thread)    │                                               │
│  │                     │                                               │
│  │  • 查询解析         │                                               │
│  │  • 计划生成         │                                               │
│  │  • 任务调度         │                                               │
│  │  • 结果收集         │                                               │
│  └──────────┬──────────┘                                               │
│             │ 提交任务                                                  │
│             ▼                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    TaskScheduler                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │                  ConcurrentQueue                            │  │  │
│  │  │    [Task1] [Task2] [Task3] [Task4] [Task5] ...             │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│             │ 获取任务                                                  │
│             ▼                                                          │
│  ┌─────────┬─────────┬─────────┬─────────┐                            │
│  │ Worker1 │ Worker2 │ Worker3 │ Worker4 │  工作线程池                 │
│  │         │         │         │         │                            │
│  │ Execute │ Execute │ Execute │ Execute │                            │
│  │  Task   │  Task   │  Task   │  Task   │                            │
│  └─────────┴─────────┴─────────┴─────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.4.2 ThreadContext 线程上下文

每个工作线程维护独立的上下文：

```cpp
// src/include/duckdb/parallel/thread_context.hpp

class ThreadContext {
public:
    explicit ThreadContext(ClientContext &context);
    ~ThreadContext();

    //! The operator profiler for the individual thread context
    OperatorProfiler profiler;
    unique_ptr<Logger> logger;
};
```

**线程上下文包含：**
- `profiler`：算子级性能分析器，记录各线程执行统计
- `logger`：线程私有日志记录器

### 1.4.3 线程安全边界

DuckDB 定义了清晰的线程安全边界：

```
┌─────────────────────────────────────────────────────────────────┐
│                    线程安全边界                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  线程私有（无需同步）           跨线程共享（需要同步）           │
│  ┌─────────────────────┐       ┌─────────────────────────────┐ │
│  │ • ThreadContext     │       │ • TaskScheduler            │ │
│  │ • PipelineExecutor  │       │ • TransactionManager       │ │
│  │ • LocalState        │       │ • Catalog                  │ │
│  │ • ExpressionState   │       │ • BufferManager            │ │
│  │ • 中间结果缓冲区    │       │ • StorageManager           │ │
│  └─────────────────────┘       └─────────────────────────────┘ │
│           │                              │                      │
│           ▼                              ▼                      │
│     无竞争访问                      需要锁保护                   │
│     最高性能                        正确性优先                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1.5 Task 任务抽象

### 1.5.1 Task 基类

```cpp
// src/include/duckdb/parallel/task.hpp

enum class TaskExecutionMode : uint8_t {
    PROCESS_ALL,      // 必须完成全部处理
    PROCESS_PARTIAL   // 可以部分处理后返回
};

enum class TaskExecutionResult : uint8_t {
    TASK_FINISHED,     // 任务完成
    TASK_NOT_FINISHED, // 任务未完成，需再次调度
    TASK_ERROR,        // 任务出错
    TASK_BLOCKED       // 任务阻塞，等待外部事件
};

class Task : public enable_shared_from_this<Task> {
public:
    virtual ~Task() {}

    //! Execute the task in the specified execution mode
    virtual TaskExecutionResult Execute(TaskExecutionMode mode) = 0;

    //! Descheduling a task ensures the task is not executed
    virtual void Deschedule() {
        throw InternalException("Cannot deschedule task of base Task class");
    }

    //! Ensures a task is rescheduled to the correct queue
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
    optional_ptr<ProducerToken> token;  // 关联的生产者令牌
};
```

### 1.5.2 任务执行模式

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        任务执行状态机                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                          ┌──────────────┐                              │
│                          │   PENDING    │                              │
│                          └──────┬───────┘                              │
│                                 │ Schedule                              │
│                                 ▼                                       │
│                          ┌──────────────┐                              │
│              ┌───────────│   RUNNING    │───────────┐                  │
│              │           └──────────────┘           │                  │
│              │                  │                   │                  │
│     PROCESS_PARTIAL        PROCESS_ALL         TASK_BLOCKED           │
│     (部分完成)              (全部完成)          (阻塞)                 │
│              │                  │                   │                  │
│              ▼                  ▼                   ▼                  │
│    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │
│    │NOT_FINISHED  │    │   FINISHED   │    │   BLOCKED    │           │
│    └──────┬───────┘    └──────────────┘    └──────┬───────┘           │
│           │                                        │                   │
│           │ 重新调度                    Callback() │ 唤醒             │
│           └────────────────────────────────────────┘                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1.6 并发配置

### 1.6.1 线程数配置

DuckDB 通过 `threads` 设置控制并行度：

```sql
-- 查看当前线程数
SELECT current_setting('threads');

-- 设置线程数
SET threads = 8;

-- 自动检测（默认）
SET threads = 0;  -- 使用 CPU 核心数
```

### 1.6.2 TaskScheduler 配置

```cpp
// 线程数动态调整
void TaskScheduler::SetThreads(idx_t total_threads, idx_t external_threads) {
    // total_threads: 总线程数（包括外部线程）
    // external_threads: 外部线程数（如主线程）
    // 实际启动 total_threads - external_threads 个工作线程

    lock_guard<mutex> lock(thread_lock);
    requested_thread_count = total_threads;
    RelaunchThreadsInternal(total_threads - external_threads);
}
```

## 1.7 内存序与性能优化

### 1.7.1 内存序基础

```cpp
// 常用内存序
std::memory_order_relaxed  // 最宽松，仅保证原子性
std::memory_order_acquire  // 读屏障，保证后续读不被重排到前面
std::memory_order_release  // 写屏障，保证之前写不被重排到后面
std::memory_order_acq_rel  // 读写屏障
std::memory_order_seq_cst  // 顺序一致性（默认，最严格）
```

### 1.7.2 DuckDB 的内存序使用

DuckDB 在大多数情况下使用默认的顺序一致性，仅在经过验证的热点路径使用宽松内存序：

```cpp
// ConcurrentQueue 中的优化示例
// 使用 memory_order_relaxed 进行计数操作
tail.store(new_tail, std::memory_order_relaxed);

// 使用 memory_order_release 确保数据可见
head.store(new_head, std::memory_order_release);

// 使用 memory_order_acquire 读取最新数据
auto current = head.load(std::memory_order_acquire);
```

## 1.8 小结

本章介绍了 DuckDB 并发基础设施的核心组件：

1. **基础同步原语**：标准库封装的 mutex、atomic，以及自定义的 atomic_ptr

2. **并发设计哲学**：
   - 单写多读模型，简化写冲突处理
   - 乐观并发控制，减少锁等待
   - 选择性使用无锁数据结构

3. **线程模型**：
   - 主线程负责调度，工作线程执行任务
   - ThreadContext 提供线程私有状态
   - 清晰的线程安全边界

4. **Task 任务抽象**：
   - 统一的任务接口
   - 支持部分执行和阻塞恢复
   - 与调度器解耦

这些基础设施为后续章节讨论的高级并发机制奠定了基础。

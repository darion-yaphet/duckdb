# 第十章：死锁预防与调试

## 10.1 概述

在前面九章中，我们详细分析了 DuckDB 的各种并发机制。本章将综合这些知识，讨论死锁预防策略、常见并发问题模式、调试技术以及最佳实践。

```
并发问题分类
===========

┌──────────────────────────────────────────────────────────────────┐
│                        并发问题                                   │
├─────────────────┬─────────────────┬─────────────────┬────────────┤
│     死锁        │     活锁         │    资源饥饿     │   数据竞争  │
│  (Deadlock)     │  (Livelock)     │  (Starvation)   │ (Data Race)│
├─────────────────┼─────────────────┼─────────────────┼────────────┤
│ 循环等待        │ 持续重试无进展    │ 低优先级线程     │ 非原子访问  │
│ 不可中断        │ 资源状态振荡      │ 永远得不到资源   │ 内存可见性  │
└─────────────────┴─────────────────┴─────────────────┴────────────┘
```

## 10.2 锁顺序约定

### 10.2.1 全局锁层次结构

DuckDB 通过严格的锁顺序约定防止死锁：

```
DuckDB 锁层次结构（从高到低）
============================

Level 0: DatabaseInstance 级别
         │
         └─▶ AttachedDatabase.config_lock
         └─▶ ExtensionHelper.lock

Level 1: 事务管理级别
         │
         └─▶ DuckTransactionManager.transaction_lock
         └─▶ DuckTransactionManager.start_transaction_lock
         └─▶ DuckTransactionManager.checkpoint_lock

Level 2: Catalog 级别
         │
         └─▶ DuckCatalog.catalog_lock
         └─▶ DuckCatalog.write_lock
         └─▶ CatalogSet 内部锁

Level 3: 存储级别
         │
         └─▶ DataTableInfo.checkpoint_lock (StorageLock)
         └─▶ DataTableInfo.name_lock
         └─▶ TableIndexList.index_entries_lock

Level 4: Buffer 级别
         │
         └─▶ BufferPool.limit_lock
         └─▶ BlockHandle.lock
         └─▶ EvictionQueue 锁

Level 5: 执行级别
         │
         └─▶ Pipeline.batch_lock
         └─▶ Operator.lock
         └─▶ SegmentTree.node_lock

规则：只能按层次从高到低获取锁，永远不能逆向获取
```

### 10.2.2 锁顺序违反检测

```cpp
// 开发时可启用锁顺序检测（伪代码示例）
#ifdef DEBUG_LOCK_ORDER
class LockOrderChecker {
    thread_local static vector<LockLevel> held_locks;

public:
    static void AcquireLock(LockLevel level) {
        if (!held_locks.empty() && held_locks.back() <= level) {
            throw InternalException("Lock order violation: %s after %s",
                                    LevelToString(level),
                                    LevelToString(held_locks.back()));
        }
        held_locks.push_back(level);
    }

    static void ReleaseLock(LockLevel level) {
        D_ASSERT(held_locks.back() == level);
        held_locks.pop_back();
    }
};
#endif
```

### 10.2.3 事务-检查点锁顺序

事务和检查点之间的锁顺序是最关键的：

```
事务与检查点锁协议
=================

                        DuckTransactionManager
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
    transaction_lock    checkpoint_lock    start_transaction_lock
    (互斥锁)            (读写锁)           (互斥锁)
            │                  │                  │
            └──────────────────┼──────────────────┘
                               │
事务操作顺序：                   │               检查点操作顺序：
1. lock(start_transaction_lock) │               1. lock(transaction_lock)
2. 分配事务ID                    │               2. exclusive(checkpoint_lock)
3. unlock(start_transaction_lock)│              3. 执行检查点
4. 获取 table checkpoint_lock   │               4. unlock all
   (shared)                      │
5. 执行操作                      │

避免死锁的关键：
- 事务使用 SharedCheckpointLock（共享锁）
- 检查点需要独占锁前先获取 transaction_lock
- TryUpgradeLock 用于锁升级，失败时不阻塞
```

## 10.3 常见并发问题模式

### 10.3.1 事务死锁

DuckDB 使用 MVCC 避免读写死锁，但写写冲突仍需处理：

```
写写冲突检测（而非死锁）
======================

Transaction A                          Transaction B
     │                                       │
     │  UPDATE t SET x=1 WHERE id=1          │
     │  ──▶ 创建新版本                        │
     │      timestamp = A.transaction_id     │
     │                                       │
     │                                       │  UPDATE t SET x=2 WHERE id=1
     │                                       │  检测到冲突版本
     │                                       │  ◀── HasConflict() = true
     │                                       │
     │  COMMIT                               │  抛出冲突异常
     │  ──▶ 版本变为已提交                    │  或等待 A 完成后重试
     ▼                                       ▼

DuckDB 策略：
- 使用版本时间戳检测冲突
- 不使用传统的行锁
- 避免了死锁，但可能导致事务回滚
```

### 10.3.2 检查点阻塞

```
检查点阻塞场景
=============

Writer Thread 1                    Checkpoint Thread
     │                                   │
     │  lock(table.checkpoint_lock,      │
     │       SHARED)                     │
     │         │                         │
     │  长时间写入操作                    │  等待获取独占锁
     │  ...                              │  lock(table.checkpoint_lock,
     │  ...                              │       EXCLUSIVE) ──▶ 阻塞
     │  ...                              │
     │  unlock(table.checkpoint_lock)    │
     │                                   │  获取锁，开始检查点
     ▼                                   ▼

缓解策略：
1. 限制单个写入事务的持续时间
2. 使用批量提交减少锁持有时间
3. 在低负载期间调度检查点
```

### 10.3.3 资源饥饿

```
BufferPool 资源饥饿
==================

High-Priority Query                 Low-Priority Query
     │                                   │
     │  Pin large blocks                 │
     │  ──▶ 占用大量内存                  │
     │                                   │
     │  EvictBlocks 驱逐其他块            │  Pin block
     │         │                         │  ──▶ 内存不足
     │         │                         │      等待驱逐
     │         │                         │      │
     │  继续工作                          │      │ 被驱逐的块
     │         │                         │      │ 可能是正在使用的
     │         │                         │      │
     ▼         ▼                         ▼      ▼

解决方案：
1. 内存配额管理
2. 公平调度策略
3. 驱逐优先级控制（LRU + 时间戳）
```

## 10.4 调试机制

### 10.4.1 D_ASSERT 断言

DuckDB 广泛使用 D_ASSERT 进行调试时检查：

```cpp
// src/include/duckdb/common/assert.hpp
#ifdef DEBUG
#define D_ASSERT(condition) \
    duckdb::DuckDBAssertInternal(bool(condition), #condition, __FILE__, __LINE__)
#else
#define D_ASSERT assert
#endif

// 强制断言（始终生效）
#define ALWAYS_ASSERT(condition) \
    duckdb::DuckDBAssertInternal(bool(condition), #condition, __FILE__, __LINE__)
```

### 10.4.2 锁状态验证

BlockHandle 使用 VerifyMutex 确保锁正确持有：

```cpp
// src/storage/buffer/block_handle.cpp
void BlockHandle::VerifyMutex(BlockLock &l) const {
    D_ASSERT(l.owns_lock());           // 确保锁已获取
    D_ASSERT(l.mutex() == &lock);      // 确保是正确的锁
}

// 在每个需要锁保护的方法中调用
void BlockHandle::ChangeMemoryUsage(BlockLock &l, int64_t delta) {
    VerifyMutex(l);  // 验证锁状态
    D_ASSERT(delta < 0);
    memory_usage += static_cast<idx_t>(delta);
    memory_charge.Resize(memory_usage);
}

unique_ptr<FileBuffer> &BlockHandle::GetBuffer(BlockLock &l) {
    VerifyMutex(l);  // 验证锁状态
    return buffer;
}
```

### 10.4.3 锁持有验证模式

```cpp
// 通用模式：使用锁守卫参数验证调用者持有锁
class SegmentTree {
    mutable mutex node_lock;

public:
    // 需要调用者传入锁，证明已获取
    optional_ptr<SegmentNode<T>> GetSegment(SegmentLock &l, idx_t row_number) const {
        // 调用者必须持有 SegmentLock
        return nodes[GetSegmentIndex(l, row_number)].get();
    }

    // 便捷方法自动获取锁
    optional_ptr<SegmentNode<T>> GetSegment(idx_t row_number) const {
        auto l = Lock();  // 获取锁
        return GetSegment(l, row_number);
    }
};
```

### 10.4.4 InterruptState 验证

```cpp
// src/include/duckdb/parallel/interrupt.hpp
class StateWithBlockableTasks {
protected:
    //! 验证锁正确持有
    void AssertLock(StateWithBlockableTasksLock &guard) {
        D_ASSERT(guard.mutex() && RefersToSameObject(*guard.mutex(), lock));
    }

public:
    void BlockTask(StateWithBlockableTasksLock &guard, ...) {
        AssertLock(guard);  // 验证调用者持有锁
        // ...
    }
};
```

## 10.5 调试工具与技术

### 10.5.1 查询性能分析器

```cpp
// src/include/duckdb/main/query_profiler.hpp
class QueryProfiler {
public:
    //! 启用后收集执行统计
    void StartOperator(PhysicalOperator *op);
    void EndOperator(optional_ptr<DataChunk> chunk);

    //! 获取性能报告
    string GetTree();
    string GetJSON();
};

// 使用示例
PRAGMA enable_profiling;
SELECT * FROM large_table WHERE ...;
SELECT * FROM duckdb_profiling_output();
```

### 10.5.2 线程上下文追踪

```cpp
// src/include/duckdb/parallel/thread_context.hpp
class ThreadContext {
public:
    QueryProfiler profiler;  // 每线程性能分析器

    void Flush(ThreadContext &thread) {
        // 合并线程统计到全局
    }
};
```

### 10.5.3 块验证模式

```cpp
// 调试模式下验证块元数据
if (DBConfig::GetSetting<DebugVerifyBlocksSetting>(db.GetDatabase())) {
    // 扩展验证计数
    for (auto &info : segment_infos) {
        verify_block_usage_count[info.block_id]++;
    }
    // 验证块使用
    block_manager.VerifyBlocks(verify_block_usage_count);
}
```

## 10.6 并发编程最佳实践

### 10.6.1 锁粒度选择

```
锁粒度选择指南
=============

┌────────────────────────────────────────────────────────────────┐
│                        粗粒度锁                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 优点：实现简单，无死锁风险                                │  │
│  │ 缺点：并发度低，可能成为瓶颈                              │  │
│  │ 示例：DuckCatalog.catalog_lock                           │  │
│  └─────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────┤
│                        中粒度锁                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 优点：平衡并发度和复杂度                                  │  │
│  │ 缺点：需要仔细设计锁顺序                                  │  │
│  │ 示例：DataTableInfo.checkpoint_lock (StorageLock)        │  │
│  └─────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────┤
│                        细粒度锁                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 优点：高并发度                                            │  │
│  │ 缺点：死锁风险高，实现复杂，内存开销                       │  │
│  │ 示例：BlockHandle.lock（每个块一个锁）                    │  │
│  └─────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────┤
│                        无锁设计                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 优点：最高并发度，无死锁                                  │  │
│  │ 缺点：实现最复杂，调试困难                                │  │
│  │ 示例：ConcurrentQueue, atomic 计数器                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 10.6.2 RAII 锁管理

```cpp
// 推荐：使用 RAII 封装确保锁释放
class Good {
    void Operation() {
        SegmentLock l = tree.Lock();  // 自动获取
        // ... 操作 ...
        // 离开作用域自动释放
    }
};

// 避免：手动锁管理容易出错
class Bad {
    void Operation() {
        tree.node_lock.lock();
        // ... 操作 ...
        if (error) {
            return;  // 忘记释放锁！死锁！
        }
        tree.node_lock.unlock();
    }
};
```

### 10.6.3 锁持有时间最小化

```cpp
// 推荐：只在必要时持有锁
void GoodPattern(DataChunk &chunk) {
    shared_ptr<RowGroup> row_group;
    {
        auto l = collection.Lock();
        row_group = collection.GetRowGroup(l, index);
    }  // 立即释放锁

    // 无锁处理数据
    ProcessData(row_group, chunk);
}

// 避免：长时间持有锁
void BadPattern(DataChunk &chunk) {
    auto l = collection.Lock();
    auto row_group = collection.GetRowGroup(l, index);
    // 持有锁处理数据 - 阻塞其他线程！
    ProcessData(row_group, chunk);
}
```

### 10.6.4 避免锁中调用未知代码

```cpp
// 危险：持有锁时调用回调
void Dangerous(Callback callback) {
    lock_guard<mutex> lock(my_lock);
    callback();  // 回调可能尝试获取其他锁 -> 死锁风险
}

// 安全：释放锁后调用回调
void Safe(Callback callback) {
    Data data;
    {
        lock_guard<mutex> lock(my_lock);
        data = CopyData();
    }  // 释放锁
    callback(data);  // 无锁调用
}
```

## 10.7 常见问题诊断

### 10.7.1 死锁诊断流程

```
死锁诊断流程
===========

症状：查询长时间无响应，CPU 空闲

Step 1: 收集线程栈
─────────────────────────────────
$ gdb -p <pid>
(gdb) thread apply all bt

Step 2: 识别阻塞点
─────────────────────────────────
查找 pthread_mutex_lock, futex 等待
Thread 1: mutex.lock() at storage_lock.cpp:XX
Thread 2: mutex.lock() at catalog_set.cpp:YY

Step 3: 分析等待关系
─────────────────────────────────
Thread 1 持有 Lock A，等待 Lock B
Thread 2 持有 Lock B，等待 Lock A
-> 循环等待 -> 死锁确认

Step 4: 检查锁顺序
─────────────────────────────────
对照锁层次结构图
确认是否违反锁顺序约定
```

### 10.7.2 性能瓶颈诊断

```
并发性能瓶颈诊断
===============

症状：多线程性能不随核心数扩展

检查点 1: 锁争用
─────────────────────────────────
$ perf record -e lock:lock_acquired ./duckdb_test
$ perf report

高争用锁 -> 考虑拆分或使用读写锁

检查点 2: 伪共享
─────────────────────────────────
相邻原子变量在同一缓存行
-> 添加 padding 或分离变量

检查点 3: 串行化点
─────────────────────────────────
查找全局锁或顺序操作
-> 考虑分区或批处理

检查点 4: 内存带宽
─────────────────────────────────
大量跨核数据传输
-> 优化数据局部性
```

## 10.8 DuckDB 并发设计总结

### 10.8.1 设计原则回顾

```
DuckDB 并发设计原则
==================

1. 单写多读 (SWMR)
   ─────────────────
   - 大多数资源使用读写锁
   - 允许并发读，串行写
   - 避免复杂的多写者同步

2. 乐观并发控制 (OCC)
   ─────────────────
   - MVCC 避免读写阻塞
   - 版本检测替代锁等待
   - 冲突时回滚而非阻塞

3. 分层锁设计
   ─────────────────
   - 明确的锁层次结构
   - 从高到低获取锁
   - 防止死锁

4. 局部性优先
   ─────────────────
   - 线程本地状态（LocalState）
   - 分区数据结构
   - 最小化同步点

5. 事件驱动异步
   ─────────────────
   - 任务可中断和恢复
   - 事件依赖图调度
   - 非阻塞式 I/O
```

### 10.8.2 锁类型速查表

| 锁名称 | 类型 | 保护对象 | 获取方式 |
|--------|------|----------|----------|
| StorageLock | 读写锁 | 存储层操作 | GetSharedLock/GetExclusiveLock |
| transaction_lock | 互斥锁 | 事务管理 | lock_guard |
| checkpoint_lock | 读写锁 | 检查点 | SharedCheckpointLock/TryGetCheckpointLock |
| start_transaction_lock | 互斥锁 | 事务启动 | lock_guard |
| catalog_lock | 互斥锁 | 目录操作 | lock_guard |
| cleanup_lock | 互斥锁 | 清理操作 | lock_guard |
| ClientContextLock | 互斥锁 | 客户端上下文 | 构造函数 |
| thread_lock | 互斥锁 | 线程池管理 | lock_guard |
| producer_lock | 互斥锁 | 生产者令牌 | lock_guard |
| node_lock | 互斥锁 | SegmentTree | SegmentLock |
| batch_lock | 互斥锁 | Pipeline 批次索引 | lock_guard |
| BlockHandle.lock | 互斥锁 | 块状态 | BlockLock |

### 10.8.3 同步原语速查表

**标准库封装：**

| 原语 | 头文件 | 用途 |
|------|--------|------|
| mutex | mutex.hpp | 互斥锁 |
| lock_guard | mutex.hpp | RAII 锁守卫 |
| unique_lock | mutex.hpp | 可移动锁守卫 |
| atomic | atomic.hpp | 原子变量 |

**DuckDB 自定义：**

| 类型 | 头文件 | 用途 |
|------|--------|------|
| StorageLock | storage_lock.hpp | 存储层读写锁 |
| StorageLockKey | storage_lock.hpp | 锁句柄 RAII |
| SegmentLock | segment_lock.hpp | 段树锁 RAII |
| BlockLock | block_handle.hpp | 块锁 (unique_lock 别名) |
| InterruptState | interrupt.hpp | 异步中断状态 |
| InterruptDoneSignalState | interrupt.hpp | 同步信号 |
| StateWithBlockableTasks | interrupt.hpp | 可阻塞任务状态 |
| Event | event.hpp | 事件同步 |
| ConcurrentQueue | concurrentqueue.hpp | 无锁任务队列 |
| atomic_ptr | atomic_ptr.hpp | 原子指针封装 |

## 10.9 小结

本章总结了 DuckDB 并发系统的关键设计和实践：

1. **锁顺序约定**：通过严格的锁层次结构防止死锁，从高层（Database/Transaction）到低层（Buffer/Segment）按序获取

2. **常见问题模式**：
   - 事务冲突通过 MVCC 检测而非锁阻塞
   - 检查点阻塞通过读写锁和锁升级缓解
   - 资源饥饿通过公平调度和配额管理解决

3. **调试机制**：
   - D_ASSERT 断言在开发模式下捕获问题
   - VerifyMutex 验证锁正确持有
   - 块验证模式检查数据一致性

4. **最佳实践**：
   - 使用 RAII 封装确保锁释放
   - 最小化锁持有时间
   - 避免在持有锁时调用未知代码
   - 根据场景选择合适的锁粒度

DuckDB 的并发设计体现了实用主义原则：在保证正确性的前提下，优先选择简单、可维护的解决方案，同时在关键路径上使用无锁数据结构和细粒度锁提高性能。

---

## 系列总结

本系列共十章，系统地分析了 DuckDB 的并发与锁机制：

| 章节 | 主题 | 核心内容 |
|------|------|----------|
| 第一章 | 并发基础设施 | mutex, atomic, 线程模型 |
| 第二章 | StorageLock 读写锁 | 读写锁实现, 锁升级, RAII |
| 第三章 | TaskScheduler 任务调度 | ConcurrentQueue, 工作线程, ProducerToken |
| 第四章 | Event 事件驱动 | 依赖计数, 事件链, 任务协作 |
| 第五章 | InterruptState 异步中断 | 中断模式, 回调机制, 阻塞恢复 |
| 第六章 | 事务并发控制 | 事务管理器, 锁体系, 清理机制 |
| 第七章 | Catalog 并发访问 | CatalogSet MVCC, 可见性, 冲突检测 |
| 第八章 | Pipeline 并行执行 | 调度策略, MetaPipeline, 批次索引 |
| 第九章 | 并发数据结构 | atomic_ptr, ConcurrentQueue, SegmentTree |
| 第十章 | 死锁预防与调试 | 锁顺序, 问题诊断, 最佳实践 |

DuckDB 的并发实现展示了如何在嵌入式分析数据库中平衡简单性、正确性和高性能。其设计可为其他数据库系统的并发控制提供有价值的参考。

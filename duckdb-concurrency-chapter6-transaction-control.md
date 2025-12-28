# 第六章：事务并发控制

## 概述

DuckTransactionManager 是 DuckDB 事务管理的核心组件，负责事务的创建、提交、回滚以及并发控制。它通过多层锁机制协调事务之间的访问，确保 ACID 属性的正确性，同时支持自动检查点机制来持久化数据。

本章深入分析事务管理器的核心职责、锁体系、事务状态管理以及清理机制。

## DuckTransactionManager 核心职责

### 类定义

```cpp
// src/include/duckdb/transaction/duck_transaction_manager.hpp
class DuckTransactionManager : public TransactionManager {
public:
    //! 启动新事务
    Transaction &StartTransaction(ClientContext &context) override;
    //! 提交事务
    ErrorData CommitTransaction(ClientContext &context, Transaction &transaction) override;
    //! 回滚事务
    void RollbackTransaction(Transaction &transaction) override;
    //! 执行检查点
    void Checkpoint(ClientContext &context, bool force = false) override;

    //! 检查点锁操作
    unique_ptr<StorageLockKey> SharedCheckpointLock();
    unique_ptr<StorageLockKey> TryGetCheckpointLock();
    unique_ptr<StorageLockKey> TryUpgradeCheckpointLock(StorageLockKey &lock);

private:
    //! 当前启动时间戳
    transaction_t current_start_timestamp;
    //! 当前事务 ID
    transaction_t current_transaction_id;
    //! 最低活跃事务 ID
    atomic<transaction_t> lowest_active_id;
    //! 最低活跃事务启动时间
    atomic<transaction_t> lowest_active_start;
    //! 最后提交时间戳
    atomic<transaction_t> last_commit;
    //! 活跃检查点 ID
    atomic<transaction_t> active_checkpoint;

    //! 活跃事务列表
    vector<unique_ptr<DuckTransaction>> active_transactions;
    //! 最近提交的事务列表
    vector<unique_ptr<DuckTransaction>> recently_committed_transactions;

    //! 事务操作锁
    mutex transaction_lock;
    //! 检查点锁
    StorageLock checkpoint_lock;
    //! 事务启动锁
    mutex start_transaction_lock;
    //! 清理操作锁
    mutex cleanup_lock;
    //! 清理队列锁
    mutex cleanup_queue_lock;
    //! 清理队列
    queue<unique_ptr<DuckCleanupInfo>> cleanup_queue;
};
```

### 职责划分

```
┌─────────────────────────────────────────────────────────────────────┐
│                DuckTransactionManager 职责                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     事务生命周期管理                         │    │
│  │  • StartTransaction - 分配事务 ID 和时间戳                   │    │
│  │  • CommitTransaction - 提交更改，触发检查点                   │    │
│  │  • RollbackTransaction - 回滚更改，清理资源                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     并发控制                                  │    │
│  │  • transaction_lock - 保护事务列表                           │    │
│  │  • checkpoint_lock - 协调检查点操作                          │    │
│  │  • start_transaction_lock - 阻止新事务启动                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     状态跟踪                                  │    │
│  │  • lowest_active_id - 垃圾回收边界                           │    │
│  │  • lowest_active_start - 可见性判断基准                      │    │
│  │  • last_commit - 最新提交点                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     清理机制                                  │    │
│  │  • cleanup_queue - 待清理事务队列                            │    │
│  │  • DuckCleanupInfo - 清理信息聚合                            │    │
│  │  • 顺序清理保证                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 事务锁体系

### 锁类型一览

| 锁名称 | 类型 | 保护对象 | 获取方式 |
|-------|------|---------|---------|
| transaction_lock | mutex | 事务列表和状态 | lock_guard |
| checkpoint_lock | StorageLock | 检查点操作 | GetSharedLock/TryGetExclusiveLock |
| start_transaction_lock | mutex | 新事务启动 | lock_guard |
| cleanup_lock | mutex | 清理执行 | lock_guard |
| cleanup_queue_lock | mutex | 清理队列 | lock_guard |

### 锁层次结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     事务锁层次结构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Level 0: start_transaction_lock                                    │
│           ↓                                                         │
│  Level 1: transaction_lock                                          │
│           ↓                                                         │
│  Level 2: checkpoint_lock                                           │
│           ↓                                                         │
│  Level 3: cleanup_queue_lock                                        │
│           ↓                                                         │
│  Level 4: cleanup_lock                                              │
│                                                                     │
│  规则：持有低级别锁时才能获取高级别锁                                 │
│        不能以相反顺序获取锁                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 启动事务

### StartTransaction 实现

```cpp
Transaction &DuckTransactionManager::StartTransaction(ClientContext &context) {
    auto &meta_transaction = MetaTransaction::Get(context);
    unique_ptr<lock_guard<mutex>> start_lock;

    // 写事务需要获取 start_transaction_lock
    if (!meta_transaction.IsReadOnly()) {
        start_lock = make_uniq<lock_guard<mutex>>(start_transaction_lock);
    }

    // 获取事务锁
    lock_guard<mutex> lock(transaction_lock);

    // 分配时间戳和事务 ID
    transaction_t start_time = current_start_timestamp++;
    transaction_t transaction_id = current_transaction_id++;

    // 更新最低活跃边界
    if (active_transactions.empty()) {
        lowest_active_start = start_time;
        lowest_active_id = transaction_id;
    }

    // 创建事务对象
    auto transaction = make_uniq<DuckTransaction>(*this, context,
                                                   start_time, transaction_id,
                                                   last_committed_version);
    auto &transaction_ref = *transaction;

    // 加入活跃事务列表
    active_transactions.push_back(std::move(transaction));
    return transaction_ref;
}
```

### 事务启动流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                      事务启动流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────┐                                        │
│  │    检查事务类型         │                                        │
│  └───────────┬─────────────┘                                        │
│              │                                                      │
│      ┌───────┴───────┐                                              │
│      │ 只读          │ 读写                                          │
│      ▼               ▼                                              │
│  (跳过)         ┌────────────────────────────┐                       │
│      │         │ 获取 start_transaction_lock │                       │
│      │         │ (阻止 FORCE CHECKPOINT)     │                       │
│      │         └────────────┬───────────────┘                       │
│      │                      │                                       │
│      └──────────────────────┤                                       │
│                             ▼                                       │
│              ┌─────────────────────────────┐                        │
│              │   获取 transaction_lock     │                        │
│              └──────────────┬──────────────┘                        │
│                             │                                       │
│                             ▼                                       │
│              ┌─────────────────────────────┐                        │
│              │   分配 start_time           │                        │
│              │   分配 transaction_id       │                        │
│              └──────────────┬──────────────┘                        │
│                             │                                       │
│                             ▼                                       │
│              ┌─────────────────────────────┐                        │
│              │   更新 lowest_active_*      │                        │
│              │   (如果是第一个活跃事务)     │                        │
│              └──────────────┬──────────────┘                        │
│                             │                                       │
│                             ▼                                       │
│              ┌─────────────────────────────┐                        │
│              │   创建 DuckTransaction      │                        │
│              │   加入 active_transactions  │                        │
│              └─────────────────────────────┘                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 提交事务

### CommitTransaction 实现（简化）

```cpp
ErrorData DuckTransactionManager::CommitTransaction(ClientContext &context, Transaction &transaction_p) {
    auto &transaction = transaction_p.Cast<DuckTransaction>();
    unique_lock<mutex> t_lock(transaction_lock);

    // 1. 检查是否可以检查点
    unique_ptr<StorageLockKey> lock;
    auto undo_properties = transaction.GetUndoProperties();
    auto checkpoint_decision = CanCheckpoint(transaction, lock, undo_properties);

    // 2. 如果需要写 WAL，释放事务锁
    unique_ptr<lock_guard<mutex>> held_wal_lock;
    if (!checkpoint_decision.can_checkpoint && transaction.ShouldWriteToWAL(db)) {
        t_lock.unlock();
        held_wal_lock = storage_manager.GetWALLock();
        transaction.WriteToWAL(context, db, commit_state);
        t_lock.lock();
    }

    // 3. 获取提交时间戳
    CommitInfo info;
    info.commit_id = GetCommitTimestamp();

    // 4. 提交 UndoBuffer
    error = transaction.Commit(db, info, std::move(commit_state));

    // 5. 更新 last_commit
    if (!error.HasError()) {
        last_commit = info.commit_id;
    }

    // 6. 从活跃事务列表移除
    auto cleanup_info = RemoveTransaction(transaction, store_transaction);
    if (cleanup_info->ScheduleCleanup()) {
        lock_guard<mutex> q_lock(cleanup_queue_lock);
        cleanup_queue.emplace(std::move(cleanup_info));
    }

    // 7. 释放事务锁，执行清理
    t_lock.unlock();

    // 8. 执行清理
    {
        lock_guard<mutex> c_lock(cleanup_lock);
        // 从队列获取并执行清理...
    }

    // 9. 如果可以，执行检查点
    if (checkpoint_decision.can_checkpoint) {
        storage_manager.CreateCheckpoint(context, options);
    }

    return error;
}
```

### 提交流程详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                        事务提交流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  持有 transaction_lock                                               │
│  ├─► 检查是否可以检查点 (CanCheckpoint)                              │
│  │   • 检查是否有其他事务                                            │
│  │   • 尝试获取 checkpoint_lock                                     │
│  │                                                                  │
│  ├─► 如果需要写 WAL                                                  │
│  │   ├─► 释放 transaction_lock                                      │
│  │   ├─► 获取 WAL 锁                                                │
│  │   ├─► 写入 WAL                                                   │
│  │   └─► 重新获取 transaction_lock                                  │
│  │                                                                  │
│  ├─► 获取 commit_id                                                 │
│  │                                                                  │
│  ├─► 执行 transaction.Commit()                                      │
│  │   • 提交 UndoBuffer                                              │
│  │   • 使更改对其他事务可见                                          │
│  │                                                                  │
│  ├─► 更新 last_commit                                               │
│  │                                                                  │
│  └─► RemoveTransaction()                                            │
│      • 从 active_transactions 移除                                  │
│      • 可能移入 recently_committed_transactions                     │
│      • 创建 DuckCleanupInfo                                         │
│                                                                     │
│  释放 transaction_lock                                               │
│  ├─► 执行清理 (cleanup_lock)                                        │
│  │                                                                  │
│  └─► 如果可以检查点                                                  │
│      └─► CreateCheckpoint()                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 检查点机制

### CanCheckpoint 决策

```cpp
CheckpointDecision DuckTransactionManager::CanCheckpoint(DuckTransaction &transaction,
                                                          unique_ptr<StorageLockKey> &lock,
                                                          const UndoBufferProperties &undo_properties) {
    // 系统事务不检查点
    if (db.IsSystem()) {
        return CheckpointDecision("system transaction");
    }

    // 只读事务不检查点
    if (transaction.IsReadOnly()) {
        return CheckpointDecision("transaction is read-only");
    }

    // 检查是否需要自动检查点
    if (!transaction.AutomaticCheckpoint(db, undo_properties)) {
        return CheckpointDecision("no reason to automatically checkpoint");
    }

    // 尝试获取检查点锁
    lock = transaction.TryGetCheckpointLock();
    if (!lock) {
        return CheckpointDecision("Failed to obtain checkpoint lock");
    }

    // 检查是否有其他事务
    bool has_other_transactions = HasOtherTransactions(transaction);
    if (has_other_transactions) {
        if (undo_properties.has_updates || undo_properties.has_dropped_entries) {
            // 有更新或删除目录项时不能检查点
            return CheckpointDecision("...");
        }
        // 需要并发检查点
        return CheckpointDecision(CheckpointType::CONCURRENT_CHECKPOINT);
    }

    return CheckpointDecision(CheckpointType::FULL_CHECKPOINT);
}
```

### 强制检查点

```cpp
void DuckTransactionManager::Checkpoint(ClientContext &context, bool force) {
    unique_ptr<StorageLockKey> lock;

    if (!force) {
        // 非强制：立即尝试获取锁
        lock = checkpoint_lock.TryGetExclusiveLock();
        if (!lock) {
            throw TransactionException("Cannot CHECKPOINT: "
                "there are other write transactions active");
        }
    } else {
        // 强制：先阻止新事务，然后等待锁
        lock_guard<mutex> start_lock(start_transaction_lock);
        while (!lock) {
            if (context.interrupted) {
                throw InterruptException();
            }
            lock = checkpoint_lock.TryGetExclusiveLock();
        }
    }

    storage_manager.CreateCheckpoint(context, options);
}
```

### FORCE CHECKPOINT 机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FORCE CHECKPOINT 流程                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────┐                        │
│  │  1. 获取 start_transaction_lock         │                        │
│  │     阻止新的读写事务启动                 │                        │
│  └──────────────────┬──────────────────────┘                        │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  2. 循环尝试获取 checkpoint_lock        │                        │
│  │     (排他锁)                            │                        │
│  │                                         │                        │
│  │     while (!lock) {                     │                        │
│  │         if (interrupted) throw;         │                        │
│  │         lock = TryGetExclusiveLock();   │                        │
│  │     }                                   │                        │
│  └──────────────────┬──────────────────────┘                        │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  3. 等待所有活跃事务完成                 │                        │
│  │     (因为 start_lock 阻止了新事务)       │                        │
│  └──────────────────┬──────────────────────┘                        │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  4. 执行检查点                           │                        │
│  │     CreateCheckpoint()                  │                        │
│  └─────────────────────────────────────────┘                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 事务状态管理

### 活跃事务跟踪

```cpp
// 活跃事务列表
vector<unique_ptr<DuckTransaction>> active_transactions;

// 最低活跃边界（用于垃圾回收）
atomic<transaction_t> lowest_active_id;     // 最低活跃事务 ID
atomic<transaction_t> lowest_active_start;  // 最低活跃启动时间
```

### 最近提交事务

```cpp
// 最近提交的事务列表（可能被其他事务引用）
vector<unique_ptr<DuckTransaction>> recently_committed_transactions;
```

这些事务不能立即清理，因为其他活跃事务可能需要访问它们的数据。

### RemoveTransaction 实现

```cpp
unique_ptr<DuckCleanupInfo> DuckTransactionManager::RemoveTransaction(
    DuckTransaction &transaction, bool store_transaction) noexcept {

    auto cleanup_info = make_uniq<DuckCleanupInfo>();

    // 1. 找到事务并计算新的最低边界
    idx_t t_index = active_transactions.size();
    auto lowest_start_time = TRANSACTION_ID_START;
    auto lowest_transaction_id = MAX_TRANSACTION_ID;

    for (idx_t i = 0; i < active_transactions.size(); i++) {
        if (active_transactions[i].get() == &transaction) {
            t_index = i;
            continue;
        }
        lowest_start_time = MinValue(lowest_start_time,
                                      active_transactions[i]->start_time);
        lowest_transaction_id = MinValue(lowest_transaction_id,
                                          active_transactions[i]->transaction_id);
    }

    // 更新全局边界
    lowest_active_start = lowest_start_time;
    lowest_active_id = lowest_transaction_id;

    // 2. 决定事务去向
    auto current_transaction = std::move(active_transactions[t_index]);
    if (store_transaction) {
        if (transaction.commit_id != 0) {
            // 已提交：加入 recently_committed
            recently_committed_transactions.push_back(std::move(current_transaction));
        } else {
            // 已回滚：直接清理
            cleanup_info->transactions.push_back(std::move(current_transaction));
        }
    }

    // 3. 从活跃列表移除
    active_transactions.unsafe_erase_at(t_index);

    // 4. 检查 recently_committed 是否可以清理
    for (idx_t i = 0; i < recently_committed_transactions.size(); i++) {
        if (recently_committed_transactions[i]->commit_id >= lowest_start_time) {
            break;  // 后续的 commit_id 更大，不需要检查
        }
        // 可以清理
        recently_committed_transactions[i]->awaiting_cleanup = true;
        cleanup_info->transactions.push_back(
            std::move(recently_committed_transactions[i]));
    }

    return cleanup_info;
}
```

### 事务状态转换

```
┌─────────────────────────────────────────────────────────────────────┐
│                      事务状态转换                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌────────────────┐                                                │
│   │ StartTransaction│                                               │
│   └────────┬───────┘                                                │
│            │                                                        │
│            ▼                                                        │
│   ┌────────────────┐                                                │
│   │    ACTIVE      │ ◄── 在 active_transactions 列表中              │
│   └───┬────────┬───┘                                                │
│       │        │                                                    │
│  Commit    Rollback                                                 │
│       │        │                                                    │
│       ▼        ▼                                                    │
│   ┌────────┐  ┌────────────────┐                                    │
│   │COMMITTED│  │   ABORTED     │                                    │
│   └────┬───┘  └───────┬────────┘                                    │
│        │              │                                             │
│        │              │ 直接进入清理队列                              │
│        │              │                                             │
│        ▼              ▼                                             │
│   ┌────────────────────────┐                                        │
│   │ recently_committed     │                                        │
│   │ (等待其他事务不再引用)   │                                        │
│   └───────────┬────────────┘                                        │
│               │                                                     │
│               │ 当 commit_id < lowest_active_start                  │
│               │                                                     │
│               ▼                                                     │
│   ┌────────────────────────┐                                        │
│   │     cleanup_queue      │                                        │
│   │     (待清理队列)         │                                        │
│   └───────────┬────────────┘                                        │
│               │                                                     │
│               ▼                                                     │
│   ┌────────────────────────┐                                        │
│   │       CLEANED          │                                        │
│   │   (资源已释放)          │                                        │
│   └────────────────────────┘                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 清理机制

### DuckCleanupInfo

```cpp
struct DuckCleanupInfo {
    //! 共享最低启动时间
    transaction_t lowest_start_time;
    //! 待清理事务列表
    vector<unique_ptr<DuckTransaction>> transactions;

    void Cleanup() noexcept;
    bool ScheduleCleanup() noexcept;
};

void DuckCleanupInfo::Cleanup() noexcept {
    for (auto &transaction : transactions) {
        if (transaction->awaiting_cleanup) {
            transaction->Cleanup(lowest_start_time);
        }
    }
}
```

### 顺序清理保证

```cpp
// 清理必须按顺序执行
// 例如：事务 A 删除表，事务 B 创建同名表
// 如果颠倒清理顺序会导致 Catalog 错误
queue<unique_ptr<DuckCleanupInfo>> cleanup_queue;
```

清理执行代码：

```cpp
{
    lock_guard<mutex> c_lock(cleanup_lock);  // 确保只有一个清理
    unique_ptr<DuckCleanupInfo> top_cleanup_info;
    {
        lock_guard<mutex> q_lock(cleanup_queue_lock);
        if (!cleanup_queue.empty()) {
            top_cleanup_info = std::move(cleanup_queue.front());
            cleanup_queue.pop();
        }
    }
    if (top_cleanup_info) {
        top_cleanup_info->Cleanup();
    }
}
```

## 回滚事务

### RollbackTransaction 实现

```cpp
void DuckTransactionManager::RollbackTransaction(Transaction &transaction_p) {
    auto &transaction = transaction_p.Cast<DuckTransaction>();

    ErrorData error;
    {
        // 持有 transaction_lock 执行回滚
        lock_guard<mutex> t_lock(transaction_lock);
        error = transaction.Rollback();

        // 从活跃事务列表移除
        auto cleanup_info = RemoveTransaction(transaction);
        if (cleanup_info->ScheduleCleanup()) {
            lock_guard<mutex> q_lock(cleanup_queue_lock);
            cleanup_queue.emplace(std::move(cleanup_info));
        }
    }

    // 执行清理
    {
        lock_guard<mutex> c_lock(cleanup_lock);
        // 处理清理队列...
    }

    if (error.HasError()) {
        throw FatalException("Failed to rollback transaction");
    }
}
```

## 设计特点

### 乐观并发控制

DuckDB 采用乐观并发控制（OCC）：
- 事务执行时不加锁
- 提交时检测冲突
- 冲突时回滚

### 快照隔离

每个事务看到的是启动时刻的数据快照：
- `start_time` 决定可见性
- `lowest_active_start` 用于垃圾回收决策

### 自动检查点

提交时自动判断是否需要检查点：
- 检查 WAL 大小
- 检查是否有其他活跃事务
- 决定检查点类型（FULL 或 CONCURRENT）

## 小结

DuckTransactionManager 事务并发控制的设计体现了几个关键原则：

1. **多层锁保护**：事务锁、检查点锁、启动锁各司其职
2. **乐观并发**：执行时不加锁，提交时检测冲突
3. **延迟清理**：事务提交后不立即清理，等待安全点
4. **顺序清理**：通过队列保证清理操作的正确顺序
5. **自动检查点**：智能决策何时执行检查点

在下一章中，我们将探讨 Catalog 并发访问，了解 DuckDB 如何保护目录结构的并发修改。

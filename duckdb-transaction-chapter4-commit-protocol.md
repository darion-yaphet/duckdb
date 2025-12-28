# 第四章：事务提交协议

事务提交是数据库系统中最关键的操作之一，它需要保证原子性、持久性，同时还要考虑并发性能。本章将深入分析 DuckDB 的事务提交协议，包括 WAL 写入、检查点协调、索引维护和错误处理机制。

## 4.1 提交协议概述

### 4.1.1 提交阶段

DuckDB 的事务提交分为多个阶段：

```
┌─────────────────────────────────────────────────────────────┐
│                     事务提交流程                             │
├─────────────────────────────────────────────────────────────┤
│  Phase 1: 准备阶段 (Prepare)                                │
│  ├── 检查是否需要检查点                                      │
│  ├── 获取必要的锁                                           │
│  └── 分配 commit_id                                         │
├─────────────────────────────────────────────────────────────┤
│  Phase 2: WAL 写入阶段 (Write-Ahead Log)                    │
│  ├── 写入 LocalStorage 数据                                 │
│  ├── 写入 UndoBuffer 条目                                   │
│  └── 写入 WAL_FLUSH 标记                                    │
├─────────────────────────────────────────────────────────────┤
│  Phase 3: 提交阶段 (Commit)                                 │
│  ├── 刷新 LocalStorage 到主存储                             │
│  ├── 提交 UndoBuffer (更新版本号)                           │
│  └── 处理索引删除                                           │
├─────────────────────────────────────────────────────────────┤
│  Phase 4: 完成阶段 (Finalize)                               │
│  ├── 从活跃事务列表移除                                      │
│  ├── 加入已提交事务队列                                      │
│  └── 可选：触发自动检查点                                    │
└─────────────────────────────────────────────────────────────┘
```

### 4.1.2 核心参与者

```
DuckTransactionManager
│   (协调整体流程)
│
├── DuckTransaction
│   │   (事务实例)
│   ├── UndoBuffer      → WAL 写入 + 版本号更新
│   └── LocalStorage    → 数据刷新到主存储
│
├── CommitState
│   │   (提交状态管理)
│   └── IndexDataRemover → 索引删除处理
│
├── WriteAheadLog
│   │   (WAL 写入)
│   └── WALWriteState   → 条目序列化
│
└── StorageLock
        (检查点协调)
```

## 4.2 时间戳分配

### 4.2.1 时间戳初始化

```cpp
// src/transaction/duck_transaction_manager.cpp
DuckTransactionManager::DuckTransactionManager() {
    // start_timestamp 从 2 开始 (0, 1 保留)
    current_start_timestamp = 2;

    // transaction_id 从大数开始，确保 > start_time
    current_transaction_id = TRANSACTION_ID_START;

    lowest_active_start = 0;
    lowest_active_id = 0;
}
```

**设计原因**：

```
时间戳空间:

0                    TRANSACTION_ID_START
│←── start_time ────→│←── transaction_id ───────────→│
│    (2, 3, 4, ...)  │    (很大的数)                  │

保证: transaction_id > 任何 start_time

作用:
  - start_time 用于可见性判断
  - transaction_id 标识未提交事务
  - 两者不重叠，简化比较逻辑
```

### 4.2.2 StartTransaction 分配

```cpp
Transaction &DuckTransactionManager::StartTransaction(ClientContext &context) {
    lock_guard<mutex> lock(transaction_lock);

    // 原子分配时间戳
    transaction_t start_time = current_start_timestamp++;
    transaction_t transaction_id = current_transaction_id++;

    auto transaction = make_uniq<DuckTransaction>(
        *this, context, start_time, transaction_id);

    // 加入活跃事务列表
    active_transactions.push_back(*transaction);

    // 更新最低活跃事务
    if (active_transactions.size() == 1) {
        lowest_active_start = start_time;
        lowest_active_id = transaction_id;
    }

    return *transaction;
}
```

### 4.2.3 GetCommitTimestamp

```cpp
transaction_t DuckTransactionManager::GetCommitTimestamp() {
    // commit_id 使用与 start_time 相同的计数器
    return current_start_timestamp++;
}
```

**时间线示例**：

```
current_start_timestamp: 100

事务 T1 开始: start_time=100, transaction_id=X
事务 T2 开始: start_time=101, transaction_id=X+1
事务 T1 提交: commit_id=102
事务 T3 开始: start_time=103, transaction_id=X+2

T3 的可见性:
  - T1 的修改可见 (commit_id=102 < start_time=103)
  - T2 的修改不可见 (T2 未提交)
```

## 4.3 WAL 写入

### 4.3.1 WriteToWAL 流程

```cpp
// src/transaction/duck_transaction.cpp
void DuckTransaction::WriteToWAL(WriteAheadLog &wal,
                                 optional_ptr<StorageCommitState> commit_state) {
    // 1. 写入 LocalStorage 的数据
    if (storage) {
        storage->WriteToWAL(wal);
    }

    // 2. 写入 UndoBuffer 的条目
    undo_buffer.WriteToWAL(wal, commit_state);

    // 3. 如果有优化写入，同步块管理器
    if (commit_state && commit_state->HasRowGroupData()) {
        // 确保后台写入的数据已落盘
        db.GetStorageManager().GetBlockManager().FileSync();
    }
}
```

### 4.3.2 WAL 条目类型

```cpp
enum class WALType : uint8_t {
    // DDL 操作
    CREATE_TABLE = 1,
    DROP_TABLE = 2,
    CREATE_SCHEMA = 3,
    CREATE_INDEX = 23,
    ALTER_INFO = 20,

    // DML 操作
    USE_TABLE = 25,       // 设置当前表
    INSERT_TUPLE = 26,    // 行插入
    DELETE_TUPLE = 27,    // 行删除
    UPDATE_TUPLE = 28,    // 行更新
    ROW_GROUP_DATA = 29,  // 行组批量数据

    // 控制
    WAL_VERSION = 98,     // WAL 版本头
    CHECKPOINT = 99,      // 检查点标记
    WAL_FLUSH = 100       // 事务边界
};
```

### 4.3.3 WALWriteState 处理

```cpp
void WALWriteState::WriteEntry(WriteAheadLog &wal, UndoFlags type,
                               data_ptr_t data, StorageCommitState *commit_state) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        auto &entry = Load<CatalogEntry*>(data);
        switch (entry->type) {
        case CatalogType::TABLE_ENTRY:
            wal.WriteCreateTable(entry->Cast<TableCatalogEntry>());
            break;
        case CatalogType::INDEX_ENTRY:
            wal.WriteCreateIndex(entry->Cast<IndexCatalogEntry>());
            break;
        // ... 其他目录类型
        }
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        auto &info = Load<AppendInfo>(data);
        // 可能写入 INSERT_TUPLE 或 ROW_GROUP_DATA
        WriteAppend(wal, info, commit_state);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        auto &info = Load<DeleteInfo>(data);
        wal.WriteDelete(info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        auto &info = UpdateInfo::Get(data);
        wal.WriteUpdate(info);
        break;
    }
    // ...
    }
}
```

### 4.3.4 锁释放优化

```cpp
// DuckTransactionManager::CommitTransaction
ErrorData DuckTransactionManager::CommitTransaction(ClientContext &context,
                                                    Transaction &transaction) {
    auto &duck_transaction = transaction.Cast<DuckTransaction>();

    // 检查是否可以检查点
    bool can_checkpoint = CanCheckpoint(duck_transaction);

    if (!can_checkpoint && duck_transaction.ShouldWriteToWAL()) {
        // 关键优化：释放 transaction_lock 再写 WAL
        lock.unlock();

        // WAL 写入可能较慢，期间其他只读事务可以开始
        auto error = duck_transaction.WriteToWAL(*wal);
        if (error.HasError()) {
            return error;
        }

        // 重新获取锁
        lock.lock();
    }

    // 继续提交流程...
}
```

**并发优势**：

```
不优化时:
┌────────────────────────────────────────────────────────────┐
│ T1: [持有 transaction_lock]                                 │
│     │──── WAL 写入 (慢) ────│                              │
│                                                             │
│ T2: [阻塞等待 transaction_lock]                             │
│     │─────────────────────────────│                        │
└────────────────────────────────────────────────────────────┘

优化后:
┌────────────────────────────────────────────────────────────┐
│ T1: [释放锁]                                                │
│     │──── WAL 写入 ────│                                   │
│                         [重新获取锁]                        │
│                                                             │
│ T2: [获取锁] ─ 开始/提交 ─ [释放锁]                         │
│     │────────│                                              │
└────────────────────────────────────────────────────────────┘
```

## 4.4 CommitState：提交状态管理

### 4.4.1 结构定义

```cpp
// src/include/duckdb/transaction/commit_state.hpp
enum class CommitMode {
    COMMIT,         // 正常提交
    REVERT_COMMIT   // 撤销已提交的更改
};

class CommitState {
    DuckTransaction &transaction;
    transaction_t commit_id;
    IndexDataRemover index_data_remover;

public:
    CommitState(DuckTransaction &transaction, transaction_t commit_id,
                ActiveTransactionState state, CommitMode mode);

    void CommitEntry(UndoFlags type, data_ptr_t data);
    void RevertCommit(UndoFlags type, data_ptr_t data);
    void Flush();   // 刷新索引删除
    void Verify();  // 调试验证
};
```

### 4.4.2 CommitEntry 处理

```cpp
void CommitState::CommitEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        // 目录条目提交
        auto &entry = Load<CatalogEntry*>(data);
        auto extra_data = /* 获取额外数据 */;

        if (entry->deleted) {
            // 处理删除的目录条目
            CommitEntryDrop(*entry, extra_data);
        } else {
            // 更新提交 ID
            entry->set->UpdateCommitId(commit_id);
        }
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        // 插入提交 → 更新 insert_id
        auto &info = Load<AppendInfo>(data);
        info.table->CommitAppend(commit_id, info.start_row, info.count);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        // 删除提交 → 更新 delete_id + 处理索引
        auto &info = Load<DeleteInfo>(data);
        CommitDelete(info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        // 更新提交 → 更新版本号
        auto &update = UpdateInfo::Get(data);
        update.version_number.store(commit_id);
        break;
    }
    // ...
    }
}
```

### 4.4.3 IndexDataRemover：索引删除处理

```cpp
struct IndexDataRemover {
    QueryContext context;
    IndexRemovalType removal_type;
    DataChunk chunk;  // 批量处理
    reference_map_t<DataTable, shared_ptr<DataTableInfo>> verify_indexes;

    void PushDelete(DeleteInfo &info) {
        // 收集要删除的行号
        row_t *row_numbers;
        idx_t count;

        if (info.is_consecutive) {
            // 连续删除：生成行号序列
            for (idx_t i = 0; i < info.count; i++) {
                temp_row_numbers[i] = info.base_row + i;
            }
            row_numbers = temp_row_numbers;
            count = info.count;
        } else {
            // 非连续删除：使用存储的行号
            auto rows = info.GetRows();
            for (idx_t i = 0; i < info.count; i++) {
                temp_row_numbers[i] = info.base_row + rows[i];
            }
            row_numbers = temp_row_numbers;
            count = info.count;
        }

        // 批量从索引中删除
        Flush(*info.table, row_numbers, count);
    }

    void Flush(DataTable &table, row_t *row_numbers, idx_t count) {
        // 包装为 Vector
        Vector row_identifiers(LogicalType::ROW_TYPE, ...);
        memcpy(row_identifiers.GetData(), row_numbers,
               count * sizeof(row_t));

        // 从所有索引中删除
        table.RemoveFromIndexes(context, row_identifiers, count);
    }
};
```

### 4.4.4 IndexRemovalType 策略

```cpp
enum class IndexRemovalType {
    MAIN_INDEX_ONLY,       // 无并发事务：立即从索引删除
    MAIN_INDEX,            // 有并发事务：保留 deleted_rows_in_use
    REVERT_MAIN_INDEX_ONLY, // 撤销无并发
    REVERT_MAIN_INDEX       // 撤销有并发
};

IndexRemovalType CommitState::GetIndexRemovalType(
    ActiveTransactionState state, CommitMode mode) {

    if (mode == CommitMode::COMMIT) {
        if (state == ActiveTransactionState::NO_OTHER_TRANSACTIONS) {
            // 没有其他事务，可以直接删除
            return IndexRemovalType::MAIN_INDEX_ONLY;
        }
        // 有并发事务，需要保留
        return IndexRemovalType::MAIN_INDEX;
    } else {
        // REVERT_COMMIT 模式
        if (state == ActiveTransactionState::NO_OTHER_TRANSACTIONS) {
            return IndexRemovalType::REVERT_MAIN_INDEX_ONLY;
        }
        return IndexRemovalType::REVERT_MAIN_INDEX;
    }
}
```

**策略说明**：

```
┌─────────────────────────────────────────────────────────────┐
│ MAIN_INDEX_ONLY (无并发)                                    │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 立即从索引中删除条目                                     │ │
│ │ 不需要追踪 deleted_rows_in_use                          │ │
│ │ 最高效                                                   │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ MAIN_INDEX (有并发)                                         │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 从索引删除，但追踪到 deleted_rows_in_use                 │ │
│ │ 其他事务可能仍需要通过索引找到这些行                     │ │
│ │ 等待其他事务结束后再完全清理                             │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 4.5 StorageLock：检查点协调

### 4.5.1 锁类型

```cpp
// src/include/duckdb/storage/storage_lock.hpp
enum class StorageLockType {
    SHARED = 0,     // 共享锁 (读事务/写事务)
    EXCLUSIVE = 1   // 排他锁 (检查点)
};

class StorageLock {
public:
    // 获取排他锁 (阻塞)
    unique_ptr<StorageLockKey> GetExclusiveLock();

    // 获取共享锁 (阻塞)
    unique_ptr<StorageLockKey> GetSharedLock();

    // 尝试获取排他锁 (非阻塞)
    unique_ptr<StorageLockKey> TryGetExclusiveLock();

    // 特殊：升级检查点锁
    unique_ptr<StorageLockKey> TryUpgradeCheckpointLock(StorageLockKey &lock);
};
```

### 4.5.2 锁升级机制

```cpp
unique_ptr<StorageLockKey> StorageLock::TryUpgradeCheckpointLock(
    StorageLockKey &shared_lock) {
    // 前提：调用者已持有共享锁

    // 检查是否是唯一的共享锁持有者
    if (internals->shared_count == 1) {
        // 可以升级：同时持有共享锁和排他锁
        // 这是检查点专用的特殊状态
        internals->exclusive_lock = true;
        return make_uniq<StorageLockKey>(internals, StorageLockType::EXCLUSIVE);
    }

    // 还有其他共享锁持有者，无法升级
    return nullptr;
}
```

**使用场景**：

```
事务提交时检查点:
┌─────────────────────────────────────────────────────────────┐
│ 1. 事务持有 SharedCheckpointLock (写操作时获取)             │
│                                                              │
│ 2. 想要在提交时触发检查点                                   │
│                                                              │
│ 3. 调用 TryUpgradeCheckpointLock()                          │
│    - 如果是唯一共享锁持有者 → 升级成功                      │
│    - 如果有其他事务持有共享锁 → 升级失败，跳过检查点        │
│                                                              │
│ 4. 升级成功后，同时持有共享锁和排他锁                       │
│    - 共享锁：保证自己的数据访问                             │
│    - 排他锁：阻止其他检查点                                 │
└─────────────────────────────────────────────────────────────┘
```

### 4.5.3 写事务的锁获取

```cpp
void DuckTransaction::SetModifications(DatabaseModificationType type) {
    if (type != DatabaseModificationType::READ) {
        // 写操作需要获取共享检查点锁
        if (!write_lock) {
            write_lock = db.GetStorageManager().SharedCheckpointLock();
        }
    }
}
```

**协调效果**：

```
场景：一个检查点想要运行

活跃事务:
  T1 (写): 持有 SharedLock
  T2 (读): 无锁
  T3 (写): 持有 SharedLock

检查点请求 ExclusiveLock:
  → 阻塞，等待 T1, T3 完成

T1 提交后:
  → 仍阻塞，等待 T3

T3 提交后:
  → 获取 ExclusiveLock，开始检查点
  → 期间新的写事务阻塞在 SharedLock
```

## 4.6 完整提交流程

### 4.6.1 DuckTransactionManager::CommitTransaction

```cpp
ErrorData DuckTransactionManager::CommitTransaction(ClientContext &context,
                                                    Transaction &transaction) {
    auto &duck_transaction = transaction.Cast<DuckTransaction>();
    unique_lock<mutex> lock(transaction_lock);

    // Phase 1: 检查点决策
    auto checkpoint_decision = CanCheckpoint(duck_transaction);

    // Phase 2: WAL 写入 (可能释放锁)
    if (!checkpoint_decision.can_checkpoint &&
        duck_transaction.ShouldWriteToWAL()) {
        lock.unlock();
        auto error = duck_transaction.WriteToWAL(*wal);
        if (error.HasError()) {
            lock.lock();
            Rollback(duck_transaction);
            return error;
        }
        lock.lock();
    }

    // Phase 3: 分配提交时间戳
    transaction_t commit_id = GetCommitTimestamp();

    // Phase 4: 确定活跃事务状态
    ActiveTransactionState state = GetActiveTransactionState(duck_transaction);

    // Phase 5: 执行提交
    try {
        auto error = duck_transaction.Commit(commit_id, state);
        if (error.HasError()) {
            throw TransactionException(error.Message());
        }
    } catch (...) {
        // 提交失败，尝试回滚
        try {
            Rollback(duck_transaction);
        } catch (...) {
            // 回滚也失败 → 致命错误
            throw FatalException("Commit and rollback both failed");
        }
        throw;
    }

    // Phase 6: 从活跃列表移除
    bool store_transaction = duck_transaction.ChangesMade();
    RemoveTransaction(duck_transaction, store_transaction);

    // Phase 7: 可选的后提交检查点
    if (checkpoint_decision.trigger_checkpoint) {
        PerformCheckpoint(checkpoint_decision.type);
    }

    return ErrorData();
}
```

### 4.6.2 DuckTransaction::Commit

```cpp
ErrorData DuckTransaction::Commit(transaction_t commit_id,
                                  ActiveTransactionState state) {
    this->commit_id = commit_id;

    try {
        // 1. 刷新 LocalStorage 到主存储
        if (storage) {
            storage->Commit(commit_id);
        }

        // 2. 提交 UndoBuffer
        UndoBuffer::IteratorState iterator_state;
        CommitInfo info{commit_id, state};
        undo_buffer.Commit(iterator_state, info);

        // 3. 刷新索引删除
        // (在 CommitState 中处理)

        // 4. 刷新 WAL 提交记录
        if (wal) {
            wal->FlushCommit();
        }

        return ErrorData();
    } catch (Exception &ex) {
        // 提交失败，尝试撤销
        RevertCommit(iterator_state, commit_id);
        return ErrorData(ex);
    }
}
```

### 4.6.3 RevertCommit：部分撤销

```cpp
void DuckTransaction::RevertCommit(UndoBuffer::IteratorState &end_state,
                                   transaction_t commit_id) {
    // 1. 撤销 UndoBuffer 中已提交的条目
    undo_buffer.RevertCommit(end_state, transaction_id);

    // 2. 撤销 LocalStorage 的刷新
    if (storage) {
        storage->RevertCommit();
    }

    // 3. 截断 WAL
    if (wal) {
        wal->TruncateToLastCommit();
    }
}
```

**使用场景**：

```
提交过程中发生异常:
┌─────────────────────────────────────────────────────────────┐
│ Commit() 执行:                                               │
│   ✓ LocalStorage.Commit()                                   │
│   ✓ UndoBuffer 条目 1-5 已处理                               │
│   ✗ UndoBuffer 条目 6 处理失败 (异常)                        │
│                                                              │
│ RevertCommit() 撤销:                                         │
│   1. 撤销 UndoBuffer 条目 1-5                                │
│   2. 撤销 LocalStorage                                       │
│   3. 截断 WAL                                                │
└─────────────────────────────────────────────────────────────┘
```

## 4.7 检查点决策

### 4.7.1 CanCheckpoint 逻辑

```cpp
CheckpointDecision DuckTransactionManager::CanCheckpoint(
    DuckTransaction &transaction) {

    // 1. 系统数据库不检查点
    if (db.IsSystem()) {
        return {false, CheckpointType::NONE};
    }

    // 2. 只读事务不触发检查点
    if (transaction.IsReadOnly()) {
        return {false, CheckpointType::NONE};
    }

    // 3. 数据库仍在加载中
    if (db.IsLoading()) {
        return {false, CheckpointType::NONE};
    }

    // 4. 检查大小阈值
    auto props = transaction.GetUndoBufferProperties();
    if (props.estimated_size < GetCheckpointThreshold()) {
        return {false, CheckpointType::NONE};
    }

    // 5. 尝试获取检查点锁
    auto checkpoint_lock = transaction.TryGetCheckpointLock();
    if (!checkpoint_lock) {
        return {false, CheckpointType::NONE};
    }

    // 6. 决定检查点类型
    if (HasOtherActiveTransactions(transaction)) {
        if (props.has_updates || props.has_dropped_entries) {
            // 有更新或删除的目录条目，不能并发检查点
            return {false, CheckpointType::NONE};
        }
        // 只有插入，可以并发检查点
        return {true, CheckpointType::CONCURRENT};
    }

    // 没有其他事务，完整检查点
    return {true, CheckpointType::FULL};
}
```

### 4.7.2 检查点类型

```
┌─────────────────────────────────────────────────────────────┐
│ CheckpointType::FULL (完整检查点)                           │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 条件: 没有其他活跃事务                                   │ │
│ │ 操作: 完整刷新所有数据                                   │ │
│ │ 效果: WAL 可以完全截断                                   │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ CheckpointType::CONCURRENT (并发检查点)                     │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 条件: 有其他活跃事务，但本事务只有插入                   │ │
│ │ 操作: 刷新不影响版本链的数据                             │ │
│ │ 限制: 不处理更新和删除的版本链                           │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ CheckpointType::NONE (不检查点)                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 条件: 不满足上述条件                                     │ │
│ │ 原因: 大小不够 / 有更新需要保留版本链 / 获取锁失败       │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 4.8 事务清理

### 4.8.1 RemoveTransaction

```cpp
void DuckTransactionManager::RemoveTransaction(DuckTransaction &transaction,
                                               bool store_transaction) {
    // 从活跃列表移除
    active_transactions.erase(
        std::find(active_transactions.begin(),
                  active_transactions.end(),
                  std::ref(transaction)));

    // 更新最低活跃事务
    UpdateLowestActiveTransaction();

    if (store_transaction) {
        // 有修改的事务需要保留一段时间
        // 用于 MVCC 版本链清理
        recently_committed_transactions.push_back(
            transaction.shared_from_this());
    }
}
```

### 4.8.2 清理队列

```cpp
void DuckTransactionManager::AddToCleanupQueue(
    unique_ptr<DuckTransaction> transaction) {
    lock_guard<mutex> lock(cleanup_queue_lock);
    cleanup_queue.push_back(std::move(transaction));
}

void DuckTransactionManager::ProcessCleanupQueue() {
    lock_guard<mutex> lock(cleanup_lock);

    while (!cleanup_queue.empty()) {
        auto &transaction = cleanup_queue.front();

        // 检查是否可以清理
        if (transaction->commit_id >= lowest_active_start) {
            // 还有事务可能需要这个版本
            break;
        }

        // 执行清理
        transaction->Cleanup();

        cleanup_queue.pop_front();
    }
}
```

**清理队列的顺序保证**：

```
清理顺序必须与提交顺序一致:

提交顺序: T1 → T2 → T3
清理顺序: T1 → T2 → T3

原因:
  - DDL 操作 (CREATE TABLE, DROP TABLE) 必须按顺序清理
  - 例: DROP TABLE t 后 CREATE TABLE t
        如果先清理 CREATE，t 还存在
        如果先清理 DROP，t 被删除
        只有按提交顺序清理才正确
```

## 4.9 错误处理

### 4.9.1 提交失败处理

```cpp
try {
    auto error = duck_transaction.Commit(commit_id, state);
    if (error.HasError()) {
        throw TransactionException(error.Message());
    }
} catch (Exception &ex) {
    // 提交失败，尝试回滚
    try {
        Rollback(duck_transaction);
    } catch (Exception &rollback_ex) {
        // 回滚也失败 → 数据库可能损坏
        throw FatalException(
            "Transaction commit failed and rollback failed: " +
            ex.Message() + " / " + rollback_ex.Message());
    }
    throw;  // 重新抛出原始异常
}
```

### 4.9.2 索引删除失败

```cpp
void IndexDataRemover::Flush(DataTable &table, row_t *row_numbers, idx_t count) {
    try {
        table.RemoveFromIndexes(context, row_identifiers, count);
    } catch (Exception &ex) {
        // 索引删除失败 → 致命错误
        // 索引与数据不一致是不可接受的
        throw FatalException(
            "Failed to remove entries from index: " + ex.Message());
    }
}
```

## 4.10 小结

DuckDB 的事务提交协议体现了以下设计特点：

| 特点 | 实现方式 |
|------|----------|
| **多阶段提交** | Prepare → WAL → Commit → Finalize |
| **锁释放优化** | WAL 写入时释放 transaction_lock |
| **检查点集成** | 提交时可触发自动检查点 |
| **锁升级** | TryUpgradeCheckpointLock 特殊机制 |
| **索引维护** | IndexDataRemover 批量处理 |
| **错误恢复** | RevertCommit 支持部分撤销 |
| **顺序清理** | 清理队列保证 DDL 操作顺序 |

核心源码位置：
- `src/transaction/duck_transaction_manager.cpp` - 提交协调
- `src/transaction/duck_transaction.cpp` - 事务提交实现
- `src/include/duckdb/transaction/commit_state.hpp` - 提交状态定义
- `src/transaction/commit_state.cpp` - 提交状态实现
- `src/include/duckdb/storage/storage_lock.hpp` - 存储锁定义
- `src/storage/storage_lock.cpp` - 存储锁实现

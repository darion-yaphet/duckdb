# 第五章：垃圾回收与清理

MVCC 系统在运行过程中会产生大量的旧版本数据和元数据。如果不及时清理，这些数据会导致内存和存储空间的无限增长。本章将深入分析 DuckDB 的垃圾回收机制，包括清理时机、清理策略和各类型数据的清理方法。

## 5.1 垃圾回收概述

### 5.1.1 需要清理的数据

```
┌─────────────────────────────────────────────────────────────┐
│                    需要垃圾回收的数据                        │
├─────────────────────────────────────────────────────────────┤
│ 1. UpdateInfo 版本链                                        │
│    - 旧版本的更新值                                         │
│    - 不再被任何事务需要的版本                               │
├─────────────────────────────────────────────────────────────┤
│ 2. DeleteInfo 删除元数据                                    │
│    - 已删除行的可见性信息                                   │
│    - 索引中的 deleted_rows_in_use 追踪                      │
├─────────────────────────────────────────────────────────────┤
│ 3. AppendInfo 插入元数据                                    │
│    - 行的事务可见性信息 (ChunkInfo)                         │
│    - 可以简化为常量状态                                     │
├─────────────────────────────────────────────────────────────┤
│ 4. CatalogEntry 目录条目                                    │
│    - 已删除的表/索引/视图定义                               │
│    - DDL 操作的旧版本                                       │
├─────────────────────────────────────────────────────────────┤
│ 5. UndoBuffer 内存                                          │
│    - 事务完成后不再需要的 Undo 条目                         │
│    - 已清理的事务对象                                       │
└─────────────────────────────────────────────────────────────┘
```

### 5.1.2 清理时机

```
事务生命周期中的清理点:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  事务提交                                                    │
│     │                                                        │
│     ├── 立即清理 (NO_OTHER_TRANSACTIONS)                    │
│     │   - 索引可以直接删除条目                              │
│     │   - 无需追踪 deleted_rows_in_use                      │
│     │                                                        │
│     ├── 延迟清理 (有并发事务)                               │
│     │   - 加入 recently_committed_transactions              │
│     │   - 等待其他事务完成                                  │
│     │                                                        │
│     └── 加入清理队列                                         │
│         │                                                    │
│         ↓                                                    │
│  定期清理                                                    │
│     │                                                        │
│     ├── 检查 lowest_active_start                            │
│     │   - 如果 commit_id < lowest_active_start              │
│     │   - 则该事务的旧版本可以清理                          │
│     │                                                        │
│     └── 执行 Cleanup()                                       │
│         - 按提交顺序处理清理队列                            │
│                                                              │
│  检查点时                                                    │
│     │                                                        │
│     └── 压缩版本链                                          │
│         - 将存活的版本合并到主存储                          │
│         - 可以完全清理已完成的版本链                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 5.2 lowest_active_transaction：清理边界

### 5.2.1 含义

```cpp
// DuckTransactionManager 中维护
atomic<transaction_t> lowest_active_start;
atomic<transaction_t> lowest_active_id;
```

**定义**：所有活跃事务中最小的 start_time

```
活跃事务:
  T1: start_time = 100
  T2: start_time = 150
  T3: start_time = 200

lowest_active_start = 100

含义:
  - 任何 commit_id < 100 的版本，所有活跃事务都已经看不到
  - 这些版本可以安全清理
```

### 5.2.2 更新逻辑

```cpp
void DuckTransactionManager::UpdateLowestActiveTransaction() {
    if (active_transactions.empty()) {
        // 没有活跃事务，所有版本都可以清理
        lowest_active_start = current_start_timestamp;
        lowest_active_id = current_transaction_id;
    } else {
        // 找到最早开始的事务
        transaction_t min_start = TRANSACTION_ID_MAX;
        transaction_t min_id = TRANSACTION_ID_MAX;
        for (auto &tx : active_transactions) {
            if (tx.start_time < min_start) {
                min_start = tx.start_time;
                min_id = tx.transaction_id;
            }
        }
        lowest_active_start = min_start;
        lowest_active_id = min_id;
    }
}
```

### 5.2.3 可见性判断

```
版本 V 是否可以清理?

条件: V.version_number < lowest_active_start

解释:
┌─────────────────────────────────────────────────────────────┐
│ 时间线:                                                      │
│                                                              │
│ V.version_number    lowest_active_start                     │
│        │                    │                                │
│        ↓                    ↓                                │
│ ───────●────────────────────●──────────────────→            │
│        │                    │                                │
│  V 提交                 最早事务开始                          │
│                                                              │
│ 所有活跃事务的 start_time >= lowest_active_start             │
│ 因此所有活跃事务都不需要看到 V                               │
│ V 可以安全清理                                               │
└─────────────────────────────────────────────────────────────┘
```

## 5.3 CleanupState：清理状态管理

### 5.3.1 结构定义

```cpp
// src/include/duckdb/transaction/cleanup_state.hpp
class CleanupState {
    QueryContext context;
    transaction_t lowest_active_transaction;  // 清理边界
    ActiveTransactionState transaction_state; // 并发状态
    IndexDataRemover index_data_remover;      // 索引清理

public:
    CleanupState(const QueryContext &context,
                 transaction_t lowest_active_transaction,
                 ActiveTransactionState transaction_state);

    void CleanupEntry(UndoFlags type, data_ptr_t data);

private:
    void CleanupDelete(DeleteInfo &info);
    void CleanupUpdate(UpdateInfo &info);
};
```

### 5.3.2 CleanupEntry 分发

```cpp
void CleanupState::CleanupEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        // 目录条目清理
        auto &entry = Load<CatalogEntry*>(data);
        if (entry->deleted) {
            // 删除的目录条目可以真正移除
            entry->set->CleanupEntry(*entry);
        }
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        // 插入清理 → 简化版本信息
        auto &info = Load<AppendInfo>(data);
        info.table->CleanupAppend(lowest_active_transaction,
                                  info.start_row, info.count);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        // 删除清理 → 处理索引
        auto &info = Load<DeleteInfo>(data);
        CleanupDelete(info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        // 更新清理 → 移除版本链节点
        auto &info = UpdateInfo::Get(data);
        CleanupUpdate(info);
        break;
    }

    case UndoFlags::SEQUENCE_VALUE:
    case UndoFlags::ATTACHED_DATABASE:
    case UndoFlags::EMPTY_ENTRY:
        // 无需清理
        break;
    }
}
```

## 5.4 各类型数据的清理

### 5.4.1 UpdateInfo 清理

```cpp
void CleanupState::CleanupUpdate(UpdateInfo &info) {
    // 从版本链中移除
    info.segment->CleanupUpdate(info);
}

// UpdateSegment::CleanupUpdate
void UpdateSegment::CleanupUpdate(UpdateInfo &info) {
    lock_guard<StorageLock> lock(segment_lock);

    // 1. 更新前驱的 next 指针
    if (info.HasPrev()) {
        auto prev_pin = info.prev.Pin();
        auto &prev = UpdateInfo::Get(prev_pin);
        prev.next = info.next;
    } else {
        // info 是链头，更新根节点
        root->info[info.vector_index] = info.next;
    }

    // 2. 更新后继的 prev 指针
    if (info.HasNext()) {
        auto next_pin = info.next.Pin();
        auto &next = UpdateInfo::Get(next_pin);
        next.prev = info.prev;
    }

    // info 现在可以被回收
    // (UndoBuffer 块在事务对象销毁时释放)
}
```

**版本链清理图示**：

```
清理前:
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Update A │←──→│ Update B │←──→│ Update C │
│ v=200    │    │ v=150    │    │ v=100    │
└──────────┘    └──────────┘    └──────────┘
                     ↑
              lowest_active = 120
              B.version < 120, 可清理

清理后:
┌──────────┐                   ┌──────────┐
│ Update A │←─────────────────→│ Update C │
│ v=200    │     B 被移除      │ v=100    │
└──────────┘                   └──────────┘
```

### 5.4.2 DeleteInfo 清理

```cpp
void CleanupState::CleanupDelete(DeleteInfo &info) {
    // 检查是否需要清理索引追踪
    if (transaction_state == ActiveTransactionState::NO_OTHER_TRANSACTIONS) {
        // 提交时使用了 MAIN_INDEX_ONLY
        // 没有写入 deleted_rows_in_use，无需清理
        return;
    }

    // 使用 IndexDataRemover 清理
    // 移除 deleted_rows_in_use 中的条目
    index_data_remover.PushDelete(info);
}
```

**IndexRemovalType 对应**：

```
提交时:
  NO_OTHER_TRANSACTIONS → MAIN_INDEX_ONLY (不追踪)
  有并发事务 → MAIN_INDEX (追踪 deleted_rows_in_use)

清理时:
  NO_OTHER_TRANSACTIONS → 跳过 (没有追踪数据)
  有并发事务 → DELETED_ROWS_IN_USE (清理追踪数据)
```

### 5.4.3 AppendInfo 清理

```cpp
void DataTable::CleanupAppend(transaction_t lowest_transaction,
                              idx_t start_row, idx_t count) {
    // 遍历受影响的行组
    row_groups->CleanupAppend(lowest_transaction, start_row, count);
}

void RowGroup::CleanupAppend(transaction_t lowest_transaction,
                             idx_t start, idx_t count) {
    if (!version_info) {
        return;
    }

    // 清理版本信息
    version_info->CleanupAppend(lowest_transaction, start, count);

    // 检查是否可以完全清理版本管理器
    if (version_info->CanCleanup()) {
        version_info.reset();
    }
}
```

**ChunkInfo 简化**：

```
清理前 (ChunkVectorInfo):
┌─────────────────────────────────────────────────────────────┐
│ inserted_data: [100, 100, 100, ..., 100]  (2048 个相同值)   │
│ deleted_data:  [0, 0, 0, ..., 0]          (全部未删除)      │
└─────────────────────────────────────────────────────────────┘

当所有事务都完成 (lowest > 100):
┌─────────────────────────────────────────────────────────────┐
│ 可以转换为 ChunkConstantInfo:                                │
│ insert_id: 100                                               │
│ delete_id: 0                                                 │
│                                                              │
│ 或者完全移除 ChunkInfo (所有行永久可见)                      │
└─────────────────────────────────────────────────────────────┘
```

### 5.4.4 CatalogEntry 清理

```cpp
void CatalogSet::CleanupEntry(CatalogEntry &entry) {
    lock_guard<mutex> lock(catalog_lock);

    // 从目录中完全移除
    entries.erase(entry.name);

    // 释放相关资源
    // (表数据、索引等已经在 DROP 时标记为可回收)
}
```

**DDL 操作的清理顺序**：

```
场景: DROP TABLE t; CREATE TABLE t(新定义);

提交顺序:
  1. DROP TABLE t   (commit_id = 100)
  2. CREATE TABLE t (commit_id = 101)

清理顺序 (必须与提交顺序一致):
  1. 清理 DROP → 移除旧表的 CatalogEntry
  2. 清理 CREATE → (新表保留，无需清理)

如果顺序错误:
  1. 先清理 CREATE → 错误! 新表被移除
  2. 再清理 DROP → 旧表被移除，但新表已经没了
```

## 5.5 清理队列管理

### 5.5.1 队列结构

```cpp
class DuckTransactionManager {
    // 近期提交的事务 (等待进入清理队列)
    vector<unique_ptr<DuckTransaction>> recently_committed_transactions;

    // 清理队列 (FIFO，保持提交顺序)
    deque<unique_ptr<DuckTransaction>> cleanup_queue;

    // 锁
    mutex cleanup_queue_lock;  // 保护队列结构
    mutex cleanup_lock;        // 序列化清理执行
};
```

### 5.5.2 事务流转

```
事务提交
    │
    ↓
┌─────────────────────────────────────────┐
│ recently_committed_transactions         │
│ (暂存，等待其他事务可见性窗口关闭)       │
└─────────────────────────────────────────┘
    │
    │ 当 commit_id < lowest_active_start
    ↓
┌─────────────────────────────────────────┐
│ cleanup_queue                           │
│ (按提交顺序排队，等待清理)               │
└─────────────────────────────────────────┘
    │
    │ ProcessCleanupQueue()
    ↓
┌─────────────────────────────────────────┐
│ 执行 Cleanup()                          │
│ 销毁事务对象                            │
└─────────────────────────────────────────┘
```

### 5.5.3 ProcessCleanupQueue

```cpp
void DuckTransactionManager::ProcessCleanupQueue() {
    // 获取清理锁 (一次只能一个清理)
    lock_guard<mutex> cleanup_guard(cleanup_lock);

    while (true) {
        unique_ptr<DuckTransaction> transaction;

        {
            lock_guard<mutex> queue_guard(cleanup_queue_lock);
            if (cleanup_queue.empty()) {
                break;
            }

            auto &front = cleanup_queue.front();

            // 检查是否可以清理
            if (front->commit_id >= lowest_active_start) {
                // 还有事务可能需要这个版本
                break;
            }

            // 取出事务
            transaction = std::move(cleanup_queue.front());
            cleanup_queue.pop_front();
        }

        // 执行清理 (锁外执行，减少锁持有时间)
        transaction->Cleanup();

        // 事务对象在此销毁
    }
}
```

### 5.5.4 DuckTransaction::Cleanup

```cpp
void DuckTransaction::Cleanup() {
    // 获取活跃事务状态 (保存在提交时)
    auto state = undo_buffer.GetActiveTransactionState();

    // 创建清理状态
    CleanupState cleanup_state(context, lowest_active_transaction, state);

    // 遍历 UndoBuffer 执行清理
    undo_buffer.Cleanup(lowest_active_transaction);
}

void UndoBuffer::Cleanup(transaction_t lowest_active_transaction) {
    IteratorState state;
    IterateEntries(state, [&](UndoFlags type, data_ptr_t data) {
        cleanup_state.CleanupEntry(type, data);
    });
}
```

## 5.6 检查点时的清理

### 5.6.1 检查点与 MVCC

```
检查点时的版本链处理:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│ 完整检查点 (无并发事务):                                     │
│   - 版本链可以完全清理                                       │
│   - 只保留最新值写入主存储                                   │
│   - 旧版本在 UndoBuffer 中，随事务销毁                       │
│                                                              │
│ 并发检查点 (有活跃事务):                                     │
│   - 必须保留版本链                                           │
│   - 活跃事务可能需要旧版本                                   │
│   - 只能写入已提交的最新值                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.6.2 RowGroup 检查点

```cpp
void RowGroup::Checkpoint(TableDataWriter &writer,
                          CheckpointOptions &options) {
    // 写入列数据
    for (auto &column : columns) {
        column->Checkpoint(writer, options);
    }

    // 处理版本信息
    if (version_info) {
        if (options.checkpoint_type == CheckpointType::FULL) {
            // 完整检查点：清理版本信息
            // 因为没有其他事务需要旧版本
            version_info.reset();
        } else {
            // 并发检查点：序列化版本信息
            version_info->Checkpoint(writer);
        }
    }
}
```

### 5.6.3 UpdateSegment 检查点

```cpp
void UpdateSegment::Checkpoint(TableDataWriter &writer) {
    // 遍历所有向量的版本链
    for (idx_t i = 0; i < vector_count; i++) {
        auto &info_ptr = root->info[i];
        if (!info_ptr.IsSet()) {
            continue;
        }

        // 只写入最新的提交值到主存储
        auto pin = info_ptr.Pin();
        auto &info = UpdateInfo::Get(pin);

        // 将更新合并到主存储
        MergeUpdatesToBase(writer, info);

        // 版本链可以清理 (如果是完整检查点)
        if (checkpoint_type == CheckpointType::FULL) {
            root->info[i].Reset();
        }
    }
}
```

## 5.7 内存回收

### 5.7.1 UndoBuffer 块回收

```
UndoBuffer 内存生命周期:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│ 事务执行期间:                                                │
│   - UndoBufferEntry 块按需分配                              │
│   - 由 BufferManager 管理                                   │
│   - 可能溢出到临时文件                                      │
│                                                              │
│ 事务提交后:                                                  │
│   - UndoBuffer 仍然持有块 (MVCC 需要)                       │
│   - 事务对象移入清理队列                                    │
│                                                              │
│ 清理完成后:                                                  │
│   - 事务对象销毁                                            │
│   - UndoBuffer 析构                                         │
│   - UndoBufferEntry 链表释放                                │
│   - BlockHandle 引用计数归零                                │
│   - 内存返回 BufferPool                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.7.2 LocalStorage 回收

```cpp
// LocalStorage 在事务提交时刷新到主存储
void LocalStorage::Commit(transaction_t commit_id) {
    for (auto &entry : table_storage) {
        auto &table = entry.first;
        auto &storage = entry.second;

        // 将本地行组合并到主存储
        storage->FlushToMainStorage(table, commit_id);
    }

    // 清空本地存储
    table_storage.clear();
}

// 回滚时直接清理
void LocalStorage::Rollback() {
    // 直接清空，本地数据从未对外可见
    table_storage.clear();
}
```

## 5.8 清理策略优化

### 5.8.1 批量清理

```cpp
// IndexDataRemover 批量处理
void IndexDataRemover::PushDelete(DeleteInfo &info) {
    // 累积删除操作
    pending_deletes.push_back(info);

    // 达到阈值时批量执行
    if (pending_deletes.size() >= BATCH_SIZE) {
        FlushBatch();
    }
}

void IndexDataRemover::FlushBatch() {
    // 按表分组
    unordered_map<DataTable*, vector<row_t>> per_table;
    for (auto &info : pending_deletes) {
        // 收集行号...
    }

    // 批量从索引删除
    for (auto &[table, rows] : per_table) {
        table->RemoveFromIndexes(context, rows);
    }

    pending_deletes.clear();
}
```

### 5.8.2 延迟清理

```
延迟清理策略:
┌─────────────────────────────────────────────────────────────┐
│ 不立即清理每个事务，而是:                                    │
│                                                              │
│ 1. 累积已提交事务到 recently_committed_transactions          │
│                                                              │
│ 2. 定期检查 (或在特定事件时):                               │
│    - 新事务开始时                                           │
│    - 事务提交时                                             │
│    - 检查点时                                               │
│                                                              │
│ 3. 批量移动到 cleanup_queue                                 │
│                                                              │
│ 4. 批量执行清理                                             │
│                                                              │
│ 优点:                                                        │
│ - 减少锁竞争                                                │
│ - 提高缓存效率                                              │
│ - 减少索引操作频率                                          │
└─────────────────────────────────────────────────────────────┘
```

### 5.8.3 ChunkInfo 优化

```cpp
bool ChunkVectorInfo::Cleanup(transaction_t lowest_transaction) {
    // 检查是否可以简化为常量
    if (HasConstantInsertionId() && !AnyDeleted()) {
        // 所有行有相同的 insert_id 且无删除
        if (ConstantInsertId() < lowest_transaction) {
            // 可以转换为 ChunkConstantInfo 或完全移除
            return true;
        }
    }
    return false;
}
```

## 5.9 并发清理

### 5.9.1 锁策略

```cpp
// 两级锁保护清理操作
mutex cleanup_queue_lock;  // 保护队列结构
mutex cleanup_lock;        // 序列化清理执行

// 清理流程
void ProcessCleanupQueue() {
    lock_guard<mutex> cleanup_guard(cleanup_lock);  // 1. 获取清理锁

    while (true) {
        unique_ptr<DuckTransaction> tx;

        {
            lock_guard<mutex> queue_guard(cleanup_queue_lock);  // 2. 短暂获取队列锁
            if (cleanup_queue.empty()) break;
            tx = std::move(cleanup_queue.front());
            cleanup_queue.pop_front();
        }  // 3. 释放队列锁

        tx->Cleanup();  // 4. 执行清理 (不持有队列锁)
    }
}
```

### 5.9.2 并发安全

```
清理操作的并发安全:
┌─────────────────────────────────────────────────────────────┐
│ UpdateSegment::CleanupUpdate                                │
│ ├── 获取 segment_lock                                       │
│ ├── 修改版本链指针                                          │
│ └── 释放 segment_lock                                       │
│                                                              │
│ 与并发读取:                                                  │
│ ├── 读取操作通过 UndoBufferPointer.Pin() 访问               │
│ ├── Pin 期间数据不会被释放                                  │
│ └── 版本链遍历在 Pin 保护下进行                             │
│                                                              │
│ 与并发写入:                                                  │
│ ├── 写入需要获取 segment_lock                               │
│ └── 与清理互斥                                              │
└─────────────────────────────────────────────────────────────┘
```

## 5.10 小结

DuckDB 的垃圾回收机制体现了以下设计特点：

| 特点 | 实现方式 |
|------|----------|
| **边界判定** | lowest_active_start 确定清理安全边界 |
| **顺序保证** | 清理队列保持提交顺序 (DDL 安全) |
| **类型分发** | CleanupEntry 根据 UndoFlags 分发处理 |
| **批量优化** | IndexDataRemover 批量处理索引删除 |
| **延迟清理** | 累积后批量处理，减少锁竞争 |
| **检查点集成** | 完整检查点时可清理更多数据 |
| **并发安全** | 两级锁 + Pin 保护 |

核心源码位置：
- `src/include/duckdb/transaction/cleanup_state.hpp` - 清理状态定义
- `src/transaction/cleanup_state.cpp` - 清理状态实现
- `src/transaction/duck_transaction_manager.cpp` - 清理队列管理
- `src/transaction/duck_transaction.cpp` - 事务清理入口
- `src/storage/table/update_segment.cpp` - 版本链清理
- `src/storage/table/row_version_manager.cpp` - 行版本清理

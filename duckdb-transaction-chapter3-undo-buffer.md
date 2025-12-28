# 第三章：Undo Buffer 与回滚机制

Undo Buffer 是 DuckDB 事务系统的核心组件，承担着两个关键职责：支持事务回滚和为 MVCC 提供旧版本数据。本章将深入分析 Undo Buffer 的数据结构、内存管理、操作类型以及回滚实现。

## 3.1 Undo Buffer 概述

### 3.1.1 设计目标

Undo Buffer 需要解决以下问题：

| 问题 | 解决方案 |
|------|----------|
| 事务回滚 | 存储操作的逆操作，按逆序应用 |
| MVCC 版本链 | 存储旧版本数据，供并发事务读取 |
| WAL 写入 | 提供事务修改记录供持久化 |
| 内存管理 | 支持大事务，可溢出到临时文件 |
| 垃圾回收 | 清理不再需要的旧版本 |

### 3.1.2 架构位置

```
DuckTransaction
│
├── UndoBuffer (本章重点)
│   │
│   ├── UndoBufferAllocator (内存分配)
│   │   └── UndoBufferEntry 链表 (存储块)
│   │
│   └── Undo 条目
│       ├── CATALOG_ENTRY (DDL 操作)
│       ├── INSERT_TUPLE (插入)
│       ├── DELETE_TUPLE (删除)
│       ├── UPDATE_TUPLE (更新)
│       ├── SEQUENCE_VALUE (序列)
│       └── ATTACHED_DATABASE (数据库挂载)
│
└── LocalStorage (插入数据的实际存储)
```

## 3.2 UndoFlags：操作类型枚举

### 3.2.1 定义

```cpp
// src/include/duckdb/common/enums/undo_flags.hpp
enum class UndoFlags : uint32_t {  // 使用 32 位保证对齐
    EMPTY_ENTRY = 0,      // 空条目 (占位)
    CATALOG_ENTRY = 1,    // DDL 操作 (CREATE/DROP/ALTER)
    INSERT_TUPLE = 2,     // 行插入
    DELETE_TUPLE = 3,     // 行删除
    UPDATE_TUPLE = 4,     // 行更新
    SEQUENCE_VALUE = 5,   // 序列值使用
    ATTACHED_DATABASE = 6 // 数据库挂载
};
```

### 3.2.2 各类型的数据结构

```
┌─────────────────────────────────────────────────────────────┐
│ CATALOG_ENTRY                                               │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ CatalogEntry* + optional extra_data        │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: CREATE TABLE, DROP INDEX, ALTER COLUMN 等 DDL        │
├─────────────────────────────────────────────────────────────┤
│ INSERT_TUPLE                                                │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ AppendInfo {table*, start_row, count}      │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: INSERT INTO ... VALUES                                │
├─────────────────────────────────────────────────────────────┤
│ DELETE_TUPLE                                                │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ DeleteInfo {table*, version_info*, ...}    │ │
│ │            │ + rows[] (可选，非连续删除时)               │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: DELETE FROM ... WHERE                                 │
├─────────────────────────────────────────────────────────────┤
│ UPDATE_TUPLE                                                │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ UpdateInfo {segment*, version_number, ...} │ │
│ │            │ + tuple_ids[] + old_values[]               │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: UPDATE ... SET ... WHERE                              │
├─────────────────────────────────────────────────────────────┤
│ SEQUENCE_VALUE                                              │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ SequenceCatalogEntry* + SequenceValue      │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: nextval('seq') 序列值分配                             │
├─────────────────────────────────────────────────────────────┤
│ ATTACHED_DATABASE                                           │
│ ┌────────────┬────────────────────────────────────────────┐ │
│ │ UndoFlags  │ AttachedDatabase* (数据库指针)             │ │
│ └────────────┴────────────────────────────────────────────┘ │
│ 用途: ATTACH DATABASE 操作                                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2.3 AppendInfo 结构

```cpp
// src/include/duckdb/transaction/append_info.hpp
struct AppendInfo {
    DataTable *table;    // 目标表
    idx_t start_row;     // 起始行号
    idx_t count;         // 插入行数
};
```

**设计特点**：
- 极简结构，仅需 24 字节
- 插入总是连续的，无需记录每行
- 回滚时删除 `[start_row, start_row + count)` 范围的行

### 3.2.4 DeleteInfo 结构

```cpp
// src/include/duckdb/transaction/delete_info.hpp
struct DeleteInfo {
    DataTable *table;              // 目标表
    RowVersionManager *version_info; // 版本管理器
    idx_t vector_idx;              // 向量索引
    idx_t base_row;                // 基础行号
    idx_t count;                   // 删除行数
    bool is_consecutive;           // 是否连续删除

    // 变长数组 (仅 is_consecutive=false 时使用)
    uint16_t rows[];  // rows[i] = 相对于 base_row 的偏移
};
```

**连续删除优化**：

```
DELETE FROM t WHERE id BETWEEN 100 AND 199;

传统方式:
  rows[] = {0, 1, 2, ..., 99}  // 需要 200 字节

优化方式:
  is_consecutive = true
  base_row = 100
  count = 100
  rows[] = 不分配           // 节省 200 字节
```

## 3.3 UndoBuffer 核心结构

### 3.3.1 类定义

```cpp
// src/include/duckdb/transaction/undo_buffer.hpp
class UndoBuffer {
public:
    struct IteratorState {
        BufferHandle handle;              // 当前块的缓冲句柄
        optional_ptr<UndoBufferEntry> current;  // 当前条目
        data_ptr_t start;                 // 块起始位置
        data_ptr_t end;                   // 块结束位置
        bool started = false;
    };

private:
    DuckTransaction &transaction;
    UndoBufferAllocator allocator;        // 内存分配器
    ActiveTransactionState active_transaction_state;

public:
    // 创建新条目
    UndoBufferReference CreateEntry(UndoFlags type, idx_t len);

    // 事务操作
    void Commit(IteratorState &state, CommitInfo &info);
    void Rollback();
    void WriteToWAL(WriteAheadLog &wal, ...);
    void Cleanup(transaction_t lowest_active_transaction);

    // 属性查询
    bool ChangesMade();
    UndoBufferProperties GetProperties();
};
```

### 3.3.2 UndoBufferProperties

```cpp
struct UndoBufferProperties {
    idx_t estimated_size = 0;        // 估计大小 (用于自动检查点)
    bool has_updates = false;        // 是否有更新操作
    bool has_deletes = false;        // 是否有删除操作
    bool has_index_deletes = false;  // 是否有索引删除
    bool has_catalog_changes = false; // 是否有 DDL
    bool has_dropped_entries = false; // 是否有删除的目录条目
};
```

**用途**：
- `estimated_size`: 决定是否触发自动检查点
- `has_catalog_changes`: 影响检查点策略
- `has_dropped_entries`: 需要特殊清理处理

## 3.4 UndoBufferAllocator：内存管理

### 3.4.1 结构设计

```cpp
// src/include/duckdb/transaction/undo_buffer_allocator.hpp
struct UndoBufferAllocator {
    mutex lock;                       // 线程安全
    BufferManager &buffer_manager;    // 缓冲管理器
    unique_ptr<UndoBufferEntry> head; // 链表头
    optional_ptr<UndoBufferEntry> tail; // 链表尾

    UndoBufferReference Allocate(idx_t alloc_len);
};

struct UndoBufferEntry {
    BufferManager &buffer_manager;
    shared_ptr<BlockHandle> block;    // 缓冲区块
    idx_t position = 0;               // 当前写入位置
    idx_t capacity = 0;               // 块容量
    unique_ptr<UndoBufferEntry> next; // 下一个条目
    optional_ptr<UndoBufferEntry> prev; // 上一个条目
};
```

### 3.4.2 内存布局

```
UndoBufferAllocator
│
├── head ────────────────────────────────────────────────────┐
│                                                             │
│   UndoBufferEntry 1 (256KB block)                          │
│   ┌──────────────────────────────────────────────────────┐ │
│   │ [Entry1][Entry2][Entry3]...         [free space]     │ │
│   │ ↑                                   ↑                │ │
│   │ 0                              position              │ │
│   └──────────────────────────────────────────────────────┘ │
│        │                                                    │
│        ↓ next                                               │
│   UndoBufferEntry 2 (256KB block)                          │
│   ┌──────────────────────────────────────────────────────┐ │
│   │ [Entry4][Entry5]...                 [free space]     │ │
│   └──────────────────────────────────────────────────────┘ │
│        │                                                    │
│        ↓ next                                               │
│       ...                                                   │
│                                                             │
└── tail ────────────────────────────────────────────────────┘
```

### 3.4.3 分配策略

```cpp
UndoBufferReference UndoBufferAllocator::Allocate(idx_t alloc_len) {
    lock_guard<mutex> guard(lock);

    // 1. 尝试在当前尾块分配
    if (tail && tail->position + alloc_len <= tail->capacity) {
        idx_t position = tail->position;
        tail->position += alloc_len;
        return UndoBufferReference(*tail, Pin(tail->block), position);
    }

    // 2. 空间不足，分配新块
    auto new_entry = make_uniq<UndoBufferEntry>(buffer_manager);
    new_entry->block = buffer_manager.Allocate(...);
    new_entry->capacity = BLOCK_SIZE;
    new_entry->position = alloc_len;

    // 3. 链接到链表
    new_entry->prev = tail;
    if (tail) {
        tail->next = std::move(new_entry);
        tail = tail->next.get();
    } else {
        head = std::move(new_entry);
        tail = head.get();
    }

    return UndoBufferReference(*tail, Pin(tail->block), 0);
}
```

### 3.4.4 两级引用设计

```cpp
// 轻量级未固定引用 (用于存储在数据结构中)
struct UndoBufferPointer {
    UndoBufferEntry *entry;  // 条目指针
    idx_t position;          // 块内偏移

    UndoBufferReference Pin() const;  // 需要访问时固定
    bool IsSet() const { return entry != nullptr; }
};

// 固定引用 (RAII，持有缓冲区)
struct UndoBufferReference {
    optional_ptr<UndoBufferEntry> entry;
    BufferHandle handle;     // 持有缓冲区 Pin
    idx_t position;

    data_ptr_t Ptr() { return handle.Ptr() + position; }
    bool IsSet() const { return entry != nullptr; }
    UndoBufferPointer GetBufferPointer();
};
```

**设计优势**：

```
UpdateInfo.next/prev 使用 UndoBufferPointer:
┌─────────────────────────────────────────────────────────────┐
│ • 不持有 BufferHandle，不占用缓冲区 Pin                      │
│ • 轻量级 (仅 16 字节: 指针 + 偏移)                          │
│ • 支持版本链条目被换出到临时文件                             │
└─────────────────────────────────────────────────────────────┘

需要访问时调用 Pin():
┌─────────────────────────────────────────────────────────────┐
│ auto ref = pointer.Pin();  // 获取 UndoBufferReference      │
│ auto &info = UpdateInfo::Get(ref);  // 访问数据             │
│ // ref 析构时自动 Unpin                                     │
└─────────────────────────────────────────────────────────────┘
```

## 3.5 条目创建与布局

### 3.5.1 CreateEntry 流程

```cpp
UndoBufferReference UndoBuffer::CreateEntry(UndoFlags type, idx_t len) {
    // 计算总长度 (头部 + 数据)
    idx_t total_len = UNDO_ENTRY_HEADER_SIZE + len;

    // 分配空间
    auto result = allocator.Allocate(total_len);

    // 写入头部
    auto ptr = result.Ptr();
    Store<UndoFlags>(type, ptr);                    // 类型
    Store<uint32_t>(len, ptr + sizeof(UndoFlags)); // 数据长度

    // 返回数据区域的引用
    result.position += UNDO_ENTRY_HEADER_SIZE;
    return result;
}
```

### 3.5.2 条目内存布局

```
单个 Undo 条目:
┌─────────────────┬─────────────────┬──────────────────────────┐
│   UndoFlags     │   uint32_t      │      Data Area           │
│   (4 bytes)     │   length        │      (variable)          │
│                 │   (4 bytes)     │                          │
└─────────────────┴─────────────────┴──────────────────────────┘
│←── UNDO_ENTRY_HEADER_SIZE (8B) ──→│←────── len ──────────────→│

示例 - UPDATE_TUPLE 条目:
┌──────────┬────────┬─────────────────────────────────────────────┐
│ UndoFlags│ length │           UpdateInfo                        │
│ = 4      │ = N    │ ┌────────────────────────────────────────┐  │
│          │        │ │ segment, table, column_index, ...      │  │
│          │        │ │ version_number, N, max, prev, next     │  │
│          │        │ ├────────────────────────────────────────┤  │
│          │        │ │ tuple_ids[max]  (sel_t 数组)           │  │
│          │        │ ├────────────────────────────────────────┤  │
│          │        │ │ old_values[max] (类型相关)             │  │
│          │        │ └────────────────────────────────────────┘  │
└──────────┴────────┴─────────────────────────────────────────────┘
```

## 3.6 事务操作接口

### 3.6.1 写入 Undo 条目

```cpp
// DuckTransaction 中的写入方法

// 写入删除信息
void DuckTransaction::PushDelete(DataTable &table, RowVersionManager &info,
                                 idx_t vector_idx, idx_t count, idx_t base_row) {
    // 检测连续删除优化
    bool is_consecutive = /* 检查 rows 是否为 0,1,2,...,count-1 */;

    idx_t size = sizeof(DeleteInfo);
    if (!is_consecutive) {
        size += sizeof(uint16_t) * count;  // 需要存储行号
    }

    auto entry = undo_buffer->CreateEntry(UndoFlags::DELETE_TUPLE, size);
    auto &delete_info = entry.Get<DeleteInfo>();
    delete_info.table = &table;
    delete_info.version_info = &info;
    delete_info.vector_idx = vector_idx;
    delete_info.count = count;
    delete_info.base_row = base_row;
    delete_info.is_consecutive = is_consecutive;

    if (!is_consecutive) {
        memcpy(delete_info.rows, rows, sizeof(uint16_t) * count);
    }
}

// 写入插入信息
void DuckTransaction::PushAppend(DataTable &table, idx_t start_row, idx_t count) {
    auto entry = undo_buffer->CreateEntry(UndoFlags::INSERT_TUPLE,
                                          sizeof(AppendInfo));
    auto &info = entry.Get<AppendInfo>();
    info.table = &table;
    info.start_row = start_row;
    info.count = count;
}

// 创建更新信息
UndoBufferReference DuckTransaction::CreateUpdateInfo(idx_t type_size,
                                                      idx_t count) {
    idx_t alloc_size = UpdateInfo::GetAllocSize(type_size, count);
    return undo_buffer->CreateEntry(UndoFlags::UPDATE_TUPLE, alloc_size);
}
```

### 3.6.2 遍历模式

```cpp
// 正向遍历 (从旧到新) - 用于 Commit, WriteToWAL
template <class T>
void UndoBuffer::IterateEntries(IteratorState &state, T &&callback) {
    // 从 tail 开始 (最旧的块)
    auto current = tail;
    while (current) {
        auto handle = Pin(current->block);
        auto ptr = handle.Ptr();
        auto end = ptr + current->position;

        while (ptr < end) {
            UndoFlags type = Load<UndoFlags>(ptr);
            uint32_t len = Load<uint32_t>(ptr + sizeof(UndoFlags));

            callback(type, ptr + UNDO_ENTRY_HEADER_SIZE);

            ptr += UNDO_ENTRY_HEADER_SIZE + len;
        }
        current = current->prev.get();  // 向前遍历
    }
}

// 逆向遍历 (从新到旧) - 用于 Rollback
template <class T>
void UndoBuffer::ReverseIterateEntries(T &&callback) {
    // 从 head 开始 (最新的块)
    auto current = head.get();
    while (current) {
        auto handle = Pin(current->block);

        // 收集该块内的所有条目位置
        vector<data_ptr_t> entries;
        auto ptr = handle.Ptr();
        auto end = ptr + current->position;
        while (ptr < end) {
            entries.push_back(ptr);
            uint32_t len = Load<uint32_t>(ptr + sizeof(UndoFlags));
            ptr += UNDO_ENTRY_HEADER_SIZE + len;
        }

        // 逆序回调
        for (idx_t i = entries.size(); i > 0; i--) {
            auto entry_ptr = entries[i - 1];
            UndoFlags type = Load<UndoFlags>(entry_ptr);
            callback(type, entry_ptr + UNDO_ENTRY_HEADER_SIZE);
        }

        current = current->next.get();  // 向后遍历
    }
}
```

## 3.7 回滚机制

### 3.7.1 Rollback 流程

```cpp
void UndoBuffer::Rollback() {
    ReverseIterateEntries([&](UndoFlags type, data_ptr_t data) {
        RollbackState::RollbackEntry(type, data);
    });
}
```

**逆序回滚的重要性**：

```
事务操作序列:
1. INSERT INTO t VALUES (1)      → AppendInfo{row=100}
2. UPDATE t SET x=2 WHERE id=1   → UpdateInfo{row=100, old=1}
3. DELETE FROM t WHERE id=1      → DeleteInfo{row=100}

正确的回滚顺序 (逆序):
3. 撤销 DELETE → 恢复行的可见性
2. 撤销 UPDATE → 恢复旧值 x=1
1. 撤销 INSERT → 删除行 100

错误的回滚顺序 (正序):
1. 撤销 INSERT → 删除行 100 ← 但行已被删除标记!
   后续操作会出现不一致
```

### 3.7.2 RollbackState 实现

```cpp
// src/transaction/rollback_state.cpp
void RollbackState::RollbackEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        // DDL 回滚
        auto &catalog_entry = Load<CatalogEntry*>(data);
        catalog_entry->set->Undo(catalog_entry);
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        // 插入回滚 → 删除已插入的行
        auto &info = Load<AppendInfo>(data);
        info.table->RevertAppend(info.start_row, info.count);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        // 删除回滚 → 恢复行的可见性
        auto &info = Load<DeleteInfo>(data);
        // 使用 NOT_DELETED_ID 标记行为"未删除"
        info.version_info->CommitDelete(NOT_DELETED_ID, info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        // 更新回滚 → 从版本链移除
        auto &info = UpdateInfo::Get(data);
        info.segment->RollbackUpdate(info);
        break;
    }

    case UndoFlags::ATTACHED_DATABASE: {
        // 数据库挂载回滚 → 分离
        auto &db = Load<AttachedDatabase*>(data);
        db_manager.DetachInternal(db);
        break;
    }

    case UndoFlags::SEQUENCE_VALUE:
        // 序列值无需显式回滚
        // 事务中的 sequence_usage 映射会被丢弃
        break;

    case UndoFlags::EMPTY_ENTRY:
        break;
    }
}
```

### 3.7.3 回滚各操作类型

```
┌─────────────────────────────────────────────────────────────┐
│ CATALOG_ENTRY 回滚                                          │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ CREATE TABLE t → Undo() 删除表 t                        │ │
│ │ DROP TABLE t   → Undo() 恢复表 t                        │ │
│ │ ALTER TABLE t  → Undo() 恢复旧定义                      │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ INSERT_TUPLE 回滚                                           │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ RevertAppend(start_row, count)                          │ │
│ │   → 将 [start_row, start_row+count) 范围的行标记为无效  │ │
│ │   → 更新 RowVersionManager 移除版本信息                 │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ DELETE_TUPLE 回滚                                           │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ CommitDelete(NOT_DELETED_ID, info)                      │ │
│ │   → 将 delete_id 重置为 0 (未删除)                      │ │
│ │   → 行重新对所有事务可见                                │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ UPDATE_TUPLE 回滚                                           │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ RollbackUpdate(info)                                    │ │
│ │   → 从版本链中移除 UpdateInfo                           │ │
│ │   → 更新 prev/next 指针                                 │ │
│ │   → 恢复原始值到列数据                                  │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3.8 提交与 WAL 写入

### 3.8.1 Commit 流程

```cpp
void UndoBuffer::Commit(IteratorState &state, CommitInfo &info) {
    // 保存当前活跃事务状态
    active_transaction_state = GetActiveTransactionState();

    IterateEntries(state, [&](UndoFlags type, data_ptr_t data) {
        CommitState::CommitEntry(info, type, data);
    });
}
```

### 3.8.2 CommitState 处理

```cpp
void CommitState::CommitEntry(CommitInfo &info, UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        // DDL 提交 → 目录变更永久生效
        auto &entry = Load<CatalogEntry*>(data);
        entry->set->UpdateCommitId(info.commit_id);
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        // 插入提交 → 更新 insert_id
        auto &append = Load<AppendInfo>(data);
        append.table->CommitAppend(info.commit_id,
                                   append.start_row, append.count);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        // 删除提交 → 更新 delete_id
        auto &delete_info = Load<DeleteInfo>(data);
        delete_info.version_info->CommitDelete(info.commit_id, delete_info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        // 更新提交 → 更新版本号
        auto &update = UpdateInfo::Get(data);
        update.version_number = info.commit_id;
        break;
    }
    // ... 其他类型
    }
}
```

### 3.8.3 WriteToWAL 流程

```cpp
void UndoBuffer::WriteToWAL(WriteAheadLog &wal,
                            optional_ptr<StorageCommitState> commit_state) {
    IteratorState state;
    IterateEntries(state, [&](UndoFlags type, data_ptr_t data) {
        WALWriteState::WriteEntry(wal, type, data, commit_state);
    });
}
```

**WAL 条目对应关系**：

| UndoFlags | WAL 类型 |
|-----------|----------|
| CATALOG_ENTRY | CREATE_TABLE / DROP_TABLE / ALTER_INFO |
| INSERT_TUPLE | INSERT_TUPLE 或 ROW_GROUP_DATA |
| DELETE_TUPLE | DELETE_TUPLE |
| UPDATE_TUPLE | UPDATE_TUPLE |
| SEQUENCE_VALUE | SEQUENCE_VALUE |

## 3.9 垃圾回收

### 3.9.1 Cleanup 流程

```cpp
void UndoBuffer::Cleanup(transaction_t lowest_active_transaction) {
    IteratorState state;
    IterateEntries(state, [&](UndoFlags type, data_ptr_t data) {
        CleanupState::CleanupEntry(lowest_active_transaction, type, data);
    });
}
```

### 3.9.2 CleanupState 处理

```cpp
void CleanupState::CleanupEntry(transaction_t lowest_active,
                                UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::CATALOG_ENTRY: {
        // 清理已删除的目录条目
        auto &entry = Load<CatalogEntry*>(data);
        if (entry->deleted && entry->commit_id < lowest_active) {
            entry->set->EraseEntry(entry);
        }
        break;
    }

    case UndoFlags::INSERT_TUPLE: {
        // 清理已提交的插入元数据
        auto &info = Load<AppendInfo>(data);
        info.table->CleanupAppend(info.start_row, info.count);
        break;
    }

    case UndoFlags::DELETE_TUPLE: {
        // 清理已提交的删除版本信息
        auto &info = Load<DeleteInfo>(data);
        info.version_info->CleanupDelete(info);
        break;
    }

    case UndoFlags::UPDATE_TUPLE: {
        // 清理不再需要的更新版本
        auto &update = UpdateInfo::Get(data);
        if (update.version_number < lowest_active) {
            update.segment->CleanupUpdate(update);
        }
        break;
    }
    // ...
    }
}
```

### 3.9.3 清理时机

```
事务提交后:
┌─────────────────────────────────────────────────────────────┐
│ 1. 事务进入 recently_committed_transactions                  │
│                                                              │
│ 2. 定期检查 lowest_active_start                              │
│    - 如果没有活跃事务需要看到该版本                          │
│    - 触发 Cleanup()                                          │
│                                                              │
│ 3. 移动到 cleanup_queue                                      │
│    - 按顺序清理 (DDL 操作需要保持顺序)                       │
│                                                              │
│ 4. 完成清理后销毁事务对象                                    │
└─────────────────────────────────────────────────────────────┘
```

## 3.10 GetProperties 分析

```cpp
UndoBufferProperties UndoBuffer::GetProperties() {
    UndoBufferProperties properties;
    IteratorState state;

    IterateEntries(state, [&](UndoFlags type, data_ptr_t data) {
        switch (type) {
        case UndoFlags::INSERT_TUPLE: {
            auto &info = Load<AppendInfo>(data);
            properties.estimated_size += sizeof(AppendInfo);
            properties.estimated_size += info.count * sizeof(row_t);
            break;
        }

        case UndoFlags::DELETE_TUPLE: {
            auto &info = Load<DeleteInfo>(data);
            properties.has_deletes = true;
            properties.estimated_size += sizeof(DeleteInfo);
            if (!info.is_consecutive) {
                properties.estimated_size += info.count * sizeof(uint16_t);
            }
            break;
        }

        case UndoFlags::UPDATE_TUPLE: {
            properties.has_updates = true;
            // 估算更新大小...
            break;
        }

        case UndoFlags::CATALOG_ENTRY: {
            properties.has_catalog_changes = true;
            auto &entry = Load<CatalogEntry*>(data);
            if (entry->deleted) {
                properties.has_dropped_entries = true;
            }
            // 检查索引创建
            if (entry->type == CatalogType::INDEX_ENTRY) {
                auto &index = entry->Cast<IndexCatalogEntry>();
                properties.estimated_size += index.initial_index_size;
            }
            break;
        }
        // ...
        }
    });

    return properties;
}
```

**用途**：
- 自动检查点决策基于 `estimated_size`
- 检查点策略基于 `has_catalog_changes`
- 清理顺序基于 `has_dropped_entries`

## 3.11 小结

DuckDB 的 Undo Buffer 设计体现了以下特点：

| 特点 | 实现方式 |
|------|----------|
| **双重用途** | 既支持回滚，又提供 MVCC 旧版本 |
| **内存效率** | 链式块分配 + 连续删除优化 |
| **懒加载** | UndoBufferPointer 延迟 Pin |
| **类型分发** | UndoFlags 枚举驱动处理逻辑 |
| **正逆遍历** | 正向 Commit，逆向 Rollback |
| **属性分析** | GetProperties() 支持决策 |

核心源码位置：
- `src/include/duckdb/transaction/undo_buffer.hpp` - UndoBuffer 定义
- `src/transaction/undo_buffer.cpp` - UndoBuffer 实现
- `src/include/duckdb/transaction/undo_buffer_allocator.hpp` - 内存分配器
- `src/include/duckdb/common/enums/undo_flags.hpp` - 操作类型枚举
- `src/include/duckdb/transaction/append_info.hpp` - 插入信息
- `src/include/duckdb/transaction/delete_info.hpp` - 删除信息
- `src/include/duckdb/transaction/update_info.hpp` - 更新信息
- `src/transaction/rollback_state.cpp` - 回滚逻辑
- `src/transaction/commit_state.cpp` - 提交逻辑
- `src/transaction/cleanup_state.cpp` - 清理逻辑

# 第七章：Catalog 并发访问

## 概述

CatalogSet 是 DuckDB 目录系统的核心数据结构，管理表、视图、函数等元数据对象。它采用版本链机制支持 MVCC（多版本并发控制），通过时间戳比较实现事务可见性判断，保证目录操作的并发正确性。

本章深入分析 CatalogSet 的并发模型、版本链管理、事务可见性判断以及写写冲突检测。

## CatalogSet 并发模型

### 核心结构

```cpp
// src/include/duckdb/catalog/catalog_set.hpp
class CatalogSet {
public:
    struct EntryLookup {
        enum class FailureReason {
            SUCCESS,      // 成功找到
            DELETED,      // 已删除
            NOT_PRESENT,  // 不存在
            INVISIBLE     // 对当前事务不可见
        };
        optional_ptr<CatalogEntry> result;
        FailureReason reason;
    };

    bool CreateEntry(CatalogTransaction transaction, const string &name,
                     unique_ptr<CatalogEntry> value,
                     const LogicalDependencyList &dependencies);

    bool AlterEntry(CatalogTransaction transaction, const string &name,
                    AlterInfo &alter_info);

    bool DropEntry(CatalogTransaction transaction, const string &name,
                   bool cascade, bool allow_drop_internal = false);

    optional_ptr<CatalogEntry> GetEntry(CatalogTransaction transaction,
                                         const string &name);

    mutex &GetCatalogLock() {
        return catalog_lock;
    }

private:
    DuckCatalog &catalog;
    //! 目录锁，用于修改数据
    mutex catalog_lock;
    //! 条目映射表
    CatalogEntryMap map;
    //! 默认条目生成器
    unique_ptr<DefaultGenerator> defaults;
};
```

### 写时复制机制

CatalogSet 采用写时复制（Copy-on-Write）策略：
- 修改操作创建新版本
- 新版本链接到旧版本
- 形成版本链

```
┌─────────────────────────────────────────────────────────────────────┐
│                     版本链结构                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CatalogEntryMap                                                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  "my_table" ──►┌─────────────────┐                         │     │
│  │                │ Version 3       │◄── 最新版本              │     │
│  │                │ timestamp: T100 │    (事务 T100 创建)      │     │
│  │                │ deleted: false  │                         │     │
│  │                └────────┬────────┘                         │     │
│  │                         │ child                            │     │
│  │                         ▼                                  │     │
│  │                ┌─────────────────┐                         │     │
│  │                │ Version 2       │◄── 已提交版本            │     │
│  │                │ timestamp: 50   │    (在时间 50 提交)      │     │
│  │                │ deleted: false  │                         │     │
│  │                └────────┬────────┘                         │     │
│  │                         │ child                            │     │
│  │                         ▼                                  │     │
│  │                ┌─────────────────┐                         │     │
│  │                │ Version 1       │◄── 初始版本              │     │
│  │                │ timestamp: 0    │    (系统创建)            │     │
│  │                │ deleted: false  │                         │     │
│  │                └─────────────────┘                         │     │
│  │                                                            │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 事务可见性判断

### 时间戳规则

```cpp
// src/catalog/catalog_set.cpp
bool CatalogSet::UseTimestamp(CatalogTransaction transaction, transaction_t timestamp) {
    if (timestamp == transaction.transaction_id) {
        // 我们自己创建的版本
        return true;
    }
    if (timestamp < transaction.start_time) {
        // 在我们启动之前已提交的版本
        return true;
    }
    return false;
}
```

一个版本对事务可见的条件：
1. **自己创建**：`timestamp == transaction.transaction_id`
2. **已提交且早于启动**：`timestamp < transaction.start_time`

### 冲突检测

```cpp
bool CatalogSet::CreatedByOtherActiveTransaction(CatalogTransaction transaction,
                                                  transaction_t timestamp) {
    // 其他活跃（未提交）事务创建的版本
    return (timestamp >= TRANSACTION_ID_START && timestamp != transaction.transaction_id);
}

bool CatalogSet::CommittedAfterStarting(CatalogTransaction transaction,
                                        transaction_t timestamp) {
    // 在我们启动后提交的版本
    return (timestamp < TRANSACTION_ID_START && timestamp > transaction.start_time);
}

bool CatalogSet::HasConflict(CatalogTransaction transaction, transaction_t timestamp) {
    return CreatedByOtherActiveTransaction(transaction, timestamp) ||
           CommittedAfterStarting(transaction, timestamp);
}
```

### 时间戳判断流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    时间戳可见性判断                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  事务 T: start_time = 100, transaction_id = T100                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    条目时间戳                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│        │                                                            │
│        ▼                                                            │
│  ┌──────────────────────┐                                           │
│  │ timestamp >= TXID_START?                                         │
│  │ (是否是事务 ID)       │                                          │
│  └──────────┬───────────┘                                           │
│             │                                                       │
│     ┌───────┴───────┐                                               │
│     │ 是            │ 否 (已提交)                                    │
│     ▼               ▼                                               │
│  ┌─────────────┐ ┌─────────────────────┐                            │
│  │ == T100?   │ │ < start_time (100)? │                            │
│  └─────┬───────┘ └──────────┬──────────┘                            │
│        │                    │                                       │
│    ┌───┴───┐            ┌───┴───┐                                   │
│    │是     │否          │是     │否                                  │
│    ▼       ▼            ▼       ▼                                   │
│ ┌──────┐ ┌──────┐   ┌──────┐ ┌──────┐                               │
│ │可见  │ │冲突  │   │可见  │ │冲突  │                               │
│ │(自己)│ │(其他)│   │(历史)│ │(新)  │                               │
│ └──────┘ └──────┘   └──────┘ └──────┘                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 版本链遍历

### GetEntryForTransaction

```cpp
CatalogEntry &CatalogSet::GetEntryForTransaction(CatalogTransaction transaction,
                                                  CatalogEntry &current,
                                                  bool &visible) {
    reference<CatalogEntry> entry(current);
    while (entry.get().HasChild()) {
        if (UseTimestamp(transaction, entry.get().timestamp)) {
            visible = true;
            return entry.get();
        }
        entry = entry.get().Child();
    }
    visible = false;
    return entry.get();
}
```

遍历版本链，找到对当前事务可见的版本：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   版本链遍历示例                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  事务 T: start_time = 80                                            │
│                                                                     │
│  版本链:                                                            │
│  ┌────────────────────┐                                             │
│  │ Version 3          │ timestamp = T100 (未提交)                    │
│  │ UseTimestamp = false│ ───► 跳过                                  │
│  └─────────┬──────────┘                                             │
│            │                                                        │
│            ▼                                                        │
│  ┌────────────────────┐                                             │
│  │ Version 2          │ timestamp = 90 (在 T 启动后提交)             │
│  │ UseTimestamp = false│ ───► 跳过                                  │
│  └─────────┬──────────┘                                             │
│            │                                                        │
│            ▼                                                        │
│  ┌────────────────────┐                                             │
│  │ Version 1          │ timestamp = 50 (在 T 启动前提交)             │
│  │ UseTimestamp = true │ ───► 返回此版本 ✓                          │
│  └────────────────────┘                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 创建条目

### CreateEntry 实现

```cpp
bool CatalogSet::CreateEntry(CatalogTransaction transaction, const string &name,
                              unique_ptr<CatalogEntry> value,
                              const LogicalDependencyList &dependencies) {
    CheckCatalogEntryInvariants(*value, name);

    // 标记为当前事务创建
    value->timestamp = transaction.transaction_id;
    value->set = this;

    // 添加依赖关系
    catalog.GetDependencyManager()->AddObject(transaction, *value, dependencies);

    // 获取写锁
    lock_guard<mutex> write_lock(catalog.GetWriteLock());
    // 获取目录集读锁
    unique_lock<mutex> read_lock(catalog_lock);

    return CreateEntryInternal(transaction, name, std::move(value), read_lock);
}
```

### 双锁协议

创建条目需要两把锁：
1. **catalog.GetWriteLock()** - 全局写锁
2. **catalog_lock** - 当前 CatalogSet 的锁

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CreateEntry 锁协议                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. lock_guard<mutex> write_lock(catalog.GetWriteLock())            │
│     └─► 确保同一时间只有一个写操作                                   │
│                                                                     │
│  2. unique_lock<mutex> read_lock(catalog_lock)                      │
│     └─► 阻止其他事务读取正在修改的数据                               │
│                                                                     │
│  顺序重要：先全局锁，后局部锁                                        │
│           避免死锁                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 版本空位检查

```cpp
bool CatalogSet::VerifyVacancy(CatalogTransaction transaction, CatalogEntry &entry) {
    if (HasConflict(transaction, entry.timestamp)) {
        // 另一个事务已经修改了这个条目
        throw TransactionException("Catalog write-write conflict on create with \"%s\"",
                                   entry.name);
    }
    // 条目必须是删除状态才能重新创建
    if (!entry.deleted) {
        return false;
    }
    return true;
}
```

### 写写冲突检测

```cpp
optional_ptr<CatalogEntry> CatalogSet::GetEntryInternal(CatalogTransaction transaction,
                                                        const string &name) {
    auto entry_value = map.GetEntry(name);
    if (!entry_value) {
        return nullptr;
    }
    auto &catalog_entry = *entry_value;

    // 检测写写冲突
    if (HasConflict(transaction, catalog_entry.timestamp)) {
        throw TransactionException("Catalog write-write conflict on alter with \"%s\"",
                                   catalog_entry.name);
    }

    if (catalog_entry.deleted) {
        return nullptr;
    }
    return &catalog_entry;
}
```

## 修改条目

### AlterEntry 流程

```cpp
bool CatalogSet::AlterEntry(CatalogTransaction transaction, const string &name,
                            AlterInfo &alter_info) {
    // 1. 先获取条目（不持锁）
    auto entry = GetEntry(transaction, name);
    if (!entry) {
        return false;
    }

    // 2. 创建新版本
    unique_ptr<CatalogEntry> value;
    if (alter_info.type == AlterType::SET_COMMENT) {
        value = entry->Copy(*transaction.context);
        value->comment = alter_info.Cast<SetCommentInfo>().comment_value;
    } else {
        value = entry->AlterEntry(transaction, alter_info);
    }

    // 3. 获取锁
    unique_lock<mutex> write_lock(catalog.GetWriteLock());
    unique_lock<mutex> read_lock(catalog_lock);

    // 4. 再次获取条目（检测写写冲突）
    entry = GetEntryInternal(transaction, name);

    // 5. 标记新版本
    value->timestamp = transaction.transaction_id;
    value->set = this;

    // 6. 更新版本链
    // ...
}
```

### 乐观锁模式

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AlterEntry 乐观锁模式                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1: 乐观读取（无锁）                                            │
│  ┌─────────────────────────────────────────┐                        │
│  │ auto entry = GetEntry(transaction, name)│                        │
│  │ 创建 altered entry (可能很耗时)          │                        │
│  └─────────────────────────────────────────┘                        │
│                                                                     │
│  阶段 2: 悲观验证（持锁）                                            │
│  ┌─────────────────────────────────────────┐                        │
│  │ write_lock + read_lock                  │                        │
│  │ entry = GetEntryInternal(...)           │                        │
│  │   └─► 如果 HasConflict → 抛异常          │                        │
│  │ 更新版本链                               │                        │
│  └─────────────────────────────────────────┘                        │
│                                                                     │
│  优势：减少锁持有时间                                                │
│  代价：可能需要重做修改操作                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 查询条目

### GetEntryDetailed 实现

```cpp
CatalogSet::EntryLookup CatalogSet::GetEntryDetailed(CatalogTransaction transaction,
                                                      const string &name) {
    unique_lock<mutex> lock(catalog_lock);
    CreateDefaultEntries(transaction, lock);

    auto entry_value = map.GetEntry(name);
    if (!entry_value) {
        // 尝试懒加载默认条目
        entry_value = CreateDefaultEntry(transaction, name, lock);
        if (!entry_value) {
            return EntryLookup {nullptr, EntryLookup::FailureReason::NOT_PRESENT};
        }
    }

    auto &catalog_entry = *entry_value;
    bool visible;
    auto &current = GetEntryForTransaction(transaction, catalog_entry, visible);

    if (current.deleted) {
        if (!visible) {
            return EntryLookup {nullptr, EntryLookup::FailureReason::INVISIBLE};
        } else {
            return EntryLookup {nullptr, EntryLookup::FailureReason::DELETED};
        }
    }

    return EntryLookup {&current, EntryLookup::FailureReason::SUCCESS};
}
```

### 查询结果类型

| FailureReason | 含义 | 用户可见消息 |
|---------------|------|-------------|
| SUCCESS | 成功找到 | - |
| NOT_PRESENT | 从未存在 | "Object not found" |
| DELETED | 已被删除 | "Object not found" |
| INVISIBLE | 被其他事务修改 | "Object not found" |

## 扫描操作

### Scan 实现

```cpp
void CatalogSet::Scan(CatalogTransaction transaction,
                      const std::function<void(CatalogEntry &)> &callback) {
    // 获取目录锁
    unique_lock<mutex> lock(catalog_lock);
    CreateDefaultEntries(transaction, lock);

    for (auto &kv : map.Entries()) {
        auto &entry = *kv.second;
        // 获取对当前事务可见的版本
        auto &entry_for_transaction = GetEntryForTransaction(transaction, entry);
        if (!entry_for_transaction.deleted) {
            callback(entry_for_transaction);
        }
    }
}
```

扫描时为每个条目找到正确的可见版本。

## Undo 回滚机制

### CleanupEntry

```cpp
void CatalogSet::CleanupEntry(CatalogEntry &catalog_entry) {
    // 从版本链中移除旧版本
    map.DropEntry(catalog_entry);
}
```

### CatalogEntryMap::DropEntry

```cpp
void CatalogEntryMap::DropEntry(CatalogEntry &entry) {
    auto &name = entry.name;
    auto chain = GetEntry(name);

    auto child = entry.TakeChild();
    if (!entry.HasParent()) {
        // 这是链顶
        auto it = entries.find(name);
        it->second.reset();
        if (child) {
            // 用子节点替换
            it->second = std::move(child);
        } else {
            entries.erase(it);
        }
    } else {
        // 中间节点：用子节点替换
        auto &parent = entry.Parent();
        parent.SetChild(std::move(child));
    }
}
```

### 版本链清理

```
┌─────────────────────────────────────────────────────────────────────┐
│                    版本链清理示例                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  清理前:                                                            │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                        │
│  │ V3      │ ──► │ V2 ◄──  │ ──► │ V1      │                        │
│  │ (最新)  │     │ (清理)  │     │ (旧)    │                        │
│  └─────────┘     └─────────┘     └─────────┘                        │
│                                                                     │
│  清理后:                                                            │
│  ┌─────────┐                     ┌─────────┐                        │
│  │ V3      │ ──────────────────► │ V1      │                        │
│  │ (最新)  │                     │ (旧)    │                        │
│  └─────────┘                     └─────────┘                        │
│                                                                     │
│  V2 被从链中移除                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 重命名操作

### RenameEntryInternal

重命名涉及两个位置的修改：

```cpp
bool CatalogSet::RenameEntryInternal(CatalogTransaction transaction,
                                      CatalogEntry &old,
                                      const string &new_name,
                                      AlterInfo &alter_info,
                                      unique_lock<mutex> &read_lock) {
    auto &original_name = old.name;

    // 1. 检查新名称是否已存在
    auto entry_value = map.GetEntry(new_name);
    if (entry_value) {
        auto &existing_entry = GetEntryForTransaction(transaction, *entry_value);
        if (!existing_entry.deleted) {
            throw CatalogException("Could not rename: name already exists!");
        }
    }

    // 2. 在旧位置添加 RENAMED_ENTRY 标记
    auto renamed_tombstone = make_uniq<InCatalogEntry>(
        CatalogType::RENAMED_ENTRY, old.ParentCatalog(), original_name);
    renamed_tombstone->timestamp = transaction.transaction_id;
    renamed_tombstone->deleted = false;
    CreateEntryInternal(transaction, original_name, std::move(renamed_tombstone), read_lock);

    // 3. 删除旧位置的条目
    DropEntryInternal(transaction, original_name, false);

    // 4. 在新位置创建条目
    auto renamed_node = make_uniq<InCatalogEntry>(
        CatalogType::RENAMED_ENTRY, catalog, new_name);
    renamed_node->timestamp = transaction.transaction_id;
    CreateEntryInternal(transaction, new_name, std::move(renamed_node), read_lock);

    return true;
}
```

## 并发场景示例

### 场景 1: 并发创建

```
事务 T1 (start=100)                事务 T2 (start=105)
     │                                   │
     │ CREATE TABLE foo                  │
     │ timestamp = T1                    │
     ├───────────────────────────────────┤
     │                                   │ CREATE TABLE foo
     │                                   │ → HasConflict(T1) = true
     │                                   │ → TransactionException!
     │                                   │
```

### 场景 2: 并发读写

```
事务 T1 (start=100)                事务 T2 (start=105)
     │                                   │
     │                                   │ ALTER TABLE foo
     │                                   │ timestamp = T2
     ├───────────────────────────────────┤
     │ SELECT * FROM foo                 │
     │ → GetEntryForTransaction          │
     │ → UseTimestamp(T2) = false        │
     │ → 返回旧版本 (T1 看不到 T2 的修改)   │
     │                                   │
```

### 场景 3: 提交后可见

```
事务 T1 (start=100)                事务 T2 (start=90)
     │                                   │
     │ CREATE TABLE foo                  │
     │ timestamp = T1                    │
     │ COMMIT (commit_id = 110)          │
     ├───────────────────────────────────┤
     │                                   │ SELECT * FROM foo
     │                                   │ → timestamp=110 > start=90
     │                                   │ → CommittedAfterStarting = true
     │                                   │ → 不可见
     │                                   │

事务 T3 (start=120)
     │
     │ SELECT * FROM foo
     │ → timestamp=110 < start=120
     │ → 可见 ✓
```

## 设计特点

### MVCC 优势

1. **读不阻塞写**：读操作只需遍历版本链
2. **写不阻塞读**：写操作创建新版本
3. **快照隔离**：每个事务看到一致的视图

### 冲突处理

1. **乐观检测**：修改时检查冲突
2. **快速失败**：发现冲突立即抛异常
3. **无等待**：不会阻塞等待其他事务

### 内存开销

版本链需要保留旧版本：
- 提交后等待清理
- 清理时机：`commit_id < lowest_active_start`

## 小结

CatalogSet 的并发访问设计体现了几个关键原则：

1. **版本链 MVCC**：通过版本链实现多版本并发控制
2. **时间戳可见性**：基于启动时间戳判断版本可见性
3. **写写冲突检测**：发现冲突立即失败，避免死锁
4. **双锁协议**：全局锁 + 局部锁确保修改原子性
5. **乐观读取**：减少锁持有时间，提高并发度

在下一章中，我们将探讨 Pipeline 并行执行，了解 DuckDB 如何实现高效的查询并行化。

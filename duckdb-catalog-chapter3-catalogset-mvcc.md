# DuckDB Catalog 系统深度解析 - 第三章：CatalogSet 与 MVCC

本章深入分析 DuckDB 中 CatalogSet 的实现，包括条目集合管理、MVCC 版本链机制、事务可见性判断以及并发控制。

## 3.1 CatalogSet 概述

### 3.1.1 核心职责

`CatalogSet` 是 DuckDB 中管理 CatalogEntry 集合的核心组件：

```cpp
// src/include/duckdb/catalog/catalog_set.hpp

class CatalogSet {
public:
    CatalogSet(Catalog &catalog, unique_ptr<DefaultGenerator> defaults = nullptr);

    //===--------------------------------------------------------------------===//
    // 条目操作
    //===--------------------------------------------------------------------===//
    bool CreateEntry(CatalogTransaction transaction, const string &name,
                     unique_ptr<CatalogEntry> value,
                     const LogicalDependencyList &dependencies);

    optional_ptr<CatalogEntry> GetEntry(CatalogTransaction transaction,
                                        const string &name);

    bool DropEntry(CatalogTransaction transaction, const string &name,
                   bool cascade, bool allow_drop_internal = false);

    bool AlterEntry(CatalogTransaction transaction, const string &name,
                    AlterInfo &alter_info);

    //===--------------------------------------------------------------------===//
    // 遍历
    //===--------------------------------------------------------------------===//
    void Scan(CatalogTransaction transaction,
              const std::function<void(CatalogEntry &)> &callback);

private:
    //! 所属的 DuckCatalog
    DuckCatalog &catalog;
    //! 读写锁
    mutex catalog_lock;
    //! 条目映射表
    CatalogEntryMap map;
    //! 默认条目生成器（懒加载内置对象）
    unique_ptr<DefaultGenerator> defaults;
};
```

### 3.1.2 CatalogEntryMap

条目存储使用大小写不敏感的树形映射：

```cpp
class CatalogEntryMap {
private:
    //! 大小写不敏感的映射
    case_insensitive_tree_t<unique_ptr<CatalogEntry>> entries;

public:
    //! 添加新条目
    void AddEntry(unique_ptr<CatalogEntry> entry);
    //! 更新条目（将新条目链接到旧条目）
    void UpdateEntry(unique_ptr<CatalogEntry> catalog_entry);
    //! 删除条目（从版本链中移除）
    void DropEntry(CatalogEntry &entry);
    //! 获取条目
    optional_ptr<CatalogEntry> GetEntry(const string &name);
};
```

---

## 3.2 MVCC 版本链

### 3.2.1 版本链结构

每个条目名称对应一个版本链，最新版本在链头：

```
CatalogEntryMap["my_table"]
         │
         ▼
    ┌─────────────────────┐
    │  CatalogEntry (V3)  │  ← 最新版本
    │  timestamp = T3     │     (活跃事务创建)
    │  deleted = false    │
    │  child ─────────────┼──┐
    │  parent = null      │  │
    └─────────────────────┘  │
                             │
                             ▼
    ┌─────────────────────┐
    │  CatalogEntry (V2)  │  ← 已提交版本
    │  timestamp = 100    │     (commit_id < TRANSACTION_ID_START)
    │  deleted = false    │
    │  child ─────────────┼──┐
    │  parent ────────────┼──┘ (指向 V3)
    └─────────────────────┘
                             │
                             ▼
    ┌─────────────────────┐
    │  CatalogEntry (V1)  │  ← 初始版本
    │  timestamp = 50     │
    │  deleted = false    │
    │  child = null       │
    │  parent ────────────┼── (指向 V2)
    └─────────────────────┘
```

### 3.2.2 版本链操作

**UpdateEntry - 添加新版本**：

```cpp
void CatalogEntryMap::UpdateEntry(unique_ptr<CatalogEntry> catalog_entry) {
    auto name = catalog_entry->name;

    auto entry = entries.find(name);
    if (entry == entries.end()) {
        throw InternalException("Entry does not exist");
    }

    // 取出现有条目
    auto existing = std::move(entry->second);
    // 用新条目替换
    entry->second = std::move(catalog_entry);
    // 将旧条目设为新条目的子节点
    entry->second->SetChild(std::move(existing));
}
```

**DropEntry - 从版本链移除**：

```cpp
void CatalogEntryMap::DropEntry(CatalogEntry &entry) {
    auto &name = entry.name;
    auto chain = GetEntry(name);

    auto child = entry.TakeChild();
    if (!entry.HasParent()) {
        // 这是链头
        auto it = entries.find(name);
        it->second.reset();
        if (child) {
            // 用子节点替换
            it->second = std::move(child);
        } else {
            // 完全删除
            entries.erase(it);
        }
    } else {
        // 中间节点，直接替换
        auto &parent = entry.Parent();
        parent.SetChild(std::move(child));
    }
}
```

---

## 3.3 事务时间戳与可见性

### 3.3.1 时间戳类型

DuckDB 使用 `transaction_t` 表示时间戳，区分两种状态：

```cpp
// 时间戳阈值（区分已提交和活跃事务）
static constexpr transaction_t TRANSACTION_ID_START = 4611686018427387904ULL;

// 时间戳含义:
// - timestamp < TRANSACTION_ID_START: 已提交事务的 commit_id
// - timestamp >= TRANSACTION_ID_START: 活跃事务的 transaction_id
```

### 3.3.2 可见性判断

```cpp
// 判断是否由其他活跃事务创建
bool CatalogSet::CreatedByOtherActiveTransaction(
    CatalogTransaction transaction, transaction_t timestamp) {
    // 条目由活跃事务创建，且不是当前事务
    return (timestamp >= TRANSACTION_ID_START &&
            timestamp != transaction.transaction_id);
}

// 判断是否在当前事务开始后提交
bool CatalogSet::CommittedAfterStarting(
    CatalogTransaction transaction, transaction_t timestamp) {
    // 条目已提交，但在我们开始后提交
    return (timestamp < TRANSACTION_ID_START &&
            timestamp > transaction.start_time);
}

// 判断是否有冲突
bool CatalogSet::HasConflict(
    CatalogTransaction transaction, transaction_t timestamp) {
    return CreatedByOtherActiveTransaction(transaction, timestamp) ||
           CommittedAfterStarting(transaction, timestamp);
}

// 判断是否应该使用该版本
bool CatalogSet::UseTimestamp(
    CatalogTransaction transaction, transaction_t timestamp) {
    if (timestamp == transaction.transaction_id) {
        // 我们创建的版本
        return true;
    }
    if (timestamp < transaction.start_time) {
        // 在我们开始前已提交的版本
        return true;
    }
    return false;
}
```

### 3.3.3 GetEntryForTransaction

遍历版本链找到对当前事务可见的版本：

```cpp
CatalogEntry &CatalogSet::GetEntryForTransaction(
    CatalogTransaction transaction, CatalogEntry &current, bool &visible) {

    reference<CatalogEntry> entry(current);

    // 从链头开始遍历
    while (entry.get().HasChild()) {
        if (UseTimestamp(transaction, entry.get().timestamp)) {
            visible = true;
            return entry.get();
        }
        entry = entry.get().Child();
    }

    // 到达链尾
    visible = false;
    return entry.get();
}
```

### 3.3.4 可见性场景图示

```
事务时间线:

                T1.start    T2.start    T1.commit    T2查询
                   │           │            │          │
    ───────────────┼───────────┼────────────┼──────────┼───────→ 时间
                   │           │            │          │
                   │           │            │          │
版本链状态:        │           │            │          │
                   │           │            │          │
  [V1]────────────►│           │            │          │
  ts=50            │           │            │          │
                   │           │            │          │
                   │ T1创建V2  │            │          │
                   │           │            │          │
              [V2]─┼──────────►│            │          │
              ts=T1│           │            │          │
                   │           │            │          │
                   │           │   T1提交   │          │
                   │           │   V2.ts=80 │          │
                   │           │            │          │
                   │           │       [V2]─┼─────────►│
                   │           │       ts=80│          │
                   │           │            │          │

T2 在查询时:
- T2.start_time = 60
- V2.timestamp = 80 > T2.start_time
- V2 对 T2 不可见
- T2 看到的是 V1
```

---

## 3.4 条目创建流程

### 3.4.1 CreateEntry

```cpp
bool CatalogSet::CreateEntry(
    CatalogTransaction transaction,
    const string &name,
    unique_ptr<CatalogEntry> value,
    const LogicalDependencyList &dependencies) {

    // 1. 检查条目不变量（internal/temporary 约束）
    CheckCatalogEntryInvariants(*value, name);

    // 2. 设置事务时间戳
    value->timestamp = transaction.transaction_id;
    value->set = this;

    // 3. 注册依赖关系
    catalog.GetDependencyManager()->AddObject(transaction, *value, dependencies);

    // 4. 获取写锁
    lock_guard<mutex> write_lock(catalog.GetWriteLock());
    // 5. 获取读锁
    unique_lock<mutex> read_lock(catalog_lock);

    // 6. 执行创建
    return CreateEntryInternal(transaction, name, std::move(value), read_lock);
}
```

### 3.4.2 CreateEntryInternal

```cpp
bool CatalogSet::CreateEntryInternal(
    CatalogTransaction transaction,
    const string &name,
    unique_ptr<CatalogEntry> value,
    unique_lock<mutex> &read_lock,
    bool should_be_empty) {

    auto entry_value = map.GetEntry(name);

    if (!entry_value) {
        // 不存在，创建初始链
        if (!StartChain(transaction, name, read_lock)) {
            return false;  // 默认条目已存在
        }
    } else if (should_be_empty) {
        // 应该为空（如 CREATE 时），验证是否已删除
        if (!VerifyVacancy(transaction, *entry_value)) {
            return false;  // 已存在且未删除
        }
    }

    // 添加新版本到链
    auto value_ptr = value.get();
    map.UpdateEntry(std::move(value));

    // 将旧条目推入 undo buffer
    if (transaction.transaction) {
        DuckTransactionManager::Get(GetCatalog().GetAttached())
            .PushCatalogEntry(*transaction.transaction, value_ptr->Child());
    }

    return true;
}
```

### 3.4.3 StartChain - 创建初始链

```cpp
bool CatalogSet::StartChain(
    CatalogTransaction transaction,
    const string &name,
    unique_lock<mutex> &read_lock) {

    D_ASSERT(!map.GetEntry(name));

    // 检查是否有默认条目
    auto entry = CreateDefaultEntry(transaction, name, read_lock);
    if (entry) {
        return false;  // 默认条目已创建
    }

    // 创建一个删除状态的哑节点作为链的起点
    // 这样其他事务在提交前看到的是"不存在"
    auto dummy_node = make_uniq<InCatalogEntry>(
        CatalogType::INVALID, catalog, name);
    dummy_node->timestamp = 0;
    dummy_node->deleted = true;
    dummy_node->set = this;

    map.AddEntry(std::move(dummy_node));
    return true;
}
```

### 3.4.4 创建流程图

```
CreateEntry 流程:

     输入: name="my_table", value=DuckTableEntry
                    │
                    ▼
    ┌───────────────────────────────────┐
    │ 1. CheckCatalogEntryInvariants    │
    │    检查 internal/temporary 约束   │
    └────────────────┬──────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 2. 设置 timestamp = transaction_id│
    └────────────────┬──────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 3. DependencyManager.AddObject    │
    │    注册依赖关系                    │
    └────────────────┬──────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 4. 获取 write_lock + catalog_lock │
    └────────────────┬──────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 5. map.GetEntry(name)             │
    │    检查是否已存在                  │
    └────────────────┬──────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
   不存在                       存在
        │                         │
        ▼                         ▼
  StartChain()              VerifyVacancy()
  创建哑节点                 检查是否已删除
        │                         │
        └────────────┬────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 6. map.UpdateEntry(value)         │
    │    将新条目加入版本链              │
    └────────────────┬──────────────────┘
                     │
                     ▼
    ┌───────────────────────────────────┐
    │ 7. PushCatalogEntry               │
    │    将旧条目推入 undo buffer        │
    └───────────────────────────────────┘
```

---

## 3.5 条目删除流程

### 3.5.1 DropEntry

```cpp
bool CatalogSet::DropEntry(
    CatalogTransaction transaction,
    const string &name,
    bool cascade,
    bool allow_drop_internal) {

    // 1. 处理依赖关系
    if (!DropDependencies(transaction, name, cascade, allow_drop_internal)) {
        return false;
    }

    // 2. 获取锁
    lock_guard<mutex> write_lock(catalog.GetWriteLock());
    lock_guard<mutex> read_lock(catalog_lock);

    // 3. 执行删除
    return DropEntryInternal(transaction, name, allow_drop_internal);
}
```

### 3.5.2 DropEntryInternal

```cpp
bool CatalogSet::DropEntryInternal(
    CatalogTransaction transaction,
    const string &name,
    bool allow_drop_internal) {

    // 获取条目（检查写写冲突）
    auto entry = GetEntryInternal(transaction, name);
    if (!entry) {
        return false;
    }

    if (entry->internal && !allow_drop_internal) {
        throw CatalogException("Cannot drop internal system entry");
    }

    // 创建墓碑条目
    auto value = make_uniq<InCatalogEntry>(
        CatalogType::DELETED_ENTRY,
        entry->ParentCatalog(),
        entry->name);
    value->timestamp = transaction.transaction_id;
    value->set = this;
    value->deleted = true;

    auto value_ptr = value.get();
    map.UpdateEntry(std::move(value));

    // 推入 undo buffer
    if (transaction.transaction) {
        DuckTransactionManager::Get(GetCatalog().GetAttached())
            .PushCatalogEntry(*transaction.transaction, value_ptr->Child());
    }

    return true;
}
```

### 3.5.3 删除后的版本链

```
删除 my_table 后:

    ┌─────────────────────┐
    │  DELETED_ENTRY      │  ← 墓碑（链头）
    │  timestamp = T_del  │
    │  deleted = true     │
    │  child ─────────────┼──┐
    └─────────────────────┘  │
                             │
                             ▼
    ┌─────────────────────┐
    │  DuckTableEntry     │  ← 被删除的表
    │  timestamp = 100    │
    │  deleted = false    │
    │  child ─────────────┼──► ...
    └─────────────────────┘

查询时:
- 事务 start_time < T_del: 看到 DuckTableEntry
- 事务 start_time >= T_del: 看到已删除状态
```

---

## 3.6 条目修改流程

### 3.6.1 AlterEntry

```cpp
bool CatalogSet::AlterEntry(
    CatalogTransaction transaction,
    const string &name,
    AlterInfo &alter_info) {

    // 1. 获取现有条目
    auto entry = GetEntry(transaction, name);
    if (!entry) {
        return false;
    }

    // 2. 创建修改后的条目
    unique_ptr<CatalogEntry> value;
    if (alter_info.type == AlterType::SET_COMMENT) {
        // 元数据修改，复制条目
        value = entry->Copy(*transaction.context);
        value->comment = alter_info.Cast<SetCommentInfo>().comment_value;
    } else {
        // 结构修改，调用 AlterEntry
        value = entry->AlterEntry(transaction, alter_info);
        if (!value) {
            return true;  // 修改成功但不需要新版本
        }
    }

    // 3. 获取锁
    unique_lock<mutex> write_lock(catalog.GetWriteLock());
    unique_lock<mutex> read_lock(catalog_lock);

    // 4. 重新获取条目（检查写写冲突）
    entry = GetEntryInternal(transaction, name);

    // 5. 设置时间戳
    value->timestamp = transaction.transaction_id;
    value->set = this;

    // 6. 处理重命名
    if (!StringUtil::CIEquals(value->name, entry->name)) {
        if (!RenameEntryInternal(transaction, *entry, value->name,
                                  alter_info, read_lock)) {
            return false;
        }
    }

    // 7. 更新版本链
    auto new_entry = value.get();
    map.UpdateEntry(std::move(value));

    // 8. 推入 undo buffer（包含 AlterInfo 用于回滚）
    if (transaction.transaction) {
        MemoryStream stream(Allocator::Get(*transaction.db));
        BinarySerializer serializer(stream);
        serializer.Begin();
        serializer.WriteProperty(100, "column_name", alter_info.GetColumnName());
        serializer.WriteProperty(101, "alter_info", &alter_info);
        serializer.End();

        DuckTransactionManager::Get(GetCatalog().GetAttached())
            .PushCatalogEntry(*transaction.transaction, new_entry->Child(),
                              stream.GetData(), stream.GetPosition());
    }

    read_lock.unlock();
    write_lock.unlock();

    // 9. 更新依赖管理器
    catalog.GetDependencyManager()->AlterObject(
        transaction, *entry, *new_entry, alter_info);

    return true;
}
```

---

## 3.7 并发控制

### 3.7.1 锁层次

```
┌─────────────────────────────────────────────────────────────────┐
│                        锁层次结构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DuckCatalog                                                     │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  write_lock (mutex)                                         ││
│  │  • 保护所有写操作                                            ││
│  │  • 确保一次只有一个事务修改 Catalog                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  CatalogSet                                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  catalog_lock (mutex)                                       ││
│  │  • 保护 map 结构                                             ││
│  │  • 读写操作都需要获取                                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.7.2 写写冲突检测

```cpp
optional_ptr<CatalogEntry> CatalogSet::GetEntryInternal(
    CatalogTransaction transaction, const string &name) {

    auto entry_value = map.GetEntry(name);
    if (!entry_value) {
        return nullptr;
    }

    auto &catalog_entry = *entry_value;

    // 检查写写冲突
    if (HasConflict(transaction, catalog_entry.timestamp)) {
        // 另一个事务已修改此条目
        // 由于 Catalog 限制，即使那个事务可能回滚，
        // 我们也无法在其基础上创建新版本
        throw TransactionException(
            "Catalog write-write conflict on alter with \"%s\"",
            catalog_entry.name);
    }

    if (catalog_entry.deleted) {
        return nullptr;
    }

    return &catalog_entry;
}
```

### 3.7.3 冲突场景

```
场景: T1 和 T2 同时修改表 "foo"

时间线:
                T1.start   T2.start   T1.ALTER   T2.ALTER
                   │          │          │          │
    ───────────────┼──────────┼──────────┼──────────┼──────→
                   │          │          │          │
                   │          │          │          │
                   │          │          │          │
                   │          │    T1 获得锁      │
                   │          │    创建新版本     │
                   │          │    ts = T1.id    │
                   │          │          │        │
                   │          │          │   T2 尝试获取锁
                   │          │          │   HasConflict(T2, T1.id) = true
                   │          │          │   抛出 TransactionException
                   │          │          │          │
                   │          │          │          ▼
                   │          │          │     T2 事务回滚

结论: 先获得锁的事务成功，后来的事务必须回滚
```

---

## 3.8 事务回滚与清理

### 3.8.1 Undo 操作

```cpp
void CatalogSet::Undo(CatalogEntry &entry) {
    lock_guard<mutex> write_lock(catalog.GetWriteLock());
    lock_guard<mutex> lock(catalog_lock);

    // entry 需要恢复
    // entry.parent 需要移除
    auto &to_be_removed_node = entry.Parent();
    to_be_removed_node.Rollback(entry);

    // 从版本链中移除父节点
    if (!to_be_removed_node.HasParent()) {
        to_be_removed_node.Child().SetAsRoot();
    }
    map.DropEntry(to_be_removed_node);

    // 如果 entry 是哑节点，也移除
    if (entry.type == CatalogType::INVALID) {
        map.DropEntry(entry);
    }
}
```

### 3.8.2 回滚示意图

```
回滚前:

    ┌─────────────────┐
    │  新版本 (V2)    │ ← 需要移除
    │  ts = T1.id     │
    │  child ─────────┼──┐
    └─────────────────┘  │
                         ▼
    ┌─────────────────┐
    │  旧版本 (V1)    │ ← 需要恢复
    │  ts = 50        │
    └─────────────────┘

回滚后:

    ┌─────────────────┐
    │  旧版本 (V1)    │ ← 恢复为链头
    │  ts = 50        │
    └─────────────────┘
```

### 3.8.3 清理操作

事务提交后清理不再需要的版本：

```cpp
void CatalogSet::CleanupEntry(CatalogEntry &catalog_entry) {
    lock_guard<mutex> write_lock(catalog.GetWriteLock());
    lock_guard<mutex> lock(catalog_lock);

    auto &parent = catalog_entry.Parent();

    // 从版本链中移除
    map.DropEntry(catalog_entry);

    // 如果父节点是墓碑且无子节点，也清理
    if (parent.deleted && !parent.HasChild() && !parent.HasParent()) {
        D_ASSERT(map.GetEntry(parent.name).get() == &parent);
        map.DropEntry(parent);
    }
}
```

---

## 3.9 条目查找

### 3.9.1 GetEntry

```cpp
optional_ptr<CatalogEntry> CatalogSet::GetEntry(
    CatalogTransaction transaction, const string &name) {

    auto lookup = GetEntryDetailed(transaction, name);
    return lookup.result;
}

CatalogSet::EntryLookup CatalogSet::GetEntryDetailed(
    CatalogTransaction transaction, const string &name) {

    unique_lock<mutex> read_lock(catalog_lock);
    auto entry_value = map.GetEntry(name);

    if (entry_value) {
        // 找到条目，检查版本
        auto &catalog_entry = *entry_value;
        bool visible;
        auto &current = GetEntryForTransaction(transaction, catalog_entry, visible);

        if (current.deleted) {
            if (!visible) {
                return EntryLookup{nullptr, FailureReason::INVISIBLE};
            } else {
                return EntryLookup{nullptr, FailureReason::DELETED};
            }
        }
        return EntryLookup{&current, FailureReason::SUCCESS};
    }

    // 尝试创建默认条目
    auto default_entry = CreateDefaultEntry(transaction, name, read_lock);
    if (!default_entry) {
        return EntryLookup{default_entry, FailureReason::NOT_PRESENT};
    }
    return EntryLookup{default_entry, FailureReason::SUCCESS};
}
```

### 3.9.2 查找结果类型

```cpp
struct EntryLookup {
    optional_ptr<CatalogEntry> result;

    enum class FailureReason {
        SUCCESS,        // 找到
        NOT_PRESENT,    // 不存在
        DELETED,        // 已删除（对当前事务可见）
        INVISIBLE       // 被其他事务删除（对当前事务不可见）
    };

    FailureReason reason;
};
```

---

## 3.10 遍历操作

### 3.10.1 Scan

```cpp
void CatalogSet::Scan(
    CatalogTransaction transaction,
    const std::function<void(CatalogEntry &)> &callback) {

    unique_lock<mutex> lock(catalog_lock);

    // 确保所有默认条目已创建
    CreateDefaultEntries(transaction, lock);

    // 遍历所有条目
    for (auto &kv : map.Entries()) {
        auto &entry = *kv.second;
        auto &entry_for_transaction = GetEntryForTransaction(transaction, entry);
        if (!entry_for_transaction.deleted) {
            callback(entry_for_transaction);
        }
    }
}
```

### 3.10.2 ScanWithPrefix

支持前缀扫描（用于自动补全等）：

```cpp
void CatalogSet::ScanWithPrefix(
    CatalogTransaction transaction,
    const std::function<void(CatalogEntry &)> &callback,
    const string &prefix) {

    unique_lock<mutex> lock(catalog_lock);
    CreateDefaultEntries(transaction, lock);

    auto &entries = map.Entries();
    // 利用有序映射进行范围扫描
    auto it = entries.lower_bound(prefix);
    auto end = entries.upper_bound(prefix + char(255));

    for (; it != end; it++) {
        auto &entry = *it->second;
        auto &entry_for_transaction = GetEntryForTransaction(transaction, entry);
        if (!entry_for_transaction.deleted) {
            callback(entry_for_transaction);
        }
    }
}
```

---

## 3.11 本章小结

本章详细分析了 CatalogSet 的 MVCC 实现：

1. **版本链结构**：
   - 每个名称对应一个版本链
   - 最新版本在链头，通过 child 指针连接旧版本
   - 删除使用墓碑节点（DELETED_ENTRY）

2. **事务时间戳**：
   - `timestamp < TRANSACTION_ID_START`：已提交事务的 commit_id
   - `timestamp >= TRANSACTION_ID_START`：活跃事务的 transaction_id
   - `UseTimestamp()` 判断版本对当前事务是否可见

3. **并发控制**：
   - 两级锁：`write_lock`（Catalog 级）+ `catalog_lock`（CatalogSet 级）
   - 写写冲突检测：`HasConflict()` 检查时间戳
   - 冲突时抛出 `TransactionException`

4. **操作流程**：
   - 创建：添加新版本到链头，推入 undo buffer
   - 删除：创建墓碑节点，推入 undo buffer
   - 修改：创建新版本，序列化 AlterInfo 用于回滚

5. **回滚与清理**：
   - `Undo()`：从版本链移除新版本，恢复旧版本
   - `CleanupEntry()`：事务提交后清理不再需要的版本

---

## 3.12 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog_set.hpp` | CatalogSet 定义 |
| `src/catalog/catalog_set.cpp` | CatalogSet 实现 |
| `src/include/duckdb/catalog/catalog_transaction.hpp` | CatalogTransaction 定义 |
| `src/catalog/catalog_transaction.cpp` | CatalogTransaction 实现 |
| `src/transaction/duck_transaction_manager.cpp` | PushCatalogEntry 实现 |

# 第二章：StorageLock 读写锁

## 概述

StorageLock 是 DuckDB 存储层的核心同步原语，实现了经典的读写锁（Reader-Writer Lock）语义。它支持多个读者并发访问，但写者需要独占访问权。与标准库的 `std::shared_mutex` 不同，StorageLock 针对 DuckDB 的检查点机制提供了特殊的锁升级支持。

本章深入分析 StorageLock 的设计原理、内部实现、RAII 封装以及检查点锁的特殊处理。

## StorageLock 设计原理

### 读写锁语义

StorageLock 提供两种锁类型：

```cpp
// src/include/duckdb/storage/storage_lock.hpp
enum class StorageLockType { SHARED = 0, EXCLUSIVE = 1 };
```

**共享锁（SHARED）**：
- 允许多个持有者同时存在
- 适用于只读操作
- 与排他锁互斥

**排他锁（EXCLUSIVE）**：
- 只允许一个持有者
- 适用于修改操作
- 与共享锁和其他排他锁都互斥

### 核心接口

```cpp
class StorageLock {
public:
    //! 获取排他锁（阻塞等待）
    unique_ptr<StorageLockKey> GetExclusiveLock();

    //! 获取共享锁（阻塞等待）
    unique_ptr<StorageLockKey> GetSharedLock();

    //! 尝试获取排他锁（非阻塞）
    unique_ptr<StorageLockKey> TryGetExclusiveLock();

    //! 检查点专用：尝试将共享锁升级为排他锁
    unique_ptr<StorageLockKey> TryUpgradeCheckpointLock(StorageLockKey &lock);
};
```

返回 `unique_ptr<StorageLockKey>` 实现了 RAII 模式，锁在析构时自动释放。

## StorageLockInternals 内部实现

### 数据结构

StorageLock 的内部状态由 `StorageLockInternals` 结构体管理：

```cpp
// src/storage/storage_lock.cpp
struct StorageLockInternals : enable_shared_from_this<StorageLockInternals> {
public:
    StorageLockInternals() : read_count(0) {
    }

    mutex exclusive_lock;        // 互斥锁，保护排他访问
    atomic<idx_t> read_count;    // 原子计数器，记录活跃读者数量
};
```

这是一种经典的读写锁实现策略：
- **互斥锁**：控制写者的独占访问
- **原子计数器**：跟踪并发读者数量

### 获取排他锁

```cpp
unique_ptr<StorageLockKey> GetExclusiveLock() {
    exclusive_lock.lock();           // 1. 获取互斥锁
    while (read_count != 0) {        // 2. 自旋等待所有读者退出
    }
    return make_uniq<StorageLockKey>(shared_from_this(), StorageLockType::EXCLUSIVE);
}
```

获取排他锁的流程：

```
┌─────────────────────────────────────────────────┐
│              获取排他锁流程                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────┐                       │
│  │ exclusive_lock.lock()│◄── 阻塞等待互斥锁      │
│  └──────────┬───────────┘                       │
│             │                                   │
│             ▼                                   │
│  ┌──────────────────────┐                       │
│  │  read_count == 0 ?   │                       │
│  └──────────┬───────────┘                       │
│             │                                   │
│     ┌───────┴───────┐                           │
│     │ No            │ Yes                       │
│     ▼               ▼                           │
│  ┌──────┐    ┌────────────────┐                 │
│  │ 自旋 │    │ 返回排他锁句柄  │                 │
│  │ 等待 │    └────────────────┘                 │
│  └──┬───┘                                       │
│     │                                           │
│     └──────► 继续检查 read_count                │
│                                                 │
└─────────────────────────────────────────────────┘
```

**自旋等待设计**：当持有互斥锁但仍有读者时，采用空循环自旋等待。这种设计假设读者很快会释放锁，避免条件变量的上下文切换开销。

### 获取共享锁

```cpp
unique_ptr<StorageLockKey> GetSharedLock() {
    exclusive_lock.lock();           // 1. 获取互斥锁
    read_count++;                    // 2. 增加读者计数
    exclusive_lock.unlock();         // 3. 立即释放互斥锁
    return make_uniq<StorageLockKey>(shared_from_this(), StorageLockType::SHARED);
}
```

共享锁的关键设计：
- **短暂持有互斥锁**：仅在修改 `read_count` 时持有
- **写者优先**：如果写者正在等待互斥锁，新读者必须排队

```
┌─────────────────────────────────────────────────┐
│              获取共享锁流程                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────┐                       │
│  │ exclusive_lock.lock()│◄── 确保与写者互斥      │
│  └──────────┬───────────┘                       │
│             │                                   │
│             ▼                                   │
│  ┌──────────────────────┐                       │
│  │    read_count++      │◄── 注册为活跃读者      │
│  └──────────┬───────────┘                       │
│             │                                   │
│             ▼                                   │
│  ┌────────────────────────┐                     │
│  │ exclusive_lock.unlock()│◄── 允许其他读者进入  │
│  └──────────┬─────────────┘                     │
│             │                                   │
│             ▼                                   │
│  ┌────────────────────┐                         │
│  │ 返回共享锁句柄      │                         │
│  └────────────────────┘                         │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 非阻塞尝试获取

```cpp
unique_ptr<StorageLockKey> TryGetExclusiveLock() {
    if (!exclusive_lock.try_lock()) {
        // 无法获取互斥锁
        return nullptr;
    }
    if (read_count != 0) {
        // 有活跃读者 - 无法获取排他锁
        exclusive_lock.unlock();
        return nullptr;
    }
    // 成功！
    return make_uniq<StorageLockKey>(shared_from_this(), StorageLockType::EXCLUSIVE);
}
```

`TryGetExclusiveLock` 的特点：
- **非阻塞**：无论成功与否都立即返回
- **原子语义**：要么完全获取，要么完全不获取
- **适用场景**：需要避免死锁或实现超时逻辑

## StorageLockKey RAII 封装

### 锁句柄设计

```cpp
// src/include/duckdb/storage/storage_lock.hpp
class StorageLockKey {
public:
    StorageLockKey(shared_ptr<StorageLockInternals> internals, StorageLockType type);
    ~StorageLockKey();

    StorageLockType GetType() const {
        return type;
    }

private:
    shared_ptr<StorageLockInternals> internals;
    StorageLockType type;
};
```

`StorageLockKey` 是锁的持有证明，其生命周期与锁的持有时间绑定。

### 自动释放机制

```cpp
// src/storage/storage_lock.cpp
StorageLockKey::~StorageLockKey() {
    if (type == StorageLockType::EXCLUSIVE) {
        internals->ReleaseExclusiveLock();
    } else {
        D_ASSERT(type == StorageLockType::SHARED);
        internals->ReleaseSharedLock();
    }
}
```

释放操作的实现：

```cpp
void ReleaseExclusiveLock() {
    exclusive_lock.unlock();     // 释放互斥锁
}

void ReleaseSharedLock() {
    read_count--;                // 原子递减读者计数
}
```

### shared_ptr 生命周期管理

`StorageLockKey` 持有 `shared_ptr<StorageLockInternals>`，这确保：
- **安全释放**：即使 `StorageLock` 对象已销毁，仍可安全释放锁
- **延迟销毁**：`StorageLockInternals` 直到所有 `StorageLockKey` 都析构后才销毁

```
┌─────────────────────────────────────────────────────────────┐
│                   生命周期关系                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐      owns       ┌─────────────────────┐   │
│  │ StorageLock  │ ──────────────► │ StorageLockInternals│   │
│  └──────────────┘  shared_ptr     └─────────────────────┘   │
│                                              ▲              │
│                                              │              │
│  ┌───────────────┐     shared_ptr            │              │
│  │ StorageLockKey│ ──────────────────────────┘              │
│  └───────────────┘                                          │
│         │                                                   │
│         │ ~StorageLockKey()                                 │
│         ▼                                                   │
│  ┌─────────────────────────────────┐                        │
│  │ ReleaseExclusiveLock() 或       │                        │
│  │ ReleaseSharedLock()             │                        │
│  └─────────────────────────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 检查点锁特殊处理

### TryUpgradeCheckpointLock 机制

检查点（Checkpoint）需要独占访问数据结构，但在某些情况下，事务已经持有共享锁。DuckDB 提供了特殊的锁升级机制：

```cpp
unique_ptr<StorageLockKey> TryUpgradeCheckpointLock(StorageLockKey &lock) {
    if (lock.GetType() != StorageLockType::SHARED) {
        throw InternalException("StorageLock::TryUpgradeLock called on an exclusive lock");
    }
    if (!exclusive_lock.try_lock()) {
        // 无法获取互斥锁
        return nullptr;
    }
    if (read_count != 1) {
        // 其他共享锁活跃：升级失败
        D_ASSERT(read_count != 0);
        exclusive_lock.unlock();
        return nullptr;
    }
    // 没有其他共享锁活跃：成功！
    return make_uniq<StorageLockKey>(shared_from_this(), StorageLockType::EXCLUSIVE);
}
```

### 升级条件

锁升级成功的条件：
1. 调用者持有共享锁
2. 能够获取互斥锁（无其他写者等待）
3. **`read_count == 1`**：只有调用者一个读者

```
┌─────────────────────────────────────────────────────────────┐
│              检查点锁升级流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  前提：事务 T 已持有共享锁（read_count >= 1）                 │
│                                                             │
│  ┌─────────────────────────┐                                │
│  │ 验证 lock.type == SHARED│                                │
│  └──────────┬──────────────┘                                │
│             │                                               │
│             ▼                                               │
│  ┌─────────────────────────┐                                │
│  │ try_lock(exclusive_lock)│                                │
│  └──────────┬──────────────┘                                │
│             │                                               │
│     ┌───────┴───────┐                                       │
│     │ 失败          │ 成功                                   │
│     ▼               ▼                                       │
│  返回 nullptr  ┌─────────────────┐                          │
│               │ read_count == 1?│                           │
│               └────────┬────────┘                           │
│                        │                                    │
│            ┌───────────┴───────────┐                        │
│            │ No                    │ Yes                    │
│            ▼                       ▼                        │
│    ┌─────────────────┐    ┌────────────────────────┐        │
│    │ unlock + nullptr│    │ 返回排他锁（保留共享锁）│        │
│    └─────────────────┘    └────────────────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 双锁状态

升级成功后，调用者同时持有：
- 原有的**共享锁**（`read_count` 仍为 1）
- 新获取的**排他锁**（持有 `exclusive_lock`）

这种双锁状态在常规场景下是不允许的，但对检查点至关重要：

```cpp
// src/transaction/duck_transaction_manager.cpp
unique_ptr<StorageLockKey> DuckTransactionManager::TryUpgradeCheckpointLock(StorageLockKey &lock) {
    return checkpoint_lock.TryUpgradeCheckpointLock(lock);
}
```

### 检查点锁的使用场景

在事务管理器中，检查点锁的获取方式取决于是否已持有写锁：

```cpp
// src/transaction/duck_transaction.cpp
unique_ptr<StorageLockKey> DuckTransaction::TryGetCheckpointLock() {
    if (!write_lock) {
        // 没有写锁，直接尝试获取排他锁
        return GetTransactionManager().TryGetCheckpointLock();
    } else {
        // 已有写锁，尝试升级
        return GetTransactionManager().TryUpgradeCheckpointLock(*write_lock);
    }
}
```

## 实际应用场景

### UpdateSegment 更新段

`UpdateSegment` 使用 StorageLock 保护更新数据：

```cpp
// src/storage/table/update_segment.cpp

// 读取操作使用共享锁
void UpdateSegment::FetchUpdates(TransactionData transaction,
                                  idx_t vector_index, Vector &result) {
    auto lock_handle = lock.GetSharedLock();
    auto node = GetUpdateNode(*lock_handle, vector_index);
    // ... 读取更新数据
}

// 写入操作使用排他锁
void UpdateSegment::RollbackUpdate(UpdateInfo &info) {
    auto lock_handle = lock.GetExclusiveLock();
    // ... 回滚更新
}

// 修改操作使用排他锁
void UpdateSegment::Update(...) {
    auto write_lock = lock.GetExclusiveLock();
    // ... 执行更新
}
```

### ExternalFileCache 外部文件缓存

外部文件缓存使用锁句柄作为访问凭证：

```cpp
// src/storage/external_file_cache.cpp

// 锁句柄作为方法参数，确保调用者持有锁
idx_t &CachedFile::FileSize(const unique_ptr<StorageLockKey> &guard) {
    return file_size;
}

bool &CachedFile::CanSeek(const unique_ptr<StorageLockKey> &guard) {
    return can_seek;
}

// 遍历缓存文件时获取共享锁
for (const auto &file : cached_files) {
    auto ranges_guard = file.second->lock.GetSharedLock();
    for (const auto &range_entry : file.second->Ranges(ranges_guard)) {
        // ... 安全访问缓存范围
    }
}
```

### DataTable 数据表

数据表级别的检查点锁：

```cpp
// src/storage/data_table.cpp
unique_ptr<StorageLockKey> DataTable::GetCheckpointLock() {
    return info->checkpoint_lock.GetExclusiveLock();
}
```

## 读写锁状态转换

下图展示了 StorageLock 的完整状态机：

```
┌─────────────────────────────────────────────────────────────────┐
│                    StorageLock 状态转换                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────┐                              │
│     GetSharedLock  │             │  GetSharedLock               │
│    ┌───────────────│    空闲     │───────────────┐              │
│    │               │  (R=0,W=0)  │               │              │
│    │               └──────┬──────┘               │              │
│    │                      │                      │              │
│    │          GetExclusiveLock                   │              │
│    │                      │                      │              │
│    ▼                      ▼                      ▼              │
│ ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│ │   单读者     │    │    写者      │    │   多读者     │        │
│ │  (R=1,W=0)   │    │  (R=0,W=1)   │    │  (R>1,W=0)   │        │
│ └──────┬───────┘    └──────┬───────┘    └──────┬───────┘        │
│        │                   │                   │                │
│        │ TryUpgrade        │ Release           │ ReleaseShared  │
│        │ CheckpointLock    │ ExclusiveLock     │                │
│        │                   │                   │                │
│        ▼                   ▼                   ▼                │
│ ┌──────────────┐    ┌─────────────┐     继续减少直到 R=0        │
│ │ 双锁（检查点）│    │    空闲     │                             │
│ │  (R=1,W=1)   │    │  (R=0,W=0)  │                             │
│ └──────────────┘    └─────────────┘                             │
│                                                                 │
│  R = read_count,  W = exclusive_lock 持有状态                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 设计特点与权衡

### 写者优先

StorageLock 实现了**写者优先**策略：
- 写者获取 `exclusive_lock` 后，新的读者必须等待
- 避免写者饥饿问题

### 自旋等待

排他锁获取时使用空循环自旋：
```cpp
while (read_count != 0) {
}
```

**优点**：
- 避免条件变量的系统调用开销
- 对短期锁持有非常高效

**缺点**：
- 长时间等待会消耗 CPU
- 不适合高并发读写竞争场景

DuckDB 的 SWMR 模型（单写多读）下，这种设计是合理的。

### RAII 与异常安全

使用 `unique_ptr<StorageLockKey>` 返回锁句柄：
- 自动管理锁的生命周期
- 异常抛出时自动释放锁
- 避免锁泄漏

### 非阻塞 API

`TryGetExclusiveLock` 提供非阻塞语义：
- 用于超时实现
- 用于死锁避免
- 用于乐观并发控制

## 与标准库对比

| 特性 | StorageLock | std::shared_mutex |
|-----|-------------|-------------------|
| 锁类型 | SHARED/EXCLUSIVE | shared/unique |
| RAII 封装 | StorageLockKey | lock_guard/unique_lock |
| 锁升级 | TryUpgradeCheckpointLock | 不支持 |
| 等待策略 | 自旋等待 | 阻塞等待 |
| 非阻塞尝试 | TryGetExclusiveLock | try_lock |

## 小结

StorageLock 是 DuckDB 存储层的核心同步原语，其设计反映了几个关键决策：

1. **读写分离**：区分共享锁和排他锁，支持多读者并发
2. **写者优先**：通过互斥锁保证写者不被饿死
3. **自旋优化**：针对短期锁持有的场景优化
4. **检查点支持**：特殊的锁升级机制满足检查点需求
5. **RAII 安全**：通过智能指针确保锁的正确释放

在下一章中，我们将探讨 TaskScheduler 任务调度器，了解 DuckDB 如何管理并行任务的执行。

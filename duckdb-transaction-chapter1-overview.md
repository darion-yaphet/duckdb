# 第一章：事务系统架构概述

DuckDB 作为一款支持 ACID 特性的分析型数据库，其事务系统采用了经典的 MVCC（多版本并发控制）架构。本章将从整体视角介绍 DuckDB 事务系统的核心组件、层次结构和设计理念。

## 1.1 事务系统架构总览

### 1.1.1 分层架构

DuckDB 事务系统采用清晰的分层设计：

```
┌─────────────────────────────────────────────────────────────┐
│                   客户端层 (Client Layer)                    │
│                      TransactionContext                      │
│              (管理客户端会话的事务状态)                        │
├─────────────────────────────────────────────────────────────┤
│                   协调层 (Coordination Layer)                │
│                       MetaTransaction                        │
│              (跨多个 Attached 数据库协调事务)                  │
├─────────────────────────────────────────────────────────────┤
│                   管理层 (Management Layer)                  │
│     TransactionManager ← DuckTransactionManager              │
│              (事务生命周期管理、时间戳分配)                    │
├─────────────────────────────────────────────────────────────┤
│                   执行层 (Execution Layer)                   │
│           Transaction ← DuckTransaction                      │
│              (MVCC 快照隔离、Undo Buffer)                    │
├─────────────────────────────────────────────────────────────┤
│                   存储层 (Storage Layer)                     │
│         LocalStorage / UndoBuffer / VersionInfo              │
│              (事务私有数据、版本链)                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.1.2 核心组件关系

```
ClientContext (客户端会话)
    │
    └── TransactionContext (事务上下文)
            │
            └── MetaTransaction (元事务 - 跨库协调)
                    │
                    ├── Transaction @ Database1
                    │       └── DuckTransaction (MVCC实现)
                    │               ├── UndoBuffer (回滚日志)
                    │               ├── LocalStorage (本地存储)
                    │               └── sequence_usage (序列使用)
                    │
                    └── Transaction @ Database2
                            └── ... (可能是其他类型的事务)

AttachedDatabase
    │
    └── TransactionManager
            └── DuckTransactionManager
                    ├── active_transactions (活跃事务)
                    ├── recently_committed_transactions (近期提交)
                    ├── cleanup_queue (清理队列)
                    └── timestamp counters (时间戳计数器)
```

## 1.2 Transaction 基类

### 1.2.1 接口定义

```cpp
// src/include/duckdb/transaction/transaction.hpp
class Transaction {
protected:
    TransactionManager &transaction_manager;
    weak_ptr<ClientContext> context;      // 弱引用避免循环
    bool is_read_only;                     // 只读优化
    atomic<transaction_t> active_query;    // 当前活跃查询

public:
    // 只读 → 读写 晋升
    void SetReadWrite();

    // 获取事务所属的数据库
    AttachedDatabase &GetAttached();

    // 类型检查
    virtual bool IsDuckTransaction() const;

    // 静态工厂方法
    static Transaction &Get(ClientContext &context, AttachedDatabase &db);
    static optional_ptr<Transaction> TryGet(ClientContext &context,
                                            AttachedDatabase &db);
};
```

### 1.2.2 只读优化模式

```
事务启动时默认为只读:
┌─────────────────────────────────────────────────────────────┐
│ BEGIN TRANSACTION                                            │
│     ↓                                                        │
│ is_read_only = true  ← 默认只读，可跳过某些锁获取            │
│     ↓                                                        │
│ SELECT ... (只读操作)                                        │
│     ↓                                                        │
│ INSERT/UPDATE/DELETE  → SetReadWrite()                       │
│     ↓                                                        │
│ is_read_only = false ← 晋升为读写事务                        │
└─────────────────────────────────────────────────────────────┘
```

**设计优势**：
- OLAP 场景下大多数事务是只读的
- 只读事务可以避免获取写锁
- 减少检查点时的协调开销

## 1.3 DuckTransaction：MVCC 实现

### 1.3.1 核心字段

```cpp
// src/include/duckdb/transaction/duck_transaction.hpp
class DuckTransaction : public Transaction {
    // === 时间戳管理 ===
    transaction_t start_time;       // 事务开始时间戳 (快照点)
    transaction_t transaction_id;   // 事务唯一标识符
    transaction_t commit_id;        // 提交时间戳 (提交后设置)

    // === 事务本地数据 ===
    unique_ptr<UndoBuffer> undo_buffer;     // 回滚日志
    unique_ptr<LocalStorage> storage;        // 本地存储 (未提交插入)

    // === 并发控制 ===
    mutex sequence_lock;
    unordered_map<SequenceCatalogEntry *, SequenceValue> sequence_usage;

    mutex modified_tables_lock;
    reference_set_t<DataTable> modified_tables;

    mutex active_locks_lock;
    vector<StorageLockKey> active_locks;

    // === 元数据 ===
    atomic<idx_t> catalog_version;          // 目录版本
    bool is_invalidated;                     // 是否已失效
};
```

### 1.3.2 三个关键时间戳

```
时间线: 0 ────── 100 ────── 200 ────── 300 ────── 400
                  ↑                     ↑         ↑
             start_time            commit开始   commit_id
             (快照点)              (写WAL等)    (完成提交)

事务 T:
├── start_time = 100
│   - 决定事务能看到哪些已提交的数据
│   - 小于 100 的 commit_id 对 T 可见
│
├── transaction_id = 某唯一值 (如 1000001)
│   - 标识事务自身
│   - 用于识别"自己的修改"
│
└── commit_id = 300 (提交时分配)
    - 标识提交顺序
    - 其他事务根据此判断可见性
```

### 1.3.3 UndoBuffer 与 LocalStorage

```
DuckTransaction
│
├── UndoBuffer (回滚日志)
│   │
│   │  存储内容:
│   ├── UPDATE: UpdateInfo (旧值 + 版本链指针)
│   ├── DELETE: DeleteInfo (删除的行信息)
│   ├── INSERT: AppendInfo (插入位置，用于回滚时删除)
│   └── CATALOG: CatalogEntry (DDL 操作的旧状态)
│
│   作用:
│   ├── ROLLBACK 时恢复原状
│   ├── 为其他事务提供旧版本数据 (MVCC)
│   └── COMMIT 时写入 WAL
│
└── LocalStorage (本地存储)
    │
    │  存储内容:
    └── 事务插入但未提交的行数据
        (对其他事务不可见)

    提交时:
    └── 合并到主存储 (RowGroupCollection)
```

### 1.3.4 事务操作流程

```cpp
// 创建新事务
unique_ptr<DuckTransaction> DuckTransactionManager::StartTransaction() {
    lock_guard<mutex> lock(transaction_lock);

    // 1. 分配时间戳
    transaction_t start_time = current_start_timestamp++;
    transaction_t transaction_id = current_transaction_id++;

    // 2. 创建事务对象
    auto transaction = make_uniq<DuckTransaction>(
        *this, context, start_time, transaction_id);

    // 3. 加入活跃事务列表
    active_transactions.push_back(*transaction);

    // 4. 更新最低活跃事务标记
    if (IsLowestStartId(start_time)) {
        lowest_active_start = start_time;
        lowest_active_id = transaction_id;
    }

    return transaction;
}
```

## 1.4 TransactionManager：事务管理器

### 1.4.1 抽象接口

```cpp
// src/include/duckdb/transaction/transaction_manager.hpp
class TransactionManager {
protected:
    AttachedDatabase &db;  // 关联的数据库

public:
    // 核心事务操作
    virtual Transaction &StartTransaction(ClientContext &context) = 0;
    virtual ErrorData CommitTransaction(ClientContext &context,
                                        Transaction &transaction) = 0;
    virtual void RollbackTransaction(Transaction &transaction) = 0;

    // 检查点
    virtual void Checkpoint(ClientContext &context, bool force) = 0;

    // 类型检查
    virtual bool IsDuckTransactionManager();

    // 静态访问
    static TransactionManager &Get(AttachedDatabase &db);
};
```

### 1.4.2 可插拔设计

```
TransactionManager (抽象基类)
        ↑
        ├── DuckTransactionManager (DuckDB 原生 MVCC)
        │       └── 用于 .duckdb 文件
        │
        ├── SQLiteTransactionManager (假设)
        │       └── 用于 ATTACH 的 SQLite 数据库
        │
        └── PostgresTransactionManager (假设)
                └── 用于 ATTACH 的 Postgres 数据库
```

**设计优势**：
- 支持多种后端数据库
- 每个 Attached Database 可以有独立的事务语义
- 统一的上层接口

## 1.5 DuckTransactionManager：核心实现

### 1.5.1 时间戳计数器

```cpp
// src/include/duckdb/transaction/duck_transaction_manager.hpp
class DuckTransactionManager : public TransactionManager {
    // === 时间戳计数器 (全局递增) ===
    atomic<transaction_t> current_start_timestamp;  // 下一个 start_time
    atomic<transaction_t> current_transaction_id;   // 下一个 transaction_id

    // === 最低活跃事务追踪 (用于垃圾回收) ===
    atomic<transaction_t> lowest_active_id;
    atomic<transaction_t> lowest_active_start;

    // === 目录版本追踪 ===
    atomic<transaction_t> last_uncommitted_catalog_version;
    atomic<transaction_t> last_committed_version;
};
```

### 1.5.2 事务集合管理

```cpp
class DuckTransactionManager {
    // === 事务集合 ===
    vector<reference<DuckTransaction>> active_transactions;       // 活跃事务
    vector<unique_ptr<DuckTransaction>> recently_committed_transactions; // 近期提交
    deque<unique_ptr<DuckTransaction>> cleanup_queue;             // 清理队列

    // === 并发控制锁 ===
    mutex transaction_lock;       // 保护事务列表
    mutex checkpoint_lock;        // 检查点协调
    mutex cleanup_lock;           // 清理操作互斥
    mutex start_transaction_lock; // 阻止新事务 (FORCE CHECKPOINT)
};
```

### 1.5.3 事务生命周期

```
                    ┌──────────────────┐
                    │  StartTransaction │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ active_transactions│ ← 活跃运行中
                    │      列表          │
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              ↓                              ↓
     ┌─────────────────┐           ┌─────────────────┐
     │     COMMIT      │           │    ROLLBACK     │
     └────────┬────────┘           └────────┬────────┘
              ↓                              ↓
┌───────────────────────────┐     ┌──────────────────┐
│ recently_committed_transactions│   │  直接销毁事务   │
│        (暂存区)            │     │  (Undo已应用)    │
└────────────┬──────────────┘     └──────────────────┘
             ↓
    ┌─────────────────┐
    │  cleanup_queue  │ ← 等待垃圾回收
    │   (清理队列)     │
    └────────┬────────┘
             ↓
    ┌─────────────────┐
    │ 清理旧版本数据   │
    │ 销毁事务对象     │
    └─────────────────┘
```

### 1.5.4 最低活跃事务追踪

```cpp
// 作用: 确定可以安全清理的版本
transaction_t GetLowestActiveStart() {
    return lowest_active_start.load();
}

// 清理规则:
// 如果某版本的 version_number < lowest_active_start
// 则该版本不再被任何活跃事务需要，可以清理
```

**示例**：

```
活跃事务:
  T1: start_time = 100
  T2: start_time = 150
  T3: start_time = 200

lowest_active_start = 100

版本链:
  UpdateInfo(v=180) → UpdateInfo(v=120) → UpdateInfo(v=80)
                                                    ↑
                                          v=80 < 100，可以清理
```

## 1.6 TransactionContext：客户端接口

### 1.6.1 结构定义

```cpp
// src/include/duckdb/transaction/transaction_context.hpp
class TransactionContext {
    ClientContext &context;
    unique_ptr<MetaTransaction> current_transaction;
    bool auto_commit;

public:
    // 事务控制
    void BeginTransaction();
    void Commit();
    void Rollback(optional_ptr<ErrorData> error = nullptr);

    // 自动提交模式
    void SetAutoCommit(bool value);
    bool IsAutoCommit();

    // 只读模式
    void SetReadOnly();

    // 活跃查询追踪
    void SetActiveQuery(transaction_t query_number);
    void ResetActiveQuery();
};
```

### 1.6.2 自动提交模式

```
auto_commit = true (默认):
┌─────────────────────────────────────────────────────────────┐
│ INSERT INTO t VALUES (1);                                    │
│     ↓                                                        │
│ 隐式 BeginTransaction()                                      │
│     ↓                                                        │
│ 执行 INSERT                                                  │
│     ↓                                                        │
│ 隐式 Commit()                                                │
└─────────────────────────────────────────────────────────────┘

auto_commit = false (显式事务):
┌─────────────────────────────────────────────────────────────┐
│ BEGIN TRANSACTION;        → BeginTransaction()              │
│ INSERT INTO t VALUES (1); → 执行，不提交                     │
│ INSERT INTO t VALUES (2); → 执行，不提交                     │
│ COMMIT;                   → Commit()                        │
└─────────────────────────────────────────────────────────────┘
```

## 1.7 MetaTransaction：多数据库协调

### 1.7.1 设计背景

DuckDB 支持 ATTACH 多个数据库，事务可能跨越多个数据库：

```sql
ATTACH 'other.duckdb' AS other_db;

BEGIN;
INSERT INTO main.t1 VALUES (1);      -- 修改主数据库
SELECT * FROM other_db.t2;           -- 读取其他数据库
COMMIT;
```

### 1.7.2 结构定义

```cpp
// src/include/duckdb/transaction/meta_transaction.hpp
class MetaTransaction {
    // 每个数据库一个事务
    unordered_map<AttachedDatabase *, TransactionReference> transactions;

    // 事务开启顺序
    vector<reference<AttachedDatabase>> all_transactions;

    // 单写约束
    optional_ptr<AttachedDatabase> modified_database;

    // 跨库唯一ID
    transaction_t global_transaction_id;

    // 防止 Detach
    unordered_map<string, shared_ptr<AttachedDatabase>> referenced_databases;
};
```

### 1.7.3 单写约束

```
MetaTransaction 规则:
┌─────────────────────────────────────────────────────────────┐
│ 一个 MetaTransaction 最多只能修改一个数据库                   │
│                                                              │
│ 原因: 避免分布式事务的复杂性 (2PC)                           │
│                                                              │
│ 允许:                                                        │
│   - 读取多个数据库                                           │
│   - 修改一个数据库 + 读取其他数据库                          │
│                                                              │
│ 禁止:                                                        │
│   - 同时修改多个数据库                                       │
└─────────────────────────────────────────────────────────────┘

实现:
void MetaTransaction::ModifyDatabase(AttachedDatabase &db) {
    if (modified_database && modified_database != &db) {
        throw TransactionException(
            "Cannot modify multiple databases in single transaction");
    }
    modified_database = &db;
}
```

### 1.7.4 协调提交

```cpp
ErrorData MetaTransaction::Commit() {
    // 按事务开启顺序提交
    for (auto &db : all_transactions) {
        auto &tx_ref = transactions[&db];
        if (tx_ref.state == TransactionState::UNCOMMITTED) {
            auto error = db.GetTransactionManager().CommitTransaction(
                context, *tx_ref.transaction);
            if (error.HasError()) {
                // 发生错误，回滚其他事务
                Rollback();
                return error;
            }
            tx_ref.state = TransactionState::COMMITTED;
        }
    }
    return ErrorData();
}
```

## 1.8 事务隔离级别

### 1.8.1 快照隔离 (Snapshot Isolation)

DuckDB 实现了快照隔离：

```
事务 T1                     事务 T2
  │                           │
  ├─ BEGIN (start=100)        │
  │                           ├─ BEGIN (start=150)
  ├─ SELECT * FROM t          │
  │   → 看到 start<100 的数据  │
  │                           ├─ UPDATE t SET x=2
  │                           ├─ COMMIT (commit_id=160)
  │                           │
  ├─ SELECT * FROM t          │
  │   → 仍然看到旧数据!        │
  │   (160 > 100, 更新不可见)  │
  │                           │
  ├─ COMMIT                   │
```

### 1.8.2 写写冲突检测

```
事务 T1                     事务 T2
  │                           │
  ├─ BEGIN                    ├─ BEGIN
  │                           │
  ├─ UPDATE t SET x=1         │
  │   WHERE id=5              │
  │                           ├─ UPDATE t SET x=2
  │                           │   WHERE id=5
  │                           │   → 检测到冲突!
  │                           │   → 等待或中止
  ├─ COMMIT                   │
```

## 1.9 与存储层的集成

### 1.9.1 事务与检查点

```
DuckTransactionManager 与 CheckpointManager 的协作:
┌─────────────────────────────────────────────────────────────┐
│ 1. 检查点需要等待活跃写事务完成                              │
│    (获取 checkpoint_lock 的排他锁)                          │
│                                                              │
│ 2. FORCE CHECKPOINT 可以阻止新事务启动                       │
│    (获取 start_transaction_lock)                            │
│                                                              │
│ 3. 事务持有 storage_lock 防止检查点期间的数据修改            │
└─────────────────────────────────────────────────────────────┘
```

### 1.9.2 事务与 WAL

```
事务提交流程:
┌─────────────────────────────────────────────────────────────┐
│ 1. 遍历 UndoBuffer                                           │
│ 2. 将修改写入 WAL                                            │
│    - INSERT_TUPLE                                            │
│    - DELETE_TUPLE                                            │
│    - UPDATE_TUPLE                                            │
│ 3. 写入 WAL_FLUSH 标记                                       │
│ 4. fsync() 确保持久化                                        │
│ 5. 分配 commit_id                                            │
│ 6. 更新版本信息，使修改对其他事务可见                         │
└─────────────────────────────────────────────────────────────┘
```

## 1.10 小结

DuckDB 事务系统的核心设计特点：

| 特点 | 实现方式 |
|------|----------|
| **快照隔离** | start_time 确定可见性边界 |
| **MVCC** | UndoBuffer 存储旧版本 |
| **只读优化** | 默认只读，按需晋升 |
| **多库支持** | MetaTransaction 协调 |
| **单写约束** | 避免分布式事务复杂性 |
| **可插拔** | TransactionManager 抽象 |
| **持久性** | WAL + Checkpoint 配合 |

核心源码位置：
- `src/include/duckdb/transaction/transaction.hpp` - 事务基类
- `src/include/duckdb/transaction/duck_transaction.hpp` - MVCC 实现
- `src/include/duckdb/transaction/transaction_manager.hpp` - 管理器接口
- `src/include/duckdb/transaction/duck_transaction_manager.hpp` - 管理器实现
- `src/include/duckdb/transaction/transaction_context.hpp` - 客户端上下文
- `src/include/duckdb/transaction/meta_transaction.hpp` - 多库协调

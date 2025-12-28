# 第二章：MVCC 实现机制

多版本并发控制 (MVCC, Multi-Version Concurrency Control) 是 DuckDB 事务系统的核心。通过维护数据的多个版本，MVCC 允许读操作和写操作并发执行而无需加锁，实现了高效的快照隔离。本章将深入分析 DuckDB 的 MVCC 实现机制。

## 2.1 MVCC 设计概览

### 2.1.1 核心设计目标

DuckDB 的 MVCC 系统需要解决以下问题：

| 问题 | 解决方案 |
|------|----------|
| 读写并发 | 每个事务看到一致的数据快照 |
| 更新追踪 | 版本链记录历史值 |
| 删除追踪 | 标记删除而非物理删除 |
| 插入可见性 | 记录插入事务ID |
| 垃圾回收 | 清理不再需要的旧版本 |

### 2.1.2 版本管理分层

DuckDB 将版本管理分为两个层次：

```
┌─────────────────────────────────────────────────────────────┐
│           行级可见性 (Insert/Delete)                         │
│    RowVersionManager → ChunkInfo (ChunkConstantInfo/        │
│                                   ChunkVectorInfo)          │
├─────────────────────────────────────────────────────────────┤
│           列级版本链 (Update)                                │
│    UpdateSegment → UpdateInfo (双向链表)                     │
└─────────────────────────────────────────────────────────────┘
```

**设计原理**：
- **Insert/Delete** 影响整行的可见性，由 RowVersionManager 统一管理
- **Update** 只修改特定列，通过每列独立的版本链追踪

## 2.2 UpdateInfo：更新版本链

### 2.2.1 数据结构

UpdateInfo 是 MVCC 更新版本链的基本单元，采用紧凑的内存布局：

```cpp
// src/include/duckdb/transaction/update_info.hpp
struct UpdateInfo {
    UpdateSegment *segment;          // 所属的更新段
    DataTable *table;                // 所属表
    idx_t column_index;              // 列索引
    idx_t row_group_start;           // 行组起始位置
    atomic<transaction_t> version_number;  // 版本号 (关键!)
    idx_t vector_index;              // 向量索引
    sel_t N;                         // 更新的元组数量
    sel_t max;                       // 最大容量
    UndoBufferPointer prev;          // 前驱 (更新的版本)
    UndoBufferPointer next;          // 后继 (更旧的版本)
};
```

**内存布局**：

```
┌──────────────────┬────────────────────┬──────────────────────┐
│   UpdateInfo     │  Tuple IDs         │   Old Values         │
│   (结构体头部)    │  sel_t[max]        │   T[max]             │
│   ~64 bytes      │  (排序的行号)       │   (旧值数据)          │
└──────────────────┴────────────────────┴──────────────────────┘
```

### 2.2.2 版本链结构

版本链是一个双向链表，从最新版本指向最旧版本：

```
          newest                                        oldest
            ↓                                             ↓
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  UpdateInfo      │    │  UpdateInfo      │    │  UpdateInfo      │
│  version: 150    │←──→│  version: 120    │←──→│  version: 100    │
│  (事务150的更新)  │next│  (事务120的更新)  │next│  (事务100的更新)  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
         ↑                      ↑                       ↑
      head (UpdateSegment)      中间节点              tail (最旧)
```

### 2.2.3 可见性判定

```cpp
bool AppliesToTransaction(transaction_t start_time, transaction_t transaction_id) {
    // 该更新对当前事务"适用"（需要看旧值）的条件：
    // 1. version_number > start_time: 该更新在当前事务开始之后提交
    // 2. version_number != transaction_id: 不是当前事务自己的更新
    return version_number > start_time && version_number != transaction_id;
}
```

**可见性规则图解**：

```
时间线:  100 ──────── 120 ──────── 150 ──────── 200
                      ↑                         ↑
                事务T开始                    事务T读取
                (start_time=120)

版本链:
  UpdateInfo(v=150) → UpdateInfo(v=100)
        ↑                    ↑
  v=150 > 120 且           v=100 <= 120
  v=150 ≠ T的ID            不适用，停止遍历
  → 适用，返回旧值
```

### 2.2.4 版本链遍历

```cpp
template <class T>
static void UpdatesForTransaction(UpdateInfo &current,
                                  transaction_t start_time,
                                  transaction_t transaction_id,
                                  T &&callback) {
    // 检查当前节点
    if (current.AppliesToTransaction(start_time, transaction_id)) {
        callback(current);
    }
    // 遍历后续节点 (从新到旧)
    auto update_ptr = current.next;
    while (update_ptr.IsSet()) {
        auto pin = update_ptr.Pin();  // 固定内存
        auto &info = Get(pin);
        if (info.AppliesToTransaction(start_time, transaction_id)) {
            callback(info);
        }
        update_ptr = info.next;
    }
}
```

## 2.3 UpdateSegment：列级更新管理

### 2.3.1 结构设计

UpdateSegment 为单个列管理所有的更新版本链：

```cpp
// src/include/duckdb/storage/table/update_segment.hpp
class UpdateSegment {
    // 根节点 - 存储每个向量的版本链头
    unique_ptr<UpdateNode> root;

    // 类型特定的函数指针 (避免虚函数开销)
    initialize_update_function_t initialize_update;
    merge_update_function_t merge_update;
    fetch_update_function_t fetch_update;
    rollback_update_function_t rollback_update;

    // 字符串堆 (变长类型)
    unique_ptr<StringHeap> string_heap;

    // 并发控制
    StorageLock lock;
};

struct UpdateNode {
    UndoBufferAllocator allocator;
    // 每个向量索引对应一个版本链头
    vector<UndoBufferPointer> info;  // info[vector_idx] → UpdateInfo链头
};
```

### 2.3.2 函数分发模式

DuckDB 使用函数指针而非虚函数来处理类型特定操作：

```cpp
// 构造时根据类型绑定函数
UpdateSegment::UpdateSegment(LogicalType type) {
    switch (type.InternalType()) {
    case PhysicalType::INT32:
        initialize_update = InitializeUpdateData<int32_t>;
        merge_update = MergeUpdateData<int32_t>;
        fetch_update = FetchUpdateData<int32_t>;
        rollback_update = RollbackUpdateData<int32_t>;
        break;
    case PhysicalType::VARCHAR:
        // 字符串类型需要特殊处理
        initialize_update = InitializeUpdateDataString;
        // ...
        break;
    // ... 其他类型
    }
}
```

### 2.3.3 更新操作流程

```
Update(transaction, row_ids, data):
┌─────────────────────────────────────────────────────────────┐
│ 1. 获取写锁 (StorageLock)                                    │
├─────────────────────────────────────────────────────────────┤
│ 2. 为每个向量创建/获取 UpdateInfo                            │
│    - 通过 UndoBuffer 分配内存                                │
│    - 调用 initialize_update 初始化                           │
├─────────────────────────────────────────────────────────────┤
│ 3. 合并到现有版本链                                          │
│    - 调用 merge_update 处理类型特定的合并逻辑                 │
│    - 维护 Tuple IDs 的排序顺序                               │
│    - 更新 prev/next 指针                                     │
├─────────────────────────────────────────────────────────────┤
│ 4. 更新统计信息                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.3.4 获取更新值

```cpp
void UpdateSegment::FetchUpdates(TransactionData transaction,
                                 idx_t vector_index,
                                 Vector &result) {
    // 1. 获取该向量的版本链头
    auto &info_ptr = root->info[vector_index];
    if (!info_ptr.IsSet()) return;

    // 2. 遍历版本链，应用可见的更新
    auto pin = info_ptr.Pin();
    auto &info = UpdateInfo::Get(pin);

    UpdateInfo::UpdatesForTransaction(
        info,
        transaction.start_time,
        transaction.transaction_id,
        [&](UpdateInfo &update) {
            // 3. 使用类型特定函数将旧值合并到结果
            fetch_update(update, result);
        }
    );
}
```

## 2.4 行级可见性管理

### 2.4.1 ChunkInfo 层次结构

```cpp
// 基类
class ChunkInfo {
    idx_t start;           // 起始行号
    ChunkInfoType type;    // 类型标识

    // 核心接口
    virtual idx_t GetSelVector(TransactionData transaction,
                               SelectionVector &sel_vector,
                               idx_t max_count) const = 0;
    virtual bool Fetch(TransactionData transaction, row_t row) = 0;
};

// 常量版本 - 所有行具有相同的插入/删除ID
class ChunkConstantInfo : public ChunkInfo {
    transaction_t insert_id;   // 统一的插入事务ID
    transaction_t delete_id;   // 统一的删除事务ID (0表示未删除)
};

// 向量版本 - 每行独立的版本信息
class ChunkVectorInfo : public ChunkInfo {
    IndexPointer inserted_data;      // 每行的插入ID数组
    transaction_t constant_insert_id; // 优化：常量插入ID
    IndexPointer deleted_data;        // 每行的删除ID数组
};
```

### 2.4.2 优化策略

```
场景分析:
┌─────────────────────────────────────────────────────────────┐
│ 场景1: 批量插入 (INSERT INTO ... VALUES (...), (...), ...) │
│ → 所有行具有相同的 insert_id                                │
│ → 使用 ChunkConstantInfo 节省内存                           │
├─────────────────────────────────────────────────────────────┤
│ 场景2: 部分行被删除                                         │
│ → 需要追踪每行的 delete_id                                  │
│ → 升级为 ChunkVectorInfo                                    │
├─────────────────────────────────────────────────────────────┤
│ 场景3: 混合插入时间                                         │
│ → 不同行有不同的 insert_id                                  │
│ → 需要 ChunkVectorInfo                                      │
└─────────────────────────────────────────────────────────────┘
```

### 2.4.3 可见性判定算法

```cpp
// ChunkVectorInfo::GetSelVector 的核心逻辑
idx_t ChunkVectorInfo::GetSelVector(transaction_t start_time,
                                    transaction_t transaction_id,
                                    SelectionVector &sel_vector,
                                    idx_t max_count) const {
    idx_t count = 0;
    for (idx_t i = 0; i < max_count; i++) {
        transaction_t insert_id = GetInsertId(i);
        transaction_t delete_id = GetDeleteId(i);

        bool visible = IsVisible(insert_id, delete_id,
                                 start_time, transaction_id);
        if (visible) {
            sel_vector.set_index(count++, i);
        }
    }
    return count;
}
```

**可见性规则**：

```cpp
bool IsVisible(transaction_t insert_id, transaction_t delete_id,
               transaction_t start_time, transaction_t transaction_id) {
    // 规则1: 行必须已经被插入且对当前事务可见
    bool inserted_visible =
        insert_id < start_time ||           // 在事务开始前提交
        insert_id == transaction_id;        // 或者是当前事务插入的

    // 规则2: 行不能被删除，或删除对当前事务不可见
    bool not_deleted =
        delete_id == 0 ||                   // 未被删除
        (delete_id >= start_time &&         // 删除在事务开始后提交
         delete_id != transaction_id);      // 且不是当前事务删除的

    return inserted_visible && not_deleted;
}
```

**图解**：

```
时间线: 0 ────── 50 ────── 100 ────── 150 ────── 200
                          ↑                     ↑
                     事务T开始              事务T读取
                   (start_time=100)

行 R1: insert_id=50,  delete_id=0    → 可见 (50<100, 未删除)
行 R2: insert_id=50,  delete_id=80   → 不可见 (80<100, 已删除)
行 R3: insert_id=50,  delete_id=150  → 可见 (删除在事务后)
行 R4: insert_id=120, delete_id=0    → 不可见 (120>100, 插入在事务后)
行 R5: insert_id=T,   delete_id=0    → 可见 (自己插入的)
```

## 2.5 RowVersionManager：行组版本协调器

### 2.5.1 职责

RowVersionManager 为整个行组协调版本管理：

```cpp
// src/include/duckdb/storage/table/row_version_manager.hpp
class RowVersionManager {
    // 每个向量一个 ChunkInfo
    vector<unique_ptr<ChunkInfo>> vector_info;

    // 内存分配器
    FixedSizeAllocator allocator;

    // 脏标记
    bool has_unserialized_changes;
};
```

### 2.5.2 操作接口

```cpp
// 获取可见行的选择向量
idx_t GetSelVector(TransactionData transaction,
                   idx_t vector_idx,
                   SelectionVector &result,
                   idx_t max_count);

// 追踪新插入的行
void AppendVersionInfo(TransactionData transaction,
                       idx_t count,
                       idx_t row_group_start);

// 提交插入
void CommitAppend(transaction_t commit_id,
                  idx_t row_group_start,
                  idx_t count);

// 撤销插入
void RevertAppend(idx_t row_group_start);

// 删除行
idx_t DeleteRows(transaction_t transaction_id,
                 row_t *row_ids,
                 idx_t count);

// 提交删除
void CommitDelete(transaction_t commit_id,
                  DeleteInfo &info);
```

### 2.5.3 与存储层的集成

```
RowGroup
├── RowVersionManager (版本管理)
│   └── vector_info[N] → ChunkInfo
│
├── columns[M] (列数据)
│   └── ColumnData
│       └── ColumnSegmentTree
│           └── ColumnSegment
│               └── UpdateSegment (列级版本链)
│
└── 扫描时的协作:
    1. RowVersionManager.GetSelVector() → 确定可见行
    2. ColumnSegment.Scan() → 读取基础数据
    3. UpdateSegment.FetchUpdates() → 应用版本链
```

## 2.6 DeleteInfo：删除元数据

### 2.6.1 结构设计

```cpp
// src/include/duckdb/transaction/delete_info.hpp
struct DeleteInfo {
    DataTable *table;              // 所属表
    RowVersionManager *version_info; // 版本管理器
    idx_t vector_idx;              // 向量索引
    idx_t base_row;                // 基础行号
    idx_t count;                   // 删除的行数
    bool is_consecutive;           // 是否连续删除

    // 变长部分: 非连续删除时的行号数组
    uint16_t rows[];  // rows[i] 表示 base_row + rows[i] 被删除
};
```

### 2.6.2 内存优化

```
连续删除优化:
┌─────────────────────────────────────────────────────────────┐
│ DELETE FROM t WHERE id BETWEEN 100 AND 199                  │
│                                                              │
│ 传统方式: 存储 100 个行号 → 200 bytes                        │
│ 优化方式: is_consecutive=true, base_row=100, count=100      │
│           → 仅需 DeleteInfo 头部 (~32 bytes)                 │
└─────────────────────────────────────────────────────────────┘

非连续删除:
┌─────────────────────────────────────────────────────────────┐
│ DELETE FROM t WHERE id IN (5, 10, 15, 20)                   │
│                                                              │
│ is_consecutive=false                                         │
│ rows[] = {5, 10, 15, 20}  (相对于 base_row 的偏移)          │
└─────────────────────────────────────────────────────────────┘
```

## 2.7 MVCC 操作流程

### 2.7.1 读取流程

```
SELECT * FROM table WHERE ...
┌─────────────────────────────────────────────────────────────┐
│ 1. 获取事务快照 (start_time, transaction_id)                 │
├─────────────────────────────────────────────────────────────┤
│ 2. 遍历 RowGroup                                             │
│    │                                                         │
│    ├── 2.1 RowVersionManager.GetSelVector()                 │
│    │        → 确定哪些行可见，返回 SelectionVector           │
│    │                                                         │
│    ├── 2.2 ColumnSegment.Scan()                             │
│    │        → 读取基础数据到结果向量                          │
│    │                                                         │
│    └── 2.3 UpdateSegment.FetchUpdates()                     │
│             → 遍历版本链，用旧值替换被后续事务更新的值        │
│                                                              │
│ 3. 返回符合事务快照的数据                                    │
└─────────────────────────────────────────────────────────────┘
```

### 2.7.2 插入流程

```
INSERT INTO table VALUES (...)
┌─────────────────────────────────────────────────────────────┐
│ 1. 数据写入 LocalStorage (事务私有)                          │
├─────────────────────────────────────────────────────────────┤
│ 2. 记录 AppendInfo 到 UndoBuffer                            │
│    - 用于回滚时删除插入的行                                  │
├─────────────────────────────────────────────────────────────┤
│ 3. 设置 insert_id = transaction_id (未提交)                  │
├─────────────────────────────────────────────────────────────┤
│ 4. COMMIT 时:                                                │
│    - 合并 LocalStorage 到主存储                              │
│    - 更新 insert_id = commit_id                              │
│    - 其他事务现在可以看到这些行                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.7.3 更新流程

```
UPDATE table SET col = new_value WHERE ...
┌─────────────────────────────────────────────────────────────┐
│ 1. 定位要更新的行                                            │
├─────────────────────────────────────────────────────────────┤
│ 2. 创建 UpdateInfo 记录旧值                                  │
│    - 分配在 UndoBuffer 中                                    │
│    - 设置 version_number = transaction_id                    │
├─────────────────────────────────────────────────────────────┤
│ 3. 插入到 UpdateSegment 的版本链头部                         │
│    - 更新 prev/next 指针                                     │
│    - 原地修改列数据为新值                                    │
├─────────────────────────────────────────────────────────────┤
│ 4. COMMIT 时:                                                │
│    - 更新 version_number = commit_id                         │
│    - 其他事务看到的值取决于其 start_time                     │
├─────────────────────────────────────────────────────────────┤
│ 5. ROLLBACK 时:                                              │
│    - 从版本链中移除 UpdateInfo                               │
│    - 恢复原始值                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.7.4 删除流程

```
DELETE FROM table WHERE ...
┌─────────────────────────────────────────────────────────────┐
│ 1. 定位要删除的行                                            │
├─────────────────────────────────────────────────────────────┤
│ 2. 创建 DeleteInfo 记录删除元数据                            │
│    - 存储在 UndoBuffer 中                                    │
├─────────────────────────────────────────────────────────────┤
│ 3. 更新 ChunkInfo                                            │
│    - 设置 delete_id = transaction_id                         │
│    - 行对其他事务仍然可见                                    │
├─────────────────────────────────────────────────────────────┤
│ 4. COMMIT 时:                                                │
│    - 更新 delete_id = commit_id                              │
│    - 后续事务将看不到这些行                                  │
├─────────────────────────────────────────────────────────────┤
│ 5. ROLLBACK 时:                                              │
│    - 重置 delete_id = 0                                      │
│    - 行恢复可见                                              │
└─────────────────────────────────────────────────────────────┘
```

## 2.8 版本链内存管理

### 2.8.1 UndoBufferPointer 设计

```cpp
// 轻量级未固定引用
class UndoBufferPointer {
    UndoBufferEntry *entry;  // 指向缓冲区条目
    idx_t position;          // 条目内偏移

    // 需要访问时固定内存
    UndoBufferReference Pin();
};

// 固定引用 - RAII 管理
class UndoBufferReference {
    BufferHandle handle;     // 持有缓冲区
    data_ptr_t data;         // 直接数据指针

    ~UndoBufferReference() { /* 自动释放 */ }
};
```

**设计优势**：
- UpdateInfo 的 prev/next 使用 UndoBufferPointer，不持有引用
- 仅在需要访问时 Pin()，避免长期占用缓冲区
- 支持版本链溢出到磁盘（临时文件）

### 2.8.2 内存布局

```
UndoBuffer 结构:
┌─────────────────────────────────────────────────────────────┐
│ UndoBufferEntry 1 (256KB block)                             │
│  ├── [UndoFlags][UpdateInfo][tuples][values]                │
│  ├── [UndoFlags][DeleteInfo][rows...]                       │
│  └── [UndoFlags][AppendInfo][...]                           │
├─────────────────────────────────────────────────────────────┤
│ UndoBufferEntry 2 (256KB block)                             │
│  ├── ...                                                     │
│  └── ...                                                     │
└─────────────────────────────────────────────────────────────┘
          ↑
   通过 UndoBufferPointer 引用
```

## 2.9 设计权衡与优化

### 2.9.1 列式存储的 MVCC 挑战

| 挑战 | DuckDB 解决方案 |
|------|-----------------|
| 更新代价高 | 版本链只记录修改的列 |
| 多版本空间开销 | 及时垃圾回收 + 检查点压缩 |
| 版本链遍历开销 | 函数指针避免虚调用 |
| 内存压力 | UndoBuffer 可溢出到磁盘 |

### 2.9.2 与 OLTP 系统的区别

```
传统 OLTP (如 PostgreSQL):
  - 行存储，更新整行
  - 版本存储在堆表或专门的回滚段
  - VACUUM 进程清理死元组

DuckDB (OLAP + 列存储):
  - 列存储，只版本化修改的列
  - 版本存储在 UndoBuffer (事务私有)
  - 事务结束时按需清理 + 检查点时压缩
  - 读操作更高效 (列投影)
```

### 2.9.3 快照隔离保证

```
事务 T1 (start_time=100)     事务 T2 (start_time=150)
      │                            │
      ├─ BEGIN                     │
      ├─ SELECT ... (看到旧值)      │
      │                            ├─ BEGIN
      │                            ├─ UPDATE col=new_value
      │                            ├─ COMMIT (commit_id=160)
      │                            │
      ├─ SELECT ... (仍看到旧值!)   │
      │   ↑ version=160 > 100      │
      │     所以使用版本链中的旧值   │
      ├─ COMMIT                    │
```

## 2.10 小结

DuckDB 的 MVCC 实现体现了以下设计特点：

| 特点 | 实现方式 |
|------|----------|
| **分层设计** | 行级 (ChunkInfo) + 列级 (UpdateInfo) 分离 |
| **空间优化** | ChunkConstantInfo 优化批量操作 |
| **类型特化** | 函数指针分发避免虚函数开销 |
| **懒加载** | UndoBufferPointer 延迟 Pin |
| **快照隔离** | 基于 version_number 的可见性判定 |

核心源码位置：
- `src/include/duckdb/transaction/update_info.hpp` - 更新版本链
- `src/include/duckdb/storage/table/chunk_info.hpp` - 行可见性
- `src/include/duckdb/storage/table/row_version_manager.hpp` - 行组版本管理
- `src/include/duckdb/storage/table/update_segment.hpp` - 列级更新管理
- `src/include/duckdb/transaction/delete_info.hpp` - 删除元数据

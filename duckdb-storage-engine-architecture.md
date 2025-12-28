# DuckDB 存储引擎架构概述

DuckDB 是一款面向 OLAP 场景的嵌入式分析型数据库，其存储引擎采用了现代列式存储架构。本文将从整体视角介绍 DuckDB 存储引擎的核心架构、主要组件及其协作机制。

## 一、架构总览

### 1.1 分层架构

DuckDB 存储引擎采用清晰的分层设计：

```
┌─────────────────────────────────────────────────────────────┐
│                    查询执行层 (Execution)                    │
│              PhysicalTableScan / PhysicalInsert              │
├─────────────────────────────────────────────────────────────┤
│                    表存储层 (Table Storage)                  │
│         DataTable → RowGroupCollection → RowGroup           │
├─────────────────────────────────────────────────────────────┤
│                    列存储层 (Column Storage)                 │
│              ColumnData → ColumnSegment                      │
├─────────────────────────────────────────────────────────────┤
│                    压缩层 (Compression)                      │
│         Dictionary / Bitpacking / RLE / ALP / ...           │
├─────────────────────────────────────────────────────────────┤
│                    缓冲管理层 (Buffer Management)            │
│              BufferManager → BufferPool → BlockHandle        │
├─────────────────────────────────────────────────────────────┤
│                    块管理层 (Block Management)               │
│              BlockManager → Block (256KB)                    │
├─────────────────────────────────────────────────────────────┤
│                    持久化层 (Persistence)                    │
│              SingleFileBlockManager / WAL                    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件关系

```
StorageManager (顶层入口)
├── BlockManager (块管理)
│   └── SingleFileBlockManager (单文件实现)
├── BufferManager (缓冲管理)
│   ├── BufferPool (内存池)
│   └── TemporaryFileManager (临时文件)
├── CheckpointManager (检查点)
├── WriteAheadLog (WAL日志)
└── TableIOManager (表IO管理)
```

## 二、存储管理器 (StorageManager)

### 2.1 职责

StorageManager 是存储引擎的顶层入口，负责：

- 管理数据库文件的打开/关闭
- 协调检查点操作
- 管理 WAL 日志
- 提供 BlockManager 和 BufferManager 访问接口

### 2.2 核心实现

```cpp
// src/include/duckdb/storage/storage_manager.hpp
class StorageManager {
protected:
    AttachedDatabase &db;
    unique_ptr<WriteAheadLog> wal;

public:
    virtual BlockManager &GetBlockManager() = 0;
    virtual void CreateCheckpoint(CheckpointOptions options) = 0;
    virtual unique_ptr<TableIOManager> GetTableIOManager(DataTableInfo &info) = 0;
};

// 单文件存储实现
class SingleFileStorageManager : public StorageManager {
    unique_ptr<SingleFileBlockManager> block_manager;
    unique_ptr<CheckpointManager> checkpoint_manager;
};
```

## 三、块管理器 (BlockManager)

### 3.1 块的概念

DuckDB 将数据组织为固定大小的块 (Block)：

```cpp
// 关键常量
constexpr idx_t DEFAULT_BLOCK_ALLOC_SIZE = 262144;  // 256KB
constexpr idx_t DEFAULT_BLOCK_HEADER_SIZE = 8;       // 校验和
constexpr idx_t DEFAULT_BLOCK_SIZE = 262144 - 8;     // 可用空间
```

### 3.2 BlockManager 接口

```cpp
class BlockManager {
public:
    // 块分配
    virtual block_id_t GetFreeBlockId() = 0;
    virtual unique_ptr<Block> CreateBlock(block_id_t id, FileBuffer *source) = 0;

    // 块读写
    virtual void Read(QueryContext context, Block &block) = 0;
    virtual void Write(FileBuffer &buffer, block_id_t block_id) = 0;

    // 块状态管理
    virtual void MarkBlockAsUsed(block_id_t block_id) = 0;
    virtual void MarkBlockAsModified(block_id_t block_id) = 0;

    // 检查点
    virtual void WriteHeader(QueryContext context, DatabaseHeader header) = 0;
};
```

### 3.3 块生命周期

```
分配新块 → newly_used_blocks
    ↓ (检查点完成)
正常使用中 (已持久化)
    ↓ (内容被修改/删除)
modified_blocks (等待下次检查点)
    ↓ (下次检查点完成)
free_list → 可重新分配
```

## 四、缓冲管理器 (BufferManager)

### 4.1 设计目标

- 管理内存中的数据块缓存
- 实现 LRU 驱逐策略
- 支持内存压力下的溢出到临时文件
- 提供 Pin/Unpin 语义保证块存活

### 4.2 核心组件

```cpp
class StandardBufferManager : public BufferManager {
    BufferPool &buffer_pool;              // 全局共享内存池
    InMemoryBlockManager temp_block_manager; // 临时块管理
    TemporaryFileManager temporary_file_manager; // 临时文件管理
    BlockAllocator buffer_allocator;       // 内存分配器
};
```

### 4.3 BlockHandle 与 BufferHandle

```cpp
// BlockHandle: 块的内存管理句柄
class BlockHandle {
    block_id_t block_id;
    BlockState state;           // BLOCK_UNLOADED / BLOCK_LOADED
    atomic<int32_t> readers;    // 并发读取器计数
    unique_ptr<FileBuffer> buffer;
    BufferPoolReservation memory_charge;
    atomic<idx_t> eviction_seq_num;  // LRU 序列号
};

// BufferHandle: RAII 包装，自动管理 Pin/Unpin
class BufferHandle {
    shared_ptr<BlockHandle> handle;
    data_ptr_t data_ptr;

    ~BufferHandle() { handle->Unpin(); }  // 自动 Unpin
};
```

### 4.4 内存管理流程

```
Pin(block_id):
┌──────────────────────────────────────────┐
│ 1. 查找或创建 BlockHandle                 │
│ 2. 如果 BLOCK_UNLOADED:                   │
│    - 预留内存 (可能触发驱逐)               │
│    - 从磁盘/临时文件加载数据               │
│    - state → BLOCK_LOADED                 │
│ 3. readers++                              │
│ 4. 返回 BufferHandle                      │
└──────────────────────────────────────────┘

Unpin():
┌──────────────────────────────────────────┐
│ 1. readers--                              │
│ 2. 如果 readers == 0:                     │
│    - 加入驱逐队列                         │
│    - 更新 eviction_seq_num                │
└──────────────────────────────────────────┘

驱逐 (Eviction):
┌──────────────────────────────────────────┐
│ 1. 从驱逐队列取出最老的块                 │
│ 2. 检查 CanUnload() (readers == 0)        │
│ 3. 如果需要持久化，写入临时文件            │
│ 4. 释放内存预留                           │
│ 5. state → BLOCK_UNLOADED                 │
└──────────────────────────────────────────┘
```

## 五、表存储模型

### 5.1 层次结构

```
DataTable (数据表)
│
├── info: DataTableInfo (表元信息)
│   ├── schema, table_name
│   ├── column_definitions
│   └── indexes: TableIndexList
│
├── row_groups: RowGroupCollection (持久化数据)
│   └── SegmentTree<RowGroup>
│       ├── RowGroup 0 (rows 0 - 122879)
│       ├── RowGroup 1 (rows 122880 - 245759)
│       └── ...
│
└── local_storage: LocalStorage (事务本地修改)
```

### 5.2 行组 (RowGroup)

行组是 DuckDB 数据组织的核心单元：

```cpp
// 默认行组大小: 122880 行
#define DEFAULT_ROW_GROUP_SIZE 122880ULL

class RowGroup {
    idx_t start;                    // 起始行号
    atomic<idx_t> count;            // 当前行数
    vector<shared_ptr<ColumnData>> columns;  // 列数据
    RowVersionManager version_info; // MVCC 版本信息
    vector<MetaBlockPointer> column_pointers; // 持久化指针
};
```

**行组设计优势：**

| 特性 | 说明 |
|------|------|
| 并行处理 | 不同行组可并行扫描 |
| 增量检查点 | 只检查点修改的行组 |
| 内存效率 | 按行组加载，避免全表加载 |
| 统计裁剪 | 行组级别 zonemap 过滤 |

### 5.3 列数据 (ColumnData)

```cpp
class ColumnData {
    LogicalType type;               // 列类型
    idx_t start;                    // 起始行
    atomic<idx_t> count;            // 行数
    ColumnSegmentTree data;         // 段树
    unique_ptr<ColumnData> validity; // NULL 位图 (可选)
};

// 复合类型支持
class StructColumnData : public ColumnData {
    vector<unique_ptr<ColumnData>> sub_columns;
};

class ListColumnData : public ColumnData {
    unique_ptr<ColumnData> child_column;
    unique_ptr<ColumnData> offset_column;
};
```

### 5.4 列段 (ColumnSegment)

列段是实际存储压缩数据的单元：

```cpp
class ColumnSegment {
    ColumnSegmentType type;         // TRANSIENT / PERSISTENT
    LogicalType column_type;
    idx_t start, count;             // 行范围

    // 持久化信息
    block_id_t block_id;
    idx_t offset;                   // 块内偏移
    idx_t segment_size;

    // 压缩相关
    CompressionType compression;
    unique_ptr<ColumnSegmentState> segment_state;

    // 统计信息
    unique_ptr<SegmentStatistics> stats;
};
```

## 六、压缩系统

### 6.1 支持的压缩算法

```
src/storage/compression/
├── uncompressed/        # 无压缩
├── dictionary/          # 字典编码
├── dict_fsst/           # FSST 字符串压缩
├── bitpacking/          # 位打包 (整数)
├── rle/                 # 游程编码
├── alp/                 # ALP (浮点数)
├── alprd/               # ALPRD (浮点数)
├── chimp/               # Chimp (浮点数)
├── patas/               # Patas (浮点数)
└── roaring/             # Roaring 位图
```

### 6.2 压缩接口

```cpp
struct CompressionFunction {
    CompressionType type;

    // 分析数据选择压缩参数
    compression_analyze_t analyze;

    // 压缩数据
    compression_compress_t compress;

    // 扫描/解压数据
    compression_scan_t scan;
    compression_fetch_t fetch;

    // 初始化扫描状态
    compression_init_scan_t init_scan;
};
```

### 6.3 压缩选择流程

```
写入数据时:
┌──────────────────────────────────────────┐
│ 1. Analyze: 收集数据统计信息              │
│    - 最小/最大值                          │
│    - 基数 (cardinality)                   │
│    - NULL 比例                            │
├──────────────────────────────────────────┤
│ 2. 选择最优压缩算法                       │
│    - 低基数字符串 → Dictionary            │
│    - 高基数字符串 → FSST                  │
│    - 小范围整数 → Bitpacking              │
│    - 连续重复值 → RLE                     │
│    - 浮点数 → ALP/Chimp                   │
├──────────────────────────────────────────┤
│ 3. Compress: 执行压缩                     │
└──────────────────────────────────────────┘
```

## 七、事务与 MVCC

### 7.1 事务隔离模型

DuckDB 实现了快照隔离 (Snapshot Isolation)：

```cpp
class DuckTransaction {
    transaction_id_t transaction_id;  // 事务ID
    transaction_id_t start_time;      // 开始时间戳
    transaction_id_t commit_id;       // 提交时间戳
    unique_ptr<LocalStorage> storage; // 本地修改
};
```

### 7.2 LocalStorage

事务的本地修改存储在 LocalStorage 中：

```cpp
class LocalStorage {
    // 每个修改的表一个 LocalTableStorage
    unordered_map<DataTable *, LocalTableStorage> table_storage;
};

class LocalTableStorage {
    shared_ptr<RowGroupCollection> row_groups;  // 本地插入的行
    unique_ptr<TableIndexList> indexes;
    idx_t deleted_rows;
};
```

### 7.3 版本管理

```cpp
class RowVersionManager {
    // 行的版本信息
    // 记录哪些行被哪个事务插入/删除

    bool IsVisible(transaction_t transaction_id, row_t row);
};
```

**可见性判定：**

```
行 R 对事务 T 可见，当且仅当:
1. R.insert_id <= T.start_time (行已提交插入)
2. R.delete_id > T.start_time 或 R 未被删除 (行未被删除)
```

### 7.4 事务提交流程

```
BEGIN TRANSACTION
    ↓
本地修改 (写入 LocalStorage)
    ↓
COMMIT
    ├── 写入 WAL
    ├── 分配 commit_id
    ├── 合并 LocalStorage 到主存储
    └── 可能触发自动检查点
```

## 八、检查点机制

### 8.1 检查点目的

- 将内存中的修改持久化到数据文件
- 允许截断 WAL 日志
- 减少恢复时间

### 8.2 检查点流程

```
CreateCheckpoint():
┌──────────────────────────────────────────┐
│ 1. 遍历所有表                             │
├──────────────────────────────────────────┤
│ 2. 对每个表:                              │
│    - 获取 TableDataWriter                 │
│    - 遍历行组                             │
│    - 对每列选择压缩并写入                 │
│    - 记录元数据指针                       │
├──────────────────────────────────────────┤
│ 3. 写入元数据块                           │
│    - 目录信息                             │
│    - 行组指针                             │
│    - 统计信息                             │
├──────────────────────────────────────────┤
│ 4. 写入空闲列表                           │
├──────────────────────────────────────────┤
│ 5. fsync 数据                             │
├──────────────────────────────────────────┤
│ 6. 写入 DatabaseHeader                    │
├──────────────────────────────────────────┤
│ 7. fsync header                           │
├──────────────────────────────────────────┤
│ 8. 截断 WAL                               │
└──────────────────────────────────────────┘
```

### 8.3 增量检查点

DuckDB 支持增量检查点，只写入修改的行组：

```cpp
void RowGroup::Checkpoint(TableDataWriter &writer) {
    if (!HasChanges()) {
        // 行组未修改，复用现有指针
        writer.ReuseRowGroup(column_pointers);
        return;
    }

    // 写入修改的列
    for (auto &column : columns) {
        column->Checkpoint(writer);
    }
}
```

## 九、WAL (Write-Ahead Log)

### 9.1 WAL 作用

- 保证事务持久性 (Durability)
- 支持崩溃恢复
- 允许延迟检查点

### 9.2 WAL 条目类型

```cpp
enum class WALType : uint8_t {
    // DDL
    CREATE_TABLE, DROP_TABLE,
    CREATE_INDEX, DROP_INDEX,
    ALTER_INFO,

    // DML
    USE_TABLE,       // 设置当前表
    INSERT_TUPLE,    // 插入数据
    DELETE_TUPLE,    // 删除数据
    UPDATE_TUPLE,    // 更新数据
    ROW_GROUP_DATA,  // 批量行组数据

    // 控制
    WAL_VERSION,     // 版本头
    CHECKPOINT,      // 检查点标记
    WAL_FLUSH        // 事务边界
};
```

### 9.3 恢复流程

```
数据库启动:
┌──────────────────────────────────────────┐
│ 1. 加载 DatabaseHeader                    │
│    - 选择 iteration 更大的 header         │
├──────────────────────────────────────────┤
│ 2. 加载检查点数据                         │
│    - 读取元数据块                         │
│    - 重建目录结构                         │
├──────────────────────────────────────────┤
│ 3. 检查 WAL                               │
│    - 验证 checkpoint_iteration            │
├──────────────────────────────────────────┤
│ 4. 重放 WAL (如果需要)                    │
│    - 逐条重放操作                         │
│    - 遇到 WAL_FLUSH 提交事务              │
├──────────────────────────────────────────┤
│ 5. 数据库就绪                             │
└──────────────────────────────────────────┘
```

## 十、扫描与查询执行

### 10.1 表扫描初始化

```cpp
void DataTable::InitializeScan(TableScanState &state,
                               const vector<column_t> &column_ids,
                               TableFilterSet *filters) {
    // 初始化行组扫描
    row_groups->InitializeScan(state.row_group_state, column_ids, filters);

    // 初始化本地存储扫描 (事务修改)
    local_storage->InitializeScan(state.local_state, filters);
}
```

### 10.2 扫描流程

```
TableScan:
┌──────────────────────────────────────────┐
│ 1. 选择下一个行组                         │
├──────────────────────────────────────────┤
│ 2. Zonemap 过滤                           │
│    - 检查行组统计信息                     │
│    - 如果不满足过滤条件，跳过             │
├──────────────────────────────────────────┤
│ 3. 扫描列段                               │
│    - 解压数据到向量                       │
│    - 应用可见性过滤 (MVCC)                │
├──────────────────────────────────────────┤
│ 4. 返回 DataChunk (向量化结果)            │
└──────────────────────────────────────────┘
```

### 10.3 向量化执行

DuckDB 采用向量化执行模型：

```cpp
// 标准向量大小
#define STANDARD_VECTOR_SIZE 2048

class DataChunk {
    vector<Vector> data;    // 列向量
    idx_t count;            // 行数 (最大 2048)
};
```

**与存储的配合：**

```
ColumnSegment::Scan(state, result_vector, count):
┌──────────────────────────────────────────┐
│ 1. 获取压缩函数                           │
│ 2. 调用 compression_scan_t               │
│    - 解压 count 个值到 result_vector      │
│ 3. 处理 NULL 值                           │
│ 4. 更新扫描状态                           │
└──────────────────────────────────────────┘
```

## 十一、统计信息与优化

### 11.1 统计类型

```cpp
class BaseStatistics {
    bool has_null;
    bool has_no_null;
};

class NumericStatistics : public BaseStatistics {
    Value min, max;
};

class StringStatistics : public BaseStatistics {
    string_t min, max;
    bool has_unicode;
    idx_t max_string_length;
};
```

### 11.2 Zonemap 过滤

每个行组和列段都维护统计信息：

```cpp
// 检查行组是否可以被跳过
FilterResult RowGroup::CheckZonemap(TableFilterSet &filters) {
    for (auto &filter : filters) {
        auto &stats = columns[filter.column_id]->stats;
        if (stats.CheckFilter(filter) == FilterResult::NO_MATCH) {
            return FilterResult::NO_MATCH;  // 跳过整个行组
        }
    }
    return FilterResult::MAY_HAVE_MATCHES;
}
```

## 十二、索引支持

### 12.1 ART 索引

DuckDB 主要使用自适应基数树 (Adaptive Radix Tree, ART)：

```cpp
class ART : public BoundIndex {
    // 支持的约束类型
    IndexConstraintType constraint_type;  // PRIMARY / UNIQUE / FOREIGN

    // 核心操作
    ErrorData Insert(DataChunk &data, Vector &row_ids);
    void Delete(DataChunk &data, Vector &row_ids);
    bool SearchEqual(Key key, idx_t &result_id);
};
```

### 12.2 索引存储

索引数据也存储在块中：

```cpp
struct IndexStorageInfo {
    string name;
    vector<BlockAllocatorInfo> allocator_infos;
    vector<vector<BlockInfo>> buffers;
    case_insensitive_map_t<Value> options;
};
```

## 十三、扩展机制

### 13.1 扩展类型

DuckDB 支持通过扩展添加新功能：

- **存储扩展**: Parquet, CSV, JSON 等格式
- **文件系统扩展**: S3, HTTP, Azure 等
- **函数扩展**: 自定义函数

### 13.2 TableFunction 接口

外部数据源通过 TableFunction 接入：

```cpp
struct TableFunction {
    table_function_bind_t bind;
    table_function_init_global_t init_global;
    table_function_init_local_t init_local;
    table_function_t function;  // 实际扫描函数

    bool projection_pushdown;
    bool filter_pushdown;
};
```

## 十四、性能特性

### 14.1 并行执行

```
并行表扫描:
┌─────────────────────────────────────────────────────────────┐
│  Thread 0        Thread 1        Thread 2        Thread 3   │
│     ↓               ↓               ↓               ↓       │
│  RowGroup 0     RowGroup 1     RowGroup 2     RowGroup 3   │
│     ↓               ↓               ↓               ↓       │
│  DataChunk      DataChunk      DataChunk      DataChunk    │
└─────────────────────────────────────────────────────────────┘
```

### 14.2 零拷贝设计

```
磁盘块 → BlockHandle (Pin) → 列扫描 (原位解压) → 结果向量
           ↑                                         ↓
           └────────── 共享同一块内存 ──────────────┘
```

### 14.3 延迟物化

```cpp
// 只读取需要的列
void DataTable::Scan(TableScanState &state,
                     const vector<column_t> &column_ids,  // 投影列
                     DataChunk &result);
```

## 十五、架构总结

### 核心设计原则

| 原则 | 实现 |
|------|------|
| **列式存储** | RowGroup + ColumnData + ColumnSegment |
| **向量化执行** | 2048 行为单位批处理 |
| **压缩优先** | 自动选择最优压缩算法 |
| **内存效率** | BufferManager + LRU 驱逐 |
| **事务支持** | MVCC + LocalStorage |
| **持久化** | 双 Header + WAL + 检查点 |
| **并行友好** | 行组级别并行 |

### 源码目录结构

```
src/storage/
├── storage_manager.cpp          # 存储管理器
├── single_file_block_manager.cpp # 单文件块管理
├── standard_buffer_manager.cpp   # 缓冲管理
├── checkpoint_manager.cpp        # 检查点
├── write_ahead_log.cpp          # WAL 写入
├── wal_replay.cpp               # WAL 重放
├── table/
│   ├── data_table.cpp           # 数据表
│   ├── row_group.cpp            # 行组
│   ├── row_group_collection.cpp # 行组集合
│   ├── column_data.cpp          # 列数据
│   ├── column_segment.cpp       # 列段
│   └── update_segment.cpp       # 更新段
├── compression/                 # 压缩算法
├── buffer/                      # 缓冲组件
├── statistics/                  # 统计信息
└── metadata/                    # 元数据管理
```

DuckDB 存储引擎通过精心设计的分层架构，在保持代码清晰的同时实现了高性能的分析型数据库存储。其列式存储、向量化执行和智能压缩的组合，使其在 OLAP 场景下表现出色，同时嵌入式的设计又保证了部署的简便性。

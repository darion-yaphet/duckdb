# DuckDB 存储引擎深度解析：系列目录

## 系列概述

本系列深入剖析 DuckDB 的存储引擎，涵盖从内存管理到持久化存储的完整技术栈。DuckDB 采用单文件存储架构，结合列式存储、高效压缩和 MVCC 事务支持，实现了高性能的分析型数据库存储层。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 存储引擎架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         应用层 / 执行引擎                              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          第一章：存储架构                              │  │
│  │  StorageManager │ DataTable │ TableIOManager │ 单文件架构              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│          ┌─────────────────────────┼─────────────────────────┐              │
│          ▼                         ▼                         ▼              │
│  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │ 第二章：内存管理 │    │ 第三章：数据组织     │    │ 第四章：压缩算法    │  │
│  │ BufferManager  │    │ RowGroup          │    │ BitPacking        │  │
│  │ BufferPool     │    │ ColumnData        │    │ Dictionary/FSST   │  │
│  │ BlockHandle    │    │ ColumnSegment     │    │ ALP/Chimp/Patas   │  │
│  │ 临时文件管理    │    │ SegmentTree       │    │ RLE/Roaring       │  │
│  └─────────────────┘    └─────────────────────┘    └─────────────────────┘  │
│                                    │                                        │
│          ┌─────────────────────────┼─────────────────────────┐              │
│          ▼                         ▼                         ▼              │
│  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │ 第五章：统计信息 │    │ 第六章：持久化      │    │ 第七章：索引系统    │  │
│  │ BaseStatistics │    │ CheckpointManager │    │ ART 索引          │  │
│  │ NumericStats   │    │ WAL 预写日志       │    │ 索引接口设计       │  │
│  │ DistinctStats  │    │ MetadataManager   │    │ 约束索引          │  │
│  │ 选择性估计      │    │ 增量检查点         │    │ 索引持久化        │  │
│  └─────────────────┘    └─────────────────────┘    └─────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          物理存储层                                    │  │
│  │  SingleFileBlockManager │ Block │ 文件头 │ 空闲列表 │ 加密支持          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第一章：存储架构总览

### 1.1 存储系统设计理念
- 单文件存储架构 (Single-File Storage)
- 列式存储优势
- 面向分析型工作负载的设计决策
- 与 SQLite 的架构对比

### 1.2 核心组件概览
- `StorageManager`：存储管理器入口
- `DataTable`：数据表抽象
- `TableIOManager`：表 I/O 管理
- `SingleFileBlockManager`：块管理器

### 1.3 存储常量与配置
- Block 大小：`DEFAULT_BLOCK_ALLOC_SIZE` (256KB)
- RowGroup 大小：`DEFAULT_ROW_GROUP_SIZE` (122880 行)
- Vector 大小：`STANDARD_VECTOR_SIZE` (2048)
- 配置参数调优

### 1.4 文件格式设计
- 文件头结构 (`MainHeader`, `DatabaseHeader`)
- 魔数与版本控制
- 双头设计与原子切换
- 加密支持

### 1.5 存储扩展接口
- `StorageExtension` 扩展点
- 自定义存储后端
- 远程存储集成

---

## 第二章：内存管理与缓冲池

### 2.1 BufferManager 架构
- `BufferManager` 接口设计
- `StandardBufferManager` 实现
- 内存预算管理
- 多实例内存隔离

### 2.2 BufferPool 设计
- LRU 淘汰策略
- 并发控制与锁设计
- 内存压力响应
- 脏页管理

### 2.3 BlockHandle 与 BufferHandle
- `BlockHandle`：块元数据
- `BufferHandle`：RAII 内存访问
- Pin/Unpin 机制
- 引用计数管理

### 2.4 Block 与 BlockAllocator
- `Block` 数据结构
- 块 ID 分配策略
- `PartialBlockManager`：部分块管理
- 内存对齐与访问优化

### 2.5 临时文件管理
- `TemporaryFileManager`：临时文件生命周期
- 溢出策略
- 临时内存管理器 (`TemporaryMemoryManager`)
- 压缩临时数据

### 2.6 Arena 分配器
- `ArenaAllocator` 设计
- 批量分配优化
- 生命周期管理

---

## 第三章：数据组织与存储结构

### 3.1 DataTable 核心设计
- 表级元数据
- 列定义管理
- 版本控制 (Main/Altered/Dropped)
- 表级锁策略

### 3.2 RowGroupCollection
- RowGroup 集合管理
- 追加与合并
- 并行扫描支持
- 重排序优化

### 3.3 RowGroup 详解
- RowGroup 结构
- 列数据组织
- 版本信息 (`ChunkInfo`)
- MVCC 可见性

### 3.4 ColumnData 层次结构
- `ColumnData` 基类设计
- `StandardColumnData`：基础类型
- `ValidityColumnData`：NULL 位图
- `StructColumnData`：结构体类型
- `ListColumnData`：列表类型
- `ArrayColumnData`：定长数组
- `VariantColumnData`：动态类型

### 3.5 ColumnSegment 设计
- Segment 生命周期
- 压缩状态管理
- Transient vs Persistent
- 段合并策略

### 3.6 SegmentTree
- `ColumnSegmentTree`：列段索引
- `RowGroupSegmentTree`：RowGroup 索引
- 高效区间查询
- 并发安全

### 3.7 扫描状态管理
- `TableScanState`：表扫描状态
- `ColumnScanState`：列扫描状态
- 并行扫描 (`ParallelTableScanState`)
- 过滤器下推

### 3.8 更新与删除处理
- `UpdateSegment`：更新段
- `RowVersionManager`：行版本管理
- 删除标记 (`ChunkVectorInfo`)
- 版本链维护

---

## 第四章：压缩算法深度解析

### 4.1 压缩框架设计
- `CompressionFunction` 接口
- 压缩/解压缩流程
- 自动压缩选择
- 分析阶段 (Analyze)

### 4.2 通用压缩算法
- **Uncompressed**：无压缩基线
- **Constant**：常量压缩
- **RLE (Run-Length Encoding)**：游程编码

### 4.3 整数压缩
- **BitPacking**：位压缩
  - Frame of Reference (FOR)
  - Delta 编码
  - 位宽优化
- **BitPacking Hugeint**：大整数支持

### 4.4 字符串压缩
- **Dictionary Compression**：字典压缩
  - 字典构建
  - 索引编码
  - 溢出处理
- **FSST (Fast Static Symbol Table)**
  - 符号表构建
  - 高压缩比场景
- **Dict+FSST**：混合压缩

### 4.5 浮点数压缩
- **ALP (Adaptive Lossless floating-Point)**
  - 指数因子选择
  - 异常值处理
  - 无损保证
- **ALPRD (ALP Real-Double)**
  - 扩展精度支持
- **Chimp**
  - XOR 压缩
  - 前导零优化
- **Patas**
  - 时序数据优化

### 4.6 位图压缩
- **Roaring Bitmap**
  - 容器类型选择
  - 运行压缩
  - 位图压缩
- **Empty Validity**：全非空优化

### 4.7 外部压缩
- **Zstd 集成**
  - 压缩级别选择
  - 字典训练

### 4.8 压缩选择策略
- 采样分析
- 压缩比估计
- CPU/空间权衡
- 类型特化选择

---

## 第五章：统计信息系统

### 5.1 统计信息架构
- `BaseStatistics` 基类
- 统计信息类型层次
- 统计收集时机
- 持久化与加载

### 5.2 数值统计 (NumericStats)
- Min/Max 追踪
- NULL 计数
- 范围统计
- 数值类型特化

### 5.3 字符串统计 (StringStats)
- 最小/最大值
- 长度统计
- 前缀统计
- Unicode 处理

### 5.4 复合类型统计
- `StructStats`：结构体统计
- `ListStats`：列表统计
- `ArrayStats`：数组统计
- `VariantStats`：动态类型统计

### 5.5 Distinct 统计
- `DistinctStatistics`：基数估计
- HyperLogLog 实现
- 合并策略
- 精度控制

### 5.6 Segment 统计
- `SegmentStatistics`：段级统计
- Zonemap 设计
- 过滤器加速
- 统计传播

### 5.7 表级统计 (TableStatistics)
- 列统计聚合
- 行数追踪
- 采样统计
- 统计更新策略

### 5.8 统计信息应用
- 查询优化中的应用
- 基数估计
- Join 顺序优化
- 过滤器选择性

---

## 第六章：持久化与恢复

### 6.1 检查点机制
- `CheckpointManager` 设计
- 检查点触发条件
- 增量检查点
- 并发检查点

### 6.2 表数据写入
- `TableDataWriter`：表写入器
- `RowGroupWriter`：RowGroup 写入
- 列数据序列化
- 溢出字符串处理

### 6.3 表数据读取
- `TableDataReader`：表读取器
- 延迟加载策略
- 按需解压缩
- 预读优化

### 6.4 元数据管理
- `MetadataManager`：元数据管理
- `MetadataWriter/Reader`：元数据 I/O
- Catalog 持久化
- Schema 版本控制

### 6.5 Write-Ahead Log (WAL)
- WAL 设计原理
- 日志记录格式
- 刷盘策略
- 事务边界

### 6.6 WAL 回放
- 崩溃恢复流程
- 日志重放逻辑
- 幂等性保证
- 错误处理

### 6.7 乐观写入
- `OptimisticDataWriter`：乐观写入器
- 并行追加优化
- 冲突检测
- 回滚处理

### 6.8 块管理
- `SingleFileBlockManager`：单文件块管理
- `InMemoryBlockManager`：内存块管理
- 空闲列表管理
- 块复用策略

---

## 第七章：索引系统

### 7.1 索引架构概述
- `Index` 基类设计
- `BoundIndex` vs `UnboundIndex`
- 索引生命周期
- 索引类型注册

### 7.2 ART 索引深度解析
- Adaptive Radix Tree 原理
- 节点类型：
  - `Node4/16/48/256`：内部节点
  - `Leaf`：叶节点
  - `Node256Leaf`：混合节点
- 前缀压缩 (`Prefix`)
- 路径压缩

### 7.3 ART 操作实现
- 查找算法
- 插入与节点分裂
- 删除与节点收缩
- 范围扫描 (`Iterator`)

### 7.4 ART 构建器
- `ARTBuilder`：批量构建
- 排序构建优化
- 内存效率

### 7.5 索引持久化
- 索引存储格式
- `FixedSizeAllocator`：块分配
- 延迟持久化
- 索引重建

### 7.6 约束索引
- PRIMARY KEY 索引
- UNIQUE 索引
- FOREIGN KEY 索引
- 约束验证

### 7.7 索引扫描
- `ARTScanner`：索引扫描器
- 点查询优化
- 范围查询
- 前缀匹配

### 7.8 索引维护
- 并发索引更新
- 索引 Vacuum
- 索引合并 (`ARTMerger`)
- 索引统计

---

## 附录

### A. 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| 存储管理 | `src/storage/storage_manager.cpp` |
| 数据表 | `src/storage/data_table.cpp`, `src/include/duckdb/storage/data_table.hpp` |
| Buffer 管理 | `src/storage/buffer/`, `src/storage/buffer_manager.cpp` |
| RowGroup | `src/storage/table/row_group.cpp`, `row_group_collection.cpp` |
| 列数据 | `src/storage/table/column_data.cpp`, `column_segment.cpp` |
| 压缩 | `src/storage/compression/` (35+ 文件) |
| 统计 | `src/storage/statistics/` (11 文件) |
| 检查点 | `src/storage/checkpoint/`, `checkpoint_manager.cpp` |
| WAL | `src/storage/write_ahead_log.cpp`, `wal_replay.cpp` |
| 元数据 | `src/storage/metadata/` |
| ART 索引 | `src/execution/index/art/` (20+ 文件) |

### B. 存储常量速查

| 常量 | 默认值 | 说明 |
|------|--------|------|
| `DEFAULT_BLOCK_ALLOC_SIZE` | 256 KB | 块分配大小 |
| `DEFAULT_ROW_GROUP_SIZE` | 122,880 | 每 RowGroup 行数 |
| `STANDARD_VECTOR_SIZE` | 2,048 | 向量大小 |
| `FILE_HEADER_SIZE` | 4 KB | 文件头大小 |
| `SECTOR_SIZE` | 4 KB | 扇区大小 |

### C. 存储层类图

```
StorageManager
     │
     ├── DataTable
     │       │
     │       ├── RowGroupCollection
     │       │       │
     │       │       └── RowGroup[]
     │       │               │
     │       │               ├── ColumnData[]
     │       │               │       │
     │       │               │       └── ColumnSegment[]
     │       │               │
     │       │               └── RowVersionManager
     │       │
     │       └── TableIndexList
     │               │
     │               └── Index[] (ART)
     │
     ├── BufferManager
     │       │
     │       └── BufferPool
     │               │
     │               └── BlockHandle[]
     │
     ├── CheckpointManager
     │       │
     │       ├── TableDataWriter
     │       └── TableDataReader
     │
     └── WriteAheadLog
```

### D. 压缩算法对比

| 算法 | 适用类型 | 压缩比 | 解压速度 | 特点 |
|------|----------|--------|----------|------|
| BitPacking | 整数 | 高 | 极快 | 位级紧凑 |
| Dictionary | 字符串 | 高 | 快 | 重复值多 |
| FSST | 字符串 | 极高 | 中 | 符号表压缩 |
| RLE | 任意 | 中 | 极快 | 连续重复 |
| ALP | 浮点 | 高 | 快 | 无损浮点 |
| Chimp | 浮点 | 高 | 快 | 时序优化 |
| Roaring | 位图 | 高 | 极快 | 稀疏位图 |
| Zstd | 任意 | 极高 | 中 | 通用压缩 |

### E. 数据写入流程

```
INSERT INTO table VALUES (...)
         │
         ▼
   LocalStorage (事务本地)
         │
         ▼ COMMIT
   RowGroupCollection.Append()
         │
         ├── 创建新 RowGroup (如需要)
         │
         ├── ColumnData.Append()
         │       │
         │       └── ColumnSegment.Append()
         │
         └── 更新统计信息
         │
         ▼
   WAL.WriteEntry()
         │
         ▼ (达到阈值或显式调用)
   CheckpointManager.Checkpoint()
         │
         ├── 压缩 ColumnSegment
         │
         ├── 写入 Block
         │
         └── 更新 DatabaseHeader
```

---

## 写作计划

| 章节 | 预计篇幅 | 核心内容 |
|------|----------|----------|
| 第一章 | ~40KB | 存储架构、文件格式、配置 |
| 第二章 | ~50KB | BufferManager、BufferPool、内存管理 |
| 第三章 | ~60KB | DataTable、RowGroup、ColumnData、Segment |
| 第四章 | ~70KB | 8+ 压缩算法详解 |
| 第五章 | ~35KB | 统计信息系统 |
| 第六章 | ~50KB | 检查点、WAL、恢复 |
| 第七章 | ~45KB | ART 索引、约束索引 |

---

## 与其他系列的关联

- **事务系统**：MVCC 可见性与 RowVersionManager 紧密配合
- **执行引擎**：TableScan 直接使用存储层的扫描接口
- **查询处理**：统计信息驱动优化器决策

# DuckDB 执行引擎深度解析（四）：物理算子体系

## 引言

在前三章中，我们分析了 DuckDB 的执行模型概述、向量化数据结构和表达式执行器。本章将深入探讨 **物理算子（Physical Operators）** 的设计与实现。

物理算子是执行计划的基本构建块，它们定义了数据的实际处理方式。DuckDB 的物理算子体系采用了精心设计的三接口模式（Source/Operator/Sink），支持高效的流水线执行和并行处理。

## 1. PhysicalOperator 基类架构

### 1.1 类设计概览

`PhysicalOperator` 是所有物理算子的抽象基类，定义了算子在执行引擎中的统一接口：

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class PhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::INVALID;

    PhysicalOperator(PhysicalPlan &physical_plan, PhysicalOperatorType type,
                     vector<LogicalType> types, idx_t estimated_cardinality);

    //! 子算子列表
    ArenaLinkedList<reference<PhysicalOperator>> children;
    //! 算子类型
    PhysicalOperatorType type;
    //! 输出列类型
    vector<LogicalType> types;
    //! 估计的输出基数
    idx_t estimated_cardinality;

    //! 全局 Sink 状态
    unique_ptr<GlobalSinkState> sink_state;
    //! 全局 Operator 状态
    unique_ptr<GlobalOperatorState> op_state;
    //! 状态访问锁
    mutex lock;
};
```

### 1.2 核心属性详解

| 属性 | 类型 | 说明 |
|------|------|------|
| `type` | `PhysicalOperatorType` | 算子类型枚举，用于识别和调试 |
| `types` | `vector<LogicalType>` | 输出数据的列类型向量 |
| `estimated_cardinality` | `idx_t` | 优化器估计的输出行数 |
| `children` | `ArenaLinkedList` | 子算子引用链表 |
| `sink_state` | `unique_ptr<GlobalSinkState>` | Sink 操作的全局状态 |
| `op_state` | `unique_ptr<GlobalOperatorState>` | Operator 操作的全局状态 |

### 1.3 算子类型枚举

DuckDB 支持丰富的物理算子类型：

```cpp
// src/include/duckdb/common/enums/physical_operator_type.hpp

enum class PhysicalOperatorType : uint8_t {
    INVALID,
    // 数据处理算子
    ORDER_BY,
    LIMIT,
    TOP_N,
    WINDOW,
    FILTER,
    PROJECTION,

    // 聚合算子
    UNGROUPED_AGGREGATE,
    HASH_GROUP_BY,
    PERFECT_HASH_GROUP_BY,
    PARTITIONED_AGGREGATE,

    // 扫描算子
    TABLE_SCAN,
    COLUMN_DATA_SCAN,
    EXPRESSION_SCAN,

    // 连接算子
    HASH_JOIN,
    NESTED_LOOP_JOIN,
    CROSS_PRODUCT,
    PIECEWISE_MERGE_JOIN,
    IE_JOIN,
    ASOF_JOIN,

    // DML 算子
    INSERT,
    DELETE_OPERATOR,
    UPDATE,

    // 其他...
};
```

## 2. 三接口执行模式

DuckDB 的物理算子体系采用三种接口模式，每种接口对应不同的数据流动方式：

```
┌─────────────────────────────────────────────────────────────┐
│                    三接口执行模式                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐                                          │
│   │    Source    │ ← 数据生成者，Pipeline 起点               │
│   │  IsSource()  │   例：TableScan, ChunkScan               │
│   └──────┬───────┘                                          │
│          │ GetData()                                        │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │   Operator   │ ← 数据转换者，Pipeline 中间节点           │
│   │  ParallelOp  │   例：Filter, Projection                 │
│   └──────┬───────┘                                          │
│          │ Execute()                                        │
│          ▼                                                  │
│   ┌──────────────┐                                          │
│   │     Sink     │ ← 数据消费者，Pipeline 终点               │
│   │   IsSink()   │   例：HashJoin Build, Aggregate          │
│   └──────────────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 Source 接口

Source 接口用于产生数据的算子，通常是 Pipeline 的起点：

```cpp
class PhysicalOperator {
public:
    // Source 接口
    virtual unique_ptr<LocalSourceState> GetLocalSourceState(
        ExecutionContext &context, GlobalSourceState &gstate) const;
    virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(
        ClientContext &context) const;

    // 核心数据获取方法
    SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                             OperatorSourceInput &input) const;

protected:
    // 子类实现的内部方法
    virtual SourceResultType GetDataInternal(ExecutionContext &context,
                                              DataChunk &chunk,
                                              OperatorSourceInput &input) const;

public:
    virtual bool IsSource() const { return false; }
    virtual bool ParallelSource() const { return false; }
    virtual ProgressData GetProgress(ClientContext &context,
                                     GlobalSourceState &gstate) const;
};
```

**Source 返回类型：**

```cpp
enum class SourceResultType : uint8_t {
    HAVE_MORE_OUTPUT,  // 还有更多数据
    FINISHED,          // 数据已读完
    BLOCKED            // 等待异步操作
};
```

### 2.2 Operator 接口

Operator 接口用于转换数据的算子，接收输入并产生输出：

```cpp
class PhysicalOperator {
public:
    // Operator 接口
    virtual unique_ptr<OperatorState> GetOperatorState(
        ExecutionContext &context) const;
    virtual unique_ptr<GlobalOperatorState> GetGlobalOperatorState(
        ClientContext &context) const;

    // 核心执行方法
    virtual OperatorResultType Execute(ExecutionContext &context,
                                       DataChunk &input,
                                       DataChunk &chunk,
                                       GlobalOperatorState &gstate,
                                       OperatorState &state) const;

    // 最终执行（处理缓存的数据）
    virtual OperatorFinalizeResultType FinalExecute(ExecutionContext &context,
                                                    DataChunk &chunk,
                                                    GlobalOperatorState &gstate,
                                                    OperatorState &state) const;

    virtual bool ParallelOperator() const { return false; }
    virtual bool RequiresFinalExecute() const { return false; }
};
```

**Operator 返回类型：**

```cpp
enum class OperatorResultType : uint8_t {
    NEED_MORE_INPUT,    // 需要更多输入数据
    HAVE_MORE_OUTPUT,   // 当前输入还能产生更多输出
    FINISHED,           // 算子执行完成
    BLOCKED             // 等待异步操作
};
```

### 2.3 Sink 接口

Sink 接口用于消费数据的算子，通常是 Pipeline 的终点：

```cpp
class PhysicalOperator {
public:
    // Sink 接口
    virtual unique_ptr<LocalSinkState> GetLocalSinkState(
        ExecutionContext &context) const;
    virtual unique_ptr<GlobalSinkState> GetGlobalSinkState(
        ClientContext &context) const;

    // 核心 Sink 方法
    virtual SinkResultType Sink(ExecutionContext &context,
                                DataChunk &chunk,
                                OperatorSinkInput &input) const;

    // 线程完成时调用
    virtual SinkCombineResultType Combine(ExecutionContext &context,
                                          OperatorSinkCombineInput &input) const;

    // Finalize 前的准备
    virtual void PrepareFinalize(ClientContext &context,
                                 GlobalSinkState &sink_state) const;

    // 所有线程完成后调用（单线程）
    virtual SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                                      ClientContext &context,
                                      OperatorSinkFinalizeInput &input) const;

    virtual bool IsSink() const { return false; }
    virtual bool ParallelSink() const { return false; }
};
```

**Sink 生命周期：**

```
┌─────────────────────────────────────────────────────────────┐
│                    Sink 执行生命周期                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GetGlobalSinkState()                                       │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────────────────────────┐                   │
│  │   Thread 1      Thread 2      Thread N│                  │
│  │      │             │             │    │                  │
│  │      ▼             ▼             ▼    │                  │
│  │ GetLocalSink  GetLocalSink  GetLocalSink                 │
│  │      │             │             │    │                  │
│  │      ▼             ▼             ▼    │                  │
│  │  Sink() ×N     Sink() ×N     Sink() ×N │  ← 并行        │
│  │      │             │             │    │                  │
│  │      ▼             ▼             ▼    │                  │
│  │  Combine()    Combine()     Combine() │  ← 线程结束     │
│  └──────┬─────────────┬─────────────┬────┘                  │
│         └─────────────┼─────────────┘                       │
│                       ▼                                     │
│               PrepareFinalize()                             │
│                       │                                     │
│                       ▼                                     │
│                 Finalize()          ← 单线程                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 状态管理系统

### 3.1 状态类层次结构

DuckDB 为不同接口定义了专门的状态类：

```cpp
// src/include/duckdb/execution/physical_operator_states.hpp

// Operator 接口的状态
class OperatorState {
public:
    virtual ~OperatorState() {}
    virtual void Finalize(const PhysicalOperator &op,
                         ExecutionContext &context) {}

    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }
};

// Operator 全局状态
class GlobalOperatorState {
public:
    virtual ~GlobalOperatorState() {}
    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }
};

// Sink 全局状态
class GlobalSinkState : public StateWithBlockableTasks {
public:
    GlobalSinkState() : state(SinkFinalizeType::READY) {}
    SinkFinalizeType state;
    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }
};

// Sink 线程本地状态
class LocalSinkState {
public:
    virtual ~LocalSinkState() {}
    SourcePartitionInfo partition_info;  // 分区信息
};

// Source 全局状态
class GlobalSourceState : public StateWithBlockableTasks {
public:
    virtual ~GlobalSourceState() {}
    virtual idx_t MaxThreads() { return 1; }
};

// Source 线程本地状态
class LocalSourceState {
public:
    virtual ~LocalSourceState() {}
};
```

### 3.2 输入结构体

为简化参数传递，DuckDB 定义了统一的输入结构：

```cpp
struct OperatorSinkInput {
    GlobalSinkState &global_state;
    LocalSinkState &local_state;
    InterruptState &interrupt_state;
};

struct OperatorSourceInput {
    GlobalSourceState &global_state;
    LocalSourceState &local_state;
    InterruptState &interrupt_state;
};

struct OperatorSinkCombineInput {
    GlobalSinkState &global_state;
    LocalSinkState &local_state;
    InterruptState &interrupt_state;
};

struct OperatorSinkFinalizeInput {
    GlobalSinkState &global_state;
    InterruptState &interrupt_state;
};
```

### 3.3 状态的线程安全设计

```
┌─────────────────────────────────────────────────────────────┐
│                    状态的线程安全模型                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   GlobalState                                               │
│   ┌─────────────────────────────────────────────┐           │
│   │  • 所有线程共享                              │           │
│   │  • 访问需要加锁或使用原子操作                │           │
│   │  • 存储聚合结果、哈希表等                    │           │
│   └─────────────────────────────────────────────┘           │
│                         │                                   │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│   LocalState 1    LocalState 2    LocalState N              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│   │ 线程私有 │   │ 线程私有 │   │ 线程私有 │               │
│   │ 无需加锁 │   │ 无需加锁 │   │ 无需加锁 │               │
│   │ 本地缓冲 │   │ 本地缓冲 │   │ 本地缓冲 │               │
│   └──────────┘   └──────────┘   └──────────┘               │
│                                                             │
│   Combine 阶段: LocalState → GlobalState (需要锁)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 核心算子实现分析

### 4.1 TableScan (Source 算子)

`PhysicalTableScan` 是最基本的 Source 算子，从存储层读取数据：

```cpp
// src/include/duckdb/execution/operator/scan/physical_table_scan.hpp

class PhysicalTableScan : public PhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE =
        PhysicalOperatorType::TABLE_SCAN;

    //! 表函数（统一的扫描接口）
    TableFunction function;
    //! 函数绑定数据
    unique_ptr<FunctionData> bind_data;
    //! 返回的列类型
    vector<LogicalType> returned_types;
    //! 需要扫描的列 ID
    vector<ColumnIndex> column_ids;
    //! 投影列 ID（进一步过滤）
    vector<idx_t> projection_ids;
    //! 下推的过滤条件
    unique_ptr<TableFilterSet> table_filters;
    //! 动态过滤器（如来自 Join 的布隆过滤器）
    shared_ptr<DynamicTableFilterSet> dynamic_filters;

public:
    bool IsSource() const override { return true; }
    bool ParallelSource() const override;  // 取决于表函数

    SourceResultType GetDataInternal(ExecutionContext &context,
                                     DataChunk &chunk,
                                     OperatorSourceInput &input) const override;
};
```

**TableScan 的关键特性：**

1. **列裁剪**：只扫描 `column_ids` 指定的列
2. **过滤下推**：`table_filters` 在扫描时过滤数据
3. **并行扫描**：支持多线程并行读取数据
4. **动态过滤**：运行时接收来自 Join 的过滤器

### 4.2 Filter (Operator 算子)

`PhysicalFilter` 是最简单的 Operator 算子：

```cpp
// src/include/duckdb/execution/operator/filter/physical_filter.hpp

class PhysicalFilter : public CachingPhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE =
        PhysicalOperatorType::FILTER;

    unique_ptr<Expression> expression;

public:
    bool ParallelOperator() const override { return true; }

protected:
    OperatorResultType ExecuteInternal(ExecutionContext &context,
                                       DataChunk &input,
                                       DataChunk &chunk,
                                       GlobalOperatorState &gstate,
                                       OperatorState &state) const override;
};
```

**Filter 实现：**

```cpp
// src/execution/operator/filter/physical_filter.cpp

class FilterState : public CachingOperatorState {
public:
    explicit FilterState(ExecutionContext &context,
                        const PhysicalFilter &op)
        : executor(context.client, op.expression.get()) {}

    ExpressionExecutor executor;
    SelectionVector sel;
};

OperatorResultType PhysicalFilter::ExecuteInternal(
    ExecutionContext &context, DataChunk &input, DataChunk &chunk,
    GlobalOperatorState &gstate, OperatorState &state_p) const {

    auto &state = state_p.Cast<FilterState>();

    // 使用 ExpressionExecutor 的 SelectExpression 模式
    idx_t result_count = state.executor.SelectExpression(input, state.sel);

    if (result_count == input.size()) {
        // 所有行都匹配，直接引用输入
        chunk.Reference(input);
    } else {
        // 使用 SelectionVector 切片
        chunk.Slice(input, state.sel, result_count);
    }

    return OperatorResultType::NEED_MORE_INPUT;
}
```

**Filter 工作流程：**

```
┌─────────────────────────────────────────────────────────────┐
│                    Filter 执行流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入 DataChunk (2048 行)                                   │
│   ┌────────────────────────────────────────────┐            │
│   │  a  │  b  │  c  │ ... │                    │            │
│   └────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│   ┌────────────────────────────────────────────┐            │
│   │ executor.SelectExpression(input, sel)      │            │
│   │ 条件: a > 10                               │            │
│   └────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│   SelectionVector sel = [3, 7, 15, 42, ...]                 │
│   result_count = 847                                        │
│                       │                                     │
│                       ▼                                     │
│   ┌────────────────────────────────────────────┐            │
│   │ chunk.Slice(input, sel, result_count)      │            │
│   └────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│   输出 DataChunk (847 行, 零拷贝)                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Projection (Operator 算子)

`PhysicalProjection` 计算表达式列表，产生新的列：

```cpp
// src/include/duckdb/execution/operator/projection/physical_projection.hpp

class PhysicalProjection : public PhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE =
        PhysicalOperatorType::PROJECTION;

    vector<unique_ptr<Expression>> select_list;

public:
    bool ParallelOperator() const override { return true; }

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input, DataChunk &chunk,
                               GlobalOperatorState &gstate,
                               OperatorState &state) const override;
};
```

**Projection 实现：**

```cpp
// src/execution/operator/projection/physical_projection.cpp

class ProjectionState : public OperatorState {
public:
    explicit ProjectionState(ExecutionContext &context,
                            const vector<unique_ptr<Expression>> &expressions)
        : executor(context.client, expressions) {}

    ExpressionExecutor executor;
};

OperatorResultType PhysicalProjection::Execute(
    ExecutionContext &context, DataChunk &input, DataChunk &chunk,
    GlobalOperatorState &gstate, OperatorState &state_p) const {

    auto &state = state_p.Cast<ProjectionState>();

    // 使用 Execute 模式计算所有表达式
    state.executor.Execute(input, chunk);

    return OperatorResultType::NEED_MORE_INPUT;
}
```

### 4.4 Hash Join (复合算子)

`PhysicalHashJoin` 是最复杂的算子之一，同时实现了三种接口：

```cpp
// src/include/duckdb/execution/operator/join/physical_hash_join.hpp

class PhysicalHashJoin : public PhysicalComparisonJoin {
public:
    static constexpr const PhysicalOperatorType TYPE =
        PhysicalOperatorType::HASH_JOIN;

    //! Join 键类型
    vector<LogicalType> condition_types;
    //! 输出列配置
    JoinProjectionColumns payload_columns;
    JoinProjectionColumns lhs_output_columns;
    JoinProjectionColumns rhs_output_columns;

public:
    // Operator 接口 (Probe 阶段)
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const override;
    bool ParallelOperator() const override { return true; }

protected:
    OperatorResultType ExecuteInternal(ExecutionContext &context,
                                       DataChunk &input, DataChunk &chunk,
                                       GlobalOperatorState &gstate,
                                       OperatorState &state) const override;

public:
    // Source 接口 (Full/Right Outer Join 扫描)
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const override;
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                     GlobalSourceState &gstate) const override;
    SourceResultType GetDataInternal(ExecutionContext &context, DataChunk &chunk,
                                     OperatorSourceInput &input) const override;
    bool IsSource() const override { return true; }
    bool ParallelSource() const override { return true; }

public:
    // Sink 接口 (Build 阶段)
    unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const override;
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const override;
    SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                        OperatorSinkInput &input) const override;
    SinkCombineResultType Combine(ExecutionContext &context,
                                  OperatorSinkCombineInput &input) const override;
    void PrepareFinalize(ClientContext &context, GlobalSinkState &global_state) const override;
    SinkFinalizeType Finalize(Pipeline &pipeline, Event &event, ClientContext &context,
                              OperatorSinkFinalizeInput &input) const override;
    bool IsSink() const override { return true; }
    bool ParallelSink() const override { return true; }
};
```

**Hash Join 执行流程：**

```
┌─────────────────────────────────────────────────────────────┐
│                   Hash Join 执行流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Build Pipeline (Sink)                                      │
│  ┌─────────────────────────────────────────────┐            │
│  │ 右表扫描 → Build Hash Table                  │            │
│  │                                              │            │
│  │ Thread 1: Sink() → local_hash_table_1       │            │
│  │ Thread 2: Sink() → local_hash_table_2       │            │
│  │    ...                                       │            │
│  │ Combine(): merge local → global             │            │
│  │ Finalize(): build pointer table             │            │
│  └─────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼ Hash Table Ready                    │
│                                                             │
│  Probe Pipeline (Operator)                                  │
│  ┌─────────────────────────────────────────────┐            │
│  │ 左表扫描 → Probe Hash Table → 输出匹配行    │            │
│  │                                              │            │
│  │ ExecuteInternal():                          │            │
│  │   1. 计算 probe keys                         │            │
│  │   2. hash_table.Probe()                     │            │
│  │   3. 输出匹配结果                            │            │
│  └─────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼ (如果是 Full/Right Outer)           │
│                                                             │
│  Scan Pipeline (Source)                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 扫描 Hash Table 中未匹配的行                 │            │
│  │ GetDataInternal(): ScanFullOuter()          │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Hash Join Sink 实现：**

```cpp
// src/execution/operator/join/physical_hash_join.cpp

class HashJoinGlobalSinkState : public GlobalSinkState {
public:
    HashJoinGlobalSinkState(const PhysicalHashJoin &op, ClientContext &context)
        : context(context), op(op),
          num_threads(TaskScheduler::GetScheduler(context).NumberOfThreads()),
          temporary_memory_state(...) {
        // 初始化全局哈希表
        hash_table = op.InitializeHashTable(context);
        // Perfect Hash Join 优化
        perfect_join_executor = make_uniq<PerfectHashJoinExecutor>(op, *hash_table);
    }

    unique_ptr<JoinHashTable> hash_table;
    unique_ptr<PerfectHashJoinExecutor> perfect_join_executor;
    vector<unique_ptr<JoinHashTable>> local_hash_tables;
    atomic<idx_t> active_local_states;
    bool finalized;
    bool external;  // 外部哈希连接标志
};

class HashJoinLocalSinkState : public LocalSinkState {
public:
    HashJoinLocalSinkState(const PhysicalHashJoin &op, ClientContext &context,
                          HashJoinGlobalSinkState &gstate)
        : join_key_executor(context) {
        // 每个线程私有的哈希表
        hash_table = op.InitializeHashTable(context);

        // 初始化 Join Key 执行器
        for (auto &cond : op.conditions) {
            join_key_executor.AddExpression(*cond.right);
        }
        join_keys.Initialize(allocator, op.condition_types);
    }

    ExpressionExecutor join_key_executor;
    DataChunk join_keys;
    unique_ptr<JoinHashTable> hash_table;
};

SinkResultType PhysicalHashJoin::Sink(ExecutionContext &context,
                                      DataChunk &chunk,
                                      OperatorSinkInput &input) const {
    auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
    auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

    // 1. 计算 Join Keys
    lstate.join_keys.Reset();
    lstate.join_key_executor.Execute(chunk, lstate.join_keys);

    // 2. 准备 Payload（非 Key 列）
    if (payload_columns.col_types.empty()) {
        lstate.payload_chunk.SetCardinality(chunk.size());
    } else {
        lstate.payload_chunk.ReferenceColumns(chunk, payload_columns.col_idxs);
    }

    // 3. 插入本地哈希表
    lstate.hash_table->Build(lstate.append_state, lstate.join_keys,
                            lstate.payload_chunk);

    return SinkResultType::NEED_MORE_INPUT;
}

SinkCombineResultType PhysicalHashJoin::Combine(ExecutionContext &context,
                                                OperatorSinkCombineInput &input) const {
    auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
    auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

    // 刷新本地状态
    lstate.hash_table->GetSinkCollection().FlushAppendState(lstate.append_state);

    // 加锁合并到全局列表
    auto guard = gstate.Lock();
    gstate.local_hash_tables.push_back(std::move(lstate.hash_table));

    return SinkCombineResultType::FINISHED;
}
```

### 4.5 Hash Aggregate (Sink 算子)

分组聚合是另一个典型的 Sink 算子：

```cpp
// src/include/duckdb/execution/operator/aggregate/physical_hash_aggregate.hpp

class PhysicalHashAggregate : public PhysicalOperator {
public:
    static constexpr const PhysicalOperatorType TYPE =
        PhysicalOperatorType::HASH_GROUP_BY;

    //! 分组键
    vector<unique_ptr<Expression>> groups;
    //! 聚合函数
    vector<unique_ptr<Expression>> aggregates;
    //! 分组类型（区分是否需要聚合）
    vector<GroupingSet> grouping_sets;

public:
    // 主要是 Sink，但也作为 Source 输出结果
    bool IsSink() const override { return true; }
    bool ParallelSink() const override { return true; }
    bool IsSource() const override { return true; }
    bool ParallelSource() const override { return true; }
};
```

**聚合执行模式：**

```
┌─────────────────────────────────────────────────────────────┐
│                  Hash Aggregate 执行流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Sink 阶段: 构建聚合哈希表                                   │
│  ┌─────────────────────────────────────────────┐            │
│  │ Thread 1                Thread N            │            │
│  │    │                       │                │            │
│  │    ▼                       ▼                │            │
│  │ local_ht_1             local_ht_N           │            │
│  │ ┌─────────┐           ┌─────────┐           │            │
│  │ │ group_a │           │ group_a │           │            │
│  │ │ sum: 10 │           │ sum: 20 │           │            │
│  │ │ count:2 │           │ count:3 │           │            │
│  │ ├─────────┤           ├─────────┤           │            │
│  │ │ group_b │           │ group_c │           │            │
│  │ │ sum: 5  │           │ sum: 15 │           │            │
│  │ └─────────┘           └─────────┘           │            │
│  └─────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼ Combine & Finalize                  │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │              Global Hash Table              │            │
│  │ ┌─────────┬─────────┬─────────┐             │            │
│  │ │ group_a │ group_b │ group_c │             │            │
│  │ │ sum: 30 │ sum: 5  │ sum: 15 │             │            │
│  │ │ count:5 │ count:1 │ count:3 │             │            │
│  │ └─────────┴─────────┴─────────┘             │            │
│  └─────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼ Source 阶段                         │
│                                                             │
│  输出聚合结果 DataChunks                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 5. CachingPhysicalOperator

### 5.1 小批次缓存优化

为处理过滤后的小批次数据，DuckDB 提供了缓存包装器：

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class CachingOperatorState : public OperatorState {
public:
    unique_ptr<DataChunk> cached_chunk;
    bool initialized = false;
    bool can_cache_chunk = false;
};

class CachingPhysicalOperator : public PhysicalOperator {
public:
    static constexpr const idx_t CACHE_THRESHOLD = 64;  // 64 行阈值
    bool caching_supported;

public:
    // 包装 Execute，添加缓存逻辑
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input, DataChunk &chunk,
                               GlobalOperatorState &gstate,
                               OperatorState &state) const final;

    // 输出缓存中的剩余数据
    OperatorFinalizeResultType FinalExecute(ExecutionContext &context,
                                            DataChunk &chunk,
                                            GlobalOperatorState &gstate,
                                            OperatorState &state) const final;

protected:
    // 子类实现的实际逻辑
    virtual OperatorResultType ExecuteInternal(ExecutionContext &context,
                                               DataChunk &input,
                                               DataChunk &chunk,
                                               GlobalOperatorState &gstate,
                                               OperatorState &state) const = 0;
};
```

**缓存工作原理：**

```
┌─────────────────────────────────────────────────────────────┐
│              CachingPhysicalOperator 工作流程               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  输入: 多个小批次 (e.g., Filter 后只剩 30, 45, 20 行)        │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Batch 1  │    │ Batch 2  │    │ Batch 3  │              │
│  │  30 行   │    │  45 行   │    │  20 行   │              │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘              │
│       │               │               │                     │
│       ▼               ▼               ▼                     │
│  ┌─────────────────────────────────────────────┐            │
│  │            CachedChunk (2048 capacity)      │            │
│  │                                              │            │
│  │  [30 rows] ← append                         │            │
│  │  [75 rows] ← append (30+45)                 │            │
│  │  [95 rows] ← append (75+20)                 │            │
│  │                                              │            │
│  │  当累积 >= CACHE_THRESHOLD (64) 时输出       │            │
│  │  或 FinalExecute 时输出剩余                  │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  优势: 避免处理大量小批次的开销                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 6. Pipeline 构建

### 6.1 BuildPipelines 接口

每个算子都参与 Pipeline 的构建：

```cpp
class PhysicalOperator {
public:
    virtual void BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline);
    virtual vector<const_reference<PhysicalOperator>> GetSources() const;
    bool AllSourcesSupportBatchIndex() const;
};
```

### 6.2 Pipeline 分割示例

以 Hash Join 为例，展示 Pipeline 如何被分割：

```
原始执行计划:

        Result
           │
      Hash Join
       /       \
  Scan A       Scan B

Pipeline 构建后:

Pipeline 1 (Build):    Pipeline 2 (Probe):

    Scan B                 Scan A
       │                      │
       ▼                      ▼
    HashJoin.Sink()       HashJoin.Operator()
                              │
                              ▼
                           Result

Pipeline 依赖: Pipeline 2 等待 Pipeline 1 完成
```

**BuildPipelines 实现模式：**

```cpp
void PhysicalHashJoin::BuildPipelines(Pipeline &current,
                                      MetaPipeline &meta_pipeline) {
    // Build 侧创建新 Pipeline
    auto &build_side = children[1].get();
    meta_pipeline.CreateChildMetaPipeline(current, *this);
    // 当前 Pipeline 继续处理 Probe 侧
    children[0].get().BuildPipelines(current, meta_pipeline);
}
```

## 7. 顺序保持

### 7.1 OrderPreservationType

算子可以声明对顺序的影响：

```cpp
enum class OrderPreservationType : uint8_t {
    INSERTION_ORDER,      // 保持插入顺序
    NO_ORDER,             // 不保证顺序
    FIXED_ORDER           // 产生固定顺序（如 ORDER BY）
};

class PhysicalOperator {
public:
    //! 作为 Operator 时对顺序的影响
    virtual OrderPreservationType OperatorOrder() const {
        return OrderPreservationType::INSERTION_ORDER;
    }

    //! 作为 Source 时输出的顺序
    virtual OrderPreservationType SourceOrder() const {
        return OrderPreservationType::INSERTION_ORDER;
    }
};
```

### 7.2 顺序保持示例

```
┌─────────────────────────────────────────────────────────────┐
│                   算子顺序保持特性                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  算子类型          │ 顺序保持           │ 说明               │
│  ─────────────────┼───────────────────┼──────────────────── │
│  Filter           │ INSERTION_ORDER   │ 保持输入顺序         │
│  Projection       │ INSERTION_ORDER   │ 保持输入顺序         │
│  TableScan        │ 取决于存储         │ 可能无序             │
│  HashJoin         │ NO_ORDER          │ 哈希重排序           │
│  OrderBy          │ FIXED_ORDER       │ 产生排序结果         │
│  HashAggregate    │ NO_ORDER          │ 哈希重排序           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 8. 进度报告

### 8.1 GetProgress 接口

算子可以报告执行进度：

```cpp
class PhysicalOperator {
public:
    //! Source 进度
    virtual ProgressData GetProgress(ClientContext &context,
                                    GlobalSourceState &gstate) const;

    //! Sink 进度
    virtual ProgressData GetSinkProgress(ClientContext &context,
                                        GlobalSinkState &gstate,
                                        const ProgressData source_progress) const;
};

struct ProgressData {
    double done;    // 已完成量
    double total;   // 总量
};
```

## 9. 完整示例：查询执行流程

让我们通过一个完整的查询示例来理解算子的协作：

```sql
SELECT a.id, SUM(b.value)
FROM table_a a
JOIN table_b b ON a.id = b.a_id
WHERE a.type = 'active'
GROUP BY a.id;
```

**执行计划和 Pipeline：**

```
┌─────────────────────────────────────────────────────────────┐
│                      查询执行流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  物理计划:                                                   │
│                                                             │
│            PhysicalHashAggregate                            │
│                    │                                        │
│            PhysicalHashJoin                                 │
│             /            \                                  │
│     PhysicalFilter    PhysicalTableScan(b)                 │
│           │                                                 │
│     PhysicalTableScan(a)                                    │
│                                                             │
│  ═══════════════════════════════════════════════════════   │
│                                                             │
│  Pipeline 划分:                                              │
│                                                             │
│  Pipeline 1 (Build Hash Table):                            │
│  ┌───────────────────────────────────────┐                  │
│  │ TableScan(b) → HashJoin.Sink()        │                  │
│  │                                        │                  │
│  │ 状态: GlobalSinkState 持有 hash_table │                  │
│  └───────────────────────────────────────┘                  │
│                     │                                       │
│                     ▼ 等待完成                              │
│                                                             │
│  Pipeline 2 (Probe + Aggregate):                           │
│  ┌───────────────────────────────────────┐                  │
│  │ TableScan(a)                          │  Source          │
│  │      │                                 │                  │
│  │      ▼                                 │                  │
│  │ Filter (type='active')                │  Operator        │
│  │      │                                 │                  │
│  │      ▼                                 │                  │
│  │ HashJoin.Execute()                    │  Operator        │
│  │      │                                 │                  │
│  │      ▼                                 │                  │
│  │ HashAggregate.Sink()                  │  Sink            │
│  └───────────────────────────────────────┘                  │
│                     │                                       │
│                     ▼ 等待完成                              │
│                                                             │
│  Pipeline 3 (Output):                                       │
│  ┌───────────────────────────────────────┐                  │
│  │ HashAggregate.Source() → Result       │                  │
│  └───────────────────────────────────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 10. 总结

### 10.1 核心设计原则

1. **三接口模式**：Source/Operator/Sink 清晰分离数据生产、转换、消费
2. **状态分离**：Global/Local 状态支持高效的并行执行
3. **零拷贝传递**：DataChunk 在算子间引用传递，避免不必要的复制
4. **缓存优化**：CachingPhysicalOperator 处理小批次数据
5. **Pipeline 感知**：算子参与 Pipeline 构建，支持流水线并行

### 10.2 设计优势

| 特性 | 优势 |
|------|------|
| 统一接口 | 新算子易于实现，遵循统一模式 |
| 并行支持 | ParallelSink/ParallelSource/ParallelOperator 声明式并行 |
| 内存控制 | 状态封装支持精确的内存管理 |
| 可扩展性 | 模块化设计便于添加新算子 |
| 可测试性 | 清晰的接口边界便于单元测试 |

### 10.3 算子分类总览

```
┌────────────────────────────────────────────────────────────────┐
│                     物理算子分类                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Source Only:                                                  │
│  ├─ TableScan, ColumnDataScan, ExpressionScan                 │
│  └─ ChunkScan, DelimScan, CTEScan                             │
│                                                                │
│  Operator Only:                                                │
│  ├─ Filter, Projection                                        │
│  ├─ StreamingLimit, StreamingSample                           │
│  └─ StreamingWindow                                           │
│                                                                │
│  Sink + Source:                                                │
│  ├─ HashAggregate, PerfectHashAggregate                       │
│  ├─ OrderBy, TopN, Window                                     │
│  └─ Union, RecursiveCTE                                       │
│                                                                │
│  Sink + Operator + Source:                                     │
│  ├─ HashJoin (Build/Probe/FullOuterScan)                      │
│  └─ NestedLoopJoin, MergeJoin                                 │
│                                                                │
│  Sink Only:                                                    │
│  ├─ Insert, Delete, Update                                    │
│  ├─ CopyToFile, BatchCopyToFile                               │
│  └─ ResultCollector                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

在下一章中，我们将深入探讨 **Pipeline 并行执行**，了解 DuckDB 如何利用多核 CPU 实现高效的并行查询处理。

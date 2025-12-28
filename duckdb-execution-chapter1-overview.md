# DuckDB 执行引擎深度解析：第一章 执行模型概述

执行引擎是数据库系统中最核心的组件之一，它决定了查询如何被实际执行以及执行的效率。DuckDB 采用了先进的**向量化执行模型**（Vectorized Execution Model）结合 **Push-based 执行策略**，实现了卓越的查询性能。本章将从宏观角度介绍数据库执行模型的演进历程，详细分析 DuckDB 的执行架构设计理念。

---

## 1.1 数据库执行模型演进

数据库执行引擎的设计经历了数十年的演进，从最初的逐行处理模型发展到现代的向量化和编译执行模型。理解这一演进过程对于深入理解 DuckDB 的设计选择至关重要。

### 1.1.1 火山模型 (Volcano/Iterator Model)

火山模型是由 Goetz Graefe 在 1994 年提出的经典执行模型，也称为迭代器模型（Iterator Model）。这一模型奠定了现代关系数据库执行引擎的基础，至今仍被许多主流数据库系统采用。

**核心思想**：

火山模型将查询计划中的每个算子（Operator）抽象为一个迭代器，每个迭代器提供统一的 `next()` 接口：

```
┌─────────────────────────────────────────┐
│            Result Collector             │
│               next() ─┐                 │
└────────────────────│──┘                 │
                     ↓                    │
┌─────────────────────────────────────────┐
│              Projection                 │
│         next() ───────┐                 │
└────────────────────│──┘                 │
                     ↓                    │
┌─────────────────────────────────────────┐
│               Filter                    │
│         next() ───────┐                 │
└────────────────────│──┘                 │
                     ↓                    │
┌─────────────────────────────────────────┐
│            Table Scan                   │
│         next() → tuple                  │
└─────────────────────────────────────────┘

数据流向：自底向上，每次返回一行
控制流向：自顶向下，递归调用 next()
```

**代码模式**：

```cpp
// 经典火山模型接口
class Operator {
public:
    virtual void Open() = 0;
    virtual Tuple* Next() = 0;   // 每次返回一行
    virtual void Close() = 0;
};

// Filter 算子的典型实现
class FilterOperator : public Operator {
    Operator* child;
    Expression* predicate;

    Tuple* Next() override {
        while (true) {
            Tuple* tuple = child->Next();
            if (tuple == nullptr) return nullptr;
            if (predicate->Evaluate(tuple)) {
                return tuple;  // 满足条件，返回
            }
            // 不满足条件，继续获取下一行
        }
    }
};
```

**优点**：
- **简单直观**：接口设计清晰，易于理解和实现
- **内存友好**：每次只处理一行，内存占用极低
- **通用性强**：统一的接口使得算子可以任意组合
- **流式处理**：天然支持流式输出，不需要物化所有结果

**缺点**：
- **函数调用开销巨大**：每处理一行数据需要多次虚函数调用，对于百万级数据查询，函数调用开销占据了相当比例的执行时间
- **CPU 缓存不友好**：频繁的函数调用和小数据量处理导致 CPU 缓存无法被有效利用
- **分支预测困难**：逐行判断条件导致 CPU 分支预测器难以发挥作用
- **无法利用 SIMD**：单行处理完全无法利用现代 CPU 的 SIMD 指令集

**性能分析**：

研究表明，在火山模型中，真正的数据处理只占总执行时间的一小部分，大量时间消耗在解释开销（interpretation overhead）上：

```
执行时间分解 (典型 OLAP 查询):
┌────────────────────────────────────────┐
│ 虚函数调用开销           ~30%           │
│ 缓存未命中开销           ~25%           │
│ 分支预测失败开销         ~15%           │
│ 实际数据处理             ~30%           │
└────────────────────────────────────────┘
```

### 1.1.2 向量化模型 (Vectorized Model)

向量化执行模型是对火山模型的重要改进，由 MonetDB/X100 项目在 2005 年提出。这一模型保留了火山模型的迭代器架构，但将处理粒度从**单行**提升到**向量（Vector）**。

**核心思想**：

向量化模型每次处理一批数据（通常 1000-2048 行），而不是单行：

```
┌─────────────────────────────────────────┐
│            Result Collector             │
│          GetChunk() ───┐                │
└────────────────────│───┘                │
                     ↓                    │
┌─────────────────────────────────────────┐
│              Projection                 │
│          GetChunk() ───┐                │
└────────────────────│───┘                │
                     ↓                    │
┌─────────────────────────────────────────┐
│               Filter                    │
│          GetChunk() ───┐                │
└────────────────────│───┘                │
                     ↓                    │
┌─────────────────────────────────────────┐
│            Table Scan                   │
│      GetChunk() → DataChunk[2048]       │
└─────────────────────────────────────────┘

每次返回 2048 行的向量化数据块
```

**关键优化原理**：

1. **摊销解释开销**：
```
火山模型:    N 行 × M 个算子 = N × M 次函数调用
向量化模型:  (N/2048) × M 次函数调用

对于 100 万行数据和 5 个算子:
火山模型:    5,000,000 次函数调用
向量化模型:  约 2,450 次函数调用 (减少 99.95%)
```

2. **提高缓存利用率**：
```
向量化批处理使得:
- L1 缓存 (32KB): 可容纳一个 DataChunk
- 同一数据被多次访问时更可能在缓存中
- 顺序内存访问模式便于预取
```

3. **SIMD 向量化**：
```cpp
// 标量加法
for (int i = 0; i < N; i++) {
    result[i] = a[i] + b[i];
}

// SIMD 加法 (AVX2, 每次处理 8 个整数)
for (int i = 0; i < N; i += 8) {
    __m256i va = _mm256_load_si256((__m256i*)&a[i]);
    __m256i vb = _mm256_load_si256((__m256i*)&b[i]);
    __m256i vr = _mm256_add_epi32(va, vb);
    _mm256_store_si256((__m256i*)&result[i], vr);
}
// 理论加速: 8x (实际约 4-6x)
```

4. **编译器优化友好**：
紧凑的循环代码使编译器更容易进行循环展开、向量化等优化。

**DuckDB 的选择**：

DuckDB 选择向量化执行模型的原因：
- **平衡解释开销和物化开销**：批量大小 2048 是经过精心选择的
- **代码可维护性**：无需复杂的代码生成基础设施
- **调试友好**：相比编译执行更容易调试
- **快速启动**：无需 JIT 编译的启动延迟

### 1.1.3 编译执行模型 (Compiled/Code Generation Model)

编译执行模型是另一种消除解释开销的方法，以 HyPer 数据库为代表。这种方法通过即时编译（JIT）将查询计划编译成原生机器码。

**核心思想**：

将整个查询（或查询的 Pipeline）编译成一个紧凑的代码块：

```
SQL Query:
  SELECT a + b FROM t WHERE c > 10

编译后的代码 (伪代码):
  for each row in table t:
      if row.c > 10:
          emit(row.a + row.b)
```

**与向量化模型的对比**：

| 特性 | 向量化模型 | 编译执行模型 |
|------|-----------|-------------|
| 解释开销 | 大幅减少 | 完全消除 |
| 启动时间 | 即时 | 需要编译时间 |
| 代码复杂度 | 中等 | 高 (需要代码生成器) |
| 调试难度 | 中等 | 高 |
| SIMD 利用 | 显式编程 | 依赖编译器 |
| 数据并行 | 自然支持 | 需要额外处理 |

**DuckDB 为何选择向量化而非编译执行**：

1. **工程复杂度**：代码生成需要维护复杂的编译器基础设施
2. **调试便利性**：向量化代码更容易追踪和调试
3. **启动延迟**：JIT 编译需要时间，对短查询不友好
4. **性能平衡**：向量化已能达到接近编译执行的性能水平

---

## 1.2 Push vs Pull 执行策略

除了数据处理粒度，执行引擎的另一个重要设计决策是**数据流动方向**：是由消费者主动拉取（Pull）还是由生产者主动推送（Push）。

### 1.2.1 Pull 模型 (消费者驱动)

Pull 模型是传统火山模型的标准方式，数据由消费者（上层算子）主动从生产者（下层算子）拉取：

```
┌─────────────────────────────────────────┐
│           Result Collector              │
│                 │                       │
│             next()                      │
│                 ↓                       │
├─────────────────────────────────────────┤
│              Aggregation                │
│                 │                       │
│             next()                      │
│                 ↓                       │
├─────────────────────────────────────────┤
│               Filter                    │
│                 │                       │
│             next()                      │
│                 ↓                       │
├─────────────────────────────────────────┤
│            Table Scan                   │
└─────────────────────────────────────────┘

控制流 (↓): 自顶向下
数据流 (↑): 自底向上
```

**Pull 模型的特点**：
- **递归调用栈**：每个 `next()` 调用会触发下层算子的 `next()`
- **流程控制简单**：顶层算子控制整个执行流程
- **阻塞等待**：上层必须等待下层返回数据

**代码模式**：

```cpp
// Pull 模型的典型执行循环
Tuple* ResultCollector::Next() {
    while (true) {
        Tuple* tuple = child->Next();
        if (tuple == nullptr) {
            return nullptr;  // 执行完毕
        }
        output_buffer.Add(tuple);
        if (output_buffer.IsFull()) {
            return output_buffer.Flush();
        }
    }
}
```

### 1.2.2 Push 模型 (生产者驱动)

Push 模型中，数据由生产者主动推送给消费者。DuckDB 采用的就是这种模式：

```
┌─────────────────────────────────────────┐
│            Table Scan                   │
│                 │                       │
│            Push(chunk)                  │
│                 ↓                       │
├─────────────────────────────────────────┤
│               Filter                    │
│                 │                       │
│            Push(chunk)                  │
│                 ↓                       │
├─────────────────────────────────────────┤
│            Aggregation                  │
│                 │                       │
│            Push(chunk)                  │
│                 ↓                       │
├─────────────────────────────────────────┤
│           Result Collector              │
└─────────────────────────────────────────┘

控制流 (↓): 自顶向下 (推送)
数据流 (↓): 自顶向下 (同向)
```

**Push 模型的核心优势**：

1. **更好的并行支持**：
```
Push 模型天然支持多线程执行:

Thread 1:  Scan Part1 → Filter → Aggregate(local)
Thread 2:  Scan Part2 → Filter → Aggregate(local)
Thread 3:  Scan Part3 → Filter → Aggregate(local)
                              ↓
                    Merge(global aggregate)

每个线程独立推送数据，只在必要时同步
```

2. **Pipeline 天然适配**：
```
Push 模型的 Pipeline 执行:

Pipeline 1 (Build):
  TableScan → Filter → HashJoin.Build
                          ↓ (物化)
Pipeline 2 (Probe):
  TableScan → HashJoin.Probe → Result
                          ↓
                     Output

数据在 Pipeline 内流式处理，无需物化中间结果
```

3. **减少函数调用栈深度**：
```
Pull 模型调用栈:
  ResultCollector.Next()
    → Aggregation.Next()
      → Filter.Next()
        → Scan.Next()

Push 模型调用栈:
  Source.GetData()
  → Filter.Execute()
  → Aggregation.Execute()
  → Sink.Sink()

调用栈更扁平，便于管理
```

4. **更容易实现异步执行**：
```cpp
// Push 模型支持异步中断和恢复
SinkResultType Sink(DataChunk &chunk) {
    if (buffer_full) {
        return SinkResultType::BLOCKED;  // 暂停，稍后重试
    }
    ProcessChunk(chunk);
    return SinkResultType::NEED_MORE_INPUT;
}
```

### 1.2.3 DuckDB 的 Push 模型实现

DuckDB 的执行采用 Push 模型，数据从 Source 算子开始，经过中间算子处理，最终推送到 Sink 算子：

```cpp
// DuckDB 的 Push 执行循环 (简化版)
// src/parallel/pipeline_executor.cpp

PipelineExecuteResult PipelineExecutor::Execute() {
    while (!exhausted_pipeline) {
        // 1. 从 Source 获取数据
        DataChunk source_chunk;
        source_chunk.Reset();
        SourceResultType source_result = FetchFromSource(source_chunk);

        if (source_result == SourceResultType::FINISHED) {
            exhausted_pipeline = true;
            break;
        }

        // 2. 将数据推送经过所有中间算子
        DataChunk final_chunk;
        OperatorResultType result = Execute(source_chunk, final_chunk);

        // 3. 将结果推送到 Sink
        if (final_chunk.size() > 0) {
            SinkResultType sink_result = Sink(final_chunk, sink_input);
            if (sink_result == SinkResultType::BLOCKED) {
                return PipelineExecuteResult::INTERRUPTED;
            }
        }
    }
    return PipelineExecuteResult::FINISHED;
}
```

---

## 1.3 DuckDB 执行架构

理解了执行模型的设计选择后，让我们深入 DuckDB 的具体执行架构。

### 1.3.1 查询执行流程

一个 SQL 查询在 DuckDB 中的完整执行流程：

```
┌────────────────────────────────────────────────────────────────┐
│                        SQL Query                                │
│   "SELECT a, SUM(b) FROM t WHERE c > 10 GROUP BY a"            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                      Parser (解析器)                            │
│                使用 libpg_query 生成 AST                        │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     Binder (绑定器)                             │
│              符号解析，类型推导，语义验证                         │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                   Logical Plan (逻辑计划)                       │
│                                                                 │
│     LogicalAggregate                                           │
│         ├── groups: [a]                                        │
│         ├── aggregates: [SUM(b)]                               │
│         └── LogicalFilter (c > 10)                             │
│                 └── LogicalGet (table: t)                      │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                    Optimizer (优化器)                           │
│             谓词下推、连接顺序优化、表达式重写等                   │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                  Physical Plan (物理计划)                       │
│                                                                 │
│     PhysicalHashAggregate                                      │
│         ├── groups: [#0]                                       │
│         ├── aggregates: [SUM(#1)]                              │
│         └── PhysicalTableScan                                  │
│                 ├── table: t                                   │
│                 ├── columns: [a, b]                            │
│                 └── filter: c > 10 (pushed down)               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│              Pipeline Construction (Pipeline 构建)              │
│                                                                 │
│     Pipeline 1: Scan → Aggregate.Build (Sink)                  │
│     Pipeline 2: Aggregate.Scan (Source) → Result               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│              Parallel Execution (并行执行)                      │
│                                                                 │
│     ┌─────────────────────────────────────────┐                │
│     │          TaskScheduler                   │                │
│     │    ┌───────────────────────────┐        │                │
│     │    │ Worker Threads            │        │                │
│     │    │  T1  T2  T3  T4  ...      │        │                │
│     │    └───────────────────────────┘        │                │
│     └─────────────────────────────────────────┘                │
└────────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                     Query Result                                │
└────────────────────────────────────────────────────────────────┘
```

### 1.3.2 核心组件及其关系

DuckDB 执行引擎的核心组件层次结构：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Executor                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    PipelineExecutor                        │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                   Pipeline                           │  │  │
│  │  │  ┌─────────────────────────────────────────────┐    │  │  │
│  │  │  │        PhysicalOperator (Source)            │    │  │  │
│  │  │  │             GetData()                       │    │  │  │
│  │  │  └─────────────────────────────────────────────┘    │  │  │
│  │  │                      ↓                               │  │  │
│  │  │  ┌─────────────────────────────────────────────┐    │  │  │
│  │  │  │     PhysicalOperator[] (Intermediate)       │    │  │  │
│  │  │  │             Execute()                       │    │  │  │
│  │  │  └─────────────────────────────────────────────┘    │  │  │
│  │  │                      ↓                               │  │  │
│  │  │  ┌─────────────────────────────────────────────┐    │  │  │
│  │  │  │         PhysicalOperator (Sink)             │    │  │  │
│  │  │  │              Sink()                         │    │  │  │
│  │  │  └─────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              ExpressionExecutor                      │  │  │
│  │  │         执行表达式 (投影、过滤条件等)                   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**组件职责说明**：

1. **Executor** (`src/parallel/executor.cpp`)
   - 管理整个查询的执行生命周期
   - 协调多个 Pipeline 的调度和执行
   - 处理错误和取消请求

```cpp
// Executor 核心结构 (src/include/duckdb/execution/executor.hpp)
class Executor {
    ClientContext &context;

    // 所有 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    vector<shared_ptr<Pipeline>> root_pipelines;

    // 事件系统
    vector<shared_ptr<Event>> events;

    // 执行状态
    atomic<idx_t> completed_pipelines;
    idx_t total_pipelines;

    // 任务调度
    unique_ptr<ProducerToken> producer;
};
```

2. **Pipeline** (`src/parallel/pipeline.cpp`)
   - 表示一条执行管道（Source → Operators → Sink）
   - 管理并行执行的任务分配

```cpp
// Pipeline 核心结构 (src/include/duckdb/parallel/pipeline.hpp)
class Pipeline {
    Executor &executor;

    // 数据源算子
    optional_ptr<PhysicalOperator> source;
    // 中间算子链
    vector<reference<PhysicalOperator>> operators;
    // 数据汇算子
    optional_ptr<PhysicalOperator> sink;

    // 依赖关系
    vector<weak_ptr<Pipeline>> parents;
    vector<weak_ptr<Pipeline>> dependencies;
};
```

3. **PipelineExecutor** (`src/parallel/pipeline_executor.cpp`)
   - 在单个线程中执行一个 Pipeline
   - 管理中间状态和数据块

```cpp
// PipelineExecutor 核心结构 (src/include/duckdb/parallel/pipeline_executor.hpp)
class PipelineExecutor {
    Pipeline &pipeline;
    ThreadContext thread;
    ExecutionContext context;

    // 中间数据块（每个算子一个）
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    // 算子状态
    vector<unique_ptr<OperatorState>> intermediate_states;

    // 本地状态
    unique_ptr<LocalSourceState> local_source_state;
    unique_ptr<LocalSinkState> local_sink_state;

    // 最终输出块
    DataChunk final_chunk;
};
```

4. **PhysicalOperator** (`src/include/duckdb/execution/physical_operator.hpp`)
   - 所有物理算子的基类
   - 定义三种核心接口：Source、Operator、Sink

```cpp
// PhysicalOperator 核心接口
class PhysicalOperator {
public:
    // Source 接口 - Pipeline 起点
    virtual SourceResultType GetData(
        ExecutionContext &context,
        DataChunk &chunk,
        OperatorSourceInput &input);

    // Operator 接口 - 中间算子
    virtual OperatorResultType Execute(
        ExecutionContext &context,
        DataChunk &input,
        DataChunk &chunk,
        GlobalOperatorState &gstate,
        OperatorState &state);

    // Sink 接口 - Pipeline 终点
    virtual SinkResultType Sink(
        ExecutionContext &context,
        DataChunk &chunk,
        OperatorSinkInput &input);
};
```

### 1.3.3 执行上下文

DuckDB 使用多级上下文来管理执行状态：

```
┌─────────────────────────────────────────────────────────────────┐
│                      ClientContext                               │
│   客户端级别上下文，包含数据库连接、事务、配置等                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Executor                              │   │
│   │    执行器级别，管理一个查询的执行                          │   │
│   │    ┌─────────────────────────────────────────────────┐   │   │
│   │    │              ThreadContext                       │   │   │
│   │    │    线程级别，每个工作线程独立                       │   │   │
│   │    │    ┌─────────────────────────────────────────┐   │   │   │
│   │    │    │          ExecutionContext               │   │   │   │
│   │    │    │    执行上下文，传递给每个算子              │   │   │   │
│   │    │    │    - ClientContext &client              │   │   │   │
│   │    │    │    - ThreadContext &thread              │   │   │   │
│   │    │    │    - Pipeline *pipeline                 │   │   │   │
│   │    │    └─────────────────────────────────────────┘   │   │   │
│   │    └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

```cpp
// ExecutionContext 定义 (src/include/duckdb/execution/execution_context.hpp)
class ExecutionContext {
public:
    ExecutionContext(ClientContext &client_p,
                     ThreadContext &thread_p,
                     optional_ptr<Pipeline> pipeline_p)
        : client(client_p), thread(thread_p), pipeline(pipeline_p) {}

    //! 客户端全局上下文 (并行时需要加锁访问)
    ClientContext &client;
    //! 线程本地上下文
    ThreadContext &thread;
    //! 当前 Pipeline 引用
    optional_ptr<Pipeline> pipeline;
};
```

---

## 1.4 向量化执行原理

### 1.4.1 为什么是 2048?

DuckDB 选择 2048 作为标准向量大小（STANDARD_VECTOR_SIZE）是经过精心考量的：

```cpp
// src/include/duckdb/common/vector_size.hpp
#define DEFAULT_STANDARD_VECTOR_SIZE 2048U
```

**选择 2048 的考量因素**：

1. **CPU L1 缓存适配**：
```
现代 CPU L1 数据缓存: 32KB
2048 个 64 位整数: 2048 × 8 = 16KB
单个 Vector + 元数据: ~16-20KB
一个 DataChunk (多列): 可部分装入 L1 缓存

目标: 让热点数据尽量停留在 L1/L2 缓存
```

2. **SIMD 对齐**：
```
AVX-512: 512 bits = 64 bytes = 8 个 64位整数
2048 = 8 × 256 (完美对齐 AVX-512)
2048 = 4 × 512 (完美对齐 AVX2)

2 的幂次便于对齐和位操作
```

3. **摊销开销与物化开销的平衡**：
```
批量太小 (如 64):
  - 解释开销仍然较高
  - 函数调用开销占比大

批量太大 (如 65536):
  - 物化成本高
  - 可能超出缓存
  - 延迟高

2048 是实验验证的最佳平衡点
```

4. **ValidityMask 效率**：
```cpp
// ValidityMask 使用 64 位整数存储 64 个标志位
// 2048 = 64 × 32 (恰好 32 个 uint64_t)
static constexpr const idx_t STANDARD_ENTRY_COUNT =
    (STANDARD_VECTOR_SIZE + 63) / 64;  // = 32
```

### 1.4.2 向量化的收益

向量化执行带来的性能收益是多方面的：

**1. 减少解释开销**

```
火山模型每行开销:
  - 虚函数调用: ~10-20 cycles
  - 条件分支: ~5-10 cycles
  - 函数调用栈维护: ~5 cycles
  总计: ~20-35 cycles/行

向量化模型每行开销:
  - 循环迭代: ~1-2 cycles
  - 数组访问: ~1 cycle
  总计: ~2-3 cycles/行

加速比: ~10x (仅解释开销)
```

**2. 提高缓存命中率**

```cpp
// 顺序处理同类型数据
void AddVectors(int32_t* a, int32_t* b, int32_t* result, idx_t count) {
    for (idx_t i = 0; i < count; i++) {
        result[i] = a[i] + b[i];  // 顺序内存访问
    }
}

// 内存访问模式:
// a[0], a[1], a[2], ... (顺序)
// b[0], b[1], b[2], ... (顺序)
//
// CPU 可以有效预取数据
// 缓存行利用率高
```

**3. SIMD 指令利用**

```cpp
// DuckDB 的向量化操作示例
// 实际实现会使用模板和 SIMD 内联函数

template <class OP>
static void BinaryOperation(Vector &left, Vector &right,
                            Vector &result, idx_t count) {
    auto ldata = FlatVector::GetData<int64_t>(left);
    auto rdata = FlatVector::GetData<int64_t>(right);
    auto result_data = FlatVector::GetData<int64_t>(result);

    // 编译器可以自动向量化这个循环
    for (idx_t i = 0; i < count; i++) {
        result_data[i] = OP::Operation(ldata[i], rdata[i]);
    }
}
```

**4. 编译器优化友好**

紧凑的循环代码使编译器更容易进行：
- 循环展开 (Loop Unrolling)
- 自动向量化 (Auto-vectorization)
- 指令调度优化 (Instruction Scheduling)
- 寄存器分配优化 (Register Allocation)

### 1.4.3 向量化的挑战

向量化执行也带来了一些技术挑战：

**1. 控制流处理 (SelectionVector)**

当存在过滤条件时，不是所有行都需要处理：

```cpp
// 问题: WHERE age > 18 过滤后，只有部分行有效
//
// 解决方案: SelectionVector
//
// 原始数据:  [15, 22, 17, 25, 30, 16, 28]
// 过滤条件:  age > 18
// SelectionVector: [1, 3, 4, 6]  (有效行的索引)
//
// 后续算子只处理 SelectionVector 指向的行
// 避免数据拷贝，提高效率

class SelectionVector {
    sel_t *sel_vector;  // 选中行的索引数组
};
```

**2. 字符串处理**

字符串是变长数据，不能像数值那样简单对齐：

```cpp
// DuckDB 的 string_t 设计
// 短字符串内联存储，长字符串使用指针
struct string_t {
    uint32_t length;
    union {
        char inlined[12];      // 短字符串 (≤12字节) 内联
        char *pointer;         // 长字符串使用指针
    } data;
};

// 优势:
// - 短字符串无需额外内存分配
// - 减少指针追踪
// - 保持良好的缓存局部性
```

**3. NULL 值处理**

需要高效地表示和传播 NULL 值：

```cpp
// ValidityMask: 位图表示 NULL
// 每一位表示对应位置是否有效 (1=有效, 0=NULL)
class ValidityMask {
    validity_t *validity;  // 64位整数数组

    inline bool RowIsValid(idx_t row_idx) const {
        idx_t entry_idx = row_idx / 64;
        idx_t bit_idx = row_idx % 64;
        return (validity[entry_idx] >> bit_idx) & 1;
    }
};

// 优化: 全部有效时可以使用 nullptr
// 避免不必要的位操作检查
```

**4. 复杂表达式处理**

嵌套表达式和 CASE WHEN 等需要特殊处理：

```cpp
// CASE WHEN 的向量化处理
// 需要分割向量，分别处理不同分支
//
// SELECT CASE WHEN x > 0 THEN x * 2 ELSE x * -1 END
//
// 1. 评估条件 x > 0，得到 SelectionVector
// 2. 用 true_sel 处理 x * 2
// 3. 用 false_sel 处理 x * -1
// 4. 合并结果
```

---

## 1.5 Pipeline 执行框架概述

DuckDB 的 Pipeline 是执行引擎的核心概念，它定义了数据如何流经算子链。

### 1.5.1 Pipeline 的定义

一个 Pipeline 是从 Source 到 Sink 的无中断数据流：

```
Pipeline = Source → [Operator₁ → Operator₂ → ... → Operatorₙ] → Sink

其中:
- Source: 产生数据的算子 (如 TableScan)
- Operators: 处理数据但不阻塞的算子 (如 Filter, Projection)
- Sink: 消费数据的算子 (如 HashJoin.Build, Aggregate)
```

### 1.5.2 Pipeline Breaker

某些算子需要收集所有输入数据后才能产生输出，这类算子称为 Pipeline Breaker：

```
Pipeline Breaker 类型:
┌────────────────────────────────────────────────────────────────┐
│ 算子                    │ 原因                                 │
├────────────────────────────────────────────────────────────────┤
│ Hash Join (Build)       │ 需要构建完整哈希表                    │
│ Hash Aggregate          │ 需要收集所有分组                      │
│ Sort                    │ 需要所有数据才能排序                  │
│ Window                  │ 需要完整分区数据                      │
└────────────────────────────────────────────────────────────────┘
```

**Pipeline 分割示例**：

```sql
SELECT a, SUM(b)
FROM t
WHERE c > 10
GROUP BY a
```

```
逻辑计划:
  Aggregate (GROUP BY a, SUM(b))
      └── Filter (c > 10)
              └── Scan (t)

Pipeline 分割:
  Pipeline 1: Scan → Filter → Aggregate.Build (Sink)
                                    ↓ (物化)
  Pipeline 2: Aggregate.Scan (Source) → ResultCollector (Sink)

执行顺序:
  1. Pipeline 1 完全执行完毕（所有数据聚合完成）
  2. Pipeline 2 开始执行（扫描聚合结果输出）
```

### 1.5.3 执行生命周期

Pipeline 执行的完整生命周期：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pipeline 执行生命周期                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Initialize (初始化)                                         │
│     ├── 创建 GlobalSinkState                                    │
│     ├── 创建 GlobalSourceState                                  │
│     └── 初始化所有算子状态                                       │
│                                                                  │
│  2. Execute (执行) - 可并行                                     │
│     ┌─────────────────────────────────────────┐                 │
│     │ Thread 1:                                │                 │
│     │   LocalSourceState, LocalSinkState       │                 │
│     │   GetData → Execute → Sink (循环)        │                 │
│     └─────────────────────────────────────────┘                 │
│     ┌─────────────────────────────────────────┐                 │
│     │ Thread 2:                                │                 │
│     │   LocalSourceState, LocalSinkState       │                 │
│     │   GetData → Execute → Sink (循环)        │                 │
│     └─────────────────────────────────────────┘                 │
│            ...                                                   │
│                                                                  │
│  3. Combine (合并) - 每个线程完成后调用                          │
│     └── 将 LocalSinkState 合并到 GlobalSinkState                │
│                                                                  │
│  4. Finalize (最终化) - 单线程                                  │
│     └── 完成 Sink 的最终处理                                     │
│                                                                  │
│  5. Complete (完成)                                              │
│     └── 通知依赖此 Pipeline 的其他 Pipeline                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.6 本章小结

本章介绍了 DuckDB 执行引擎的设计理念和整体架构：

1. **执行模型选择**：DuckDB 采用向量化执行模型，相比传统火山模型大幅减少解释开销，同时保持代码的简洁性和可维护性。

2. **Push vs Pull**：DuckDB 使用 Push-based 执行策略，数据由 Source 主动推送，天然支持并行执行和 Pipeline 架构。

3. **2048 行批处理**：选择 2048 作为标准向量大小是基于 CPU 缓存、SIMD 对齐、以及解释开销与物化开销平衡的综合考量。

4. **Pipeline 架构**：查询被分解为多个 Pipeline，Pipeline 之间通过 Sink/Source 连接，Pipeline 内部数据流式处理。

5. **核心组件**：Executor、Pipeline、PipelineExecutor、PhysicalOperator 等组件协作完成查询执行。

下一章我们将深入探讨向量化数据结构（Vector、DataChunk、SelectionVector）的具体实现。

---

## 源码参考

| 文件 | 描述 |
|------|------|
| `src/include/duckdb/execution/executor.hpp` | Executor 类定义 |
| `src/parallel/executor.cpp` | Executor 实现 |
| `src/include/duckdb/parallel/pipeline.hpp` | Pipeline 类定义 |
| `src/parallel/pipeline.cpp` | Pipeline 实现 |
| `src/include/duckdb/parallel/pipeline_executor.hpp` | PipelineExecutor 定义 |
| `src/parallel/pipeline_executor.cpp` | PipelineExecutor 实现 |
| `src/include/duckdb/execution/physical_operator.hpp` | PhysicalOperator 基类 |
| `src/include/duckdb/execution/execution_context.hpp` | ExecutionContext 定义 |
| `src/include/duckdb/common/vector_size.hpp` | STANDARD_VECTOR_SIZE 定义 |

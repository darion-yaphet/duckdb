# 第八章：Pipeline 并行执行

## 8.1 概述

Pipeline 是 DuckDB 查询执行的核心并行化单元。一个 Pipeline 代表从数据源 (source) 到数据汇 (sink) 的数据流，中间可能包含多个算子 (operators)。DuckDB 通过多个 PipelineTask 并行执行同一个 Pipeline，实现 intra-pipeline 并行；同时通过 MetaPipeline 管理多个 Pipeline 之间的依赖关系，实现 inter-pipeline 并行。

```
Pipeline 并行执行模型
====================

                    ┌─────────────────────────────────────────────────────────┐
                    │                     MetaPipeline                        │
                    │  (管理共享相同 Sink 的多个 Pipeline)                     │
                    └─────────────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    ▼                       ▼                       ▼
            ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
            │  Pipeline 1 │         │  Pipeline 2 │         │  Pipeline 3 │
            │  (base)     │         │  (union)    │         │  (child)    │
            └─────────────┘         └─────────────┘         └─────────────┘
                    │                       │                       │
        ┌───────────┼───────────┐           │                       │
        ▼           ▼           ▼           ▼                       ▼
   ┌────────┐ ┌────────┐ ┌────────┐   ┌────────┐               ┌────────┐
   │ Task 1 │ │ Task 2 │ │ Task N │   │ Task 1 │               │ Task 1 │
   └────────┘ └────────┘ └────────┘   └────────┘               └────────┘
        │           │           │           │                       │
        ▼           ▼           ▼           ▼                       ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                         TaskScheduler                                 │
   │                    (ConcurrentQueue + Workers)                        │
   └──────────────────────────────────────────────────────────────────────┘
```

## 8.2 Pipeline 核心结构

### 8.2.1 Pipeline 类定义

Pipeline 封装了执行计划中一段连续的算子链：

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class Pipeline : public enable_shared_from_this<Pipeline> {
    friend class Executor;
    friend class PipelineExecutor;
    friend class PipelineEvent;
    friend class MetaPipeline;

public:
    Executor &executor;

private:
    //! 是否已准备就绪
    bool ready;
    //! 是否已初始化
    atomic<bool> initialized;

    //! 数据源算子
    optional_ptr<PhysicalOperator> source;
    //! 中间算子链
    vector<reference<PhysicalOperator>> operators;
    //! 数据汇算子
    optional_ptr<PhysicalOperator> sink;

    //! 全局源状态
    unique_ptr<GlobalSourceState> source_state;

    //! 父 Pipeline（依赖此 Pipeline 完成的）
    vector<weak_ptr<Pipeline>> parents;
    //! 依赖的 Pipeline
    vector<weak_ptr<Pipeline>> dependencies;

    //! 批次索引管理
    idx_t base_batch_index = 0;
    mutex batch_lock;
    multiset<idx_t> batch_indexes;
};
```

### 8.2.2 Pipeline 结构示意

```
Pipeline 内部结构
=================

    ┌─────────────────────────────────────────────────────────────────┐
    │                          Pipeline                                │
    │                                                                  │
    │  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐     │
    │  │  Source  │──▶│ Operator │──▶│ Operator │──▶│   Sink   │     │
    │  │ (scan)   │   │ (filter) │   │ (project)│   │ (build)  │     │
    │  └──────────┘   └──────────┘   └──────────┘   └──────────┘     │
    │       │                                              │          │
    │       ▼                                              ▼          │
    │  source_state                                   sink_state      │
    │  (GlobalSourceState)                       (GlobalSinkState)    │
    │                                                                  │
    │  ┌────────────────────────────────────────────────────────────┐ │
    │  │                   Parallel Execution                       │ │
    │  │  Task 1: local_source_state ──▶ ... ──▶ local_sink_state   │ │
    │  │  Task 2: local_source_state ──▶ ... ──▶ local_sink_state   │ │
    │  │  Task N: local_source_state ──▶ ... ──▶ local_sink_state   │ │
    │  └────────────────────────────────────────────────────────────┘ │
    └─────────────────────────────────────────────────────────────────┘
```

## 8.3 Pipeline 调度

### 8.3.1 调度入口

Pipeline 调度的入口是 `Schedule` 方法，它首先尝试并行调度，如果不可行则回退到顺序执行：

```cpp
// src/parallel/pipeline.cpp
void Pipeline::Schedule(shared_ptr<Event> &event) {
    D_ASSERT(ready);
    D_ASSERT(sink);
    Reset();
    if (!ScheduleParallel(event)) {
        // 无法并行化：创建顺序任务
        ScheduleSequentialTask(event);
    }
}

void Pipeline::ScheduleSequentialTask(shared_ptr<Event> &event) {
    vector<shared_ptr<Task>> tasks;
    tasks.push_back(make_uniq<PipelineTask>(*this, event));
    event->SetTasks(std::move(tasks));
}
```

### 8.3.2 并行调度决策

`ScheduleParallel` 方法检查整个 Pipeline 是否支持并行执行：

```cpp
// src/parallel/pipeline.cpp
bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // 1. 检查 sink 是否支持并行
    if (!sink->ParallelSink()) {
        return false;
    }

    // 2. 检查 source 是否支持并行
    if (!source->ParallelSource()) {
        return false;
    }

    // 3. 计算最大线程数（从 source 开始）
    auto max_threads = source_state->MaxThreads();

    // 4. 检查中间算子是否支持并行
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (!op.ParallelOperator()) {
            return false;
        }
        // 每个算子可能限制并行度
        max_threads = MinValue<idx_t>(max_threads,
                                       op.op_state->MaxThreads(max_threads));
    }

    // 5. 检查分区要求
    auto partition_info = sink->RequiredPartitionInfo();
    if (partition_info.batch_index) {
        if (!source->SupportsPartitioning(OperatorPartitionInfo::BatchIndex())) {
            throw InternalException("Sink requires batch index but source does not support it");
        }
    }

    // 6. 限制为实际可用线程数
    auto &scheduler = TaskScheduler::GetScheduler(executor.context);
    auto active_threads = NumericCast<idx_t>(scheduler.NumberOfThreads());
    if (max_threads > active_threads) {
        max_threads = active_threads;
    }

    // 7. sink 也可能限制并行度
    if (sink && sink->sink_state) {
        max_threads = sink->sink_state->MaxThreads(max_threads);
    }
    if (max_threads > active_threads) {
        max_threads = active_threads;
    }

    return LaunchScanTasks(event, max_threads);
}
```

### 8.3.3 线程数决策流程

```
MaxThreads 计算流程
==================

                          Source
                            │
                            ▼
            ┌───────────────────────────────┐
            │ source_state->MaxThreads()    │
            │ (例如：表有 1000 行分块)       │
            │ 返回：max_threads = 10        │
            └───────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────┐
            │ Operator 1: ParallelOperator? │
            │ op_state->MaxThreads(10)      │
            │ 返回：min(10, 10) = 10        │
            └───────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────┐
            │ Operator 2: ParallelOperator? │
            │ op_state->MaxThreads(10)      │
            │ 返回：min(10, 8) = 8          │
            └───────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────┐
            │ Scheduler.NumberOfThreads()   │
            │ 系统可用：4 线程               │
            │ 返回：min(8, 4) = 4           │
            └───────────────────────────────┘
                            │
                            ▼
            ┌───────────────────────────────┐
            │ sink_state->MaxThreads(4)     │
            │ 返回：min(4, 4) = 4           │
            └───────────────────────────────┘
                            │
                            ▼
                  LaunchScanTasks(4)
                  创建 4 个 PipelineTask
```

### 8.3.4 启动并行任务

```cpp
// src/parallel/pipeline.cpp
bool Pipeline::LaunchScanTasks(shared_ptr<Event> &event, idx_t max_threads) {
    // 如果只有一个线程，回退到顺序执行
    if (max_threads <= 1) {
        return false;
    }

    // 为每个线程创建一个 PipelineTask
    vector<shared_ptr<Task>> tasks;
    for (idx_t i = 0; i < max_threads; i++) {
        tasks.push_back(make_uniq<PipelineTask>(*this, event));
    }
    event->SetTasks(std::move(tasks));
    return true;
}
```

## 8.4 PipelineTask 执行

### 8.4.1 PipelineTask 定义

PipelineTask 继承自 ExecutorTask，代表 Pipeline 中的一个并行执行单元：

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class PipelineTask : public ExecutorTask {
    static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

public:
    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

    string TaskType() const override {
        return "PipelineTask";
    }

    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override;
    bool TaskBlockedOnResult() const override;
};
```

### 8.4.2 任务执行逻辑

```cpp
// src/parallel/pipeline.cpp
TaskExecutionResult PipelineTask::ExecuteTask(TaskExecutionMode mode) {
    // 1. 延迟创建 PipelineExecutor（每个 Task 独有）
    if (!pipeline_executor) {
        pipeline_executor = make_uniq<PipelineExecutor>(
            pipeline.GetClientContext(), pipeline);
    }

    // 2. 设置中断处理
    pipeline_executor->SetTaskForInterrupts(shared_from_this());

    // 3. 执行 Pipeline
    if (mode == TaskExecutionMode::PROCESS_PARTIAL) {
        // 部分执行模式：处理 50 个 chunk 后返回
        auto res = pipeline_executor->Execute(PARTIAL_CHUNK_COUNT);

        switch (res) {
        case PipelineExecuteResult::NOT_FINISHED:
            return TaskExecutionResult::TASK_NOT_FINISHED;
        case PipelineExecuteResult::INTERRUPTED:
            return TaskExecutionResult::TASK_BLOCKED;
        case PipelineExecuteResult::FINISHED:
            break;
        }
    } else {
        // 完全执行模式：执行直到完成
        auto res = pipeline_executor->Execute();
        switch (res) {
        case PipelineExecuteResult::NOT_FINISHED:
            throw InternalException("Execute without limit should not return NOT_FINISHED");
        case PipelineExecuteResult::INTERRUPTED:
            return TaskExecutionResult::TASK_BLOCKED;
        case PipelineExecuteResult::FINISHED:
            break;
        }
    }

    // 4. 完成后通知 Event
    event->FinishTask();
    pipeline_executor.reset();
    return TaskExecutionResult::TASK_FINISHED;
}
```

### 8.4.3 任务执行状态机

```
PipelineTask 执行状态机
======================

          ┌────────────────────────────────────┐
          │              CREATED               │
          │        (pipeline_executor=null)    │
          └────────────────────────────────────┘
                          │
                          │ ExecuteTask() 首次调用
                          ▼
          ┌────────────────────────────────────┐
          │            EXECUTING               │
          │      创建 PipelineExecutor         │
          │      设置 InterruptState           │
          └────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │NOT_FINISH│   │INTERRUPTED│  │ FINISHED │
    │  (部分)  │   │  (阻塞)   │  │ (完成)   │
    └──────────┘   └──────────┘   └──────────┘
         │              │              │
         │              │              │
         ▼              ▼              ▼
   TASK_NOT_      TASK_BLOCKED    TASK_FINISHED
   FINISHED           │               │
         │              │              │
         │              │              ▼
         │              │      event->FinishTask()
         │              │              │
         └──────────────┴──────────────┘
                        │
                        ▼
              重新入队等待调度
```

## 8.5 PipelineExecutor 执行器

### 8.5.1 执行器状态

PipelineExecutor 管理单个任务的执行状态：

```cpp
// src/include/duckdb/parallel/pipeline_executor.hpp
class PipelineExecutor {
private:
    Pipeline &pipeline;
    ThreadContext thread;
    ExecutionContext context;

    // 中间数据块和状态
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    vector<unique_ptr<OperatorState>> intermediate_states;

    // 本地状态（每个 Task 独有）
    unique_ptr<LocalSourceState> local_source_state;
    unique_ptr<LocalSinkState> local_sink_state;
    InterruptState interrupt_state;

    DataChunk final_chunk;

    // 执行状态跟踪
    stack<idx_t> in_process_operators;  // 有剩余输出的算子
    bool finalized = false;
    int32_t finished_processing_idx = -1;
    bool exhausted_pipeline = false;
    bool started_flushing = false;
    bool done_flushing = false;
    bool remaining_sink_chunk = false;
    bool next_batch_blocked = false;
};
```

### 8.5.2 执行主循环

```cpp
// src/parallel/pipeline_executor.cpp
PipelineExecuteResult PipelineExecutor::Execute(idx_t max_chunks) {
    D_ASSERT(pipeline.sink);
    auto &source_chunk = pipeline.operators.empty() ? final_chunk : *intermediate_chunks[0];
    ExecutionBudget chunk_budget(max_chunks);

    do {
        if (context.client.interrupted) {
            throw InterruptException();
        }

        OperatorResultType result;

        // 检查当前状态决定下一步操作
        if (exhausted_pipeline && done_flushing && !remaining_sink_chunk &&
            !next_batch_blocked && in_process_operators.empty()) {
            // 全部完成
            break;
        } else if (remaining_sink_chunk) {
            // Sink 之前被中断，重试
            result = ExecutePushInternal(final_chunk, chunk_budget);
            remaining_sink_chunk = false;
        } else if (!in_process_operators.empty() && !started_flushing) {
            // 有算子返回了 HAVE_MORE_OUTPUT
            result = ExecutePushInternal(source_chunk, chunk_budget);
        } else if (exhausted_pipeline && !next_batch_blocked && !done_flushing) {
            // Source 耗尽，刷新缓存算子
            auto flush_completed = TryFlushCachingOperators(chunk_budget);
            if (flush_completed) {
                done_flushing = true;
                break;
            } else {
                // flush 被阻塞或预算耗尽
                return remaining_sink_chunk ? PipelineExecuteResult::INTERRUPTED
                                            : PipelineExecuteResult::NOT_FINISHED;
            }
        } else if (!exhausted_pipeline || next_batch_blocked) {
            // 正常路径：从 source 获取数据
            source_chunk.Reset();
            auto source_result = FetchFromSource(source_chunk);
            if (source_result == SourceResultType::BLOCKED) {
                return PipelineExecuteResult::INTERRUPTED;
            }
            if (source_result == SourceResultType::FINISHED) {
                exhausted_pipeline = true;
            }

            // 处理批次索引
            if (required_partition_info.AnyRequired()) {
                auto next_batch_result = NextBatch(source_chunk, ...);
                if (next_batch_result == SinkNextBatchType::BLOCKED) {
                    return PipelineExecuteResult::INTERRUPTED;
                }
            }

            result = ExecutePushInternal(source_chunk, chunk_budget);
        }

        // 处理 Sink 中断
        if (result == OperatorResultType::BLOCKED) {
            remaining_sink_chunk = true;
            return PipelineExecuteResult::INTERRUPTED;
        }

        if (result == OperatorResultType::FINISHED) {
            exhausted_pipeline = true;
            break;
        }
    } while (chunk_budget.Next());

    // 检查是否真正完成
    if ((!exhausted_pipeline || !done_flushing) && !IsFinished()) {
        return PipelineExecuteResult::NOT_FINISHED;
    }

    return PushFinalize();
}
```

### 8.5.3 数据推送流程

```
ExecutePushInternal 数据流
=========================

Source                   Operator 1              Operator 2              Sink
  │                          │                       │                    │
  │   intermediate_chunks[0] │  intermediate_chunks[1]│    final_chunk    │
  │          │               │           │            │         │         │
  ▼          ▼               ▼           ▼            ▼         ▼         ▼
┌────┐    ┌────┐          ┌────┐      ┌────┐       ┌────┐    ┌────┐    ┌────┐
│Get │───▶│Chk0│─────────▶│Exec│─────▶│Chk1│──────▶│Exec│───▶│Final│──▶│Sink│
│Data│    └────┘          └────┘      └────┘       └────┘    └────┘    └────┘
└────┘                                                                    │
                                                                          ▼
循环执行直到：                                              BLOCKED  FINISHED  CONTINUE
1. Source 返回 FINISHED                                        │        │        │
2. Sink 返回 BLOCKED                                           │        │        │
3. chunk_budget 耗尽                                           ▼        ▼        ▼
4. 中间算子返回 FINISHED                                   中断返回  完成返回  继续循环
```

## 8.6 MetaPipeline 协调

### 8.6.1 MetaPipeline 概念

MetaPipeline 管理一组共享相同 Sink 的 Pipeline：

```cpp
// src/include/duckdb/parallel/meta_pipeline.hpp
enum class MetaPipelineType : uint8_t {
    REGULAR = 0,    // 普通 MetaPipeline
    JOIN_BUILD = 1  // Join 构建端
};

class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
private:
    Executor &executor;
    PipelineBuildState &state;
    optional_ptr<Pipeline> parent;
    optional_ptr<PhysicalOperator> sink;
    MetaPipelineType type;
    bool recursive_cte;

    //! 共享相同 sink 的所有 Pipeline
    vector<shared_ptr<Pipeline>> pipelines;
    //! Pipeline 间的依赖关系
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
    //! 子 MetaPipeline
    vector<shared_ptr<MetaPipeline>> children;
    //! 下一个批次索引
    idx_t next_batch_index;
};
```

### 8.6.2 MetaPipeline 层次结构

```
MetaPipeline 层次示例（Hash Join）
================================

                        ┌─────────────────────────────────┐
                        │        Root MetaPipeline        │
                        │     sink = Result Collector     │
                        └─────────────────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
        ┌───────────────────────┐               ┌───────────────────────┐
        │   Base Pipeline       │               │  Child MetaPipeline   │
        │   (Probe Side)        │               │  (Build Side)         │
        │   Scan T1 ──▶ Join    │               │  type = JOIN_BUILD    │
        └───────────────────────┘               └───────────────────────┘
                    │                                       │
                    │ depends on                            ▼
                    │                           ┌───────────────────────┐
                    └──────────────────────────▶│   Build Pipeline      │
                                                │   Scan T2 ──▶ HT Build│
                                                └───────────────────────┘

执行顺序：
1. Build Pipeline 先执行（构建哈希表）
2. Base Pipeline 后执行（探测哈希表）
```

### 8.6.3 创建子 MetaPipeline

```cpp
// src/parallel/meta_pipeline.cpp
MetaPipeline &MetaPipeline::CreateChildMetaPipeline(Pipeline &current,
                                                     PhysicalOperator &op,
                                                     MetaPipelineType type) {
    // 1. 创建新的 MetaPipeline
    children.push_back(make_shared_ptr<MetaPipeline>(executor, state, &op, type));
    auto &child_meta_pipeline = *children.back().get();

    // 2. 记录父 Pipeline
    child_meta_pipeline.parent = &current;

    // 3. 建立依赖：current 依赖于 child 的 base pipeline
    current.AddDependency(child_meta_pipeline.GetBasePipeline());

    // 4. 继承 recursive CTE 标记
    child_meta_pipeline.recursive_cte = recursive_cte;

    return child_meta_pipeline;
}
```

### 8.6.4 Union Pipeline 创建

对于 UNION ALL 操作，创建共享 sink 的多个 Pipeline：

```cpp
// src/parallel/meta_pipeline.cpp
Pipeline &MetaPipeline::CreateUnionPipeline(Pipeline &current, bool order_matters) {
    // 1. 创建新 Pipeline
    auto &union_pipeline = CreatePipeline();
    state.SetPipelineOperators(union_pipeline, state.GetPipelineOperators(current));
    state.SetPipelineSink(union_pipeline, sink, 0);

    // 2. 继承 current 的所有依赖
    union_pipeline.dependencies = current.dependencies;
    auto it = pipeline_dependencies.find(current);
    if (it != pipeline_dependencies.end()) {
        pipeline_dependencies[union_pipeline] = it->second;
    }

    // 3. 如果需要保持顺序，添加对 current 的依赖
    if (order_matters) {
        pipeline_dependencies[union_pipeline].push_back(current);
    }

    return union_pipeline;
}
```

## 8.7 批次索引管理

### 8.7.1 批次索引用途

批次索引 (batch index) 用于在并行执行时保持输出顺序：

```cpp
// src/include/duckdb/parallel/pipeline.hpp
class PipelineBuildState {
public:
    // 批次索引增量：10万亿，确保不同 Pipeline 的批次索引不重叠
    constexpr static idx_t BATCH_INCREMENT = 10000000000000;
};

// Pipeline 成员
class Pipeline {
    idx_t base_batch_index = 0;    // 基础批次索引
    mutex batch_lock;               // 保护 batch_indexes
    multiset<idx_t> batch_indexes;  // 当前活跃的批次索引
};
```

### 8.7.2 批次索引操作

```cpp
// src/parallel/pipeline.cpp
idx_t Pipeline::RegisterNewBatchIndex() {
    lock_guard<mutex> l(batch_lock);
    // 获取当前最小批次索引
    idx_t minimum = batch_indexes.empty() ? base_batch_index : *batch_indexes.begin();
    // 插入作为占位符
    batch_indexes.insert(minimum);
    return minimum;
}

idx_t Pipeline::UpdateBatchIndex(idx_t old_index, idx_t new_index) {
    lock_guard<mutex> l(batch_lock);
    // 验证新索引不小于最小值
    if (new_index < *batch_indexes.begin()) {
        throw InternalException("Processing batch index %llu, but previous min was %llu",
                                new_index, *batch_indexes.begin());
    }
    // 找到并替换旧索引
    auto entry = batch_indexes.find(old_index);
    if (entry == batch_indexes.end()) {
        throw InternalException("Batch index %llu was not found", old_index);
    }
    batch_indexes.erase(entry);
    batch_indexes.insert(new_index);
    // 返回新的最小批次索引
    return *batch_indexes.begin();
}
```

### 8.7.3 批次索引并发访问

```
批次索引并发管理
===============

Pipeline (base_batch_index = 1000)
         │
         │ batch_lock 保护
         ▼
    batch_indexes (multiset)
    ┌────────────────────────────┐
    │  [1000, 1003, 1005, 1007]  │
    │     ▲                      │
    │     │                      │
    │   最小值                    │
    └────────────────────────────┘
         ▲           ▲           ▲
         │           │           │
    ┌────────┐  ┌────────┐  ┌────────┐
    │ Task 1 │  │ Task 2 │  │ Task 3 │
    │idx=1000│  │idx=1003│  │idx=1005│
    └────────┘  └────────┘  └────────┘
         │
         │ UpdateBatchIndex(1000, 1010)
         ▼
    batch_indexes 变为:
    [1003, 1005, 1007, 1010]
         ▲
         │
      新最小值

目的：Sink 可以按批次索引顺序输出数据
```

## 8.8 顺序保持

### 8.8.1 顺序依赖检测

```cpp
// src/parallel/pipeline.cpp
bool Pipeline::IsOrderDependent() const {
    // 1. 检查 source 的顺序要求
    if (source) {
        auto source_order = source->SourceOrder();
        if (source_order == OrderPreservationType::FIXED_ORDER) {
            return true;  // source 要求固定顺序
        }
        if (source_order == OrderPreservationType::NO_ORDER) {
            return false; // source 明确无顺序要求
        }
    }

    // 2. 检查中间算子
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (op.OperatorOrder() == OrderPreservationType::NO_ORDER) {
            return false;
        }
        if (op.OperatorOrder() == OrderPreservationType::FIXED_ORDER) {
            return true;
        }
    }

    // 3. 检查全局设置
    if (!DBConfig::GetSetting<PreserveInsertionOrderSetting>(executor.context)) {
        return false;
    }

    // 4. 检查 sink
    if (sink && sink->SinkOrderDependent()) {
        return true;
    }

    return false;
}
```

### 8.8.2 顺序保持策略

```
顺序保持策略
===========

策略 1：顺序执行
━━━━━━━━━━━━━━━
如果无法并行保持顺序，使用 ScheduleSequentialTask
只创建一个 PipelineTask

策略 2：批次索引
━━━━━━━━━━━━━━━
并行执行但按批次索引排序输出
┌──────────────────────────────────────────────┐
│ Task 1 ─▶ batch 1000  ─┐                     │
│ Task 2 ─▶ batch 1001  ─┼─▶ OrderedSink       │
│ Task 3 ─▶ batch 1002  ─┘   (按批次排序后输出) │
└──────────────────────────────────────────────┘

策略 3：Pipeline 依赖
━━━━━━━━━━━━━━━━━━━
Union Pipeline 在需要顺序时建立依赖
Pipeline 1 完成 ─▶ Pipeline 2 开始 ─▶ Pipeline 3 开始
```

## 8.9 Pipeline 初始化

### 8.9.1 Reset 初始化

```cpp
// src/parallel/pipeline.cpp
void Pipeline::Reset() {
    // 1. 初始化 Sink 状态
    ResetSink();

    // 2. 初始化中间算子状态
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        lock_guard<mutex> guard(op.lock);
        if (!op.op_state) {
            op.op_state = op.GetGlobalOperatorState(GetClientContext());
        }
    }

    // 3. 初始化 Source 状态
    ResetSource(false);

    initialized = true;
}

void Pipeline::ResetSink() {
    if (sink) {
        if (!sink->IsSink()) {
            throw InternalException("Sink of pipeline does not have IsSink set");
        }
        lock_guard<mutex> guard(sink->lock);
        if (!sink->sink_state) {
            sink->sink_state = sink->GetGlobalSinkState(GetClientContext());
        }
    }
}
```

### 8.9.2 状态层次

```
Pipeline 状态层次
=================

Global State (Pipeline 共享)
┌──────────────────────────────────────────────────────────────────┐
│                          Pipeline                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ source_state    │  │ op.op_state     │  │ sink.sink_state │  │
│  │ (GlobalSource)  │  │ (GlobalOperator)│  │ (GlobalSink)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

Local State (每个 Task 独有)
┌──────────────────────────────────────────────────────────────────┐
│                      PipelineExecutor                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │local_source_state│ │intermediate_     │  │local_sink_state │  │
│  │(LocalSource)     │ │states[i]        │  │(LocalSink)      │  │
│  └─────────────────┘  │(OperatorState)  │  └─────────────────┘  │
│                       └─────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘

线程安全：
- Global State 创建时使用算子的 lock 保护
- Local State 每个 Task 独有，无需同步
- 并行任务通过 Local State 隔离
```

## 8.10 PipelineExecutor 生命周期

### 8.10.1 构造函数

```cpp
// src/parallel/pipeline_executor.cpp
PipelineExecutor::PipelineExecutor(ClientContext &context_p, Pipeline &pipeline_p)
    : pipeline(pipeline_p), thread(context_p), context(context_p, thread, &pipeline_p) {

    // 1. 初始化 Sink 本地状态
    if (pipeline.sink) {
        local_sink_state = pipeline.sink->GetLocalSinkState(context);
        required_partition_info = pipeline.sink->RequiredPartitionInfo();

        // 如果需要分区信息，初始化批次索引
        if (required_partition_info.AnyRequired()) {
            auto &partition_info = local_sink_state->partition_info;
            partition_info.batch_index = pipeline.RegisterNewBatchIndex();
            partition_info.min_batch_index = partition_info.batch_index;
        }
    }

    // 2. 初始化 Source 本地状态
    local_source_state = pipeline.source->GetLocalSourceState(context, *pipeline.source_state);

    // 3. 为每个中间算子创建数据块和状态
    intermediate_chunks.reserve(pipeline.operators.size());
    intermediate_states.reserve(pipeline.operators.size());

    for (idx_t i = 0; i < pipeline.operators.size(); i++) {
        auto &prev_operator = i == 0 ? *pipeline.source : pipeline.operators[i - 1].get();
        auto &current_operator = pipeline.operators[i].get();

        // 创建中间数据块
        auto chunk = make_uniq<DataChunk>();
        chunk->Initialize(BufferAllocator::Get(context.client), prev_operator.GetTypes());
        intermediate_chunks.push_back(std::move(chunk));

        // 创建算子本地状态
        auto op_state = current_operator.GetOperatorState(context);
        intermediate_states.push_back(std::move(op_state));

        // 检查是否可以提前终止
        if (current_operator.IsSink() &&
            current_operator.sink_state->state == SinkFinalizeType::NO_OUTPUT_POSSIBLE) {
            FinishProcessing();
        }
    }

    // 4. 初始化最终数据块
    InitializeChunk(final_chunk);
}
```

### 8.10.2 完成处理

```cpp
// src/parallel/pipeline_executor.cpp
PipelineExecuteResult PipelineExecutor::PushFinalize() {
    if (finalized) {
        throw InternalException("Calling PushFinalize on a pipeline that has been finalized");
    }

    // 1. 调用 Sink 的 Combine
    OperatorSinkCombineInput combine_input {*pipeline.sink->sink_state,
                                            *local_sink_state,
                                            interrupt_state};
    auto result = pipeline.sink->Combine(context, combine_input);

    if (result == SinkCombineResultType::BLOCKED) {
        return PipelineExecuteResult::INTERRUPTED;
    }

    finalized = true;

    // 2. 刷新所有性能分析信息
    for (idx_t i = 0; i < intermediate_states.size(); i++) {
        intermediate_states[i]->Finalize(pipeline.operators[i].get(), context);
    }

    // 3. 刷新线程上下文
    pipeline.executor.Flush(thread);

    // 4. 清理本地状态
    local_sink_state.reset();

    return PipelineExecuteResult::FINISHED;
}
```

## 8.11 并行执行时序

### 8.11.1 完整执行流程

```
Pipeline 并行执行时序
===================

Main Thread                    Worker Thread 1              Worker Thread 2
     │                              │                            │
     │  Pipeline::Schedule()        │                            │
     │  └─ ScheduleParallel()       │                            │
     │      └─ LaunchScanTasks(N)   │                            │
     │          └─ SetTasks()       │                            │
     │                              │                            │
     ├──────────────────────────────┼────────────────────────────┤
     │        Tasks 入队             │                            │
     │                              │                            │
     │                         ┌────┴────┐                  ┌────┴────┐
     │                         │DequeueTask                 │DequeueTask
     │                         │ PipelineTask 1             │ PipelineTask 2
     │                         └────┬────┘                  └────┬────┘
     │                              │                            │
     │                         ExecuteTask()                ExecuteTask()
     │                              │                            │
     │                    ┌─────────┴─────────┐         ┌────────┴────────┐
     │                    │创建 PipelineExecutor│        │创建 PipelineExecutor│
     │                    │local_source_state  │        │local_source_state  │
     │                    │local_sink_state    │        │local_sink_state    │
     │                    └─────────┬─────────┘         └────────┬────────┘
     │                              │                            │
     │                    ┌─────────┴─────────┐         ┌────────┴────────┐
     │                    │Execute() 循环:     │        │Execute() 循环:    │
     │                    │ FetchFromSource   │        │ FetchFromSource   │
     │                    │ ExecutePushInternal│       │ ExecutePushInternal│
     │                    │ Sink              │        │ Sink              │
     │                    └─────────┬─────────┘         └────────┬────────┘
     │                              │                            │
     │                    ┌─────────┴─────────┐         ┌────────┴────────┐
     │                    │PushFinalize()     │        │PushFinalize()    │
     │                    │ Combine           │        │ Combine          │
     │                    └─────────┬─────────┘         └────────┬────────┘
     │                              │                            │
     │                    event->FinishTask()          event->FinishTask()
     │                              │                            │
     │◀─────────────────────────────┴────────────────────────────┘
     │                     所有 Task 完成
     │                     Event 完成
     ▼
```

### 8.11.2 数据流隔离

```
并行任务数据隔离
===============

                            Global State
                    ┌─────────────────────────┐
                    │     source_state        │
                    │     (分块信息)           │
                    │                         │
                    │     sink_state          │
                    │     (汇总状态)           │
                    └─────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
    ┌─────────┐           ┌─────────┐           ┌─────────┐
    │ Task 1  │           │ Task 2  │           │ Task 3  │
    ├─────────┤           ├─────────┤           ├─────────┤
    │local_   │           │local_   │           │local_   │
    │source_  │           │source_  │           │source_  │
    │state    │           │state    │           │state    │
    │         │           │         │           │         │
    │inter-   │           │inter-   │           │inter-   │
    │mediate_ │           │mediate_ │           │mediate_ │
    │chunks   │           │chunks   │           │chunks   │
    │         │           │         │           │         │
    │local_   │           │local_   │           │local_   │
    │sink_    │           │sink_    │           │sink_    │
    │state    │           │state    │           │state    │
    └─────────┘           └─────────┘           └─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
                    ┌─────────────────────────┐
                    │     Sink::Combine()     │
                    │     合并所有 local_sink  │
                    └─────────────────────────┘

线程安全保证：
1. 每个 Task 有独立的 local_*_state
2. intermediate_chunks 每个 Task 独有
3. 只有 Global State 需要同步访问
4. Combine 在 Task 完成后顺序调用
```

## 8.12 小结

Pipeline 并行执行是 DuckDB 高性能查询处理的核心机制：

1. **Pipeline 结构**：source → operators → sink 的线性数据流
2. **并行调度**：ScheduleParallel 检查所有算子是否支持并行，计算最大线程数
3. **PipelineTask**：每个并行任务有独立的 PipelineExecutor 和 local state
4. **MetaPipeline**：管理多个共享 sink 的 Pipeline，处理 Join、Union 等复杂场景
5. **批次索引**：通过 batch_indexes multiset 实现并行执行下的顺序保持
6. **状态隔离**：Global State 共享，Local State 隔离，确保线程安全
7. **中断支持**：通过 InterruptState 支持异步阻塞和恢复

关键并发控制点：
- `batch_lock` 保护批次索引更新
- 算子的 `lock` 保护 Global State 创建
- 每个 Task 的 Local State 无需同步
- Event 完成计数使用原子操作

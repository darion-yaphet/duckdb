# DuckDB 查询优化器深度解析：系列目录

## 系列概述

本系列深入剖析 DuckDB 的查询优化器，从逻辑计划优化到物理计划生成的完整优化过程。涵盖表达式重写、谓词优化、Join 顺序优化、基数估计、代价模型等核心组件。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DuckDB 优化器架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  逻辑计划输入 (LogicalOperator Tree)                                │
│     │                                                               │
│     ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第一章：优化器架构与表达式重写                                │   │
│  │ Optimizer、ExpressionRewriter、Rule System                  │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第二章：谓词优化                                              │   │
│  │ FilterPushdown、FilterPullup、FilterCombiner                │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第三章：列与投影优化                                          │   │
│  │ RemoveUnusedColumns、ColumnLifetime、LateMaterialization    │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第四章：Join 顺序优化                                         │   │
│  │ JoinOrderOptimizer、QueryGraph、PlanEnumerator              │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第五章：基数估计与代价模型                                    │   │
│  │ CardinalityEstimator、CostModel、Statistics                 │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第六章：子查询与 CTE 优化                                     │   │
│  │ Deliminator、CTEInlining、CommonSubplanOptimizer            │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第七章：TopN 与窗口函数优化                                   │   │
│  │ TopNOptimizer、TopNWindowElimination、LimitPushdown         │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第八章：Join 实现优化                                         │   │
│  │ BuildProbeSide、JoinElimination、JoinFilterPushdown         │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│                           ▼                                         │
│  优化后的逻辑计划                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 第一章：优化器架构与表达式重写

### 1.1 Optimizer 类设计
- 优化器入口和生命周期
- 优化阶段顺序
- 优化器禁用机制 (`OptimizerDisabled`)
- 扩展点 (`optimizer_extensions`)

### 1.2 ExpressionRewriter 框架
- `ExpressionRewriter` 类设计
- 规则注册与执行
- 递归表达式遍历

### 1.3 Rule 抽象与模式匹配
- `Rule` 基类设计
- `ExpressionMatcher` 模式匹配
- 规则优先级

### 1.4 常量折叠 (ConstantFoldingRule)
- 编译时常量计算
- 函数求值
- NULL 传播

### 1.5 算术简化 (ArithmeticSimplificationRule)
- `x + 0 → x`
- `x * 1 → x`
- `x * 0 → 0`
- 加法/乘法结合律

### 1.6 比较简化 (ComparisonSimplificationRule)
- `x = x → TRUE`
- `x <> x → FALSE`
- 常量比较优化

### 1.7 逻辑简化 (ConjunctionSimplificationRule)
- `TRUE AND x → x`
- `FALSE OR x → x`
- 重复条件消除

### 1.8 CASE 简化 (CaseSimplificationRule)
- 常量条件消除
- 单分支优化
- COALESCE 转换

### 1.9 LIKE 优化 (LikeOptimizationRule)
- 前缀匹配优化
- 后缀匹配优化
- 无通配符转换为等值

### 1.10 正则表达式优化 (RegexOptimizationRule)
- 简单模式转换为 LIKE
- 锚点优化
- 字符类简化

### 1.11 日期函数简化
- `DatePartSimplificationRule`
- `DateTruncSimplificationRule`
- 时间戳比较优化

### 1.12 其他表达式规则
- `DistributivityRule` (分配律)
- `MoveConstantsRule` (常量移动)
- `InClauseSimplificationRule`
- `EqualOrNullSimplification`
- `EmptyNeedleRemovalRule`
- `EnumComparisonRule`

---

## 第二章：谓词优化

### 2.1 谓词优化概述
- 谓词下推 vs 谓词上拉
- 优化目标与收益
- 谓词等价类

### 2.2 FilterPushdown 架构
- `FilterPushdown` 类设计
- 下推策略选择
- 不可下推条件处理

### 2.3 跨算子下推
- `PushdownInnerJoin` - 内连接下推
- `PushdownLeftJoin` - 左连接下推
- `PushdownOuterJoin` - 外连接下推
- `PushdownSemiAntiJoin` - Semi/Anti Join 下推
- `PushdownMarkJoin` - Mark Join 下推

### 2.4 其他算子下推
- `PushdownAggregate` - 聚合下推
- `PushdownProjection` - 投影下推
- `PushdownWindow` - 窗口函数下推
- `PushdownDistinct` - Distinct 下推
- `PushdownLimit` - Limit 下推
- `PushdownSetOperation` - 集合操作下推
- `PushdownUnnest` - Unnest 下推

### 2.5 下推到扫描
- `PushdownGet` - 下推到 TableScan
- 索引利用
- Zonemap 过滤

### 2.6 FilterPullup
- 上拉时机
- `PullupFilter` - Filter 上拉
- `PullupProjection` - 投影上拉
- `PullupSetOperation` - 集合操作上拉
- `PullupFromLeft` / `PullupBothSide`

### 2.7 FilterCombiner
- 等价类推导
- 范围推导与合并
- 矛盾检测（空结果优化）
- 传递闭包计算

### 2.8 RegexRangeFilter
- 正则表达式范围提取
- 范围条件生成

### 2.9 InClauseRewriter
- IN 列表优化
- Mark Join 转换
- 大 IN 列表处理

---

## 第三章：列与投影优化

### 3.1 列裁剪概述
- 无用列识别
- 列使用追踪
- 优化收益

### 3.2 RemoveUnusedColumns
- 算子遍历策略
- 列使用分析
- 投影消除
- 聚合键裁剪

### 3.3 ColumnLifetimeAnalyzer
- 列生命周期分析
- 投影映射生成
- 提前投影插入

### 3.4 LateMaterialization
- 延迟物化原理
- 适用场景判断
- 物化点选择
- 列 ID 追踪

### 3.5 CSE 优化 (CommonSubExpressionOptimizer)
- 公共子表达式识别
- 表达式提取
- 缓存复用

### 3.6 ColumnBindingReplacer
- 列绑定替换
- 算子间列映射

---

## 第四章：Join 顺序优化

### 4.1 Join 优化概述
- Join 顺序问题
- 搜索空间复杂度
- DPhyp 算法简介

### 4.2 JoinOrderOptimizer 架构
- 优化器入口
- 优化流程
- 结果重构

### 4.3 QueryGraph
- 查询图构建
- 节点与边表示
- 超边处理

### 4.4 JoinRelationSet
- 关系集合表示
- 位图编码
- 集合操作

### 4.5 RelationManager
- 关系管理
- 基表识别
- 关系统计信息

### 4.6 PlanEnumerator
- DPhyp 算法实现
- 子计划枚举
- 最优计划记录
- 剪枝策略

### 4.7 JoinNode
- Join 节点表示
- 代价记录
- 计划重构

### 4.8 QueryGraphManager
- 查询图管理
- 连接条件分析
- 边生成

### 4.9 Cross Product 处理
- Cross Product 识别
- Filter 提升为 Join 条件
- 笛卡尔积优化

---

## 第五章：基数估计与代价模型

### 5.1 基数估计概述
- 估计重要性
- 估计误差影响
- 估计方法概览

### 5.2 CardinalityEstimator
- 基数估计器设计
- 基表基数获取
- Join 基数估计

### 5.3 统计信息类型
- 行数统计
- 列统计 (min, max, distinct count)
- 直方图
- HyperLogLog

### 5.4 选择性估计
- 等值条件选择性
- 范围条件选择性
- LIKE/IN 选择性
- 复合条件选择性

### 5.5 RelationStatisticsHelper
- 统计信息收集
- 统计信息更新
- 采样估计

### 5.6 CostModel
- 代价模型设计
- 代价因子
- 算子代价计算

### 5.7 Join 代价估计
- Hash Join 代价
- Merge Join 代价
- Nested Loop Join 代价
- Build/Probe 代价

### 5.8 StatisticsPropagator
- 统计信息传播
- 算子统计推导
- 表达式统计推导

### 5.9 统计信息传播规则
- `PropagateGet` - 扫描统计
- `PropagateFilter` - 过滤统计
- `PropagateJoin` - 连接统计
- `PropagateAggregate` - 聚合统计
- `PropagateProjection` - 投影统计
- `PropagateWindow` - 窗口统计

---

## 第六章：子查询与 CTE 优化

### 6.1 子查询优化概述
- 相关子查询问题
- 子查询去相关化
- 物化 vs 内联

### 6.2 Deliminator
- DelimGet/DelimJoin 机制
- 冗余消除
- 优化规则

### 6.3 CTEInlining
- CTE 内联条件
- 内联收益分析
- 多次引用处理

### 6.4 CTEFilterPusher
- CTE 过滤下推
- 物化 CTE 优化
- 条件派生

### 6.5 CommonSubplanOptimizer
- 公共子计划识别
- 子计划物化
- CTE 转换

### 6.6 UnnestRewriter
- UNNEST 重写
- DelimJoin 中的 UNNEST
- 投影提升

---

## 第七章：TopN 与窗口函数优化

### 7.1 TopN 优化概述
- ORDER BY + LIMIT 模式
- 堆优化原理
- 动态过滤

### 7.2 TopNOptimizer
- TopN 识别
- ORDER BY + LIMIT 融合
- OFFSET 处理

### 7.3 LimitPushdown
- Limit 下推策略
- 跨算子下推
- 下推限制

### 7.4 TopNWindowElimination
- ROW_NUMBER 优化
- Filter on ROW_NUMBER
- 窗口函数消除
- 转换为聚合

### 7.5 窗口函数优化
- 分区优化
- 排序共享
- Frame 优化

---

## 第八章：Join 实现优化

### 8.1 Join 物理实现选择
- Hash Join vs Merge Join vs Nested Loop
- 选择依据
- 代价比较

### 8.2 BuildProbeSideOptimizer
- Build/Probe 侧选择
- 列生命周期考虑
- 基数考虑

### 8.3 JoinElimination
- 冗余 Join 检测
- 主外键推理
- Distinct 消除

### 8.4 JoinFilterPushdownOptimizer
- 动态过滤生成
- Bloom Filter 下推
- Min/Max 下推
- 运行时过滤

### 8.5 JoinDependentFilterRule
- Join 依赖过滤
- 条件派生
- 等值传播

### 8.6 RowGroupPruner
- Row Group 裁剪
- Zonemap 利用
- 动态过滤集成

### 8.7 SamplingPushdown
- 采样下推
- 采样策略
- 下推限制

---

## 第九章：聚合与其他优化

### 9.1 聚合优化
- `CommonAggregateOptimizer` - 重复聚合消除
- `RemoveDuplicateGroups` - 重复分组消除
- `OrderedAggregateOptimizer` - 有序聚合优化

### 9.2 SumRewriter
- SUM(x + C) 重写
- 计算下推
- 精度考虑

### 9.3 ExpressionHeuristics
- 表达式重排序
- 选择性估计
- 短路求值优化

### 9.4 EmptyResultPullup
- 空结果识别
- 空结果上拉
- 早期终止

### 9.5 CompressedMaterialization
- 压缩物化
- Join 压缩
- Order 压缩
- Aggregate 压缩
- Distinct 压缩

---

## 附录

### A. 优化阶段执行顺序

```
1.  EXPRESSION_REWRITER      - 表达式重写
2.  CTE_INLINING            - CTE 内联 (第一次)
3.  SUM_REWRITER            - SUM 重写
4.  FILTER_PULLUP           - 谓词上拉
5.  FILTER_PUSHDOWN         - 谓词下推
6.  CTE_FILTER_PUSHER       - CTE 过滤下推
7.  REGEX_RANGE             - 正则范围过滤
8.  IN_CLAUSE               - IN 子句重写
9.  DELIMINATOR             - Delim 优化
10. CTE_INLINING            - CTE 内联 (第二次)
11. EMPTY_RESULT_PULLUP     - 空结果上拉
12. JOIN_ORDER              - Join 顺序优化
13. JOIN_ELIMINATION        - Join 消除
14. UNNEST_REWRITER         - UNNEST 重写
15. UNUSED_COLUMNS          - 无用列移除
16. DUPLICATE_GROUPS        - 重复分组移除
17. COMMON_SUBEXPRESSIONS   - 公共子表达式
18. COLUMN_LIFETIME         - 列生命周期 (第一次)
19. BUILD_SIDE_PROBE_SIDE   - Build/Probe 选择
20. COMMON_SUBPLAN          - 公共子计划
21. LIMIT_PUSHDOWN          - Limit 下推
22. ROW_GROUP_PRUNER        - Row Group 裁剪
23. SAMPLING_PUSHDOWN       - 采样下推
24. TOP_N                   - TopN 优化
25. LATE_MATERIALIZATION    - 延迟物化
26. STATISTICS_PROPAGATION  - 统计传播
27. TOP_N_WINDOW_ELIMINATION - TopN 窗口消除
28. COMMON_AGGREGATE        - 公共聚合
29. COLUMN_LIFETIME         - 列生命周期 (第二次)
30. REORDER_FILTER          - 过滤重排序
31. JOIN_FILTER_PUSHDOWN    - Join 过滤下推
```

### B. 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| Optimizer | `src/optimizer/optimizer.cpp` |
| ExpressionRewriter | `src/optimizer/expression_rewriter.cpp` |
| 表达式规则 | `src/optimizer/rule/*.cpp` |
| FilterPushdown | `src/optimizer/filter_pushdown.cpp`, `src/optimizer/pushdown/*.cpp` |
| FilterPullup | `src/optimizer/filter_pullup.cpp`, `src/optimizer/pullup/*.cpp` |
| FilterCombiner | `src/optimizer/filter_combiner.cpp` |
| JoinOrderOptimizer | `src/optimizer/join_order/*.cpp` |
| CardinalityEstimator | `src/optimizer/join_order/cardinality_estimator.cpp` |
| CostModel | `src/optimizer/join_order/cost_model.cpp` |
| StatisticsPropagator | `src/optimizer/statistics_propagator.cpp`, `src/optimizer/statistics/**/*.cpp` |
| RemoveUnusedColumns | `src/optimizer/remove_unused_columns.cpp` |
| ColumnLifetimeAnalyzer | `src/optimizer/column_lifetime_analyzer.cpp` |
| TopNOptimizer | `src/optimizer/topn_optimizer.cpp` |
| CTEInlining | `src/optimizer/cte_inlining.cpp` |
| Deliminator | `src/optimizer/deliminator.cpp` |
| LateMaterialization | `src/optimizer/late_materialization.cpp` |
| BuildProbeSideOptimizer | `src/optimizer/build_probe_side_optimizer.cpp` |
| JoinFilterPushdown | `src/optimizer/join_filter_pushdown_optimizer.cpp` |

### C. 关键数据结构

```
LogicalOperator
     │
     ▼ Optimizer.Optimize()
LogicalOperator (优化后)
     │
     ▼ PhysicalPlanner
PhysicalOperator Tree


优化过程中的关键结构:
├── ExpressionRewriter::rules (规则列表)
├── FilterPushdown::filters (待下推过滤器)
├── JoinOrderOptimizer::query_graph (查询图)
├── PlanEnumerator::dpTable (动态规划表)
├── CardinalityEstimator::statistics (统计信息)
└── CostModel::cost_parameters (代价参数)
```

### D. 查询优化示例

```sql
-- 示例查询
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
  AND o.date >= '2024-01-01'
GROUP BY c.name
HAVING SUM(o.amount) > 1000
ORDER BY SUM(o.amount) DESC
LIMIT 10;
```

优化过程：
1. **表达式重写**: 常量折叠日期字面量
2. **谓词下推**:
   - `c.country = 'China'` → 下推到 customers 扫描
   - `o.date >= '2024-01-01'` → 下推到 orders 扫描
3. **Join 顺序优化**:
   - 估计 customers (filtered) 基数
   - 估计 orders (filtered) 基数
   - 选择最优 Join 顺序
4. **列裁剪**: 只保留需要的列
5. **TopN 优化**: ORDER BY + LIMIT 10 → TopN
6. **统计传播**: 传播过滤后的基数估计
7. **Join 过滤下推**: 生成动态 Bloom Filter

---

## 写作计划

| 章节 | 预计篇幅 | 核心内容 |
|------|----------|----------|
| 第一章 | ~45KB | 优化器架构、ExpressionRewriter、所有表达式规则 |
| 第二章 | ~50KB | FilterPushdown、FilterPullup、FilterCombiner |
| 第三章 | ~35KB | 列裁剪、ColumnLifetime、LateMaterialization、CSE |
| 第四章 | ~50KB | JoinOrderOptimizer、QueryGraph、PlanEnumerator |
| 第五章 | ~45KB | CardinalityEstimator、CostModel、StatisticsPropagator |
| 第六章 | ~35KB | Deliminator、CTEInlining、CommonSubplan |
| 第七章 | ~30KB | TopN、LimitPushdown、TopNWindowElimination |
| 第八章 | ~40KB | BuildProbeSide、JoinElimination、JoinFilterPushdown |
| 第九章 | ~25KB | 聚合优化、压缩物化、其他优化 |

总计约 9 章，预计 350-400KB 内容。

# DuckDB 查询处理深度解析：系列目录

## 系列概述

本系列深入剖析 DuckDB 的查询处理流程，从 SQL 文本到可执行计划的完整转换过程。涵盖解析器、绑定器、逻辑计划生成、查询优化和物理计划生成五大核心组件。

```
┌─────────────────────────────────────────────────────────────┐
│                   DuckDB 查询处理流程                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SQL 文本                                                    │
│     │                                                       │
│     ▼                                                       │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第一章：Parser（解析器）                      │            │
│  │ libpg_query → Transformer → ParsedStatement │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第二章：Binder（绑定器）                      │            │
│  │ 符号解析、类型推导、语义检查                  │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第三章：Planner（逻辑计划生成）               │            │
│  │ BoundStatement → LogicalOperator Tree       │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第四章：Optimizer（查询优化 - 上）            │            │
│  │ 表达式重写、谓词下推、列裁剪                  │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第五章：Optimizer（查询优化 - 下）            │            │
│  │ Join 顺序优化、基数估计、代价模型             │            │
│  └────────────────────┬────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────┐            │
│  │ 第六章：Physical Planner（物理计划生成）      │            │
│  │ LogicalOperator → PhysicalOperator          │            │
│  └─────────────────────────────────────────────┘            │
│                       │                                     │
│                       ▼                                     │
│  可执行物理计划                                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 第一章：Parser - SQL 解析器

### 1.1 整体架构
- Parser 类设计
- libpg_query 集成（PostgreSQL 解析器）
- SQL 方言支持

### 1.2 词法和语法分析
- Token 定义 (`tokens.hpp`)
- 解析选项 (`ParserOptions`)
- 错误处理和位置追踪

### 1.3 Transformer：AST 转换
- PostgreSQL AST → DuckDB AST
- 主要转换方法：
  - `TransformStatement()` - 语句转换
  - `TransformSelect()` - SELECT 语句
  - `TransformExpression()` - 表达式转换
  - `TransformTableRef()` - 表引用转换

### 1.4 核心数据结构
- `SQLStatement` - 语句基类
  - `SelectStatement`
  - `InsertStatement`
  - `UpdateStatement`
  - `DeleteStatement`
  - `CreateStatement`
  - ...
- `ParsedExpression` - 表达式基类
  - `ColumnRefExpression`
  - `ConstantExpression`
  - `FunctionExpression`
  - `ComparisonExpression`
  - `ConjunctionExpression`
  - ...
- `TableRef` - 表引用基类
  - `BaseTableRef`
  - `JoinRef`
  - `SubqueryRef`
  - `TableFunctionRef`
  - ...

### 1.5 QueryNode 结构
- `SelectNode`
- `SetOperationNode` (UNION/INTERSECT/EXCEPT)
- `RecursiveCTENode`

### 1.6 扩展机制
- `ParserExtension` 接口
- 自定义语法支持

---

## 第二章：Binder - 语义绑定器

### 2.1 Binder 架构
- `Binder` 类设计
- 绑定上下文 (`BindContext`)
- 作用域管理

### 2.2 符号解析
- 表和列解析
- `TableBinding` 和 `ColumnBinding`
- Schema 和 Catalog 查找

### 2.3 类型系统
- 类型推导规则
- 隐式类型转换
- 类型兼容性检查

### 2.4 ExpressionBinder
- `ExpressionBinder` 基类
- 不同上下文的绑定器：
  - `SelectBinder`
  - `WhereBinder`
  - `HavingBinder`
  - `OrderBinder`
  - `GroupBinder`

### 2.5 表达式绑定
- `ParsedExpression` → `Expression`
- 函数绑定和重载解析
- 聚合函数处理
- 窗口函数处理

### 2.6 子查询处理
- 相关子查询检测
- 子查询去相关化
- EXISTS/IN 转换

### 2.7 CTE 处理
- 普通 CTE 绑定
- 递归 CTE 处理
- CTE 内联决策

### 2.8 BoundStatement 输出
- `BoundSelectStatement`
- `BoundInsertStatement`
- 其他绑定语句类型

---

## 第三章：Planner - 逻辑计划生成

### 3.1 Planner 架构
- `Planner` 类设计
- 计划生成流程

### 3.2 LogicalOperator 层次结构
- `LogicalOperator` 基类
- 主要算子类型：
  - **扫描**: `LogicalGet`, `LogicalChunkGet`
  - **投影**: `LogicalProjection`
  - **过滤**: `LogicalFilter`
  - **连接**: `LogicalJoin`, `LogicalComparisonJoin`, `LogicalCrossProduct`
  - **聚合**: `LogicalAggregate`
  - **排序**: `LogicalOrder`, `LogicalTopN`
  - **集合**: `LogicalUnion`, `LogicalIntersect`, `LogicalExcept`
  - **DML**: `LogicalInsert`, `LogicalUpdate`, `LogicalDelete`

### 3.3 表达式系统
- `Expression` 基类
- 表达式类型：
  - `BoundColumnRefExpression`
  - `BoundConstantExpression`
  - `BoundFunctionExpression`
  - `BoundAggregateExpression`
  - `BoundWindowExpression`
  - `BoundSubqueryExpression`

### 3.4 计划构建
- SELECT 语句计划生成
- JOIN 计划构建
- 聚合计划构建
- 窗口函数计划

### 3.5 列绑定
- `ColumnBinding` 设计
- 列引用追踪
- 输出列映射

---

## 第四章：Optimizer（上）- 表达式和谓词优化

### 4.1 Optimizer 架构
- `Optimizer` 类设计
- 优化阶段顺序
- 优化开关配置

### 4.2 表达式重写框架
- `ExpressionRewriter` 设计
- `Rule` 抽象和模式匹配
- 规则执行顺序

### 4.3 常量折叠 (Constant Folding)
- 编译时常量计算
- 表达式简化

### 4.4 表达式简化规则
- 算术简化 (`arithmetic_simplification`)
- 比较简化 (`comparison_simplification`)
- 逻辑简化 (`conjunction_simplification`)
- CASE 简化 (`case_simplification`)
- LIKE 优化 (`like_optimizations`)
- 正则优化 (`regex_optimizations`)

### 4.5 Filter Pushdown（谓词下推）
- 下推策略
- 跨 JOIN 下推
- 下推到 TableScan

### 4.6 Filter Pullup（谓词上拉）
- 上拉时机
- 合并条件

### 4.7 Filter Combiner（谓词合并）
- 等价类推导
- 范围推导
- 矛盾检测（空结果优化）

### 4.8 列裁剪 (Remove Unused Columns)
- 列使用分析
- 投影消除
- 无用列移除

### 4.9 CSE 优化 (Common Subexpression Elimination)
- 公共子表达式识别
- 表达式提取

---

## 第五章：Optimizer（下）- Join 优化和统计信息

### 5.1 Join 顺序优化器
- `JoinOrderOptimizer` 架构
- 动态规划 vs 贪婪算法
- 搜索空间控制

### 5.2 QueryGraph
- 查询图构建
- 连接条件建模
- 超边表示

### 5.3 基数估计 (Cardinality Estimation)
- `CardinalityEstimator` 设计
- 直方图统计
- 采样估计
- HyperLogLog 应用

### 5.4 代价模型 (Cost Model)
- 代价因子定义
- 算子代价计算
- Join 代价估计

### 5.5 计划枚举 (Plan Enumerator)
- DPhyp 算法
- 子计划最优性
- 剪枝策略

### 5.6 Join 类型选择
- Hash Join vs Merge Join vs Nested Loop
- Build 侧选择
- `BuildProbeSideOptimizer`

### 5.7 Join 消除 (Join Elimination)
- 冗余 JOIN 检测
- 主外键推理
- Distinct 消除

### 5.8 统计信息传播
- `StatisticsPropagator` 设计
- 算子统计推导
- 过滤选择性估计

### 5.9 其他高级优化
- TopN 优化 (`TopNOptimizer`)
- Limit 下推 (`LimitPushdown`)
- 延迟物化 (`LateMaterialization`)
- 窗口消除 (`TopNWindowElimination`)
- Unnest 重写 (`UnnestRewriter`)
- CTE 内联 (`CTEInlining`)

---

## 第六章：Physical Planner - 物理计划生成

### 6.1 Physical Planner 架构
- 计划转换流程
- 算子映射规则

### 6.2 LogicalOperator → PhysicalOperator
- 扫描算子转换
- 连接算子转换
- 聚合算子转换
- 排序算子转换

### 6.3 Join 物理实现选择
- Hash Join 选择条件
- Merge Join 选择条件
- Nested Loop Join 选择条件
- Cross Product 处理

### 6.4 聚合物理实现
- Hash Aggregate vs Perfect Hash Aggregate
- Ungrouped Aggregate
- Partitioned Aggregate

### 6.5 Pipeline 构建
- `BuildPipelines()` 方法
- 阻塞点识别
- 依赖关系生成

### 6.6 并行计划生成
- 并行度决定
- 分区策略
- Exchange 算子

### 6.7 动态过滤器
- Bloom Filter 下推
- Join Filter 生成
- 运行时过滤

### 6.8 执行计划输出
- EXPLAIN 输出格式
- EXPLAIN ANALYZE 统计
- 可视化支持

---

## 附录

### A. 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| Parser | `src/parser/`, `src/include/duckdb/parser/` |
| Transformer | `src/parser/transform/` |
| Binder | `src/planner/binder/` |
| Planner | `src/planner/planner.cpp` |
| Optimizer | `src/optimizer/` |
| Physical Planner | `src/execution/physical_plan/` |

### B. 关键类图

```
SQLStatement (Parser Output)
     │
     ▼
BoundStatement (Binder Output)
     │
     ▼
LogicalOperator Tree (Planner Output)
     │
     ▼ Optimizer
LogicalOperator Tree (Optimized)
     │
     ▼ Physical Planner
PhysicalOperator Tree (Execution Plan)
```

### C. 优化规则速查表

| 优化器 | 功能 | 阶段 |
|--------|------|------|
| ExpressionRewriter | 表达式重写规则 | 早期 |
| FilterPushdown | 谓词下推 | 早期 |
| FilterCombiner | 谓词合并 | 早期 |
| RemoveUnusedColumns | 列裁剪 | 中期 |
| JoinOrderOptimizer | Join 顺序 | 中期 |
| TopNOptimizer | TopN 优化 | 中期 |
| StatisticsPropagator | 统计传播 | 中期 |
| BuildProbeSideOptimizer | Build 侧选择 | 后期 |
| ColumnLifetimeAnalyzer | 列生命周期 | 后期 |

### D. 查询处理示例

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

处理流程：
1. **Parser**: 解析 SQL，生成 SelectStatement
2. **Binder**: 解析表名、列名，类型检查
3. **Planner**: 生成 LogicalGet → LogicalJoin → LogicalFilter → LogicalAggregate → LogicalOrder → LogicalLimit
4. **Optimizer**:
   - 谓词下推: WHERE 条件下推到各表
   - Join 顺序优化: 估计基数选择最优顺序
   - TopN 优化: ORDER BY + LIMIT 合并
5. **Physical Planner**: 生成 TableScan → HashJoin → HashAggregate → TopN

---

## 写作计划

| 章节 | 预计篇幅 | 核心内容 |
|------|----------|----------|
| 第一章 | ~35KB | Parser、Transformer、AST 结构 |
| 第二章 | ~40KB | Binder、类型系统、符号解析 |
| 第三章 | ~35KB | Planner、LogicalOperator、表达式 |
| 第四章 | ~40KB | 表达式重写、谓词优化、列裁剪 |
| 第五章 | ~45KB | Join 优化、基数估计、代价模型 |
| 第六章 | ~35KB | 物理计划生成、Pipeline 构建 |

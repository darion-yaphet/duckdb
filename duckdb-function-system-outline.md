# DuckDB 函数系统深度解析 - 系列提纲

## 系列概述

DuckDB 的函数系统是其核心组件之一，支持标量函数、聚合函数、表函数、宏函数等多种类型。本系列将深入剖析函数系统的架构设计、类型系统、绑定机制、执行模型以及用户自定义函数的实现。

## 章节规划

### 第一章：函数系统架构概述

**核心内容：**
- 函数类型分类与层次结构
  - Function → SimpleFunction → BaseScalarFunction
  - ScalarFunction、AggregateFunction、TableFunction
- 核心抽象设计
  - FunctionData：绑定时生成的函数状态
  - FunctionLocalState：线程局部状态
  - FunctionSet：函数重载管理
- 函数属性系统
  - FunctionStability：CONSISTENT / VOLATILE / CONSISTENT_WITHIN_QUERY
  - FunctionNullHandling：DEFAULT_NULL_HANDLING / SPECIAL_HANDLING
  - FunctionErrors：错误处理模式
  - FunctionCollationHandling：排序规则处理
- 函数注册与发现机制
  - FunctionList 静态注册
  - BuiltinFunctions 启动注册
  - DefaultGenerator 懒加载

**源码文件：**
- `src/include/duckdb/function/function.hpp`
- `src/include/duckdb/function/function_set.hpp`
- `src/function/function.cpp`
- `src/function/function_set.cpp`

**预计篇幅：** 约5000字

---

### 第二章：标量函数实现机制

**核心内容：**
- ScalarFunction 核心结构
  - 函数签名：参数类型 → 返回类型
  - 回调函数体系：function、bind、statistics、init_local_state
  - 序列化/反序列化支持
- 向量化执行器模板
  - UnaryExecutor：一元函数执行
  - BinaryExecutor：二元函数执行
  - TernaryExecutor：三元函数执行
  - GenericExecutor：通用执行器
- 典型标量函数实现分析
  - 算术运算符：+、-、*、/
  - 字符串函数：upper、lower、substring
  - 日期函数：date_part、strftime
- Lambda 函数特殊支持
  - bind_lambda 回调
  - ListLambdaBindData 绑定数据
  - list_transform、list_filter 实现

**源码文件：**
- `src/include/duckdb/function/scalar_function.hpp`
- `src/include/duckdb/function/lambda_functions.hpp`
- `src/function/scalar/operator/arithmetic.cpp`
- `src/function/scalar/string/caseconvert.cpp`

**预计篇幅：** 约6000字

---

### 第三章：聚合函数状态机模型

**核心内容：**
- AggregateFunction 生命周期
  - state_size：状态大小计算
  - initialize：状态初始化
  - update：增量更新（分散模式）
  - combine：状态合并（并行聚合）
  - finalize：最终结果计算
  - destructor：状态销毁
- 聚合执行器模板
  - AggregateExecutor：核心执行逻辑
  - UnaryScatter/UnaryUpdate：一元聚合
  - BinaryScatter/BinaryUpdate：二元聚合
- 简单更新优化
  - simple_update：非分组聚合优化
  - 避免状态向量开销
- 窗口聚合特殊支持
  - window 回调：自定义窗口计算
  - window_init：分区初始化
  - FrameDelta/FrameStats：帧边界信息
- 典型聚合函数实现
  - count/sum/avg：基础统计
  - min/max：极值计算
  - first/last：顺序相关聚合
  - string_agg：字符串聚合

**源码文件：**
- `src/include/duckdb/function/aggregate_function.hpp`
- `src/include/duckdb/function/aggregate_state.hpp`
- `src/function/aggregate/distributive/count.cpp`
- `src/function/aggregate/distributive/minmax.cpp`
- `src/function/aggregate/sorted_aggregate_function.cpp`

**预计篇幅：** 约7000字

---

### 第四章：表函数与数据源抽象

**核心内容：**
- TableFunction 核心结构
  - bind：绑定输入参数，返回列定义
  - init_global：全局状态初始化
  - init_local：线程局部状态初始化
  - function：数据生成主函数
- 状态管理模型
  - GlobalTableFunctionState：跨线程共享状态
  - LocalTableFunctionState：线程私有状态
  - MaxThreads：并行度控制
- 下推优化支持
  - projection_pushdown：列裁剪
  - filter_pushdown：过滤下推
  - filter_prune：过滤列剪枝
  - pushdown_complex_filter：复杂表达式下推
- In-Out 表函数
  - in_out_function：接收输入数据块
  - in_out_function_final：处理最后一批数据
- 典型表函数实现
  - read_csv/read_parquet：文件读取
  - duckdb_tables/duckdb_columns：系统元数据
  - generate_series/range：序列生成
- 进度与统计
  - table_scan_progress：扫描进度
  - cardinality：基数估算
  - statistics：列统计信息

**源码文件：**
- `src/include/duckdb/function/table_function.hpp`
- `src/function/table/arrow.cpp`
- `src/function/table/system/duckdb_tables.cpp`

**预计篇幅：** 约6500字

---

### 第五章：宏函数与SQL级抽象

**核心内容：**
- 宏类型体系
  - MacroType：VOID_MACRO / TABLE_MACRO / SCALAR_MACRO
  - ScalarMacroFunction：标量表达式宏
  - TableMacroFunction：表查询宏
- 宏定义与参数
  - parameters：位置参数列表
  - default_parameters：默认值映射
  - 参数替换机制
- 宏绑定流程
  - BindMacroFunction：参数匹配
  - CreateDummyBinding：虚拟绑定
  - 表达式替换与展开
- 内置宏系统
  - DefaultFunctionGenerator：SQL 宏懒加载
  - DefaultTableFunctionGenerator：表宏懒加载
  - 宏定义表：internal_macros[]
- 宏与普通函数对比
  - 执行时机：绑定期展开 vs 执行期调用
  - 性能特征：表达式内联 vs 函数调用
  - 适用场景：组合逻辑 vs 核心计算

**源码文件：**
- `src/include/duckdb/function/macro_function.hpp`
- `src/include/duckdb/function/scalar_macro_function.hpp`
- `src/include/duckdb/function/table_macro_function.hpp`
- `src/function/macro_function.cpp`
- `src/catalog/default/default_functions.cpp`

**预计篇幅：** 约5000字

---

### 第六章：函数绑定与重载解析

**核心内容：**
- FunctionBinder 核心职责
  - 重载解析：从 FunctionSet 选择最佳匹配
  - 类型转换：隐式类型提升与强制转换
  - 错误报告：无匹配时的诊断信息
- 重载解析算法
  - BindFunctionCost：计算类型转换代价
  - BindVarArgsFunctionCost：变参函数代价
  - MultipleCandidateException：多候选处理
- 类型转换规则
  - CastRules：隐式转换成本矩阵
  - 精确匹配 → 隐式提升 → 显式转换
  - ANY 类型与模板类型解析
- 表达式绑定流程
  - BindScalarFunction：标量函数绑定
  - BindAggregateFunction：聚合函数绑定
  - CastToFunctionArguments：参数类型对齐
- 特殊绑定场景
  - Lambda 参数绑定：bind_lambda 回调
  - 排序聚合绑定：BindSortedAggregate
  - 模板类型解析：ResolveTemplateTypes

**源码文件：**
- `src/include/duckdb/function/function_binder.hpp`
- `src/function/function_binder.cpp`
- `src/include/duckdb/function/cast_rules.hpp`
- `src/function/cast_rules.cpp`

**预计篇幅：** 约5500字

---

### 第七章：用户自定义函数（UDF）

**核心内容：**
- UDFWrapper 模板框架
  - CreateScalarFunction：C++ 函数包装
  - 类型自动推导：模板参数 → LogicalType
  - 一元/二元/三元函数支持
- 标量 UDF 注册
  - RegisterFunction：注册到 Catalog
  - 类型映射：C++ 类型 ↔ SQL 类型
  - 执行器适配：UnaryUDFExecutor
- 聚合 UDF 注册
  - CreateAggregateFunction：聚合函数包装
  - 状态类型定义：STATE 模板参数
  - 操作接口：Initialize/Update/Combine/Finalize
- C API 扩展接口
  - duckdb_create_scalar_function
  - duckdb_scalar_function_set_function
  - duckdb_register_scalar_function
- Python UDF 支持
  - 类型转换层
  - GIL 管理
  - 向量化执行适配

**源码文件：**
- `src/include/duckdb/function/udf_function.hpp`
- `src/main/capi/scalar_function-c.cpp`
- `src/main/capi/aggregate_function-c.cpp`

**预计篇幅：** 约5000字

---

### 第八章：类型转换函数体系

**核心内容：**
- CastFunctionSet 架构
  - 类型转换函数注册
  - 转换代价计算
  - 隐式 vs 显式转换
- 内置转换实现
  - 数值类型转换：整数 ↔ 浮点 ↔ 小数
  - 字符串转换：类型 ↔ VARCHAR
  - 时间类型转换：DATE ↔ TIMESTAMP ↔ INTERVAL
- 嵌套类型转换
  - LIST 转换：元素类型递归转换
  - STRUCT 转换：字段匹配与转换
  - MAP 转换：键值对转换
  - UNION 转换：变体类型处理
- TryCast 与错误处理
  - 严格模式：转换失败抛异常
  - 宽松模式：转换失败返回 NULL
  - 错误信息收集

**源码文件：**
- `src/function/cast/cast_function_set.cpp`
- `src/function/cast/default_casts.cpp`
- `src/function/cast/numeric_casts.cpp`
- `src/function/cast/string_cast.cpp`
- `src/function/cast/list_casts.cpp`
- `src/function/cast/struct_cast.cpp`

**预计篇幅：** 约5500字

---

### 第九章：Pragma 与系统函数

**核心内容：**
- PragmaFunction 设计
  - PRAGMA 命令类型
  - 参数解析与验证
  - 执行流程
- 常用 Pragma 实现
  - pragma_database_list：数据库列表
  - pragma_table_info：表结构信息
  - pragma_version：版本信息
- 系统函数分类
  - 元数据查询：current_database、current_schema
  - 配置管理：current_setting、set
  - 调试辅助：explain、profile
- COPY 函数机制
  - CopyFunction 接口
  - 格式处理器：CSV、Parquet、JSON
  - 编码函数：压缩与编码

**源码文件：**
- `src/include/duckdb/function/pragma_function.hpp`
- `src/function/pragma/pragma_functions.cpp`
- `src/function/pragma/pragma_queries.cpp`
- `src/include/duckdb/function/copy_function.hpp`

**预计篇幅：** 约4500字

---

### 第十章：函数执行与优化

**核心内容：**
- ExpressionExecutor 集成
  - BoundFunctionExpression 执行
  - 状态初始化与管理
  - 结果向量处理
- 函数级优化
  - 常量折叠：编译期计算
  - 表达式简化：代数变换
  - 统计传播：statistics 回调
- 特殊执行模式
  - 短路求值：AND/OR 优化
  - NULL 处理：DEFAULT vs SPECIAL
  - 错误处理：TRY 表达式
- 并行执行考虑
  - 线程安全保证
  - 状态隔离策略
  - 合并操作设计

**源码文件：**
- `src/execution/expression_executor.cpp`
- `src/execution/expression_executor/execute_function.cpp`
- `src/optimizer/rule/constant_folding.cpp`

**预计篇幅：** 约5000字

---

## 附录：函数类型一览表

| 函数类型 | 基类 | 主要用途 | 典型示例 |
|---------|------|---------|---------|
| ScalarFunction | BaseScalarFunction | 行级计算 | upper, +, substring |
| AggregateFunction | BaseScalarFunction | 聚合计算 | sum, count, avg |
| TableFunction | SimpleNamedParameterFunction | 数据源 | read_csv, generate_series |
| ScalarMacroFunction | MacroFunction | SQL 表达式宏 | current_user |
| TableMacroFunction | MacroFunction | SQL 查询宏 | histogram |
| PragmaFunction | Function | 系统命令 | pragma_table_info |
| CopyFunction | Function | 数据导入导出 | csv, parquet |

---

## 函数回调速查表

### ScalarFunction 回调

| 回调 | 类型 | 用途 |
|-----|------|------|
| function | scalar_function_t | 主执行函数 |
| bind | bind_scalar_function_t | 绑定时类型推导 |
| init_local_state | init_local_state_t | 线程局部状态初始化 |
| statistics | function_statistics_t | 统计信息传播 |
| bind_lambda | bind_lambda_function_t | Lambda 参数类型绑定 |
| serialize/deserialize | - | 计划序列化支持 |

### AggregateFunction 回调

| 回调 | 类型 | 用途 |
|-----|------|------|
| state_size | aggregate_size_t | 状态大小 |
| initialize | aggregate_initialize_t | 状态初始化 |
| update | aggregate_update_t | 分散更新 |
| combine | aggregate_combine_t | 状态合并 |
| finalize | aggregate_finalize_t | 最终计算 |
| simple_update | aggregate_simple_update_t | 简单更新优化 |
| window | aggregate_window_t | 窗口计算 |
| destructor | aggregate_destructor_t | 状态销毁 |

### TableFunction 回调

| 回调 | 类型 | 用途 |
|-----|------|------|
| bind | table_function_bind_t | 列定义与参数绑定 |
| init_global | table_function_init_global_t | 全局状态初始化 |
| init_local | table_function_init_local_t | 局部状态初始化 |
| function | table_function_t | 数据生成主函数 |
| cardinality | table_function_cardinality_t | 基数估算 |
| pushdown_complex_filter | - | 过滤下推 |
| table_scan_progress | table_function_progress_t | 扫描进度 |

---

## 写作原则

1. **层次分明**：从抽象到具体，先讲接口设计再讲实现细节
2. **代码驱动**：每个概念都以实际代码为基础，附带关键片段
3. **图文结合**：使用 ASCII 图表展示类层次、执行流程、状态转换
4. **实践导向**：提供 UDF 开发示例，帮助读者理解如何扩展函数

## 预计总篇幅

- 10 章 × 平均 5500 字 ≈ 55000 字
- 加上代码示例和图表，总计约 60000-65000 字

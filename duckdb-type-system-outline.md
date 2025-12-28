# DuckDB 类型系统深度解析 - 系列提纲

## 系列概述

DuckDB 的类型系统是其数据处理能力的基础，采用双层类型设计（逻辑类型 + 物理类型）、向量化存储模型和灵活的类型转换机制。本系列将深入剖析类型系统的架构设计、标量类型、嵌套类型、向量化存储、类型转换以及特殊类型的实现。

## 章节规划

### 第一章：类型系统架构概述

**核心内容：**
- 双层类型设计理念
  - LogicalTypeId：SQL 语义类型
  - PhysicalType：物理存储类型
  - 逻辑类型到物理类型的映射
- LogicalType 类设计
  - 类型标识（id_）
  - 物理类型（physical_type_）
  - 扩展类型信息（type_info_）
  - 类型属性方法
- ExtraTypeInfo 扩展机制
  - DecimalTypeInfo：精度和标度
  - StringTypeInfo：排序规则
  - ListTypeInfo：子元素类型
  - StructTypeInfo：字段定义
  - UserTypeInfo：用户定义类型
- 类型工厂方法
  - LogicalType::DECIMAL()
  - LogicalType::LIST()
  - LogicalType::STRUCT()
  - LogicalType::MAP()

**源码文件：**
- `src/include/duckdb/common/types.hpp`
- `src/common/types.cpp`
- `src/common/extra_type_info.cpp`

**预计篇幅：** 约5500字

---

### 第二章：标量类型实现

**核心内容：**
- 数值类型
  - 整数类型：TINYINT/SMALLINT/INTEGER/BIGINT
  - 无符号整数：UTINYINT/USMALLINT/UINTEGER/UBIGINT
  - 大整数：HUGEINT（128位）/UHUGEINT
  - 浮点数：FLOAT/DOUBLE
  - 定点数：DECIMAL（width, scale）
- 字符串类型
  - VARCHAR：变长字符串
  - string_t 内联优化（12字节阈值）
  - 前缀存储优化
  - StringHeap 堆管理
- 二进制类型
  - BLOB：二进制大对象
  - BIT：位串类型
- 布尔类型
  - BOOLEAN 存储与比较
- 特殊标量类型
  - UUID：128位标识符
  - POINTER：内部指针类型

**源码文件：**
- `src/common/types/hugeint.cpp`
- `src/common/types/uhugeint.cpp`
- `src/common/types/decimal.cpp`
- `src/common/types/string_type.cpp`
- `src/common/types/blob.cpp`
- `src/common/types/uuid.cpp`
- `src/common/types/bit.cpp`

**预计篇幅：** 约6000字

---

### 第三章：日期时间类型

**核心内容：**
- DATE 日期类型
  - 内部表示：距 1970-01-01 的天数
  - Date 工具类：FromDate/ExtractYear/Month/Day
  - 日期解析与格式化
- TIME 时间类型
  - 内部表示：午夜以来的微秒数
  - TIME_TZ：带时区时间
  - TIME_NS：纳秒精度
- TIMESTAMP 时间戳类型
  - TIMESTAMP：微秒精度（默认）
  - TIMESTAMP_MS：毫秒精度
  - TIMESTAMP_SEC：秒精度
  - TIMESTAMP_NS：纳秒精度
  - TIMESTAMP_TZ：带时区时间戳
- INTERVAL 间隔类型
  - interval_t 结构：months, days, micros
  - 间隔算术运算
  - 与其他日期时间类型的交互
- 日期时间工具
  - 闰年判断
  - 日期加减
  - 时区转换

**源码文件：**
- `src/common/types/date.cpp`
- `src/common/types/time.cpp`
- `src/common/types/timestamp.cpp`
- `src/common/types/interval.cpp`
- `src/include/duckdb/common/types/datetime.hpp`

**预计篇幅：** 约5500字

---

### 第四章：嵌套类型系统

**核心内容：**
- LIST 列表类型
  - ListTypeInfo：子类型定义
  - list_entry_t：offset + length
  - ListVector 操作
  - 嵌套列表支持
- STRUCT 结构体类型
  - StructTypeInfo：字段列表
  - 命名字段访问
  - StructVector 操作
  - 结构体展开与构造
- MAP 映射类型
  - 基于 LIST(STRUCT) 实现
  - 键唯一性约束
  - MapVector 操作
- ARRAY 固定长度数组
  - ArrayTypeInfo：子类型 + 长度
  - 与 LIST 的区别
  - 内存布局优化
- UNION 联合类型
  - UnionTypeInfo：成员类型
  - 标签字段（tag）
  - UnionVector 操作
  - 类型判别与提取
- 嵌套类型的统一处理
  - 递归类型遍历
  - 嵌套深度限制
  - 类型兼容性检查

**源码文件：**
- `src/common/types/vector.cpp`（LIST/STRUCT/ARRAY/UNION 相关）
- `src/include/duckdb/common/types/vector.hpp`
- `src/common/types/list_segment.cpp`

**预计篇幅：** 约6500字

---

### 第五章：向量化存储模型

**核心内容：**
- Vector 类设计
  - VectorType 枚举
    - FLAT_VECTOR：扁平向量
    - CONSTANT_VECTOR：常量向量
    - DICTIONARY_VECTOR：字典向量
    - SEQUENCE_VECTOR：序列向量
    - FSST_VECTOR：FSST 压缩向量
  - 数据指针与缓冲区
  - 有效性掩码集成
- VectorBuffer 缓冲区
  - VectorBufferType 类型
  - 引用计数管理
  - 子缓冲区层次
- UnifiedVectorFormat
  - 统一数据访问接口
  - 选择向量（SelectionVector）
  - 数据指针获取
  - 有效性检查
- 向量操作
  - FlatVector：扁平向量操作
  - ConstantVector：常量向量操作
  - DictionaryVector：字典向量操作
  - ListVector：列表向量操作
  - StructVector：结构体向量操作
- 向量辅助类
  - StringVector：字符串操作
  - ArrayVector：数组操作
  - UnionVector：联合体操作

**源码文件：**
- `src/include/duckdb/common/types/vector.hpp`
- `src/common/types/vector.cpp`
- `src/include/duckdb/common/types/vector_buffer.hpp`
- `src/common/types/vector_buffer.cpp`

**预计篇幅：** 约6000字

---

### 第六章：DataChunk 与数据批处理

**核心内容：**
- DataChunk 类设计
  - 列向量集合
  - 行数管理（cardinality）
  - 列类型定义
- DataChunk 操作
  - Initialize：初始化
  - Reset：重置状态
  - SetCardinality：设置行数
  - Flatten：展平向量
  - Slice：切片操作
  - Append：追加数据
  - Copy：复制数据
- 选择向量（SelectionVector）
  - sel_t 索引数组
  - 选择向量缓冲
  - 与 DataChunk 配合
- 有效性掩码（ValidityMask）
  - 位图存储
  - NULL 值表示
  - 有效性操作
- DataChunk 序列化
  - 二进制格式
  - 跨进程传输

**源码文件：**
- `src/include/duckdb/common/types/data_chunk.hpp`
- `src/common/types/data_chunk.cpp`
- `src/include/duckdb/common/types/selection_vector.hpp`
- `src/common/types/selection_vector.cpp`
- `src/include/duckdb/common/types/validity_mask.hpp`
- `src/common/types/validity_mask.cpp`

**预计篇幅：** 约5500字

---

### 第七章：Value 类型系统

**核心内容：**
- Value 类设计
  - 类型持有（LogicalType）
  - 值存储（value_ 联合体）
  - NULL 标记
- Value 工厂方法
  - Value::BOOLEAN()
  - Value::INTEGER()
  - Value::VARCHAR()
  - Value::STRUCT()
  - Value::LIST()
  - Value::MAP()
- Value 访问接口
  - GetValue<T>()：类型化获取
  - GetValueUnsafe<T>()：无检查获取
  - ToString()：字符串表示
  - IsNull()：空值检查
- Value 类型转换
  - DefaultCastAs()：默认转换
  - TryCastAs()：尝试转换
  - 隐式转换规则
- Value 比较与哈希
  - 相等性比较
  - 排序比较
  - 哈希计算
- 嵌套 Value 操作
  - StructValue：结构体值
  - ListValue：列表值
  - MapValue：映射值
  - UnionValue：联合值

**源码文件：**
- `src/include/duckdb/common/types/value.hpp`
- `src/common/types/value.cpp`

**预计篇幅：** 约5500字

---

### 第八章：类型转换系统

**核心内容：**
- CastFunctionSet 架构
  - 转换函数注册
  - 转换成本计算
  - 隐式/显式转换
- BoundCastInfo 结构
  - 转换函数指针
  - 绑定数据
  - 本地状态初始化
- 内置转换函数
  - 数值类型转换
  - 字符串转换
  - 日期时间转换
  - 嵌套类型转换
- 转换成本模型
  - ImplicitCastCost：隐式转换成本
  - 最小成本路径选择
  - 类型提升规则
- 自定义类型转换
  - 扩展注册转换
  - 转换函数实现
  - 错误处理策略
- 特殊转换场景
  - DECIMAL 精度转换
  - ENUM 类型转换
  - VARIANT 通用转换

**源码文件：**
- `src/include/duckdb/function/cast/cast_function_set.hpp`
- `src/function/cast/cast_function_set.cpp`
- `src/function/cast/default_casts.cpp`
- `src/function/cast/numeric_casts.cpp`
- `src/function/cast/string_cast.cpp`
- `src/function/cast/list_casts.cpp`
- `src/function/cast/struct_cast.cpp`

**预计篇幅：** 约6000字

---

### 第九章：行列数据集合

**核心内容：**
- TupleDataCollection 行存储
  - TupleDataLayout：行布局
  - TupleDataAllocator：内存分配
  - TupleDataSegment：数据分段
  - Scatter/Gather 操作
- ColumnDataCollection 列存储
  - 分段列存储
  - 内存管理策略
  - 迭代器接口
- PartitionedTupleData 分区数据
  - 哈希分区
  - 分区管理
  - 并行处理支持
- PartitionedColumnData 分区列数据
  - 列式分区存储
  - 分区合并
- RowDataCollection 行数据集合
  - 行数据扫描器
  - 批量操作
- 数据集合转换
  - 行列转换
  - DataChunk 互操作

**源码文件：**
- `src/include/duckdb/common/types/row/tuple_data_collection.hpp`
- `src/common/types/row/tuple_data_collection.cpp`
- `src/common/types/row/tuple_data_layout.cpp`
- `src/include/duckdb/common/types/column/column_data_collection.hpp`
- `src/common/types/column/column_data_collection.cpp`
- `src/common/types/row/partitioned_tuple_data.cpp`

**预计篇幅：** 约6000字

---

### 第十章：特殊类型与扩展

**核心内容：**
- ENUM 枚举类型
  - EnumTypeInfo：枚举定义
  - 字典编码存储
  - 枚举值操作
  - 动态枚举创建
- VARIANT 变体类型
  - 动态类型存储
  - 类型判别
  - 二进制编码格式
  - JSON 互操作
- GEOMETRY 几何类型
  - 扩展类型标记
  - WKB 编码
  - 与 spatial 扩展集成
- 用户定义类型
  - USER 类型注册
  - 类型别名
  - 类型修饰符
- 扩展类型信息
  - ExtensionTypeInfo
  - 类型元数据
  - 序列化支持
- 类型系统扩展
  - 扩展类型注册
  - 类型转换扩展
  - 类型操作扩展

**源码文件：**
- `src/function/cast/enum_casts.cpp`
- `src/common/types/variant/variant.cpp`
- `src/common/types/variant/variant_value.cpp`
- `src/common/types/geometry.cpp`
- `src/common/extra_type_info.cpp`

**预计篇幅：** 约5500字

---

## 附录：类型 ID 速查表

### 基础类型

| LogicalTypeId | PhysicalType | 说明 |
|---------------|--------------|------|
| BOOLEAN | BOOL | 布尔值 |
| TINYINT | INT8 | 8位有符号整数 |
| SMALLINT | INT16 | 16位有符号整数 |
| INTEGER | INT32 | 32位有符号整数 |
| BIGINT | INT64 | 64位有符号整数 |
| HUGEINT | INT128 | 128位有符号整数 |
| FLOAT | FLOAT | 32位浮点数 |
| DOUBLE | DOUBLE | 64位浮点数 |
| VARCHAR | VARCHAR | 变长字符串 |
| BLOB | VARCHAR | 二进制大对象 |

### 日期时间类型

| LogicalTypeId | PhysicalType | 说明 |
|---------------|--------------|------|
| DATE | INT32 | 日期（天数） |
| TIME | INT64 | 时间（微秒） |
| TIMESTAMP | INT64 | 时间戳（微秒） |
| INTERVAL | INTERVAL | 时间间隔 |

### 嵌套类型

| LogicalTypeId | PhysicalType | 说明 |
|---------------|--------------|------|
| LIST | LIST | 可变长度列表 |
| STRUCT | STRUCT | 结构体 |
| MAP | LIST | 键值映射 |
| ARRAY | ARRAY | 固定长度数组 |
| UNION | STRUCT | 联合类型 |

---

## VectorType 速查表

| VectorType | 特点 | 使用场景 |
|------------|------|---------|
| FLAT_VECTOR | 扁平存储 | 通用向量 |
| CONSTANT_VECTOR | 常量复制 | 标量广播 |
| DICTIONARY_VECTOR | 字典编码 | 重复值压缩 |
| SEQUENCE_VECTOR | 序列生成 | 连续整数 |
| FSST_VECTOR | FSST 压缩 | 字符串压缩 |

---

## 类型转换成本

| 源类型 → 目标类型 | 成本 | 说明 |
|------------------|------|------|
| 相同类型 | 0 | 无转换 |
| 整数提升 | 1-3 | TINYINT→BIGINT |
| 整数→浮点 | 5 | INTEGER→DOUBLE |
| 数值→字符串 | 10 | INTEGER→VARCHAR |
| 任意→VARIANT | 1 | 通用容器 |

---

## 写作原则

1. **分层解析**：从逻辑类型到物理类型，再到向量存储
2. **代码驱动**：每个类型配合关键实现代码
3. **性能导向**：解释设计背后的性能考量
4. **实践导向**：提供类型使用示例

## 预计总篇幅

- 10 章 × 平均 5800 字 ≈ 58000 字
- 加上代码示例和表格，总计约 62000-66000 字

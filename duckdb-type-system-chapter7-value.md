# DuckDB 类型系统深度解析 - 第七章：Value 类型系统

## 引言

虽然 DuckDB 的执行引擎以向量化处理为核心，但在某些场景下仍需处理单个值，例如常量表达式求值、参数绑定、结果展示等。`Value` 类是 DuckDB 中用于表示单个任意类型值的通用容器。本章将深入分析 Value 类的设计和使用。

## Value 类概述

`Value` 可以存储数据库中任何类型的单个值：

```cpp
// value.hpp
class Value {
private:
    LogicalType type_;         // 逻辑类型
    bool is_null;              // 是否为 NULL

    union Val {
        bool boolean;
        int8_t tinyint;
        int16_t smallint;
        int32_t integer;
        int64_t bigint;
        uint8_t utinyint;
        uint16_t usmallint;
        uint32_t uinteger;
        uint64_t ubigint;
        hugeint_t hugeint;
        uhugeint_t uhugeint;
        float float_;
        double double_;
        uintptr_t pointer;
        uint64_t hash;
        date_t date;
        dtime_t time;
        dtime_ns_t time_ns;
        dtime_tz_t timetz;
        timestamp_t timestamp;
        timestamp_sec_t timestamp_s;
        timestamp_ms_t timestamp_ms;
        timestamp_ns_t timestamp_ns;
        timestamp_tz_t timestamp_tz;
        interval_t interval;
    } value_;

    shared_ptr<ExtraValueInfo> value_info_;  // 扩展信息（字符串、嵌套类型等）

public:
    inline const LogicalType &type() const { return type_; }
    inline bool IsNull() const { return is_null; }
};
```

## 构造函数

Value 提供了丰富的构造函数，支持从各种 C++ 类型隐式或显式构造：

```cpp
// 空 NULL 值
explicit Value(LogicalType type = LogicalType::SQLNULL);

// 从基本类型构造（允许隐式转换）
Value(int32_t val);     // INTEGER
Value(int64_t val);     // BIGINT
Value(float val);       // FLOAT
Value(double val);      // DOUBLE
Value(const char *val); // VARCHAR
Value(string val);      // VARCHAR
Value(string_t val);    // VARCHAR
Value(std::nullptr_t);  // NULL

// 显式布尔构造
explicit Value(bool val);

// 拷贝和移动构造
Value(const Value &other);
Value(Value &&other) noexcept;
```

## 静态工厂方法

为了类型安全和代码清晰，Value 提供了大量静态工厂方法：

### 基本类型

```cpp
static Value BOOLEAN(bool value);
static Value TINYINT(int8_t value);
static Value SMALLINT(int16_t value);
static Value INTEGER(int32_t value);
static Value BIGINT(int64_t value);
static Value UTINYINT(uint8_t value);
static Value USMALLINT(uint16_t value);
static Value UINTEGER(uint32_t value);
static Value UBIGINT(uint64_t value);
static Value HUGEINT(hugeint_t value);
static Value UHUGEINT(uhugeint_t value);
static Value FLOAT(float value);
static Value DOUBLE(double value);
```

### 日期时间类型

```cpp
// DATE
static Value DATE(date_t date);
static Value DATE(int32_t year, int32_t month, int32_t day);

// TIME
static Value TIME(dtime_t time);
static Value TIME(int32_t hour, int32_t min, int32_t sec, int32_t micros);
static Value TIME_NS(dtime_ns_t time);
static Value TIMETZ(dtime_tz_t time);

// TIMESTAMP
static Value TIMESTAMP(timestamp_t timestamp);
static Value TIMESTAMP(date_t date, dtime_t time);
static Value TIMESTAMP(int32_t year, int32_t month, int32_t day,
                       int32_t hour, int32_t min, int32_t sec, int32_t micros);
static Value TIMESTAMPSEC(timestamp_sec_t timestamp);
static Value TIMESTAMPMS(timestamp_ms_t timestamp);
static Value TIMESTAMPNS(timestamp_ns_t timestamp);
static Value TIMESTAMPTZ(timestamp_tz_t timestamp);

// INTERVAL
static Value INTERVAL(int32_t months, int32_t days, int64_t micros);
static Value INTERVAL(interval_t interval);
```

### DECIMAL 类型

```cpp
static Value DECIMAL(int16_t value, uint8_t width, uint8_t scale);
static Value DECIMAL(int32_t value, uint8_t width, uint8_t scale);
static Value DECIMAL(int64_t value, uint8_t width, uint8_t scale);
static Value DECIMAL(hugeint_t value, uint8_t width, uint8_t scale);
```

### 嵌套类型

```cpp
// STRUCT
static Value STRUCT(child_list_t<Value> values);
static Value STRUCT(const LogicalType &type, vector<Value> struct_values);

// LIST
static Value LIST(const LogicalType &child_type, vector<Value> values);
static Value LIST(vector<Value> values);  // 类型从第一个元素推断

// ARRAY
static Value ARRAY(const LogicalType &type, vector<Value> values);

// MAP
static Value MAP(const LogicalType &child_type, vector<Value> values);
static Value MAP(const LogicalType &key_type, const LogicalType &value_type,
                 vector<Value> keys, vector<Value> values);

// UNION
static Value UNION(child_list_t<LogicalType> members, uint8_t tag, Value value);
```

### 二进制类型

```cpp
static Value BLOB(const_data_ptr_t data, idx_t len);
static Value BLOB(const string &data);  // 解析 \x 转义
static Value BLOB_RAW(const string &data);  // 原始数据

static Value BIT(const_data_ptr_t data, idx_t len);
static Value BIT(const string &data);
```

### 特殊类型

```cpp
static Value UUID(const string &value);
static Value UUID(hugeint_t value);
static Value HASH(hash_t value);
static Value POINTER(uintptr_t value);
static Value ENUM(uint64_t value, const LogicalType &original_type);
static Value AGGREGATE_STATE(const LogicalType &type, const_data_ptr_t data, idx_t len);
```

### 数值边界

```cpp
static Value MinimumValue(const LogicalType &type);   // 最小值
static Value MaximumValue(const LogicalType &type);   // 最大值
static Value NegativeInfinity(const LogicalType &type); // 负无穷
static Value Infinity(const LogicalType &type);       // 正无穷
static Value Numeric(const LogicalType &type, int64_t value);
```

## 值访问

### GetValue 模板方法

带类型转换的值访问：

```cpp
template <class T>
T GetValue() const;

// 使用示例
Value v = Value::INTEGER(42);
int64_t i = v.GetValue<int64_t>();  // 自动转换为 int64_t
string s = v.GetValue<string>();    // 自动转换为字符串
```

### GetValueUnsafe 模板方法

不进行类型检查的直接访问（高性能场景）：

```cpp
template <class T>
T GetValueUnsafe() const;

// 使用示例（必须确保类型匹配）
Value v = Value::INTEGER(42);
int32_t i = v.GetValueUnsafe<int32_t>();  // 直接访问，无转换
```

### 类型特定访问器

为常用类型提供的便捷访问器：

```cpp
struct BooleanValue {
    static bool Get(const Value &value);
};

struct TinyIntValue {
    static int8_t Get(const Value &value);
};

struct SmallIntValue {
    static int16_t Get(const Value &value);
};

struct IntegerValue {
    static int32_t Get(const Value &value);
};

struct BigIntValue {
    static int64_t Get(const Value &value);
};

struct HugeIntValue {
    static hugeint_t Get(const Value &value);
};

struct FloatValue {
    static float Get(const Value &value);
};

struct DoubleValue {
    static double Get(const Value &value);
};

struct StringValue {
    static const string &Get(const Value &value);
};

struct DateValue {
    static date_t Get(const Value &value);
};

struct TimeValue {
    static dtime_t Get(const Value &value);
};

struct TimestampValue {
    static timestamp_t Get(const Value &value);
};

struct IntervalValue {
    static interval_t Get(const Value &value);
};
```

### 嵌套类型访问器

```cpp
struct StructValue {
    static const vector<Value> &GetChildren(const Value &value);
};

struct ListValue {
    static const vector<Value> &GetChildren(const Value &value);
};

struct ArrayValue {
    static const vector<Value> &GetChildren(const Value &value);
};

struct MapValue {
    static const vector<Value> &GetChildren(const Value &value);
};

struct UnionValue {
    static const Value &GetValue(const Value &value);
    static uint8_t GetTag(const Value &value);
    static const LogicalType &GetType(const Value &value);
};

// 通用整数访问器（用于 DECIMAL、DATE 等内部使用整数存储的类型）
struct IntegralValue {
    static hugeint_t Get(const Value &value);
};
```

## CreateValue 模板

从 C++ 类型创建 Value：

```cpp
template <class T>
static Value CreateValue(T value);

// 特化示例
template <>
Value Value::CreateValue(bool value) {
    return Value::BOOLEAN(value);
}

template <>
Value Value::CreateValue(int32_t value) {
    return Value::INTEGER(value);
}

template <>
Value Value::CreateValue(string value) {
    return Value(std::move(value));
}

template <>
Value Value::CreateValue(date_t value) {
    return Value::DATE(value);
}
```

## 类型转换

### CastAs

将 Value 转换为目标类型：

```cpp
// 使用默认转换规则
Value CastAs(ClientContext &context, const LogicalType &target_type,
             bool strict = false) const;

// 使用指定的转换函数集
Value CastAs(CastFunctionSet &set, GetCastFunctionInput &get_input,
             const LogicalType &target_type, bool strict = false) const;

// 使用默认转换函数集
Value DefaultCastAs(const LogicalType &target_type, bool strict = false) const;
```

### TryCastAs

尝试转换，失败时不抛出异常：

```cpp
bool TryCastAs(ClientContext &context, const LogicalType &target_type,
               Value &new_value, string *error_message, bool strict = false) const;

bool TryCastAs(ClientContext &context, const LogicalType &target_type,
               bool strict = false);  // 就地转换

bool DefaultTryCastAs(const LogicalType &target_type, Value &new_value,
                      string *error_message, bool strict = false) const;
```

### Reinterpret

重新解释类型（不改变底层数据）：

```cpp
void Reinterpret(LogicalType new_type);
```

## 比较操作

Value 支持完整的比较操作：

```cpp
bool operator==(const Value &rhs) const;
bool operator!=(const Value &rhs) const;
bool operator<(const Value &rhs) const;
bool operator>(const Value &rhs) const;
bool operator<=(const Value &rhs) const;
bool operator>=(const Value &rhs) const;

// 与整数的比较
bool operator==(const int64_t &rhs) const;
bool operator!=(const int64_t &rhs) const;
bool operator<(const int64_t &rhs) const;
bool operator>(const int64_t &rhs) const;
bool operator<=(const int64_t &rhs) const;
bool operator>=(const int64_t &rhs) const;
```

### 近似相等

用于浮点数和测试场景：

```cpp
// 近似相等（NULL 等于 NULL，浮点数使用容差比较）
static bool ValuesAreEqual(ClientContext &context,
                            const Value &result_value, const Value &value);

// NOT DISTINCT FROM 语义（遵循 SQL 标准）
static bool NotDistinctFrom(const Value &lvalue, const Value &rvalue);
```

## 特殊值检测

```cpp
// 浮点数有限性检测
static bool FloatIsFinite(float value);
static bool DoubleIsFinite(double value);

// NaN 检测
template <class T>
static bool IsNan(T value);

template <>
bool Value::IsNan(float input);
template <>
bool Value::IsNan(double input);

// 通用有限性检测
template <class T>
static bool IsFinite(T value);

template <>
bool Value::IsFinite(float input);
template <>
bool Value::IsFinite(double input);
template <>
bool Value::IsFinite(date_t input);
template <>
bool Value::IsFinite(timestamp_t input);
// ...
```

### 字符串有效性

```cpp
static bool StringIsValid(const char *str, idx_t length);
static bool StringIsValid(const string &str);
```

## 字符串转换

```cpp
// 转换为人类可读的字符串
string ToString() const;

// 转换为 SQL 可解析的字符串
string ToSQLString() const;

// 打印到标准输出
void Print() const;

// 流输出
friend std::ostream &operator<<(std::ostream &out, const Value &val);
```

## 哈希

```cpp
hash_t Hash() const;
```

## 序列化

```cpp
void Serialize(Serializer &serializer) const;
static Value Deserialize(Deserializer &deserializer);
```

## ExtraValueInfo

对于需要额外存储的类型（如字符串、嵌套类型），使用 `ExtraValueInfo`：

```cpp
struct ExtraValueInfo {
    // 虚基类，用于存储字符串和嵌套值
};

// 字符串存储
struct StringValueInfo : ExtraValueInfo {
    string str;
};

// 嵌套类型存储
struct NestedValueInfo : ExtraValueInfo {
    vector<Value> values;
};
```

## 使用示例

### 基本用法

```cpp
// 创建值
Value int_val = Value::INTEGER(42);
Value str_val = Value("Hello, World!");
Value date_val = Value::DATE(2024, 1, 15);
Value null_val = Value(LogicalType::INTEGER);  // NULL INTEGER

// 访问值
int32_t i = IntegerValue::Get(int_val);
string s = StringValue::Get(str_val);
date_t d = DateValue::Get(date_val);

// 检查 NULL
if (null_val.IsNull()) {
    // 处理 NULL
}

// 类型转换
Value converted = int_val.DefaultCastAs(LogicalType::VARCHAR);
string converted_str = StringValue::Get(converted);  // "42"
```

### 嵌套类型

```cpp
// 创建 STRUCT
Value struct_val = Value::STRUCT({
    {"name", Value("Alice")},
    {"age", Value::INTEGER(30)}
});

// 访问 STRUCT 字段
auto &children = StructValue::GetChildren(struct_val);
string name = StringValue::Get(children[0]);
int32_t age = IntegerValue::Get(children[1]);

// 创建 LIST
Value list_val = Value::LIST(LogicalType::INTEGER, {
    Value::INTEGER(1),
    Value::INTEGER(2),
    Value::INTEGER(3)
});

// 访问 LIST 元素
auto &elements = ListValue::GetChildren(list_val);
for (auto &elem : elements) {
    int32_t v = IntegerValue::Get(elem);
}

// 创建 MAP
Value map_val = Value::MAP(
    LogicalType::VARCHAR,
    LogicalType::INTEGER,
    {Value("a"), Value("b")},
    {Value::INTEGER(1), Value::INTEGER(2)}
);
```

### 与 Vector 交互

```cpp
// 从 Vector 获取 Value
Value val = vector.GetValue(index);

// 设置 Vector 的值
vector.SetValue(index, Value::INTEGER(42));

// 从 DataChunk 获取值
Value val = chunk.GetValue(col_idx, row_idx);
chunk.SetValue(col_idx, row_idx, val);
```

## 性能考虑

Value 类为单值操作提供便利，但有以下性能特点：

1. **内存分配**：字符串和嵌套类型需要堆分配
2. **类型擦除**：运行时类型检查有开销
3. **拷贝开销**：深拷贝嵌套结构可能昂贵

**最佳实践**：
- 批量操作优先使用 Vector 和 DataChunk
- Value 适合常量、参数绑定、结果展示等场景
- 使用 `GetValueUnsafe` 在确定类型时避免检查开销
- 使用移动语义减少拷贝

## 小结

本章详细介绍了 DuckDB 的 Value 类型系统：

1. **Value 结构**：
   - 联合体存储固定大小类型
   - ExtraValueInfo 存储可变大小类型
   - LogicalType 标记类型信息

2. **工厂方法**：
   - 类型安全的静态构造方法
   - 支持所有 DuckDB 类型

3. **值访问**：
   - GetValue：带类型转换
   - GetValueUnsafe：无检查直接访问
   - 类型特定访问器：便捷且类型安全

4. **类型转换**：
   - CastAs/TryCastAs：完整的类型转换支持
   - Reinterpret：重新解释类型

5. **比较与操作**：
   - 完整的比较运算符
   - 特殊值检测（NaN、Infinity）
   - 哈希和序列化支持

Value 是 DuckDB 类型系统中连接静态类型（C++）和动态类型（SQL）的桥梁，为单值操作提供了统一、类型安全的接口。

下一章将探讨类型转换系统，了解 DuckDB 如何在不同类型之间进行转换。

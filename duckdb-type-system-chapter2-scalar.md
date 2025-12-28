# DuckDB 类型系统深度解析 - 第二章：标量类型实现

## 引言

标量类型是 DuckDB 类型系统的基础，包括数值类型、字符串类型以及特殊的二进制类型。本章将深入分析这些标量类型的内部表示和实现细节，揭示 DuckDB 如何在保证类型安全的同时实现高效的数据存储和处理。

## 整数类型

DuckDB 支持完整的整数类型家族，覆盖从 8 位到 128 位的有符号和无符号整数。

### 标准整数类型

| SQL 类型 | LogicalTypeId | PhysicalType | C++ 类型 | 范围 |
|---------|--------------|--------------|----------|------|
| TINYINT | TINYINT | INT8 | int8_t | -128 ~ 127 |
| SMALLINT | SMALLINT | INT16 | int16_t | -32,768 ~ 32,767 |
| INTEGER | INTEGER | INT32 | int32_t | -2^31 ~ 2^31-1 |
| BIGINT | BIGINT | INT64 | int64_t | -2^63 ~ 2^63-1 |
| UTINYINT | UTINYINT | UINT8 | uint8_t | 0 ~ 255 |
| USMALLINT | USMALLINT | UINT16 | uint16_t | 0 ~ 65,535 |
| UINTEGER | UINTEGER | UINT32 | uint32_t | 0 ~ 4,294,967,295 |
| UBIGINT | UBIGINT | UINT64 | uint64_t | 0 ~ 2^64-1 |

这些类型直接映射到 C++ 原生类型，没有额外的内存开销：

```cpp
// 整数类型的物理类型映射（types.cpp）
PhysicalType LogicalType::GetInternalType() {
    switch (id_) {
    case LogicalTypeId::TINYINT:
        return PhysicalType::INT8;
    case LogicalTypeId::UTINYINT:
        return PhysicalType::UINT8;
    case LogicalTypeId::SMALLINT:
        return PhysicalType::INT16;
    case LogicalTypeId::USMALLINT:
        return PhysicalType::UINT16;
    case LogicalTypeId::INTEGER:
        return PhysicalType::INT32;
    case LogicalTypeId::UINTEGER:
        return PhysicalType::UINT32;
    case LogicalTypeId::BIGINT:
        return PhysicalType::INT64;
    case LogicalTypeId::UBIGINT:
        return PhysicalType::UINT64;
    // ...
    }
}
```

### HUGEINT 和 UHUGEINT（128 位整数）

对于超出 64 位范围的整数，DuckDB 提供了 128 位整数类型：

```cpp
// hugeint.hpp
struct hugeint_t {
    uint64_t lower;  // 低 64 位
    int64_t upper;   // 高 64 位（有符号）

    hugeint_t() = default;
    hugeint_t(int64_t value);
    constexpr hugeint_t(int64_t upper, uint64_t lower)
        : lower(lower), upper(upper) {}

    // 算术运算符
    hugeint_t operator+(const hugeint_t &rhs) const;
    hugeint_t operator-(const hugeint_t &rhs) const;
    hugeint_t operator*(const hugeint_t &rhs) const;
    hugeint_t operator/(const hugeint_t &rhs) const;
    hugeint_t operator%(const hugeint_t &rhs) const;
    hugeint_t operator-() const;

    // 比较运算符
    bool operator==(const hugeint_t &rhs) const;
    bool operator<(const hugeint_t &rhs) const;
    // ...

    // 位运算符
    hugeint_t operator>>(const hugeint_t &rhs) const;
    hugeint_t operator<<(const hugeint_t &rhs) const;
    hugeint_t operator&(const hugeint_t &rhs) const;
    hugeint_t operator|(const hugeint_t &rhs) const;
    hugeint_t operator^(const hugeint_t &rhs) const;
    hugeint_t operator~() const;

    // 类型转换
    explicit operator int64_t() const;
    explicit operator uint64_t() const;
    // ...
};
```

无符号版本 `uhugeint_t` 结构类似，但高 64 位也是无符号的：

```cpp
// uhugeint.hpp
struct uhugeint_t {
    uint64_t lower;
    uint64_t upper;  // 无符号

    // 类似的运算符重载...
};
```

128 位整数主要用于：
- 高精度 DECIMAL 计算
- UUID 存储（内部使用 hugeint_t）
- 大数值运算

### 整数类型层次

```
                 ┌─────────────────────────────────────────┐
                 │           128-bit Integers              │
                 │  hugeint_t (INT128)  uhugeint_t (UINT128)│
                 └────────────────┬────────────────────────┘
                                  │
                 ┌────────────────┴────────────────────────┐
                 │            64-bit Integers              │
                 │    int64_t (INT64)    uint64_t (UINT64) │
                 └────────────────┬────────────────────────┘
                                  │
                 ┌────────────────┴────────────────────────┐
                 │            32-bit Integers              │
                 │    int32_t (INT32)    uint32_t (UINT32) │
                 └────────────────┬────────────────────────┘
                                  │
                 ┌────────────────┴────────────────────────┐
                 │            16-bit Integers              │
                 │    int16_t (INT16)    uint16_t (UINT16) │
                 └────────────────┬────────────────────────┘
                                  │
                 ┌────────────────┴────────────────────────┐
                 │             8-bit Integers              │
                 │     int8_t (INT8)      uint8_t (UINT8)  │
                 └─────────────────────────────────────────┘
```

## 浮点类型

DuckDB 支持 IEEE 754 标准的浮点类型：

| SQL 类型 | LogicalTypeId | PhysicalType | C++ 类型 | 精度 |
|---------|--------------|--------------|----------|------|
| FLOAT / REAL | FLOAT | FLOAT | float | ~7 位有效数字 |
| DOUBLE | DOUBLE | DOUBLE | double | ~15 位有效数字 |

```cpp
case LogicalTypeId::FLOAT:
    return PhysicalType::FLOAT;
case LogicalTypeId::DOUBLE:
    return PhysicalType::DOUBLE;
```

### 浮点数近似比较

由于浮点数的精度问题，DuckDB 提供了近似比较函数：

```cpp
// types.cpp
bool ApproxEqual(float ldecimal, float rdecimal) {
    if (Value::IsNan(ldecimal) && Value::IsNan(rdecimal)) {
        return true;
    }
    if (!Value::FloatIsFinite(ldecimal) || !Value::FloatIsFinite(rdecimal)) {
        return ldecimal == rdecimal;
    }
    float epsilon = static_cast<float>(std::fabs(rdecimal) * 0.01 + 0.00000001);
    return std::fabs(ldecimal - rdecimal) <= epsilon;
}

bool ApproxEqual(double ldecimal, double rdecimal) {
    if (Value::IsNan(ldecimal) && Value::IsNan(rdecimal)) {
        return true;
    }
    if (!Value::DoubleIsFinite(ldecimal) || !Value::DoubleIsFinite(rdecimal)) {
        return ldecimal == rdecimal;
    }
    double epsilon = std::fabs(rdecimal) * 0.01 + 0.00000001;
    return std::fabs(ldecimal - rdecimal) <= epsilon;
}
```

## DECIMAL 类型

DECIMAL 是 DuckDB 中最复杂的数值类型，它提供精确的十进制运算，避免浮点数的精度问题。

### DECIMAL 精度与存储

DECIMAL 类型有两个参数：
- **width（精度）**：总位数，范围 1-38
- **scale（小数位数）**：小数点后的位数

```cpp
// decimal.hpp
template <class PHYSICAL_TYPE>
struct DecimalWidth {};

template <>
struct DecimalWidth<int16_t> {
    static constexpr uint8_t max = 4;   // DECIMAL(1-4)
};

template <>
struct DecimalWidth<int32_t> {
    static constexpr uint8_t max = 9;   // DECIMAL(5-9)
};

template <>
struct DecimalWidth<int64_t> {
    static constexpr uint8_t max = 18;  // DECIMAL(10-18)
};

template <>
struct DecimalWidth<hugeint_t> {
    static constexpr uint8_t max = 38;  // DECIMAL(19-38)
};

class Decimal {
public:
    static constexpr uint8_t MAX_WIDTH_INT16 = 4;
    static constexpr uint8_t MAX_WIDTH_INT32 = 9;
    static constexpr uint8_t MAX_WIDTH_INT64 = 18;
    static constexpr uint8_t MAX_WIDTH_INT128 = 38;
    static constexpr uint8_t MAX_WIDTH_DECIMAL = MAX_WIDTH_INT128;

    static string ToString(int16_t value, uint8_t width, uint8_t scale);
    static string ToString(int32_t value, uint8_t width, uint8_t scale);
    static string ToString(int64_t value, uint8_t width, uint8_t scale);
    static string ToString(hugeint_t value, uint8_t width, uint8_t scale);
};
```

### DECIMAL 物理类型选择

DECIMAL 的物理类型根据精度自动选择最小的整数类型：

```cpp
// types.cpp
case LogicalTypeId::DECIMAL: {
    if (!type_info_) {
        return PhysicalType::INVALID;
    }
    auto width = DecimalType::GetWidth(*this);
    if (width <= Decimal::MAX_WIDTH_INT16) {
        return PhysicalType::INT16;    // DECIMAL(1-4)
    } else if (width <= Decimal::MAX_WIDTH_INT32) {
        return PhysicalType::INT32;    // DECIMAL(5-9)
    } else if (width <= Decimal::MAX_WIDTH_INT64) {
        return PhysicalType::INT64;    // DECIMAL(10-18)
    } else if (width <= Decimal::MAX_WIDTH_INT128) {
        return PhysicalType::INT128;   // DECIMAL(19-38)
    } else {
        throw InternalException("Decimal width too large");
    }
}
```

### DECIMAL 类型信息

DECIMAL 的精度信息通过 `DecimalTypeInfo` 存储：

```cpp
// extra_type_info.hpp
struct DecimalTypeInfo : public ExtraTypeInfo {
    uint8_t width;   // 总位数
    uint8_t scale;   // 小数位数

    DecimalTypeInfo(uint8_t width_p, uint8_t scale_p)
        : ExtraTypeInfo(ExtraTypeInfoType::DECIMAL_TYPE_INFO),
          width(width_p), scale(scale_p) {
        D_ASSERT(width_p >= scale_p);
    }
};
```

辅助类提供便捷访问：

```cpp
// types.cpp
uint8_t DecimalType::GetWidth(const LogicalType &type) {
    D_ASSERT(type.id() == LogicalTypeId::DECIMAL);
    auto info = type.AuxInfo();
    D_ASSERT(info);
    return info->Cast<DecimalTypeInfo>().width;
}

uint8_t DecimalType::GetScale(const LogicalType &type) {
    D_ASSERT(type.id() == LogicalTypeId::DECIMAL);
    auto info = type.AuxInfo();
    D_ASSERT(info);
    return info->Cast<DecimalTypeInfo>().scale;
}
```

### DECIMAL 运算原理

DECIMAL 值以整数形式存储，实际值 = 存储值 / 10^scale：

```
DECIMAL(5,2) 存储 123.45:
  存储值 = 12345 (int16_t)
  实际值 = 12345 / 10^2 = 123.45
```

DECIMAL 转字符串时需要正确处理小数点位置：

```cpp
// decimal.cpp
template <class SIGNED>
string TemplatedDecimalToString(SIGNED value, uint8_t width, uint8_t scale) {
    auto len = DecimalToString::DecimalLength<SIGNED>(value, width, scale);
    auto data = make_unsafe_uniq_array_uninitialized<char>(len + 1);
    DecimalToString::FormatDecimal<SIGNED>(value, width, scale, data.get(), len);
    return string(data.get(), len);
}
```

## 字符串类型

DuckDB 的字符串处理是其高性能的关键之一，采用了精心设计的内联优化。

### string_t 结构

`string_t` 是 DuckDB 的核心字符串类型，大小固定为 16 字节：

```cpp
// string_type.hpp
struct string_t {
    static constexpr idx_t PREFIX_BYTES = 4;    // 前缀字节数
    static constexpr idx_t INLINE_BYTES = 12;   // 内联存储字节数
    static constexpr idx_t HEADER_SIZE = sizeof(uint32_t) + PREFIX_BYTES; // 8 bytes
    static constexpr idx_t MAX_STRING_SIZE = NumericLimits<uint32_t>::Maximum();

private:
    union {
        // 短字符串：直接内联存储
        struct {
            uint32_t length;     // 4 bytes
            char inlined[12];    // 12 bytes
        } inlined;

        // 长字符串：存储前缀 + 指针
        struct {
            uint32_t length;     // 4 bytes
            char prefix[4];      // 4 bytes（前4字节用于快速比较）
            char *ptr;           // 8 bytes（指向实际数据）
        } pointer;
    } value;
};
```

### 内联优化

字符串长度 ≤ 12 字节时，数据直接存储在 `string_t` 结构内部：

```cpp
bool IsInlined() const {
    return GetSize() <= INLINE_LENGTH;  // INLINE_LENGTH = 12
}

const char *GetData() const {
    return IsInlined() ? value.inlined.inlined : value.pointer.ptr;
}
```

内存布局：

```
短字符串 (≤12 字节):
┌────────────┬────────────────────────────────────────┐
│   length   │           inlined data (12 bytes)      │
│  (4 bytes) │                                        │
└────────────┴────────────────────────────────────────┘
                         16 bytes

长字符串 (>12 字节):
┌────────────┬────────────┬───────────────────────────┐
│   length   │   prefix   │       pointer             │
│  (4 bytes) │  (4 bytes) │      (8 bytes)            │
└────────────┴────────────┴───────────────────────────┘
                         16 bytes
```

### 字符串比较优化

`string_t` 的比较利用了前缀和长度信息进行快速判断：

```cpp
// 相等性比较
static inline bool Equals(const string_t &a, const string_t &b) {
    // 批量比较前 8 字节（length + 前4字节数据/前缀）
    uint64_t a_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&a));
    uint64_t b_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&b));
    if (a_bulk_comp != b_bulk_comp) {
        // 长度或前缀不同，直接返回 false
        return false;
    }

    // 比较后 8 字节
    a_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&a) + 8u);
    b_bulk_comp = Load<uint64_t>(const_data_ptr_cast(&b) + 8u);
    if (a_bulk_comp == b_bulk_comp) {
        // 完全相同（内联字符串）或指向同一地址
        return true;
    }

    if (!a.IsInlined()) {
        // 长字符串：需要完整比较
        if (memcmp(a.value.pointer.ptr, b.value.pointer.ptr, a.GetSize()) == 0) {
            return true;
        }
    }
    return false;
}

// 大于比较
static bool GreaterThan(const string_t &left, const string_t &right) {
    const uint32_t left_length = left.GetSize();
    const uint32_t right_length = right.GetSize();
    const uint32_t min_length = std::min(left_length, right_length);

    // 首先比较前缀
    uint32_t a_prefix = Load<uint32_t>(left.GetPrefix());
    uint32_t b_prefix = Load<uint32_t>(right.GetPrefix());
    if (a_prefix != b_prefix) {
        return BSwapIfLE(a_prefix) > BSwapIfLE(b_prefix);
    }

    // 前缀相同，完整比较
    auto memcmp_res = memcmp(left.GetData(), right.GetData(), min_length);
    return memcmp_res > 0 || (memcmp_res == 0 && left_length > right_length);
}
```

### 字符串类型变体

DuckDB 有多种基于 `string_t` 的字符串类型：

| SQL 类型 | LogicalTypeId | 特点 |
|---------|--------------|------|
| VARCHAR | VARCHAR | 可变长度字符串 |
| CHAR(n) | CHAR | 固定长度字符串（空格填充） |
| TEXT | VARCHAR | VARCHAR 的别名 |
| BLOB | BLOB | 二进制数据 |
| BIT | BIT | 位串 |

它们都使用相同的物理类型 `PhysicalType::VARCHAR`：

```cpp
case LogicalTypeId::VARCHAR:
case LogicalTypeId::CHAR:
case LogicalTypeId::BLOB:
case LogicalTypeId::BIT:
    return PhysicalType::VARCHAR;
```

### 字符串排序规则（Collation）

VARCHAR 类型可以附加排序规则：

```cpp
struct StringTypeInfo : public ExtraTypeInfo {
    string collation;

    explicit StringTypeInfo(string collation_p)
        : ExtraTypeInfo(ExtraTypeInfoType::STRING_TYPE_INFO),
          collation(std::move(collation_p)) {}
};

// 获取排序规则
string StringType::GetCollation(const LogicalType &type) {
    if (type.id() != LogicalTypeId::VARCHAR) {
        return string();
    }
    auto info = type.AuxInfo();
    if (!info || info->type == ExtraTypeInfoType::GENERIC_TYPE_INFO) {
        return string();
    }
    return info->Cast<StringTypeInfo>().collation;
}

// 创建带排序规则的 VARCHAR
LogicalType LogicalType::VARCHAR_COLLATION(string collation) {
    auto string_info = make_shared_ptr<StringTypeInfo>(std::move(collation));
    return LogicalType(LogicalTypeId::VARCHAR, std::move(string_info));
}
```

## BLOB 类型

BLOB（Binary Large Object）用于存储任意二进制数据。

### BLOB 与字符串的转换

BLOB 使用十六进制编码与字符串相互转换：

```cpp
// blob.hpp
class Blob {
public:
    // 十六进制映射表
    static constexpr const char *HEX_TABLE = "0123456789ABCDEF";
    static const int HEX_MAP[256];  // 反向映射

    // Base64 编码支持
    static constexpr const char *BASE64_MAP =
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    static constexpr const char BASE64_PADDING = '=';

    // BLOB -> 字符串
    static idx_t GetStringSize(string_t blob);
    static void ToString(string_t blob, char *output);
    static string ToString(string_t blob);

    // 字符串 -> BLOB
    static bool TryGetBlobSize(string_t str, idx_t &result_size, CastParameters &params);
    static void ToBlob(string_t str, data_ptr_t output);
    static string ToBlob(string_t str);

    // Base64 编码
    static string ToBase64(string_t blob);
    static idx_t ToBase64Size(string_t blob);
    static string FromBase64(string_t str);
    static idx_t FromBase64Size(string_t str);
};
```

BLOB 的字符串表示使用 `\x` 前缀的十六进制格式：

```sql
-- 示例
SELECT '\xDEADBEEF'::BLOB;  -- 4 字节二进制数据
```

## BIT 类型

BIT 类型用于存储位串，内部使用 `string_t` 存储：

```cpp
// bit.hpp
using bitstring_t = duckdb::string_t;

class Bit {
public:
    // 位串信息
    static idx_t BitLength(bitstring_t bits);    // 位数
    static idx_t BitCount(bitstring_t bits);     // 置位数量
    static idx_t OctetLength(bitstring_t bits);  // 字节数

    // 位操作
    static idx_t GetBit(bitstring_t bit_string, idx_t n);
    static void SetBit(bitstring_t &bit_string, idx_t n, idx_t new_value);

    // 位运算
    static void RightShift(const bitstring_t &bit_string, idx_t shift, bitstring_t &result);
    static void LeftShift(const bitstring_t &bit_string, idx_t shift, bitstring_t &result);
    static void BitwiseAnd(const bitstring_t &rhs, const bitstring_t &lhs, bitstring_t &result);
    static void BitwiseOr(const bitstring_t &rhs, const bitstring_t &lhs, bitstring_t &result);
    static void BitwiseXor(const bitstring_t &rhs, const bitstring_t &lhs, bitstring_t &result);
    static void BitwiseNot(const bitstring_t &rhs, bitstring_t &result);

    // 与数值类型的转换
    template <class T>
    static void NumericToBit(T numeric, bitstring_t &output_str);

    template <class T>
    static void BitToNumeric(bitstring_t bit, T &output_num);
};
```

BIT 字符串的存储格式：
- 第一个字节存储填充位信息
- 后续字节存储实际位数据

```cpp
template <class T>
void Bit::NumericToBit(T numeric, bitstring_t &output_str) {
    D_ASSERT(output_str.GetSize() >= sizeof(T) + 1);

    auto le_numeric = BSwapIfBE(numeric);
    auto output = output_str.GetDataWriteable();
    auto data = const_data_ptr_cast(&le_numeric);

    *output = 0; // 设置填充为 0
    ++output;
    for (idx_t idx = 0; idx < sizeof(T); ++idx) {
        output[idx] = static_cast<char>(data[sizeof(T) - idx - 1]);
    }
    Bit::Finalize(output_str);
}
```

## UUID 类型

UUID（Universally Unique Identifier）使用 128 位整数存储：

```cpp
// uuid.hpp
class BaseUUID {
public:
    constexpr static const uint8_t STRING_SIZE = 36;  // xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

    // 字符串转换
    static bool FromString(const string &str, hugeint_t &result, bool strict = false);
    static void ToString(hugeint_t input, char *buf);
    static string ToString(hugeint_t input);

    // 二进制转换
    static hugeint_t FromBlob(const_data_ptr_t input);
    static void ToBlob(hugeint_t input, data_ptr_t output);

    // 与 uhugeint_t 相互转换
    static hugeint_t FromUHugeint(uhugeint_t input);
    static uhugeint_t ToUHugeint(hugeint_t input);
};

// UUID v4（随机生成）
class UUID : public BaseUUID {
public:
    static hugeint_t GenerateRandomUUID(RandomEngine &engine);
    static hugeint_t GenerateRandomUUID();
};

// UUID v7（时间排序）
class UUIDv7 : public BaseUUID {
public:
    static hugeint_t GenerateRandomUUID(RandomEngine &engine);
    static hugeint_t GenerateRandomUUID();
};
```

UUID 的物理类型是 `INT128`：

```cpp
case LogicalTypeId::UUID:
    return PhysicalType::INT128;
```

UUID 字符串格式为标准的 36 字符形式：`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

## INTERVAL 类型

INTERVAL 表示时间间隔，由三个组件组成：

```cpp
// interval.hpp
struct interval_t {
    int32_t months;  // 月份
    int32_t days;    // 天数
    int64_t micros;  // 微秒

    // 归一化
    void Normalize(int64_t &months, int64_t &days, int64_t &micros) const;
    interval_t Normalize() const;

    // 比较运算符（归一化后比较）
    bool operator==(const interval_t &right) const;
    bool operator>(const interval_t &right) const;
    bool operator<(const interval_t &right) const;
};
```

INTERVAL 物理类型是专用的 `INTERVAL`：

```cpp
case LogicalTypeId::INTERVAL:
    return PhysicalType::INTERVAL;

// 大小计算
case PhysicalType::INTERVAL:
    return sizeof(interval_t);  // 16 bytes
```

### INTERVAL 常量

```cpp
class Interval {
public:
    static constexpr const int32_t MONTHS_PER_YEAR = 12;
    static constexpr const int32_t DAYS_PER_WEEK = 7;
    static constexpr const int64_t DAYS_PER_MONTH = 30;  // 用于比较

    static constexpr const int64_t MICROS_PER_MSEC = 1000;
    static constexpr const int64_t MICROS_PER_SEC = 1000000;
    static constexpr const int64_t MICROS_PER_MINUTE = 60000000;
    static constexpr const int64_t MICROS_PER_HOUR = 3600000000;
    static constexpr const int64_t MICROS_PER_DAY = 86400000000;

    // INTERVAL 运算
    static date_t Add(date_t left, interval_t right);
    static timestamp_t Add(timestamp_t left, interval_t right);
    static interval_t Invert(interval_t interval);
    static interval_t GetAge(timestamp_t t1, timestamp_t t2);
};
```

### INTERVAL 归一化

INTERVAL 值在比较时需要归一化：

```cpp
void interval_t::Normalize(int64_t &months, int64_t &days, int64_t &micros) const {
    // 从微秒借位到天
    micros = this->micros;
    int64_t carry_days = micros / Interval::MICROS_PER_DAY;
    micros -= carry_days * Interval::MICROS_PER_DAY;

    // 从天借位到月
    days = this->days + carry_days;
    int64_t carry_months = days / Interval::DAYS_PER_MONTH;
    days -= carry_months * Interval::DAYS_PER_MONTH;

    months = this->months + carry_months;
}
```

## 布尔类型

布尔类型是最简单的标量类型：

```cpp
case LogicalTypeId::BOOLEAN:
    return PhysicalType::BOOL;

case PhysicalType::BOOL:
    return sizeof(bool);  // 1 byte
```

布尔值存储为单字节，但在向量中可能使用位压缩存储（validity mask）。

## 类型大小汇总

```cpp
idx_t GetTypeIdSize(PhysicalType type) {
    switch (type) {
    case PhysicalType::BOOL:
        return sizeof(bool);           // 1 byte
    case PhysicalType::INT8:
    case PhysicalType::UINT8:
        return sizeof(int8_t);         // 1 byte
    case PhysicalType::INT16:
    case PhysicalType::UINT16:
        return sizeof(int16_t);        // 2 bytes
    case PhysicalType::INT32:
    case PhysicalType::UINT32:
        return sizeof(int32_t);        // 4 bytes
    case PhysicalType::INT64:
    case PhysicalType::UINT64:
        return sizeof(int64_t);        // 8 bytes
    case PhysicalType::INT128:
    case PhysicalType::UINT128:
        return sizeof(hugeint_t);      // 16 bytes
    case PhysicalType::FLOAT:
        return sizeof(float);          // 4 bytes
    case PhysicalType::DOUBLE:
        return sizeof(double);         // 8 bytes
    case PhysicalType::VARCHAR:
        return sizeof(string_t);       // 16 bytes
    case PhysicalType::INTERVAL:
        return sizeof(interval_t);     // 16 bytes
    case PhysicalType::LIST:
        return sizeof(list_entry_t);   // 16 bytes (offset + len)
    // ...
    }
}
```

## 小结

本章深入分析了 DuckDB 标量类型的实现：

1. **整数类型**：从 8 位到 128 位的完整整数家族，直接映射到 C++ 原生类型
2. **浮点类型**：IEEE 754 标准的 FLOAT 和 DOUBLE
3. **DECIMAL 类型**：基于整数的精确十进制运算，自动选择最小存储类型
4. **字符串类型**：`string_t` 的 16 字节固定大小设计，短字符串内联优化
5. **二进制类型**：BLOB 的十六进制/Base64 编码，BIT 的位操作支持
6. **UUID 类型**：128 位整数存储，支持 v4 和 v7 版本
7. **INTERVAL 类型**：三分量时间间隔，支持归一化比较

这些标量类型的设计体现了 DuckDB 的核心设计理念：
- 使用固定大小的内存布局，便于向量化处理
- 短字符串内联优化，减少内存访问
- 自适应存储选择，平衡空间和性能
- 完整的 SQL 标准兼容性

下一章将探讨日期时间类型的实现，包括 DATE、TIME、TIMESTAMP 及其时区变体。

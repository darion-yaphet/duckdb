# DuckDB 类型系统深度解析 - 第三章：日期和时间类型

## 引言

日期和时间类型是数据库系统中最复杂的数据类型之一，涉及历法计算、时区处理、精度管理等多个维度。DuckDB 提供了完整的日期时间类型家族，支持从简单的日期到带时区的高精度时间戳。本章将深入分析这些类型的内部实现。

## 日期时间类型概览

DuckDB 的日期时间类型家族：

| SQL 类型 | LogicalTypeId | 物理类型 | 存储单位 | 大小 |
|---------|--------------|---------|---------|------|
| DATE | DATE | INT32 | 天数 | 4 bytes |
| TIME | TIME | INT64 | 微秒 | 8 bytes |
| TIME_NS | TIME_NS | INT64 | 纳秒 | 8 bytes |
| TIMESTAMP | TIMESTAMP | INT64 | 微秒 | 8 bytes |
| TIMESTAMP_S | TIMESTAMP_SEC | INT64 | 秒 | 8 bytes |
| TIMESTAMP_MS | TIMESTAMP_MS | INT64 | 毫秒 | 8 bytes |
| TIMESTAMP_NS | TIMESTAMP_NS | INT64 | 纳秒 | 8 bytes |
| TIMESTAMPTZ | TIMESTAMP_TZ | INT64 | 微秒(UTC) | 8 bytes |
| TIMETZ | TIME_TZ | UINT64 | 微秒+偏移 | 8 bytes |

## DATE 类型

DATE 类型表示日历日期，存储从 Unix Epoch（1970-01-01）开始的天数。

### date_t 结构

```cpp
// date.hpp
struct date_t {
    int32_t days;  // 从 1970-01-01 起的天数

    date_t() = default;
    explicit inline date_t(int32_t days_p) : days(days_p) {}

    // 显式转换
    explicit inline operator int32_t() const {
        return days;
    }

    // 比较运算符
    inline bool operator==(const date_t &rhs) const { return days == rhs.days; }
    inline bool operator<(const date_t &rhs) const { return days < rhs.days; }
    // ...

    // 算术运算
    inline date_t operator+(const int32_t &days) const {
        return date_t(this->days + days);
    }
    inline date_t operator-(const int32_t &days) const {
        return date_t(this->days - days);
    }

    // 特殊值
    static inline date_t infinity() {
        return date_t(NumericLimits<int32_t>::Maximum());
    }
    static inline date_t ninfinity() {
        return date_t(-NumericLimits<int32_t>::Maximum());
    }
    static inline date_t epoch() {
        return date_t(0);  // 1970-01-01
    }
};
```

### 日期范围

DuckDB 的日期范围极广：

```cpp
class Date {
public:
    // 最小日期：公元前 5877641 年 6 月 25 日
    constexpr static const int32_t DATE_MIN_YEAR = -5877641;
    constexpr static const int32_t DATE_MIN_MONTH = 6;
    constexpr static const int32_t DATE_MIN_DAY = 25;

    // 最大日期：公元 5881580 年 7 月 10 日
    constexpr static const int32_t DATE_MAX_YEAR = 5881580;
    constexpr static const int32_t DATE_MAX_MONTH = 7;
    constexpr static const int32_t DATE_MAX_DAY = 10;

    // Epoch 年份
    constexpr static const int32_t EPOCH_YEAR = 1970;

    // 400 年周期（用于快速计算）
    constexpr static const int32_t YEAR_INTERVAL = 400;
    constexpr static const int32_t DAYS_PER_YEAR_INTERVAL = 146097;
};
```

### 日期转换函数

```cpp
class Date {
public:
    // 字符串解析
    static date_t FromString(const string &str, bool strict = false);
    static date_t FromCString(const char *str, idx_t len, bool strict = false);

    // 字符串格式化
    static string ToString(date_t date);
    static string Format(int32_t year, int32_t month, int32_t day);

    // 年月日分解与组合
    static void Convert(date_t date, int32_t &out_year,
                        int32_t &out_month, int32_t &out_day);
    static date_t FromDate(int32_t year, int32_t month, int32_t day);
    static bool TryFromDate(int32_t year, int32_t month, int32_t day,
                            date_t &result);

    // Epoch 转换
    static int64_t Epoch(date_t date);           // 秒
    static int64_t EpochMicroseconds(date_t date);
    static int64_t EpochNanoseconds(date_t date);
    static date_t EpochToDate(int64_t epoch);

    // 日期组件提取
    static int32_t ExtractYear(date_t date);
    static int32_t ExtractMonth(date_t date);
    static int32_t ExtractDay(date_t date);
    static int32_t ExtractISODayOfTheWeek(date_t date);  // 1-7
    static int32_t ExtractDayOfTheYear(date_t date);     // 1-366
    static int64_t ExtractJulianDay(date_t date);

    // ISO 周
    static void ExtractISOYearWeek(date_t date, int32_t &year, int32_t &week);
    static int32_t ExtractISOWeekNumber(date_t date);
    static int32_t ExtractISOYearNumber(date_t date);

    // 辅助函数
    static bool IsLeapYear(int32_t year);
    static bool IsValid(int32_t year, int32_t month, int32_t day);
    static int32_t MonthDays(int32_t year, int32_t month);
};
```

### 日期常量表

DuckDB 预先计算了许多日期相关的常量表，用于快速转换：

```cpp
class Date {
public:
    // 月份名称
    static const string_t MONTH_NAMES[12];
    static const string_t MONTH_NAMES_ABBREVIATED[12];
    static const string_t DAY_NAMES[7];
    static const string_t DAY_NAMES_ABBREVIATED[7];

    // 每月天数
    static const int32_t NORMAL_DAYS[13];          // 普通年
    static const int32_t CUMULATIVE_DAYS[13];      // 累计天数
    static const int32_t LEAP_DAYS[13];            // 闰年
    static const int32_t CUMULATIVE_LEAP_DAYS[13]; // 闰年累计

    // 400 年周期的累计天数
    static const int32_t CUMULATIVE_YEAR_DAYS[401];

    // 日期到月份的映射（用于快速查找）
    static const int8_t MONTH_PER_DAY_OF_YEAR[365];
    static const int8_t LEAP_MONTH_PER_DAY_OF_YEAR[366];
};
```

### 特殊日期值

```cpp
struct DateSpecial {
    const char *str;  // 完整字符串
    const idx_t abbr; // 缩写长度
};

class Date {
public:
    static const DateSpecial PINF;   // "infinity"
    static const DateSpecial NINF;   // "-infinity"
    static const DateSpecial EPOCH;  // "epoch"

    static bool TryConvertDateSpecial(const char *buf, idx_t len,
                                      idx_t &pos, const DateSpecial &special);
};
```

## TIME 类型

TIME 类型表示一天中的时间，不包含日期信息。

### dtime_t 结构

```cpp
// datetime.hpp
struct dtime_t {
    int64_t micros;  // 从午夜开始的微秒数

    dtime_t() = default;
    explicit inline constexpr dtime_t(int64_t micros_p) : micros(micros_p) {}

    // 显式转换
    explicit inline operator int64_t() const { return micros; }
    explicit inline operator double() const { return static_cast<double>(micros); }

    // 比较运算符
    inline bool operator==(const dtime_t &rhs) const { return micros == rhs.micros; }
    inline bool operator<(const dtime_t &rhs) const { return micros < rhs.micros; }
    // ...

    // 算术运算
    inline dtime_t operator+(const int64_t &micros) const {
        return dtime_t(this->micros + micros);
    }
    inline int64_t operator-(const dtime_t &other) const {
        return this->micros - other.micros;
    }

    // 特殊值
    static inline dtime_t allballs() {  // 00:00:00
        return dtime_t(0);
    }
};
```

### 时间范围

时间值的有效范围是 00:00:00.000000 到 23:59:59.999999，对应 0 到 86,399,999,999 微秒：

```cpp
// 一天的微秒数
constexpr int64_t MICROS_PER_DAY = 24 * 60 * 60 * 1000000LL;  // 86,400,000,000
```

### TIME_NS 类型

纳秒精度时间：

```cpp
struct dtime_ns_t : public dtime_t {
    dtime_ns_t() = default;
    explicit inline constexpr dtime_ns_t(const int64_t nanos) : dtime_t(nanos) {}

    // 转换为微秒时间
    inline dtime_t time() const {
        return dtime_t(micros / 1000);  // micros 实际存储纳秒
    }
};
```

### Time 辅助类

```cpp
class Time {
public:
    // 字符串转换
    static dtime_t FromString(const string &str, bool strict = false,
                              optional_ptr<int32_t> nanos = nullptr);
    static dtime_t FromCString(const char *buf, idx_t len, bool strict = false,
                               optional_ptr<int32_t> nanos = nullptr);
    static string ToString(dtime_t time);

    // 时间组件转换
    static dtime_t FromTime(int32_t hour, int32_t minute,
                            int32_t second, int32_t microseconds = 0);
    static void Convert(dtime_t time, int32_t &out_hour, int32_t &out_min,
                        int32_t &out_sec, int32_t &out_micros);

    // 其他精度转换
    static dtime_t FromTimeMs(int64_t time_ms);
    static dtime_t FromTimeNs(int64_t time_ns);
    static int64_t ToNanoTime(int32_t hour, int32_t minute,
                              int32_t second, int32_t nanoseconds = 0);

    // 验证
    static bool IsValidTime(int32_t hour, int32_t minute,
                            int32_t second, int32_t microseconds);
};
```

## TIMETZ 类型（带时区的时间）

TIMETZ 将时间和 UTC 偏移量编码在一个 64 位整数中。

### dtime_tz_t 结构

```cpp
struct dtime_tz_t {
    // 位布局常量
    static constexpr const int TIME_BITS = 40;    // 时间使用 40 位
    static constexpr const int OFFSET_BITS = 24;  // 偏移使用 24 位
    static constexpr const uint64_t OFFSET_MASK = ~uint64_t(0) >> TIME_BITS;

    // 偏移范围：±15:59:59
    static constexpr const int32_t MAX_OFFSET = 16 * 60 * 60 - 1;  // 57599 秒
    static constexpr const int32_t MIN_OFFSET = -MAX_OFFSET;
    static constexpr const uint64_t OFFSET_MICROS = 1000000;

    uint64_t bits;  // 组合存储

    // 构造函数
    inline dtime_tz_t(dtime_t t, int32_t offset)
        : bits(encode_micros(t.micros) | encode_offset(offset)) {}
    explicit inline dtime_tz_t(uint64_t bits_p) : bits(bits_p) {}

    // 访问器
    inline dtime_t time() const { return dtime_t(decode_micros(bits)); }
    inline int32_t offset() const { return decode_offset(bits); }
};
```

### 位编码方案

```
┌────────────────────────────────────────────────────────────────┐
│                          64 bits                               │
├────────────────────────────────────────┬───────────────────────┤
│            Time (40 bits)              │  Offset (24 bits)     │
│         微秒数 (0 ~ 86399999999)        │  偏移秒数 (编码后)     │
└────────────────────────────────────────┴───────────────────────┘
```

### 偏移编码

偏移量使用反向编码，使得排序时能正确比较：

```cpp
// 偏移编码：反转顺序使得 13:00:00+01 < 12:00:00+00 < 11:00:00-01
static inline uint64_t encode_offset(int32_t offset) {
    return uint64_t(MAX_OFFSET - offset);
}
static inline int32_t decode_offset(uint64_t bits) {
    return MAX_OFFSET - int32_t(bits & OFFSET_MASK);
}

// 时间编码
static inline uint64_t encode_micros(int64_t micros) {
    return uint64_t(micros) << OFFSET_BITS;
}
static inline int64_t decode_micros(uint64_t bits) {
    return int64_t(bits >> OFFSET_BITS);
}
```

### 排序键

比较带时区时间时，需要调整到同一时区（UTC）后再比较：

```cpp
inline uint64_t sort_key() const {
    // 加上偏移转换到 UTC
    return bits + encode_micros((bits & OFFSET_MASK) * OFFSET_MICROS);
}

// 比较运算符使用排序键
inline bool operator<(const dtime_tz_t &rhs) const {
    return sort_key() < rhs.sort_key();
}
```

## TIMESTAMP 类型

TIMESTAMP 类型表示精确的时间点，存储从 Unix Epoch 开始的微秒数。

### timestamp_t 结构

```cpp
// timestamp.hpp
struct timestamp_t {
    int64_t value;  // 从 1970-01-01 00:00:00 UTC 开始的微秒数

    timestamp_t() = default;
    explicit inline constexpr timestamp_t(int64_t micros) : value(micros) {}

    // 显式转换
    explicit inline operator int64_t() const { return value; }

    // 比较运算符
    inline bool operator==(const timestamp_t &rhs) const { return value == rhs.value; }
    inline bool operator<(const timestamp_t &rhs) const { return value < rhs.value; }
    // ...

    // 算术运算
    timestamp_t operator+(const double &value) const;
    int64_t operator-(const timestamp_t &other) const;
    timestamp_t &operator+=(const int64_t &delta);
    timestamp_t &operator-=(const int64_t &delta);

    // 特殊值
    static constexpr timestamp_t infinity() {
        return timestamp_t(NumericLimits<int64_t>::Maximum());
    }
    static constexpr timestamp_t ninfinity() {
        return timestamp_t(-NumericLimits<int64_t>::Maximum());
    }
    static constexpr timestamp_t epoch() {
        return timestamp_t(0);
    }
};
```

### 时间戳精度变体

DuckDB 支持多种精度的时间戳：

```cpp
// 秒精度
struct timestamp_sec_t : public timestamp_t {
    timestamp_sec_t() = default;
    explicit inline constexpr timestamp_sec_t(int64_t seconds)
        : timestamp_t(seconds) {}
};

// 毫秒精度
struct timestamp_ms_t : public timestamp_t {
    timestamp_ms_t() = default;
    explicit inline constexpr timestamp_ms_t(int64_t millis)
        : timestamp_t(millis) {}
};

// 纳秒精度
struct timestamp_ns_t : public timestamp_t {
    timestamp_ns_t() = default;
    explicit inline constexpr timestamp_ns_t(int64_t nanos)
        : timestamp_t(nanos) {}
};

// 带时区（物理上与 timestamp_t 相同，但语义上是 UTC）
struct timestamp_tz_t : public timestamp_t {
    timestamp_tz_t() = default;
    explicit inline constexpr timestamp_tz_t(int64_t micros)
        : timestamp_t(micros) {}
    explicit inline constexpr timestamp_tz_t(timestamp_t ts)
        : timestamp_t(ts) {}
};
```

### 时间戳范围

```cpp
class Timestamp {
public:
    // 最小时间戳：公元前 290308 年 12 月 22 日
    constexpr static const int32_t MIN_YEAR = -290308;
    constexpr static const int32_t MIN_MONTH = 12;
    constexpr static const int32_t MIN_DAY = 22;
    // 最大时间戳由 INT64_MAX 微秒决定
};
```

### Timestamp 辅助类

```cpp
class Timestamp {
public:
    // 字符串转换
    static timestamp_t FromString(const string &str, bool use_offset);
    static timestamp_t FromCString(const char *str, idx_t len, bool use_offset = false,
                                   optional_ptr<int32_t> nanos = nullptr);
    static string ToString(timestamp_t timestamp);

    // 日期时间分解与组合
    static date_t GetDate(timestamp_t timestamp);
    static dtime_t GetTime(timestamp_t timestamp);
    static timestamp_t FromDatetime(date_t date, dtime_t time);
    static void Convert(timestamp_t date, date_t &out_date, dtime_t &out_time);

    // Epoch 转换
    static timestamp_t FromEpochSeconds(int64_t s);
    static timestamp_t FromEpochMs(int64_t ms);
    static timestamp_t FromEpochMicroSeconds(int64_t micros);
    static timestamp_t FromEpochNanoSeconds(int64_t nanos);

    static int64_t GetEpochSeconds(timestamp_t timestamp);
    static int64_t GetEpochMs(timestamp_t timestamp);
    static int64_t GetEpochMicroSeconds(timestamp_t timestamp);
    static int64_t GetEpochNanoSeconds(timestamp_t timestamp);

    // 组件分解
    static TimestampComponents GetComponents(timestamp_t timestamp);

    // 有限性检查
    static inline bool IsFinite(timestamp_t timestamp) {
        return timestamp != timestamp_t::infinity() &&
               timestamp != timestamp_t::ninfinity();
    }

    // 当前时间
    static timestamp_t GetCurrentTimestamp();

    // Julian Day 转换
    static double GetJulianDay(timestamp_t timestamp);
};
```

### 时间戳组件

```cpp
struct TimestampComponents {
    int32_t year;
    int32_t month;
    int32_t day;
    int32_t hour;
    int32_t minute;
    int32_t second;
    int32_t microsecond;
};
```

### 纳秒时间戳的特殊处理

纳秒时间戳需要额外的纳秒字段：

```cpp
static dtime_ns_t GetTimeNs(timestamp_ns_t timestamp);

static void Convert(timestamp_ns_t date, date_t &out_date,
                    dtime_t &out_time, int32_t &out_nanos);

static bool TryFromTimestampNanos(timestamp_t ts, int32_t nanos,
                                  timestamp_ns_t &result);
```

## 类型映射

日期时间类型到物理类型的映射：

```cpp
PhysicalType LogicalType::GetInternalType() {
    switch (id_) {
    // DATE 使用 INT32（天数）
    case LogicalTypeId::DATE:
        return PhysicalType::INT32;

    // 所有时间和时间戳使用 INT64
    case LogicalTypeId::TIME:
    case LogicalTypeId::TIME_TZ:
    case LogicalTypeId::TIME_NS:
    case LogicalTypeId::TIMESTAMP:
    case LogicalTypeId::TIMESTAMP_TZ:
    case LogicalTypeId::TIMESTAMP_SEC:
    case LogicalTypeId::TIMESTAMP_MS:
    case LogicalTypeId::TIMESTAMP_NS:
        return PhysicalType::INT64;

    // ...
    }
}
```

## 字符串格式

### DATE 格式

```
YYYY-MM-DD

示例：
2024-01-15
-0001-12-31  （公元前 2 年）
5881580-07-10（最大日期）
```

### TIME 格式

```
HH:MM:SS[.ffffff]

示例：
14:30:00
14:30:00.123456
```

### TIMESTAMP 格式

```
YYYY-MM-DD HH:MM:SS[.ffffff][±HH:MM]

示例：
2024-01-15 14:30:00
2024-01-15 14:30:00.123456
2024-01-15 14:30:00+08:00
```

### TIMETZ 格式

```
HH:MM:SS[.ffffff]±HH:MM

示例：
14:30:00+08:00
14:30:00.123456-05:30
```

## 时区处理

### TIMESTAMPTZ 语义

`TIMESTAMPTZ` 存储的是 UTC 时间点，在输入输出时会根据当前时区进行转换：

```cpp
// 解析时处理时区
static TimestampCastResult TryConvertTimestampTZ(
    const char *str, idx_t len, timestamp_t &result,
    const bool use_offset, bool &has_offset, string_t &tz,
    optional_ptr<int32_t> nanos = nullptr);
```

### 时区字符辨识

```cpp
static inline bool CharacterIsTimeZone(char c) {
    return StringUtil::CharacterIsAlpha(c) ||
           StringUtil::CharacterIsDigit(c) ||
           c == '_' || c == '/' ||
           c == '+' || c == '-';
}
```

### UTC 偏移解析

```cpp
static bool TryParseUTCOffset(const char *str, idx_t &pos, idx_t len,
                              int &hh, int &mm, int &ss);
```

## 日期时间运算

### DATE + INTERVAL

```cpp
// interval.hpp
class Interval {
public:
    static date_t Add(date_t left, interval_t right);
    static timestamp_t Add(timestamp_t left, interval_t right);
    static dtime_t Add(dtime_t left, interval_t right, date_t &date);
    static dtime_tz_t Add(dtime_tz_t left, interval_t right, date_t &date);
};
```

### 日期差值

两个日期相减返回天数差：

```cpp
template <>
int64_t SubtractOperator::Operation(date_t left, date_t right);
```

### 时间戳差值

两个时间戳相减返回微秒差：

```cpp
int64_t operator-(const timestamp_t &other) const {
    return value - other.value;
}
```

## 特殊值处理

### 无穷大

日期和时间戳支持正负无穷大：

```cpp
// 无穷大检查
static inline bool IsFinite(date_t date) {
    return date != date_t::infinity() && date != date_t::ninfinity();
}

static inline bool IsFinite(timestamp_t timestamp) {
    return timestamp != timestamp_t::infinity() &&
           timestamp != timestamp_t::ninfinity();
}
```

### 解析结果枚举

```cpp
enum class DateCastResult : uint8_t {
    SUCCESS,
    ERROR_INCORRECT_FORMAT,
    ERROR_RANGE
};

enum class TimestampCastResult : uint8_t {
    SUCCESS,
    ERROR_INCORRECT_FORMAT,
    ERROR_NON_UTC_TIMEZONE,
    ERROR_RANGE,
    STRICT_UTC
};
```

## 性能优化

### 预计算查找表

DuckDB 使用预计算的查找表加速日期转换：

```cpp
// 400 年周期（公历周期）
// 400 年 = 146097 天（包含 97 个闰年）
constexpr static const int32_t YEAR_INTERVAL = 400;
constexpr static const int32_t DAYS_PER_YEAR_INTERVAL = 146097;

// 通过查表快速确定某天属于哪个月
static const int8_t MONTH_PER_DAY_OF_YEAR[365];
static const int8_t LEAP_MONTH_PER_DAY_OF_YEAR[366];

// 累计天数表
static const int32_t CUMULATIVE_YEAR_DAYS[401];
```

### 直接整数运算

所有日期时间值底层都是整数，比较和算术运算直接操作整数：

```cpp
// 快速比较
inline bool operator<(const date_t &rhs) const {
    return days < rhs.days;
}

// 快速算术
inline date_t operator+(const int32_t &days) const {
    return date_t(this->days + days);
}
```

## 小结

本章详细介绍了 DuckDB 的日期时间类型系统：

1. **DATE 类型**：32 位整数存储天数，支持广泛的日期范围
2. **TIME 类型**：64 位整数存储微秒，支持高精度时间
3. **TIMETZ 类型**：64 位组合编码时间和时区偏移
4. **TIMESTAMP 类型**：64 位微秒时间戳，多种精度变体
5. **TIMESTAMPTZ 类型**：UTC 时间点，输入输出时区转换

关键设计决策：

- **整数存储**：所有类型底层使用整数，便于比较和运算
- **微秒精度**：默认提供微秒级精度，可选纳秒级
- **预计算表**：使用查找表加速日期转换
- **特殊值支持**：支持正负无穷大表示边界条件
- **时区编码**：TIMETZ 巧妙地将时间和偏移编码在一起

下一章将探讨嵌套类型系统，包括 LIST、STRUCT、MAP、UNION 和 ARRAY 类型的实现。

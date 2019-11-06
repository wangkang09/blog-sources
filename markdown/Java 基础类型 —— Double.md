[TOC]

## 1 Float 类方法

- 返回类型标 * 的是重点方法

### 1.1 toString —— 将 double 转为 String

| 返回类型        | 方法名                | 描述                                  |
| --------------- | --------------------- | ------------------------------------- |
| String          | toString()            | toString(value)                       |
| * static String | toString(double d)    | FloatingDecimal.toJavaFormatString(d) |
| * static String | toHexString(double d) | 比较复杂                              |

### 1.2 parseDouble —— 将 String 转为 double

| 返回类型        | 方法名                | 描述                           |
| --------------- | --------------------- | ------------------------------ |
| * static double | parseDouble(String s) | FloatingDecimal.parseDouble(s) |

### 1.3 valueOf —— 将 String、float 转为 Double

| 返回类型      | 方法名            | 描述                       |
| ------------- | ----------------- | -------------------------- |
| static Double | valueOf(String s) | new Double(parseDouble(s)) |
| static Double | valueOf(double s) | new Double(d)              |

### 1.4 hashcode/compare

| 返回类型   | 方法名                          | 描述                                       |
| ---------- | ------------------------------- | ------------------------------------------ |
| int        | hashCode()                      | Double.hashCode(value)                     |
| static int | hashCode(float value)           | doubleToLongBits(value) 等                 |
| int        | compareTo(Double anotherDouble) | Double.compare(value, anotherDouble.value) |
| static int | compare(double d1, double d2)   | 考虑了 -0.0 NaN 情况等                     |

### 1.5 sum/max/min

| 返回类型      | 方法名                  | 描述           |
| ------------- | ----------------------- | -------------- |
| static double | sum(double a, double b) | a + b          |
| static double | max(double a, double b) | Math.max(a, b) |
| static double | min(double a, double b) | Math.min(a, b) |

### 1.6 doubleToLongBits/doubleToRawLongBits/longBitsToDouble

- [参考于此文章](https://juejin.im/post/5c7fe55a6fb9a049f23d8814)
- [doubletorawlongbits定义及介绍](https://www.169it.com/article/14257277675540547002.html)

| 返回类型               | 方法名                            | 描述                                                         |
| ---------------------- | --------------------------------- | ------------------------------------------------------------ |
| * static long          | doubleToLongBits(double value)    | 基本等同于 `floatToRawIntBits()` 方法，区别在于这里对于 `NaN` 作了检测，如果结果为 `NaN`, 直接返回 `0x7fc00000` |
| * static native long   | doubleToRawLongBits(double value) | 将 `float` 浮点数转换为其 `IEEE 754` 标准二进制形式对应的 `int` 值 |
| * static native double | longBitsToDouble(long bits)       | `native` 方法，也是通过联合体 `union` 来实现的。只要参数中的 `int` 值满足 `IEEE 754` 对于 `NaN` 的标准，就可以产生值不为 `Float.NaN` 的 `NaN` 值了 |

### 1.7 isNaN/isFinite

| 返回类型       | 方法名               | 描述                                                   |
| -------------- | -------------------- | ------------------------------------------------------ |
| static boolean | isNaN(double v)      | v != v                                                 |
| static boolean | isInfinite(double v) | (v == POSITIVE_INFINITY) \|\| (v == NEGATIVE_INFINITY) |
| static boolean | isFinite(double d)   | Math.abs(d) <= DoubleConsts.MAX_VALUE                  |
| boolean        | isNaN()              | isNaN(value)                                           |
| boolean        | isInfinite()         | isInfinite(value)                                      |



### 1.8 构造函数 Double(String)

- 内部调用  parseDouble(s)

## 参考

jdk1.8_171
[TOC]

## 1 Float 类方法

- 返回类型标 * 的是重点方法

### 1.1 toString —— 将 float 转为 String

| 返回类型        | 方法名               | 描述                                  |
| --------------- | -------------------- | ------------------------------------- |
| String          | toString()           | Float.toString(value)                 |
| * static String | toString(float f)    | FloatingDecimal.toJavaFormatString(f) |
| * static String | toHexString(float f) | Double.toHexString(f)                 |

### 1.2 parseFloat —— 将 String 转为 float

| 返回类型       | 方法名               | 描述                          |
| -------------- | -------------------- | ----------------------------- |
| * static float | parseFloat(String s) | FloatingDecimal.parseFloat(s) |

### 1.3 valueOf —— 将 String、float 转为 Float

| 返回类型     | 方法名            | 描述                     |
| ------------ | ----------------- | ------------------------ |
| static Float | valueOf(String s) | new Float(parseFloat(s)) |
| static Float | valueOf(float s)  | new Float(f)             |

### 1.4 hashcode/compare

| 返回类型   | 方法名                        | 描述                                     |
| ---------- | ----------------------------- | ---------------------------------------- |
| int        | hashCode()                    | Float.hashCode(value)                    |
| static int | hashCode(float value)         | floatToIntBits(value)                    |
| int        | compareTo(Float anotherFloat) | Float.compare(value, anotherFloat.value) |
| static int | compare(float f1, float f2)   | 考虑了 -0.0 NaN 情况                     |

### 1.5 sum/max/min

| 返回类型     | 方法名                | 描述           |
| ------------ | --------------------- | -------------- |
| static float | sum(float a, float b) | a + b          |
| static float | max(float a, float b) | Math.max(a, b) |
| static float | min(float a, float b) | Math.min(a, b) |

### 1.6 floatToIntBits/floatToRawIntBits/intBitsToFloat

- [参考于此文章](https://juejin.im/post/5c7fe55a6fb9a049f23d8814)

| 返回类型              | 方法名                         | 描述                                                         |
| --------------------- | ------------------------------ | ------------------------------------------------------------ |
| * static int          | floatToIntBits(float value)    | 基本等同于 `floatToRawIntBits()` 方法，区别在于这里对于 `NaN` 作了检测，如果结果为 `NaN`, 直接返回 `0x7fc00000` |
| * static native int   | floatToRawIntBits(float value) | 将 `float` 浮点数转换为其 `IEEE 754` 标准二进制形式对应的 `int` 值 |
| * static native float | intBitsToFloat(int bits)       | `native` 方法，也是通过联合体 `union` 来实现的。只要参数中的 `int` 值满足 `IEEE 754` 对于 `NaN` 的标准，就可以产生值不为 `Float.NaN` 的 `NaN` 值了 |

### 1.7 isNaN/isFinite

| 返回类型       | 方法名              | 描述                                                   |
| -------------- | ------------------- | ------------------------------------------------------ |
| static boolean | isNaN(float v)      | v != v                                                 |
| static boolean | isInfinite(float v) | (v == POSITIVE_INFINITY) \|\| (v == NEGATIVE_INFINITY) |
| static boolean | isFinite(float f)   | Math.abs(f) <= FloatConsts.MAX_VALUE                   |
| boolean        | isNaN()             | isNaN(value)                                           |
| boolean        | isInfinite()        | isInfinite(value)                                      |



### 1.8 构造函数 Float(String)

- 内部调用  parseFloat(s)

## 参考

jdk1.8_171
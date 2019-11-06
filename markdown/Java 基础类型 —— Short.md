[TOC]

## 1 Short 类方法

- 返回类型标 * 的是重点方法

### 1.1 toString —— 将 Short 转为 String

| 返回类型      | 方法名            | 描述                         |
| ------------- | ----------------- | ---------------------------- |
| static String | toString(short s) | Integer.toString((int)s, 10) |
| String        | toString()        | Integer.toString((int)value) |

### 1.2 parseShort —— 将 String 转为 short

| 返回类型     | 方法名                          | 描述                                                         |
| ------------ | ------------------------------- | ------------------------------------------------------------ |
| static short | parseShort(String s)            | parseShort(s, 10)                                            |
| static short | parseShort(String s, int radix) | Integer.parseInt(s, radix)，如果返回的 int 值 不在 short 的范围内，则报错 |
### 1.3 valueOf —— 将 String、short 转为 Short

| 返回类型     | 方法名                       | 描述                          |
| ------------ | ---------------------------- | ----------------------------- |
| static Short | valueOf(String s)            | valueOf(s, 10)                |
| static Short | valueOf(String s, int radix) | valueOf(parseShort(s, radix)) |
| static Short | valueOf(short s)             | 在 [-128,127] 之内走 cache    |
### 1.4 hashcode/compare/reverseBytes 

| 返回类型     | 方法名                        | 描述                                      |
| ------------ | ----------------------------- | ----------------------------------------- |
| int          | hashCode()                    | Short.hashCode(value)                     |
| static int   | hashCode(short value)         | (int)value                                |
| int          | compareTo(Short anotherShort) | compare(this.value, anotherShort.value)   |
| static int   | compare(short x, short y)     | x - y，这个返回的不是固定的 -1,0,1        |
| static short | reverseBytes(short i)         | (short) (((i & 0xFF00) >> 8) \| (i << 8)) |
### 1.5 toUnsignedInt/toUnsignedLong

| 返回类型    | 方法名                  | 描述                 |
| ----------- | ----------------------- | -------------------- |
| static int  | toUnsignedInt(short x)  | ((int) x) & 0xffff   |
| static long | toUnsignedLong(short x) | ((long) x) & 0xffffL |

### 1.6 构造函数 Short(String)

- 内部调用  parseShort(s,10)
- 如果确定 int 很小，推荐使用 Short.valueOf(String)：这样可以走 cache



## 参考

jdk1.8_171
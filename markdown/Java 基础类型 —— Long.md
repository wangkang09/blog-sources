[TOC]

## 1 Long 类方法

- 返回类型标 * 的是重点方法

### 1.1 toXXXString

| 返回类型            | 方法名                                 | 描述                                                         |
| ------------------- | -------------------------------------- | ------------------------------------------------------------ |
| String              | toString()                             | toString(value)                                              |
| * static String     | toString(long i)                       | 有意思，这和 i+"" 区别在哪呢？内部调用 getChar()             |
| * static String     | toString(long i, int radix)            | 将 long 值转成 对应进制的 String 形式                        |
| static String       | toUnsignedString(long i)               | toUnsignedString(long i,10)                                  |
| static String       | toUnsignedString(long i, int radix)    | 内部调用 toUnsignedString0(long val, int shift)              |
| static String       | toHexString(long i)                    | toUnsignedString0(i, 4)                                      |
| static String       | toOctalString(long i)                  | toUnsignedString0(i, 3)                                      |
| static String       | toBinaryString(long i)                 | toUnsignedString0(i, 1)                                      |
| static String       | toUnsignedString0(long val, int shift) | 内部是全是通过 val & mask，val >>>= shift，所以仅仅是通过 mask 取到每一部分的值对应的 radix，所以是无符号的 |
| * static String     | formatUnsignedLong(...)                | toUnsignedString0 内部调用它，进行的掩模计算                 |
| * static BigInteger | toUnsignedBigInteger(long i)           | 内部调用 BigInteger.valueOf(i)                               |
### 1.2 parseLong(...) —— 将各种格式的 String 转成 long

| 返回类型      | 方法名                                 | 描述                                                        |
| ------------- | -------------------------------------- | ----------------------------------------------------------- |
| static long   | parseLong(String s)                    | parseLong(String s, 10)                                     |
| * static long | parseLong(String s, int radix)         | 根据对应的进制，将 s 解析成对应的 long 值，用负数算(有意思) |
| static long   | parseUnsignedLong(String s)            | parseUnsignedLong(String s, 10)                             |
| * static long | parseUnsignedLong(String s, int radix) | 内部调用 parseLong(String s, int radix) 做处理              |
### 1.3 valueOf —— 将 String、long 转成 Long

| 返回类型    | 方法名                       | 描述                                        |
| ----------- | ---------------------------- | ------------------------------------------- |
| static Long | valueOf(String s)            | Integer.valueOf(parseLong(s, 10))           |
| static Long | valueOf(String s, int radix) | Integer.valueOf(parseLong(s,radix))         |
| static Long | valueOf(long l)              | 内部走Cache,LongCache 的范围固定 [-128,127] |
### 1.4 getLong—— 获取系统参数值，并转换为 Long

| 返回类型    | 方法名                       | 描述                                                         |
| ----------- | ---------------------------- | ------------------------------------------------------------ |
| static Long | getLong(String nm)           | getLong(nm, null)                                            |
| static Long | getLong(String nm, long val) | 内部调用 getLong(nm, null)                                   |
| static Long | getLong(String nm, Long val) | 获取系统 property 参数值，v = System.getProperty(nm)，如果为 null，返回 val 值 |
### 1.5 compare —— long 值比较

| 返回类型   | 方法名                          | 描述                                      |
| ---------- | ------------------------------- | ----------------------------------------- |
| int        | compareTo(Long anotherLong)     | compare(this.value, anotherInteger.value) |
| static int | compare(long x, long y)         | (x < y) ? -1 : ((x == y) ? 0 : 1)         |
| static int | compareUnsigned(long x, long y) | compare(x + MIN_VALUE, y + MIN_VALUE)     |
### 1.6 unsigned long 的除/取余

| 返回类型    | 方法名                                         | 描述                                                         |
| ----------- | ---------------------------------------------- | ------------------------------------------------------------ |
| static long | divideUnsigned(long dividend, long divisor)    | toUnsignedBigInteger(dividend).     divide(toUnsignedBigInteger(divisor)).longValue() |
| static long | remainderUnsigned(long dividend, long divisor) | toUnsignedBigInteger(dividend).     remainder(toUnsignedBigInteger(divisor)).longValue() |
### 1.7 取 long 值最高/低位 1 对应的 值

| 返回类型      | 方法名                | 描述                                                         |
| ------------- | --------------------- | ------------------------------------------------------------ |
| * static long | highestOneBit(long i) | 最高位取最高 1 及后面全置0的值，[代码解析](https://blog.csdn.net/jessenpan/article/details/9617749) |
| * static long | lowestOneBit(long i)  | i & -i                                                       |
### 1.8 LeadingZeros/TrailingZeros/countBit

| 返回类型     | 方法名                        | 描述                         |
| ------------ | ----------------------------- | ---------------------------- |
| * static int | numberOfLeadingZeros(long i)  | i 对应 2 进制的开头 0 的个数 |
| * static int | numberOfTrailingZeros(long i) | i 对应 2 进制的末尾 0 的个数 |
| * static int | bitCount(long i)              | i 对应 2 进制中 1 的个数     |
### 1.9 long 对应二进制的反转

| 返回类型      | 方法名                            | 描述                                                         |
| ------------- | --------------------------------- | ------------------------------------------------------------ |
| * static long | rotateLeft(long i, int distance)  | (i << distance) \| (i >>> -distance)                         |
| * static long | rotateRight(long i, int distance) | (i >>> distance) \| (i << -distance)                         |
| * static long | reverse(long i)                   | [二进制按位反转](https://www.jianshu.com/p/be272c8704d9) ：对应的2进制180°反转 |
| * static long | reverseBytes(long i)              | [二进制按byte反转](https://www.jianshu.com/p/be272c8704d9)   |

```java
00000000_00000000_01001101_00011111 -> 原始
11111000_10110010_00000000_00000000 -> 按位反转
00011111_01001101_00000000_00000000 -> 按字节反转
```

### 1.10 sinum/sum/max/min

| 返回类型    | 方法名              | 描述                              |
| ----------- | ------------------- | --------------------------------- |
| static long | signum(long i)      | 负数返回 -1，0 返回 0，正数返回 1 |
| static long | sum(long a, long b) | a + b                             |
| static long | max(long a, long b) | Math.max(a, b)                    |
| static long | min(long a, long b) | Math.min(a, b)                    |

### 1.11 decode/hashCode/stringSize/toUnsignedLong

| 返回类型      | 方法名               | 描述                                                |
| ------------- | -------------------- | --------------------------------------------------- |
| static int    | stringSize(long x)   | x 的位数，x 必须为非负                              |
| int           | hashCode()           | Long.hashCode(value)                                |
| static int    | hashCode(long value) | (int)(value ^ (value >>> 32))                       |
| * static Long | decode(String nm)    | 根据 nm 的类型，判断它的进制，然后转成 10 进制 Long |

### 1.12 构造函数 Long(String)

- 内部调用  parseLong(s,10)
- 如果确定 long 很小，推荐使用 Long.valueOf(String)：这样可以走 cache



## 2 重点方法解析

- 留待以后补充



## 参考

jdk1.8_171
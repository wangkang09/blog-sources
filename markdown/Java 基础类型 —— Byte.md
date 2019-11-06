[TOC]

## 1 Byte 类方法

- 返回类型标 * 的是重点方法

### 1.1 toString —— 将 Short 转为 String

| 返回类型      | 方法名           | 描述                         |
| ------------- | ---------------- | ---------------------------- |
| static String | toString(byte b) | Integer.toString((int)s, 10) |
| String        | toString()       | Integer.toString((int)value) |

### 1.2 parseByte —— 将 String 转为 byte

| 返回类型    | 方法名                         | 描述                                                         |
| ----------- | ------------------------------ | ------------------------------------------------------------ |
| static byte | parseByte(String s)            | parseByte(s, 10)                                             |
| static byte | parseByte(String s, int radix) | Integer.parseInt(s, radix)，如果返回的 int 值 不在 byte 的范围内，则报错 |
### 1.3 valueOf —— 将 String、short 转为 Byte 

| 返回类型    | 方法名                       | 描述                         |
| ----------- | ---------------------------- | ---------------------------- |
| static Byte | valueOf(String s)            | valueOf(s, 10)               |
| static Byte | valueOf(String s, int radix) | valueOf(parseByte(s, radix)) |
| static Byte | valueOf(byte b)              | **直接走 cache**             |
### 1.4 hashcode/compare

| 返回类型   | 方法名                      | 描述                                   |
| ---------- | --------------------------- | -------------------------------------- |
| int        | hashCode()                  | Byte.hashCode(value)                   |
| static int | hashCode(byte value)        | (int)value                             |
| int        | compareTo(Byte anotherByte) | compare(this.value, anotherByte.value) |
| static int | compare(byte x, byte y)     | x - y，这个返回的不是固定的 -1,0,1     |
### 1.5 toUnsignedInt/toUnsignedLong

| 返回类型    | 方法名                 | 描述               |
| ----------- | ---------------------- | ------------------ |
| static int  | toUnsignedInt(byte x)  | ((int) x) & 0xff   |
| static long | toUnsignedLong(byte x) | ((long) x) & 0xffL |

### 1.6 构造函数 Byte(String)

- 内部调用  parseByte(s,10)
- **推荐使用 Byte.valueOf(s)**，因为这样返回的肯定是 cache



## 参考

jdk1.8_171
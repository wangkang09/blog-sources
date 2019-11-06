[TOC]

## 1 Boolean 类方法

- 返回类型标 * 的是重点方法

### 1.1 toString —— 将 boolean 转为 String

| 返回类型      | 方法名              | 描述                     |
| ------------- | ------------------- | ------------------------ |
| static String | toString(boolean b) | b ? "true" : "false"     |
| String        | toString()          | value ? "true" : "false" |

### 1.2 parseBoolean—— 将 String 转为 boolean

| 返回类型       | 方法名                 | 描述                                        |
| -------------- | ---------------------- | ------------------------------------------- |
| static boolean | parseBoolean(String s) | ((s != null) && s.equalsIgnoreCase("true")) |
### 1.3 valueOf —— 将 String、boolean 转为 Boolean

| 返回类型       | 方法名             | 描述                           |
| -------------- | ------------------ | ------------------------------ |
| static Boolean | valueOf(String s)  | parseBoolean(s) ? TRUE : FALSE |
| static Boolean | valueOf(boolean b) | b ? TRUE : FALSE               |
### 1.4 hashcode/compare

| 返回类型   | 方法名                    | 描述                         |
| ---------- | ------------------------- | ---------------------------- |
| int        | hashCode()                | Boolean.hashCode(value)      |
| static int | hashCode(boolean value)   | value ? 1231 : 1237          |
| int        | compareTo(Boolean b)      | compare(this.value, b.value) |
| static int | compare(short x, short y) | (x == y) ? 0 : (x ? 1 : -1)  |
### 1.5 logicalAnd/logicalOr/logicalXor

| 返回类型       | 方法名                           | 描述     |
| -------------- | -------------------------------- | -------- |
| static boolean | logicalAnd(boolean a, boolean b) | a && b   |
| static boolean | logicalOr(boolean a, boolean b)  | a \|\| b |
| static boolean | logicalXor(boolean a, boolean b) | a ^ b    |

### 1.6 构造函数 Boolean(boolean)/Boolean(String)

- 内部调用  parseBoolean(s)
- **推荐使用 valueOf() 方法创建 Boolean 对象**



## 参考

jdk1.8_171
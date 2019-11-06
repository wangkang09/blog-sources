[TOC]

- java 中的数值类型都继承于 Number 抽象类
- 本文将介绍几种常见的 数值类，理清如何正确使用数值类方法

## 1 Number 抽象类

```java
public abstract class Number implements java.io.Serializable {

    //所有的数值类实现了这 4 个方法
    //所有的数值类可以直接调用这 4 个方法得到任意数值类值，不会抛错，但可能会丢失精度
    public abstract int intValue();
    public abstract long longValue();
    public abstract float floatValue();
    public abstract double doubleValue();

    public byte byteValue() {return (byte)intValue();}
    public short shortValue() {return (short)intValue();}
    private static final long serialVersionUID = -8742448824652078965L;
}
```

### 1.1 Integer 类解析

- [Java 基础类型 —— Integer](<https://blog.csdn.net/kangsa998/article/details/102931776>)

### 1.2 Long 类解析

- [Java 基础类型 —— Long](<https://blog.csdn.net/kangsa998/article/details/102931824>)

### 1.3 Short 类解析

- [Java 基础类型 —— Short](<https://blog.csdn.net/kangsa998/article/details/102936478>)

### 1.4 Byte 类解析

- [Java 基础类型 —— Byte](<https://blog.csdn.net/kangsa998/article/details/102936521>)

### 1.5 Double 类解析

- [Java 基础类型 —— Double](<https://blog.csdn.net/kangsa998/article/details/102936623>)

### 1.6 Float 类解析

- [ Java 基础类型 —— Float](<https://blog.csdn.net/kangsa998/article/details/102936588>)



## 2 Number 类型归纳

- 本小节总结所有 Number 类型的通用思想方法

### 2.1 parseXXX —— 将 String 转换为 各个基本类型

### 2.2 valueOf —— 将 基本类型、String 转换为 包装类型

### 2.3 decode —— 自动根据字符串的形式，转换为包装类型（Integer/Long）

### 2.4 xxxValue —— 转换成所有的基本类型数据

- 所有的 Number 子类，都必须实现抽象方法
- 并且所有的 子类，都是返回简单的类型转换值（所以，可能会丢失精度）

### 2.5 toString  toXXXString —— 转换为各种形式的 String

### 2.6 getXXX —— 确定具有指定名称的系统属性的各种类型的值（Integer/Long）

### 2.7 toUnsignedXXX —— 转换为 高类型的无符号类型

## 参考

jdk1.8_171
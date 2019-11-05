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

## 2 Integer

- 

## 3 Long 

- 
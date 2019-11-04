[TOC]

### 1 什么是 Class 对象

- 

```java
public final class Class<T> implements ...{
    // 得到类的类型，和类的加载解析有关
    public static Class<?> forName(String className){...};
    // 通过类类型获取它的实例
    public T newInstance() {..}
}
```
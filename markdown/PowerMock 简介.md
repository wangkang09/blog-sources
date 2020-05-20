[TOC]

- mock/spy：创建一个类的模拟实例

- stubbing：为以上创建出的模拟实例，对应的方法，打桩（即定制方法返回值）

---

### 1 简介

- 使用测试框架做单元测试一直以来都是一个非常好的实践。尤其是 mockito 测试框架，最近这些年更是占据了主导地位
- 为了避免代码坏味道，一些功能以及被移除了。然而，某些情况下，这时可能迫使测试者们编写繁琐的代码只是为了使mock的创建成为可能

这就是 powermock 框架的由来

- powermockito 是 powermock 的实现。它提供了以简单的方式使用Java反射API的能力，以克服Mockito的问题，例如缺少模拟final、静态或私有方法的能力
- 本教程将介绍PowerMockito API及其在测试中的应用



---

### 2 准备环境

#### 2.1 依赖

```xml
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>2.0.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito2</artifactId>
    <version>2.0.2</version>
    <scope>test</scope>
</dependency>
```

#### 2.2 使用 powermock 增强功能

```java
//开启mock注解等功能
@RunWith(PowerMockRunner.class)
//需要mock的类的路径，如果mock static/final/private 必须指定
@PrepareForTest(fullyQualifiedNames = "com.wangkang.test.*")
```



---

### 3 mock 模式

#### 3.1 mock 构造器和 final 方法

- 本小节，我们将展示不用 new 操作得到类的一个 mock 对象，然后使用这个 mock 对象，stubbing 它的一个 final 方法。此类如下：

```java
public class CollaboratorWithFinalMethods {
    public final String helloMethod() {
        return "Hello World!";
    }
}
```

- **1 创建 mock 对象**

```java
CollaboratorWithFinalMethods mock = mock(CollaboratorWithFinalMethods.class);
```

- **2 无参构造器打桩验证**

```java
whenNew(CollaboratorWithFinalMethods.class).withNoArguments().thenReturn(mock);
CollaboratorWithFinalMethods collaborator = new CollaboratorWithFinalMethods();
//验证是否调用了无参构造
verifyNew(CollaboratorWithFinalMethods.class).withNoArguments();
```

- **3 final 方法打桩验证**

```java
when(collaborator.helloMethod()).thenReturn("Hello Baeldung!");
//验证
String welcome = collaborator.helloMethod();
//验证调用了
Mockito.verify(collaborator).helloMethod();
//验证返回值
assertEquals("Hello Baeldung!", welcome);
```



#### 3.2 mock static 方法

- 我们需要 mock 的类如下

```java
public class CollaboratorWithStaticMethods {
    public static String firstMethod(String name) {
        return "Hello " + name + " !";
    }
 
    public static String secondMethod() {
        return "Hello no one!";
    }
 
    public static String thirdMethod() {
        return "Hello no one again!";
    }
}
```

- **1 注册封闭类**

```java
mockStatic(CollaboratorWithStaticMethods.class);
//可以：spy(CollaboratorForPartialMocking.class);
```

- **2 打桩**

```java
//stubbing
when(CollaboratorWithStaticMethods.firstMethod(Mockito.anyString())).thenReturn("Hello Baeldung!");       when(CollaboratorWithStaticMethods.secondMethod()).thenReturn("Nothing special");
//callRealMethod！！
assert CollaboratorWithStaticMethods.thirdMethod().equals("Hello no one again!");   when(CollaboratorWithStaticMethods.thirdMethod()).thenReturn("thirdFirstReturn", "thirdSecondReturn").thenThrow(new RuntimeException("dd"));
```

- **3 验证**

```java
String firstWelcome1 = CollaboratorWithStaticMethods.firstMethod("dds");
String firstWelcome2 = CollaboratorWithStaticMethods.secondMethod();
assertEquals("Hello Baeldung!", firstWelcome1);
assertEquals("Nothing special", firstWelcome2);

//验证串联式打桩声明
assert CollaboratorWithStaticMethods.thirdMethod().equals("thirdFirstReturn");
assert CollaboratorWithStaticMethods.thirdMethod().equals("thirdSecondReturn");
try {
    CollaboratorWithStaticMethods.thirdMethod();
    Assert.fail();
} catch (Exception e) {
    assert e.getMessage().equals("dd");
}
```



#### 3.3 mock private 方法

- 需要 mock 的类

```java
public class LuckyNumberGenerator {
    public int getLuckyNumber(String name) {
        saveIntoDatabase(name);
        if (name == null) {
            return getDefaultLuckyNumber();
        }
        return getComputedLuckyNumber(name.length());
    }
    private void saveIntoDatabase(String name) {}
    private int getDefaultLuckyNumber() {
        return 100;
    }
    private int getComputedLuckyNumber(int length) {
        return length < 5 ? 5 : 10000;
    }
}
```

- **1 创建 mock 对象**

```java
//因为 mock 的是方法内调用的 private 方法，所以默认为 callRealMethod，仅对 private 方法，额外打桩
mock = mock(LuckyNumberGenerator.class, InvocationOnMock::callRealMethod);
```

- **2 打桩并验证**

```java
when(mock, "getDefaultLuckyNumber").thenReturn(301);
assertEquals(301, mock.getLuckyNumber(null));
```



---

### 4 spy 模式

- 前几 章节，通过 mock() 方法，得到的是一个完全的 mock 对象，所有的方法调用默认都不会走真实方法
- 本章节，将通过 spy() 方法，得到的是一个部分 mock 对象，所有方法调用默认走的是真实方法，可以对其中一部分方法进行 stubbing 来模拟方法行为

```java
public class CollaboratorForPartialMocking {
    public static String staticMethod() {
        return "Hello Baeldung!";
    }
    public final String finalMethod() {
        return "Hello Baeldung!";
    }
    private String privateMethod() {
        return "Hello Baeldung!";
    }
    public String privateMethodCaller() {
        return privateMethod() + " Welcome to the Java world.";
    }
}
```



#### 4.1 spy static 方法 

- **1 spy 模拟对象**

```java
spy(CollaboratorForPartialMocking.class);
when(CollaboratorForPartialMocking.staticMethod()).thenReturn("I am a static mock method.");
```

- **2 验证**

```java
returnValue = CollaboratorForPartialMocking.staticMethod();
//验证值
assertEquals("I am a static mock method.", returnValue);
```



#### 4.2 spy final 和 private 方法

- **1 spy 模拟对象**

```java
CollaboratorForPartialMocking collaborator = new CollaboratorForPartialMocking();
CollaboratorForPartialMocking mock = spy(collaborator);
```

- **2 打桩并验证 final 方法**

```java
when(mock.finalMethod()).thenReturn("I am a final mock method.");
returnValue = mock.finalMethod();
assertEquals("I am a final mock method.", returnValue);
```

- **3 打桩并验证 private 方法**

```java
when(mock, "privateMethod").thenReturn("I am a private mock method.");
returnValue = mock.privateMethodCaller();
assertEquals("I am a private mock method. Welcome to the Java world.", returnValue);
```



### 参考

[github 源码地址](<https://github.com/wangkang09/unit-test/tree/master/power-mockito-test>)

[Introduction to PowerMock](<https://www.baeldung.com/intro-to-powermock>)




























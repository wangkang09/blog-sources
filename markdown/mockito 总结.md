## 1 mock/spy

### 1.1 mock

```java
List mockedList = mock(LinkedList.class);//可以mock具体类，建议
List interfaceList = mock(List.class);//也可以mock接口

//以下两个等效，并且是默认值
mock(LinkedList.class,Mockito.RETURNS_DEFAULTS);
mock(LinkedList.class,withSettings().defaultAnswer(Mockito.RETURNS_DEFAULTS));
//默认返回一个 SmartNull，可记录更详细的日志信息，mockito 4.0 后为默认
mock(LinkedList.class,Mockito.RETURNS_SMART_NULLS);
//如果返回 null，且不是 final，就返回一个 mock，不建议使用
mock(LinkedList.class,Mockito.RETURNS_MOCKS);
//可以级联调 when()，不建议使用
mock(LinkedList.class,Mockito.RETURNS_DEEP_STUBS);
//默认调用真实方法，这个方法等效于 spy
mock(LinkedList.class,Mockito.CALLS_REAL_METHODS);
//适用于 Builder 模式，一般用不上
mock(LinkedList.class,Mockito.RETURNS_SELF);
```

### 1.2 spy

```java
//等效于：mock(ListedList.class,Mockito.CALLS_REAL_METHODS);
spy(LinkedList.class);
List list = new ArrayList();
spy(list);
```

### 1.3 @Mock/@Spy/@InjectMocks/@Captor/@MockBean/@SpyBean

#### 1.3.1 如何使这些注解生效

- 在测试类上使用：@RunWith(MockitoJUnitRunner.class)
- 在 @Before 中调用：MockitoAnnotations.initMocks(this)
- 在类中定义：@Rule public MockitoRule mockito = MockitoJUnit.rule();

#### 1.3.2 作用

```java
@Mock/@Spy：相当于 mock/spy 的快捷方式
@InjectMocks：注解的对象相对于 spy 对象，并且可以将 mock/spy 对应的对象自动注入到此对象对应的属性中
@Captor：可以获取 Matcher 实际执行时对应的参数
@MockBean/@SpyBean：相对于 @Mock/@Spy，并且此注释的对象，被加入到 spring 容器中
```

#### 1.3.3 @InjectMocks 使用

- https://www.jianshu.com/p/bb705a56f620



## 2 when/then

- 有两种形式
  - doXXX().when()，推荐
  - when().doXXX()，不推荐（当调用真实方法时，会先直接when中的方法，这时的真实方法可能有问题，例如没有实现而报错）
  - any()：可能造成 NPX

```java
//调用 list.get() 三次以后都返回 third
doReturn("first","second","third").when(list).get(anyInt());
//当使用 matcher 时，所有参数都必须是 matcher 形式
//set(1,anyInt()) 将报错，可写成 set(eq(1),anyInt())
doThrow(new RuntimeException(),new IllegalAccessError()).when(list).set(anyInt(),anyInt())
//调用真实方法
doCallRealMethod().when(list).get(0);    
doNothing().when(list).add(anyInt());
//doAnswer
doAnswer(invocation -> 11).when(list).get(1);
assert 11 == (int)list.get(1);
assert list.get(2) == null;
```



## 3 verify

### 3.1 验证 mock/spy 对象已经执行了什么行为

```java
 //验证 list 执行了 list.add("one")
 verify(list).add("one");
 //验证 list 执行了 list.clear()
 verify(list).clear();
```

### 3.2 验证行为执行的次数

```java
//以下两个等效：验证 list.add("once") 执行了 1次
verify(list).add("once");
verify(list, times(1)).add("once");
//验证执行了 2 次
verify(list, times(2)).add("once");

verify(list, never()).add("never happened");
verify(list, atMostOnce()).add("once");
verify(list, atLeastOnce()).add("three times");
verify(list, atLeast(2)).add("three times");
verify(list, atMost(5)).add("three times");

//验证 listTwo,listThree 没有被调用过
verifyZeroInteractions(listTwo, listThree);

//验证已经验证了所有调用的方法了
verifyNoMoreInteractions(list);
```

### 3.3 验证调用顺序

```java
//create an inOrder verifier for a single mock
InOrder inOrder = inOrder(list);
inOrder.verify(list).add("was added first");
inOrder.verify(list).add("was added second");

InOrder inOrder = inOrder(firstMock, secondMock);
//两个 mock 实例之间的方法的调用顺序
inOrder.verify(firstMock).add("was called first");
inOrder.verify(secondMock).add("was called second");
```

### 3.4 延迟验证

- 可用于异步情况，做延迟验证

```java
//100ms 之后验证方法调用1次
verify(mock, timeout(100)).someMethod();
//100ms 之后验证方法调用2次
verify(mock, timeout(100).times(2)).someMethod();
```

### 3.5 忽略校验：ignoreStubs

#### 3.5.1 忽略 verify

- 忽略所有

```java
//mocking lists for the sake of the example (if you mock List in real you will burn in hell)
List mock1 = mock(List.class), mock2 = mock(List.class);
//stubbing mocks:
when(mock1.get(0)).thenReturn(10);
when(mock2.get(0)).thenReturn(20);
//using mocks by calling stubbed get(0) methods:
System.out.println(mock1.get(0)); //prints 10
System.out.println(mock2.get(0)); //prints 20
//using mocks by calling clear() methods:
mock1.clear();
mock2.clear();
//verification:
verify(mock1).clear();
verify(mock2).clear();

//verifyNoMoreInteractions() fails because get() methods were not accounted for.
try { verifyNoMoreInteractions(mock1, mock2); } catch (NoInteractionsWanted e);

//因为忽略了 mock1 mock2，即使 get 方法没有 verify 也通过
verifyNoMoreInteractions(ignoreStubs(mock1, mock2));
verifyZeroInteractions() 等效于 verifyNoMoreInteractions
```

#### 3.5.2 忽略 InOrder

- 忽略所有

```java
  List list = mock(List.class);
  when(list.get(0)).thenReturn("foo");
  list.add(0);
  list.clear();
  System.out.println(list.get(0)); //we don't want to verify this

  InOrder inOrder = inOrder(ignoreStubs(list));
  inOrder.verify(list).add(0);
  inOrder.verify(list).clear();
  inOrder.verifyNoMoreInteractions();
```

#### 3.5.3 Strictness.STRICT_STUBS

- 通过 when() 打桩的方法，只要调用了就会被自动校验
- 没有打桩的方法，必须手动校验
- 打桩方法，没有被调用，相当于没有校验

```java
@Rule public MockitoRule mockito = MockitoJUnit.rule().strictness(Strictness.STRICT_STUBS);
List list = mock(List.class);
when(list.get(0)).thenReturn("foo");
list.size();
verify(list).size();
list.get(0); // Automatically verified by STRICT_STUBS
verifyNoMoreInteractions(list); // No need of ignoreStubs()
```



## 4 Argument matchers

- 通过 matchers 模糊指定打桩方法的参数
- 参数匹配器允许灵活的 verify 或 stubbing
- 参数匹配器相关文档：[MockitoHamcrest](https://javadoc.io/static/org.mockito/mockito-core/3.3.3/org/mockito/hamcrest/MockitoHamcrest.html)、[ArgumentMatchers](https://javadoc.io/static/org.mockito/mockito-core/3.3.3/org/mockito/ArgumentMatchers.html)
- 只要方法中的一个参数使用 matchers 其余方法必须也都使用 matchers

```java
//stubbing using built-in anyInt() argument matcher
when(mockedList.get(anyInt())).thenReturn("element");

//stubbing using custom matcher (let's say isValid() returns your own matcher implementation):
//isValid 效果同下面的 lambda 表达式
when(mockedList.contains(argThat(isValid()))).thenReturn(true);

//following prints "element"
System.out.println(mockedList.get(999));

//you can also verify using an argument matcher
verify(mockedList).get(anyInt());

//argument matchers can also be written as Java 8 Lambdas
verify(mockedList).add(argThat(someString -> someString.length() > 5));

//以下是正确的
verify(mock).someMethod(anyInt(), anyString(), eq("third argument"));
//以下是错误的
verify(mock).someMethod(anyInt(), anyString(), "third argument");
```

### 4.1 自定义 matcher

```java
class ListOfTwoElements implements ArgumentMatcher<List> {
    public boolean matches(List list) {
        return list.size() == 2;
    }
    public String toString() {
        //printed in verification errors
        return "[list of 2 elements]";
    }
}
List mock = mock(List.class);
when(mock.addAll(argThat(new ListOfTwoElements))).thenReturn(true);
//Arrays.asList("one","two") 满足 matches 所以能被使用
mock.addAll(Arrays.asList("one", "two"));
verify(mock).addAll(argThat(new ListOfTwoElements()));

//简写
verify(mock).addAll(argThat(new ListOfTwoElements()));
//becomes，将 argThat(new ListOfTwoElements()) 抽成方法
verify(mock).addAll(listOfTwoElements());

//lambda
verify(mock).addAll(argThat(list -> list.size() == 2));
```



## 5 ArgumentCaptor 获取执行方法时的参数

```java
List list = mock(ArrayList.class);
ArgumentCaptor<Integer> argument = ArgumentCaptor.forClass(Integer.class);
list.add(1);
int temp = ThreadLocalRandom.current().nextInt(1000);
list.add(temp);
//argument 只有 verify 之后才有值
verify(list, times(2)).add(argument.capture());
//getValue 是最后一次的参数值
assert argument.getValue() == temp;
//getAllValues() 包含所有调用的参数值
assert argument.getAllValues().contains(temp);
assert argument.getAllValues().contains(1);
```



## 6 自定义 mock 返回的默认值/doAnswer

### 6.1 自定义默认返回值

```java
List list = mock(ArrayList.class, new ArrayListAnswer());
assert "[1, 2]".equals(list.get(1).toString());
verify(list).get(anyInt());
//lambda 形式
List list1 = mock(ArrayList.class, invocation -> Arrays.asList(1, 3));
assert "[1,3]".equals(list1.get(1).toString());
verify(list).get(anyInt());

class ArrayListAnswer implements Answer<List> {
    @Override
    public List answer(InvocationOnMock invocation) throws Throwable {
        return Arrays.asList(1, 2);
    }
}
```

### 6.2 doAnswer

```java
List list = mock(ArrayList.class);
doAnswer(invocation -> 11).when(list).get(1);
assert 11 == (int)list.get(1);
assert list.get(2) == null;
```



## 7 MockingDetails 获取对象信息

```java
//To identify whether a particular object is a mock or a spy:
Mockito.mockingDetails(someObject).isMock();
Mockito.mockingDetails(someObject).isSpy();

//Getting details like type to mock or default answer:
MockingDetails details = mockingDetails(mock);
details.getMockCreationSettings().getTypeToMock();
details.getMockCreationSettings().getDefaultAnswer();

//Getting invocations and stubbings of the mock:
MockingDetails details = mockingDetails(mock);
details.getInvocations();
details.getStubbings();

//Printing all interactions (including stubbing, unused stubs)
System.out.println(mockingDetails(mock).printInvocations());
```



## 8 代理难以 mock 的对象

- Final classes but with an interface
- Already custom proxied object
- Special objects with a finalize method, i.e. to avoid executing it 2 times

```java
//本来是打算 mock 此对象的，但是是 final
DontYouDareToMockMe dare = new DontYouDareToMockMe();
List<Integer> mock = mock(List.class, delegatesTo(dare));
//mock DontYouDareToMockMe 的 get() 方法！！
doReturn(2).when(mock).get(anyInt());
//只要方法名和参数一样就可以
assert mock.get(1) == 2;
assert !systemOutRule.getLog().contains("----------->调用了 final 类的方法");
final class DontYouDareToMockMe {
    public Integer get(int idx) { return idx;}
}
```



## 参考

[mockito 官方文档](https://javadoc.io/static/org.mockito/mockito-core/3.3.3/org/mockito/Mockito.html#do_family_methods_stubs)


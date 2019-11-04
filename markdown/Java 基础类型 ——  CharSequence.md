```java
// 一个 charSequence 是一个可读的 char 序列
//此接口为不同的 char 序列，提供能了一个统一的、只读的方法
public interface CharSequence {
    //返回 char 序列中的， char 的个数（不是 unicode 字符的个数，一个 unicode 可能有 两个 char 构成，即 a pair of surrogate） 
    int length();
    //得到指定下标的 char 值（可能只是一个 surrogate,code unit）
    char charAt(int index);
    //返回子序列，[star,end)
    CharSequence subSequence(int start, int end);
    //返回此 charsequence 构成的 string
    public String toString();
    //return an IntStream of char values from this sequence(不考虑 surragate pairs)
    public default IntStream chars() {...}
    //return an IntStream of Unicode code points from this sequence(组合 surragate pairs，即返回的是真正的每个字符的 unicode 码)
    public default IntStream codePoints() {...}
    //用于判断基本类型，其它几种对应的基本类型也有这样的 TYPE
    public static final Class<Character> TYPE = (Class<Character>) Class.getPrimitiveClass("char");
   
}
```

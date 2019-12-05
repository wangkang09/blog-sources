此类定义了字节缓冲区上的六类操作：

- 绝对和相对的get和put方法(读写单个字节)
- 相对批量获取方法，将连续的字节序列从该缓冲区传输到数组中
- 从字节数组或其他字节缓冲区向该缓冲区传输连续字节序列的相对批量放入方法
- 绝对和相对get和put方法，用于读取和写入其他基本类型的值，并按特定字节顺序在字节序列之间来回转换这些值
- 用于创建视图缓冲区的方法，该方法允许将字节缓冲区视为包含某些其他基本类型的值的缓冲区
- 压缩、复制和切片字节缓冲区的方法

字节缓冲区可以通过分配来创建，该分配为缓冲区的内容分配空间，或者将现有的字节数组封装到缓冲区中。



#### Direct *vs.* non-direct buffers

字节缓冲区可以是直接(direct)的，也可以是非直接(heap)的。给定一个 direct buffer，Java虚拟机将尽最大努力直接在其上执行 native I/O操作。也就是说，它将尝试避免在每次调用底层操作系统的 native I/O操作之前（或之后）将缓冲区的内容复制到（或从）中间缓冲区。



可以通过调用此类的a llocateDirect工厂方法 来创建 direct buffer。此方法返回的缓冲区通常比非直接缓冲区具有**更高的分配和释放成本**。直接缓冲区的内容可能位于**正常垃圾收集堆之外**，因此它们对应用程序内存占用的影响可能不明显。因此，建议将直接缓冲区主要分配给受底层系统native I/O操作影响的大型、长寿命缓冲区。一般来说，只有当直接缓冲区在程序性能上产生**可测量的增益时**，才最好分配直接缓冲区。



direct buffer 也可以通过将文件的一个区域直接映射到内存中来创建(fileChannel 的 map 方法)。Java平台的实现可以选择性地支持通过JNI从本机代码创建直接字节缓冲区。如果其中一种缓冲区的实例引用了内存中**不可访问的区域**，则尝试访问该区域不会更改缓冲区的内容，并将导致在访问时或稍后某个时间引发未指定的异常。



字节缓冲区是直接的还是非直接的，可以通过调用其isDirect方法来确定。提供此方法是为了在性能关键型代码中执行显式缓冲区管理。



#### Access to binary data

这个类定义了读取和写入除布尔值以外的所有其他基本类型的值的方法。基本类型的值根据缓冲区的当前字节顺序转换为（或从）字节序列，可以通过order方法检索和修改这些字节序列。特定字节顺序由字节顺序类的实例表示。字节缓冲区的初始顺序总是大端的（BIG_ENDIAN）。



对于异构二进制数据（值序列的类型不同）的访问，该类为每种类型定义了一系列绝对和相对get、put方法。例如，对于32位浮点值，此类定义：

```
 float  getFloat()
 float  getFloat(int index)
  void  putFloat(float f)
  void  putFloat(int index, float f)
```



为char、short、int、long和double类型定义了相应的方法。**绝对**get和put方法的索引参数**以字节为单位**，而不是以读或写的类型为单位。



为了访问同构二进制数据（值序列的类型相同），该类定义了可以创建给定字节缓冲区**视图**的方法。视图缓冲区只是另一个缓冲区，其内容由字节缓冲区支持。对字节缓冲区内容的更改将在视图缓冲区中可见，反之亦然；**两个缓冲区的位置、限制和标记值是独立的**。例如，asFloatBuffer方法创建FloatBuffer类的实例，该实例由调用该方法的字节缓冲区支持。为char、short、int、long和double类型定义了相应的视图创建方法。



与上面描述的特定于类型的get和put方法系列相比，视图缓冲区有三个重要优势：

- 视图缓冲区的索引不是以字节为单位，而是以其值的类型特定大小为单位
- 视图缓冲区提供相对的大容量get和put方法，这些方法可以在缓冲区和数组或同一类型的其他缓冲区之间传输连续的值序列
- 视图缓冲区的效率可能要高得多，如果它是 direct buffer



视图缓冲区的字节顺序固定为创建视图时其字节缓冲区的字节顺序。



#### Invocation chaining

这类的方法返回的是当前的 buffer。此类方法可以被链式调用

```text
以下代码
 bb.putInt(0xCAFEBABE);
 bb.putShort(3);
 bb.putShort(45);
可被以下代码代替
 bb.putInt(0xCAFEBABE).putShort(3).putShort(45);
```



| 返回类型                                                     | 方法名                                                       | 描述                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------- |
| static [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **allocate**(int capacity)                                   | 分配一个新的 byte buffer               |
| static [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **allocateDirect**(int capacity)                             | 分配一个新的 direct byte buffer        |
| byte[]                                                       | **array**()                                                  | 返回此 bytebuffer 对应的 byte[]        |
| int                                                          | **arrayOffset**()                                            | 返回第一个元素的偏移量                 |
| abstract [CharBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/CharBuffer.html) | **asCharBuffer**()                                           | 创建一个 char buffer 视图              |
| abstract XXXBuffer                                           | **asXXXBuffer**()                                            | 创建一个 XXX buffer 视图               |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **compact**()                                                | 压缩此 bytebuffer                      |
| int                                                          | **compareTo**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) that) | Compares this buffer to another        |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **duplicate**()                                              | 创建一个新的 byte buffer 共享此 buffer |
| boolean                                                      | **equals**([Object](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html) ob) |                                        |
| abstract byte                                                | **get**()                                                    | 相对获取方法，获取一个字节             |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **get**(byte[] dst)                                          | 从此缓存区获取字节到 dst 中            |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **get**(byte[] dst, int offset, int length)                  | 从此缓存区获取字节到 dst 中的指定位置  |
| abstract byte                                                | **get**(int index)                                           | 读取缓存区指定位置的字节               |
| abstract XXX                                                 | **getXXX**()                                                 | 从缓存区读取一个 XXX                   |
| abstract XXX                                                 | **getXXX**(int index)                                        | 从缓存区指定位置读取一个 XXX           |
| boolean                                                      | **hasArray**()                                               | 该 buffer 是否由 byte array 创建而来   |
| int                                                          | **hashCode**()                                               |                                        |
| abstract boolean                                             | **isDirect**()                                               | 该 buffer 是否是 direct buffer         |
| [ByteOrder](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteOrder.html) | **order**()                                                  | Retrieves this buffer's byte order     |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **order**([ByteOrder](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteOrder.html) bo) | Modifies this buffer's byte order      |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **put**(byte b)                                              | 将字节加入缓冲区                       |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **put**(byte[] src)                                          | 将字节数组中的字节加入缓冲区           |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **put**(byte[] src, int offset, int length)                  | 将字节数组指定部分的字节加入缓冲区     |
| [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **put**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) src) | 将其他 buffer 加入此缓存区             |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **put**(int index, byte b)                                   | 向该 buffer 的指定位置加入字节         |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **putXXX**(char value)                                       | 将XXX加入缓冲区                        |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **putXXX**(int index, char value)                            | 将XXX加入缓冲区的指定位置              |
| abstract [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **slice**()                                                  | 创建一个新的独立的 byte buffer         |
| [String](https://docs.oracle.com/javase/7/docs/api/java/lang/String.html) | **toString**()                                               |                                        |
| static [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **wrap**(byte[] array)                                       | 将字节数组包装成缓冲区                 |
| static [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) | **wrap**(byte[] array, int offset, int length)               | 将字节数组包装成缓冲区                 |



















<https://my.oschina.net/flashsword/blog/159613>
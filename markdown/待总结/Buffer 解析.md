一个存储基础类型数据的容器

缓冲器是特定基本类型元素的线性、有限序列。除了内容外，缓冲区的基本特性是capacity、limit和position：

- capacity：缓冲区的容量即包含的元素个数。缓冲区的容量永远不会是负的，也不会改变。
- limit：读写操作的界限。缓冲区的限制永远不会为负，也永远不会大于 capacity。
- position：要读或写的**下一个元素**的索引。缓冲区的位置从不为负，也从不超过 limit。

对于每个非布尔基本类型，此类都有一个子类。



#### Transferring data

此类的每个子类都定义了两类get和put操作：

- 相对操作从当前位置开始读取或写入一个或多个元素，然后按传输的元素数递增该位置。如果请求的传输超过限制，则相对get操作抛出BufferUnderflowException，相对put操作抛出BufferOverflowException；在这两种情况下，都不会传输数据。
- 绝对操作采用显式元素索引，不影响位置。如果索引参数超过限制，绝对get和put操作将引发indexOutboundsException。

数据也可以通过适当 channel 的I/O操作传入或传出缓冲区，该信道总是相对于当前位置的。



#### Marking and resetting

缓冲区的 mark 是调用 reset 方法时将其 position 重置到 mark 处的位置。mark 并不一定会被使用，但当它被定义时，它不会为负，也不会大于当前 position。如果定义了 mark，则当将 position 或 limit 调整为小于 mark 的值时，该标记将被丢弃。如果未定义 mark ，则调用reset方法将引发InvalidMarkException。



#### Invariants

`0` `<=` *mark* `<=` *position* `<=` *limit* `<=` *capacity* 

新创建的缓冲区总是有一个零位置和一个未定义的 mark。初始限制可以是零，也可以是其他一些值，这取决于缓冲区的类型和构造方式。新分配的缓冲区的每个元素都初始化为零



#### Clearing, flipping, and rewinding

 除了 position, limit,capacity 以及 mark 和 reset 方法外，此类还定义了对缓冲区的以下操作：

- clear()：将当前 buffer 的 limit 设为 capacity，position 设为 0。可以进行 put 操作。limit == capacity，position == 0
- flip()：将当前 buffer 的 limit 设置为 position，position 设为 0。**为了进行 get 操作**。limit == position，position == 0
- rewind()：使缓冲区准备好重新读取它已经包含的数据：它保持 limit 不变，并将 position 设置为零。





### clear()

```
 buf.clear();     // Prepare buffer for reading
 in.read(buf);    // Read data
```

### flip()

```
 buf.put(magic);    // Prepend header
 in.read(buf);      // Read data into rest of buffer
 buf.flip();        // Flip buffer
 out.write(buf);    // Write header + data to channel
```

### rewind()

```
 out.write(buf);    // Write remaining data
 buf.rewind();      // Rewind buffer
 buf.get(array);    // Copy data into array
```

#### Read-only buffers

- 每个缓冲区都是可读的，但不是每个缓冲区都是可写的。当在只读缓冲区上做非读操作时，这些操作将抛出ReadOnlyBufferException
- 只读缓冲区不允许更改其内容，但其标记、位置和限制值是可变的
- 缓冲区是否为只读可以通过调用其isReadOnly方法来确定



#### Thread safety

- buffer 不是线程安全的



#### Invocation chaining

 ```text
 b.flip();
 b.position(23);
 b.limit(42);
can be replaced by the single, more compact statement
 b.flip().position(23).limit(42);
 ```



| 返回类型                                                     | 方法名                        | 描述                                                      |
| ------------------------------------------------------------ | ----------------------------- | --------------------------------------------------------- |
| abstract [Object](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html) | **array**()                   | 返回 buffer 对应的字节数组                                |
| abstract int                                                 | **arrayOffset**()             |                                                           |
| int                                                          | **capacity**()                | buffer 的容量                                             |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **clear**()                   | limit = capacity，position = 0.Prepare buffer for reading |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **flip**()                    | limit = position，position = 0                            |
| abstract boolean                                             | **hasArray**()                |                                                           |
| boolean                                                      | **hasRemaining**()            | position < limit ? true : false                           |
| abstract boolean                                             | **isDirect**()                |                                                           |
| abstract boolean                                             | **isReadOnly**()              |                                                           |
| int                                                          | **limit**()                   | 返回 limit 的值                                           |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **limit**(int newLimit)       | 设置 limit 的值                                           |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **mark**()                    | 设置 mark 为 当前 position 的值                           |
| int                                                          | **position**()                | 返回当前位置                                              |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **position**(int newPosition) | 设置 position 的值                                        |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **remaining**()               | limit - position                                          |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **reset**()                   | 将 position 设置为 mark                                   |
| [Buffer](https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html) | **rewind**()                  | position = 0                                              |











<https://docs.oracle.com/javase/7/docs/api/java/nio/Buffer.html>
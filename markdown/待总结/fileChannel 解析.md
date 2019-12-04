fileChannel 是用于读取、写入、映射和操作文件的通道。

文件通道是连接到文件的SeekableByteChannel。它在文件中有一个可以用于查询和修改的当前位置。文件本身包含一个可变长度的字节序列，可以读取和写入，并且可以查询其当前大小。当写入的字节超过当前大小时，**文件大小会增加**；当文件被截断时，文件大小会减小。该文件还可能具有一些关联的元数据，如访问权限、内容类型和上次修改时间；此类不定义元数据访问的方法。



除了熟悉的字节通道的读、写和关闭操作外，此类还定义了以下特定于文件的操作：

- 字节可以以**不影响通道当前位置**的方式在文件的绝对位置读取或写入
- 文件的一个区域可以直接映射到内存中；对于大型文件，这通常比调用通常的读或写方法更有效
- 对文件所做的更新可能会被强制输出到底层存储设备，以确保在系统崩溃时不会丢失数据
- 字节可以从一个文件传输到另一个通道，反之亦然，这种方式可以由许多操作系统优化为直接到文件系统缓存或从文件系统缓存快速传输
- 文件的某个区域可能被其他程序锁定，无法访问



fileChannel 是线程安全的。可以随时调用close方法。在任何给定的时间内，只有一个操作涉及通道的位置或可以更改其文件大小，在第一个操作仍在进行时尝试启动第二个此类操作将被阻止，直到第一个操作完成。其他操作(特别是那些指定特定位子的操作)可以并发进行；它们是否真的这样做取决于底层实现，因此没有具体说明。



此类实例提供的文件视图保证与同一程序中其他实例提供的同一文件的其他视图一致。但是，由于底层操作系统执行的缓存和网络文件系统协议引起的延迟，此类实例提供的视图可能与其他并发运行程序看到的视图一致，也可能不一致。不管这些其他程序是用什么语言编写的，也不管它们是在同一台机器上运行还是在其他机器上运行，这都是正确的。任何此类不一致的确切性质都取决于系统，因此没有具体说明。



通过调用该类定义的打开方法之一创建 file channel 。file channel 也可以通过调用 FileInputStream、FileOutputStream 或RandomAccessFile 对象的 getChannel 方法得到 。如果从现有的流或random access file 获得文件通道，那么文件通道的状态与 getChannel 方法返回通道的对象的状态紧密相连。无论是显式地还是通过读取或写入字节来更改通道的位置，都将更改原始对象的文件位置，反之亦然。通过文件通道更改文件的长度将更改通过原始对象看到的长度，反之亦然。通过写入字节更改文件内容将更改原始对象看到的内容，反之亦然。



通过不同方式创建的 file channel 会得到不同类型的文件的读写权限。

- 通过 fileInputStream 得到的具有 read 权限
- 通过 fileOutputStream 得到的具有 write 权限，需要设置 append 参数，默认为 false
- 通过 RandomAccessFile 得到的文件权限，由传入参数指定（"r"：read,"w":write,"rw":read and write）,**这种不能设置 append，前置为 false**



打开以供写入的 file channel 可能处于追加模式，例如，它是从通过调用 FileOutputStream(File,true) 而创建的文件输出流中获取的。在这种模式下，对相对写操作的每次调用首先将位置推进到文件的末尾，然后写入请求的数据。位置的推进和数据的写入是否在单个原子操作中完成取决于系统，因此未指明。



| 返回类型                                                     | 方法名                                                       | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| void                                                         | **force**(boolean metaData)                                  | 强制将 pageCache 中的缓存刷新到磁盘，true 代表同时刷新元数据 |
| [FileLock](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileLock.html) | **lock**()                                                   | 获取此 fileChannel 对应的 file 的独占锁                      |
| [FileLock](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileLock.html) | **lock**(long position, long size, boolean shared)           | 获取 file 指定区域的锁                                       |
| [MappedByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/MappedByteBuffer.html) | **map**([FileChannel.MapMode](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileChannel.MapMode.html) mode, long position, long size) | 将此 file 指定区域映射到内存中                               |
| static [FileChannel](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileChannel.html) | **open**([Path](https://docs.oracle.com/javase/7/docs/api/java/nio/file/Path.html) path, [Set](https://docs.oracle.com/javase/7/docs/api/java/util/Set.html)\<?> options,FileAtrribute\<?> attrs | 打开或创建一个 file，并返回对应的 channel                    |
| long                                                         | **position**()                                               | 返回 file channel 当前的位置                                 |
| [FileChannel](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileChannel.html) | **position**(long newPosition)                               | 设置当前 fileChannel 的 position                             |
| int                                                          | **read**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) dst) | 从当前 channel 中读取字节流到 dst 中                         |
| long                                                         | **read**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)[] dsts) | 将 dsts 中的所有 buffer 填满                                 |
| long                                                         | **read**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)[] dsts, int offset, int length) | 将 dsts 指定位置的 buffer 填满                               |
| int                                                          | **read**(ByteBuffer dst, long position)                      | 从指定file位置读取字节流填满 dst                             |
| long                                                         | **size**()                                                   | 返回当前 file 的大小                                         |
| long                                                         | **transferFrom**(ReadableByteChannel src, long position, long count) | 将字节从给定的可读字节通道传输到当前通道的文件中             |
| long                                                         | **transferTo**(long position, long count, [WritableByteChannel](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/WritableByteChannel.html) target) | 将字节从当前通道写入指定通道的文件中                         |
| long                                                         | **truncate**(long size)                                      | 截断文件为指定大小                                           |
| [FileLock](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileLock.html) | **tryLock**()                                                | 同 lock() 但不阻塞                                           |
| [FileLock](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileLock.html) | **tryLock**(long position, long size, boolean shared)        | 同 lock() 但不阻塞                                           |
| int                                                          | **write**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) src) | 将 src 的可读字节流写入文件                                  |
| long                                                         | **write**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)[] srcs) | 将 srcs 的所有可读字节流写入文件                             |
| long                                                         | **write**([ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)[] srcs, int offset, int length) | 指定 src 写入文件                                            |
| int                                                          | **write**(ByteBuffer src, long position)                     | 写入文件的特定位置                                           |



transferFrom、**transferTo** 优势：

这种方法可能比从源通道读取并写入该通道的简单循环更有效。许多操作系统可以直接将字节从源通道传输到文件系统缓存，而不必实际复制它们。



map 操作问题：

对于大多数操作系统，将文件映射到内存中比通过通常的读写方法读取或写入几十千字节的数据要昂贵。从性能的角度来看，通常只需要将相对较大的文件映射到内存中。



read/write 注意事项：

read/write 操作的 buffer 如果不是 DirectBuffer 最终都会创建一个 DirectBuffer buffer 来读写的



| Option                                                       | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`APPEND`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#APPEND) | If this option is present then the file is opened for writing and each invocation of the channel's `write` method first advances the position to the end of the file and then writes the requested data. Whether the advancement of the position and the writing of the data are done in a single atomic operation is system-dependent and therefore unspecified. This option may not be used in conjunction with the `READ` or `TRUNCATE_EXISTING` options. |
| [`TRUNCATE_EXISTING`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#TRUNCATE_EXISTING) | If this option is present then the existing file is truncated to a size of 0 bytes. This option is ignored when the file is opened only for reading. |
| [`CREATE_NEW`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#CREATE_NEW) | If this option is present then a new file is created, failing if the file already exists. When creating a file the check for the existence of the file and the creation of the file if it does not exist is atomic with respect to other file system operations. This option is ignored when the file is opened only for reading. |
| [`CREATE`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#CREATE) | If this option is present then an existing file is opened if it exists, otherwise a new file is created. When creating a file the check for the existence of the file and the creation of the file if it does not exist is atomic with respect to other file system operations. This option is ignored if the `CREATE_NEW` option is also present or the file is opened only for reading. |
| [`DELETE_ON_CLOSE`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#DELETE_ON_CLOSE) | When this option is present then the implementation makes a *best effort* attempt to delete the file when closed by the the [`close`](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/spi/AbstractInterruptibleChannel.html#close()) method. If the `close` method is not invoked then a *best effort* attempt is made to delete the file when the Java virtual machine terminates. |
| [`SPARSE`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#SPARSE) | When creating a new file this option is a *hint* that the new file will be sparse. This option is ignored when not creating a new file. |
| [`SYNC`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#SYNC) | Requires that every update to the file's content or metadata be written synchronously to the underlying storage device. (see [Synchronized I/O file integrity](https://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html#integrity)). |
|                                                              |                                                              |
| [`DSYNC`](https://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardOpenOption.html#DSYNC) | Requires that every update to the file's content be written synchronously to the underlying storage device. (see [Synchronized I/O file integrity](https://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html#integrity)). |
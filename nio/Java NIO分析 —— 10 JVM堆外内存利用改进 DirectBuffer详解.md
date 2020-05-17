前面我们详细讲了[Java NIO分析(8): 高并发核心Selector详解](http://sound2gd.wang/2018/07/12/Java-NIO%E5%88%86%E6%9E%90-8-Selector%E8%AF%A6%E8%A7%A3/)和[Java NIO分析(9): 从BSD socket到SocketChannel](http://sound2gd.wang/2018/07/16/Java-NIO%E5%88%86%E6%9E%90-9-%E4%BB%8EBSD-socket%E5%88%B0SocketChannel/), 分别是NIO的事件分发器和非阻塞处理器.

为了支持`Channel`的双向读写和`Scatter/Gather`操作，我们还需要`Buffer`,将I/O数据存储备用。普通的Buffer都是JVM堆内的Buffer, 比较好理解.

接下来我们聊聊JVM使用堆外内存的沧桑历史以及为什么要设计出`DirectBuffer`。

首先我们要从没有NIO的前JDK1.4时代开始说起，那会儿大家使用的是`SocketInputStream`和`SocketOutputStream`.

# 1. Java传统的Socket是如何收发数据的?

以`SocketOutputStream`为例，继承类图如下：

[![img](http://img.sound2gd.wang/2018/07/22/20180722092445.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/22/20180722092445.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

其实大家应该比较熟悉，`java.io`包里最根本的抽象就是`InputStream`和`OutputStream`,其余的类抽象和实现都是一堆**装饰器**。
所以我们要跟踪的就是`write方法`

```
// java.io.SocketOutputStream
public void write(byte b[], int off, int len) throws IOException {
    // 委托给本类的socketWrite方法
    socketWrite(b, off, len);
}

    /**
     * Writes to the socket with appropriate locking of the
     * FileDescriptor.
     * @param b the data to be written
     * @param off the start offset in the data
     * @param len the number of bytes that are written
     * @exception IOException If an I/O error has occurred.
     */
private void socketWrite(byte b[], int off, int len) throws IOException {
    ...省略非关键代码
    FileDescriptor fd = impl.acquireFD();
    try {
        // 上面就是做了一些参数判断，然后委托给socketWrite0
        socketWrite0(fd, b, off, len);
    } catch (SocketException se) {
    ...
    } finally {
        impl.releaseFD();
    }
}

    /**
     * Writes to the socket.
     * @param fd the FileDescriptor
     * @param b the data to be written
     * @param off the start offset in the data
     * @param len the number of bytes that are written
     * @exception IOException If an I/O error has occurred.
     */
private native void socketWrite0(FileDescriptor fd, byte[] b, int off,
                                 int len) throws IOException;
```

最后调用的是一个native方法`socketWrite0`,打开`jdk/src/solaris/native/java/net/SocketOutputStream.c`

```
/*
 * Class:     java_net_SocketOutputStream
 * Method:    socketWrite0
 * Signature: (Ljava/io/FileDescriptor;[BII)V
 */
JNIEXPORT void JNICALL
Java_java_net_SocketOutputStream_socketWrite0(JNIEnv *env, jobject this,
                                              jobject fdObj,
                                              jbyteArray data,
                                              jint off, jint len) {
    char *bufP;
    char BUF[MAX_BUFFER_LEN];
    int buflen;
    int fd;

    if (IS_NULL(fdObj)) {
        JNU_ThrowByName(env, "java/net/SocketException", "Socket closed");
        return;
    } else {
        fd = (*env)->GetIntField(env, fdObj, IO_fd_fdID);
        /* Bug 4086704 - If the Socket associated with this file descriptor
         * was closed (sysCloseFD), the the file descriptor is set to -1.
         */
        if (fd == -1) {
            JNU_ThrowByName(env, "java/net/SocketException", "Socket closed");
            return;
        }

    }

    if (len <= MAX_BUFFER_LEN) {
        bufP = BUF;
        buflen = MAX_BUFFER_LEN;
    } else {
        buflen = min(MAX_HEAP_BUFFER_LEN, len);
        // 初始化一块直接内存
        bufP = (char *)malloc((size_t)buflen);

        /* if heap exhausted resort to stack buffer */
        if (bufP == NULL) {
            bufP = BUF;
            buflen = MAX_BUFFER_LEN;
        }
    }

    while(len > 0) {
        int loff = 0;
        int chunkLen = min(buflen, len);
        int llen = chunkLen;
        // 将堆内的data复制到刚才初始化的内存bufP里
        (*env)->GetByteArrayRegion(env, data, off, chunkLen, (jbyte *)bufP);

        while(llen > 0) {
            // 发送数据
            int n = NET_Send(fd, bufP + loff, llen, 0);
            if (n > 0) {
                llen -= n;
                loff += n;
                continue;
            }
            if (n == JVM_IO_INTR) {
                JNU_ThrowByName(env, "java/io/InterruptedIOException", 0);
            } else {
                if (errno == ECONNRESET) {
                    JNU_ThrowByName(env, "sun/net/ConnectionResetException",
                        "Connection reset");
                } else {
                    NET_ThrowByNameWithLastError(env, "java/net/SocketException",
                        "Write failed");
                }
            }
            if (bufP != BUF) {
                free(bufP);
            }
            return;
        }
        len -= chunkLen;
        off += chunkLen;
    }

    if (bufP != BUF) {
        free(bufP);
    }
}
```

这段比较短，就没有省略代码了。
大致分3步:

1. 申请一块直接内存bufP, 长度为len
2. 将堆内的data复制到刚才初始化的内存bufP里
3. 调用Net_send函数发送数据

`Net_send`是一个宏，定义在`net_util_md.h`里

```
// net_util_md.h
#define NET_Send        JVM_Send

// jvm.cpp
JVM_LEAF(jint, JVM_Send(jint fd, char *buf, jint nBytes, jint flags))
  JVMWrapper2("JVM_Send (0x%x)", fd);
  //%note jvm_r6
  return os::send(fd, buf, (size_t)nBytes, (uint)flags);
JVM_END

// os_linux.inline.hpp
inline int os::send(int fd, char* buf, size_t nBytes, uint flags) {
  RESTARTABLE_RETURN_INT(::send(fd, buf, nBytes, flags));
}
```

这个宏替换的是`JVM_Send`,是一个cpp封装方法，最后会调用`os::send`,这个在`os_linux.inline.hpp`里，可以看到，最后调用的是一个全局函数`send`。

`send`是我们的老朋友了，属于POSIX的标准`Socket API`, 如果你在类Unix系统上，都可以通过`man send`来翻看它的文档。它的函数签名如下：

```
#include <sys/socket.h>

 ssize_t
 send(int socket, const void *buffer, size_t length, int flags);
```

这个函数用来将首地址为buffer, 长度为length的数据发送到`socket fd`.

[![img](http://img.sound2gd.wang/2018/07/22/20180722095054.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/22/20180722095054.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

所以传统的Java Socket编程每次发送数据的时候，都会申请一块直接内存(堆外)，然后从堆内复制到堆外，最后在调用send发送

为什么要把数据从堆内复制到堆外呢？因为**堆内的对象地址会随着gc改变，在send的时候会崩**。

在高并发场景下，这是非常费内存的，假如每个链接发送的数据是1k, 那么堆内有1K的数据，堆外还要申请1K的数据，还要做数据拷贝，百万链接就需要2T的内存(极端场景), 无疑是瓶颈之一。

# 2. NIO的解决方案: DirectBuffer

既然Socket Api一定要用堆外的内存，一个解决思路就是**复用这块内存**，这样就**不必每次都申请一块新的内存，减少系统调用损耗**。

**计算机世界就是解决一个问题的同时带来新的问题**，如果复用了这块内存，如何在不用的时候进行回收就是一个问题了，因为这块**内存在堆外，JVM的gc管不到**,放着不管又迟早触发Linux的`OOM Killer`.

jdk1.4以后的NIO带来了解决方案: `DirectBuffer`

## 2.1 DirectBuffer

JVM堆内的对象是带gc的，那么将JVM堆内的对象关联到堆外，在回收堆内对象的时候触发一个回收堆外内存的操作，就可以解决这个问题了。

这个设计思路的常用实现就是`DirectByteBuffer`, 打开其代码

```
// Primary constructor
//
DirectByteBuffer(int cap) {                   // package-private
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    // 主要是记录jdk已经使用的直接内存的数量，当分配直接内存时，需要进行增加，当释放时，需要减少
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 分配直接内存
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    // 内存清零
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 创建Cleaner对象
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}

private ByteBuffer putShort(long a, short x) {
    if (unaligned) {
        short y = (x);
        unsafe.putShort(a, (nativeByteOrder ? y : Bits.swap(y)));
    } else {
        Bits.putShort(a, x, bigEndian);
    }
    return this;
}
```

在构造器里就会申请一块直接内存，大小就是`ByteBuffer.allocateDirect`的时候指定的。

在调用`DirectByteBuffer`的`PutXXX`的时候，都是写入到这块直接内存的，无需再去`malloc`.

所以`SocketChannel`使用`DirectBuffer`进行读写的时候，性能远比`SocketOutputStream`高，需要的内存也没有它多, 对`SocketChannel`有疑问的可以看看前面分析的[Java NIO分析(9): 从BSD socket到SocketChannel](http://sound2gd.wang/2018/07/16/Java-NIO%E5%88%86%E6%9E%90-9-%E4%BB%8EBSD-socket%E5%88%B0SocketChannel/)

## 2.2 何时回收直接内存?

上面的构造器最后一行是使用了`Cleaner`对象, 这个类是个`虚引用`类,继承自`PhantomReference`.

```
// 创建Cleaner对象
cleaner = Cleaner.create(this,
        new Deallocator(base, size, cap));
```

create第一个参数是要观察的引用，第二个参数是引用被回收触发的操作。

在JVM中，虚引用是为了实现**更细粒度的内存控制的手段**，在创建虚引用的时候必须传入一个**引用队列(ReferenceQueue)**，在一个对象的finalize函数被调用之后，这个对象的虚引用会被加入**引用队列**, 通过检查队列就可以知道对象是不是要被回收了。

`sun.misc.Cleaner`就是一个自带`ReferenceQueue`的类, 在`create`的时候会将`DirectBuffer`的引用加入观察，一旦引用被回收，JVM将会通知Cleaner去执行回收操作。

```
public class Cleaner
    extends PhantomReference<Object>
{

    // 引用队列
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();

    ...
    private Cleaner(Object referent, Runnable thunk) {
        super(referent, dummyQueue);
        this.thunk = thunk;
    }
    ...
}
```

## 2.3 如何回收直接内存?

在`DirectBuffer`中有个内部类`Deallocator`

```
private static class Deallocator
    implements Runnable
{

    private static Unsafe unsafe = Unsafe.getUnsafe();

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        // 调用unsafe释放内存
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}
```

在回收的时候调用其run方法，可以看到就是调用`sun.misc.unsafe`去释放的。这里提下,unsafe类里都是静态native方法，其实就是堆大家常用的各种`free`,`malloc`这样的内存操作函数的封装，因为java不能操作直接内存。

## One more thing

构造`DirectBuffer`的时候，有一行`Bits.reserveMemory`, 功能主要是记录jdk已经使用的直接内存的数量, but会判断有没有足够多的直接内存使用，如果内存不够，会触发gc..

```
// These methods should be called whenever direct memory is allocated or
// freed.  They allow the user to control the amount of direct memory
// which a process may access.  All sizes are specified in bytes.
static void reserveMemory(long size, int cap) {
    synchronized (Bits.class) {
        // 判断内存够不够， 够的话直接增加内存引用的计数
        if (!memoryLimitSet && VM.isBooted()) {
            maxMemory = VM.maxDirectMemory();
            memoryLimitSet = true;
        }
        // -XX:MaxDirectMemorySize limits the total capacity rather than the
        // actual memory usage, which will differ when buffers are page
        // aligned.
        if (cap <= maxMemory - totalCapacity) {
            reservedMemory += size;
            totalCapacity += cap;
            count++;
            return;
        }
    }

    // 内存不够就去建议JVM gc了。。
    System.gc();
    try {
        // 等待垃圾回收
        Thread.sleep(100);
    } catch (InterruptedException x) {
        // Restore interrupt status
        Thread.currentThread().interrupt();
    }
    synchronized (Bits.class) {
        // 增加直接内存的计数
        if (totalCapacity + cap > maxMemory)
            throw new OutOfMemoryError("Direct buffer memory");
        reservedMemory += size;
        totalCapacity += cap;
        count++;
    }

}
```

# 3. 总结

本文从`SocketStream`开始讲起，分析了祖传BIO里Socket的内存缺陷和性能瓶颈:

1. **每次发送都需要malloc一块新直接内存，**
2. **要将数据从堆内copy到堆外,然后再send**.

然后介绍了NIO的对堆外内存改进–即**复用堆外内存**, 以及复用堆外内存引出的堆外内存回收问题， 并从源码层面解读了其如何创建和回收。NIO的改进只是针对1，但是对2没改进，没准儿这是java吃内存的原因之一。

`DirectBuffer`是我们常用的`冰山对象`，使用的时候要慎重，如果该对象在年轻代, 容易gc, 回收的时候直接内存可以被释放掉。但是在存活了几次gc之后，被移入年老代，就不容易gc了。 其所关联的直接内存也不会被释放。

这也是常见的一个内存泄漏原因，看JVM堆内没用多少内存，但是机器上内存所剩无几，还频繁gc, 频繁触发`OOM Killer`, 希望看完本文大家能意识到问题出在哪 :)
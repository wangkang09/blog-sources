前面我们讲了高并发核心`Selector`的源码分析，看到其对操作系统I/O多路复用的简单封装。
有了I/O多路复用之后，我们还需要**非阻塞的socket读写操作**.

因为内核告诉你**A连接**有数据可读，你想要读1K, 事实上只读到了0.5K, 如果使用传统的
socket API, 那么进程或者线程会在这里**阻塞**，浪费了CPU的时钟周期和珍贵的线程资源。
使用非阻塞就能在没有读满之前立刻返回，数据先放内存里，然后继续读下一个**B连接**的数据。

`SocketChannel`就是NIO对于非阻塞socket操作的支持的组件，其在socket上封装了一层, 所以我们先从`Socket API`说起。

# 1. 大名鼎鼎的Socket简介

在[Java NIO分析(2): I/O多路复用历史杂谈](http://sound2gd.wang/2018/06/17/Java-NIO%E5%88%86%E6%9E%90-2-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E5%8E%86%E5%8F%B2%E6%9D%82%E8%B0%88/)中我们讲过，1982年, BSD那帮人发布了[Socket和TCP/IP协议栈](http://digitalassets.lib.berkeley.edu/techreports/ucb/text/CSD-83-146.pdf)和select系统调用.

当时Unix系统严重缺乏进程间通信的手段, 全靠一手出神入化的fork大法在维持
所以Socket发布的时候，是用来做`进程间通信(IPC)的`.

放在36年后的今天，这个定义也不过时
**网络通信其实就是不同机器的不同操作系统上socket之间的通信**.

## 1.1 Socket API怎么玩？

先放张图，从[IBM那里的文章](https://www.ibm.com/support/knowledgecenter/ssw_ibm_i_71/rzab6/howdosockets.htm)抄过来的

[![img](http://img.sound2gd.wang/2018/07/16/20180716081628.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/16/20180716081628.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

这就是典型的socket通信的流程。

1. 通过socket()函数创建一个socket fd, 代表通信端点
2. 绑定端口，协议栈，Socket类型(TCP就是流式Socket)
3. 监听, 完了就可以接口客户端的TCP链接了，这个时候建立的TCP链接和accept的不一样，会存在内核的某个队列里，长度由你们都熟悉的`SO_BACKLOG`指定
4. 接收链接
5. 通信, 愉快的交换数据
6. 关闭链接

socket通信需要知道5元组(本地IP+端口，服务器IP+端口，协议)

话说TCP协议实现上这里有个历史缺陷，也是`SYN Flood`的攻击原理，在TCP3次握手的时候，假如只握手一次，之后就不管了。那么`SYN队列`会被撑满，之后就不能建立TCP链接了(一般会被拒绝，看操作系统实现).比较可怕的是这都是发生在内核的，还没到应用层, 开发人员一脸懵逼

[![img](http://img.sound2gd.wang/2018/07/16/20180716083309.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/16/20180716083309.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

## 1.2 TCP/IP发过来的包去哪了？

TCP/IP协议栈将网络分成四层，分别是:

- 应用层: 通常就是指你自己写的程序
- 传输层(TCP): 其实传输层协议还有UDP,只是我们平时用得少。TCP是一种面向流的可靠传输
- 网络层(IP): 基本就是靠IP协议，知道地址和端口来寻找目标机器
- 物理层: 就是光纤，网线这种东西

[![img](http://img.sound2gd.wang/2018/07/17/20180717081127.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/17/20180717081127.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

粗略得讲，IP协议保证网络包通过路由器能投递给目标机器网卡，经过网卡驱动会触发一个中断给
内核，内核会根据TCP/IP协议栈做CRC校检，然后层层解包还原用户数据,然后复制数据到socket
的读写缓冲区. 以上操作都是在**内核空间**完成的。

[![img](http://img.sound2gd.wang/2018/07/16/20180716224333.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/16/20180716224333.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

如果socket fd被加入到了多路复用的监听队列里，如`epoll_ctl`加入的fd,那么在下次`epoll_wait`的时候，将会返回该socket有数据可读可写, 过程即完整的一次网络I/O事件通知。

这个时候**用户空间**内的应用进程直接调用socket的read方法，内核就会将数据从socket的读缓冲区复制到应用进程的缓冲区了。

# 2. SocketChannel详解

`SocketChannel`是对传统Java Socket API的改进，主要是支持了**非阻塞的读写**。同时改进了传统的单向流API, Channel同时支持读写(其实就是加了个中间层`Buffer`)。

## 2.1 创建一个SocketChannel时做了什么

通过`SocketChannel.open()`可以打开一个`SocketChannel`, 最后还是委托给`SelectorProvider`的`openSocketChannel`方法

```
// sun.nio.ch.SelectorProvider
public SocketChannel openSocketChannel() throws IOException {
    // 调用SocketChannelImpl的构造器
    return new SocketChannelImpl(this);
}

// sun.nio.ch.SocketChannelImpl
SocketChannelImpl(SelectorProvider sp) throws IOException {
    super(sp);
    // 创建socket fd
    this.fd = Net.socket(true);
    // 获取socket fd的值
    this.fdVal = IOUtil.fdVal(fd);
    // 初始化SocketChannel状态, 状态不多，总共就6个
    // 未初始化，未连接，正在连接，已连接，断开连接中，已断开
    this.state = ST_UNCONNECTED;
}

// sun.nio.ch.Net
static FileDescriptor socket(ProtocolFamily family, boolean stream)
    throws IOException {
    boolean preferIPv6 = isIPv6Available() &&
        (family != StandardProtocolFamily.INET);
    // 最后调用的是socket0
    return IOUtil.newFD(socket0(preferIPv6, stream, false));
}

// Due to oddities SO_REUSEADDR on windows reuse is ignored
private static native int socket0(boolean preferIPv6, boolean stream, boolean reuse);
```

可以看到，最后还是靠一个native方法`socket0`来创建socket fd,打开`jdk/src/solaris/native/sun/nio/ch/Net.c`

```
JNIEXPORT int JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse)
{
    int fd;
    int type = (stream ? SOCK_STREAM : SOCK_DGRAM);

    // 老朋友socket函数
    fd = socket(domain, type, 0);
    if (fd < 0) {
        return handleSocketError(env, errno);
    }

    ....省略非关键代码

    // 设置是否重用地址，如果打开的是ServerSocketChannel
    // 默认是重用的，其他普通SocketChannel默认不重用
    // 重用和不重用的区别在于，就算你关掉了程序，你绑定的
    // 本地端口也在一定时间内是已使用的(address already in use)
    if (reuse) {
        int arg = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set SO_REUSEADDR");
            close(fd);
            return -1;
        }
    }
    ...
    return fd;
}
```

果然，底层还是`socket`函数，这样一个socket fd就创建好了。

PS: 其实创建个socket fd操作系统内核做了很多事情的，要判断一大堆东西，还要创建和初始化读写缓冲区，加自旋锁等.

## 2.2 如何实现非阻塞

正常在c里我们实现非阻塞是靠`fcntl`这个函数，这个函数全称就是`file control`,
通过它可以管理fd的各种属性，比如设置fd的阻塞与否。

`fcntl`的函数签名为:

```
#include <fcntl.h>

int fcntl(int fildes, int cmd, ...);
```

第一个参数是传入的fd, 第二个参数是操作类型，后面是flag
要设置非阻塞，操作类型是`F_SETFL`和`F_GETFL`,flag是`O_NONBLOCK`

那么JVM是怎么做的呢，在SocketChannel上有一个`configureBlocking`函数，这个函数是设置当前`SocketChannel`是否是阻塞的，和selector一起用的时候一定要设置成非阻塞才有意义, 阻塞的话就不需要IO多路复用的事件通知了。

```
// java.nio.channels.spi.AbstractSelectableChannel
public final SelectableChannel configureBlocking(boolean block)
    throws IOException
{
    ...
    // 模板方法模式，调用子类的实现
    implConfigureBlocking(block);
    ...
    return this;
}
```

在`SocketChannelImpl`里看

```
protected void implConfigureBlocking(boolean block) throws IOException {
    IOUtil.configureBlocking(fd, block);
}
```

将这个操作又交给了`IOUtil`的`configureBlocking`, 同时还传入了我们上面创建的socket fd. 打开`IOUtil`一看

```
public static native void configureBlocking(FileDescriptor fd,
                                            boolean blocking)
    throws IOException;
```

还是要找c的实现，打开`IOUtil.c`

```
JNIEXPORT void JNICALL
Java_sun_nio_ch_IOUtil_configureBlocking(JNIEnv *env, jclass clazz,
                                         jobject fdo, jboolean blocking)
{
    if (configureBlocking(fdval(env, fdo), blocking) < 0)
        JNU_ThrowIOExceptionWithLastError(env, "Configure blocking failed");
}

static int
configureBlocking(int fd, jboolean blocking)
{
    // 所以还是靠file control
    int flags = fcntl(fd, F_GETFL);
    int newflags = blocking ? (flags & ~O_NONBLOCK) : (flags | O_NONBLOCK);

    return (flags == newflags) ? 0 : fcntl(fd, F_SETFL, newflags);
}
```

可以看到，JVM也是靠`fcntl`来实现非阻塞的，所以服务端编程知道一些底层的API还是有价值的和有必要的。

## 2.3 SocketChannel的读写

那么SocketChannel是如何读写呢，打开`SocketChannelImpl`

```
public int read(ByteBuffer buf) throws IOException {
  ...
  // n表示读到的数据长度
  int n = 0;
  for (;;) {
      // 从socket fd里读数据，长度由buf决定
      n = IOUtil.read(fd, buf, -1, nd);
      if ((n == IOStatus.INTERRUPTED) && isOpen()) {
          // The system call was interrupted but the channel
          // is still open, so retry
          continue;
      }
      return IOStatus.normalize(n);
  }
  ...
}
```

读交给了`IOUtil`的`read`方法

```
static int read(FileDescriptor fd, ByteBuffer dst, long position,
                NativeDispatcher nd)
    throws IOException
{
    if (dst.isReadOnly())
        throw new IllegalArgumentException("Read-only buffer");
    // 判断是不是DirectBuffer，是直接读进去
    // DirectBuffer是有名的冰山对象，其后可能关联着一堆直接内存
    if (dst instanceof DirectBuffer)
        return readIntoNativeBuffer(fd, dst, position, nd);

    // 如果传入的不是DirectBuffer,那么使用临时的DirectBuffer
    // Substitute a native buffer
    ByteBuffer bb = Util.getTemporaryDirectBuffer(dst.remaining());
    try {
        int n = readIntoNativeBuffer(fd, bb, position, nd);
        bb.flip();
        if (n > 0)
            dst.put(bb);
        return n;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(bb);
    }
}

private static int readIntoNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                        long position, NativeDispatcher nd)
    throws IOException
{
    int pos = bb.position();
    int lim = bb.limit();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

    if (rem == 0)
        return 0;
    int n = 0;

    // 调用本地方法去读
    // 要读socket fd一定要知道起始地址
    // 感兴趣可以看看https://stackoverflow.com/questions/11981474/pread-and-lseek-not-working-on-socket-file-descriptor
    // 调用完毕bb的那个DirectBuffer的直接内存里就有数据了
    if (position != -1) {
        n = nd.pread(fd, ((DirectBuffer)bb).address() + pos,
                     rem, position);
    } else {
        n = nd.read(fd, ((DirectBuffer)bb).address() + pos, rem);
    }
    if (n > 0)
        bb.position(pos + n);
    return n;
}

static native int pread0(FileDescriptor fd, long address, int len,
                         long position) throws IOException;
```

这里解释下为啥一定要用`DirectBuffer`, 在JVM里是有GC的，但在调用Socket Api进行读写通信的时候，需传入的是一个固定的内存地址，假如数据使用的是堆内地址，GC之后对象地址就变了，这时socket读写就会崩。

上面还有最后一个`pread0`本地方法，这个是文件IO函数，第一个参数传入socket fd的时候，**将会从socket的读缓冲区复制数据**到目标地址，这里不细讲,感兴趣可以看看[这篇文章](https://blog.csdn.net/zongcai249/article/details/17598411)

`SocketChannel`如何实现**写操作**就交给读者自行完成了，和读差不多。

# 3. 总结

本节介绍了祖传的`Socket API`是怎么一回事，以及数据是怎么从网络到达应用进程的(因为老有人问).

同时分析了openjdk对`SocketChannel`的实现细节，其实都是对底层`socket`的API封装, 所以熟知一些关键API还是有必要的
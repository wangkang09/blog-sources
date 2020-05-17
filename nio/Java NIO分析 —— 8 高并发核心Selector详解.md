上节[Java NIO分析(7): NIO核心之Channel,Buffer和Selector简介](http://sound2gd.wang/2018/07/10/Java-NIO%E5%88%86%E6%9E%90-7-NIO%E6%A0%B8%E5%BF%83%E5%88%86%E6%9E%90%E4%B9%8BChannel-Buffer%E5%92%8CSelector/)介绍了`Channel`，`Buffer`和`Selector`的基本用法
有了感性认识之后，来看看Selector的底层是如何实现的。

# 1. Selector设计

笔者下载得是[openjdk8](https://download.java.net/openjdk/jdk8)的源码, 画出类图

[![img](http://img.sound2gd.wang/2018/07/14/20180714234754.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/14/20180714234754.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

比较清晰得看到，openjdk中Selector的实现是`SelectorImpl`,
然后SelectorImpl又将职责委托给了具体的平台，比如图中框出的linux2.6以后才有的`EpollSelectorImpl`, Windows平台则是`WindowsSelectorImpl`, `MacOSX`平台是`KQueueSelectorImpl`.

从名字也可以猜到，openjdk肯定在底层还是用`epoll`,`kqueue`，`iocp`这些技术来实现的I/O多路复用。前面 [Java NIO分析(3): I/O多路复用之select系统调用](http://sound2gd.wang/2018/06/28/Java-NIO%E5%88%86%E6%9E%90-3-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bselect%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/) ,[Java NIO分析(4): I/O多路复用之poll系统调用](http://sound2gd.wang/2018/06/29/Java-NIO%E5%88%86%E6%9E%90-4-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bpoll%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/) , [Java NIO分析(5): I/O多路复用之epoll系统调用](http://sound2gd.wang/2018/07/01/Java-NIO%E5%88%86%E6%9E%90-5-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bepoll%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/)写了3篇来说明其用法，感兴趣的读者可以回头看看。

# 2. 获取Selector

众所周知，`Selector.open()`可以得到一个`Selector`实例，怎么实现的呢？

```
// Selector.java
public static Selector open() throws IOException {
    // 首先找到provider,然后再打开Selector
    return SelectorProvider.provider().openSelector();
}

// java.nio.channels.spi.SelectorProvider
    public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                        if (loadProviderFromProperty())
                            return provider;
                        if (loadProviderAsService())
                            return provider;
                            // 这里就是打开Selector的真正方法
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
```

在openjdk中，每个操作系统都有一个`sun.nio.ch.DefaultSelectorProvider`实现，以solaris为例:

```
/**
 * Returns the default SelectorProvider.
 */
public static SelectorProvider create() {
    // 获取OS名称
    String osname = AccessController
        .doPrivileged(new GetPropertyAction("os.name"));
    // 根据名称来创建不同的Selctor
    if (osname.equals("SunOS"))
        return createProvider("sun.nio.ch.DevPollSelectorProvider");
    if (osname.equals("Linux"))
        return createProvider("sun.nio.ch.EPollSelectorProvider");
    return new sun.nio.ch.PollSelectorProvider();
}
```

如果系统名称是`Linux`的话，真正创建的是`sun.nio.ch.EPollSelectorProvider`。如果不是SunOS也不是Linux,就使用`sun.nio.ch.PollSelectorProvider`, 关于`PollSelector`有兴趣的读者自行了解下, 本文仅以实际常用的`EpollSelector`为例探讨。

打开`sun.nio.ch.EPollSelectorProvider`查看`openSelector`方法

```
public AbstractSelector openSelector() throws IOException {
    return new EPollSelectorImpl(this);
}
```

很直观，这样我们在Linux平台就得到了最终的Selector实现:`sun.nio.ch.EPollSelectorImpl`

# 3. EPollSelector如何进行select

epoll系统调用主要分为3个函数

- epoll_create: 创建一个epoll fd,并开辟epoll自己的内核高速cache区，建立红黑树，分配好想要的size的内存对象，建立一个list链表，用于存储准备就绪的事件。
- epoll_wait: 等待内核返回IO事件
- epoll_ctl: 对新旧事件进行新增修改或者删除

## 3.1 Epoll fd的创建

`EPollSelectorImpl`的构造器代码如下:

```
EPollSelectorImpl(SelectorProvider sp) throws IOException {
    super(sp);
    // makePipe返回管道的2个文件描述符，编码在一个long类型的变量中
    // 高32位代表读 低32位代表写
    // 使用pipe为了实现Selector的wakeup逻辑
    long pipeFds = IOUtil.makePipe(false);
    fd0 = (int) (pipeFds >>> 32);
    fd1 = (int) pipeFds;
    // 新建一个EPollArrayWrapper
    pollWrapper = new EPollArrayWrapper();
    pollWrapper.initInterrupt(fd0, fd1);
    fdToKey = new HashMap<>();
}
```

再看`EPollArrayWrapper`的初始化过程

```
EPollArrayWrapper() throws IOException {
    // creates the epoll file descriptor
    // 创建epoll fd
    epfd = epollCreate();

    // the epoll_event array passed to epoll_wait
    int allocationSize = NUM_EPOLLEVENTS * SIZE_EPOLLEVENT;
    pollArray = new AllocatedNativeObject(allocationSize, true);
    pollArrayAddress = pollArray.address();

    // eventHigh needed when using file descriptors > 64k
    if (OPEN_MAX > MAX_UPDATE_ARRAY_SIZE)
        eventsHigh = new HashMap<>();
}

private native int epollCreate();
```

在初始化过程中调用了`epollCreate`方法，这是个native方法。
打开`jdk/src/solaris/native/sun/nio/ch/EPollArrayWrapper.c`

```
JNIEXPORT jint JNICALL
Java_sun_nio_ch_EPollArrayWrapper_epollCreate(JNIEnv *env, jobject this)
{
    /*
     * epoll_create expects a size as a hint to the kernel about how to
     * dimension internal structures. We can't predict the size in advance.
     */
     // 这里的size可以不指定，从Linux2.6.8之后，改用了红黑树结构，指定了大小也没啥用
    int epfd = epoll_create(256);
    if (epfd < 0) {
       JNU_ThrowIOExceptionWithLastError(env, "epoll_create failed");
    }
    return epfd;
}
```

可以看到最后还是使用了操作系统的api: `epoll_create`函数

## 3.2 Epoll wait等待内核IO事件

调用`Selector.select()`,最后会委托给各个实现的`doSelect`方法,限于篇幅不贴出太详细的，这里看下`EpollSelectorImpl`的`doSelect`方法

```
protected int doSelect(long timeout) throws IOException {
    if (closed)
        throw new ClosedSelectorException();
    processDeregisterQueue();
    try {
        begin();
        // 真正的实现是这行
        pollWrapper.poll(timeout);
    } finally {
        end();
    }
    processDeregisterQueue();
    int numKeysUpdated = updateSelectedKeys();

    // 以下基本都是异常处理
    if (pollWrapper.interrupted()) {
        // Clear the wakeup pipe
        pollWrapper.putEventOps(pollWrapper.interruptedIndex(), 0);
        synchronized (interruptLock) {
            pollWrapper.clearInterrupted();
            IOUtil.drain(fd0);
            interruptTriggered = false;
        }
    }
    return numKeysUpdated;
}
```

然后我们去看`pollWrapper.poll`, 打开`jdk/src/solaris/classes/sun/nio/ch/EPollArrayWrapper.java`:

```
int poll(long timeout) throws IOException {
    updateRegistrations();
    // 这个epollWait是不是有点熟悉呢？
    updated = epollWait(pollArrayAddress, NUM_EPOLLEVENTS, timeout, epfd);
    for (int i=0; i<updated; i++) {
        if (getDescriptor(i) == incomingInterruptFD) {
            interruptedIndex = i;
            interrupted = true;
            break;
        }
    }
    return updated;
}

private native int epollWait(long pollAddress, int numfds, long timeout,
                             int epfd) throws IOException;
```

`epollWait`也是个native方法,打开c代码一看:

```
JNIEXPORT jint JNICALL
Java_sun_nio_ch_EPollArrayWrapper_epollWait(JNIEnv *env, jobject this,
                                            jlong address, jint numfds,
                                            jlong timeout, jint epfd)
{
    struct epoll_event *events = jlong_to_ptr(address);
    int res;

    if (timeout <= 0) {           /* Indefinite or no wait */
        // 发起epoll_wait系统调用等待内核事件
        RESTARTABLE(epoll_wait(epfd, events, numfds, timeout), res);
    } else {                      /* Bounded wait; bounded restarts */
        res = iepoll(epfd, events, numfds, timeout);
    }

    if (res < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "epoll_wait failed");
    }
    return res;
}
```

可以看到，最后还是发起的`epoll_wait`系统调用.

## 3.3 epoll control以及openjdk对事件管理的封装

JDK中对于注册到Selector上的IO事件关系是使用`SelectionKey`来表示，代表了`Channel`感兴趣的事件，如`Read`,`Write`,`Connect`,`Accept`.

调用`Selector.register()`时均会将事件存储到`EpollArrayWrapper`的成员变量`eventsLow`和`eventsHigh`中

```
// events for file descriptors with registration changes pending, indexed
// by file descriptor and stored as bytes for efficiency reasons. For
// file descriptors higher than MAX_UPDATE_ARRAY_SIZE (unlimited case at
// least) then the update is stored in a map.
// 使用数组保存事件变更, 数组的最大长度是MAX_UPDATE_ARRAY_SIZE, 最大64*1024
private final byte[] eventsLow = new byte[MAX_UPDATE_ARRAY_SIZE];
// 超过数组长度的事件会缓存到这个map中，等待下次处理
private Map<Integer,Byte> eventsHigh;


/**
 * Sets the pending update events for the given file descriptor. This
 * method has no effect if the update events is already set to KILLED,
 * unless {@code force} is {@code true}.
 */
private void setUpdateEvents(int fd, byte events, boolean force) {
    // 判断fd和数组长度
    if (fd < MAX_UPDATE_ARRAY_SIZE) {
        if ((eventsLow[fd] != KILLED) || force) {
            eventsLow[fd] = events;
        }
    } else {
        Integer key = Integer.valueOf(fd);
        if (!isEventsHighKilled(key) || force) {
            eventsHigh.put(key, Byte.valueOf(events));
        }
    }
}
```

上面看到`EpollArrayWrapper.poll()`的时候, 首先会调用`updateRegistrations`

```
/**
 * Returns the pending update events for the given file descriptor.
 */
private byte getUpdateEvents(int fd) {
    if (fd < MAX_UPDATE_ARRAY_SIZE) {
        return eventsLow[fd];
    } else {
        Byte result = eventsHigh.get(Integer.valueOf(fd));
        // result should never be null
        return result.byteValue();
    }
}

/**
 * Update the pending registrations.
 */
private void updateRegistrations() {
    synchronized (updateLock) {
        int j = 0;
        while (j < updateCount) {
            int fd = updateDescriptors[j];
            // 从保存的eventsLow和eventsHigh里取出事件
            short events = getUpdateEvents(fd);
            boolean isRegistered = registered.get(fd);
            int opcode = 0;

            if (events != KILLED) {
                // 判断操作类型以传给epoll_ctl
                // 没有指定EPOLLET事件类型
                if (isRegistered) {
                    opcode = (events != 0) ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
                } else {
                    opcode = (events != 0) ? EPOLL_CTL_ADD : 0;
                }
                if (opcode != 0) {
                    // 熟悉的epoll_ctl
                    epollCtl(epfd, opcode, fd, events);
                    if (opcode == EPOLL_CTL_ADD) {
                        registered.set(fd);
                    } else if (opcode == EPOLL_CTL_DEL) {
                        registered.clear(fd);
                    }
                }
            }
            j++;
        }
        updateCount = 0;
    }
}
private native void epollCtl(int epfd, int opcode, int fd, int events);
```

在获取到事件之后将操作委托给了`epollCtl`,这又是个native方法，打开相应的c代码一看:

```
JNIEXPORT void JNICALL
Java_sun_nio_ch_EPollArrayWrapper_epollCtl(JNIEnv *env, jobject this, jint epfd,
                                           jint opcode, jint fd, jint events)
{
    struct epoll_event event;
    int res;

    event.events = events;
    event.data.fd = fd;

    // 发起epoll_ctl调用来进行IO事件的管理
    RESTARTABLE(epoll_ctl(epfd, (int)opcode, (int)fd, &event), res);

    /*
     * A channel may be registered with several Selectors. When each Selector
     * is polled a EPOLL_CTL_DEL op will be inserted into its pending update
     * list to remove the file descriptor from epoll. The "last" Selector will
     * close the file descriptor which automatically unregisters it from each
     * epoll descriptor. To avoid costly synchronization between Selectors we
     * allow pending updates to be processed, ignoring errors. The errors are
     * harmless as the last update for the file descriptor is guaranteed to
     * be EPOLL_CTL_DEL.
     */
    if (res < 0 && errno != EBADF && errno != ENOENT && errno != EPERM) {
        JNU_ThrowIOExceptionWithLastError(env, "epoll_ctl failed");
    }
}
```

原来还是我们的老朋友`epoll_ctl`.
有个小细节是jdk没有指定`ET(边缘触发)`还是`LT(水平触发)`,所以默认会用`LT`:)

在`AbstractSelectorImpl`中有3个set保存事件

```
// Public views of the key sets
// 注册的所有事件
private Set<SelectionKey> publicKeys;             // Immutable
// 内核返回的IO事件封装，表示哪些fd有数据可读可写
private Set<SelectionKey> publicSelectedKeys;     // Removal allowed, but not addition

// 取消的事件
private final Set<SelectionKey> cancelledKeys = new HashSet<SelectionKey>();
```

在`EpollArrayWrapper.poll`调用完成之后, 会调用`updateSelectedKeys`来更新上面的仨set

```
private int updateSelectedKeys() {
    int entries = pollWrapper.updated;
    int numKeysUpdated = 0;
    for (int i=0; i<entries; i++) {
        int nextFD = pollWrapper.getDescriptor(i);
        SelectionKeyImpl ski = fdToKey.get(Integer.valueOf(nextFD));
        // ski is null in the case of an interrupt
        if (ski != null) {
            int rOps = pollWrapper.getEventOps(i);
            if (selectedKeys.contains(ski)) {
                if (ski.channel.translateAndSetReadyOps(rOps, ski)) {
                    numKeysUpdated++;
                }
            } else {
                ski.channel.translateAndSetReadyOps(rOps, ski);
                if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
                    selectedKeys.add(ski);
                    numKeysUpdated++;
                }
            }
        }
    }
    return numKeysUpdated;
}
```

代码很直白，拿出事件对set比对操作。

# 4. 总结

[![img](http://img.sound2gd.wang/2018/07/15/20180715101407.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/15/20180715101407.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

jdk中`Selector`是对操作系统的IO多路复用调用的一个封装，在Linux中就是对`epoll`的封装。epoll实质上是将`event loop`交给了内核，因为网络数据都是首先到内核的，直接内核处理可以避免无谓的系统调用和数据拷贝, 性能是最好的。

jdk中对IO事件的封装是`SelectionKey`, 保存`Channel`关心的事件。

至此对Selector的前后实现比较清晰了
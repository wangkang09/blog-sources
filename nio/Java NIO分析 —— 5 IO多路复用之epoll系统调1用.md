前面介绍了Unix的I/O模型以及多路复用的c实现，为什么要介绍这些呢？ 因为JVM是用c++写的，JDK的native方法也都是用c写的，最后它们调用
的还是操作系统底层的api,所以了解一些关键的底层原理还是有必要的。
讲Java的NIO之前，先讲些基础知识.

# 1. 基础知识

## 1.1 阻塞和非阻塞

`阻塞`和`非阻塞`关注的是**应用进程在等待调用结果时的状态**,
如果应用进程在等结果的时候**将自己挂起**，直到得到结果再返回，这是`阻塞`调用
如果应用进程在等结果的时候**不会将自己挂起**就返回, 这是`非阻塞`调用。

举个栗子，你去买星巴克, 如果你点了杯咖啡然后一直等着啥也不干直到咖啡送到你手上，这叫`阻塞等待`，如果你点了杯咖啡后去刷刷微博，然后不时回来看看咖啡好了没有，这叫`非阻塞等待`

## 1.2 同步和异步

而`同步`和`异步`关注的是**通信**

同步就是应用进程在发起一个调用之后，在没有得到结果之前，就不返回(不管有没有挂起自己)，也就是进程(调用方)主动等待结果。

异步是只应用进程在发起一个调用之后，不等待结果就返回了。也就是说，是靠被调用方通过状态或者回调函数来通知进程结果。

举个栗子:
你去买星巴克，点了杯咖啡之后，老板说我先去买原料ABCD,你等啊等，等老板买完做完了给你了你才离开, 这个是同步。
异步就是你点了咖啡之后，你先去happy, 等好了老板打电话大吼一声通知你来取。

## 1.3 之间有关联？

回顾我们之前的Unix基本I/O模型

> `Non-Blocking I/O`和`Signal-Driven I/O`在数据准备阶段都不会阻塞，前者要轮询内核数据是否准备好，后者是直接等待内核通知回调
> `Asynchronous I/O`是在真正意义上的`POSIX`定义的异步io操作, 在数据准备和复制阶段都不会阻塞应用,但是编程难度大
> `I/O Multiplexing`在数据准备阶段也会阻塞，但是可以处理更多I/O请求，也就是说加了层中间人抽象，虽然阻塞了应用进程，但是能知道多个fd可读可写,也是大名鼎鼎的`reactor模式`的基础

从上面的角度看，其实只是关注点不一样，阻塞的调用关注的是等待结果时的状态，所以

- 如果进程或线程是被挂起了，就是阻塞的，那么这个进程/线程也干不了别的，直到拿到结果后返回，所以也是同步的.
- 如果进程或线程没被挂起，就是非阻塞的, 这个进程可以先返回，等啥时候好了再回调或者改状态，这个叫异步. 如果这个进程一直在check直到拿到结果了才返回，这就是同步

## 1.4 BIO介绍

在jdk1.4之前，Java的I/O是使用**基于流的抽象模型**来做的，IO流模型是把设备抽象成一个个管道，管道里每个数据单元依次排列,这是一种**同步阻塞模型**.

类似于一个自来水管，源头放在数据源上，出口是你的Java程序。
流是有方向的，水流出去了就没法让它流回来。

[![img](http://img.sound2gd.wang/2018/07/07/20180707085301.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/07/20180707085301.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

最基础的2个抽象是`InputStream`和`OutputStream`,I/O流都使用了隐式的指针来记录当前准备从哪个数据单元读取，其余的类功能都是在这两个类的基础上做装饰的。

例如为了简化操作提高I/O效率的`BufferedInputStreamReader`，一次java函数调用就能读写大量的内容(应用层看来)，而不是每次处理一个数据单元。

# 2. BIO的局限

JDK1.4之前，java的io包里的输入流和输出流抽象都是**面向流的模型**, 一次只能处理一个字节，效率不高，并且会阻塞进/线程。

在网络编程中，BIO没有引入I/O多路复用模型，并且BIO里的方法都是同步阻塞的，所以通常都是起一个accepter线程去阻塞监听客户端连接,接收到请求之后起一个新的线程去处理。

[![BIO server](http://img.sound2gd.wang/2018/07/06/20180706074513.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/06/20180706074513.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)BIO server

这样有几个明显的缺点:

1. 客户端连接很多的时候，会创建大量的处理线程，每创建一个线程都需要分配一定的栈空间,一般是1K~1M,那么4G内存也只能起4000~40000个线程。
2. 线程多导致上下文切换严重
3. 阻塞导致负责网络数据读写的线程不可复用

其中， 1可以使用线程池来解决，但是线程池的处理能力是有限的。
阻塞导致负责网络数据读写的线程不可复用, 在高并发大量连接的场景下，假如某个线程要从socket里读1k的数据，但是现在客户端网络不行，只发了0.1k, 那么这个线程池里的线程也阻塞在那里, 没有让出资源去读别的socket中的数据,导致整体效率不高，整体连接数也有限。

# 3. NIO的设计

为了解决BIO的问题，在JDK1.4以后，加入了NIO(New IO, 也有说法称Non-Blocking-IO)
NIO的设计依然是基于流的，只是可以非阻塞的读写了。

大家都知道，每个网络服务都有的基础结构:

1. 接收网络请求
2. 解析请求
3. 处理请求,得到结果
4. 加密结果
5. 发送网络响应

NIO的作者`Doug Lea`在[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)里讲到, NIO的设计目标是:

- 持续增加的负载下的优雅降级
- 硬件资源(CPU, 内存，磁盘，宽带)的提升带来IO性能的提升
- 低延迟，流量高峰，可调优

处理过程可以分治，将一个大过程分成n个小任务，这将需要`非阻塞(Non Blocking)`的支持
在合适的时机，触发小任务, 这需要某种`事件分发(Event Dispatch)`手段

`Doug Lea`大佬似乎从`AWT`中得到了启发，使用了Java的`事件驱动的设计(Event Driven Design)`:

- AWT线程 —> `Reactor`,接收I/O事件并分发到合适的处理器
- AWT的ActionListener —> `Handlers`, 执行非阻塞操作
- AWT的addActionListener —-> handler绑定事件

[![img](http://img.sound2gd.wang/2018/07/09/20180709160855.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/09/20180709160855.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

在NIO中，它们的概念实现分别是(NIO网络核心):

- `Channel`, 对非阻塞的支持, `Channel`可以连接文件，socket等
- `Buffer`, 类似数组, 可以直接由`Channnel`读写, DirectByteBuffer可以分配堆外内存
- `Selector`, 就是上面的Reactor, 告知哪些`Channel`上发生了I/O事件
- `SelectionKey`, 代表I/O事件状态和绑定

其中，`Selector`就是I/O多路复用在Java里的封装，由内核来完成事件分发和告知，这样即使只有1个Java线程也能处理很多链接。

其他还有些NIO的特性后续也会谈到，如:

- **内存映射文件**, 这样就可以像访问内存一样来访问文件了。
- 文件传输, 有个`send_file`系统调用，可以直接让磁盘文件数据拷贝到socket缓冲区,或者从socket缓冲区拷贝到文件
- 直接内存， Java的socket往外读写数据都要先复制堆内的数据到堆外，然后再调用send(因为堆内的对象地址会变，socket的send用c写的,地址不能变)

------

参考资料

1. [java中的zero-copy高性能文件数据传输](https://www.ibm.com/developerworks/library/j-zerocopy/)
2. [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
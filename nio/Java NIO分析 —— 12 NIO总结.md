迟来的总结，NIO系列写了11篇了，本篇做个总结吧
写这个系列的起因是各个框架比如`netty`, `tomcat`, `jetty`这些高性能框架的
基石就是`NIO`, 一直想讲讲它们高性能的原因。

# 1. NIO网络编程体系

本系列主要是讲NIO网络编程相关的，因为Java服务端开发最关心的就是这些。
**我们记不住孤立的事实， 知识得体系化**. 这里上一张笔者写本系列画的脑图

[![img](http://img.sound2gd.wang/2018/08/06/20180806105257.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/08/06/20180806105257.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

笔者先从Unix网络编程开始, 讲了5种IO模型，

- 阻塞I/O(blocking I/O)
- 非阻塞I/O(non-blocking I/O)
- I/O复用(I/O multiplexing)
- 信号驱动式I/O(signal-driven I/O)
- 异步I/O(asynchronous I/O)

并指出所有的I/O模型都要经历2个阶段，即`数据准备`和`内核拷贝数据到应用进程`
异步I/O模型虽然效率高，但是编程复杂，在实际用的不算多。前四种模型的区别主要在第一阶段数据准备
第二阶段拷贝数据都会阻塞。

I/O多路复用是使用最广泛的I/O模型，内核帮助你完成了`Event Loop`, 你`select`之后得到的就是
已经准备就绪的fd集合.

接下来研究了下I/O多路复用的历史，知道了没有Socket和select之前的黑历史，全靠进程的fork来完成
进程之间的通信，一个负责读一个负责写，还经常有同步问题, 更早的还有非分时操作系统，那个感兴趣的
可以自己去维基百科瞅瞅。

在[Java NIO分析(3): I/O多路复用之select系统调用](http://sound2gd.wang/2018/06/28/Java-NIO%E5%88%86%E6%9E%90-3-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bselect%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/) ,[Java NIO分析(4): I/O多路复用之poll系统调用](http://sound2gd.wang/2018/06/29/Java-NIO%E5%88%86%E6%9E%90-4-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bpoll%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/) , [Java NIO分析(5): I/O多路复用之epoll系统调用](http://sound2gd.wang/2018/07/01/Java-NIO%E5%88%86%E6%9E%90-5-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B9%8Bepoll%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/)里举了几个实际的使用I/O多路复用的api例子，看了下我们是如何在操作系统提供的api上具体使用多路复用的。

以上基础完成之后，我们介绍了NIO的一些基本概念，如**同步异步，阻塞非阻塞**， 并指出`阻塞`和`非阻塞`关注的是**应用进程在等待调用结果时的状态**, 而`同步`和`异步`关注的是**通信**.再从BIO的设计和缺陷开始讲起，引出为什么要搞NIO.

> 1. 客户端连接很多的时候，会创建大量的处理线程，每创建一个线程都需要分配一定的栈空间,一般是1K~1M,那么4G内存也只能起4000~40000个线程。
> 2. 线程多导致上下文切换严重
> 3. 阻塞导致负责网络数据读写的线程不可复用

之后我们具体分析了Selector, SocketChannel, DirectBuffer的底层实现，指出其本质上使用的依然是底层的`epoll`,`poll`, `select`, `fcntl`这些api,
只是jvm做了一层封装而已。后面我们又分析了Linux中的`zero copy`技术和NIO对它的支持，底层使用的还是`mmap`和`send_file`系统调用。

至此，NIO系列的核心知识基本就讲完了, 关键点就是这些。

# 2. NIO的其他知识

其实NIO还有一些别的改进, 只是对服务端编程用处不大，比如**Java7以后提供了全面的文件系统API支持**

- 使用Path抽象来替代java.io.File, 代表了一个与平台无关的路径
- 提供Files和Paths类来快捷操作文件，比如新建临时文件，删除文件等等
- 提供WatchService来监听文件系统中的文件变化
- 全面的基于异步Channel的IO

举个监听文件变化的简单的例子:

```
/**
 * 使用Path类提供的WatchService来监控文件的变化
 * 
 * @author sound2gd
 *
 */
public class WatchServiceTest {

	public static void main(String[] args) throws Exception {
		WatchService watchService = FileSystems.getDefault().newWatchService();
		// 为C:盘根路径设置注册监听
		Paths.get("C:/").register(watchService, StandardWatchEventKinds.ENTRY_CREATE,
				StandardWatchEventKinds.ENTRY_DELETE, StandardWatchEventKinds.ENTRY_MODIFY);
		while (true) {
			// 获取下一个文件变化事件
			WatchKey key = watchService.take();
			for (WatchEvent<?> event : key.pollEvents()) {
				System.out.println(event.context()+" 文件发生了"+event.kind()+"事件");
			}
			//重设watchkey
			boolean valid = key.reset();
			if(!valid){
				//重设失败则退出
				break;
			}
		}

	}
}
```

# 3. 总结

Java4以后, 通过NIO提供了一套基于I/O多路复用的API, 本系列就讲到这里，笔者认为服务端编程还是需要
比较系统的底层知识的，可以在实践中慢慢学习。

使用Java语言写服务端的好处是轮子都是现成的，坏处亦如是。
框架屏蔽了底层细节，在高并发场景下，如果读者不知底层实现的细节，优化将无从谈起。
而且只看框架的话，Java学到深处就只学会了一些字节码诡计，比如javassist, asm等来做一些黑科技。

也不是说它们不好，它们方便了我们的开发，当然是好事。只是作为一个服务端开发, 真正要专注的东西，笔者认为是

1. 软件架构知识
2. 网络通信和各种协议, 如TCP/UDP细节
3. 操作系统的知识
4. 数据结构和算法
5. 一些工程化和业务划分，异常处理的思路

这些是和语言，框架无关的扎实知识，比如哪天你不用Java了换`golang`了依然用的是`epoll`.
以上，共勉.
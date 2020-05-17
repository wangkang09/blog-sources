上次[Java NIO分析(6): 从BIO到NIO-设计和概念](http://sound2gd.wang/2018/07/06/Java-NIO%E5%88%86%E6%9E%90-6-Java-NIO%E4%B8%AD%E7%9A%84%E6%A6%82%E5%BF%B5/)讲到了NIO的设计思想,
即`Doug Lea`大佬受`AWT`启发得到的**事件驱动机制**, 关键点在于

- 非阻塞处理器
- 事件分发组件

在NIO的API中，`Channel`就是实现非阻塞的组件，而事件分发(Dispatcher)使用的是`Selector`组件，
在传统的I/O流(`Stream`)是有方向的，而NIO支持双向读写，这样就需要将流中的数据读取到某个缓冲组件里，
即`Buffer`组件.

`Buffer`组件还有个特殊的实现`DirectByteBuffer`, 可以申请**堆外内存**，关于为什么要申请堆外内存后续会谈。

# 1. Channel

`Channel`是NIO中用来实现非阻塞数据操作的桥梁，笔者猜测是借鉴的`Berkly Socket`的设计，代表某种通道，
和`I/O Stream`只支持读或者写(单向)不一样，`Channel`同时支持读和写, 但是只能读和写到`Buffer`中，因为
**支持了非阻塞，读出的数据要找个地方临时存放**.

[![img](http://img.sound2gd.wang/2018/07/10/20180710220002.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/10/20180710220002.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

Channel主要实现有:

- SocketChannel
- ServerSocketChannel
- DatagramChannel
- FileChannel

基本类图如下:

[![点击查看大图](http://img.sound2gd.wang/2018/07/10/20180710222719.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/10/20180710222719.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)点击查看大图

以上Channel涵盖了文件，TCP, UDP网络的支持, 也是我们用的最多的。
**Channel都不是手动new出来的**,基本都是用静态方法Open出来的，或者从`BIO`的Stream里封装得到的(本质上也是调用某Channel的open方法)。
比如使用`FileChannel`来读写文件的一个例子:

```
/**
 * 测试FileChannel模拟传统IO用竹筒多次取水的过程
 * 
 * @author sound2gd
 *
 */
public class FileChannelTest2 {

	public static void main(String[] args) throws Exception{
		FileInputStream sr = new FileInputStream("src/com/cris/chapter15/f6/FileChannelTest2.java");
		FileChannel fc = sr.getChannel();
		ByteBuffer bf = ByteBuffer.allocate(256);
		
		//创建Charset对象
		Charset charset = Charset.forName("UTF-8");
		CharsetDecoder decoder = charset.newDecoder();
		
		while((fc.read(bf))!=-1){
			//锁定Buffer的空白区
			bf.flip();
			//转码
			CharBuffer cbuff = decoder.decode(bf);
			System.out.print(cbuff);
			//buffer初始化，用于下一次读取
			bf.clear();
			
		}
		
	}
}
```

用法还是比较简单的，从Channel读数据到Buffer用`read`, 从Buffer写数据到Channel用`write`
这里的`FileChannel`就是从`FileInputStream`上得到的, 查看其源码:

```
public FileChannel getChannel() {
    synchronized (this) {
        if (channel == null) {
            channel = FileChannelImpl.open(fd, path, true, false, this);
        }
        return channel;
    }
}
```

可以看到，还是调用了FileChannelImpl的open方法

## Scatter && Gather

上面的类图还可以看到`ScatteringByteChannel`和`GatheringByteChannel`,它们分别代表`Scatter`和`Gather`操作
Scatter是分散操作，可以将一个Channel里的数据读取到多个Buffer
Gather是聚合操作，可以将多个Buffer的数据读取到一个Channel

在网络编程中这俩是常用操作，比如http协议的解析通常会将header和body分散到俩`Buffer`,方便后续处理
Scatter和Gather的细节限于篇幅不展开叙述，感兴趣的读者可以自行了解

# 2. Buffer

`Buffer`是一个容器，本质上就是一个数组.用于接受从Channel里传过来的数据
Buffer的实现常见有:

[![img](http://img.sound2gd.wang/2018/07/11/20180711215338.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)](http://img.sound2gd.wang/2018/07/11/20180711215338.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

看名字就知道是存放什么类型数据的`Buffer`.
Buffer的创建时通过Buffer类的静态方法来创建的。 Buffer有三个核心概念:

- position:位置，用于指明下一个可以被读出的或者写入的缓冲区位置索引
- limit:界限，第一个不应该被读出或者写入的缓冲区位置索引
- capacity:容量，创建后不能改变

Buffer类有一个实例方法:flip()。其作用是将limit设置为position所在的位置，然后将position置为0 ，这就使得Buffer的读写指针又回到了开始位置。 clear()方法就是将position置为0，limit置为capacity.
为啥要有这种操作？因为Buffer是支持读和写的，写完了给别的地方用就要flip, 免得数据处理出错
下面以`CharBuffer`为例举个简单的例子

```
public static void main(String[] args) {
		// 创建Buffer
		CharBuffer buffer = CharBuffer.allocate(8);
		System.out.println("buffer的容量:" + buffer.capacity());
		System.out.println("buffer的位置:" + buffer.position());
		System.out.println("buffer的界限:" + buffer.limit());

		buffer.put('s');
		buffer.put('o');
		buffer.put('u');
		buffer.put('n');
		buffer.put('d');
		System.out.println("加入5个元素后position:" + buffer.position());

		// 调用flip
		buffer.flip();
		System.out.println("buffer的容量:" + buffer.capacity());
		System.out.println("buffer的位置:" + buffer.position());
		System.out.println("buffer的界限:" + buffer.limit());

		// 取出第一个元素
		System.out.print("buffer中的元素:" + buffer.get());
		while (buffer.hasRemaining()) {
			System.out.print(buffer.get());
		}
		System.out.println();
		System.out.println("取出第一个元素后position=" + buffer.position());

		// 调用clear
		buffer.clear();
		System.out.println("第3个元素" + buffer.get(2));

	}
```

输出结果请读者自行理解下

## DirectByteBuffer

上面还有个特殊的类，就是这个`DirectByteBuffer`, 这个是有名的`冰山对象`，
分配DirectByteBuffer的时候，JVM是申请一块直接内存(堆外), 然后地址关联到`DirectByteBufer`里的address
它的回收器`sun.misc.Cleaner`使用的是`虚引用`, 当DirectByteBuffer被回收的时候，其关联的堆外内存也会使用`Unsafe`释放掉
虽然在DireactByteBuffer堆内占用内存少，但是可能关联一块非常大的堆外内存，和冰山一样，所以称为冰山对象

后面还会对DirectByteBuffer进行解析，这个是NIO的一个重要feature之一

# 3. Selector

`Selector`是NIO中用来实现`事件分发`的组件,受`AWT线程`的启发，用于接收I/O事件并分发到合适的处理器。

`Selector`底层使用的依然是操作系统的`select`,`poll`和`epoll`系统调用，支持使用一个线程来监听多个fd的I/O事件, 也即前面讲的`I/O多路复用`模型.

`Selector`可以同时监控多个`SelectableChannel`的IO状况，是非阻塞IO的核心，一个Selector 有三个SelectionKey集合

- 所有的SelectionKey集合，代表了注册在该Selector上的Channel
- 被选择的SelectionKey集合:代表了所有可以通过select 方法获取的，需要进行IO处理的Channel
- 被取消的SelectionKey集合：代表了所有被取消注册关系的Channel,在下次执行select方法时。这些 Channel对应的SelectKey会被彻底删除

`SelectableChannel`代表可以支持非阻塞IO操作的Channel对象，它可以被注册到Selector上， 这种注册关系由`SelectionKey`实例表示

下面举个聊天室的例子

```
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

/**
 * 使用NIO来实现聊天室
 */
public class NServer {

    // 用于检测所有Channel状态的selector
    private Selector selector = null;

    // 定义实现编码，解码的字符集对象
    private Charset charset = StandardCharsets.UTF_8;

    public void init() throws Exception {
        selector = Selector.open();
        //通过open方法来打开一个未绑定的ServerSocketChannel实例
        ServerSocketChannel server = ServerSocketChannel.open();
        InetSocketAddress isa = new InetSocketAddress("127.0.0.1", 8888);
        // 绑定到指定地址
        server.bind(isa);
        // 设置以非阻塞的方式工作
        server.configureBlocking(false);
        // 将Server注册到指定的Selector对象
        server.register(selector, SelectionKey.OP_ACCEPT);

        while (selector.select() > 0) {
            // 依次处理selector上的已选择的SelectionKey
            for (SelectionKey sk : selector.selectedKeys()) {
                //从selector上的已选择key集中删除正在处理的SelectionKey
                selector.selectedKeys().remove(sk);
                //如果sk对应的Channel包含客户端的连接请求
                if (sk.isAcceptable()) {
                    //调用accept方法接受此连接,产生服务器端的SocketChannel
                    SocketChannel accept = server.accept();
                    //采用非阻塞模式
                    accept.configureBlocking(false);
                    //将该SocketChannel注册到selector
                    accept.register(selector, SelectionKey.OP_READ);
                    //将sk对应的Channel设置成准备接受其他请求
                    sk.interestOps(SelectionKey.OP_ACCEPT);

                }

                // 如果sk对应的Channel有数据需要读取
                if (sk.isReadable()) {
                    // 获取该SelctionKey对应的Channel,该Channel有可读的数据
                    SocketChannel channel = (SocketChannel) sk.channel();
                    //定义准备执行读取数据的ByteBuffer
                    ByteBuffer buffer = ByteBuffer.allocate(1024);

                    String content = "";
                    //开始读取数据
                    try {
                        while (channel.read(buffer) > 0) {
                            buffer.flip();
                            content += charset.decode(buffer);
                        }
                        //打印从该SK对应的Channel读取到的数据
                        System.out.println("读取的数据" + content);
                        //将sk对应的channel设置成准备下一次读取
                        sk.interestOps(SelectionKey.OP_READ);
                    } catch (Exception e) {
                        //如果捕获到了该SK对应的Channel出现了异常，即表明
                        //该Channel对应的Client出现了问题，所以从selctor中取消该Sk的注册

                        sk.cancel();
                        if (sk.channel() != null) {
                            sk.channel().close();
                        }

                    }
                    //如果content的长度大于0,即聊天信息不为空，
                    if (content.length() > 0) {
                        //遍历该selecor里注册的所有SelectionKey
                        for (SelectionKey key : selector.keys()) {
                            //获取该key对应的channel
                            SelectableChannel target = key.channel();
                            //如果该Channel是SocketChannel对象
                            if (target instanceof SocketChannel) {
                                // 将读取到的内容写入该channel中
                                SocketChannel dest = (SocketChannel) target;
                                dest.write(charset.encode(content));
                            }
                        }
                    }

                }
            }
        }

    }

    public static void main(String[] args) {
        try {
            new NServer().init();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

使用`nc localhost 8888`就可以测试了，多开几个终端以模拟多个客户端。

这个例子里使用了`ServerSocketChannel`，类似于BIO中`ServerSocket`,用于监听某个地址和端口, 是TCP服务端的代表.同时还可以看到accept之后得到了一个`SocketChannel`, 代表一个TCP socket通道.

我们均使用了非阻塞模式, 在read的时候如果读取的数据不够，也不会阻塞调用线程。

# 4. 总结

本节介绍了NIO的核心Channel, Buffer和Selector，它们的设计意图和解决的问题,同时举了些简单的例子来说明用法。

NIO的根本还是I/O多路复用, 操作系统告诉你哪个fd可读可写，内核帮你做了`Event Loop`,比在应用层用户空间做无疑是提升了太多的
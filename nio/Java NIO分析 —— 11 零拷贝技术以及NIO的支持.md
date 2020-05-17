前面已经讲了`Selector`,`SocketChannel`和`DirectBuffer`, 这些是NIO网络编程中最核心的组件
接下来我们会再讲几点非核心的优化(不代表不重要, 只是API不占NIO设计的大头):

- 文件传输(File Transfer): 文件内容直接发送到网卡, 或者从网卡直接读到文件里
- 内存映射文件(Memory-mapped Files): 将文件的一块映射到内存

这两项本质上都基于`零拷贝(zero copy)`技术。

## 1.1 简介

> 零拷贝(Zero-Copy)是指计算机在执行操作时，CPU不需要先将数据从某处内存复制到一个特定区域，从而节省CPU时钟周期和内存带宽 —-维基百科

拿常用的**网络文件传输**过程举个栗子:

1. [DMA](https://zh.wikipedia.org/wiki/%E7%9B%B4%E6%8E%A5%E8%A8%98%E6%86%B6%E9%AB%94%E5%AD%98%E5%8F%96)`read`读取磁盘文件内容到内核缓冲区
2. copy内核缓冲区数据到应用进程缓冲区
3. 从应用进程缓冲区copy数据到socket缓冲区
4. `DMA copy`给网卡发送

画个图:

![img](http://img.sound2gd.wang/2018/07/24/20180724225945.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

可以清楚得看到，有2次copy是没必要的, 就是上面的2和3，还会平白增加2次**用户态和内核态上下文切换**, 在高并发场景下，这些会很致命。

## 1.2 Zero-Copy分类

解决上面这个问题有几个思路

1. 直接I/O: 应用进程直接操作硬件存储
2. 避免在用户空间和内核空间地址之间拷贝数据
3. 优化`页缓存`和`应用进程缓冲区`的传输

1和2都是**避免应用程序地址空间和内核地址空间两者之间的缓冲区拷贝**, 3是从传输的角度优化，因为DMA进行数据传输基本不需要CPU参与，但是用户地址空间的缓冲区和内核的`页缓存`传输没有类似DMA的手段, 3就是从这个角度优化。

## 1.3 Linux的解决方案

`直接I/O`和`传输优化`都涉及到硬件层面我们暂且不讲，主要讲`避免上下文切换和数据来回拷贝`这个思路, Linux内核提供了

- mmap: 内存映射文件, 即将文件的一段直接映射到内存，内核和应用进程共用一块内存地址，这样就不需要拷贝了
- sendfile: 从上图的内核缓冲区直接复制到socket缓冲区, 不需要向应用进程缓冲区拷贝

如图，`mmap`将buffer映射到了用户空间，操作的是同一块内存，也不需要切换了, 但是`mmap`有个缺点就是, 如果其他进程在向这个文件`write`, 那么会被认为是一个错误的存储访问

![img](http://img.sound2gd.wang/2018/07/24/20180724234117.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

而`sendfile`则**没有映射**, 保留了`mmap`的**不需要来回拷贝**优点，适用于应用进程不需要对读取的数据做任何处理的场景，如图:

![img](http://img.sound2gd.wang/2018/07/24/20180724234610.png?watermark/2/text/U291bmQyZ2TnmoTljZrlrqIK/font/5Lu_5a6L/fontsize/320/fill/IzEzMjRFQg==/dissolve/60/gravity/SouthEast/dx/0/dy/-10)

2.6以后还提供了`splice`, splice可以在内核态将数据整块的从A复制到B地址。

# 2. NIO中的零拷贝

NIO中通过`FileChannel`来提供`Zero-Copy`的支持，分别是

- FileChannel.map: 将文件的一部分映射到内存
- FileChannel.transferTo: 将本Channel的文件字节转移到指定的可写Channel

`FileChannel.map`的基本用法如下:

```
 * 测试FileChannel的用法
 * 
 * @author sound2gd
 *
 */
public class  {

	public static void main(String[] args) {
		File file = new File("src/com/cris/chapter15/f6/FileChannnelTest.java");
		try (
				// FileInputStream打开的FileChannel只能读取
				FileChannel fc = new FileInputStream(file).getChannel();
				// FileOutputStream打开的FileChannel只能写入
				FileChannel fo = new FileOutputStream("src/com/cris/chapter15/f6/a.txt").getChannel();) {

			// 将FileChannel的数据全部映射成ByteBuffer
			MappedByteBuffer mbb = fc.map(MapMode.READ_ONLY, 0, file.length());
			// 使用UTF-8的字符集来创建解码器
			Charset charset = Charset.forName("UTF-8");
			// 直接将buffer里的数据全部输出
			fo.write(mbb);
			mbb.clear();
			// 创建解码器
			CharsetDecoder decoder = charset.newDecoder();
			// 使用解码器将byteBuffer转换为CharBuffer
			CharBuffer decode = decoder.decode(mbb);
			System.out.println(decode);
		} catch (Exception e) {

		}
	}

}
```

这就是一个基本的例子，用于文件复制，可以看到`fo.write(mbb)`的时候，是将`mbb Buffer`的数据输出到另一个文件的，看起来就像是
在内存中，而不是在文件里, 这就是**内存映射文件**.

我们来看看map的实现

```
public MappedByteBuffer map(MapMode mode, long position, long size)
  ...省略非关键代码
      try {
          // 调用map0这个native方法
          addr = map0(imode, mapPosition, mapSize);
      } catch (OutOfMemoryError x) {
          // An OutOfMemoryError may indicate that we've exhausted memory
          // so force gc and re-attempt map
          // gc下防止内存不够
          System.gc();
          try {
              // 等待gc结束
              Thread.sleep(100);
          } catch (InterruptedException y) {
              Thread.currentThread().interrupt();
          }
          try {
              // 再试一次
              addr = map0(imode, mapPosition, mapSize);
          } catch (OutOfMemoryError y) {
              // After a second OOME, fail
              throw new IOException("Map failed", y);
          }
      }
  ...
  }
  
  private native long map0(int prot, long position, long length)
  throws IOException;
```

打开`FileChannelImpl.c`

```
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                                     jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    jint fd = fdval(env, fdo);
    int protections = 0;
    int flags = 0;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        protections = PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        protections = PROT_WRITE | PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        protections =  PROT_WRITE | PROT_READ;
        flags = MAP_PRIVATE;
    }

    // 所以还是使用的mmap这个API
    mapAddress = mmap64(
        0,                    /* Let OS decide location */
        len,                  /* Number of bytes to map */
        protections,          /* File permissions */
        flags,                /* Changes are shared */
        fd,                   /* File descriptor of mapped file */
        off);                 /* Offset into file */

    if (mapAddress == MAP_FAILED) {
        if (errno == ENOMEM) {
            JNU_ThrowOutOfMemoryError(env, "Map failed");
            return IOS_THROWN;
        }
        return handle(env, -1, "Map failed");
    }

    return ((jlong) (unsigned long) mapAddress);
}
```

可以看到，还是使用的我们`mmap`的api, 了解一些底层知识还是有必要的, JVM很多东西都是对底层的一层封装.

另一个API `transferTo`同理，最后调用的是`transferTo0`方法:

```
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_transferTo0(JNIEnv *env, jobject this,
                                            jint srcFD,
                                            jlong position, jlong count,
                                            jint dstFD)
{
    off64_t offset = (off64_t)position;
    // 调用sendfile方法
    jlong n = sendfile64(dstFD, srcFD, &offset, (size_t)count);
    if (n < 0) {
        if (errno == EAGAIN)
            return IOS_UNAVAILABLE;
        if ((errno == EINVAL) && ((ssize_t)count >= 0))
            return IOS_UNSUPPORTED_CASE;
        if (errno == EINTR) {
            return IOS_INTERRUPTED;
        }
        JNU_ThrowIOExceptionWithLastError(env, "Transfer failed");
        return IOS_THROWN;
    }
    return n;
}
```

可以看到封装的是`sendfile`这个方法，这里看的是jvm在linux系统的的实现。

# 3. 总结

本文主要介绍了Linux中`Zero-Copy零拷贝`的概念，分类和解决方案。

同时介绍了NIO对`Zero-Copy`的支持, 分别是`FileChannel.map`以及`FileChannel.transferTo`.

在高并发场景下，这点提升是很关键的，著名框架`Netty`, `Kafka`都大量使用了零拷贝的API, 是其高性能的原因之一。

------

参考资料

1. [Zero Copy I: User-Mode Perspective](https://www.linuxjournal.com/article/6345)
2. [wikipedia: zero copy](https://en.wikipedia.org/wiki/Zero-copy)
3. [Linux 中的零拷贝技术，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)
[TOC]

## 1 InputStream 子类结构

- 下图是 InputStream 及其子类的结构图。可以看出 InputStream 是一个典型的装饰者模式的示例
- InputStream：抽象的组件类，具体的组件类继承它，来实现不同的输入流功能 
- FilterInputStream：抽象装饰者类，具体装饰者类继承它，来对它包装的组件类，进行扩展 
- Java IO 的扩展：
  - 添加具体组件类，实现特定的输入流功能
  - 添加具体装饰类，达到保证组件类的目的

![在这里插入图片描述](.\img\inputStream 类图.png)

### 1.1 InputStream 源码分析

- inputStream抽象类功能
  - 定义读取一个字节 或 多个字节的 方法
  - 定义了 mark、reset 的方法，由具体子类实现 重复读取一段数据流的功能
  - 定义了关闭输入流的方法
  - 定义了返回剩余可读取字节的方法
  - **注：** 只有读取一个字节的 read() 方法是抽象的，也就是说，具体类只要实现这一个方法即可，其它的可不实现

```java
public abstract class InputStream implements Closeable {

    //返回下一个字节的 int 表示，如果输入流已经结束了，则返回 -1。此方法会一直阻塞到有值返回，或抛异常
	public abstract int read() throws IOException;
    //从输入流中读取一些字节放入 字节数组 b 中，返回值即是读取的真实字节数。此方法会一直阻塞到有值返回，或抛异常
    //如果 字节数组 b 的长度为 0,则不会读取任何字节，并且返回 0。如果输入流已经结束则返回 -1
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }
    //和 read(byte b[]) 方法的区别仅仅是，读出来的数据放入 字节数组的位置不一样。此方法是从 字节数组的 off 索引处开始存放字节。返回值即是读取的真实字节数
    //如果 字节数组 b 的长度为 0,则不会读取任何字节，并且返回 0。如果输入流已经结束则返回 -1
    public int read(byte b[], int off, int len) throws IOException {...}

    //丢弃输入流中的一些字节，丢弃的个数为返回值。如果 n<=0，则不会操作流。
	public long skip(long n) throws IOException {...}
    //返回输入流可以读取的字节数（不准确），由子类实现
    public int available() throws IOException {
        return 0;
    }
    //关闭输入流，并且释放和流相关的系统资源
    public void close() throws IOException {}
    //标记当前输入流的位置，这样之后调用 reset() 方法时，将会重新将流设置到该标记的位置。实现重复读取
    //如果 调用 mark() 方法后 又读取了超过 readlimit 字节，mark 将会失效
    public synchronized void mark(int readlimit) {}

    //重新将流设置到 mark() 方法标记的位置，如果 mark() 已经失效则报错
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }
    //该输入流是否可以被 mark
    public boolean markSupported() {
        return false;
    }
    //skip 方法最多跳过的字节数，如果 skip 的参数值大于此值，则实际使用的是此值
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;
}
```

## 2 InputStream 具体实现类源码分析

### 2.1 ByteArrayInputStream —— 字节输入流

- **核心：** buf[] 数组，这个数组即是数据流的 源
  - buf[] 数组，通过构造函数传入！
- 字节流操作即是，操作数组的位置 pos。思想很简单
- 基本所有操作都被 synchronized 修饰

```java
public class ByteArrayInputStream extends InputStream {
    protected byte buf[];//输入流的源
    protected int pos;//输入流当前位置
    protected int mark = 0;//标记的位置
    protected int count;//流的最大位置
    //返回当前位置的字节，并 pos 加 1
    public synchronized int read() {
        return (pos < count) ? (buf[pos++] & 0xff) : -1;
    }
    //将流中的自己赋值到 字节数组 b 中，并更新流的位置
    public synchronized int read(byte b[], int off, int len) {...}
    //跳过 n 个字节
    public synchronized long skip(long n) {}
    //返回还有多少字节(准确)
    public synchronized int available() {}
    public void mark(int readAheadLimit) {
        mark = pos;//标记当前流位置，以便后面的 reset
    }
    public synchronized void reset() {
        pos = mark;//重置流到 mark 位置
    }
    //因为 ByteArrayInputStream 流对其它资源无任何影响，所以不用任何操作
    public void close() throws IOException {}
    
    //用 buf[] 数组初始化流
    public ByteArrayInputStream(byte buf[]) {
        this.buf = buf;
        this.pos = 0;
        this.count = buf.length;
    }
    //用 buf[] 数组的 offset-length 阶段初始化流
    public ByteArrayInputStream(byte buf[], int offset, int length) {
        this.buf = buf;
        this.pos = offset;
        this.count = Math.min(offset + length, buf.length);
        this.mark = offset;
    }
}
```

### 2.2 FileInputStream —— 文件输入流 

- **核心：** file、fileDescriptor
  - 通过构造函数 传入 fileName 或 File 类，内部创建一个 文件链接
- 核心功能实现方法（本地方法）：
  - read0()：获取下一个字节
  - readBytes(byte b[], int off, int len)：从偏移量 off 开始，读取 len 个字节，放入 b[] 数组中
  - skip0(long n)：跳过 n 个字节
  - available0()：还剩多少字节
  - close0()：关闭文件流
- 方法没有被 synchronized 修饰，因为最终调用的是本地方法，本地方法应该加了手段

```java
public class FileInputStream extends InputStream {
    
    /* File Descriptor - handle to the open file */
    private final FileDescriptor fd;
    // 文件的路径
    private final String path;
    //适配 java nio
    private FileChannel channel = null;

    private final Object closeLock = new Object();//保证只有一个线程调用 close 方法
    private volatile boolean closed = false;// double check 用途
    
    //通过打开一个与实际文件的连接，来创建一个 文件流。同时会创建一个 fileDescriptor 来标识这个连接。如果 file 不存在或是一个文件夹，抛异常。
    //如果存在 security manager 则会调用 security.checkRead(name) 方法验证
    public FileInputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null);
    }
    //上面的构造方法调用的就是这个，内部会调用 open() 方法
    public FileInputStream(File file) throws FileNotFoundException {...}
    //通过使用一个 fileDescriptor 来关联一个已经存在的数据流。如果这个 fileDescriptor 无效，创建的时候不会报错，当获取 流 的时候就会报 IOException
    public FileInputStream(FileDescriptor fdObj) {...}
    //打开文件流
    private void open(String name) throws FileNotFoundException {
        open0(name);//native 方法
    }
    //从文件流中读取一个字节，如果还没有流准备好就会阻塞。如果流结束了则返回 -1
    public int read() throws IOException {
        return read0();
    }
    //读取至多 len 字节数到 b 数组中，同 inputstream 方法
    private native int readBytes(byte b[], int off, int len) throws IOException;
    //读取字节到 b 数组中，返回读取的字节数
    public int read(byte b[]) throws IOException {
        return readBytes(b, 0, b.length);
    }
    //读取到的字节放到 b 数组的 off 位置往后，最多读取 len 字节
    public int read(byte b[], int off, int len) throws IOException {
        return readBytes(b, off, len);
    }
    //同 inputstream 方法
    public long skip(long n) throws IOException {
        return skip0(n);// native 方法
    }
    //同 inputstream 方法
    public int available() throws IOException {
        return available0();//native 方法
    }
    //关闭流相关资源
    public void close() throws IOException {
        //保证一个流只会 close 一次
        synchronized (closeLock) {
            if (closed) {
                return;
            }
            closed = true;
        }
        if (channel != null) {
           channel.close();//关闭 channel
        }
        fd.closeAll(new Closeable() {
            public void close() throws IOException {
               close0();//native 方法关闭流
           }
        });
    }
    //获取 fileDescriptor
    public final FileDescriptor getFD() throws IOException {...}
    //获取该 文件流 对应的 channel
    public FileChannel getChannel() {
        synchronized (this) {
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, true, false, this);
            }
            return channel;
        }
    }
    //最后的防护机制，防止用户忘记关闭流
    protected void finalize() throws IOException {
        if ((fd != null) &&  (fd != FileDescriptor.in)) {
            close();
        }
    }
}
```
### 2.3 PipedInputStream —— 管道输入流

- 管道输入流，必须**配合管道输出流** (通过构造函数 传入 一个 管道出入流)
- 核心参数：
  - buffer[]：输入流的存储位置，只有输出流，才能向此 buffer 中放入字节
  - in：可输入的位置，-1表示输入流没启动，输出流 调用 receive() 方法时给 buffer[] 添加值，并 in++
  - out：可读取的位置，从输入流 执行 read() 方法时，读取 buffer[out]， 并out++
- 核心方法：
  - receive()：由输出流调用，以此来将字节流传入输入流(buffer[] 数组中)
  - read()：有输入流调用，获取输入流的字节

```java
public class PipedInputStream extends InputStream {
    protected byte buffer[];//输入流的存储位置，只有输出流，才能向此 buffer 中放入字节！！
    protected int in = -1;//可输入的位置，-1表示输入流没启动，输出流 调用 receive() 方法时给 buffer[] 添加值，并 in++
    protected int out = 0;//可读取的位置，从输入流 执行 read() 方法时，读取 buffer[out]， 并out++
    //一个输入流的参数肯定是一个输出流
    public PipedInputStream(PipedOutputStream src) throws IOException {
        this(src, DEFAULT_PIPE_SIZE);//默认的 buffer 为 1024 大小
    }
    //设置 buffer 大小
    public PipedInputStream(PipedOutputStream src, int pipeSize)
            throws IOException {
         initPipe(pipeSize);
         connect(src);
    }
    //连接到 src 输出流
    public void connect(PipedOutputStream src) throws IOException {
        //即，将 this(PipedInputStream 传入 输出流中，这样输出流，通过 write() 方法，写入字节到，输入流中(调用输入流的receive方法))
        src.connect(this);
    }
    //这个是 PipedOutputStream 中的方法
    public synchronized void connect(PipedInputStream snk) throws IOException {
        if (snk == null) {
            throw new NullPointerException();
        } else if (sink != null || snk.connected) {
            throw new IOException("Already connected");
        }
        sink = snk;
        snk.in = -1;
        snk.out = 0;
        snk.connected = true;
    }
    //接收一个 b 自己到 输入流中，in++。会阻塞（如生产者消费者一样）
    protected synchronized void receive(int b) throws IOException {}
    //接收 b 数组的 off 到 off+len 字节，放入 buffer 中，in+len。会阻塞（如生产者消费者一样）
    synchronized void receive(byte b[], int off, int len)  throws IOException {}
    //读取一个字节，out++
    public synchronized int read()  throws IOException {}
    //读取字节到 b 数组中，out+len。最多读取 len 个
    public synchronized int read(byte b[], int off, int len)  throws IOException {}
    public void close()  throws IOException {
        closedByReader = true;
        synchronized (this) {
            in = -1;
        }
    }
}
```

### 2.4 ObjectInputStream —— Java 反序列化输入流 

- 用于反序列化作用的流，其实相当于 装饰者类，提供反序列化功能

```java
// ObjectInputStream 用来反序列化，传入的 inputStream 流！
// 只有 实现 Serializable 或 Externalizable 的类可以被 ObjectInputStream 反序列化
// 使用 readObject() 方法，读取一个 object（如 String、arrays）
// 原始类型，可以通过 DataInput 中的方法来读取
// 对象的默认反序列化机制将每个字段的内容还原为其在写入时的值和类型。反序列化过程将忽略声明为 transient 或 static 的字段。对其他对象的引用会根据需要从流中读取这些对象。反序列化时总是分配新对象，这样可以防止覆盖现有对象。
//读取对象类似于运行新对象的构造函数。为对象分配内存并初始化为零（空）
/*
    FileInputStream fis = new FileInputStream("t.tmp");
    ObjectInputStream ois = new ObjectInputStream(fis);

    int i = ois.readInt();
    String today = (String) ois.readObject();
    Date date = (Date) ois.readObject();
    //因为 ObjectInputStream 只是 FileInputStream 包装类，关闭它实际上关闭的就是具体的 FileInputStream 数据流
    ois.close();
*/
//如果在序列化或反序列化的时候执行一些特殊的逻辑，可以在实现类中重写 writeObject、readObject、readObjectNoData 方法
//当一个父类实现序列化，子类自动实现序列化，不需要显式实现Serializable接口
//当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化
public class ObjectInputStream extends InputStream {
    //存储传入的 输入流（如 fileinputstream）
    private final BlockDataInputStream bin;
    //接收一个 输入流
    public ObjectInputStream(InputStream in) throws IOException {}
    //反序列化一个对象
    public final Object readObject() throws IOException, ClassNotFoundException {}
    //和 readObject() 方法功能相同，只是当调用此方法返回一个对象后，如果后面有返回一个同样的对象，就会报错（即使用 readUnshared() 返回的对象必须在流中是唯一的）
    public Object readUnshared() throws IOException, ClassNotFoundException {}
    //从流中读取持久字段并按名称使其可用
    public ObjectInputStream.GetField readFields() {}
    //以下是反序列化基本类型数据 ---------------
    public int read() throws IOException {}
    public int read(byte[] buf, int off, int len) throws IOException {}
    public boolean readBoolean() throws IOException {}
    public byte readByte() throws IOException  {}
    public int readUnsignedByte()  throws IOException {}
    public char readChar()  throws IOException {}
    public short readShort()  throws IOException {}
    public int readUnsignedShort() throws IOException {}
    public int readInt()  throws IOException {}
    public long readLong()  throws IOException {}
    public float readFloat() throws IOException {}
    public double readDouble() throws IOException {}
    //直接在输入流中读取字节到 数组 b 中
    public void readFully(byte[] buf) throws IOException {}
    public void readFully(byte[] buf, int off, int len) throws IOException {}
    //读取 UTF 对应的字节作为字符
    public String readUTF() throws IOException {}
}
```

#### 2.4.1 BlockDataInputStream

- 是 ObjectInputStream 的内部类

- 具有两种模式的输入流
  - 在“默认”模式下，以与dataoutputstream相同的格式输入数据
  - 在“块数据”模式下，输入流以块的形式被缓存。当处于默认模式时，不预先缓冲数据；当处于块数据模式时，立即读取当前数据块的所有数据（并缓冲）。
- 块模式就相当于给输入流加了一层缓冲区

```java
private class BlockDataInputStream extends InputStream implements DataInput {
    /** block data mode */
    private boolean blkmode = false;//默认不是块数据模式，需要设置 setBlockDataMode()
	//包装输入流，达到预期功能作用
    BlockDataInputStream(InputStream in) {
        this.in = new PeekInputStream(in);
        din = new DataInputStream(this);//装饰者组件
    }
    //获取字节流，但不消耗！read() 方法消耗数据流！
    int peek() throws IOException {
        if (blkmode) {
            if (pos == end) {
                refill();
            }
            return (end >= 0) ? (buf[pos] & 0xFF) : -1;
        } else {
            return in.peek();
        }
    }
    //当块缓存区消耗完了后，就调用此方法，读取输入流中数据到缓存区中
    private void refill() throws IOException {...}
    
    public float readFloat() throws IOException {
        if (!blkmode) {
            pos = 0;//如果不是块模式则去 peekInputStream 中读取流，放入 buf 中
            in.readFully(buf, 0, 4);
        } else if (end - pos < 4) {
            return din.readFloat();//如果是块模式，但是缓存区字节不够
        }
        float v = Bits.getFloat(buf, pos);
        pos += 4;
        return v;
    }
}
```
#### 2.4.2 PeekInputStream

- 是 ObjectInputStream 的内部类
- 提供了获取字节流但不消耗字节流的功能

- 提供了统计获取字节数的功能

```java
private static class PeekInputStream extends InputStream {
    //获取字节流但不消耗字节流！下一次 read() 操作又会将 peekb 赋值为 -1！
    int peek() throws IOException {
        if (peekb >= 0) {
            return peekb;
        }
        peekb = in.read();
        totalBytesRead += peekb >= 0 ? 1 : 0;
        return peekb;
    }
}
```
### 2.5 SocketInputStream —— 网络输入流

- socket 网络连接使用的 输入流
- 核心方法：
  - **socketRead0()：**本地方法，得到网络输入流字节，读到 -1 时，表示流结束
  - read()：内部调用 socketRead0() 方法

```java
class SocketInputStream extends FileInputStream {
    SocketInputStream(AbstractPlainSocketImpl impl) throws IOException {
        //通过 AbstractPlainSocketImpl 得到 fileD 来打开（标志）一个文件流，并通过这个关闭输入流，和得到其它一些参数（如，timeout）
        super(impl.getFileDescriptor());
        this.impl = impl;
        //通过 AbstractPlainSocketImpl 得到 一个 socket，用来关闭输入流
        socket = impl.getSocket();
    }
    private int socketRead(FileDescriptor fd,byte b[], int off, int len,int timeout) {
        //读取 socket 流到 b 字节数组中，超时时间为 timeout，偏移为 off，最大长度为 len
        return socketRead0(fd, b, off, len, timeout);
    }
    //Reads into a byte array data from the socket.如果流结束了，则返回 -1
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }
    public void close() throws IOException {
        // Prevent recursion. See BugId 4484411
        if (closing)
            return;
        closing = true;
        if (socket != null) {
            if (!socket.isClosed())
                socket.close();//关闭 socket 
        } else
            impl.close();//如果 socket 为 null 还需要关闭 impl
        closing = false;
    }
}
```
## 3 Inputstream 装饰者类源码分析

### 3.1 BufferedInputStream —— 缓冲输入流

- 通过构造函数，包装一个 输入流，对这个输入流做缓冲，同时实现 mark 和 reset 的功能
- 核心字段
  - buf[]：用来缓冲输入流的字节数组
  - count：buf 字节数组能读取到的最大下标
  - pos：目前读取 buf[] 的索引
  - markpos ：调用 reset 后，pos 会被设为 markpos
  - marklimit：每次 mark 的时候，设置一个值，超过这值时，mark 就失效了(markpos 设为 -1)
- 核心方法：
  - fill()：用来填充 buf[]，当缓冲不够时
  - read()：从缓冲中读取一个字节
  - read1(b,off ,len)：从缓冲中的 off 偏移，读 len 字节到 b 字节数组中

```java
//bufferedInputStream 通过包装 具体 inputstream ，使用一个 buffer 字节数组，实现了输入流缓存功能，同时实现了 mark 和 reset 的功能
public class BufferedInputStream extends FilterInputStream {
    protected volatile byte buf[];
    protected int count;//buf 字节数组能读取到的最大下标
    protected int pos;//目前读取 buf[] 的索引
    protected int markpos = -1;//调用 reset 后，pos 会被设为 markpos
    protected int marklimit;//每次 mark 的时候，设置一个值，超过这值时，mark 就失效了(markpos 设为 -1)
    //包装 inputstream，并设置 buffer 字节数组大小
    public BufferedInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }
    public synchronized void mark(int readlimit) {
        marklimit = readlimit;
        markpos = pos;
    }
    public synchronized void reset() throws IOException {
        getBufIfOpen(); // Cause exception if closed
        if (markpos < 0)
            throw new IOException("Resetting to invalid mark");
        pos = markpos;
    }
    //从 buffer 数组中读取下一个字节
    public synchronized int read() throws IOException {
        if (pos >= count) {
            fill();
            if (pos >= count)
                return -1;
        }
        return getBufIfOpen()[pos++] & 0xff;
    }
    //重新从输入流中读取一定字节存储到 buffer 数组中，并更新 pos 和 count！
    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos < 0)
            pos = 0;//重置 pos   
        else {//复杂情况暂不考虑
        }
        count = pos;
        //从输入流中读取数据到 buffer 中，返回读取到的数据
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
            count = n + pos;//count 索引就是 buffer 数组有效字节的最大位置
    }
}
```
### 3.2 DataInputStream —— 基本类型反序列化输入流

- 通过构造函数，传入输入流，提供对输入流进行基本类型反序列化功能

- 完全可以配合 BufferInputStream

- 核心方法：

  - readFully(b, off, len)：从输入流中读取字节，一直循环读取到 len 字节才返回，如果一直到没有字节了还没读满，则报错
  - readBoolean()：读取下一个字节，如果不是 0，则返回 true，否则返回 false
  - readLine()：此方法不能正确的将字节转为字符，被 BufferedReader 取代

  ```java
  BufferedReader reader = new BufferedReader(new InputStreamReader(in));
  reader.readLine();
  ```

  - readUTF()：获取下一个 UTF-8 的字符

```java
public final int readInt() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    int ch3 = in.read();
    int ch4 = in.read();
    if ((ch1 | ch2 | ch3 | ch4) < 0)
        throw new EOFException();
    return ((ch1 << 24) + (ch2 << 16) + (ch3 << 8) + (ch4 << 0));
}
public final char readChar() throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    if ((ch1 | ch2) < 0)
        throw new EOFException();
    return (char)((ch1 << 8) + (ch2 << 0));
}
public final byte readByte() throws IOException {
    int ch = in.read();
    if (ch < 0)
        throw new EOFException();
    return (byte)(ch);
}
```

### 3.3 GZIPInputStream —— 读取GZIP 格式输入流

- 内部原理留待以后分析
- 估计是 readTrailer() 方法起主要作用

```java
public class GZIPInputStream extends InflaterInputStream {
    /**
     * CRC-32 for uncompressed data.
     */
    protected CRC32 crc = new CRC32();

    /**
     * Indicates end of input stream.
     */
    protected boolean eos;

    private boolean closed = false;
    public GZIPInputStream(InputStream in, int size) throws IOException {
        super(in, new Inflater(true), size);
        usesDefaultInflater = true;
        readHeader(in);
    }
    public int read(byte[] buf, int off, int len) throws IOException {
        ensureOpen();
        if (eos) {
            return -1;
        }
        int n = super.read(buf, off, len);
        if (n == -1) {
            //估计是 readTrailer() 方法起主要作用
            if (readTrailer())
                eos = true;
            else
                return this.read(buf, off, len);
        } else {
            crc.update(buf, off, n);
        }
        return n;
    }
}
```

### 3.4 ZIPInputStream —— 读取ZIP 格式输入流

- 内部原理留待以后分析
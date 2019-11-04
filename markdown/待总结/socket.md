#### 2.5.0 Socket  把 2.5.0 2.5.1 移到 socket 文档中，统一介绍

#### 2.5.1 AbstractPlainSocketImpl

- connect()：创建连接
- bind()：监听连接

```java
abstract class AbstractPlainSocketImpl extends SocketImpl {
    /* instance variable for SO_TIMEOUT */
    int timeout;   // timeout in millisec
    // traffic class
    private int trafficClass;
    
    //创建一个 socket,并且连接到给定的 地址和端口
    protected void connect(InetAddress address, int port) throws IOException {
        this.port = port;
        this.address = address;

        try {
            connectToAddress(address, port, timeout);
            return;
        } catch (IOException e) {
            // everything failed
            close();
            throw e;
        }
    }
    private void connectToAddress(InetAddress address, int port, int timeout) {
        if (address.isAnyLocalAddress()) {
            //连接给定地址与端口
            doConnect(InetAddress.getLocalHost(), port, timeout);
        } else {
            doConnect(address, port, timeout);
        }
    }
    
    //绑定 socket 到给定地址的给定端口，用于监听连接！！
    protected synchronized void bind(InetAddress address, int lport) {
       synchronized (fdLock) {
            if (!closePending && (socket == null || !socket.isBound())) {
                NetHooks.beforeTcpBind(fd, address, lport);
            }
        }
        //绑定 socket 到给定地址的给定端口，用于监听连接！！
        socketBind(address, lport);
        if (socket != null)
            socket.setBound();
        if (serverSocket != null)
            serverSocket.setBound();
    }
    //设置 backlog 值
    protected synchronized void listen(int count) throws IOException {}

    //接收连接，s 用于回调，接收到连接后，将连接信息设置到 s 中
    protected void accept(SocketImpl s) throws IOException {
        acquireFD();
        try {
            socketAccept(s);
        } finally {
            releaseFD();
        }
    }
    void socketClose0(boolean useDeferredClose/*unused*/) throws IOException {
        if (fd == null)
            throw new SocketException("Socket closed");
        if (!fd.valid())
            return;
        final int nativefd = fdAccess.get(fd);//获取 fd 的 数字标志
        fdAccess.set(fd, -1);
        close0(nativefd);//通过 nativefd 关闭 socket 连接
    }
    //设置连接的一些属性：SO_TIMEOUT、TCP_NODELAY、SO_KEEPALIVE 等
    public void setOption(int opt, Object val) throws SocketException {...}
    //创建 socket（server或client）
    protected synchronized void create(boolean stream) throws IOException {
        this.stream = stream;
        if (!stream) {
            ResourceManager.beforeUdpCreate();
            // only create the fd after we know we will be able to create the socket
            fd = new FileDescriptor();
            try {
                socketCreate(false);
            } catch (IOException ioe) {
                ResourceManager.afterUdpClose();
                fd = null;
                throw ioe;
            }
        } else {
            fd = new FileDescriptor();
            socketCreate(true);
        }
        if (socket != null)
            socket.setCreated();
        if (serverSocket != null)
            serverSocket.setCreated();
    }
}
```
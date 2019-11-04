#### 创建 eventLoopGroup 

```java
// 创建EventLoopGroup   accept线程组 NioEventLoop
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
//最后走到这个方法中了
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {

    if (executor == null) {// Tony: 如果执行器为空，则创建一个
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    children = new EventExecutor[nThreads];
	//开始循环创建 EventExecutor(实际执行器)
    for (int i = 0; i < nThreads; i ++) {
        // args：selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject()
        // 这里回去创建一个 nioEventLoop 事件，当有任务提交的时候，即调用 execute() 方法时，会触发
        children[i] = newChild(executor, args);
    }

    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

#### eventLoop 中提交任务时，触发的事件

```java
@Override
public void execute(Runnable task) {

    // Tony: 判断execute方法的调用者是不是EventLoop同一个线程
    boolean inEventLoop = inEventLoop();
    addTask(task);// Tony: 增加到任务队列
    if (!inEventLoop) {// Tony: 不是同一个线程，则调用启动方法,启动后，肯定时同一个线程
        //最终会调用 run 方法
        startThread();
    }
    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
//开启线程后，这个线程就会处理响应的事件了，for 循环中
protected void run() {// Tony: 有任务提交后，被触发执行
    for (;;) {// Tony: 执行两件事selector,select的事件 和 taskQueue里面的内容
        try {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                    default:
                }
            }
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {// Tony: 处理事件
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        }
    }
}
```

- 到目前为止，我们已经直到了 NioEventLoop 的运作方式了，这时，我们已经定义了两组 EvenLoop（一组用来 执行 accept，一组用来执行 i/o ），i/o 完了时 workPool 执行业务
- 那么 eventLoop 和 channel 和 selector 的关系呢？后面再说

#### 绑定端口前的 创建/初始化ServerSocketChannel对象，并注册到Selector

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();// Tony: 创建/初始化ServerSocketChannel对象，并注册到Selector
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }
    // Tony: 等注册完成之后，再绑定端口。 防止端口开放了，却不能处理请求
    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);// Tony: 实际操作绑定端口的代码
        return promise;
    } else {
    }
}
```

#### 初始化 Channel  和 注册 channel

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    //通过 channel() 方法传进来的 class ，实例化 channel
    channel = channelFactory.newChannel();
    //初始化 channel 类里面的属性
    init(channel);

    // Tony: （一开始初始化的group）MultithreadEventLoopGroup里面选择一个eventLoop进行绑定
    ChannelFuture regFuture = config().group().register(channel);
}

private void register0(ChannelPromise promise) {
	doRegister();// Tony: NIOchannel中，将Channel和NioEventLoop里面的Selector进行绑定
    pipeline.invokeHandlerAddedIfNeeded();
    pipeline.fireChannelRegistered();// Tony: 传播通道完成注册的事件
    
    if (isActive()) {// Tony: ServerSocketChannel服务端完成bind之后，才会变成active。
        if (firstRegistration) {// Tony: 如果是socketChannel，active的判断就是是否连接、是否开启
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();// Tony: 如果是取消register，再重新绑定的，就会直接注册到OP_READ
        }
    }
}
```

#### 实际绑定端口代码

```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {// Tony: 这里向EventLoop提交任务，一旦有任务提交则会触发EventLoop的轮询
            if (regFuture.isSuccess()) {// Tony: 本质又绕回到channel的bind方法上面。
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```
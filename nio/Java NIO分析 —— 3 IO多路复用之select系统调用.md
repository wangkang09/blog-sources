前面讲了一些[Java NIO分析(2): I/O多路复用历史杂谈](http://sound2gd.wang/2018/06/17/Java-NIO%E5%88%86%E6%9E%90-2-I-O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E5%8E%86%E5%8F%B2%E6%9D%82%E8%B0%88/), 谈到了多路复用的发展历史 以及为什么需要它。今天讲广受各大内核支持的`select`系统调用,`select`允许进程 指定内核等待1个或者多个事件的任何一个发生, 并且只在有它们发生之后或者等待一段时间后才唤醒进程。 



# 1 select介绍

使用select, 我们即使应用进程里只有1个线程也能够接受多个连接并且做出处理,
因为应用进程不需要阻塞在socket的read,write或者accept系统调用上, 而是
内核告诉应用有事件到来了, 应用进程遍历`fd_set`看是哪个`fd`就绪了，然后再处理

`fd_set`称为`描述符集`，通常是一个整数数组，其中**每个整数的每一位对应1个描述符的状态**.

select的函数签名是:

```
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfdpl, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout)
```

其中，返回值表示`fd_set`里就绪的元素总个数,包括读,写,异常`fd_set`
第一个参数表示待测试的fd个数,值一般是待测试的fd总数+1
中间仨代表要监听的读set, 写set和异常set
最后一个参数代表select每个fd经历的时间

操作`描述符集`的方法是4个宏:

- FD_ZERO 清除fdset里的所有位
- FD_SET 开启某fd在fdset里对应的位, 一般就置1
- FD_CLR 清除某fd在fdset里对应的位, 一般就置0
- FD_ISSET 判断fdset里是否包含某个fd的位

**socket描述符**的读就绪条件有:

- 该socket的读缓冲区的数据字节数 >= 读缓冲区低水位标记的当前大小
- 连接的读半关闭(FIN)
- socket是监听socket且已完成的连接数不为0
- socket有异常需要处理

**socket描述符**的写就绪条件有:

- 该socket的发送缓冲区的可用空间字节数 >= socket发送缓冲区低水位的大小
- 连接的写半关闭
- 使用非阻塞式的connect的socket已建立连接
- socket有异常要处理

通俗点说就是， 缓冲区是一个大小为n的仓库，数据是货物。
读缓冲区就类似于，你去取货，首先要货物有一定数量(读缓冲区低水位标记), 不然就拿不到货了
写缓冲区好比，你要把货物入库，仓库得还有剩余空间(写缓冲区低水位标记), 不然仓库放不下

# 2 select写server实战

下面是一个使用c写的`echo server`示例，可以直接编译运行

```
gcc -o select ./select.c
./select
```

主要有5个步骤

1. 新建socket, 用于监听端口
2. socket的fd绑定端口
3. socket监听
4. 初始化fd_set
5. 发起select系统调用获取所有fd_set，寻找和处理事件

```
#include <sys/select.h>
#include <sys/socket.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    // 1. 新建socket, 用于监听端口
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) printf("创建socket失败， error: %s (errno: %d)\n", strerror(errno), errno);

    // 2. 绑定端口
    unsigned short listenPort = 8090;
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(listenPort);

    int on = 1;
    // 设置socket绑定的端口，再程序关闭之后可以重复使用
    if ((setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(int))) < 0) {
        exit(1);
    }
    int bindRet = bind(listenfd, (struct sockaddr *) &servaddr, sizeof(servaddr));
    if (bindRet == -1) {
        printf("socket绑定地址失败， error: %s (errno: %d)\n", strerror(errno), errno);
        exit(1);
    }

    // 3. 监听端口
    int listenRet = listen(listenfd, 10);
    if (listenRet == -1) printf("socket监听端口失败， error: %s (errno: %d)\n", strerror(errno), errno);
    printf("socket 监听完毕, 地址: 127.0.0.1:%d", listenPort);

    // 4. 初始化fd_set
    int client[FD_SETSIZE]; // 保存已连接fd
    for (int i = 0; i < FD_SETSIZE; ++i) {
        client[i] = -1;
    }
    int maxfd = listenfd, i, maxi = -1, MAXLEN = 1024;
    char buf[MAXLEN];

    fd_set rset, allset; // fd_set代表的是描述符集，本质上是一个整数, 根据操作系统的不同有可能是64位或者32位，通过每位的0或者1来判断fd是否就绪
    FD_ZERO(&allset); // 清零
    FD_SET(listenfd, &allset); // 把listenfd加入到allset
    printf("ready");

    while (1) {
        rset = allset;
        // 函数签名int select(int maxfdpl, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout)
        // 返回值表示fd_set里就绪的元素总个数,包括读,写,异常fd_set
        // 第一个参数表示待测试的fd个数,值一般是待测试的fd总数+1
        // 中间仨代表要监听的读set, 写set和异常set
        // 最后一个参数代表select每个fd经历的时间
        // 5. select获取 连接fd,断开连接fd, 有数据可读fd
        int nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
        if (FD_ISSET(listenfd, &rset)) { // 接受新连接
            struct sockaddr_in clientaddr;
            int clilen = sizeof(clientaddr);
            int connfd = accept(listenfd, (struct sockaddr *) &clientaddr, &clilen);

            // 保存
            for (i = 0; i < FD_SETSIZE; ++i) {
                if (client[i] < 0) {
                    client[i] = connfd;
                    break;
                }
            }

            if (i == FD_SETSIZE) exit(1);

            FD_SET(connfd, &allset);
            if (connfd > maxfd) maxfd = connfd;
            if (i > maxi) maxi = i;
            if (--nready <= 0) continue;
        }

        for (i = 0; i <= maxi; i++) {
            int sockfd = client[i];
            if (sockfd < 0) continue;

            if (FD_ISSET(sockfd, &rset)) {
                int n = recv(sockfd, buf, MAXLEN, 0);
                if (n == 0) { // 读到0表示连接关闭
                    close(sockfd);
                    FD_CLR(sockfd, &allset);
                    client[i] = -1;
                } else write(sockfd, buf, n);

                if (--nready == 0) break;
            }
        }
    }
}
```

然后使用任何tcp测试工具都能测， 例如好用的`nc`, 多开几个终端用nc去发消息，程序也能正常处理

```
nc localhost 8090
```

# 3 select总结

select是早期berkly随着tcp/ip协议栈一起发出来的。有非常良好的兼容性，各个平台都支持,
是早期I/O多路复用比较好的方案

select的fd_size(最大描述符数)只有1024, 是在头文件里用宏写死的,
早期bsd内核最多只能开20个进程，在当时看来1024长度的fd_set已经大到用不完了, 放在今天肯定是不够了

一旦fd_set里任意fd上有事件发生，内核会立刻返回，将数据从内核空间拷贝到用户空间，然后应用进程需要遍历整个fd_set去寻找哪个fd有事件了。

所以，select的缺点主要有:

1. 最大并发限制: 1024个描述符
2. 内核/用户空间的fd_set数据拷贝
3. 遍历整个fd_set集合效率低，集合越大浪费的时间越多

为了解决这些问题，后人主要有2个改进，分别是`poll`和`epoll`, 后面我们会讲到。
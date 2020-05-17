# 1 epoll概念

`poll`系统调用相比于`select`主要解决了文件描述符的数量限制，但是在高并发场景下没有解决根本问题:

1. **fd数组整体在内核空间和用户空间之间拷贝**
2. **遍历整个fd数组找事件浪费资源**

这俩性能问题在**Banga**在1999年写了篇论文[A Scalable and Explicit Event
Delivery Mechanism for UNIX](http://static.usenix.org/event/usenix99/full_papers/banga/banga.pdf),提出`select`和`poll`都是无状态的，需要用户空间的进程**自行遍历查找事件**, 一种改进方案是**内核内部自己维护事件集合**.通过一个类似`declare_interest`的系统调用，内核能够**增量得更新进程感兴趣的事件集合列表**, 应用进程通过使用`get_next_event`调用能派发新事件给内核。

根据论文的研究成果，`LINUX`和`FreeBSD`各自给出的解决方案：`epoll`和`kqueue`.我们主要讨论epoll, 毕竟日常服务端环境都是LINUX.

在LINUX内核2.6以上，`epoll`才受到支持。

# 2 epoll函数

epoll操作过程有3个函数

```
#include <sys/epoll.h>

int epoll_create(int size);

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

1. `epoll_create`创建一个`epoll fd`和事件表, 在LINUX2.6.8以后是使用红黑树来管理epoll事件表，所以size没有太大作用
2. `epoll_ctl`操作上面创建的epoll事件表, 可以加入socket读写事件
3. `epoll_wait`类似于以前的`select`和`poll`,得到发生的事件, 如socket可读可写

`epoll_ctl`的第二个参数使用3个宏来表示动作:

- `EPOLL_CTL_ADD`: 注册新的fd到epfd中
- `EPOLL_CTL_MOD`: 修改意见注册的fd的监听事件
- `EPOLL_CTL_DEL`: 从epfd中删除一个fd

第四个参数用来告诉内核需要监听什么事件, `epoll_event`结构如下

```
struct epoll_event {
  __uint32_t events; // epoll事件
  epoll_data_t data; // 用户数据变量
}
```

epoll事件和以前`poll`的事件类型差不多,主要还是这仨:

- EPOLLIN: 对应的fd可读
- EPOLLOUT: 对应的fd可写
- EPOLLERR: 对应的fd发送错误

# 3 epoll实战

首先，MacOS是基于BSD的，所以是没有`epoll`函数的，在Mac下开发，得去LINUX环境编译。依然是老5步

1. 新建socket, 用于监听端口
2. socket的fd绑定端口
3. socket监听
4. 创建epoll fd
5. 发起epoll_wait获取事件，使用epoll_ctl管理事件

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <sys/epoll.h>

#define MAX_FD_NUM 1024
#define MAXLEN 1024

int buf_len = 0;

int main()
{
    // 1. 新建socket, 用于监听端口
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) printf("创建socket失败， error: %s (errno: %d)\n", strerror(errno), errno);

    // 2. 绑定端口
    unsigned short listenPort = 8090;
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(listenPort);

    int on = 1;
    // 设置socket绑定的端口，再程序关闭之后可以重复使用
    if ((setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(int))) < 0) {
        exit(1);
    }
    int bindRet = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    if (bindRet == -1) {
        printf("socket绑定地址失败， error: %s (errno: %d)\n", strerror(errno), errno);
        exit(1);
    }

    // 3. 监听端口
    int listenRet = listen(listenfd, 10);
    if (listenRet == -1) printf("socket监听端口失败， error: %s (errno: %d)\n", strerror(errno), errno);
    printf("socket 监听完毕, 地址: 127.0.0.1:%d", listenPort);

    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(struct sockaddr_in);

    // 4. 创建一个 epfd，并且把 listenfd 注册到这个 epfd上。
    int epfd = epoll_create(1024);
    struct epoll_event ev,events[20];
    ev.data.fd = listenfd;
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

    int cur_fd_num = 1;
    char buf[MAXLEN]={0};

    while (1) {
        // 5. 调用epoll_wait获取IO事件
        // nReady 就是 events 数组的长度。
        int nready = epoll_wait(epfd, events, 20, 50);

        int i = 0;
        for (; i < nready; i++) {
            if (events[i].data.fd == listenfd) {
                int client_sockfd = accept(listenfd,(struct sockaddr*)&client_addr,&client_addr_len);

                if(client_sockfd < 0) {
                    perror("accept");
                }
                else {
                    printf("accept client_addr %s\n",inet_ntoa(client_addr.sin_addr));
                    ev.data.fd = client_sockfd;
                    ev.events=EPOLLIN;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_sockfd, &ev);
                }
            }
            else if (events[i].events & EPOLLIN) {
                int connfd = events[i].data.fd;
                int n = recv(connfd, buf, MAXLEN, 0);
                if(n <= 0) {
                    if(ECONNRESET == errno) {
                        close(connfd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL, connfd, 0);
                    }
                    else {
                        perror("recv");
                    }
                }

                printf("receive %s", buf);
                buf_len = n;

                ev.data.fd = connfd;
                ev.events = EPOLLOUT;
                epoll_ctl(epfd, EPOLL_CTL_MOD, connfd, &ev);
            }
            else if (events[i].events & EPOLLOUT) {
                int connfd = events[i].data.fd;
                write(connfd, buf, buf_len);

                ev.data.fd = connfd;
                ev.events = EPOLLIN;
                epoll_ctl(epfd, EPOLL_CTL_MOD, connfd, &ev);
            }
        }
    }

    return 0;
}
```

使用gcc编译，运行

```
$ gcc -o epoll ./epoll.c
$ ./epoll
```

然后使用`nc`测试就好了

# 4 epoll总结

`epoll`解决了`select`和`poll`时代遗留的2个性能问题，不需要使fd数组整体在内核空间和用户空间之间来回拷贝，同时
不需要应用进程遍历整个fd数组以查找发生的事件。`epoll`使用`mmap`加速了内核和用户空间的消息传递，避免不必要的内存拷贝。
`epoll`只会返回活跃的socket fd,所以I/O效率不会随着fd数目增加而显著下降。

`epoll`还支持`ET(边缘触发)`和`LT(水平触发)`，这些就不细讲了
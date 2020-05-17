`poll`系统调用主要解决了`select`系统调用的2个问题：

1. 文件描述符数量(fd_setsize = 32)太小, 而且数值是使用宏写死的,这样在32位机器上最大文件描述符数量只有32*32=1024
2. 文件描述符集(fd_set)这种`值-结果参数`的api设计不是很好, select系统调用的时候要分别传读set,写set，更多事件不好细分

`poll`系统调用使用了pollfd数据结构来表示事件数组，没有了`fd_setsize`的限制,同时支持更多的事件类型

# 1  pollfd结构

`pollfd`的结构是:

```
struct pollfd {
  int fd; // 需要检查的fd
  short events; // 该fd感兴趣的事件
  short revents; // 该fd当前发生的事件
}
```

其中事件类型有很多，比如`POLLIN`代表可读，`POLLRDNORM`代表普通消息可读,
更多事件可以查看[poll事件](http://man7.org/linux/man-pages/man2/poll.2.html)
相比于`select`可以更细化的监听事件, 同时分开使用字段来表示事件和结果(events和revents)。

# 2  poll函数

poll函数的签名如下:

```
#include <sys/poll.h>

int poll(struct pollfd *fdarray, unsigned long nfds, int timeout);
```

其中，第一个参数是指向1个pollfd结构数组的第一个元素的指针,第2个参数是pollfd数组元素的个数,
第三个参数是`poll`函数返回前等待多长时间，单位是毫秒。

# 3  poll版echo server实例

使用poll函数来实现`echo server`, 整个过程依然还是分5步

1. 新建socket, 用于监听端口
2. socket的fd绑定端口
3. socket监听
4. 初始化pollfd数组
5. 发起poll系统调用获取所有pollfd，寻找和处理事件

```
//
// Created by cris wang on 2018/6/28.
//
#include <sys/poll.h>
#include <sys/socket.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/syslimits.h>

int main() {
    // 1. 新建tcp流式socket
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
    if ((setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(int)))) {
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

    // 3. 新建pollfd数组
    struct pollfd client[OPEN_MAX];
    client[0].fd = listenfd;
    client[0].events = POLL_IN;
    for (int i = 1; i < OPEN_MAX; i++) {
        client[i].fd = -1;
    }
    int maxi = 0, i = 0, MAXLEN = 1024;
    char buf[MAXLEN];

    // 4. 使用poll
    for (;;) {
        int nready = poll(client, (maxi + 1), -1);

        if (client[0].revents & POLL_IN) {
            // 接受新连接
            struct sockaddr_in clientaddr;
            int clilen = sizeof(clientaddr);
            int connfd = accept(listenfd, (struct sockaddr *) &clientaddr, &clilen);

            // 找到pollfd数组里第一个可用的pollfd
            for (i = 0; i < OPEN_MAX; i++) {
                if (client[i].fd < 0) {
                    client[i].fd = connfd;
                    break;
                }
            }

            if (i == OPEN_MAX) exit(1);
            // 设置fd感兴趣的事件
            client[i].events = POLL_MSG;
            if (i > maxi) maxi = i;
            if (--nready <= 0) continue;
        }

        for (i = 1; i <= maxi; i++) {
            struct pollfd sockfd = client[i];
            if (sockfd.fd < 0) continue;

            if (sockfd.revents & (POLL_MSG | POLL_ERR)) {
                int n = recv(sockfd.fd, buf, MAXLEN, 0);
                if (n == 0 || n < 0) {// 0表示连接关闭 <0表示连接重置
                    close(sockfd.fd);
                    client[i].fd = -1;
                } else write(sockfd.fd, buf, n);

                if (--nready < 0) break; // 没有更多的可读fd
            }
        }
    }
}
```

依然使用好用的`nc`, 多开几个终端用nc去发消息，程序也能正常处理

```
nc localhost 8090
```

# 4 poll总结

`poll`系统调用解决了`select`的文件描述符限制，但是依然有`select`留下的性能缺点:

1. fd数组(不管是fd_set还是pollfd)都要在`用户空间`和`内核空间`之间来回拷贝
2. 被监控的fd数组有事件的时候，需要遍历整个数组

下面会谈到终极解决方案`epoll`.
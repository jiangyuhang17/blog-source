---
title: Beej's Guide to Network Programming 读书笔记（下）
date: 2020-04-18 16:27:25
categories: 网络编程
tags: [网络编程]
---

# Client-Server 基础

## 一个简单的 Stream Server

``` C
/*
** server.c － 展示一个 stream socket server
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <sys/wait.h>
#include <signal.h>

#define PORT " 3490" // 提供给用戶连接的 port
#define BACKLOG 10 // 有多少个特定的连接队列（pending connections queue）

void sigchld_handler(int s)
{
    while(waitpid(-1, NULL, WNOHANG) > 0);
}

// 取得 sockaddr， IPv4 或 IPv6：
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }
    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
    int sockfd, new_fd; // 在 sock_fd 进行 listen， new_fd 是新的连接
    struct addrinfo hints, *servinfo, *p;
    struct sockaddr_storage their_addr; // 连接者的地址资料
    socklen_t sin_size;
    struct sigaction sa;
    int yes=1;
    char s[INET6_ADDRSTRLEN];
    int rv;

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE; // 使用我的 IP

    if ((rv = getaddrinfo(NULL, PORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, " getaddrinfo: %s\n" , gai_strerror(rv));
        return 1;
    }

    // 以循环找出全部的结果， 并绑定（bind）到第一个能用的结果
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
            perror(" server: socket" );
            continue;
        }
        if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int)) == -1) {
            perror(" setsockopt" );
            exit(1);
        }
        if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
            close(sockfd);
            perror(" server: bind" );
            continue;
        }
        break;
    }

    if (p == NULL) {
        fprintf(stderr, " server: failed to bind\n" );
        return 2;
    }

    freeaddrinfo(servinfo); // 全部都用这个 structure

    if (listen(sockfd, BACKLOG) == -1) {
        perror(" listen" );
        exit(1);
    }

    sa.sa_handler = sigchld_handler; // 收拾全部死掉的 processes
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;

    if (sigaction(SIGCHLD, &sa, NULL) == -1) {
        perror(" sigaction" );
        exit(1);
    }

    printf(" server: waiting for connections...\n" );

    while(1) { // 主要的 accept() 循环
        sin_size = sizeof their_addr;
        new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);
        if (new_fd == -1) {
            perror(" accept" );
            continue;
        }

        inet_ntop(their_addr.ss_family, get_in_addr((struct sockaddr *)&their_addr), s, sizeof s);
        printf(" server: got connection from %s\n" , s);

        if (!fork()) { // 这个是 child process
            close(sockfd); // child 不需要 listener
            if (send(new_fd, " Hello, world!" , 13, 0) == -1)
                perror(" send" );
            close(new_fd);
            exit(0);
        }
        close(new_fd); // parent 不需要这个
    }

    return 0;
}
```

`sigaction()`这个代码是用来清理 zombie processes（僵尸进程），当 parent process 所`fork()`出来的 child process 结束时，且 parent process 没有取得 child process 的离开状态时，就会出现 zombie process。如果你产生了许多 zombies，但却无法清除他们时， 你的系统管理员就会开始焦虑不安了。

## 一个简单的 Stream Client

``` C
/*
** client.c -- 一个 stream socket client 的 demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define PORT " 3490" // Client 所要连接的 port
#define MAXDATASIZE 100 // 我们一次可以收到的最大字节数量（number of bytes）

// 取得 IPv4 或 IPv6 的 sockaddr：

void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }
    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(int argc, char *argv[])
{
    int sockfd, numbytes;
    char buf[MAXDATASIZE];
    struct addrinfo hints, *servinfo, *p;
    int rv;
    char s[INET6_ADDRSTRLEN];

    if (argc != 2) {
        fprintf(stderr," usage: client hostname\n" );
        exit(1);
    }

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    if ((rv = getaddrinfo(argv[1], PORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, " getaddrinfo: %s\n" , gai_strerror(rv));
        return 1;
    }

    // 用循环取得全部的结果， 并先连接到能成功连接的
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype,
            p->ai_protocol)) == -1) {
            perror(" client: socket" );
            continue;
        }
        if (connect(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
            close(sockfd);
            perror(" client: connect" );
            continue;
        }
        break;
    }

    if (p == NULL) {
        fprintf(stderr, " client: failed to connect\n" );
        return 2;
    }

    inet_ntop(p->ai_family, get_in_addr((struct sockaddr *)p->ai_addr), s, sizeof s);

    printf(" client: connecting to %s\n" , s);

    freeaddrinfo(servinfo); // 全部皆以这个 structure 完成

    if ((numbytes = recv(sockfd, buf, MAXDATASIZE-1, 0)) == -1) {
        perror(" recv" );
        exit(1);
    }

    buf[numbytes] = '\0';
    printf(" client: received '%s'\n" ,buf);

    close(sockfd);

    return 0;
}
```

## Datagram Sockets

``` C
/*
** listener.c -- 一个 datagram sockets " server" 的 demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define MYPORT " 4950" // 用戶所要连线的 port
#define MAXBUFLEN 100

// get sockaddr, IPv4 or IPv6:
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }
    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
    int sockfd;
    struct addrinfo hints, *servinfo, *p;
    int rv;
    int numbytes;
    struct sockaddr_storage their_addr;
    char buf[MAXBUFLEN];
    socklen_t addr_len;
    char s[INET6_ADDRSTRLEN];

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC; // 设定 AF_INET 以强制使用 IPv4
    hints.ai_socktype = SOCK_DGRAM;
    hints.ai_flags = AI_PASSIVE; // 使用我的 IP

    if ((rv = getaddrinfo(NULL, MYPORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, " getaddrinfo: %s\n" , gai_strerror(rv));
        return 1;
    }

    // 用循环找出全部的结果， 并 bind 到首先找到能 bind 的
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
        perror(" listener: socket" );
        continue;
        }
        if (bind(sockfd, p->ai_addr, p->ai_addrlen) == -1) {
            close(sockfd);
            perror(" listener: bind" );
            continue;
        }
        break;
    }

    if (p == NULL) {
    fprintf(stderr, " listener: failed to bind socket\n" );
    return 2;
    }

    freeaddrinfo(servinfo);

    printf(" listener: waiting to recvfrom...\n" );

    addr_len = sizeof their_addr;
    if ((numbytes = recvfrom(sockfd, buf, MAXBUFLEN-1 , 0, (struct sockaddr *)&their_addr, &addr_len)) == -1) {
        perror(" recvfrom" );
        exit(1);
    }

    printf(" listener: got packet from %s\n", inet_ntop(their_addr.ss_family, get_in_addr((struct sockaddr *)&their_addr), s, sizeof s));

    printf(" listener: packet is %d bytes long\n" , numbytes);

    buf[numbytes] = '\0';
    printf(" listener: packet contains \" %s\" \n" , buf);

    close(sockfd);

    return 0;
    }
```

``` C
/*
** talker.c -- 一个 datagram " client" 的 demo
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define SERVERPORT " 4950" // 用户所要连接的 port

int main(int argc, char *argv[])
{
    int sockfd;
    struct addrinfo hints, *servinfo, *p;
    int rv;
    int numbytes;

    if (argc != 3) {
        fprintf(stderr," usage: talker hostname message\n" );
        exit(1);
    }

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_DGRAM;

    if ((rv = getaddrinfo(argv[1], SERVERPORT, &hints, &servinfo)) != 0) {
        fprintf(stderr, " getaddrinfo: %s\n" , gai_strerror(rv));
        return 1;
    }

    // 用循环找出全部的结果， 并产生一个 socket
    for(p = servinfo; p != NULL; p = p->ai_next) {
        if ((sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
            perror(" talker: socket" );
            continue;
        }
        break;
    }

    if (p == NULL) {
        fprintf(stderr, " talker: failed to bind socket\n" );
        return 2;
    }

    if ((numbytes = sendto(sockfd, argv[2], strlen(argv[2]), 0, p->ai_addr, p->ai_addrlen)) == -1) {
        perror(" talker: sendto" );
        exit(1);
    }

    freeaddrinfo(servinfo);
    printf(" talker: sent %d bytes to %s\n" , numbytes, argv[1]);

    close(sockfd);

    return 0;
}
```

要记得：使用 UDP datagram socket 传送的数据是不会使命必达的！

如果 talker 调用`connect()`并指定 listener 的地址。从这开始，talker 就只能从`connect()`所指定的地址进行传送与接收。因此，你不用使用`sendto()`与`recvfrom()`，可以单纯使用`send()`与`recv()`就好。

# 高等技术

## Blocking（阻塞）

很多函数都会 block，`accept()`会 block，全部的`recv()`函数都会 block。原因是它们有权这么做。当你先用`socket()`建立  socket descriptor 时，kernel 会将它设置为 blocking。若你不想要 blocking socket，你必须调用`fcntl()`：

``` C
#include <unistd.h>
#include <fcntl.h>

sockfd = socket(PF_INET, SOCK_STREAM, 0);
fcntl(sockfd, F_SETFL, O_NONBLOCK);
```

将 socket 设置为 non-blocking（非阻塞），你就能 poll（轮询） socket 以取得数据。如果你试着读取 non-blocking  socket，而 socket 没有数据时，函数就不会发生 block，而是返回 -1，并将 errno 设置为`EWOULDBLOCK`，你可以过一段时间再检查socket是否有数据可以读了。

然而，一般来说，这样 polling 是不好的想法。如果你让程序一直忙着查 socket 上是否有数据，则会浪费 CPU 的时间，这样是不合适的。比较漂亮的解法是利用下一节的`select()`来检查 socket 是否有数据需要读取。

## select() － 同步 IO 多路复用

`select()`授予你同时监视多个 sockets 的权力，它会告诉你哪些 sockets 已经有数据可以读取、哪些 sockets 已经可以写入，如果你真的想知道， 还会告诉你哪些 sockets 触发了异常。

即使`select()`有很好的可移植性，不过却是监视 sockets 最慢的方法。一个比较可行的替代方案是`libevent`（an event notification library）或者其它类似的方法， 封装全部的系统相依要素，用以取得 socket 的通知。

``` C
// 函数原型

#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

// numfds 参数应该要设置为 file descriptor 的最高值加 1。
int select(int numfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

每个 sets 的型别都是 fd_set，下列是用来控制这个型别的 macro：

``` C
FD_SET(int fd, fd_set *set); // 将 fd 新增到 set。
FD_CLR(int fd, fd_set *set); // 从 set 移除 fd。
FD_ISSET(int fd, fd_set *set); // 若 fd 在 set 中，返回 true。
FD_ZERO(fd_set *set); // 将 set 整个清为零。
```

`struct timeval`让你可以设置 timeout 的周期。如果时间超过了，而`select()`还没有找到任何就绪的 file descriptor 时，它会回传，让你可以继续做其它事情。还有，当函数回传时，会更新 timeout，用以表示还剩下多少时间，这个行为取决于你所使用的 Unix。不过无论你将`struct timeval`设置的多小，你可能还要等待一小段 standard Unix timeslice（标准 Unix 时间片段）。

``` C
struct timeval {
    int tv_sec; // 秒（second）
    int tv_usec; // 微秒（microseconds）
};
```

``` C
/*
** select.c -- a select() demo
*/

#include <stdio.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

#define STDIN 0 // standard input 的 file descriptor

int main(void)
{
    struct timeval tv;
    fd_set readfds;

    tv.tv_sec = 2;
    tv.tv_usec = 500000;

    FD_ZERO(&readfds);
    FD_SET(STDIN, &readfds);

    // 不用管 writefds 与 exceptfds：
    select(STDIN+1, &readfds, NULL, NULL, &tv);

    if (FD_ISSET(STDIN, &readfds))
        printf(" A key was pressed!\n" );
    else
        printf(" Timed out.\n" );
    return 0;
}
```

这个方法用在需要等待数据的 datagram socket 上很好。

有些系统会用这个方式来使用`select()`，而有些不行，如果你想要用它，你应该要参考你系统上的 man 使用手册说明看是否会有问题。有些系统会更新`struct timeval`的时间，用来反映`select()`原本还剩下多少时间 timeout；不过有些却不会。如果你想要程序是可移植的，那就不要倚赖这个特性。（如果你需要追踪剩下的时间， 可以使用`gettimeofday()`）

``` C
/*
** selectserver.c -- 一个 cheezy 的多人聊天室 server
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define PORT " 9034" // 我们正在 listen 的 port

// 取得 sockaddr， IPv4 或 IPv6：
void *get_in_addr(struct sockaddr *sa)
{
    if (sa->sa_family == AF_INET) {
        return &(((struct sockaddr_in*)sa)->sin_addr);
    }
    return &(((struct sockaddr_in6*)sa)->sin6_addr);
}

int main(void)
{
    fd_set master; // master file descriptor 表
    fd_set read_fds; // 给 select() 用的暂时 file descriptor 表
    int fdmax; // 最大的 file descriptor 数目
    int listener; // listening socket descriptor
    int newfd; // 新接受的 accept() socket descriptor
    struct sockaddr_storage remoteaddr; // client address
    socklen_t addrlen;
    char buf[256]; // 储存 client 数据的缓冲区
    int nbytes;
    char remoteIP[INET6_ADDRSTRLEN];
    int yes=1; // 供底下的 setsockopt() 设置 SO_REUSEADDR
    int i, j, rv;
    struct addrinfo hints, *ai, *p;

    FD_ZERO(&master); // 清除 master 与 temp sets
    FD_ZERO(&read_fds);

    // 给我们一个 socket， 并且 bind 它
    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;

    if ((rv = getaddrinfo(NULL, PORT, &hints, &ai)) != 0) {
        fprintf(stderr, " selectserver: %s\n" , gai_strerror(rv));
        exit(1);
    }

    for(p = ai; p != NULL; p = p->ai_next) {
        listener = socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (listener < 0) {
            continue;
        }

        // 避开这个错误信息：" address already in use"
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int));

        if (bind(listener, p->ai_addr, p->ai_addrlen) < 0) {
            close(listener);
            continue;
        }

        break;
    }

    // 若我们进入这个判断式， 则表示我们 bind() 失败
    if (p == NULL) {
        fprintf(stderr, " selectserver: failed to bind\n" );
        exit(2);
    }
    freeaddrinfo(ai); // all done with this

    // listen
    if (listen(listener, 10) == -1) {
        perror(" listen" );
        exit(3);
    }

    // 将 listener 新增到 master set
    FD_SET(listener, &master);

    // 持续追踪最大的 file descriptor
    fdmax = listener; // 到此为止， 就是它了

    // 主要循环
    for( ; ; ) {
        read_fds = master; // 复制 master

        if (select(fdmax+1, &read_fds, NULL, NULL, NULL) == -1) {
            perror(" select" );
            exit(4);
        }

        // 在现存的连接中寻找需要读取的数据
        for(i = 0; i <= fdmax; i++) {
            if (FD_ISSET(i, &read_fds)) { // 我们找到一个！！
                if (i == listener) {
                    // handle new connections
                    addrlen = sizeof remoteaddr;
                    newfd = accept(listener, (struct sockaddr *)&remoteaddr, &addrlen);

                    if (newfd == -1) {
                        perror(" accept" );
                    } else {
                        FD_SET(newfd, &master); // 新增到 master set
                        if (newfd > fdmax) { // 持续追踪最大的 fd
                            fdmax = newfd;
                        }

                        printf(" selectserver: new connection from %s on socket %d\n" , inet_ntop(remoteaddr.ss_family, get_in_addr((struct sockaddr*)&remoteaddr), remoteIP, INET6_ADDRSTRLEN), newfd);
                    }
                } else {
                    // 处理来自 client 的数据
                    if ((nbytes = recv(i, buf, sizeof buf, 0)) <= 0) {
                        // got error or connection closed by client
                        if (nbytes == 0) {
                            // 关闭连接
                            printf(" selectserver: socket %d hung up\n" , i);
                        } else {
                            perror(" recv" );
                        }
                        close(i); // bye!
                        FD_CLR(i, &master); // 从 master set 中移除
                    } else {
                        // 我们从 client 收到一些数据
                        for(j = 0; j <>= fdmax; j++) {
                            // 发送给聊天室的其他人！
                            if (FD_ISSET(j, &master)) {
                                // 不用送给 listener 跟我们自己
                                if (j != listener && j != i) {
                                    if (send(j, buf, nbytes, 0) == -1) {
                                        perror(" send" );
                                    }
                                }
                            }
                        }
                    }
                } // END handle data from client
            } // END got new incoming connection
        } // END looping through file descriptors
    } // END for( ; ; )--and you thought it would never end!

    return 0;

}
```       

还有一个名为`poll()`的函数， 它的行为与`select()`很像， 但是在管理file descriptor sets 时用不一样的结构。

## 不完整的传送

`sendall()`会尽力将数据送出， 不过如果有错误发生时， 它就会立刻回传给你。函数在错误时返回 -1，而 errno 仍然从调用`send()`中设置。

``` C
#include <sys/types.h>
#include <sys/socket.h>

int sendall(int s, char *buf, int *len)
{
    int total = 0; // 我们已经送出多少 bytes 的数据
    int bytesleft = *len; // 我们还有多少数据要送
    int n;
    while(total < *len) {
        n = send(s, buf+total, bytesleft, 0);
        if (n == -1) { break; }
        total += n;
        bytesleft -= n;
    }
    *len = total; // 返回实际上送出的数据量
    return n==-1? -1:0; // 失败时返回 -1 丶成功时返回 0
}

// 调用函数的示例
char buf[10] = " Beej!" ;
int len;

len = strlen(buf);
if (sendall(s, buf, &len) == -1) {
    perror(" sendall" );
    printf(" We only sent %d bytes because of the error!\n" , len);
}
```

如果数据包的长度是会变动的（variable），接收端要如何知道另一端的数据包何时开始与结束呢？是的， 你或许必须封装（encapsulate）

## Serialization - 如何封装数据

我们把变量从内存中变成可存储或传输的过程称为 Serialization（序列化）。序列化之后，就可以把序列化后的内容写入磁盘，或者通过网络传输到别的机器上。反过来，把变量内容从序列化的对象重新读到内存里称之为反序列化。

### 文本数据的封装

封装数据，以最简单的例子而言，需要 header（标头），用来代表识别的资料或数据长度，或者都有。

举例来说，使用 SOCK_STREAM 的多用户聊天程序。当某个用户输入［＂says＂］某些字，会有两笔资料要传送给 server：＂谁＂以及＂说了什么＂。这样的问题是信息的长度是会变动的。一个叫做＂tom＂的人可能会说＂Hi＂， 而另一个叫做＂Benjamin＂的人可能说：＂Hey guys what is up？）＂所以你在收到全部的数据之後， 将它全部`send()`给 clients。 你输出的 data stream 类似这样：

`tomHiBenjaminHeyguyswhatisup?`

那 client 要如何知道信息何时开始与结束呢？可以让全部的讯息都一样长，并调用`sendall()`就行了。但是这样会浪费带宽！

所以我们以小巧的 header 与数据包结构封装（encapsulate）数据。Client 与 server 都知道如何封装（pack）与解封装（unpack）这笔数据。我们会开始定义一个协议（protocol），用来描述 client 与 server 是如何沟通的！

在这个例子中， 假设用户的名称是固定 8 个字元， 并用 '\0' 结尾。 然后接着让我们假设数据的长度是变动的，最多高达 128 个字元。我们看个可能在这个情况会用到的数据包结构范例。

1. len［1 个 byte，unsigned］：数据包的总长度，计算 8 个 bytes 的用户名称，以及聊天数据

2. name［8 个 bytes］：用户名称

3. chatdata［n 个 bytes］：数据本身，最多 128 bytes。 数据包的长度应该要以这个数据长度加 8 ［上面的 name 栏位长度］来计算

当你传送数据时，你应该要谨慎点，使用类似前面的`sendall()`指令，因而你可以知道全部的数据都有送出。

同样地，当你接收这笔数据时，你需要额外做些处理。如果要保险一点，你应该假设你可能只会收到部分的数据包内容，但是我们这次调用`recv()`全部就只收到这些数据。我们需要一次又一次的调用`recv()`，直到完整地收到数据包内容。

我们可以知道所要接收的数据包它全部的 byte 数量，因为这个数量会记载在数据包前面。我们也知道最大的数据包大小是 1 + 8 + 128，或者 137 bytes，因为这是之前我们定义的。

我们可以采用两种recv的策略：

第一种：

你知道每个数据包是以长度做开头，所以你可以调用 `recv()`只取得数据包长度。接着，你知道长度以後你就可以再次调用`recv()`，这时候你就可以正确地指定剩下的数据包长度，直到你收到完整的数据包内容为止。

这个方法的优点是你只需有一个足以存放一个数据包的缓冲区，而缺点是你为了要接收全部的数据， 至少调用两次的`recv()`。

第二种：

直接调用`recv()`，并且指定你所要接收的数据包之最大数据量。这样的话，无论你收到多少，都将它写入缓冲区，并最后检查数据包是否完整。当然，你可能会收到下一个数据包的内容，所以你需要有足够的空间。你所能做的是声明一个足以容纳两个数据包的数组，这是你在数据包到达时， 你可以重新建构数据包的地方。

当你处理完第一个数据包后，你可以将第一个数据包的数据从工作缓冲区中清掉，并将第二个数据包的部分内容移到缓冲区的前面，准备进行下一次的`recv()`。程序可以写成利用`环状缓冲区`（circular buffer），消除第二个数据包移动到开头的开销。

### 二进制数据的封装

要将文字数据透过网路传送很简单，你已经知道了，不过如果你想要送一些＂二进制＂的数据，如 int 或 float，会发生什麽事情呢？这里有一些选择。

1. 将数字转换为文字，使用如`sprintf()`的函数，接着传送文字。接收者会使用如`strtol()`函数解析文字，并转换为数字。

2. 直接以原始数据传送，将指向数据的指针传递给`send()`。

3. 将数字编码（encode）为可移植的二进制格式，接收者会将它译码（decode）。

实际上，上面全部的方法都有它们的缺点与优点，但是通常偏好第三个方法

第一个方法：在传送以前先将数字编码为文字，优点是你可以很容易打印出及读取来自网路的数据。有时，人类易读的协定比较适用于频带不敏感（non-bandwidth-intensive）的情况，例如：Internet Relay Chat（IRC）。然而，缺点是转换耗时，且总是需要比原本的数字使用更多的空间。

第二个方法：传送原始数据（raw data），这个方法相当简单，但是危险！只要将数据指针提供给`send()`。事实证明不是全部的架构都能表示`double`或`int`。如果你不需要可移植性，在这样的情况下这个方法很好，而且快速。

``` C
// 发送者
double d = 3490.15926535;

send(s, &d, sizeof d, 0); /* 危险， 不具可移植性！ */

// 接收者
double d;

recv(s, &d, sizeof d, 0); /* 危险， 不具可移植性！ */
```

第三个方法：我们可以将数据封装为接收者已知的二进制格式， 让接收着可以在远端解码。我们已经看过了`htons()`范例了，它将数字从 host 格式编码为 Network Byte Order 格式；如果要解码这个数字，接收端会调用`ntohs()`。

但是没有这样的函数可供非整数型别使用，这是因为C语言并没有规范标准的方式来做。要做的事情是将数据封装到已知的格式， 并透过网路送出。例如：封装`float`，这里的东西有很大的改善空间。这部分的内容就不详细展开，具体的可以参考这个教材。

## Broadcast Packet（广播数据包）

到了这里，本文已经谈了如何将数据从一台主机传送到另一台主机。现在考虑同时将数据送给多个主机！

用 UDP（TCP 不行）与标准的 IPv4，可以透过一种叫作广播（broadcasting）的机制达成。IPv6 不支援广播，所以你必须要采用比较高级的技术－群播（multicasting），本文不讨论这个。

你必须在将广播数据包送到网路之前，先设置`SO_BROADCAST` socket 选项。

指定广播信息的目地地址有两种选择：

1. 将数据送给子网路（subnet）的广播地址，就是将 subnet's network（子网路网段）的主机号那部分全部填 1。如果网路是 192.168.1.0，而netmask是 255.255.255.0，广播地址就是 192.168.1.255。

2. 将数据送给 global（全局的）广播地址，255.255.255.255，又称为 INADDR_BROADCAST。Routers 不会将这类的广
播数据包转发出你的区域网络

``` C
/*
** broadcaster.c -- 一个类似 talker.c 的 datagram " client" ，
** 差异在於这个可以广播
**
** 这里使用的都是IPv4的旧方法
** gethostbyname， 手动填struct sockaddr_in（inet_addr），inet_ntoa
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>

#define SERVERPORT 4950 // 所要连接的 port

int main(int argc, char *argv[])
{
    int sockfd;
    struct sockaddr_in their_addr; // 连接者的地址资料
    struct hostent *he;
    int numbytes;
    int broadcast = 1;
    //char broadcast = '1'; // 如果上面这行不能用的话， 改用这行

    if (argc != 3) {
        fprintf(stderr," usage: broadcaster hostname message\n" );
        exit(1);
    }

    if ((he=gethostbyname(argv[1])) == NULL) { // 取得 host 资料
        perror(" gethostbyname" );
        exit(1);
    }

    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) == -1) {
        perror(" socket" );
        exit(1);
    }

    // 这个 call 就是要让 sockfd 可以送广播数据包
    if (setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &broadcast, sizeof broadcast) == -1) {
        perror(" setsockopt (SO_BROADCAST)" );
        exit(1);
    }

    their_addr.sin_family = AF_INET; // host byte order
    their_addr.sin_port = htons(SERVERPORT); // short, network byte order
    their_addr.sin_addr = *((struct in_addr *)he->h_addr);
    memset(their_addr.sin_zero, '\0', sizeof their_addr.sin_zero);

    if ((numbytes=sendto(sockfd, argv[2], strlen(argv[2]), 0, (struct sockaddr *)&their_addr, sizeof their_addr)) == -1) {
        perror(" sendto" ); 
        exit(1);
    }

    printf(" sent %d bytes to %s\n", numbytes,inet_ntoa(their_addr.sin_addr));

    close(sockfd);

    return 0;
}
```

这个跟一般的 UDP client/server 没什么不同，除了 client 可以送出广播数据包。如果 listener 没有回应，可能是因为它绑到 IPv6 地址了，试着将 listener.c 中的`AF_UNSPEC`改成`AF_INET`，强制使用 IPv4。

如果 listener 收到你直接送给它的数据， 但不是在广播地址没有数据， 可能是因为你本地端上有防火墙（firewall）封锁了这些数据包。

防火墙，是指一种将内部网络和公有网络（如Internet）分开的方法。它实际上是一种隔离技术，在两个网络通信时执行的一种访问控制尺度，它能允许你"同意"的人和数据进入你的网络，同时将你“不同意”的人和数据拒之门外，最大限度地阻止网络中的黑客来访问你的网络。 

# 常见的问题

## 实现调用 connect() 的 timeout

你要用`socket()`建立一个 socket descriptor，将它设置为 non-blocking，然后调用`connect()`，而如果一切顺利，`connect()` 会立即返回 -1，并将 errno 设置为`EINPROGRESS`。接着你要调用`select()`并设置你想要的 timeout 时间，传递读写组（read and write sets）的 socket descriptor。如果`select()`没有发生 timeout，这表示`connect()`调用已经完成。此时，你必须使用`getsockopt()`设置`SO_ERROR`选择项以取得`connect()`的返回值，在没有错误时，这个值应该是零。最后，在你开始透过 socket 传输数据以前，你可能想要再将它设置回 blocking。

要注意的是，这么做的好处是让你的程序在连接（connecting）期间也可以另外做点事情。比如：你可以将 timeout 时间设定为类似 500 毫秒，并在每次 timeout 发生时更新显示器画面，然後再次调用`select()`。当你已经调用了`select()`时，并且 timeout 了，像这样重复了 20 次，你就会知道应该放弃这个连接了。

## 实现调用 recv() 的 timeout

还是使用`select()`，请注意，recvtimeout() 在 timeout 的例子中会返回 -2。这是因为在呼叫`recv()`返回 0 值时所代表的意思是对方已经关闭了连接，而 -1 表示＂错误＂，所以我选择 -2 做为我的 timeout 表示。

``` C
#include <unistd.h>
#include <sys/time.h>
#include <sys/types.h>
#include <sys/socket.h>

int recvtimeout(int s, char *buf, int len, int timeout)
{
    fd_set fds;
    int n;
    struct timeval tv;

    // 设置 file descriptor set
    FD_ZERO(&fds);
    FD_SET(s, &fds);

    // 设置 timeout 的数据结构 struct timeval
    tv.tv_sec = timeout;
    tv.tv_usec = 0;

    // 一直等到 timeout 或收到数据
    n = select(s+1, &fds, NULL, NULL, &tv);
    if (n == 0) return -2; // timeout!
    if (n == -1) return -1; // error
    // 数据一定有在这里， 所以调用一般的 recv()
    return recv(s, buf, len, 0);
}

// 调用 recvtimeout() 的示例：
n = recvtimeout(s, buf, sizeof buf, 10); // 10 second timeout
if (n == -1) {
    // 发生错误
    perror(" recvtimeout" );
}
else if (n == -2) {
    // 发生 timeout
} else {
    // 从 buf 收到一些数据
}
```
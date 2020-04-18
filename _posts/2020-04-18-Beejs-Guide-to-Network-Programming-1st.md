---
title: Beej's Guide to Network Programming 读书笔记（上）
date: 2020-04-18 16:25:20
categories: 网络编程
tags: [网络编程]
---

# 何谓 Socket

Socket 是利用标准 UNIX file descriptors（文件描述符）与其它程序沟通的一种方式。可以用一般的`read()`与`write()`调用通过socket进行通讯，但是`send()`与`recv()`让你能对数据传输有更多的控制权。

一般的 socket 只能读取传输层以上（不含）的数据， raw socket 一般用在设计 network sniffer，可以让应用程序取得网路数据包底层的数据（如 TCP 层、IP 层、数据链路层），并用以分析数据包。

主要讨论两种 Internet sockets，一个是 Stream Sockets；而另一个是 Datagram Sockets，分别以`SOCK_STREAM`与`SOCK_DGRAM`来表示。

为什麽你要用一个不可靠的底层协议（UDP）？有两个理由：第一个理由是速度，第二个理由还是速度。

# 结构与数据转换

## 字节顺序

网络序为大端序，即数字先存储比较大那一边（高位放在左边）

``` C
// Network Byte Order <-> Host Byte Order
htons() // host to network short
htonl() // host to network long
ntohs() // network to host short
ntohl() // network to host long
```

## 数据结构

`struct addrinfo`，这个数据结构是最近的发明，用来准备之后要用的 socket 地址数据结构，也用在主机名（host name）及服务名（service name）的查询。

``` C
struct addrinfo {
    int ai_flags; // AI_PASSIVE, AI_CANONNAME等
    int ai_family; // AF_INET, AF_INET6, AF_UNSPEC
    int ai_socktype; // SOCK_STREAM, SOCK_DGRAM
    int ai_protocol; // 用 0 当作 "any"
    size_t ai_addrlen; // ai_addr 的大小， 单位是 byte
    struct sockaddr *ai_addr; // struct sockaddr_in 或 _in6
    char *ai_canonname; // 典型的 hostname
    struct addrinfo *ai_next; // 链表的下个节点
};
```

要注意的是，这是个链表，`ai_next`是指向下一个成员（element）， 可能会有多个结果让你选择。

`struct sockaddr`记录了很多 sockets 类型的 socket 的地址资料。

``` C
struct sockaddr {
    unsigned short sa_family; // address family, AF_xxx
    char sa_data[14]; // 14 bytes of protocol address
};
```

`sa_data`包含一个 socket 的目地地址与端口号。这样很不方便，因为你不会想要手动的将地址封装到`sa_data`里。

为了处理`struct sockaddr`，程序设计师建立了对等平行的数据结构：`struct sockaddr_in`（in是代表internet）用在 IPv4。指向`struct sockaddr_in`的指针可以转型为指向`struct sockaddr`的指针，反之亦然。

``` C
// (IPv4 only--see struct sockaddr_in6 for IPv6)

struct sockaddr_in {
    short int sin_family; // Address family, AF_INET
    unsigned short int sin_port; // Port number
    struct in_addr sin_addr; // Internet address
    unsigned char sin_zero[8]; // 与 struct sockaddr 相同的大小
};

// (仅限 IPv4 — Ipv6 请参考 struct in6_addr)

// Internet address (a structure for historical reasons)

struct in_addr {
    uint32_t s_addr; // that's a 32-bit int (4 bytes)
};
```

要注意的是`sin_zero`（这是用来将数据结构补足符合`struct sockaddr`的长度），应该要使用`memset()`函数将`sin_zero`整个清为零。还有`struct in_addr`和`sin_port`必须是 Network Byte Order（利用`htons()`）。

``` C
// (IPv6 only--see struct sockaddr_in and struct in_addr for IPv4)

struct sockaddr_in6 {
    u_int16_t sin6_family; // address family, AF_INET6
    u_int16_t sin6_port; // port number, Network Byte Order
    u_int32_t sin6_flowinfo; // IPv6 flow information
    struct in6_addr sin6_addr; // IPv6 address
    u_int32_t sin6_scope_id; // Scope ID
};

struct in6_addr {
    unsigned char s6_addr[16]; // IPv6 address
};
```

这个简单的`struct sockaddr_storage`是设计用来足够储存 IPv4 与 IPv6 结构的结构体。你可以在`ss_family`栏位看到地址家族，检查它是`AF_INET`或`AF_INET6`（是 IPv4 或 IPv6）。之后如果你愿意的话，你就可以将它转型为`struct sockaddr_in`或`struct sockaddr_in6`。

``` C
struct sockaddr_storage {
    sa_family_t ss_family; // address family
    // all this is padding, implementation specific, ignore it:
    char __ss_pad1[_SS_PAD1SIZE];
    int64_t __ss_align;
    char __ss_pad2[_SS_PAD2SIZE];
};
```

## IP地址

使用`inet_pton()`函数将 IP address 字符串格式转换成网络地址格式，并依照你指定的`AF_INET`或`AF_INET6`来决定要储存在`struct in_addr`或`struct in6_addr`。

``` C
struct sockaddr_in sa; // IPv4
struct sockaddr_in6 sa6; // IPv6
inet_pton(AF_INET, " 192.0.2.1" , &(sa.sin_addr)); // IPv4
inet_pton(AF_INET6, " 2001:db8:63b3:1::3490" , &(sa6.sin6_addr)); // IPv6
```

目前上述的代码片段还不是很可靠，因为没有错误检查。`inet_pton()`在错误时会返回-1，而若地址被搞砸了，则会返回0。所以在使用之前要检查，并确认结果是大于0的。

``` C
// 你有一个 struct in_addr 且你想要以数字与句号的字符串格式打印出来

// IPv4:

char ip4[INET_ADDRSTRLEN]; // 储存 IPv4 字符串的空间
struct sockaddr_in sa; // pretend this is loaded with something
inet_ntop(AF_INET, &(sa.sin_addr), ip4, INET_ADDRSTRLEN);
printf(" The IPv4 address is: %s\n" , ip4);

// IPv6:

char ip6[INET6_ADDRSTRLEN]; // 储存 IPv6 字符串的空间
struct sockaddr_in6 sa6; // pretend this is loaded with something
inet_ntop(AF_INET6, &(sa6.sin6_addr), ip6, INET6_ADDRSTRLEN);
printf(" The address is: %s\n" , ip6);
```

## 私有网络

防火墙会用所谓的网路地址转换（NAT，Network Address Translation）的方法，将内部的 IP 地址转换为外部的 IP address。

10.x.x.x 是其中一个少数保留的网路，只能用在完全无法连上 Internet 的网路，或是在防火墙的网路。一般而言，你较常见的是 10.x.x.x 及 192.168.x.x，这里的 x 是指 0-255。较少见的是 172.y.x.x，这里的 y 范围在 16 与 31 之间。

# System Call

## getaddrinfo() - 准备开始

它前身是你用来做 DNS 查询的`gethostbyname()`，你需要手动将资料写入`struct sockaddr_in`，并在你的调用中使用。现在有`getaddrinfo()`函数，可以帮你做许多事情，包含 DNS 与 service name 查询，并填好你所需的structs。

``` C
// 函数原型

#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *node, // 例如： " www.example.com" 或 IP
const char *service, // 例如： " http" 或 port number
const struct addrinfo *hints,
struct addrinfo **res);
```

`hints`参数指向一个你已经填好相关资料的`struct addrinfo`。下面是一个调用示例， 如果你是一个 server， 想要在你主机上的 IP address 及 port 3490 运行 listen。

``` C
int status;
struct addrinfo hints;
struct addrinfo *servinfo; // 将指向结果

memset(&hints, 0, sizeof hints); // 确保 struct 为空
hints.ai_family = AF_UNSPEC; // 不用管是 IPv4 或 IPv6
hints.ai_socktype = SOCK_STREAM; // TCP stream sockets
hints.ai_flags = AI_PASSIVE; // 帮我填好我的 IP

if ((status = getaddrinfo(NULL, " 3490" , &hints, &servinfo)) != 0) {
    fprintf(stderr, " getaddrinfo error: %s\n" , gai_strerror(status));
    exit(1);
}

// servinfo 目前指向一个或多个 struct addrinfos 的链表

// ... 做每件事情， 一直到你不再需要 servinfo ....

freeaddrinfo(servinfo); // 释放这个链表
```

在这里看到`AI_PASSIVE`flag；这个会告诉 `getaddrinfo()`要将我本地端的地址指定给 socket structure。 这样你就不用写固定的地址了，或者你可以将特定的地址放在`getaddrinfo() `的第一个参数中，现在写 NULL 的那个参数。

``` C
/*
** showip.c -- 显示命令行中所给的主机 IP address
*/

#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <netinet/in.h>

int main(int argc, char *argv[])
{
    struct addrinfo hints, *res, *p;
    int status;
    char ipstr[INET6_ADDRSTRLEN];

    if (argc != 2) {
        fprintf(stderr," usage: showip hostname\n" );
        return 1;
    }

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC; // AF_INET 或 AF_INET6 可以指定版本
    hints.ai_socktype = SOCK_STREAM;

    if ((status = getaddrinfo(argv[1], NULL, &hints, &res)) != 0) {
        fprintf(stderr, " getaddrinfo: %s\n" , gai_strerror(status));
        return 2;
    }

    printf(" IP addresses for %s:\n\n" , argv[1]);

    for(p = res;p != NULL; p = p->ai_next) {
        void *addr;
        char *ipver;

        // 取得本身地址的指针，
        // 在 IPv4 与 IPv6 中的栏位不同：
        if (p->ai_family == AF_INET) { // IPv4
            struct sockaddr_in *ipv4 = (struct sockaddr_in *)p->ai_addr;
            addr = &(ipv4->sin_addr);
            ipver = " IPv4" ;
        } else { // IPv6
            struct sockaddr_in6 *ipv6 = (struct sockaddr_in6 *)p->ai_addr;
            addr = &(ipv6->sin6_addr);
            ipver = " IPv6" ;
        }
        // convert the IP to a string and print it:
        inet_ntop(p->ai_family, addr, ipstr, sizeof ipstr);
        printf(" %s: %s\n" , ipver, ipstr);

    }
    
    freeaddrinfo(res); // 释放链表

    return 0;
}
```

## socket() - 取得 File Descriptor

``` C
// 函数原型

#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

`domain`是`PF_INET`或`PF_INET6`（在`struct sockaddr_in`中使用`AF_INET`，而在调用`socket()`时使用`PF_INET`）；`type`是`SOCK_STREAM`或`SOCK_DGRAM`；而`protocol`可以设置为0，用来帮给予的`type`选择适当的协议。 或者你可以调用 getprotobyname() 来查询你想要的协议，＂tcp＂或＂udp＂。

``` C
int s;
struct addrinfo hints, *res;

// 运行查询，假装我们已经填好 " hints" struct
getaddrinfo(" www.example.com" , " http" , &hints, &res);

// 你应该要对 getaddrinfo() 进行错误检查, 并走到 " res" 链表查询能用的资料，而不是假设第一笔资料就是好的
s = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
```

`socket()`单纯返回一个之后的 system call 要用的 socket descriptor 给你，错误时会返回 -1，errno 全局变量会设置为该错误的值。

## bind() － 绑定端口

port number 是用来让 kernel 可以比对出进入的数据包是属于哪个 process 的 socket descriptor。一个 process 可以有多个 socket fd，绑定多个 port number。

``` C
// 函数原型

#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
```

``` C
struct addrinfo hints, *res;
int sockfd;

// 首先， 用 getaddrinfo() 载入地址结构：

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC; // use IPv4 or IPv6, whichever
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE; // fill in my IP for me

getaddrinfo(NULL, " 3490" , &hints, &res);

// 建立一个 socket：

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// 将 socket bind 到我们传递给 getaddrinfo() 的 port：

bind(sockfd, res->ai_addr, res->ai_addrlen);
```

bind() 在错误时也会返回 -1， 并将 errno 设置为该错误的值。

``` C
// !!! 这 是 老 方 法 !!!

// 许多旧程序都在调用 bind() 以前手动封装 struct sockaddr_in

int sockfd;
struct sockaddr_in my_addr;

sockfd = socket(PF_INET, SOCK_STREAM, 0);

my_addr.sin_family = AF_INET;
my_addr.sin_port = htons(MYPORT); // short, network byte order
my_addr.sin_addr.s_addr = inet_addr(" 10.12.110.57" );
memset(my_addr.sin_zero, '\0', sizeof my_addr.sin_zero);

bind(sockfd, (struct sockaddr *)&my_addr, sizeof my_addr);
```

在上列的代码中， 如果你想要 bind 到你本地端的 IP address（就像上面的`AI_PASSIVE` flag），你也可以将`INADDR_ANY`指定给`s_addr`栏位。`INADDR_ANY`的 IPv6 版本是一个`in6addr_any`全局变量，它会被指定给你的`struct sockaddr_in6`的`sin6_addr`栏位。

`INADDR_ANY`转换过来就是 0.0.0.0，泛指本机的意思，也就是表示本机的所有 IP，因为有些机子不止一块网卡，多网卡的情况下，这个就表示所有网卡 IP 地址的意思。所以出现`INADDR_ANY`，你只需绑定`INADDR_ANY`，管理一个套接字就行，不管数据是从哪个网卡过来的，只要是绑定的端口号过来的数据，都可以接收到。

你可能有注意到，有时候你试着重新运行 server，而`bind()`却失败了，它声称＂Address already in use.＂。有些连接到 socket 的连接还悬在 kernel 里面，而它占据了这个 port。你可以等待它自行清除（一分钟左右），或者在你的程序中新增代码，让它重新使用这个 port，类似这样：

``` C
int yes=1;

// 可以跳过 " Address already in use" 错误信息

if (setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&yes,sizeof(int)) == -1) {
    perror(" setsockopt" );
    exit(1);
}
```

若你正使用`connect()`连接到远端的机器，你可以不用管 local port 是多少，你可以单纯地调用`connect()`，它会检查 socket 是否尚未绑定， 并在有需要的时候自动将 socket `bind()`到一个尚未使用的 local port。

## connect() - 嘿！你好

``` C
// 函数原型

#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
```

``` C
struct addrinfo hints, *res;
int sockfd;

// 首先， 用 getaddrinfo() 载入 address structs：

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo(" www.example.com" , " 3490" , &hints, &res);

// 建立一个 socket：

sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

// connect!

connect(sockfd, res->ai_addr, res->ai_addrlen);
```

要确定有检查`connect()`返回的值，它在错误时会返回 -1，并设定 errno 变量。

## listen() - 有人会调用我吗

``` C
// 函数原型

int listen(int sockfd, int backlog);
```

`backlog`是进入的队列（incoming queue）中所允许的连接数目。如同往常，`listen()`会返回 -1 并在错误时设置 errno。

``` C
如果你正在 listen 进入的连接， 你会运行的 system call 顺序如下

getaddrinfo();
socket();
bind();
listen();
/* accept() 从这里开始 */
```

## accept() - 谢谢你的调用

很远的人会试着`connect()`到你的电脑正在`listen()`的 port。他们的连接会排队等待被`accept()`。你调用`accept()`，并告诉它要取得搁置的（pending）连接。它会返回专属这个连接的一个新 socket file descriptor 给你！你突然有了两个 socket file descriptor！原本的 socket file descriptor 仍然正在 listen 之后的连线，而新建立的 socket file descriptor 则是在最後要准备给`send()`与`recv()`用的。

``` C
// 函数原型

#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

sockfd 是正在进行`listen()`的 socket descriptor。很简单，`addr`通常是一个指向 local struct
sockaddr_storage 的指针，是关于进来的连接将往哪里去的资料，你可以用它来得知是哪一台主机从哪一个 port 调用你的。`addrlen`是一个 local 的整数变量，应该在将它的地址传递给`accept()`以前， 将它设置为`sizeof(struct sockaddr_storage)`。

``` C
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define MYPORT " 3490" // 使用者将连接的 port
#define BACKLOG 10 // 在队列中可以有多少个连接在等待

int main(void)
{
    struct sockaddr_storage their_addr;
    socklen_t addr_size;
    struct addrinfo hints, *res;
    int sockfd, new_fd;

    // !! 不要忘了帮这些调用做错误检查 !!

    // 首先， 使用 getaddrinfo() 载入 address struct：

    memset(&hints, 0, sizeof hints);
    hints.ai_family = AF_UNSPEC; // 使用 IPv4 或 IPv6， 都可以
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE; // 帮我填上我的 IP

    getaddrinfo(NULL, MYPORT, &hints, &res);

    // 产生一个 socket， bind socket， 并 listen socket：

    sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
    bind(sockfd, res->ai_addr, res->ai_addrlen);
    listen(sockfd, BACKLOG);

    // 现在接受一个进入的连接：

    addr_size = sizeof their_addr;
    new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &addr_size);

    // 准备好与 new_fd 这个 socket descriptor 进行沟通！

    .
    .
    .
```

若你只是要取得一个连接，你可以用`close()`关闭正在 listen 的 sockfd，以避免有更多的连接进入同一个 port， 若你有这个需要的话。

## send() 与 recv() － 跟我说说话

这两个用来通讯的函数是透过 stream socket 或 connected datagram socket。若你想要使用常规的 unconnected datagram socket，你会需要参考底下的 `sendto()`及`recvfrom()`的章节。

``` C
// 函数原型

int send(int sockfd, const void *msg, int len, int flags); // flags设置0就好了
```

`send()`会返回实际有送出的 byte 数目，这可能会少于你所要传送的数目！有时候你告诉`send()`要送整笔的资料，而它就是无法处理这麽多资料。它只会尽量将资料送出，并认为你之后会再次送出剩下没送出的部分。要记住，如果`send()`返回的值与 len 的值不符合的话，你就需要再送出字串剩下的部分。

``` C
// 函数原型

int recv(int sockfd, void *buf, int len, int flags);
```

`recv()`返回实际读到并写入到缓冲区的 byte 数目，而错误时返回 -1。`recv()`会返回 0，这只能表示一件事情：远端那边已经关闭了你的连接！

## sendto() 与 recvfrom() － 用 DGRAM 风格跟我说说话

``` C
// 函数原型

sendto(int sockfd, const void *msg, int len, unsigned int flags, const struct sockaddr *to, socklen_t tolen);
```

`tolen`是一个 int，可以单纯地将它设置为`sizeof *to`或`sizeof(struct sockaddr_storage)`。（`sizeof`对于变量不用加括号，对于类型需要加括号）

为什麽我们要用`struct sockaddr_storage`做为 socket 的型别呢？为什麽不用`struct sockaddr_in`呢？因为我们不想要让自己绑在 IPv4 或 IPv6， 所以我们使用通用的泛型`struct sockaddr_storage`，我们知道这样有足够的空间可以用在 IPv4 与 IPv6。

记住，如果你`connect()`到一个 datagram socket，你可以在你全部的交易中只使用`send()`与`recv()`。socket本身仍然是 datagram socket，数据包仍然使用 UDP，但是 socket interface 会自动帮你增加目的与来源资料。

## close() 与 shutdown() － 从我面前消失吧

``` C
//函数原型

close(sockfd);
int shutdown(int sockfd, int how);
```

如果你想要能多点控制 socket 如何关闭，可以使用 `shutdown()`函数。它让你可以切断单向的通信，或者双向（就像是`close()`所做的）。0 不允许再接收数据；1 不允许再传送数据；2 不允许再传送与接收数据。

重要的是`shutdown()`实际上没有关闭 file descriptor，它只是改变了它的可用性。如果要释放 socket descriptor， 你还是需要使用`close()`。

若你在 unconnected datagram socket 上使用`shutdown()`，它只会单纯的让 socket 无法再进行`send()`与`recv()`调用；要记住你只能在有`connect()`到 datagram socket 的时候使用。

## getpeername() － 你是谁

``` C
// 函数原型

#include <sys/socket.h>

int getpeername(int sockfd, struct sockaddr *addr, int *addrlen);
```

`getpeername()`函数会告诉你另一端连接的 stream socket 是谁。一旦你取得了它们的地址，你就可以用 `inet_ntop()`、`getnameinfo()`或`gethostbyaddr()`取得更多的资料。

## gethostname() － 我是谁

``` C
// 函数原型

#include <unistd.h>

int gethostname(char *hostname, size_t size);
```

比`getpeername()`更简单的函数是`gethostname()`，它会返回你运行程序的电脑名，这个名称之后可以用在`gethostbyname()`，用来定义你本地端电脑的 IP 地址。

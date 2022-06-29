---
title: "Linux套接字socket入门笔记"
date: 2021-03-13T11:22:39+08:00
draft: false
toc: true
---

套接字`socket`是通信端点的抽象。正如使用文件描述符访问文件，应用程序使用套接字描述符访问套接字。套接字描述符在Unix系统中被当作是一种文件描述符。事实上，许多处理文件描述符的函数（如read和write）都可以用于处理套接字描述符。

# 创建套接字

创建一个套接字，调用`socket`函数：

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
// 返回值：成功，返回套接字描述符；出错，返回-1
```

参数：
- domain

  domain用于确定通信的特征，值一般都是以`AF_`开头的宏，最常使用的就是如下3种：

  | 域       | 描述         |
  | -------- | ------------ |
  | AF_INET  | IPV4英特网域 |
  | AF_INET6 | IPV6英特网域 |
  | AF_UNIX  | UNINX域      |

- type

  type用于确定套接字的类型，进一步确定通信的特征。可以选择的值一般是以`SOCK_`开头的宏，最常使用的就是如下几种：

  | 类型        | 描述                                              |
  | ----------- | ------------------------------------------------- |
  | SOCK_DGRAM  | UDP: 固定长度的，无连接的，不可靠的报文传输       |
  | SOCK_STREAM | TCP: 有序的，可靠的，双向的，面向连接的字节流传输 |

- protocol

  参数protocol一般为0，让操作系统自动选择协议即可。如果domain和type确定的情况下，protocol仍然有多个可以选择的值再考虑使用参数protocol选择一个特定的协议。

# 将套接字与地址关联

服务端的socket需要绑定到一个地址。使用`bind`函数：

```c
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 成功返回0，出错返回-1
```

对于使用的地址有以下的一些限制：

1. 在进程正在运行的计算机上，指定的地址必须有效，不能指定为一个其他机器的地址；
2. 地址必须和创建套接字时地址簇所支持的格式相匹配；
3. 只有进程的用户是root时，端口号port能小于1024，非root用户只能使用大于等于1024的端口；
4. 一般只能将一个套接字端点绑定到一个给定的地址上，尽管有些协议允许多重绑定。

参数：

- sockfd

  这个参数是创建套接字的函数socket返回的套接字描述符

- addr

  要将socket绑定到的地址，`struct sockaddr`的定义如下：

  ```c
  struct sockaddr {
      sa_family_t sa_family;
      char        sa_data[14];
  }
  ```
  > 定义在头文件`sys/socket.h`

- addrlen

  这个参数的作用说明起来涉及到的内容有点多：

  在具体调用bind时，虽然bind的签名规定了第二个参数addr是指向`struct sockaddr`结构的指针，但是`struct sockaddr`一个“通用结构”，实际具体调用bind时使用的地址结构一般不是`struct sockaddr`结构，比如绑定到一个(ipv4, port),结构是：

  ```c
  struct sockaddr_in {
      sa_family_t    sa_family;
      in_port_t      sin_port;
      struct in_addr sin_addr;
      unsigned_char  sin_zero[8];
  }
  ```
  > 定义在头文件`arpa/inet.h`

  那么调用bind时就要将指向`struct sockaddr_in`结构的指针强制转为指向`struct sockaddr`结构的指针，代码表示起来就是：

  ```c
  struct sockaddr_in ipv4_addr;
  // 为ipv4_addr各个属性赋值
  // 调用bind时强制转化指针
  bind(sockfd, (struct sockaddr*)&ipv4_addr, sizeof(ipv4_addr));
  ```

  因为强制转化指针后，原来结构的内存大小信息就丢失了，所以用`addrlen`这个参数传递，如上面代码用`sizeof`求了原来结构的大小。

# 服务端监听客户端连接

服务端用`listen`函数来宣告它愿意接收客户端连接请求：

```c
#include <sys/socket.h>

int listen(int sockfd, int backlog);
// 返回值：成功，返回0；出错，返回-1
```

参数：

- sockfd

  套接字描述符

- backlog

  等待队列的长度。比如设置为5，那么如果此时有5个客户端发来了建立连接请求都还没有被accept，此时第6个请求就会被直接拒绝。

# 服务端接收客户端连接

服务端接收客户端连接要用到函数`accept`

```c
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// 返回值：成功，返回用于和客户端通信的文件描述符；出错，返回-1
```

如果不关心客户端的标识，可以将addr和addrlen都设置为NULL。

如果没有连接请求在等待队列中等待，`accept`会阻塞。

# 客户端主动建立

客户端主动去连接服务端，需要用到`connect`函数：

```c
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 返回值：成功，返回0；出错，返回-1
```

函数的参数和`bind`函数一样，意义也是一样的。就不多做说明了。

# 发送和接收数据

既然一个套接字端点表示为一个文件描述符，那么只要建立起连接，就可以使用`read`和`write`函数来通过套接字通信。

虽然使用`read`和`write`可以实现2个socket交换数据，但是网络传输和普通文件读写还是有不一样的地方，因此也有专门用于socket交换数据的函数，一共有6个，用于发送和接收的各3个。

3个发送数据的函数：`send`，`sendto`，`sendmsg`

```c
#include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t len, int flags);

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```

3个用于接收数据的函数：`recv`，`recvfrom`，`recvmsg`

```c
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);

ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

# 一个简单的例子程序

## 服务端

```c
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

/*
初始化一个TCP或者UDP服务
*/
int init_server(int type, const struct sockaddr *addr, socklen_t alen, int qlen) {
    int fd = socket(addr->sa_family, type, 0);
    if (fd < 0) {
        return -1;
    }
    if (bind(fd, addr, alen) < 0 || (type == SOCK_STREAM && listen(fd, qlen) < 0)) {
        return -1;
    }
    return fd;
}

void serve(int server_fd) {
    for (;;) {
        int client_fd = accept(server_fd, NULL, NULL);
        if (client_fd < 0) {
            continue;
        }
        char *message = "Hello, Welcome!!\n";
        write(client_fd, message, strlen(message));
        char buf[1024];
        int read_count = read(client_fd, buf, 1024);
        if (read_count > 0) {
            write(STDOUT_FILENO, buf, read_count);
        }
        close(client_fd);
    }
}

int main(void) {
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htons(INADDR_ANY);
    server_addr.sin_port = htons(8080);
    // 监听本地：0.0.0.0:8080
    int server_socket_fd = init_server(SOCK_STREAM, (struct sockaddr *)&server_addr, sizeof(server_addr), 10);
    serve(server_socket_fd);
    return 0;
}
```

## 客户端

---
title: "epoll学习笔记"
date: 2021-04-06T11:22:39+08:00
draft: false
toc: true
---

# 系统调用学习

## epoll_create, epoll_create1

系统调用签名：

```c
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_create1(int flags);

// 返回-1表示失败，成功则是一个正整数
```

系统调用`epoll_create`会创建一个`epoll`实例，返回一个int类型的文件描述符，指代对应的`epoll`实例。Linux 2.6.8版本以后，参数`size`没有作用了，会被忽略，但是还是要为一个正数，如果是0或者负数，函数返回-1，创建`epoll`失败。

`epoll_create1(0)`的效果和`epoll_create(1)`一样，所以一般就没有必要调用epoll_create了，可以忘记其存在（不过看到旧代码要知道是什么意思）。当flags不为0时，有额外的作用。

返回的文件符不需要再使用后，用系统调用`close`关闭。

```c
#include <unistd.h>

int close(int fd);
```

## epoll_ctl

系统调用签名:

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 返回-1表示失败
// 返回0表示成功
```

这个系统调用用于新增，修改，或者删除由文件描述符`epfd`关联的`epoll`实例。

- epfd: 文件描述符，代表相关的`epoll`实例
- fd: 要执行IO操作的文件描述符，比如文件，socket等
- op: 操作

    有以下几个可选值：
    - EPOLL_CTL_ADD: 新增一个让`epfd`关联的`epoll`实例“管理”的文件描述符`fd`，关注的事件由`event`定义
    - EPOLL_CTL_MOD: 修改已经被`epfd`关联的`epoll`实例“管理”的文件描述符`fd`，关注事件的修改由`event`定义
    - EPOLL_CTL_DEL: 删除已经被`epfd`关联的`epoll`实例“管理”的文件描述符`fd`，既然是删除，`event`参数会被忽略，可以为NULL

- event:
    ```c
    typedef union epoll_data {
        void        *ptr;
        int          fd;
        uint32_t     u32;
        uint64_t     u64;
    } epoll_data_t;

    struct epoll_event {
        uint32_t     events;      /* Epoll events */
        epoll_data_t data;        /* User data variable */
    };
    ```

    `events`是一个`uint32_t`
    - EPOLLIN: 读
    - EPOLLOUT: 写

## epoll_wait, epoll_pwait

系统调用签名:

```c
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout, const sigset_t *sigmask);
```

系统调用`epoll_wait`等待由文件描述符`epfd`关联的epoll实例上的IO事件，调用返回后，指针`events`指向的epoll_event实例包含发生的事件，`maxevents`是返回的最大事件数，必须大于0。参数`timeout`表示的是最长阻塞时间，单位是毫秒。

系统调用`epoll_pwait`暂不深究，当`sigmask`为`NULL`时和`epoll_pwait`等价。

# 一个简单的使用例子

```c
#include <sys/epoll.h>
#include <stdio.h>

int main(int argc, char **argv) {
    int max_events = 10;
    struct epoll_event ev, events[max_events];
    int listen_sock, conn_sock, nfds, epoll_fd;

    epoll_fd = epoll_create1(0);
    if (epollfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    // TODO : open server socket

    ev.events = EPOLLIN;
    ev.data.fd = listen_sock;

    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
        perror("epoll_ctl: listen_sock");
        exit(EXIT_FAILURE);
    }

    // TODO : 继续

    return 0;
}
```

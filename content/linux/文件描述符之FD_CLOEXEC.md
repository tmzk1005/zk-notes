---
title: "文件描述符之FD_CLOEXEC"
date: 2021-03-14T11:22:39+08:00
draft: false
---

> <<Unix环境高级编程>>阅读笔记

在服务器进程中调用`fork`然后`exec`执行另外一个程序来向客户端进程提供服务是很常见的。如果服务器进程打开着一些文件描述符，比如服务器打开了某个文件，或者监听着某个网络地址等。如果服务器进程不希望这些文件描述符在exec执行的程序中能访问到，那么就要在fork之后exec之前，用`close`挨个关闭这些文件描述符。如果要关系的文件描述符多了，很容易遗漏。有一个简单的解决此问题的方法，就是给文件描述符设置**FD_CLOEXEC**标志。这样，当调用`fork`然后`exec`执行另外一个程序时，在exec执行的程序中，设置了**FD_CLOEXEC**标志的文件描述符就被自动的关闭了，不能再被访问。

一个设置FD_CLOEXEC标志的函数：

```c
#include <fcntl.h>

int set_close_exec(int fd) {
    int val;
    if ((val = fcntl(fd, F_GETFD, 0)) < 0) {
        return -1;
    }
    val |= FD_CLOEXEC;
    return fcntl(fd, F_SETFD, 0);
}
```

---
title: "linux多线程入门"
date: 2021-04-26T20:22:39+08:00
draft: true
toc: true
---

因为想学习hotspot的源码，需要了解下linux如何进行简单的多线程开发，特此学习并记录入门笔记。

# 线程创建

创建线程的系统调用是`pthread_create`:

```c
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);

// 返回值：成功则返回0，错误则返回一个正数表示的错误编号
```

这个调用有4个参数：

- pthread_t *thread

  存放创建成功的线程id

- const pthread_attr_t *attr

  用于配置一些线程的属性，无定义属性需求可以传NULL

- void *(*start_routine) (void *)

  子线程里要执行的函数，函数没有返回值，但是参数是一个(void *)，表示可以指向任何类型

- void *arg

  传给start_routine函数的参数，可以是任何类型


一个简单的用2个线程累加的例子程序：

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

int num = 0;

void printids(const char *s) {
    pid_t pid;
    pthread_t tid;
    pid = getpid();
    tid = pthread_self();
    printf("%s, pid = %lu, thread_id = %lu\n", s, (unsigned long)pid, (unsigned long)tid);
}

void *increment_num(void *arg) {
    printids("sub");
    unsigned long c = (unsigned long)arg;
    for (int i = 0; i < c; ++i) {
        num += 1;
    }
    return (void *)NULL;
}

pthread_t increment_num_in_sub_thread(long c) {
    int err;
    pthread_t thread_id;
    err = pthread_create(&thread_id, NULL, increment_num, (void *)c);
    if (err != 0) {
        printf("Failed to create thread!");
    }
    return thread_id;
}

int main(void) {
    printids("main");
    pthread_t sub_thread1 = increment_num_in_sub_thread(10000000);
    pthread_t sub_thread2 = increment_num_in_sub_thread(10000000);
    pthread_join(sub_thread1, NULL);
    pthread_join(sub_thread2, NULL);
    printf("num = %d\n", num);
    return 0;
}
```

这个例子顺便演示了+1操作不是原子的，结果比预期的少很多。

# 线程退出

# 线程同步


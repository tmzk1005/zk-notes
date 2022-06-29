---
title: "理解Netty中的NioEventLoopGroup"
date: 2021-04-19T10:00:00+08:00
draft: false
tags: ["java"]
toc: true
---

在学习Netty的时候，首先接触到的一个类就是`NioEventLoopGroup`，它确实是Netty高性能的关键。因此特地系统的通过debug源码的方式学习了其原理。

# NioEventLoopGroup的表面使用

Netty的入门Guide通过ServerBootstrap写了一个简单的TCP服务，在不用ServerBootstrap的前提下NioEventLoopGroup到底提供了什么能力呢？

首先，NioEventLoopGroup是一个线程池，它是实现了`ExecutorService`接口的，既然是线程池，那就可以提交任务给它执行：

```java
import io.netty.channel.nio.NioEventLoopGroup;

public class TestNioEventLoopGroup {

    public static void main(String[] args) {
        NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
        nioEventLoopGroup.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " : Hello");
        });
    }

}
```

显然，上面的代码会输出一个线程名字和Hello，那它内部是如何执行的，和普通的线程池有什么区别呢？

上面的代码只有2个动作：

1. 构造函数
2. execute一个任务

## 构造函数

首先来看看构造函数做了什么，`NioEventLoopGroup`继承了`MultithreadEventExecutorGroup`，一路追踪，`new NioEventLoopGroup()`最后是落脚在了`MultithreadEventExecutorGroup`的构造函数：

```java
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}
```

```java
protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (threadFactory == null) {
        threadFactory = newDefaultThreadFactory();
    }

    children = new SingleThreadEventExecutor[nThreads];
    if (isPowerOfTwo(children.length)) {
        chooser = new PowerOfTwoEventExecutorChooser();
    } else {
        chooser = new GenericEventExecutorChooser();
    }

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(threadFactory, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }
```

nThreads是cpu逻辑核心数*2， 如果电脑有4个核心，那么nThreads就是8。

上面的构造函数的核心逻辑就是初始化了`children`数组，可以看到数组元素的类型是`SingleThreadEventExecutor`，另外，还初始化了一个`chooser`，这个`chooser`s其实即是一个在`children`数组里选择一个元素出来的策略，它在后面执行任务的时候会用到。

## 执行任务

`execute`方法的实现：

```java
public void execute(Runnable command) {
    next().execute(command);
}
```

而`next`函数的实现是：

```java
public EventExecutor next() {
    return chooser.next();
}
```

它其实就是在前面说的`children`数组中按照某种策略选择出一个元素，然后调用其`execute`方法。

## 总结

所以，现在对`NioEventLoopGroup`可以有如下的简单认识：

1. `NioEventLoopGroup`有一个`children`数组，数组元素类型是`SingleThreadEventExecutor`
2. 提交任务到`NioEventLoopGroup`执行，其实就是把任务交给了`NioEventLoopGroup`背后管理的一群`SingleThreadEventExecutor`中的一个去执行。这也就是为什么它叫`Group`

那么我们的重点研究对象就转到了`SingleThreadEventExecutor`，看看它到底是如何执行分给它的任务的。

# SingleThreadEventExecutor

首先，`SingleThreadEventExecutor`能执行提交给他的任务，它其实也是个线程池，只不过是一个只有一个线程的线程池。

`SingleThreadEventExecutor`是一个抽象类，它只有一个抽象方法：

```java
protected abstract void run();
```

我们可以简单实现一个`SingleThreadEventExecutor`的子类：

```java
import io.netty.util.concurrent.EventExecutorGroup;
import io.netty.util.concurrent.SingleThreadEventExecutor;
import java.util.concurrent.ThreadFactory;

public class MyExecutor extends SingleThreadEventExecutor {

    protected MyExecutor(EventExecutorGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp) {
        super(parent, threadFactory, addTaskWakesUp);
    }

    @Override
    protected void run() {
        System.out.println("Hello, MyExecutor");
    }

}
```

然后测试，看提交一个任务给`MyExecutor`会发生什么：

```java
import io.netty.util.concurrent.DefaultThreadFactory;
import io.netty.util.concurrent.SingleThreadEventExecutor;

public class App {
    public static void main(String[] args) {
        SingleThreadEventExecutor stet = new MyExecutor(null, new DefaultThreadFactory("test"), true);
        stet.execute(() -> {
            System.out.println("run task1");
        });
    }
}
```

执行，会发现先输出"Hello, MyExecutor"，然后输出"run task1"，说明先执行了MyExecutor的run方法，然后才执行的提交的任务的run方法。如果，在打印信息是加上当前线程的名字，会发现2行打印是同一个线程，并且不是主线程。

看看`execute`背后的实现：

```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

在main线程里调用`execute`方法，inEventLoop是`false`，因此会走else分支，执行`startThread()`方法：

```java
private void startThread() {
    if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            thread.start();
        }
    }
}
```

可以看出，第一次调用时，会启动一个线程，并记录下状态为已经启动，这样，第二次再调用就直接返回了。那启动的是个什么线程，它的run方法是什么样的呢？

这个thread就是`SingleThreadEventExecutor`这个只有一个线程的线程池所包含的那个线程了，它是在`SingleThreadEventExecutor`构造的时候初始化的：

```java
protected SingleThreadEventExecutor(
        EventExecutorGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp, int maxPendingTasks,
        RejectedExecutionHandler rejectedHandler) {
    if (threadFactory == null) {
        throw new NullPointerException("threadFactory");
    }

    this.parent = parent;
    this.addTaskWakesUp = addTaskWakesUp;

    thread = threadFactory.newThread(new Runnable() {
        @Override
        public void run() {
            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = STATE_UPDATER.get(SingleThreadEventExecutor.this);
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }
                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    logger.error(
                            "Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                            SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " +
                            "before run() implementation terminates.");
                }

                try {
                    // Run all remaining tasks and shutdown hooks.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }
                } finally {
                    try {
                        cleanup();
                    } finally {
                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.release();
                        if (!taskQueue.isEmpty()) {
                            logger.warn(
                                    "An event executor terminated with " +
                                    "non-empty task queue (" + taskQueue.size() + ')');
                        }

                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
    threadProperties = new DefaultThreadProperties(thread);
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    taskQueue = newTaskQueue();
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

可以看到这一行：

```java
SingleThreadEventExecutor.this.run();
```

就是在这里执行了`SingleThreadEventExecutor`的继承者也就是`MyExecutor`的`run`方法。这个`run`方法返回后，其实这个线程池就准备退出（shutdown）了，但是在这之前，它调用了`confirmShutdown()`，`confirmShutdown()`里又调用了`runAllTasks()`方法：

```java
protected boolean runAllTasks() {
    boolean fetchedAll;
    do {
        fetchedAll = fetchFromScheduledTaskQueue();
        Runnable task = pollTask();
        if (task == null) {
            return false;
        }

        for (;;) {
            try {
                task.run();
            } catch (Throwable t) {
                logger.warn("A task raised an exception.", t);
            }

            task = pollTask();
            if (task == null) {
                break;
            }
        }
    } while (!fetchedAll); // keep on processing until we fetched all scheduled tasks.

    lastExecutionTime = ScheduledFutureTask.nanoTime();
    return true;
}
```

可以看到，这个线程池在退出之前呢，会尝试把已经提交给它（在它的一个任务队列里）但是它还没来得及执行完的任务先执行完了后再退出，真是一个“负责任”的线程池。

从这段逻辑，可以看出2点：

1. 如果把前面的代码稍微改以下，在main线程里提交第一个任务后，main线程休眠一段时间，然后再提交第二个任务，这个时候线程池应该已经退出了，执行第二个任务会报异常：

```java
import io.netty.util.concurrent.DefaultThreadFactory;
import io.netty.util.concurrent.SingleThreadEventExecutor;

public class App {
    public static void main(String[] args) throws Exception {
        SingleThreadEventExecutor stet = new MyExecutor(null, new DefaultThreadFactory("test"), true);
        stet.execute(() -> {
            System.out.println("run task1");
        });
        Thread.sleep(1000);
        stet.execute(() -> {
            System.out.println("run task2");
        });
    }
}
```

会输出"run task1"，但是不会输出"run task2"，而是异常。（其实可能"run task1"都不会输出，如果main线程执行了构造就被剥夺了CPU）

2. 如果`MyExecutor`的run方法是个死循环，那么提交给`MyExecutor`的任务根本得不到执行的机会。

修改下代码验证：

```java
import io.netty.util.concurrent.EventExecutorGroup;
import io.netty.util.concurrent.SingleThreadEventExecutor;
import java.util.concurrent.ThreadFactory;

public class MyExecutor extends SingleThreadEventExecutor {

    protected MyExecutor(EventExecutorGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp) {
        super(parent, threadFactory, addTaskWakesUp);
    }

    @Override
    protected void run() {
        System.out.println("Hello, MyExecutor");
        long count = 0L;
        while (true) {
            System.out.println(++count);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                //
            }
        }
    }

}
```

再执行App类的main方法，可以看到，确实提交的任务，没有执行机会。那如果我就是要再run方法里死循环，同时可以执行提交的任务怎么办呢？前面执行提交的任务是调用了`runAllTasks()`方法，那么我们再run方法里执行`runAllTasks()`就可以了嘛：

```java
import io.netty.util.concurrent.EventExecutorGroup;
import io.netty.util.concurrent.SingleThreadEventExecutor;
import java.util.concurrent.ThreadFactory;

public class MyExecutor extends SingleThreadEventExecutor {

    protected MyExecutor(EventExecutorGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp) {
        super(parent, threadFactory, addTaskWakesUp);
    }

    @Override
    protected void run() {
        System.out.println("Hello, MyExecutor");
        while (true) {
            // Do something else
            runAllTasks();
        }
    }

}
```

比如上面的`// Do something else`就可以是对一个`Selector`调用其`select`方法，事实上`NioEventLoopGroup`的children数组里的类型是`NioEventLoop`，它就是`SingleThreadEventExecutor`的子类，`NioEventLoop`的run方法里就是一个死循环，同时也不停的调用`runAllTasks()`方法。

# NioEventLoop

接着上面的逻辑，我们的关注点转移到了`NioEventLoop`的`run`方法：

```java
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                    // fallthrough
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
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
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

显然`NioEventLoop`维护者一个IO多路复用器Selector,它是在其构造函数里实例化的：

```java
NioEventLoop(NioEventLoopGroup parent, ThreadFactory threadFactory, SelectorProvider selectorProvider,
                SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, threadFactory, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    selector = openSelector();
    selectStrategy = strategy;
}
```

通过上面的分析，可以看出`NioEventLoop`它即是一个IO多路复用器，同时也是一个只有一个线程的线程池，个人的理解，这个就是所谓的**EventLoop事件循环处理器**的内涵。通过`Selector`,可以注册自己关心的事件，当关注的事件类型发生时，自己又是线程池可以处理。

`NioEventLoopGroup`内含有多个`NioEventLoop`，也就含有多个`Selector`，而Netty网络编程时，一般还弄2个`NioEventLoopGroup`，一个boss，一个worker，也就是Netty内部又好多个`Selector`，好多个线程池，它们有内部的分工，有层次，通过分发（Dispatch，类似与SpringMVC的DispatcherServlet的思想）的思想组织为逻辑上的上下层次关系，有的只关注accept事件，有的只关注Read，Write事件，从而构成了所谓的**Reactor模式**


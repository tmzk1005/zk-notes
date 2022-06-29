---
title: "异步任务FutureTask和Future"
date: 2021-04-17T10:00:00+08:00
draft: false
tags: ["java"]
toc: true
---

# 原理梳理

Java中是如何实现异步任务的呢，逻辑原理其实比较简单，描述起来大概就是这样的：新建一个线程去执行一个任务，这个线程会记录任务的状态（未开始？执行中？正常已完成？异常？），并将最终的结果放在某个约定的地方，当主线程想要得到结果时，先询问状态，如果是已完成（正常或者异常），那就到约定的地方取出结果，未完成就等一会再询问。

逻辑上涉及的主体都是那些关联的类呢？

- 线程：Thread
- 任务：Callable
- 结果：Future

那记录的任务状态，以及约定的存放结果的地方呢？是一个叫`FutureTask`的类，它有一个`state`属性表示状态：

```java
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

然后有一个`outcome`属性表示结果：

```java
private Object outcome;
```

其实任务和任务的结果是逻辑强关联的，如果是2个类还需要用一个手段把它们关联起来。因此很自然的，用一个类，即表示这个任务，这个任务的执行状态和结果也由这个类来维护。这个类就是**FutureTask**。也就是说`FutureTask`即表示任务（是一个`Callable`），也表示任务的结果（是一个`Future`）。从`FutureTask`的类签名看确实如此：

```java
public class FutureTask<V> implements RunnableFuture<V> {
    // ...
}
```

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

从上面的类和接口看，`FutureTask`只是`Runnable`和`Future`的复合，没有`Callable`?，其实不然，有了`Runnable`，再加上从`Future`的泛型参数`V`知道了返回值的类型，就能确定`Callable`的细节，这个体现在`FutureTask`的构造函数：

```java
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

```java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

从一个`Runnable`和一个结果类型，或者直接从一个`Callable`都可以构造一个`FutureTask`，只不过前一种方式，内部把`Runnable`包装为`Callable`

# 例子演示

- 方式1:

  更加本质的方式：自己"亲自"新建一个`FutureTask`实例和线程：

```java
import java.util.concurrent.FutureTask;

public class TestFuture {

    public static void main(String[] args) throws Exception {
        FutureTask<Integer> task = new FutureTask<>(() -> {
            Thread.sleep(3000);
            return 100;
        });
        new Thread(task).start();
        Integer integer = task.get();
        System.out.println("result : " + integer);
    }

}
```

- 方式2:

  实际更加常用的方式，用线程池submit：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class TestFuture {

    public static void main(String[] args) throws Exception {
        ExecutorService service = Executors.newSingleThreadExecutor();
        Future<Integer> task = service.submit(() -> {
            Thread.sleep(3000);
            return 100;
        });
        System.out.println("result : " + task.get());
    }

}
```

# 逻辑分析

既然要**异步**，显然要在一个另外的线程去完成任务，要不然怎么异步得起来呢？那在这个线程里干了啥？那就要看`FutureTask`的`run`方法了：

```java
public void run() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

核心逻辑有3:
1. 状态判断，通过CAS的方式；
2. 执行`Callable`接口的`call`方法，拿到执行结果；
3. 设置结果，正常的话就是`set`，异常就是`setException`

再看看`set`的代码：

```java
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state
        finishCompletion();
    }
}
```

先通过CAS方式设置状态成功，再设置`outcome`的值。

这样，最终在主线程里调用`get`方法就可以拿到`outcome`的值了。

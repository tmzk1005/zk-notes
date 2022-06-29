---
title: "Java锁的原理"
date: 2020-12-20T11:22:39+08:00
draft: false
tags: ["java"]
toc: true
---

# 锁的理解

锁是用来做多线程同步的工具，其本质就是在进入临界区之前，做一个准入控制，只允许一个或者设定好的n个线程同时进入。所以锁的核心之一就是怎么做好这个准入控制。另一个核心问题就是，获取锁失败，也就是没有获准进入临界区怎么暂停执行并排队，以及其他已经获得锁的线程释放锁后，怎么退出排队并重新恢复执行。

假如要自己实现一个简单的锁工具，就要解决上面的2个问题。

先说排队问题，不考虑可用性而只要实现功能的话，可以根本就不排队。如果一个线程没有获得到锁的话，就sleep一段时间，等醒过来后再次尝试去获取锁，失败的话再sleep，一致循环，直到获取到锁。

而准入控制的思想很简单，就是用一个数字表示锁状态，为0的话，表示没有被占，大于0的话表示被占，所谓的加锁过程就是给这个状态数值加1，再释放锁的时候再把加的数减回去。

一个错误的锁工具代码：

```java

public class WrongLock {

    /**
     * 锁状态，0表示没有被占，大于0表示被占
     */
    private int state = 0;

    public void lock() {
        while (true) {
            if (tryAcquire(1)) {
                return;
            } else {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                }
            }
        }
    }

    public void unlock() {
        release(1);
    }

    private boolean tryAcquire(int count) {
        if (state == 0) {
            state += count;
            return true;
        }
        return false;
    }

    private boolean release(int count) {
        if (state < count) {
            return false;
        }
        state = state - count;
        return true;
    }

}

```

为什么说这个WrongLock工具锁是错误的，很显然，因为给state加1的动作，不是原子的。所以这个工具根本不能起到正确的准入控制作用。因此：**能够原子的修改准入状态是锁的基石**

如果我们把上面的代码稍作修改，用`synchronized`块把state的读取和修改动作做成原子的，那么这个锁工具还算“可正确的使用”。但是`synchronized`性能其实是比较差的，应该想办法追求更好的实现。同时，获取不到锁的线程是在死循环里通过sleep来等待的，如果sleep时间过长，实时性会受影响，在生产环境这是不可接受的，如果sleep时间过断的话，又导致线程频繁的sleep和被唤醒，影响性能。怎么解决这个问题呢？所示说，**如何控制好等待线程和唤醒也是锁的基石**


# Unsafe

要理解好Java锁的实现原理，就是要理清楚JDK是如何解决上面2个基石问题的。JDK解决这2个基石问题的背后都是一个叫**Unsafe**的类（全名sun.misc.Unsafe），这个类提供的API和可以解决高效原子操作以及高效的线程启停控制。这个类比较底层，大多数方法都是native的，连名字都叫做“不安全”，就是告诉我们不要轻易使用这个类，而且确实，在我们的应用层代码想要获取到这个类的一个实例都还都要费一番周折。


Unsafe类并发基础的基石API：

- `compareAndSwap`实现原子操作

compareAndSwap实际上是所谓的CAS算法实现的一系列的方法，包括但不限于以下的几个函数：

```java
public final boolean compareAndSwapInt(Object o, long offset, int expected, int x)；
public final boolean compareAndSwapLong(Object o, long offset, long expected, long x)；
public final boolean compareAndSwapObject(Object o, long offset, Object expected, Object x)；
```

因为在JDK实现锁的过程中要原子的修改对象的int属性，long属性，Object属性等，所以这里列觉了这几个方法。


- `park`和`unpark`控制线程恢复执行和挂起

`park`和`unpark`是`Unsafe`提供的2个函数：

```java
public void park(boolean isAbsolute, long time);
public void unpark(Object thread);
```

park可以让当前的线程阻塞，阻塞期间操作系统不会分配时间片给这个线程，直到阻塞返回。它有2个参数，当isAbsolute=true时，说明time表示的是一个单位为毫秒的Unix时间戳，如果我们想让线程挂起10秒种，那么time应该是当前的系统时间戳加上10*1000，如果time小于当前系统时间，会立即返回。当isAbsolute=flase时，如果time等于0，就会一直阻塞，直到其他线程运行时，以此线程为参数调用了unpark，如果time大于0，time是一个单位为毫微秒（即百万分之一秒）的休眠时间长度。

另外，除了可以通过参数控制可预期park何时返回的情形外，操作系统会可能会无理由的让park返回，所以我们不能以park何时返回100%可控为前提来编程，需要考虑到这种情况。

unpark就比较简单了，就是让park的线程从休眠中唤醒。

在说明JDK如何用这些API实现我们常用的ReentrantLock等这些锁的底层逻辑之前，先熟悉下Unsafe的API


## 得到Unsafe实例

Unsafe没有public的构造方法，也没有工厂方法，我们只能通过反射的方式获取实例。

TestUnsafe.java
```java
import java.lang.reflect.Field;

import sun.misc.Unsafe;

public class TestUnsafe {

    private static Unsafe unsafe;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (Exception e) {
            System.exit(-1);
        }
    }

}
```

## compareAndSwap

假设有这么2个简单的类：

Bar.java
```java
public class Bar {

    private int a;
    private int b;

    public Bar(int a, int b) {
        this.a = a;
        this.b = b;
    }

    public int getA() {
        return a;
    }

    public int getB() {
        return b;
    }

    @Override
    public String toString() {
        return String.format("[Bar a = %s b = %s]", a, b);
    }
}

```

Foo.java
```java
public class Foo {

    private volatile int a;
    private volatile long b;
    private String c;

    private volatile Bar bar;

    public int getA() {
        return a;
    }

    public long getB() {
        return b;
    }

    public String getC() {
        return c;
    }

    public Bar getBar() {
        return bar;
    }
}
```

Foo和Bar这2个类的属性都是私有的，而且没有public的set方法，Bar的存在只是为了演示Foo这个类有一个非基本类型的类属性，用来演示如何修改Foo中的Bar。

我们如果要修改一个Foo实例的b属性和bar属性怎么办呢？可以用反射的方式，获取到Field对象后，放开访问控制就可以了，但是这种方式不是原子的，在实现锁的时候关键逻辑处修改实例属性的动作必须是原子的才能保证锁的正确实现。那么怎么实现原子的修改属性呢。这就是Unsafe的compareAndSwap干的事了。


修改下前面的TestUnsafe类，演示相关API

TestUnsafe.java
```java
import java.lang.reflect.Field;

import sun.misc.Unsafe;

public class TestUnsafe {

    private static Unsafe unsafe;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (Exception e) {
            System.exit(-1);
        }
    }

    private static void showFoo(Foo foo) {
        System.out.println("foo.b = " + foo.getB());
        System.out.println("foo.bar = " + foo.getBar());
    }
    public static void main(String[] args) throws Exception {
        Foo foo = new Foo();
        showFoo(foo);
        Field b = Foo.class.getDeclaredField("b");
        Field bar = Foo.class.getDeclaredField("bar");
        long bOffset = unsafe.objectFieldOffset(b);
        long barOffset = unsafe.objectFieldOffset(bar);

        while (unsafe.compareAndSwapLong(foo, bOffset, 0L, 100L));
        while (unsafe.compareAndSwapObject(foo, barOffset, null, new Bar(8, 9)));

        showFoo(foo);
    }

}

```

执行compareAndSwap一般配合死循环来用，compareAndSwap需要4个参数，会返回一个设值失败还是成功的结果，如果失败的话，一般就要继续重试，可能是等待，可能是先干点别的，我们这里是啥也不干，就死循环直到成功。

1. 第1个参数是要修改的对象
2. 第2个参数是要修改的属性在修改的对象中的偏移地址，这个可以类比C语言中的结构体，每个属性具体结构体其实地址的偏移。获取这个偏移还是需要用到Unsafe的方法objectFieldOffset，这个方法的参数是一个java.lang.reflect.Field，通过反射获取就可以了。
3. 第3个参数是待修改的属性在修改之前的预期值
4. 第4个参数是要设置的新值。

如果执行compareAndSwap时发现当前的实际属性值和第3个值不是“==”的，那么就会返回失败结果，反之就会返回成功结果，而且这个过程是原子的。假设2个线程同时以一摸一样的参数执行compareAndSwap，而且第3个参数和第4个参数不等，那么只有一个会成功，另一个会失败。在实现锁工具是，成功的相当于获取到锁，去执行任务，失败相当与获取锁失败，会park自己，等成功获取锁的线程执行完任务后，再把值设置回去，同时unpark失败的线程，失败的线程继续循环，发现获取锁成功。


## park和unpark

park和unpark的API其实很简单，这里用一个简单的程序演示下：

TestPark.java
```java
import java.lang.reflect.Field;

import sun.misc.Unsafe;

public class TestPark {

    private static Unsafe unsafe;
    private static volatile boolean parked = false;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (Exception e) {
            System.exit(-1);
        }
    }

    static class Park implements Runnable {

        @Override
        public void run() {
            long start = System.currentTimeMillis();
            long end = start + 3 * 1000;
            long count = 0;
            while (System.currentTimeMillis() < end) {
                ++count;
                if (count % 1000 == 0) {
                    System.out.println(count);
                }
                if (count == 100000) {
                    parked = true;
                    System.out.println("park");
                    unsafe.park(false, 0);
                    System.out.println("park return");
                }
            }
            System.out.println("finish");
        }

    }

    private static boolean getParked() {
        return parked;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Park());
        thread.start();
        while (true) {
            if (getParked()) {
                Thread.sleep(2000);
                System.out.println("unpark");
                unsafe.unpark(thread);
                break;
            }
        }
    }

}

```

子线程数数字数到100000后park了自己，在这之前通过一个volatile的boolean值parked告诉主线程自己park了自己（在上面的代码里一定要用volatile类型），主线程知道子线程park自己后，自己sleep了2秒后unpark了子线程，于是现象就是子线程数数字的过程中断暂停了2秒。


# 实现自己的简单锁工具

了解了Unsafe的相关API后，去阅读JDK中的锁实现的核心类AQS：AbstractQueuedSynchronizer应该会更清晰些。但是AQS仍然很复制，为了追求性能，AQS写的比较晦涩，经常用`&&`操作符号将复杂的逻辑写在一句话中，而且AQS还会实现独占模式，共享模式，等等之类的一些其他功能，如果只关注上面提到的核心问题，通过自己写一个简易版的AQS会更容易理解些。简易版的AQS只实现JDK的AQS的功能子集，并且添加debug信息，易于调试分析程序逻辑。

JDK的AQS有好几个内部类，比如Node，为了将单个类尽量简单化，实现简易锁不使用内部类，而是都用独立的类，同时不像JDK将锁过程的设计抽象好几层，简易锁工具把逻辑都些在一个类MyLock中。

主要有4个类：

1. MyLock ： 锁工具类
2. Node   ： 用于实现等待队列的类
3. Logger ： 跟锁工具的实现无关，用于打印debug信息
4. MyLockTest ： 测试MyLock的效果的类


- Logger.java
```java
public class Logger {

    private static boolean debug = false;

    static void debug(String message) {
        if (debug) {
            System.out.println(Thread.currentThread().getName() + " debug : " + message);
        }
    }

}

```
要debug的话把debug改为true


- Node.java

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

public class Node {

    private static final VarHandle NEXT;
    private static final VarHandle PREV;
    private static final VarHandle THREAD;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
            PREV = l.findVarHandle(Node.class, "prev", Node.class);
            THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    private volatile Node prev;
    private volatile Node next;
    private volatile Thread thread;

    public Node getPrev() {
        Node p = prev;
        if (p == null) {
            throw new NullPointerException();
        }
        return p;
    }

    public Node(Thread thd) {
        THREAD.set(this, thd);
    }

    public Node() {

    }

    public void setPrev(Node p) {
        PREV.set(this, p);
    }

    public void setNext(Node n) {
        NEXT.set(this, n);
    }

    public void setThread(Thread t) {
        THREAD.set(this, t);
    }

    public Thread getThread() {
        return thread;
    }

    public Node getNext() {
        return next;
    }

    @Override
    public String toString() {
        // Just for print debug info
        String p = "null", n = "null", t = "null";
        if (prev != null) {
            p = "Node@" + prev.hashCode();
        }
        if (next != null) {
            n = "Node@" + next.hashCode();
        }
        if (thread != null) {
            t = thread.getName();
        }
        return String.format("[Node@%s prev=%s next=%s thread=%s]", this.hashCode(), p, n, t);
    }
}

```

说明一下：这里面用了`VarHandle`这个东西，这个其实就是用来干compareAndSwap操作的，这是java9新增的API，java8和以前的版本没有。这个东西可以取代一部分Unsafe的功能，毕竟Unsafe很底层，难用。JDK8及以前的源码里很多用Unsafe的地方在JDK9都改为用`VarHandle`来实现了，包括AQS，所以8和9的AQS实现的代码有些不一样。

- MyLock.java
```java
import java.lang.reflect.Field;
import java.util.concurrent.locks.AbstractOwnableSynchronizer;

import sun.misc.Unsafe;

public class MyLock extends AbstractOwnableSynchronizer {

    private static Unsafe unsafe;
    private static long HEAD_OFFSET;
    private static long TAIL_OFFSET;
    private static long STATE_OFFSET;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
            HEAD_OFFSET = unsafe.objectFieldOffset(MyLock.class.getDeclaredField("head"));
            Logger.debug("offset of head in MyLock is " + HEAD_OFFSET);
            TAIL_OFFSET = unsafe.objectFieldOffset(MyLock.class.getDeclaredField("tail"));
            Logger.debug("offset of tail in MyLock is " + TAIL_OFFSET);
            STATE_OFFSET = unsafe.objectFieldOffset(MyLock.class.getDeclaredField("state"));
            Logger.debug("offset of state in MyLock is " + STATE_OFFSET);
        } catch (Exception e) {
            System.exit(-1);
        }
    }

    private volatile Node head;
    private volatile Node tail;
    private volatile int state;

    public MyLock() {
        Logger.debug("新建MyLock实例， queueString ： " + queueToString());
    }

    private void acquire(int count) {
        if (tryAcquire(count)) {
            Logger.debug("获取锁成功 " + queueToString());
            return;
        }
        Logger.debug("获取锁失败");
        Node node = new Node(Thread.currentThread());
        Logger.debug("失败后开始尝试把自己添加到等待队列的队尾");
        while (true) {
            Node oldTail = tail;
            if (oldTail != null) {
                if (unsafe.compareAndSwapObject(this, TAIL_OFFSET, oldTail, node)) {
                    node.setPrev(oldTail);
                    oldTail.setNext(node);
                    Logger.debug("排队成功 " + queueToString());
                    break;
                }
                Logger.debug("排队失败，其他线程已抢先排入队尾，开始重新插入队尾");
                Logger.debug(queueToString());
            } else {
                Logger.debug("第一次开始有线程排队，初始化等待队列");
                if (unsafe.compareAndSwapObject(this, HEAD_OFFSET, null, new Node())) {
                    tail = head;
                    Logger.debug("初始化队列成功 " + queueToString());
                } else {
                    Logger.debug("初始化等待队列失败，再次尝试。");
                }
            }
        }

        boolean isInterrupted = false;

        Logger.debug("排队成功后开始等待,直到成为队首");

        try {
            while (true) {
                final Node p = node.getPrev();
                if (p == head) {
                    Logger.debug("发现自己已经排在队首，尝试获取锁  " + queueToString());
                    if (tryAcquire(count)) {
                        Logger.debug("设置新的队首");
                        head = node;
                        node.setThread(null);   // 不需要原子
                        node.setPrev(null);     // 不需要原子
                        p.setNext(null);   // 不需要原子， help GC
                        // 带着isInterrupted = false的状态退出死循环
                        Logger.debug("在队首，获取锁成功。" + queueToString());
                        break;
                    }
                    Logger.debug("在队首，但是获取锁失败 " + queueToString());
                }
                Logger.debug("休眠");
                unsafe.park(false, 0);  // 阻塞
                Logger.debug("被唤醒");
                isInterrupted |= Thread.interrupted();
            }
        } catch (Throwable t) {
            Logger.debug("异常发生");
            if (isInterrupted) {
                Logger.debug("等待期间虽然被唤醒，但是是被打断的非正常唤醒，上层并不知道等待期间被打断，因此自己再打断自己，让上层知道并处理。");
                Thread.currentThread().interrupt();
            }
            throw t;
        }

        if (isInterrupted) {
            Logger.debug("等待期间虽然被唤醒，但是是被打断的非正常唤醒，上层并不知道等待期间被打断，因此自己再打断自己，让上层知道并处理。");
            Thread.currentThread().interrupt();
        }

    }

    private final boolean release(int count) {
        if (tryRelease(count)) {
            if (head != null) {
                Node node = head.getNext();
                if (node != null) {
                    Logger.debug("释放锁，等待队列非空，唤醒队首等待者，被唤醒线程是：" + node.getThread().getName());
                    unsafe.unpark(node.getThread());
                }
            }
            Logger.debug("释放锁，无其他线程在等待锁");
            return true;
        }
        return false;
    }

    private boolean tryAcquire(int count) {
        final Thread current = Thread.currentThread();
        if (state == 0) {
            if (unsafe.compareAndSwapInt(this, STATE_OFFSET, 0, count)) {
                setExclusiveOwnerThread(current);
            }
            return true;
        } else if (current == getExclusiveOwnerThread()) {
            // 可重入
            Logger.debug("重入");
            int nextc = state + count;
            if (nextc < 0) {
                // 溢出
                throw new Error("Maximum lock count exceeded");
            }
            state = nextc;  // 上下文逻辑已保证同一时刻，只有一个线程会到这里来，复制操作无需保证原子性
            return true;
        }
        return false;
    }

    private boolean tryRelease(int count) {
        int c = state - count;
        if (Thread.currentThread() != getExclusiveOwnerThread()) {
            String err = String.format("IllegalMonitorState： currentThread %s, exclusiveThread : %s", Thread.currentThread().getName(), getExclusiveOwnerThread().getName());
            throw new RuntimeException(err);
        }
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        } else {
            Logger.debug("释放的是重入");
        }
        state = c;
        return free;
    }

    public void lock() {
        acquire(1);
    }

    public void unlock() {
        release(1);
    }

    private String queueToString() {
        // Just for print debug info
        StringBuilder sb = new StringBuilder();
        if (head == null) {
            sb.append("  Queue = null");
        } else {
            Node node = head;
            sb.append("  Queue = ");
            sb.append(node.toString());
            node = node.getNext();
            while (node != null) {
                sb.append(" --> " + node.toString());
                node = node.getNext();
            }
        }
        sb.append("     ");
        if (head != null) {
            sb.append("head : " +  head);
        } else {
            sb.append("head : " +  null);
        }
        sb.append("     ");
        if (tail != null) {
            sb.append("tail : " +  tail);
        } else {
            sb.append("tail : " +  null);
        }

        return sb.toString();
    }

}

```
MyLock实现的是一个“独占式”，“可重入”，只暴露有lock和unlock2个public接口的锁，没有实现什么共享式，条件锁之类的复杂功能。

MyLock的代码基本是对JDK中AQS的部分逻辑的一个简化和翻译，把晦涩的写法改为易读的写法，而不考虑性能，同时关键步骤加入debug信息以便分析执行逻辑和流程。

在了解了Unsafe的API后阅读起来不难，其核型逻辑就是2个死循环，以及满足什么样的条件时break出死循环。个人认为不太易理清的就是等待队列的入队尾和出队首时几个链接容易迷糊。尤其是入队尾，因为可能同时有很多个线程获取锁失败，导致它们争抢者入队尾，为了保证这个的正确性，使用了死循环加compareAndSwap操作。



- MyLockTest.java

```java
public class MyLockTest {

    static MyLock lock = new MyLock();

    static class Counter implements Runnable {

        @Override
        public void run() {
            String name = Thread.currentThread().getName();
            System.out.println(name + " : start");
            try {
                lock.lock();
                for (int i = 1; i <= 5; ++i) {
                    System.out.println(name + " : " + i);
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                    }
                }
                System.out.println(name + " : print over!");
            } catch (Throwable e) {
                Logger.debug(name + " throw");
                e.printStackTrace();
            } finally {
                Logger.debug(name + " going to unlock");
                lock.unlock();
            }
            System.out.println(name + " : finish");
        }

    }

    public static void main(String[] args) {

        for (int i = 10; i < 30; ++i) {
            Thread thd = new Thread(new Counter());
            thd.setName("T-" + i);
            thd.start();
        }

    }

}

```

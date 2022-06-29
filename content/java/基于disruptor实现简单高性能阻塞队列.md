---
title: "基于disruptor实现简单高性能阻塞队列"
date: 2021-04-02T11:22:39+08:00
draft: false
toc: true
---

Disruptor是英国外汇交易公司LMAX开发的一个高性能队列，它一般比JDK内置的阻塞队列ArrayBlockingQueue实现要快很多，Apache Storm、Camel、Log4j2等很多知名项目都应用了Disruptor以获取高性能。

Disruptor没有直接暴露get，take等队列方法，需要简答的包装下其API。本文实现基于Disruptor实现一个简答的阻塞队列，并和ArrayBlockingQueue做一个简单的性能对比。

## 先定义个简单Queue接口

Queue.java
```java
public interface Queue<T> {

    T take() throws InterruptedException;

    void put(T t) throws InterruptedException;

}
```

## 定义生产者模型

Producer.java
```java
public class Producer extends Thread {

    private final Queue<Integer> queue;

    private final long count;

    public Producer(Queue<Integer> queue, long count) {
        this.queue = queue;
        this.count = count;
    }

    @Override
    public void run() {
        int c = 0;
        try {
            while (c < count) {
                queue.put(++c);
            }
        } catch (InterruptedException e) {
            //
        }
    }

}
```

## 定义消费者模型

Consumer.java
```java
public class Consumer extends Thread {

    private final Queue<Integer> queue;
    private final long count;

    public Consumer(Queue<Integer> queue, long count) {
        this.queue = queue;
        this.count = count;
    }

    @Override
    public void run() {
        int c = 0;
        try {
            while (c < count) {
                ++c;
                queue.take();
            }
        } catch (InterruptedException e) {
            //
        }
    }
}
```

## 定义基于Jdk内置ArrayBlockingQueue实现的队列

JdkQueue.java
```java
import java.util.concurrent.ArrayBlockingQueue;

public class JdkQueue<T> implements Queue<T> {

    private final ArrayBlockingQueue<T> queue;

    public JdkQueue(int c) {
        queue = new ArrayBlockingQueue<>(c);
    }

    @Override
    public T take() throws InterruptedException {
       return queue.take();
    }

    @Override
    public void put(T t) throws InterruptedException {
        queue.put(t);
    }
}
```

## 定义基于Disruptor实现的队列

DisruptorQueue.java
```java
import com.lmax.disruptor.AlertException;
import com.lmax.disruptor.EventTranslatorOneArg;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.Sequence;
import com.lmax.disruptor.SequenceBarrier;
import com.lmax.disruptor.Sequencer;
import com.lmax.disruptor.TimeoutException;

public class DisruptorQueue<T> implements Queue<T> {

    private final EventTranslatorOneArg<Wrapper<T>, T> TRANSLATOR = (event, sequence1, t) -> event.setT(t);

    private final RingBuffer<Wrapper<T>> ringBuffer;
    private final Sequence sequence = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);
    private final SequenceBarrier sequenceBarrier;
    private long available = sequence.get();

    public DisruptorQueue(int c) {
        ringBuffer = RingBuffer.createMultiProducer(Wrapper<T>::new, c);
        sequenceBarrier = ringBuffer.newBarrier();
        sequenceBarrier.clearAlert();
        ringBuffer.addGatingSequences(sequence);
    }

    @Override
    public void put(T t) {
        ringBuffer.publishEvent(TRANSLATOR, t);
    }

    @Override
    public T take() {
        do {
            long next = sequence.get() + 1;
            while (next > available) {
                next = sequence.get() + 1;
                try {
                    available = sequenceBarrier.waitFor(next);
                } catch (AlertException | TimeoutException | InterruptedException e) {
                    //
                }
            }
            T t = ringBuffer.get(next).getT();
            if (sequence.compareAndSet(next - 1, next)) {
                return t;
            }
        }
        while (true);
    }


    public static class Wrapper<T> {
        private T t;

        public Wrapper() {
        }

        public T getT() {
            return t;
        }

        public void setT(T t) {
            this.t = t;
        }
    }

}
```

## 测试代码

TestMain.java
```java
public class TestMain {

    public static void main(String[] args) {
        Queue<Integer> jdkQ = new JdkQueue<>(1024);
        Queue<Integer> disruptorQ = new DisruptorQueue<>(1024);
        int testCount = 100_000_000;
        testQueue(jdkQ, testCount);
        testQueue(disruptorQ, testCount);
    }

    public static void testQueue(Queue<Integer> queue, int count) {
        Producer producer = new Producer(queue, count);
        Consumer consumer = new Consumer(queue, count);
        long start = System.currentTimeMillis();
        producer.start();
        consumer.start();
        try {
            producer.join();
            consumer.join();
        } catch (InterruptedException e) {
            //
        }
        long end = System.currentTimeMillis();
        System.out.printf("Queue %-15s cost : %d\n", queue.getClass().getSimpleName(), end - start);
    }

}
```

## 结论

上面的代码中，队列长度为1024（Disruptor要求是2的幂），消费1亿个Integer，在我自己的机器上，JDk的Queue耗时17-18秒左右，而基于Disruptor实现的队列耗时为7-8秒左右。因此**Disruptor比ArrayBlockingQueue快2.5倍接近3倍**。

上面的结果只代表本文中的测试代码的结果，代码应该还有优化的空间。但是肯定可以说明Disruptor比ArrayBlockingQueue快。（队列的长度要大一点，比如512，1024，太小的话，Disruptor显示不出优势，可能还比ArrayBlockingQueue慢）。

**Disruptor之所以快，是因为它内部没有锁，而是大量用了CAS操作，另外还用一些技巧缓解了“伪共享”的问题，提高CPU缓存命中率。**

更深入的学习Disruptor可以看美团技术团队的博客：[高性能队列Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html)

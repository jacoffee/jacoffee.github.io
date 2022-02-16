---
layout: post
category: kafka
date: 2021-12-01 15:29:03 UTC
title: 【Kafka实战】生产端内存池实现剖析
tags: [消息队列，Kafka，生产者，内存池]
permalink: /kafka/producer/bufferpool
key:
description: 本文主要剖析Kafka生产端内存池的实现
keywords: [消息队列，Kafka，生产者，内存池]
---

# 1. 是什么

对于一个高性能的生产者，由于频繁创建ByteBuffer进行数据传输，所以这个环节要尽量避免反复的ByteBuffer创建，减少GC。Kafka在生产端设计了一个**缓冲池**，用于管理内存- 底层使用ByteBuffer抽象。这种思想在MySQL的Buffer pool的free list(available)page中也有体现。

# 2. 核心属性

![kafka-producer-buffer-pool](/static/images/charts/2021-12-01/kafka-producer-buffer-pool.png)

+ **totalMemory**: 整个内存池的内存上限

+ **poolableSize**: 最小内存分配单位，也就是一个ByteBuffer分配的大小。也就是池化内存主体ByteBuffer分配内存的最小单位 -- 定位大小规整的内存区间

+ **nonPooledAvailableMemory**: 非池化内存

+ **free: Deque<ByteBuffer>**: 可用的内存队列，由一个个分配好大小的Bytebuffer组成。 

+ **waiters: Deque<Condition>**: 等待可用内存空间的条件队列

+ **lock: ReentrantLock**:  用来控制多线程同时进行资源申请时候的内存分配


关于池化和非池化内存的概念，下面会进一步解释。

# 3.  实现剖析

对于内存池而言，核心功能就两个: **处理内存申请请求和内存释放请求**

## 3.1 申请内存

### 3.1.1 申请内存时，要在什么环节加锁？

进入申请流程，就要加锁了，因为内部的核心数据结构(totalMemory、free、waiters)需要保护

### 3.1.2 如何设计池化和非池化内存？

提到池化，我们可以联想到线程池、数据库连接池、JedisPool等，核心就是提前初始化好**相应的资源(Thread、Connection、Jedis等)**，然后需要使用的时候直接返回，而不用临时创建。

当然也可以是懒加载，初次使用创建然后维护在内部的数据结构中，比如说JedisPool**采用队列来维护活跃Jedis**，线程池采用Set来维护活跃线程。

具体到BufferPool，池化内存采用双端队列(ArrayDeque)，以ByteBuffer为主体进行管理。也就是每次需要获取的时候从这直接读取，返回ByteBuffer。 而非池化内存从定义**nonPooledAvailableMemory**可以看出，还需要我们临时去封装成ByteBuffer，相当于要进行初始化，也就是不能直接从池中返回所以称为非池化内存。

### 3.1.3 内存分配的时机

非池化内存就是在BufferPool初始化的时候分配，实际上就是内存池的最大内存。池化内存是在释放内存，也就是归还之前申请的内存时，如果是按照最小单位申请的，则直接放入free队列中。

### 3.1.4 如果内存池中内存不够

这个如果抽象一下就是生产者和消费者模型，当没有内存可用时，那作为内存申请者的线程(消费者)必然会被挂起并设定最大等待时间。如果指定时间内，没有其它的线程归还申请的内存，则直接报错(TimeoutException)。

BufferPool在这一块利用了ReentrantLock的条件队列，如果内存不足则直接放入到条件队列中。最后调用condition.await，返回之后确认有没有在指定时间内被唤醒，如果是说明没有超时，反之则超时，说明内存申请失败。

申请成功之后:

+ 如果申请大小等于poolableSize且free队列不为空，则直接从free队列中返回
+ 否则先尝试遍历  池化内存中的ByteBuffer  然后添加到nonPooledAvailableMemory中，这个过程可能需要经历多次遍历(因为一个free队列的ByteBuffer可能不太满足要求)

这个地方也体现，虽然Kafka生产者是异步发送消息的，但是也可能因为短时间内存不足而阻塞，甚至最终失败。

## 3.2 释放内存(归还内存)

### 3.2.1 往哪还？池化还是非池化？

归还内存的时候，是以ByteBuffer为单位进行的，那简单理解就是将ByteBuffer放回队列即可。但如果不是池化内存的最小单位，则直接归还到非池化内存，正好和分配内存一一对应。

### 3.2.2 归还之后做什么？

之前可能因为等待获取可用内存，导致一些线程等待(存放ReentrantLock的条件队列中)，所以此时需要唤醒。注意是采用**peek & singnal**，实际上调用的就是AQS中的signal方法，也就是将**等待内存的线程(封装成Node)从条件队列移动到同步队列**，然后尝试获取锁。

```java
/**
 * Moves the longest-waiting thread, if one exists, from the
 * wait queue for this condition to the wait queue for the
 * owning lock.
 *
 * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
 *         returns {@code false}
 */
public final void signal() {
    if (!isHeldExclusively()) {
        throw new IllegalMonitorStateException();
    }

    Node first = firstWaiter;
    if (first != null) {
        doSignal(first);
    }
}
```

# 4. 实战演练

## 4.1 一个实现bug引发的死锁问题

Kafka单元测试中设计了一个单元测试，以验证多个多线程同时访问时，内存池是否正常工作。


```java
class StressThread extends Thread {

        private final int iterations;

        private final BufferPool pool;

        private final long maxBlockTimeMs = 1000;

        public final AtomicBoolean success = new AtomicBoolean(false);

        private Random random = new Random();

        public StressTestThread(BufferPool pool, int iterations) {
            this.iterations = iterations;
            this.pool = pool;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < iterations; i++) {
                    int size;
                    if (random.nextBoolean()) {
                        // 分配固定大小
                        size = pool.pooledMemoryUnit();
                    } else {
                        // 分配随机大小
                        size = random.nextInt(pool.totalMemory());
                    }
                    ByteBuffer buffer = pool.allocate(size, maxBlockTimeMs);
                    pool.deallocate(buffer);
                }
                success.set(true);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

}
```

总结起来就是在多个线程运行的时候，模拟随机的内存申请: 固定大小和随机的，每次先申请后释放，每个过程都需要获取锁。正常来说，这个过程是很快的。但是如果某一个流程没有即使释放锁，就会导致其它线程阻塞。

而allocate在申请内存结束之后，会尝试去唤醒等待的线程;  bug版本直接将下面的**if写成了while**，所以一旦进入这个循环，就一直出不来也就不会释放锁了。

```java
if (!(nonPooledAvailableMemorySize == 0 && free.isEmpty()) && !waiters.isEmpty()) {
    String msg = String.format(
        "Thread %s try to signal other threads, nonPooledAvailableMemorySize %d, free size %d",
        Thread.currentThread().getName(),
        nonPooledAvailableMemorySize, free.size()
    );
    log.info(msg);

    waiters.peekFirst().signal();
}
```

排查思路:  通过jstack可以很容易发现，很多线程在尝试获取`"Thread-2"`持有的锁(`Locked ownable synchronizers: <0x000000076bae9220>`)， 比如说"Thread-1"中的

**parking to wait for  <0x000000076bae9220>**。

```bash
"Thread-3" #14 prio=5 os_prio=31 tid=0x00007ff5e9812800 nid=0xa503 waiting on condition [0x0000700001cfd000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000076bae9220> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2168)
	at com.jacoffee.dfs.client.BufferPool.allocate(BufferPool.java:178)
	at com.jacoffee.dfs.client.BufferPoolTest$StressTestThread.run(BufferPoolTest.java:166)

   Locked ownable synchronizers:
	- None

"Thread-2" #13 prio=5 os_prio=31 tid=0x00007ff5e18c7800 nid=0x5b03 runnable [0x0000700001bfa000]
   java.lang.Thread.State: RUNNABLE
	at com.jacoffee.dfs.client.BufferPool.allocate(BufferPool.java:226)
	at com.jacoffee.dfs.client.BufferPoolTest$StressTestThread.run(BufferPoolTest.java:166)

   Locked ownable synchronizers:
	- <0x000000076bae9220> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)

"Thread-1" #12 prio=5 os_prio=31 tid=0x00007ff5e18c6800 nid=0x5903 waiting on condition [0x0000700001af7000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000076bae9220> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.jacoffee.dfs.client.BufferPool.allocate(BufferPool.java:153)
	at com.jacoffee.dfs.client.BufferPoolTest$StressTestThread.run(BufferPoolTest.java:166)

   Locked ownable synchronizers:
	- None
```
















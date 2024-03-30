---
layout: post
category: java
date: 2020-10-14 17:26:03 UTC
title: 【Java基础】锁之ReentrantLock
tags: [synchronized, critical section]
permalink: /java/basics/lock
key:
description: 本文整理了Java中ReentrantLock相关的知识
keywords: [synchronized, critical section]
---

`java.util.concurrent.locks.ReentrantLock`基于AQS的可重入的互斥锁(其实也可以实现共享锁)实现，和基于内置的对象监视器(implicit monitor lock)锁实现的**synchronized**具有相同的语义。只不过提供更多扩展性的功能，如**支持公平锁**、**锁获取超时机制**、锁中断(**获取锁过程被interrupted**)。底层实现**java.util.concurrent.locks.AbstractQueuedSynchronizer**(抽象队列同步器，基于CLH Queue的变体，比如说多了next指针)， 像`java.util.concurrent.Semaphore`也是基于它实现的。

# 1. 基本使用

对于ReentrantLock的常规使用就是**加锁 -- 执行共享资源操作 -- 释放锁(置于finally块中)**，注意加锁操作的时长。

```java
public void test() throw Exception {
    
    // 默认非公平锁 sync = new NonfairSync()
    ReentrantLock lock = new ReentrantLock(); 
    lock.lock();
    try {
        xxxx
        // 嵌套 重入
        lock.lock();
        try {
            yyyy
        } finally {
            lock.unlock()                
        }
    } finally {
     	lock.unlock()
    }
}
```

# 2. 关于java.util.concurrent.locks.Lock

该接口定义了Java中锁实现的规范，如果我们需要在Java中实现显示锁(相较于内置锁intrinsic lock 也就是 monitor lock, synchronized就是基于此实现的)，需要实现该接口的相关方法。

Lock是为了实现**单JVM多线程对于共享资源访问而实现的**。一般来说，lock确保同一时间只有一个线程对共享资源访问(**mutex**)，每次对于共享资源访问的时候都要先获取锁，但是有些锁实现允许多个线程同时访问共享资源(比如说ReadWriteLock中的读锁)。

```java
// 核心API
public interface Lock {

	// 拼尽全力的获取锁
    // 尝试获取锁 -- 失败，disable scheduling && lie dormant直到获取锁; 
    void lock();
    
   	// 浅尝辄止的获取锁
    // 尝试获取锁，如果当前锁可以获取，则获取；反之直接返回，这个try表达了试一下就算了
    boolean tryLock();
    
    // 较上面的，增加了获取超时时长
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    
    // 尝试获取锁 -- 失败，disable scheduling && lie dormant -- 直到该线程被其它中断(interuptted) 或者是 该线程获取锁
    void lockInterruptibly() throws InterruptedException;
    
    
    // 与加锁相对应，释放锁
    void unlock();
    
    // 条件队列
    Condition newCondition();
    
}
```

# 3. synchronized vs ReentrantLock

关于这两者的比较是一个绕不过去的问题，我们在接触Java Lock这一块的时候，最早接触的应该就是**synchronized**，几乎使用的是最多的。下表简单总结它们之间的区别。

## 3.1 区别

| 选项       | ReentrantLock                                                | synchronized                                                 |
| ---------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 锁实现机制 | 依赖AQS，支持公平锁和非公平锁                                | 获取和每个对象关联的**隐式监视器锁**                                                                                                    只支持`非公平锁`，基于悲观锁实现 |
| 灵活性     | 1.  获取锁时可被中断  2.  获取锁的时候指定超时时间           | 不灵活, 不支持中断                                           |
| 释放形式   | 必须显示调用**unlock()**                                     | 自动释放监视器                                               |
| 条件队列   | 可关联多个条件队列                                           | 关联一个条件队列(与对象关联的waitset)                        |
| 可重入性   | 可重入 -- **判断尝试加锁线程和当前持有锁线程是否一样** synchronizedState + 1 | 可重入 monitor reentrant + 1                                 |
| 是否结构化 | 否，lock() 和 unlock() 和随意组合，甚至可以分散在不同的方法中 | 固定 synchronized(this) {}; synchronized(object) {};         |

## 3.2 如何选择

在实际场景中，你需要使用到ReentrantLock提供的特性嘛？ 比如说公平锁支持、获取锁的时候设置超时时间等,  如果需要则选择ReentrantLock。如果没有的话，一般情况下还是synchronized，既然是Java内置锁，性能肯定是不会差。


# 4. AQS(`AbstractQueuedSynchronizer`) -- 抽象队列同步器

## 4.1 What & Why

ReentrantLock的底层核心就是AQS，Java并发中还有很多其它帮助类(LinkedBlockingQueue、Semaphore)也是依赖该框架。

### 4.1.1 获取锁的核心原理 -- state & exclusiveOwnerThread

就是将 `共享的同步状态(state)设置成指定的值`，如ReentrantLock中由0变1代表获取锁成功同时会将`exclusiviveOwnerThread`设置成当前线程

### 4.1.2 同步队列 -- sync queue && 条件队列  -- Condition

当多个线程尝试获取锁的时候，肯定有些会失败。如果失败了则直接放弃获取锁，那么**并发会大大降低**。此时需要

+ **同步队列**用来存放获取锁失败而阻塞的线程(通过LockSupport.park实现)，然后每次线程释放锁的时候，就会尝试从对头开始唤醒线程，进一步去获取锁。 这也就是命名中Queue的体现。
+ 如果获取锁失败，是要在指定条件下，才能再次去获取锁，此时需要条件队列，意味在某种情况下才能被唤醒。

下图大致展示了AQS的核心数据结构:

![aqs-structure](/static/images/charts/2020-10-14/aqs-structure.png)


## 4.2 重要属性

```java
public abstract class AbstractQueuedSynchronizer {
    
    static final class Node {
        
        // 当前节点的等待状态，因为本质上来讲这些入队的线程都是入队以在将来获取执行时间的
        
        /*
        	0: 初始化状态
            SIGNAL: 代表后继结点需要被唤醒, 该状态表示的是后继结点的状态而不是本结点状态
        	CANCELLED: 线程获取锁的请求取消
        	PROPAGATE: 当线程处于SHARD情况下, 该字段才会使用
        	CONDITION: 表示结点再同步队列中，节点线程等待唤醒
        */
        
        volatile int waitStatus;
        
        // 当前入队线程的前驱
        volatile Node prev;
        
        // 当前入队线程的后继
        volatile Node next;
        
        // 当前入队的线程
        volatile Thread thread;
        
        // 指向下一个等待condition的线程
        Node nextWaiter;
        
    }
    
    // 同步队列的头部
    private transient volatile Node head;
       
    // 同步队列的尾部
    private transient volatile Node tail;
    
     /**
     * The synchronization state -- AQS和共享资源一一对应
     */
    private volatile int state;
    
    
    // 条件队列
    public class ConditionObject implements Condition, java.io.Serializable {
        
    }

}
```

关于重要属性的解释:

+ state: 同步状态由实现来决定什么状态为获取锁，什么状态为释放锁

+ waitStatus: 当前Node的等待状态，**最特别的一点就是当前的节点唤醒状态(SIGNAL)是存储在前驱中的**

+ Node即同步队列的抽象，本质上是一个双向队列

+ ConditionObject为条件队列的抽象，这个在LinkedBlockingQueue和ArrayBlockingQueue中有大量的使用

| waitStatus | int value |      |
| ---------- | --------- | ---- |
| 0          | 0         |      |
| SIGNAL     | -1        |      |
| CANCELLED  | 1         |      |
| PROPAGATE  | -3        |      |
| CONDITION  | -2        |      |

## 4.3 加锁和解锁 -- Doug Lea AQS

同步器的实现需要解决两个主要问题:  加锁和释放锁(用伪代码解释就是)

```java
// 加锁操作
while (synchronization state does not allow acquire) {
	enqueue current thread if not already queued;
	possibly block currrent thread and waited to be signaled
}
dequeue current thread if it is queued and acquire again

// 释放锁流程
update synchronization state;
if (state may permit a blocked thread to accquire)
  unblock one or more queued threads;
```

要实现上述过程，需要做好三个流程的协调:

+ 同步状态的原子性，也就是多个线程同时更新时，要保证状态正确；当前资源是空闲、还是被某个线程独占、还是被某个线程可重入了

+ 阻塞线程和唤醒线程；不能一失败就阻塞、由谁唤醒等

+ 队列的管理；如果队列不能再次入队、已经被取消的要即时清楚


## 4.4 CLH(Craig、Landin and Hagersten) Lock 队列


AQS是基于该队列进行改造的, 实际为FIFO双端队列(对头和对尾均可以插入)。

```bash
The wait queue is a variant of a "CLH" (Craig, Landin, and Hagersten) lock queue. CLH locks are normally used for spinlocks.
```

<font color="red">
关于waitStatus=SIGNAL的设计，其实是源于Doug Lee的AQS框架论文，CLH queue原本设计，每个结点的release status是存放在前驱中的(**the release status for each node is kept in its predecessor node**)。
</font>

>  Cancellation support mainly entails checking for interrupt or timeout upon each return from park inside the acquire loop. A cancelled thread due to timeout or interrupt **sets its node status**  and **unparks its successor so it may reset links**. 
>
>  With cancellation,  determining predecessors and successors and resetting status may include *O(n)* traversals (where *n* is the length of the queue). 
>
>  Because a thread never again blocks for a cancelled operation, links and status fields tend to restabilize quickly.

`取消等待`支持是因为线程在挂起的可能会被打断或者设置阻塞多长时间，如果出现上述情况，线程从获锁中跳出(实际上就是acquireQueued方法)。

与此同时，由于获取锁失败，所以自己的顺位失效，因此需要唤醒它的后继(`cancelAcquire`)。

该队列将尝试获取锁的线程(thread)抽象成为Node, 基本结构图如下:

AQS中队列的维护更多是自治的，比如说剔除waitStatus=CANCELLED, 是加锁过程中遍历的时候自行剔除的。

# 5. 加锁过程 -- lock()

## 5.1 lock()

CAS设置**synchronizatio state**，state=0 -> 1，能设置成1，然后设置独占线程(`setExclusiveOwnerThread(Thread.currentThread())`) 则说明成功；反之进入acquire流程。

![aqs-simple-lock](/static/images/charts/2020-10-14/aqs-simple-lock.png)

## 5.2 acquire() - tryAccquire()根据公平锁还是非公平锁而有不同

入队之前再次尝试去获取锁，如果失败则加入到同步队列中。在这个过程中，可能出现反复的阻塞和被唤醒。

这个地方在ReentrantLock中因**公平和非公平策略**而由不同,  如下面的代码所示。

> 公平锁会首先检查队列中是否有线程排队，如果有的话，则会"乖乖"排队; 
>
> 而非公平锁，则会直接尝试获取锁，也就是AQS论文中提到的barge(**突然闯入**)。

### 5.2.1 FairSync

```java
static final class FairSync extends Sync {
    
    // 检查队列中是否已经有线程再排队，没有才进行CAS操作
    protected final boolean tryAcquire(int acquires) {
        ...
        if (
            !hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)
        ) {
            setExclusiveOwnerThread(current);
            return true;
        }
        ...
    }
    
}
```

### 5.2.2 NonFairSync(在父类中实现的)

```java
abstract static class Sync extends AbstractQueuedSynchronizer { 
	...
    final boolean nonfairTryAcquire(int acquires) {
        ...
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
        ...
    }
	...
}
```

上述流程思路很清晰，不过引申出另外一个问题。

### 5.2.3 Why non-fair sync by default

为什么ReentrantLock默认使用的是非公平锁？

> In reality, the fairness guarantee for locks is a very strong one, and comes at a **significant performance cost**. The **bookkeeping and synchronization** required to ensure fairness mean that contended fair locks will have much lower throughput than unfair lock

答案就是：

> 基于上面提到的，非公平锁相较于公平锁会提供更高的吞吐，抛开ReentrantLock，公平锁要尽量保证先尝试获取锁的线程先获取，当contended lock再次available, 一定会有协调问题

减少唤醒开销(reduce time during which a contended lock is available but no thread has it because the intended next thread is in the process of unblocking)，公平锁如果已经有线程排队且获取锁失败就会被放入队列，而非公平锁却是强势插入，对于那种短时间持有锁的，非公平锁显然会更好。但如果持有锁时间较长，那么相对来说公平锁就会更好，因为释放锁不会那么及时，所以抢占的意义就不大了。

## 5.3 `addWaiter()` - 将线程自旋的加入同步队列的尾部

```java
// 方法主要目的：node 追加到同步队列的队尾
// 入参 mode 表示Node的模式（排它模式还是共享模式)
// 出参是新增的 node

// 主要思路：
// 新 node.pre = 队尾
// 队尾.next = 新 node
private Node addWaiter(Node mode) {
    // 初始化 Node
    Node node = new Node(Thread.currentThread(), mode);
  
    // 这里的逻辑和enq一致，enq的逻辑仅仅多了队尾是空，初始化的逻辑
    // 这个思路在java源码中很常见，先简单的尝试放一下，成功立马返回，如果不行，再 while循环
    // 很多时候，这种算法可以帮忙解决大部分的问题，大部分的入队可能一次都能成功，无需自旋
    
    // 在ReplicaManager内部进行其它副本测试的，也算是假如延时任务体系时候会先再检查一下
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //自旋保证node加入到队尾
    enq(node);
    return node;
}

// 线程加入同步队列中方法，追加到队尾
// 这里需要重点注意的是，返回值是添加 node 的前一个节点
private Node enq(final Node node) {
    for (;;) {
        // 得到队尾节点
        Node t = tail;
        // 如果队尾为空，说明当前同步队列都没有初始化，进行初始化
        // tail = head = new Node();
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        // 队尾不为空，将当前节点追加到队尾
        } else {
            node.prev = t;
            // node 追加到队尾
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

![aqs-add-waiter.png](/static/images/charts/2020-10-14/aqs-add-waiter.png)

## 5.4 `accquireQueued()`

线程入队之后再次尝试获取锁，失败则尝试挂起(`要成功挂起必须前驱或者CAS设置队列中的前驱状态为SIGNAL`)。

该方法的返回值是:  获取锁的过程中有没有被打断

```java
final boolean acquireQueued(final Node node, int arg) {
    
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前驱是头部，所以该当前线程是队列唯一结点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            
            // 尝试获取锁失败之后，是否需要被挂起(即设置前驱waitStatus == SIGNAL)
            if (
                shouldParkAfterFailedAcquire(p, node) &&
                
                // 这个地方是线程挂起的地方，后续有线程释放锁，会唤醒该线程然后继续自旋加锁
                // LockSupport.park()
                parkAndCheckInterrupt()
            )
            interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
    
}
```

### 5.4.1 `shouldParkAfterFailedAcquire` - 获取锁失败之后是否挂起

设置前驱waitStatus=SIGNAL状态，成功之后自己才能`安心的挂起`。该过程中还剔除了**waitStatus=CANCELLED**，直到找到一个没有被CANCELLED的前驱。

这个方法要和外面的死循环结合起来看。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
        * This node has already set status asking a release
        * to signal it, so it can safely park.
        */
        return true;
    
    if (ws > 0) {
        /*
             Predecessor was cancelled. Skip over predecessors and
             indicate retry.
             
             前驱被放弃获取锁，所以需要将其剔除
		   */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);

        pred.next = node;
    } else {
        /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

![](/static/images/charts/2020-10-14/aqs-shouldParkAfterFailedAcquire.png)

### 5.4.2 parkAndCheckInterrupt() - 调用之后当前线程被阻塞

`parkAndCheckInterrupt()`，在如下场景下会被唤醒：

```java
private final boolean parkAndCheckInterrupt() {
  	LockSupport.park(this);
    return Thread.interrupted();
}
```

+ 当某个结点释放锁的时候，会唤醒它的后继结点(wake up node's succesor if one exists)

+ 当前结点获取锁失败，尝试唤醒它的后继结点

## 5.5 `cancelAcquire` - 取消当前结点获取锁的行动

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer
private void cancelAcquire(Node node) {
  // 将无效节点过滤
	if (node == null)
		return;

  // 设置该节点不关联任何线程，也就是虚节点
	node.thread = null;
	Node pred = node.prev;

   // 通过前驱节点，跳过取消状态的node
	while (pred.waitStatus > 0)
		node.prev = pred = pred.prev;
    
  // 获取过滤后的前驱节点的后继节点
	Node predNext = pred.next;
  // 把当前node的状态设置为CANCELLED
	node.waitStatus = Node.CANCELLED;
  // 如果当前节点是尾节点，将从后往前的第一个非取消状态的节点设置为尾节点
  // 更新失败的话，则进入else，如果更新成功，将tail的后继节点设置为null
	if (node == tail && compareAndSetTail(node, pred)) {
		compareAndSetNext(pred, predNext, null);
	} else {
		int ws;
    // 如果当前节点不是head的后继节点，1:判断当前节点前驱节点的是否为SIGNAL，2:如果不是，则把前驱节点设置为SINGAL看是否成功
    // 如果1和2中有一个为true，再判断前驱的线程是否为null
    // 如果上述条件都满足，把当前节点的前驱节点的后继指针指向当前节点的后继节点

        if (
            pred != head && 
           ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) 
            && pred.thread != null
        ) {
			Node next = node.next;
			if (next != null && next.waitStatus <= 0)
				compareAndSetNext(pred, predNext, next);
		} else {
      		// 如果当前节点是head的后继节点，或者上述条件不满足，那就唤醒当前节点的后继节点
    		// 因为当前线程获取锁失败了，所以尝试唤醒下一个排队的。
            // 就像竞标买房，上一个出价的人失败后(设置cancel状态)，轮到下一个人出价
			unparkSuccessor(node);
		}
		node.next = node; // help GC
	}
}
```



整个过程还是比较复杂的，无论如何还是不要陷入代码细节(作为个人喜好是没有撒到的问题)，最后通过一组同步队列的节点变化来更直观的感受下。

![aqs-sync-queue-1](/static/images/charts/2020-10-14/aqs-sync-queue-1.png)

![aqs-sync-queue-1](/static/images/charts/2020-10-14/aqs-sync-queue-2.png)

# 6. 释放锁过程 -- unlock

通过更改synchronization state来实现，同样和synchronized、notify、notifyAll会有**IllegalMonitorStateException**的问题，就是如果当前线程不是锁的持有者却尝试释放锁。

```java
public void unlock() {
	sync.release(1)
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer
public final boolean release(int arg) {
  
    // 上边自定义的tryRelease如果返回true，说明该锁没有被任何线程持有
    if (tryRelease(arg)) {
        // 获取头结点 为null说明没有线程排队
        Node h = head;
        // 头结点不为空并且头结点的waitStatus不是初始化节点情况，解除线程挂起状态
        if (h != null && h.waitStatus != 0) {
            unparkSuccessor(h);
        }
        return true;
    }
  
    return false;
}

public boolean tryRelease(int arg) {
    int c = getState()
    int state = c - arg;
    boolean free = false;
	  if (state == 0) {
        // 变为0, 成功释放锁，同时设置Owner线程为null
        setExclusiveOwnerThread(null);
        free = true;
    }
    setState(state); 
    return free;
}
```

特别注意上面唤醒后继结点线程的判断: `h != null && h.waitStatus != 0`

当第一个线程入队的时候，如果head结点没有被初始化就会初始化(**lazy initialization**), 此种情况说明还没有来得及初始化。

释放过程主要是更新synchronized state: 独占的会变为0，可重入每释放依次减少1(为0之前都不算真正释放)。

## 6.1 唤醒后继结点

当前结点或者某个线程释放锁之后，其它排队线程有机会重新获取锁，但由于有些线程已经被挂起(`blocked`)，所以需要去唤醒(**LockSupport.unpark(s.thread)**)

```java
private void unparkSuccessor(Node node) { 
	Node s = node.next;
    
    // 后继结点为null 或者 已经放弃获取锁; 则尝试从尾到头再挑选一个Node进行唤醒; 最终返回的离node最近的 需要唤醒的
    // s.waitStatus 大于0，代表 s 节点已经被取消了
    if (s == null || s.waitStatus > 0) {
      	s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) {
        	if (t.waitStatus <= 0) {
				s = t
        	}
        }
    }
    
    // 正常一般都是这个情况
    if (s != null) {
     	LockSupport.unpark(s.thread);   
    }
    
}
```

第一次看到找需要唤醒的结点时，有一个疑问:  `为什么从后往前?  这一点在AQS论文中也提到过`

> If a node's successor does not appear to exist (or appears to be cancelled) via its next field, it is always possible to **start at the tail of the list and traverse backwards using the pred field to accurately check if there really is one**.

如果一个结点的后继 -- 不存在或者已经被取消了， 所以**无法通过后继去寻找待唤醒的节点了**

因此这里`采用了从尾部使用prev指针反向遍历`(next目前就是一个辅助属性，为了更快的锁定结点)。

![aqs-unlock](/static/images/charts/2020-10-14/aqs-unlock.png)


# 7. 项目中的使用

## 7.1 Java集合

+ **LinkedBlockingQueue**利用ReentrantLock的，分别实现takeLock和putLock，来协调生产者和消费者
+ **CountDownLatch** 基于抽象同步队列(**AbstractQueuedSynchronizer**), 定义了自己的同步控制

Java **java.util.concurrent** 包中有很多利用AQS的，在使用的时候不妨跟进到源码中去看看。

## 7.2 Kafka

比如说KafkaConsuemer底层的网络层抽象`org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient`，内部就大量使用。因为虽然官方不建议使用

multi-thread consumer，一般都是一个Java Node(可以理解为一个JVM)作为消费者，但是也没有完全阻止，所以从接口层面需要处理这种情况，那么lock就不可避免。

```java
// ConsumerNetworkClient

// We do not need high throughput, so use a fair lock to try to avoid starvation;
// 公平锁避免 starvation; 从这句话可以看出，非公平锁的吞吐量会更高 
private final ReentrantLock lock = new ReentrantLock(true);

public void poll(Timer timer, PollCondition pollCondition, boolean disableWakeup) {
    
    try {
	    lock.lock();
    } finally {
    	lock.unlock();
	}

}
```

# 8. Q&A

## 8.1 ReentrantLock 获取锁过程中可中断是如何实现的?

AQS中提供了`acquireInterruptibly`

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted()) {
        throw new InterruptedException();
    }
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);    
    
    }
}
```

+ 尝试获取锁之前，**先检查下当前线程有没有被interrupted**

+ 如果获取失败被加入到同步队列的时候，再次获取失败之后，还会去检查尝试获取锁的线程有没有被打断


```java
private void doAcquireInterruptibly(int arg) {
    
    if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
        throw new InterruptedException();
    }
                    
}
```

## 8.2 如何检测线程是否加入了同步队列

这个问题是回应上面那句`enqueue current thread if not already queued`

## 8.3 更改状态 setState vs compareAndSetState

为什么更改 state 有 setState() ,  compareAndSetState() 两种方式，感觉后者更安全，但是锁的实现中有好多地方都使用了setState()，安全吗？

这个其实跟线程安全性有关，如果操作的时候没有线程安全性问题，则直接使用setState,  比如说可重入的时候，如果当前线程持有锁了，则直接增加 获取锁的次数

```java
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    
    // ! 自己已经获取过锁了，当然可以安全的进行修改
    setState(nextc);
	return true;
}
```



# 9. 参考

\>  [Doug Lee关于AQS实现原理的论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)

\> [美团 从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

\> [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)

\> [Infoq AbstractQueuedSynchronizer深度解析](https://www.infoq.cn/article/jdk1.8-abstractqueuedsynchronizer)

\> [high performance synchronization for shared-memory parallel program](https://www.cs.rochester.edu/wcms/research/systems/high_performance_synch/)

\> 面试官系统精讲Java源码及大厂真题 30 AbstractQueuedSynchronizer 源码解析（上）
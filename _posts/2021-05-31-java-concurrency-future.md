---
layout: post
category: java
date: 2021-05-13 16:29:03 UTC
title: 【Concurrency】Java Future实现剖析
tags: [异步执行、Future、FutureTask]
permalink: /java/concurrency/future
key:
description: 本文主要剖析Future的核心实现
keywords: [异步执行、Future、FutureTask]
---

# 1. 是什么

Future的出现与异步计算是密不可分的，它代表着异步计算的结果。同时提供了**检测计算是否完成**、**阻塞等待完成**以及**取回计算完成的结果**等方法。它的核心实现可以概括为如下几点:

+ 提交任务的线程和获取状态的线程可能不同
+ 内部维护state，记录任务指定的不同状态，运行中、执行完以及取消等，以对外提供状态检查的功能 `isDonde`、`isCancelled`
+ 内部维护当前执行任务线程runner，通过CAS设置，避免多个线程同时执行同一个任务同时也可以作为任务取消打断的线程
+ 内部维护链表记录等待线程，以实现在指定时间内等待任务完成的操作 `get(long timeout, TimeUnit unit)`

![future-task-struture](/static/images/charts/2021-05-13/future-task-struture.png)

# 2. 核心方法

```java
public interface Future<V> {
    
    /*
        对于正在运行的任务进行取消
    */
    boolean cancel(boolean mayInterruptIfRunning);

    
    /**
        是否已经完成
    */
    boolean isDone();
    
    /*
        是否已经取消
    */
    // 线程如果已经运行结束了，isCancelled 和 isDone 返回的都是 true
    boolean isCancelled();
    
    /*
        阻塞等待计算结果
        如果任务被取消了，抛CancellationException 异常
        如果等待过程中被打断了，抛 InterruptedException 异常
    */
    V get() throws InterruptedException, ExecutionException;
    
    /**
        带超时的等待任务执行完成，如果超时时间外仍然没有响应，抛 TimeoutException 异常
    */
    V get(long timeout, TimeUnit unit)
       throws InterruptedException, ExecutionException, TimeoutException;

}
```

由于Future是一个接口，那我们以它最常用的实现类`FutureTask`，一般我们提交到线程池的Runnable就会封装成这个类型来看看它核心方法的大致实现。

# 3. FutureTask -- 共享执行状态 + WaitNode

由于线程池提交任务的时候，有的期望有返回值(`submit(Callable)`)，有的不需要有返回值(**submit(Runnable)**)。所以需要将这两者统一起来:  **设计一个兼具运行任务以及适当的时候拿回返回值的数据结构**，于是有了RunnableFuture，它继承了Runnable和Future,  而FutureTask则实现了RunnableFuture。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    
    void run();

}

// ThreadPoolExecutor
public Future<?> submit(Runnable task) {
    RunnableTask<Void> futureTask = new FutureTask<>(task, null);
    execute(ftask);
    return futureTask;
}
```

## 3.1 重要属性

```java
public class FutureTask<V> implements RunnableFuture<V> {

    private volatile int state;
    
    private Callable<V> callable;
    
    private Object outcome;

    private volatile Thread runner;

}
```

| 属性     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| state    | 当前任务的状态，最开始的是NEW，执行完成是NORMAL;  由于涉及到不同的线程去读写，所以使用volatile修饰当然具体操作的时候结合CAS确保线程安全 |
| callable | 底层待运行任务的抽象，如果是Runnable会通过RunnableAdapter进行转换，最终执行是调用它的call方法 |
| outcome  | 异步执行的结构(具体的值或者是异常)，主要通过volatile的state来保护读写，也就是state决定什么时候可以写，什么时候读 |
| runner   | 当前正在执行任务的线程，CAS设置                              |

## 3.2 run() -- CAS Runner & 执行任务 & 更新state

以下面这个线程池任务提交来看看FutureTask的**run方法**做了哪些工作？

```java
ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(() -> {
    System.out.println("Test FutureTask");
});
```

```java
public void run() {
    // 初始化的时候已经将state装改设置了成NEW另外如果尝试CAS设置当前线程会执行者失败，所以已经有线程在执行
    if (
        state != NEW ||
        !UNSAFE.UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread())
    ) {
        return;
    }
    
    try {
        Callable c = callable;
        boolean successful;
        
        try {
            
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                // boolean ran;
                boolean succesful;
                try {
                    // 调用执行 System.out.println("Test FutureTask");
                    result = c.call();
                    succesful = true;
                } catch (Throwable ex) {
                    result = null;
                    succesful = false;
                    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
                        outcome = ex;
                        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
                        finishCompletion();
                    }
                }
                
                if (successful) {
                    // 任务已经执行完成，CAS设置state为COMPLETING成功
                    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
                        outcome = v;
                        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
                        finishCompletion();
                   }
                }
            } finally {
            
            }
        }
        
    }

}
```


+ 通过线程池提交的Runnable会被转换成Callable, 只不过返回值为null。`new FutureTask<T>(runnable, null)`
+ 线程执行的任务的时候，触发FutureTask的run()方法
+ 首先**尝试CAS设置自己为当前执行任务的线程**，成功则开始执行真正的逻辑
+ 任务执行完成之后，CAS设置**state=NORMAL**，异常则设置**state=EXCEPTIONAL**


## 3.2 get(timeout, Timeunit) -- 等待执行结果处理，最多多长时间

```java
public V get(long timeout, TimeUnit unit) {
    
    int currentState = state;
    if (currentState <= COMPLETING) {
        currentState = awaitDone(true, unit.toNanos(timeout));
        if (currentState <= COMPLETING) {
            throw new TimeoutException();
        }
    }
    
    return report(currentState)
    
}
```

这个实现的核心就是借助Lock.support来挂起线程以及WaitNode维护当前正在等待获取结果的线程。


## 3.3 cancel -- 取消任务执行

由于底层执行任务的是线程，所以**cancel的实现实际上就是打断Thread**。只有在当前FutureTask状态=NEW且CAS成功设置线程状态为INTERRUPTING(如果执行中允许打断就是设置成该状态)或CANCELLED才能进行真正打断。

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (
        state == NEW  && UNSAFE.compareAndSwapInt(this, stateOffset, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)
    ) {
        // 进行取消操作，打断可能会抛出异常，选择 try finally 的结构
        try {
            // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null) {
                        t.interrupt();
                    }
                } finally { 
                    // final state
                    //状态设置成已打断
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 清理线程，如果有线程在等待获取状态，则依次唤醒
            finishCompletion();
        }

        return true;
    } else {
        return false;
    }
}
```



# 参考

\> 面试官系统精讲Java源码及大厂真题 28 Future、ExecutorService源码解析

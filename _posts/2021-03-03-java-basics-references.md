---
layout: post
category: spring
date: 2021-03-03 17:26:03 UTC
title: 【Java基础】引用类型
tags: [Strong Reference、Weak Reference、Soft Reference]
permalink: /java/basics/references
key:
description: 本文整理了Java引用的基础知识
keywords: [Strong Reference、Weak Reference、Soft Reference]
---


# 1. 强引用(Strong Reference)

这是使用多的一种引用，当我们通过new关键字创建出对象的时候，然后通过引用指向该对象，这个引用就是强引用

```java
Object A = new Object
```

# 2. 弱引用(Weak Reference)

java.lang.ref.WeakReference,  **GC发生的时候会立即回收**(GC must collect this kind of object)

```java
/*
  要特别这种，将强引用地址传递； gc的时候并不会受应用
  Byte[] bytes = new Byte[1024 * 1024 * 5];
  WeakReference weakReference = new WeakReference(bytes);
*/
WeakReference weakReference = new WeakReference(new Byte[1024 * 1024 * 5]);

System.out.println(weakReference.get());

System.gc();
System.out.println(weakReference.get()); // null
```

## 2.1 ThreadLocal

涉及到重要的成员变量

```java
// Thread.java
public class Thread {

    // 每个线程内部维护了一个ThreadLocalMap
    ThreadLocal.ThreadLocalMap threadLocals = null;

}
```

```java
public class ThreadLocal<T> {

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue)
    }
    
    static class ThreadLocalMap {
        
        // 继承了WeakReference 但是referent是 ThreadLocal 
        // 所以需要GC的时候 threadLocal对象被回收了，但是tl 指向的 Value 还是将引用
        static Entry extends WeakReference<ThreadLocal<?>> {
              Object value; 
        }
        
    }
    
}
```

### 2.1.1 初始化并新增

检查当前线程是否初始化了ThreadLocalMap

+ if yes, 将自己(ThreadLocal)添加到ThreadLocalMap中，value是T
+ if no,  初始化ThreadLocalMap, 绑定到当前线程的threadLocals成员变量

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

而ThreadLocalMap的Entry继承了WeakReference

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
    
}

private Entry getEntry(ThreadLocal<?> key) {}

private WeakReference<ThreadLocal<?>> getEntry(ThreadLocal<?> key) {}
```

也就是<font color="red">threadlocal ref 和 threadlocal之间是弱引用关系</font>，所以当 gc 发生时候 ThreadLocalMap的key 就被回收了

### 2.1.2 导致内存泄露的原因

以ReentrantReadWriteLock中**ThreadLocalHoldCounter(维护每一个线程对应的读锁数量)**的使用来梳理内部结构

```java
// ThreadLocalHoldCounter <: ThreadLocal
ThreadLocalHoldCounter<HoldCounter> readHoldCounterByThread = new ThreadLocalHoldCounter<>();

HoldCounter holdCounter = new HoldCounter();

threadLocal.set(holdCounter);

threadLocal.remove();
```

![threadlocal.png](/static/images/charts/2021-03-03/threadlocal.png)

+ ThreadLocalMap key导致 ---> 线程闲置，但是内部的threadlocal仍然占用着空间，将ThreaLocalMap的key转换成WeakReference, GC时候直接回收ThreaLocal对象
+ ThreadLocalMap value导致 --->  根据上图的GC 可达性分析流程，第一步被清除之后，`value ref --> value object` 这一层还是存在的，所以还是有内存泄露的风险

### 2.1.3 对应解决方案

使用完成之后通过`threadlocal.remove`移除对应的映射

其实主要就是如何移除value的，源码中已经解释的很清楚了, 直接将对应的value引用指向null, 这样根据GC可达性就会被GC掉

```java
private int expungeStaleEntry(int staleSlot) {

    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    
}
```


```java
ThreadLocal<M> tl = new ThreadLocal<>();
tl.set(new M());
tl.remove();
```

基于上面的代码，借助下面的图可以很好的理解，如果ThreadLocal不是弱引用，那么即使tl不再指向`堆中的ThreadLocal对象`，但是key1(类型为ThreadLocal)仍然指向了`堆中的ThreadLocal对象`； 与此同时即使key1因为弱引用被GC了，但是value再也无法被访问到，因此依然存在内存泄露。

> 所以当ThreadLocal对象不再使用时，一定要调用`remove`将其移除

## 2.2 ThreadLocal的一些应用

+ Spring transaction事务相关的信息存储在ThreadLocal中，在实现事务同步的时候，会维护相应的维护，这个场景就非常适合ThreadLocal

```java
public abstract class TransactionSynchronizationManager {
    
    /**
     * ! 因为 事务 和  线程绑定， 所以同步操作 正好可以利用ThreadLocal
     * ! 一个事务 可以注册多个 同步逻辑
     */
	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");
    
}
```

## 2.3 InheritableThreadLocal 与继承性

通过 ThreadLocal 创建的线程变量，其子线程是无法继承的。也就是说你在线程中通过ThreadLocal 创建了线程变量V，而后该线程创建了子线程，你在子线程中是无法通过
ThreadLocal来访问父线程的线程变量 V 的。

如果你需要子线程继承父线程的线程变量，那该怎么办呢？其实很简单，Java 提供了InheritableThreadLocal 来支持这种特性，InheritableThreadLocal 是ThreadLocal子类，所以用法和 ThreadLocal 相同，这里就不多介绍了。

不过，**我完全不建议你在线程池中使用InheritableThreadLocal**，不仅仅是因为它具有
ThreadLocal 相同的缺点——可能导致内存泄露，更重要的原因是：**线程池中线程的创建是动态的，很容易导致继承关系错乱，如果你的业务逻辑依赖 InheritableThreadLocal**，那么很可能导致业务逻辑计算错误，而这个错误往往比内存泄露更要命

# 3. 软引用(Soft Reference) - java.lang.ref.SoftReference

相较于弱引用，**可以简单理解它会更坚挺，就是不会一GC就没有了**，当然也是在内存还充足的情况下

> `SoftReferences` aren't required to behave any differently than `WeakReferences`, but in practice softly reachable objects are generally retained as long as memory is in plentiful supply. This makes them an excellent foundation for a cache.

> If you only have `soft references` to an object (with no strong references), then the object will be reclaimed by GC only when JVM runs out of memory

> 如果对于某个对象，你仅仅只有软引用指向它，那么当JVM内存不足的时候就会尝试干掉它

一般用于实现 `memory-sentitive`缓存

```java
// -Xms20m -Xmx20m
SoftReference<Object> softReference = new SoftReference<Object>(new Object());
if (null == softReference.get()) {
    throw new IllegalStateException("Reference should NOT be null");
}

try {
    byte[] ignored = new byte[1024*1024*20];
} catch (OutOfMemoryError e) {
    // Ignore
    e.printStackTrace();
}

if (null != softReference.get()) {
    throw new IllegalStateException("Reference should be null");
}

System.out.println("It worked!");
```

当ignored尝试申请20m内存时，JVM就会尝试回收softReference

# 4. 虚引用(Phantom Reference)

基本不会使用到 -- 管理`堆外内存`  `DirectByteBuffer`


# 5. 参考

\> [Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)

\> [zhihu ThreadLocal的Entry为什么要继承WeakReference?](https://www.zhihu.com/question/458432418)

\> [Why does ThreadLocalMap.Entry extend WeakReference?](https://stackoverflow.com/questions/64585510/why-does-threadlocalmap-entry-extend-weakreference)

\> [深度解析 ThreadLocal 原理](https://xie.infoq.cn/article/eb09a8ddb6df4edf1505fa81c) 

\> [30 | 线程本地存储模式：没有共享，就没有伤害](https://time.geekbang.org/column/article/93745)
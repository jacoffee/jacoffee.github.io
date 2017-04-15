---
layout: post
category: jvm
date: 2016-05-25 18:08:55 UTC
title: Java虚拟机基础之CMS垃圾回收
tags: [CMS收集器，垃圾回收，新生代(Young Generation)，老生代(Old Generation)，Minor GC ，Major GC，Full GC]
permalink: /jvm/cms_garbage_collection
key: a80b0b869874833ac8485d2f472af6b8
description: 本文介绍了Java虚拟机的垃圾回收机制、使用到的算法以及CMS垃圾回收的过程
keywords: [CMS收集器，垃圾回收，新生代(Young Generation)，老生代(Old Generation)，Minor GC ，Major GC，Full GC]
---

在Java项目中，我们经常会在项目部署的时候打印垃圾回收(Garbage Collection)日志来帮助我们监控系统的运行状况。

```bash
java -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseConcMarkSweepGC \
-Xloggc:/Users/GC/jvm_gc.log -jar xxx/yyy.jar
```

在上面的配置中**-XX:+UseConcMarkSweepGC**表示垃圾回收使用的收集器，对应的还有**-XX:+UseSerialGC**，**-XX:+UseParallelGC**等。并发标记-清除收集器(Concurrent Mark and Sweep Collector)是基于**标记-清除**算法的。这种算法的基本思路就是先对于堆中的对象进行标记，将能够通过GC roots访问到的对象标记live，然后在接下来的阶段对于没有标记的对象进行清除。

我们可以通过如下命令获取当前Java虚拟机使用的垃圾回收器

```bash
java -XX:+PrintCommandLineFlags -version

# 通过jps获取java对应的PID
jmap -heap PID
```

## 垃圾回收算法

对于垃圾回收算法，目前已经有很多实现(Parellel、CMS、G1)，但大致会涉及到如下几个过程。

<ul class="item">
    <li>
        <b>标记可达对象(mark reachable objects)</b>，也就是通过GC roots能够访问到的对象然后标记成live。这个阶段往往会伴随应用程序的阻塞(Stop-The-World)。
    </li>
</ul>

<ul class="item">
    <li>
        <b>清除没用的对象(remove unused objects)</b>，这个阶段的实现算法大致可以分为如下三类:
    </li>
    <ul>
        <li>
            <b>标记-清除(mark-and-sweep): </b>也就是将之前没有被标记的对象直接清除。但是会存在一个问题，由于清除的对象所占用的空间可能并不是连续的，所以会造成大量碎片(fragment)，这样下次为新对象分配空间的时候，可能会由于空间不够(准确的说是连续的内存空间不够)而提前触发一次垃圾回收。
            <div>
                <img src="http://static.zybuluo.com/jacoffee/3kyyo1dithw1bdykj0arzwcq/image_1bcp8pp9l169nioe1t9q1v8a7ot9.png" alt="标记-清除图示">
            </div>
        </li>
        <li>
            <b>标记-清除压缩(mark-and-sweep-compact): </b>这个显然就是上一种算法的加强版。既然标记-清除会造成空间的碎片化，那如果我们能将所有的live对象移动到一起，那么剩下的内存空间就是连续的，就可以用于分配新对象了，在实际中一般是将live对象移动内存区域(memory region)的开头。但是这种方法会带来额外的开销，<b>增加了GC pause时间、复制live对象并且更新所有对于该对象的引用</b>。
            <div>
                <img src="http://static.zybuluo.com/jacoffee/eyuij0nx05fo382a26tk8qu6/image_1bcp8v8jfqh01n1p1a9v7tf19dkm.png" alt="标记-清除压缩">
            </div>
        </li>
        <li>
            <b style="color:red">标记复制: </b>它同样也移动了对象不过为那些需要移动的对象单独开辟了一块内存区域，另外一个优势就是它可以在<b>标记的同时并发的进行对象移动</b>
            <div>
                <img src="http://static.zybuluo.com/jacoffee/8i0b4unvsd7phbi91fpdxtvq/image_1bcp9ndi1fqh10ttupf1ds71o9l13.png" alt="标记复制">
            </div>
        </li>
    </ul>
</ul>


## 分代垃圾回收

目前大多数虚拟机的垃圾回收机制都采用了分代回收(Generational Collection)的思想，也就是将堆中的内存区域分为新生代和老生代，分别采用不同的机制进行垃圾回收，比如说对于对象存活率较低的新生代，我们可以采用使用复制算法的收集器(如Parallel New)，只用付出复制一小部分对象的代价。而对于存活率较高的老生代，一般很难再开辟出空间进行复制，所以一般采用了使用标记-清除(如CMS)或者标记压缩(如G1)的收集器。

<ul class="item">
    <li>
        <b>新生代垃圾回收: </b>通常被叫做Minor GC，它一般是在新生代区无法给新对象分配空间的时候触发的。
比如说ParNew是一种会导致应用线程阻塞(stop-the-world)并且会复制对象(复制到Survivor或Old区)的收集器。
    </li>
    <li>
        <b>老生代垃圾回收(Major GC)</b>，首先关于Full GC和Major GC这两个名词，Java虚拟机规范中并没有做出相应的规定。这里我们使用Major GC来指定老生代GC，使用Full GC来指代整个堆的GC。
    </li>
</ul>


## 并发标记-清除垃圾收集(Concurrnt Mark Sweep Garbage Collection)

下面主要通过日志来了解并发标记-清除收集器的具体过程。

```bash
# JVM GC相关参数(jdk 1.8)
-XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps
```

在这种配置下，新生代采用**Parallel New**(ParNew, 标记复制)进行垃圾回收，老生代采用CMS(并发标记-清除)的方法进行垃圾回收。

```bash
# 新生代GC
2016-05-25T12:02:28.988+0800: 76791.792: [GC2016-05-25T12:02:28.988+0800: 76791.792: 
[ParNew: 857276K->24956K(943744K), 0.0295430 secs] 3540527K->2709351K(5138048K), 0.0298600 secs] 
[Times: user=0.10 sys=0.00, real=0.03 secs] 

# 初始标记阶段 - stop the world
2016-05-25T12:02:29.020+0800: 76791.824: 
[GC [1 CMS-initial-mark: 2684394K(4194304K)] 2712309K(5138048K), 0.0220000 secs] 
[Times: user=0.03 sys=0.00, real=0.03 secs] 

# 并发标记阶段  
2016-05-25T12:02:29.042+0800: 76791.847: [CMS-concurrent-mark-start]
2016-05-25T12:02:29.432+0800: 76792.236: [CMS-concurrent-mark: 0.381/0.390 secs] 
[Times: user=0.92 sys=0.01, real=0.39 secs] 

# 并发的预清理阶段
2016-05-25T12:02:29.432+0800: 76792.237: [CMS-concurrent-preclean-start]
2016-05-25T12:02:29.478+0800: 76792.282: [CMS-concurrent-preclean: 0.045/0.046 secs] 
[Times: user=0.05 sys=0.00, real=0.04 secs] 
2016-05-25T12:02:29.478+0800: 76792.282: [CMS-concurrent-abortable-preclean-start]
CMS: abort preclean due to time 2016-05-25T12:02:34.670+0800: 76797.475: 
[CMS-concurrent-abortable-preclean: 4.686/5.192 secs] 
[Times: user=4.89 sys=0.06, real=5.20 secs] 
 
# 重新标记阶段 - again stop the world
2016-05-25T12:02:34.671+0800: 76797.476: [GC (CMS Final Remark) [YG occupancy: 331133 K (943744 K)]
2016-05-25T12:02:34.671+0800: 76797.476: [Rescan (parallel) , 0.3236910 secs]
2016-05-25T12:02:34.995+0800: 76797.800: [weak refs processing, 0.1028850 secs]
2016-05-25T12:02:35.098+0800: 76797.903: [class unloading, 0.0291390 secs]
2016-05-25T12:02:35.127+0800: 76797.932: [scrub symbol table, 0.0159020 secs]
2016-05-25T12:02:35.143+0800: 76797.948: [scrub string table, 0.0018620 secs] 
[1 CMS-remark: 2684394K(4194304K)] 3015528K(5138048K), 0.5157410 secs] 
[Times: user=1.45 sys=0.01, real=0.51 secs] 


# 并发清理阶段
2016-05-25T12:02:35.189+0800: 76797.993: [CMS-concurrent-sweep-start]
2016-05-25T12:02:38.625+0800: 76801.430: [CMS-concurrent-sweep: 3.410/3.437 secs] 
[Times: user=3.73 sys=0.05, real=3.43 secs] 

# 并发重置阶段
2016-05-25T12:02:38.626+0800: 76801.430: [CMS-concurrent-reset-start]
2016-05-25T12:02:38.636+0800: 76801.440: [CMS-concurrent-reset: 0.010/0.010 secs] 
[Times: user=0.01 sys=0.01, real=0.01 secs]
```

上面展示的是部分垃圾回收日志，下面我们分阶段进行解释:

![ParNew && CMS](http://static.zybuluo.com/jacoffee/1y7bmy3v41kx4ii2r1p7ffkj/image_1bcplls60s8d6541krv11opb7f9.png)

### 新生代垃圾回收(Minor GC)

```bash
2016-05-25T12:02:28.988+0800: 76791.792: 
[
  GC (Allocation Failure) 2016-05-25T12:02:28.988+0800: 76791.792: 
  [ParNew: 857276K->24956 K(943744K), 0.0295430 secs] 
  3540527K->2709351K(5138048K), 0.0298600 secs
] 
[Times: user=0.10 sys=0.00, real=0.03 secs] 
```

<ul class="item">
    <li>
        <b>76791.792: </b>垃圾回收开始的时间，相对于Java虚拟机的启动时间(start-up time)，也就是Java虚拟机启动之后76791.792秒之后触发了这次GC。
    </li>
    <li>
        <b>GC (Allocation Failure): </b> 表明触发垃圾回收的原因是对象无法被放入新生代。
    </li>
    <li>
        <b>ParNew: </b>垃圾回收所使用的收集器，上面已经提到过
    </li>
    <li>
        <b>857276K-> 24956K(943744K): </b>垃圾回收前后，新生代的内存占用; 括号内的是新生代的初始内存大小
    </li>
    <li>
        <b>3540527K->2709351K(5138048K): </b> 堆区在新生代垃圾回收前后的内存占用，以及初始内存大小
    </li>
    <li>
        <b>0.0298600 secs: </b> 收集器标记新生代中的对象并且将live对象移动到Survivor区或者是Survivor区够老的对象直接提升到老生代区，以及最后一些收尾工作的时间。
    </li>
    <li>
        <b>[Times: user=0.05 sys=0.01, real=0.03 secs]: </b> user=0.05，垃圾回收线程使用的总CPU时间; real=0.03，系统"停止"的时间也就是应用线程阻塞的时间。
    </li>
</ul>

垃圾回收完成之后，堆区内存占用从**3540527**变成了**2709351**，节省出**831176**;
新生代占用内存从**857276**变成了**24956**，节省出**832320**，这部分的对象被移动到了老生代。

### 老生代垃圾回收(Major GC)

<b class="highlight">(1) 初始标记(Initial Remark)</b>

标记GC Roots能够**直接**关联的对象，会导致应用线程阻塞(stop-the-world)。

**[GC [1 CMS-initial-mark: 2684394K(4194304K)] 2712309K(5138048K), 0.0220000 secs]**  
老生代占用内存(老生代分配内存) -> 堆占用内存(堆分配内存) -> 用时。

<b class="highlight">(2) 并发标记阶段(Concurrent Mark)</b>

由前一阶段标记过的对象出发，开始追踪(trace)老生代所有可达对象并且开始标记，会和应用线程并行。

**[CMS-concurrent-mark-start]** ==> 并发标记开始
**[CMS-concurrent-mark: 0.381/0.390 secs] [Times: user=0.92 sys=0.01, real=0.39 secs]**

0.390secs -> 并发标记用时，后面的**Times**对于并发阶段的参考意义并不大，因为这个阶段还有很多其它的事发生。
 
<b class="highlight">(3) 并发的预清理阶段(Concurrent Preclean)</b>

在并发标记的过程中，一些对象的引用已经发生改变，如果有对象的属性发生改变，则会被Java虚拟机标记为Dirty(也称之为Card Marking)。在预清理阶段，这些对象会被标记为live。该阶段为重新标记阶段提前做好一些准备。

<b class="highlight">(5) 可中止的并发预清理阶段(Concurrent Abortable Preclean)</b>

这一过程与前一个过程有点类似，不过侧重点是分担一些最后标记阶段的任务。之所以叫可中止的是因为在某些条件满足的时候就不再进行预清理过程了(重复执行的次数达到上限，超过时间上限等)

<b class="highlight">(6) 最终标记(Final Remark)</b>

同初始标记一样，也会导致应用线程阻塞。该阶段主要是对于老生代的对象(也包括之前被标记为Dirty的对象)再次遍历并进行标记。通常情况下会在新生代**占用量尽可能小**的情况下进行最后标记，这样可以避免连续的应用阻塞(stop-the-world)。

```bash
2016-05-25T12:02:34.671+0800: 76797.476: 
[
    GC (CMS Final Remark) [YG occupancy: 331133 K (943744 K)]
    2016-05-25T12:02:34.671+0800: 76797.476: [Rescan (parallel) , 0.3236910 secs]
    2016-05-25T12:02:34.995+0800: 76797.800: [weak refs processing, 0.1028850 secs]
    2016-05-25T12:02:35.098+0800: 76797.903: [class unloading, 0.0291390 secs]
    2016-05-25T12:02:35.127+0800: 76797.932: [scrub symbol table, 0.0159020 secs]
    2016-05-25T12:02:35.143+0800: 76797.948: [scrub string table, 0.0018620 secs] 
    [1 CMS-remark: 2684394K(4194304K)] 3015528K(5138048K), 0.5157410 secs
] 
[Times: user=1.45 sys=0.01, real=0.51 secs] 
```

<ul class="item">
    <li>
        <b>GC (CMS Final Remark) [YG occupancy: 331133 K (943744 K)]: </b>
        新生代在最后标记时的内存使用量和初始分配大小。
    </li>
    <li>
        <b>[Rescan (parallel) , 0.3236910 secs]: </b> 应用线程阻塞，然后去标记老生代中的对象，用时0.3236910秒。
    </li>
    <li>
        <b>[weak refs processing, 0.1028850 secs]: </b>弱引用的处理以及用时
    </li>
    <li>
        <b>[class unloading, 0.0291390 secs]: </b>类卸载(与装载相对应)以及用时
    </li>
    <li>
        <b>[scrub symbol table, 0.0159020 secs], [scrub string table, 0.0018620 secs]: </b> 符号表以及字符串表的清理(具体内容包括类级别的元数据)
    </li>
    <li>
        <b>[1 CMS-remark: 2684394K(4194304K)] 3015528K(5138048K): </b> 标记完成之后的老生代内存占用和初始分配内存大小，堆内存占用和初始分配大小。
    </li>
</ul>

<b class="highlight">(7) 并发清除(Conccurent Sweep)和重置(Conccurent Reset)</b>

开始清理那些没有被标记为live对象，也就是Java虚拟机认为无用的对象。清除完成之后，进行CMS收集器内部数据结构的调整以准备下一阶段的GC。


## 参考

\> [Oracle官网对于CMS的介绍](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)

\> Plumbr Handbook Java Garbage Collection

\> [CMS GC日志概览](https://blogs.oracle.com/poonam/entry/understanding_cms_gc_logs)

\> [新生代空间分配失败(Allocation Failure)](https://greencircle.vmturbo.com/community/products/blog/2016/02/12/understanding-gc-allocation-failure-messages-in-the-logs)

\> [为什么Full GC需要STW](http://stackoverflow.com/questions/16695874/why-does-the-jvm-full-gc-need-to-stop-the-world)

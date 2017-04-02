---
layout: post
category: jvm
date: 2015-04-29 15:38:55 UTC
title: Java虚拟机基础之运行时数据区
tags: [运行时数据区, 堆, 虚拟基栈, 本地方法栈, 方法区, 新生代, 老生代, 标记复制]
permalink: /jvm/runtime_data_areas
key: 2c94cda87471610c670373353a3b43b2
description: 本文介绍了Java虚拟机运行时数据区的组成
keywords: [运行时数据区, 堆, 虚拟基栈, 本地方法栈, 方法区, 新生代, 老生代, 标记复制]
---

对于大多数Java程序员来说，如果不去关注Java虚拟机方面的东西，可能对于Java虚拟机内存结构最熟悉一句话的就是**本地变量分配在栈上，类实例分配在堆上**。但如果阅读《Java虚拟机指南》就会发现它包含的每一个部分都可以单独拿出来研究好久，所以本文只是对Java虚拟机运行时数据区的一个概述。

Java虚拟机运行时数据区大致如下图所示:

![运行时的数据区](http://static.zybuluo.com/jacoffee/4xx5mebu4d935hmsx5oqowed/image_1bckucvas1bdu1phi1fsj14526jm9.png)

##程序计数器(The pc Register)

可以理解为线程执行字节码的**行号指示器**，在查看字节码的时候我们可以看到各种数字。字节码解释器就是通过改变计数器来执行相应的字节码指令。

```bash
 0: iconst_0
 1: istore_1
 2: ldc           #2                  // String allen
 4: astore_2
 5: bipush        26
 7: istore_3
```

在Java虚拟机多线程的情况，线程轮流切换并分配处理器时间来执行。在任何确定时间，一个处理器(如果多核，就是一个内核)都会只会执行一个线程的指令。那么当线程切换之后要知道上一次的执行位置，就需要为每一个线程分配一个区域去记录这个位置以便回来之后能够继续执行。

##虚拟机栈(JVM Stacks)

> Each Java Virtual Machine thread has a private Java Virtual Machine stack, created at the same time as the thread

**虚拟机栈是线程私有的与线程的生命周期相同**。每一个方法(main方法，类中的其它方法)在执行的时候都会创建一个**栈帧**来存储局部变量，操作数栈等。栈帧从入虚拟机栈到出虚拟机栈也对应着一个方法的执行完成。

这个栈实际上就是我们经常提到的那个栈。

![图例1. Java虚拟机栈](http://static.zybuluo.com/jacoffee/5oju39tesa0qlacggc2e4953/image_1aqufoa34ojufg514971q3clm5m.png)

##本地方法栈(native method stack)</b>

虚拟机栈为Java虚拟机执行Java方法(字节码)服务，而本地方法栈则为Java虚拟机执行Native方法服务

##堆(Heap)

这个区域可能是我们日常中接触最多的，因为几乎任何Java项目都会配置相应的堆参数(形如-Xmx2048, -Xms2048); 堆的内存区域大致可以分为新生代(young generation)和老生代(old generation)，其中新生代又可以细分为(Eden区、Survivor区)。

![图例2. Java堆区结构图](http://static.zybuluo.com/jacoffee/r4dqfvrfi889gyo13hsk4lzk/image_1bcmii0ti37dv4418tr1lhr13t09.png)

<b class="highlight">(1) Eden区</b>

> The heap is the runtime data area from which memory for all class instances and arrays are allocated

按照Java虚拟机规范中的解释，堆区主要用来放类实例以及数组的。当Java虚拟机创建新对象的时候，它们会被分配在Eden区。通常来说，有很多线程同时创建很多新的对象，所以Eden区又被划分成了一个个**局部线程分配缓冲区(Thread local allocation buffer)**，这样单个线程就可以直接将新创建的对象放到对应TLAB中，避免同步的开销。

当TLAB没有足够的空间时，新创建的对象就会被放到**共享Eden区**，当该区域的空间也不足时，就会触发新生代GC，如果GC之后还不足以为新对象腾出空间，那么**新对象就会被放动到老生代**。

当Java虚拟机对于Eden进行GC的时候，它会从各种GC Roots开始遍历所有可达对象，并将它们标记成live。
当所有遍历完成之后，所有live对象将会被复制到Survivor区中去，所以这种GC方法也被成为标记复制法(mark and copy)。

![图例3. GC Roots](http://static.zybuluo.com/jacoffee/iajkf0qdnx54wc6dafqbv4m7/image_1aqm0np421erq1a616phlge111m9.png)

典型的GC Roots对象包括:

<ul class="item">
    <li>
        当前执行线程中的局部变量引用的对象(Person p = new Person())
    </li>
    <li>
        堆区中静态属性所引用的对象(public static Person p = new Person())
    </li>
    <li>
  活跃的线程对象(Active Thread)，新生成的并且没有被终止的线程，那么所有之后放到栈上的引用变量都可以通过它进行可达性分析。  
    </li>
    <li>
    本地方法栈中的JNI(Java Native Interface)引用的对象
    </li>
</ul>

**注**: 关于静态属性的归属，在《深入理解Java虚拟机》中提到的是"方法区中类的静态属性引用的对象"(P64)。后来查阅资料后发现，首先对于静态属性位于何处，Java虚拟机规范是没有硬性规定的，但结合stackoverflow上的相关回答以及[Java PermGen 去哪里了?](http://ifeve.com/java-permgen-removed/)这篇文章的观点，在Java 7以及以后的版本，**静态属性(class statics)应该是在堆上的**。

<b class="highlight">(2) Survivor 区</b>

在Eden区的旁边有两个区叫做Survivor区，一个叫做from，一个叫做to。**实际上它们两个总有一个是空的**。空的那个在下次新生代GC的时候就会有对象放进来，所有live对象(**包括Eden中的和from中的**)都会被放到to区。整个过程完成之后，to Survivor区就包含了from区没有的对象。然后交换角色，有点类似于两个量筒，不断将水转移到另外一个空的里面去。

在from和to之间，对象的复制会反复进行好几次直到有些对象被认定为已经足够"老"，而可以被放入到老生代去了。从实现来看，Java虚拟机会记录某个对象经历的GC次数，每一次GC之后还存活的对象，它们的年龄会被加1，当对象年龄超过一定的值(**-XX:+MaxTenuringThreshold=number**, 设置为0会导致对象直接被放入到老生代，而不用在Survivor区之间进行复制，一般来说是15)就会被放入老生代中去。

当然，如果这种提升(Survivor to Old)也可能会因为Survivor区的空间不足于存放新生代的所有live对象而提前发生的。

<b class="highlight">(3) Old Generation区</b>

老生代区通常会比较大，它包含的是不太可能被回收的对象而且实现上也更复杂。

在老生代区发生的GC并不像新生代那样频繁。此外由于老生代中的**大多数**对象都被认定为是live的，所以也并没有标记复制发生，取而代之的是移动对象来减小碎片化，这种减小空间占用算法会根据实现有所不同，但大体的原则如下:

<ul class="item">
    <li>给能够通过GC roots访问的对象设置marked标志位</li>
    <li>删除不可达对象</li>
    <li>通过将连续的live对象复制到老生代区的开头来压缩老生代区的空间</li>
</ul>

<b class="highlight">(4) MetaSpace</b>

> Metaspace is NOT part of Heap. Rather Metaspace is part of Native Memory (process memory) which is only limited by the Host Operating System.

在Java 8之前永久代(PermGen)是堆区的一部分用来存放**类相关信息**，还有像internalized 字符串之类的东西。这也就是我们经常看到的异常**java.lang.OutOfMemoryError: Permgen space**的来源。在Java 8中该部分被移除，使用了一个新的区域MetaSpace并且直接放在了OS内存。

如果我们希望设置相应的参数来控制MetaSpace大小，则可以通过如下命令:

```bash
java -XX:MaxMetaSpaceSize=256m com.xxx.xxx
```

##方法区(Method Area)

> It stores per-class structures such as run-time constant pool, field and method data, the code for methods and constructors

**它用于存放虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等**，和堆一样也被所有的线程共享。而在Hot VM中则是使用永生代(PermGen)来实现的，但是在Java 7、8版本中，该区域经历了较大的变动，包括内部一些东西的迁移，比如说静态变量放回到堆中等。

每一个类或接口都有常量池，在**.class**中表现为contant_pool表，通过**javap -verbose classname**命令可以观察到。运行时常量池在类被加载到虚拟机之后被创建，位于方法区，也就是常量池在运行时的表现。它包括若干种不同的常量: **编译期可知的数值字面量**(28l, 8.9d, 4.0f等)，在运行期间解析之后才能获得**方法或字段引用**(常量池表中的Methodref，Fieldref)。

```bash
Constant pool:
   #1 = Methodref          #5.#17         // java/lang/Object."<init>":()V
   #2 = Methodref          #18.#19        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #3 = Fieldref           #4.#20         // Test.x:I
   #4 = Class              #21            // Test
   #5 = Class              #22            // java/lang/Object
   ...
```

##参考

\> 深入理解Java虚拟机

\> The Java Virtual Machine Specification Java SE 8 Edition

\> [方法区的Class信息,又称为永久代,是否属于Java堆？](https://www.zhihu.com/question/49044988)

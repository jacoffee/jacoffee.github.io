---
layout: post
category: scala
date: 2015-03-16 09:51:22 UTC
title: Scala基础问题汇总
tags: [修饰符]
permalink: /scala/questions/
key: b801553a6ab218a20268697af6676ce5
description: "本文收集一些Scala基本问题的解答"
keywords: [修饰符]
---

## 1.类和对象

### 1.1 作为属性修饰符，private[this]和private的区别

```scala
class Foo {
    private var i = 0 
    private[this] var j = 0 
    def iThing = i * i 
    def jThing = j * j 
}
```

访问权限层面，前者仅限当前类实例中访问，后者稍微宽松一点能在类伴生对象。两者都不能在新生成的类实例中访问。

```scala
object Foo {
  (new Foo).i // Okay
  (new Foo).j // Not Okay
}
object Bar {
  (new Foo).i // Not Okay
  (new Foo).j // Not Okay
}
```

字节码层面，两者的调用机制不一样，前者是在方法访问的时候直接调用的属性，而后者调用的是编译器生成的属性对应的getter(如下所示)。个人猜想，因为private[this]属性是**当前实例才能访问的，不需要向外暴露什么**，所以没有getter，直接访问属性。但private属性在伴生对象中还可以访问，所以需要提供getter。

```scala
// 部分反编译的代码
public int iThing() { return i() * i(); } 
public int jThing() { return this.j * this.j; }
```

### 1.2 case object与object的区别

```scala
case object Message
object Message1
```

前者之于后者，可以用于模式匹配的Pattern，这一点在Akka Actor中作为消息使用的特别多;
继承了**Serializable**，在网络通讯中可以被序列化，这一点同样在Actor的消息传递时体现的非常明显，特别是当Actor位于不同的机器上的时候;
类似于case class提供了**toString**的默认实现。在实际编程中，由于经常会接触到Actor，所以第二点体现的更明显。

通过反编译的代码，能更清楚的看到区别:

```java
// Message$
public final class Message$ implements Product, Serializable {
  public static final MODULE$;
  ...
  public String toString() { return "Message"; } 
  ...
}
```

### 1.3 this与this.type的区别

```scala
def -= (s: String): this.type  = { remove(s); this }
```

this -- 当前对象的引用

## 2.命令行技巧

(1) `:paste`, `:kind`, `:type`, `:settings`, `:javap`等命令使用

`:paste`用于输出多行代码，可以简写成`:pa`

```bash
scala> :pa
// Entering paste mode (ctrl-D to finish)

   val seriesX: RDD[Double] = sc.parallelize(List[Double](2, 4, 5))
   val seriesY: RDD[Double] = sc.parallelize(List[Double](5, 10, 12))

   val correlation: Double = Statistics.corr(seriesX, seriesY)

// Exiting paste mode, now interpreting.
```

`:kind`用于打印类型的信息

```scala
scala> :kind -v Int
scala.Int's kind is A
*
This is a proper type.

scala> :kind -v Either
scala.util.Either's kind is F[+A1,+A2]
* -(+)-> * -(+)-> *
This is a type constructor: a 1st-order-kinded type

scala> class Ref[M[_]]
warning: there was one feature warning; re-run with -feature for details
defined class Ref

scala> :kind -v Ref[List]
Ref's kind is X[F[A]]
(* -> *) -> *
This is a type constructor that takes type constructor(s): a higher-kinded type
```

上面的打印出的信息告诉我们`Int`的类型是A, 并且是一个特定类型(参见[Scala基础之一阶类型 vs 高阶类型](http://localhost:4000/scala/first-order-higher-types/))。
`Either`是类型构造器并且是一阶类型, `Ref`是类型构造器并且是高阶类型。

`:settings`用于查看警告详情，上面代码有一行提示: 加上**-feature**重新运行一下然后查看详情。

```scala
scala> :settings -feature

scala> class Ref[M[_]]
<console>:7: warning: higher-kinded type should be enabled
by making the implicit value scala.language.higherKinds visible.
This can be achieved by adding the import clause 'import scala.language.higherKinds'
or by setting the compiler option -language:higherKinds.
See the Scala docs for value scala.language.higherKinds for a discussion
why the feature should be explicitly enabled.
       class Ref[M[_]]
                 ^
defined class Ref
```

上面的信息告诉我们应该显示的引入高阶类型

`:javap`可以用于获取字节码的相关信息


```scala
scala> object Foo { def apply = 10; def bar = 5 }
defined object Foo

scala> :javap Foo#apply
  public int apply();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: bipush        10
         2: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       3     0  this   LFoo$;
      LineNumberTable:
        line 7: 0

```


## 参考

\> [private[this] vs private的字节码区别](https://gist.github.com/twolfe18/5767545)

\> [实用的Console命令](http://docs.scala-lang.org/scala/2.11/)


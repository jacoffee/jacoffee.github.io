---
layout: post
category: scala
date: 2016-03-07 10:29:03 UTC
title: Scala基础之路径依赖类型以及类型投影
tags: [路径依赖类型，类型投影，类型相等检验]
permalink: /scala/path-dependent-type/
key: "f94b41f03d3fedce821eaa2fd106853e"
description: "本文研究了Scala的路径依赖以及类型投影"
keywords: [路径依赖类型，类型投影，类型相等检验]
---

在Scala中我们经常会使用**O.T**(路径依赖类型)以及**O#T**(类型投影)来表达类型。由于Scala中有强大的类型系统加上平时很少去封装一些复杂的结构，所以并没有系统地去了解这两种类型的区别。

### 路径依赖类型

它可以看做是对象的成员所指向的类型(Scala In Depth中有这样一句话It refers to a type found on a specific object instance)，之所以路径依赖是因为如果**没有对象或者是类的实例**，那路径依赖类型也无从谈起；另外一点就是由于依赖了当前的实例，所以**该类型便不能绑定在其它的实例上**。

```scala
class Outer {
    trait Inner 
    def y = new Inner {}
    def foo(x: this.Inner) = null
    def bar(x: Outer#Inner) = null
}

scala> val x = new Outer
x: Outer = Outer@7960847b

scala> val y = new Outer
y: Outer = Outer@2db0f6b2
```

当实例x去调用相应的foo时，它实际上是

```scala
def foo(x: x.Inner) = null
```

所以才会出现下面的情况:

```scala
scala> x.foo(x.y)
res0: Null = null

scala> x.foo(y.y)
<console>:11: error: type mismatch;
 found   : y.Inner
 required: x.Inner
              x.foo(y.y)
                      ^
```

### 类型投影

它相对于路径依赖类型减少了一些限制，并不依赖类的实例。针对上例就是，路径依赖类型`Outer#Inner`，它指代的类型是任何Outer实例中的任何Inner类型

```scala
scala> x.bar(y.y)
res0: Null = null
```

实际上，类型依赖的类型都可以使用类型投影来代替，如下例。


```scala
scala> class Foo { type T = String }
defined class Foo

scala> val foo = new Foo
foo: Foo = Foo@182decdb

scala> implicitly[foo.type#T =:= foo.T]
res1: =:=[foo.T,foo.T] = <function1>
```

实际上类型```=:=[foo.type#T, foo.T]```对应的是一个带类型参数的类。 所以，implicitly实际上触发了一次**显示的隐式转换的查找**(
在所有可能的范围内，能否找到一个```=:=[foo.type#T, foo.T]```类的隐式实例)。

由于```scala.Prefef```中已经在```=:=```的伴生对象中定义了一个隐式转换， 如下代码:

```scala
implicitNotFound(msg = "Cannot prove that ${FROM} =:= ${TO}")
sealed abstract class =:=[From, To] extends (From => To) with Serializable
private[this] final val singleton_=:= = new =:=[Any,Any] { def apply(x: Any): Any = x }
object =:= {
 // implicit def tpEquals[A]: =:=[A, A] = singleton_=:=.asInstanceOf[A =:= A]
 implicit def tpEquals[A]: A =:= A = singleton_=:=.asInstanceOf[A =:= A]
}
```

显然只有在中置操作符前后的两个类型一致，```singleton_=:=.asInstanceOf[A =:= A]```， 这部分才能通过，则隐式转换才能被成功锁定，也间接的证明两个类型相等。
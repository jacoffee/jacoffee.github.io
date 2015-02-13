---
layout: post
category: scala
date: 2015-02-10 21:03:04 UTC
title: Scala基础之类型类(type class pattern)
tags: [Haskell, 类型类]
permalink: /scala/type-class/
key: 45e8dc0d31da82335dc1f50ae63c3f33
description: "本文研究了Haskell类型类的概念以及类型类的概念在Scala Context Bound的运用"
keywords: [Haskell，类型类，上下文定界]
---

前几天在写[Scala基础之隐式转换]({% post_url 2015-02-08-implicits %})的时候，在隐式类型中实际上还有一种叫Context Bound(上下文定界)，由于当时涉及到另外一个概念Type class pattern(类型类)并且自己不熟，所以并没有写出来。今天特意研究了一下，决定写下来。

# 问题
什么是类型类？它在Scala中的主要应用是什么？

# 解决
<h3 style="text-indent: 25px;">基本概念</h3>
首先Type class pattern是来源于Haskell的, 一种纯函数式的语言。Scala从它里面借鉴了很多思想。
由于本人也没有学习过Haskell，所以下面主要介绍类型类的思想，如果对于Haskell感兴趣的话，可以去这个网站上[Real World Haskell](http://rwh.readthedocs.org/en/latest/)看看。

> Typeclasses define a set of functions that can have different implementations depending on the type of data they are given

类型类定义了一套函数，它们会根据类型的不同而有不同的实现。

eg1: 比较颜色的不同，Haskell的作者并<b style= "color: red">没有提供"=="方法</b>(感觉有点不可思议)，所以需要自己定义。

```Haskell
data Color = Red | Green | Blue

colorEq :: Color -> Color -> Bool 
/*
    类似于Scala中的
    def colorEq(c1: Color, c2: Color): Boolean
*/
colorEq Red   Red   = True
colorEq Green Green = True
colorEq Blue  Blue  = True
colorEq _     _     = False

// 调用
colorEq Red Green // false
colorEq Green Green // true
```

eg2: 比较各种角色是否相同

```Haskell
data Role = Boss | Manager | Employee

roleEq :: Role -> Role -> Bool
roleEq Employee Employee = True
roleEq Manager Manager = True
roleEq Boss Boss = True
roleEq _ _  = False

// 调用
roleEq Employee Employee // true
roleEq Employee Boss // false
```

上面两种写法分别针对Color和Role类型进行相等性的比较，如果有更多的类型需要进行相等性的比较，**那我们需要定义更多的相等性比较的规则**。

如果能定义一套通用的函数就是<b style="color:red">用于比较</b>，那么我们代码的通用性也会增强。只需要让编译器知道如何对传入的类型进行比较即可。

```Haskell
-- Haskell的class与一般语言的class不是同一个概念，它用于定义类型类
class BasicEq x where
    isEqual x -> x -> Bool
```

<b style="color:red">上面可以理解为对于任何类型x，只要它是BasicEq的实例，isEqual就会接收两个x类型的参数并且返回Bool。</b>

下面定义一种类型类的实例，我们为Bool和之前的Color定义是否相等的规则。

```Haskell
instance BasicEq Bool where
    isEqual True True = True 
    isEqual False False = True 
    isEqual _ _ = False 
    
instance BasicEq Color where
    isEqual Red   Red   = True
    isEqual Green Green = True
    isEqual Blue  Blue  = True
    isEqual _     _     = False
```

按照上面的情况，我们可以用isEqual去比较<b style="color:red">任何类型</b>，只要传入的类型是BasicEq的实例。这种结论正好呼应了开头提到的类型类的概念，定义一套函数(isEqual)，根据类型不同(Bool, Color)而有不同的实现(Bool类型相等的规则和Color相等的规则是不一样的)。

<h3 style="text-indent: 25px;"> 在Context Bound(上下文定界)中的应用</h3>

> Context bounds were introduced in Scala 2.8.0, and are typically used with the so-called type class pattern, a pattern of code that emulates the functionality provided by Haskell type classes, though in a more verbose manner.

上下文定界是在Scala 2.8中引入进来的，它通常被当作类型类去使用，实际上是从Haskell中借鉴的但是用了<b style="color:red">更繁杂的方法</b>去实现的。

```scala
def f[A: B](a: A) = g(a)
```
<b style="color:red">对于任何类型A，都有一个隐式转换使之变成B[A]。然后调用B[A]的某个方法g，并且传入类型为A的参数a。</b>

eg: scala.math.Ordered就是一个比较典型的例子

```scala
def simpleF[A: Ordering](x: A, y: A) = implicitly[Ordered[A]].compare(x, y)

simpleF(3, 4) // -1 即 3 < 4   
```

simpleF还有一种等价的写法。

```scala
def verboseF[A](x: A, y:A)(implicit ord: Ordering[A]) = {
    // public static <A> boolean verboseF(A paramA1, A paramA2, Ordering<A> paramOrdering)
    import ord._ // 引入Ordering中的隐转
    x < y // ord.mkOrderingOps(x)
}
```

针对上面那个例子，有三点需要说明。

<1> simpleF的Ordering没有带参数类型，因为它实际上是verboseF的语法糖。

<2> implicitly是为了获取我们需要的类型Ordered[A]，有点"显示"触发隐转的味道。

<3> Int => Ordered[Int]的过程

```scala
// Int => RichInt, 这个隐转定义在LowPriorityImplicits中，
// 并且被Predef继承。另外RichInt实际上是Ordered的子类。
@inline implicit def intWrapper(x: Int) = new runtime.RichInt(x)
```

这样Int是Ordered的"间接实例"(因为Int被转换成了RichInt，这也就是上面所说的更繁杂的意思吧)，所以Int可以被作为参数传进去。

> Basically, this pattern implements an alternative to inheritance by making functionality available through a sort of implicit adapter pattern.

这种类型类的方式就好像给继承提供了一种补充方案，Int并不是Ordered[Int]的实例。但是通过隐转，Int就能够作为类型A被传进去。

# 结语
一时语塞，尽在文中。

# 参考

<1> [Int to RichInt](http://stackoverflow.com/questions/7669627/scala-source-implicit-conversion-from-int-to-richint)

<2> [Context and View Bound](http://docs.scala-lang.org/tutorials/FAQ/context-and-view-bounds.html)


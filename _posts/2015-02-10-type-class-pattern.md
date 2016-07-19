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

前几天在写[Scala基础之隐式转换]({% post_url 2015-02-08-implicits %})的时候，在隐式类型中实际上还有一种叫上下文定界(Context Bound)，由于当时涉及到另外一个概念类型类模式(Type class pattern)

首先类型类模式是来源于Haskell的, 一种纯函数式的语言。Scala从里面借鉴了很多思想。
由于自己并没有接触过Haskell，所以下面主要介绍类型类的思想，如果对于Haskell感兴趣的话，可以参考这本书<<[Real World Haskell](http://rwh.readthedocs.org/en/latest/)>>。

类型类定义了一系列函数，它们会根据类型的不同而有不同的实现。

案例1: 比较颜色的不同，Haskell中并没有提供<b style= "color: red">==</b>方法，所以需要自己定义。

```haskell
-- 定义新的类型Color(在Haskell中叫做类型构造器)，该类型有三种值构造器(也可以理解为该类型的三种实例)，并且都不用接受参数
data Color = Red | Green | Blue

-- 定义新的函数类型 类似于Scala中的def colorEq(c1: Color, c2: Color): Boolean
colorEq :: Color -> Color -> Bool 

colorEq Red   Red   = True
colorEq Green Green = True
colorEq Blue  Blue  = True
colorEq _     _     = False

-- 调用
colorEq Red Green // false
colorEq Green Green // true
```

上面的Color类型定义在Scala中可以表达为:

```scala
sealed trait Color
case object Red extends Color
case object Green extends Color
case object Blue extends Color
```

案例2: 比较各种角色是否相同

```haskell
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

上面两种写法分别针对Color和Role类型进行相等性的比较，如果有更多的类型需要进行相等性的比较，**那我们需要定义更多的相等性比较的规则**。所以需要定义一套通用的函数<b style="color:red">用于比较</b>。

```haskell
-- Haskell的class与面对对象语言的class不是同一个概念，它用于定义类型类
class BasicEq x where
    isEqual x -> x -> Bool
```

<b style="color:red">上面的代码可以理解为对于任何类型x，只要它是BasicEq的实例，isEqual就会接收两个x类型的参数并且返回Bool。</b>

下面定义一种类型类的实例，我们为Bool和之前的Color定义是否相等的规则。

```haskell
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

按照上面的情况，我们可以用isEqual去比较<b style="color:red">任何类型</b>，只要传入的类型是BasicEq的实例。这种结论正好呼应了开头提到的类型类的概念，定义一系列函数(isEqual)，根据类型不同(Bool, Color)而有不同的实现(Bool类型相等的规则和Color相等的规则是不一样的)。

**上下文定界**通常被当作类型类模式去使用，实际上是借鉴了Haskell的类型类但是使用了更复杂的方式去实现。

```scala
def f[A: B](a: A) = g(a)
```

对于任何类型A，都有一个隐式转换使之变成B[A]。然后调用B[A]的某个方法g，并且传入类型为A的参数a。`scala.math.Ordered`就是一个比较典型的例子。

```scala
def simpleF[A: Ordering](x: A, y: A) = implicitly[Ordering[A]].compare(x, y)

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

我们可以看到类型类就好像给继承提供了补充方案，通过隐式转换A类型被转换成了B[A]类型，同时拥有了B[A]类型的各种方法。

## 参考

\> [上下文定界限和视界](http://docs.scala-lang.org/tutorials/FAQ/context-and-view-bounds.html)


---
layout: post
category: scala
date: 2015-08-15 09:51:22 UTC
title: Scala基础之一阶类型 vs 高阶类型
tags: [类型推断，existential类型，一阶类型, 高阶类型]
permalink: /scala/first-order-higher-types/
key: 9730097b5f850e65a71abab10cbce2aa
description: "本文解释了Scala中一阶类型和高阶类型的区别"
keywords: [类型推断，existential类型，一阶类型, 高阶类型]
---

最早有一阶类型(first-order type)和高阶类型(higher-kinded type)的概念源于Stackoverflow上的一个[关于类型的帖子](http://stackoverflow.com/questions/6246719/what-is-a-higher-kinded-type-in-scala/6427289#6427289), 很多答者从不同的角度说明了这两个概念。在我看来类型的基本功能在于**抽象**，从本质上来讲，这两者的区别在与抽象能力的不同。

在[Scala的基础问题汇总](/scala/questions/)中提到过一个Console命令**:kind**用于查看类型信息，我们可以以此为原型进行展开。

```scala
scala> :kind -v Int
scala.Int's kind is A
*
This is a proper type.

scala> :kind -v List 
scala.collection.immutable.List's kind is F[+A]
* -(+)-> *
This is a type constructor: a 1st-order-kinded type

scala> :kind -v List[_]
scala.collection.immutable.List's kind is F[+A]
* -(+)-> *
This is a type constructor: a 1st-order-kinded type.

scala> :kind -v Ordering[_]
scala.math.Ordering's kind is F[A]
* -> *
This is a type constructor: a 1st-order-kinded type.

scala> class Ref[M[_]]
scala> :kind -v Ref[List]
Ref's kind is X[F[A]]
(* -> *) -> *
This is a type constructor that takes type constructor(s): a higher-kinded type
```

在展开之前，解释一个概念Proper type(**特定类型**)或者说是Monomorphic type(单态类型与之相对应的是多态类型Polymorphic type)
，它指的是那些不需要类型参数的类型，比如说原生类型`Int，Long，Double`，自定义的类`class Foo, class Bar`。作为类型构造器的一阶类型和高阶类型，它们都需要传入类型参数才能生成一个类型。

**(1) 一阶类型**

一阶类型就是针对**特定类型**的抽象，通过给**类型构造器**提供**特定类型**来得到一个具体的(concrete)类型

```scala
List
class Ref[T] {} // Ref属于一阶类型
Option
```

**(2) 高阶类型**

高阶类型就是在**对特定类型抽象的类型**的基础上再进行抽象，也就是针对一阶类型的再次抽象；
高阶类型就是对**类型构造器**进行抽象的类型。

```scala
class Ref[M[_]] // Ref属于高阶类型
trait Functor[F[_]] // Functor属于高阶类型
trait Apply[F[_]] // Apply属于高阶类型
```

对于`type MyMap = Map[String, List[String]]`，MyMap并不是高阶类型，因为不需要传入任何的类型参数；
对于`type MyMap[A] = Map[A, List[String]]`, MyMap也不是高阶类型而是一阶类型，因为A不能再接受类型参数，也就是不再具有抽象的能力。


下面通过上下文定界中的一个实例来进一步理解这两个概念，在[Scala基础之类型类(type class pattern)](/scala/type-class/)中讲到过上下文定界。

```for [A: T]，编译器将会尝试寻找形如T[A]的隐式类型实例```

在这个地方，我们还需要考虑到`T`能够接受的类型。`scala.reflect.ClassTag`是一阶类型，所以它只能接受特定类型`T`，因此下面的第二种情况无法通过编译。

```scala
scala> def func[C: ClassTag] = implicitly[ClassTag[C]]
func: [C](implicit evidence$1: scala.reflect.ClassTag[C])scala.reflect.ClassTag[C]
```

```scala
scala> def func[CC[_]: ClassTag] = implicitly[ClassTag[CC]]
<console>:8: error: type CC takes type parameters
       def func[CC[_]: ClassTag] = implicitly[ClassTag[CC]]
                                                       ^
```

我们可以使用类型别名(type alias)来解决这个问题:

```scala
type HigherOrdering[CC[T]] = ClassTag[CC[_]]
def func[CC[_]: HigherOrdering] = implicitly[HigherOrdering[CC]]
def func1[CC[_]: HigherOrdering](implicit evidence: HigherOrdering[CC]) = evidence

// One liner
def func2[CC[_]: ({type λ[CC[T]] = ClassTag[CC[_]]})#λ] = implicitly[ClassTag[CC[_]]] 

scala> func2[List]
res0: scala.reflect.ClassTag[List[_]] = scala.collection.immutable.List

scala> func2[Option]
res1: scala.reflect.ClassTag[Option[_]] = scala.Option

scala> func2[Int]
<console>:10: error: Int takes no type parameters, expected: one
              func2[Int]
                    ^
```

注意第二种写法，func1中`evidence`的类型是**```HigherOrdering[CC]```**，而不能是**```HigherOrdering[CC[_]]```**，HigherOrdering类型需要接受**一个可以接受类型参数的类型**。


##参考

\> [使用高阶类型定义上下文定界](http://stackoverflow.com/questions/5541154/context-bounds-shortcut-with-higher-kinded-types#)

\> [类型构造器的推断](http://adriaanm.github.io/research/2010/10/06/new-in-scala-2.8-type-constructor-inference/)


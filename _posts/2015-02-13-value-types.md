---
layout: post
category: scala
date: 2015-02-13 15:29:03 UTC
title: Scala基础之值类型(value types)
tags: [值类型，类型投影]
permalink: /scala/value-types/
key: 0e78a72174e18bd5eb0b22b1dfc6f75a
description: "本文研究了Scala值类型及基本的运用"
keywords: [值类型，类型投影，类型指示器，参数类型，元祖类型，带注解的类型，复合类型，中置类型，函数类型，存在类型]
---

在Scala中每一个值都是有类型的，具体如下:

###  单例类型(Singleton Types)

基本格式: **p.type** 

说明: p必须是<b style="color:red">scala.AnyRef</b>的子类。这种通常用于返回自身类型，从而用于链式调用。

```scala
abstract class Abstract0 { self =>
    def set(j: Int): self.type
}

class Concrete0 extends Abstract0 {
    var i = 0
    def set(j: Int) = {
        i = j
        this
    }
    def get = i
}

(new Concrete0).set(3).get
```

###  类型投影(Type Projection)

基本格式: **T#X**

说明: X必须是T的"类型成员"

```scala
# Outer#Inner
class Outer {
    trait Inner 
    def y = new Inner {}
}
```

###  参数化类型(Parameterized Types)

基本格式: T[U1, U2, U3]

说明: 类型参数U1到Un，n>=1。T必须是一个<b style="color:red">类型的构造器</b>并且接受n个类型参数。

**类型构造器**也就是一个接受类型参数的类型。

实例:

```scala
trait Function1[-T1, +R] extends AnyRef { self => }
class TreeMap[A <: Comparable[A], B] { ... }
class List[A] { ... }

class F[M[_], X] { ... }
class S[K <: String] { ... }
```

###  元祖类型(Tuple Types)

基本格式: **(T1 ... Tn), n>=2**

说明: 等价写法Tuple[T1...Tn]

###  注解类型(Annotated Types)

基本格式: **SimpleType {Annotation}**

说明：带注解类型实际就是在类型后面添加必要的注解以告诉编译器你要的处理

```scala
class Foo
class Bar extends Foo

import scala.reflect.runtime.{ universe => ju }

def meth[A: ju.TypeTag](xs: List[A]) = xs match {
/* 
    我们知道像List这种带参数类型的类型，在运行时类型会被擦除，
    所以一般无法直接进行类型匹配。另外在编译的时候，也会有警告。所以为了抑制编译器的警告，
    给类型参数加上@unchecked，也就是上面所说的带注解的类型。
*/
case strList: List[String @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[String] => "list of Strings"
case barList: List[Bar @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[Bar] => "list of Bar"
}
```

###  复合类型(Compound Types)

基本格式: **T1 with ... with Tn { Refinement }**


上面定义了一个复合类型，它<b style="color:red">拥有类型T1..Tn以及Refinement中的成员</b>。Refinement实际就是一些定义和类型声明，比如说方法什么的。

```scala
trait Type1 {
  println(" Type1 被初始化")
}
trait Type2 {
  println(" Type2 被初始化")
  def foo(s: String) = println(s)
}

class SingleType[A] { self: Type1 with Type2 {
  def foo(s: String)
} =>

}

// 定义获取类型的方法
import scala.reflect.runtime.universe.{ TypeTag, typeTag }
def getTypeTag[T: TypeTag](o: T) = typeTag[T].tpe

val singleType = new SingleType with Type1 with Type2 {}

getTypeTag(singleType)

// singleType类型
SingleType with Type1 with Type2{ def bar(s: String): Unit }
```

下面的定义也是可以的。

```scala
class SingleType { self: AnyRef { def structual(s: String) } }
class SingleType { self: { def structual(s: String) } }
class SingleType { self: type1 with type2 }
class SingleType { self: type1 with type2 {} }
```

最后用一个实例总结:

```scala
class Bird(val name: String) extends Object {
  def fly(height: Int) = println("鸟的飞行高度: " + height)
}

class Plane(val code: String) extends Object {
  def fly(height: Int) = println("代号:  " +
     code + "的飞机的飞行高度: " + height)
}

object HeightTest extends App {
  /*
    通过定义复合类型的参数r， takeOff的第二个参数
    可以接受任何定义了code属性和fly方法的对象
  */
  def takeOff(
    location: String,
    r: { val code: String; def fly(height: Int) }
  ) = {
    println(" 起飞点 " + location)
    r.fly(300)
  }

  // 因为Bird中没有code属性，所以在此定义
  val bird = new Bird("wudi") {
    val code = "flappy bird"
  }
  val plane = new Plane("Boeing")
  takeOff("terminal1", plane)
  takeOff("tree", bird)
}
```

###  中置类型(Infix Types)

基本格式: **T1 op T2**

说明: 上面的表达式中中置操作符会在T1,T2上面调用，它等价于op(T1, T2)。

所有的中置操作符都是有优先级的。除了<b style="color:red">:</b>是右连接，其余的都是左连接。

```scala
// 关于:, 最典型的当然是scala.collection.immutable.List，运算是从右往左的
val newList = 1 :: 2 :: Nil
```

同样一个实例总结:

```scala
abstract class Converter[T, R] extends (T => R) {
  def apply(v1: T): R
}

class StringConverter[T] extends Converter[T, String] {
  def apply(v1: T): String = v1.toString
}

object InfixTest extends App {
  type ==>[T, R] = Converter[T, R]

  val infixType: ==>[Int, String] = 
      new ==>[Int, String] {
       def apply(v1: Int) = v1.toString
      }

  val infixTypeComplex: Int ==> String = 
      new ==>[Int, String] {
      def apply(v1: Int) = v1.toString
      }
}
```

###  函数类型(Function Types)

基本格式: **FunctionArgs => Type**

\> (T1, ..., Tn) => U 表示的是一个函数接受n个参数，并且返回类型U

\> 函数类型都是右连接的，S => T => U相当于S => (T => U)

\> 函数类型等价于定义了<b style="color:red">apply方法并且接受相应类型参数的类</b>。

```scala
package scala

// 另外Scala函数的参数是逆变的，返回值是协变的
trait Functionn[-T1,..., -Tn, +R] {
    def apply(x1: T1,...,xn: Tn): R
    override def toString = "<function>" 
}
```
###  存在类型(Existential Types) - 2015-08-02 更新

对于存在类型的定义，参见《Scala In Depth》:

> Existential types are a means of constructing types where portions of the type signature are existential, where existential means that although some real type meets that portion of a type signature, we don’t care about the specific type. Existential types were introduced into Scala as a means to interoperate with Java’s generic types, such as Iterator<?> or Iterator<? extends Component>

存在类型用于构造类型的时候，有一部分已经是确定的类型(库中的各种已知的数据类型，自己定义的类型类或特质)。Scala引入它主要是为了更方便的调用Java中的泛型类, Iterator<?>或者是Iterator<? extends Component>

基本格式: `T forSome { ‘type’ TypeDcl | ‘val’ ValDcl }`，默认情况下`type T => type T >: scala.Nothing <: scala.AnyRef`

```scala
// 在实际使用中，我们一般是 _ 来表示
def foo(x: List[_]) = x 
// 完整形式
def foo(x: List[T forSome { type T }]) = x

// Scala compiler 不在意传入什么类型，只要该类型是Int或者是Int的超类即可
def foo(x: List[T forSome { type T >: Int}]) = x
foo(List("hello")) // 因为String与Int的共同父类是Any, 所以上例的T的类型是Any
```

另外forSome块中定义的类型在T中都是可以被引用的。

```scala
trait Outer {
  type AbsT
    
  def handle(proc: this.type => Unit) 
}

type Ref = x.AbsT forSome { val x: Outer }
```

##  参考

\> Scala Reference 3.2

\> [Scala中置操作符](http://jim-mcbeath.blogspot.com/2008/11/scala-type-infix-operators.html)

\> [类型操作符](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/typeoperators.scala)

\> [Scala方法操作不能接受forSome形式的存在类型](http://stackoverflow.com/questions/31937965/scala-method-type-parameter-can-not-accept-existential-type-in-forsome-form/31978204#31978204)







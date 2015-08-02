---
layout: post
category: scala
date: 2015-02-13 15:29:03 UTC
title: Scala基础之值类型(value types)
tags: [值类型，类型投影]
permalink: /scala/value-types/
key:
description: "本文研究了Scala值类型及基本的运用"
keywords: [值类型，类型投影，类型指示器，参数类型，元祖类型，带注解的类型，复合类型，中置类型，函数类型]
---

今天定义一个函数的时候，突然发现需要使用一个类型里面的成员，于是想到类型投影(type projection)。在查阅Scala Reference的时候，发现类型投影只是Scala值类型中的一种，于是决定将Scala值类型研究一下，基本上就是对于英文版的Scala Reference的值类型章节进行了翻译，并使用一些案例说明。

# 问题
Scala的值类型有哪几种，每一种如何去使用？

# 解决
首先我们需要明白的是，Scala中每一个值都是有类型的。

<1> **Singleton Types(单例类型)**

基本格式: p.type 

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

// 调用(上面的例子有点low，但是能说明问题)
(new Concrete0).set(3).get
```

<2> **Type Projection(类型投影)**

基本格式: T#X

说明: X必须是T的"类型成员"(我的理解是X必须是scala.AnyRef的子类，而不能是scala.AnyVal的子类)

<3> **Parameterized Types(参数化类型)**

基本格式: T[U1, U2, U3]

说明: 参数化的类型T[U1, U2, U3]包含类型指示器T，类型参数U1到Un，n>=1。T必须是一个<b style="color:red">类型的构造器</b>并且接受n个类型参数。

**类型构造器**的含义实际上就是必须是一个接受类型参数的类型，说的更直白一点就是必须要接受泛型参数。

实例:

```scala
trait Function1[-T1, +R] extends AnyRef { self => }
class TreeMap[A <: Comparable[A], B] { ... }
class List[A] { ... }

class F[M[_], X] { ... }
class S[K <: String] { ... }
```

<4> **Tuple Types(元祖类型)**

基本格式: (T1 ... Tn), n>=2

说明: 上面那种写法和这种写法是等价的Tuplen[T1...Tn]

<5> **Annotated Types(注解类型)**

基本格式: SimpleType {Annotation}

说明：带注解类型实际就是在类型后面添加必要的注解以告诉编译器你要的处理

```scala
class Foo
class Bar extends Foo

import scala.reflect.runtime.{ universe => ju }

def meth[A: ju.TypeTag](xs: List[A]) = xs match {
/* 
我们知道像List这种带参数类型的类型，
在运行时类型会被擦除，
所以一般无法直接进行类型匹配。
另外在编译的时候，也会有警告。所以为了抑制编译器的警告，
给类型参数加上@unchecked，也就是上面所说的带注解的类型。
*/

case strList: List[String @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[String] => "list of Strings"
case barList: List[Bar @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[Bar] => "list of Bar"
}
```

<6> **Compound Types(复合类型)**

基本格式：T1 with ... with Tn { Refinement }

说明: 下面两段是按照Scala Reference原文翻译的，可能有点不那么优雅，我会尽量用实例来解释其中所提到的观点的。

(1) 上面定义了一个复合类型，它<b style="color:red">拥有类型T1..Tn以及Refinement中成员的对象</b>。
Refinement(我也不知道中文怎么翻译，╮(╯▽╰)╭)，实际就是一些的定义和类型声明，比如说定义一个方法什么的。

如果声明或定义覆盖了<b style="color:red">T1 with ... with Tn</b>中的某个声明或定义，那么适用于一般的覆盖规则。如果没有的话，那么这种声明或定义被称为<b style="color:red">结构化</b>的。

(2) 在结构化的声明或定义的 Refinement部分，所有值的类型只能引用在Refinement中的类型参数或者是定义的抽象类型。

(3) 如果没有Refinement的话，那么上面的定义就相当于<b style="color:red">T1 with ... with Tn { }</b>。反过来也可以没有前面的类型T1...Tn，也就是AnyRef { R }。

下面一一解释上面的知识点。

定义:

```scala
trait type1 {
    println(" type1 被初始化")
}
trait type2 { 
    println(" type2 被初始化")
    def structual(s: String) = println(s)
}
class SingleType { 
    self: type1 with type2 { def structual(s: String) }
    => 
}
/* 
  要求SingleType的实例必须是 
  type1 with type2 { def structual(s: String) } 这样的类型，
  具体到这段代码就是SingleType's instance必须继承type1, type2
  并且还需要定义def structual(s: String)这样一个函数  
*/
}

// 定义获取类型的方法，使用到Scala反射(TODO: 以后会用专门的一篇博客来说明这个问题的)。
import scala.reflect.runtime.universe.{ TypeTag, typeTag }
def getTypeTag[T: TypeTag](o: T) = typeTag[T].tpe
```

调用:

```scala
// 由于type2中定义了structual方法，所以singleType相当于定义了structual，编译通过
val structuralSingleType = new SingleType with type1 with type2
getTypeTag(structuralSingleType)

// singleType类型
jacoffee.scalabasic.SingleType
    with jacoffee.scalabasic.Type1 
    with jacoffee.scalabasic.Type2
```

<b style="color:red">(1)</b>， 上面那种singleType就被称之为结构化的。如果定义中覆盖了T1..Tn中的方法

```scala
// 注意定义的时候初始化的方向 SingleType -> type1 -> type2
// 下面这种方式的话，那么SingleType就是非结构化的。
val unStructuralSingleType = new SingleType with type1 with type2 {
    override def structural(t: String) = println(t)
}
```


> A reference to a structurally defined member (method call or access to a value or variable) may generate binary code that is significantly slower than an equivalent code to a non-structural member

引用结构化定义的成员(方法调用，访问变量)会导致生成的二进制码比非结构化的效率低(<b style="color:red">目前的水平还无法解释</b>)

<b style="color:red">(2)</b>，如果修改SingleType的定义如下:

```scala
class SingleType[A] { self: type1 with type2 { 
    def structual(s: String): A } =>
}
```

> Parameter type in structural refinement may not refer to an abstract type defined outside that refinement。
  在结构化的Refinement中不能引用它之外的类型参数。
  

<b style="color:red">(3)</b>，下面的定义也是可以的。

```scala
class SingleType { self: AnyRef { def structual(s: String) } }
class SingleType { self: { def structual(s: String) } }
class SingleType { self: type1 with type2 }
class SingleType { self: type1 with type2 {} }
```

最后用一个实际的例子总结:

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

	val bird = new Bird("wudi") {
		val code = "flappy bird"
	}
	val plane = new Plane("Boeing")
	takeOff("terminal1", plane)
	takeOff("tree", bird)
}

```

<7> **Infix Types(中置语法)**

基本格式: T1 op T2

说明: 上面的表达式中中置操作符会在T1,T2上面调用，它等价于op[T1, T2]。
中置操作符(我们可以自定义，但是不能使用*，因为它已经用于后缀修饰符表示可以传递多个参数)。

所有的中置操作符都是有优先级的。除了<b style="color:red">:</b>是右连接，其余的都是左连接。

```scala
// Array.scala
def apply[T: ClassTag](xs: T*): Array[T] = { ... }

//数组初始化的时候调用的是上面的方法
// T* 表示传递多个T类型的参数
val newArr = Array(1,2,3) 

// :, 最典型的当然是scala.collection.immutable.List
//的构造啦，运算是从右往左的
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

	val infixTypeComplex: Int => String = 
	    new ==>[Int, String] {
		  def apply(v1: Int) = v1.toString
	    }
}
```

<8> **Function Types(函数类型)**

基本格式: FunctionArgs => Type

说明: 这个相对来说就比较好理解了，不过有几点需要额外说明。

& (T1, ..., Tn) => U 表示的是一个函数接受n个参数，并且返回类型U

& 函数类型都是右连接的，S => T => U相当于S => (T => U)

& 函数类型等价于定义了<b style="color:red">apply方法并且接受相应类型参数的类</b>。

```scala
package scala

// 另外Scala函数的参数是逆变的，返回值是协变的(TODO: 以后会用专门的一篇博客来说明这个问题的)。
trait Functionn[-T1,..., -Tn, +R] {
    def apply(x1: T1,...,xn: Tn): R
    override def toString = "<function>" 
}
```

### 2015-08-02 更新
<9> **Existential Types**

对于什么是Existential Types，在Scala In Depth这本书中有段解释还是比较易懂的。

> Existential types are a means of constructing types where portions of the type signature are existential, where existential means that although some real type meets that portion of a type signature, we don’t care about the specific type. Existential types were introduced into Scala as a means to interoperate with Java’s generic types

Existential Types主要是用于类型构造的，只不过定义的时候只留下了部分的定义，因为Scala Compiler不在意具体的类型，只要求到时候传递的类型满足部分的定义即可。

基本格式: T forSome { ‘type’ TypeDcl | ‘val’ ValDcl }

```scala
// 在实际使用中，我们一般是 _ 来表示
def foo(x: List[_]) = x
// 完整形式
def foo(x: List[T forSome { type T }]) = x
```

从完整版的定义中，我们可以看出 T实际上是有可以Upper Bound 和 Lower Bound的，默认情况下 type T => type T >: scala.Nothing <: scala.AnyRef

```scala
// Scala compiler不在意传入什么类型，只要该类型是Int或者是Int的超类即可
def foo(x: List[T forSome { type T >: Int}]) = x

foo(List("hello")) // 因为String与Int的共同父类是Any, 所以上例的T的类型是Any
```

另外forSome block中定义的类型在T中都是可以被引用的。

```scala
trait Outer {
    type AbsT

    def handle(proc: this.type => Unit)
}

type Ref = x.AbsT forSome { val x: Outer }
```



# 结语

虽然，讲的类型比较杂但是个人感觉<b style="color:red">中置类型</b>和<b style="color:red">复合类型</b>是需要特别理解的。前者我们可以使用它定义自己的一些操作符，使程序更加的优雅和达意，就像 ==>之于Converter。

复合类型就更不用说了，在实际的项目中它是无处不在的，理解它的含义有助于我们解读程序中一些类型的意义因为大多数的时候，我们可能会依赖编译器的类型推断。



# 参考
<1> Scala Reference 3.2

<2> [scala-type-infix-operators](http://jim-mcbeath.blogspot.com/2008/11/scala-type-infix-operators.html)

<3> [typeoperators.scala](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/typeoperators.scala)

<4> [Existential Type ](http://stackoverflow.com/questions/292274/what-is-an-existential-type)





---
layout: post
category: scala
date: 2015-04-11 09:51:22 UTC
title: Scala基础之注解(annotation)
tags: []
permalink: /scala/annotation/
key: 226d7bec35395bb71452e8b95765c635
description: "本文罗列了Scala中常见的注解以及基本的应用"
keywords: [注解，序列化，瞬时的状态]
---

在学习Scala的过程中，总会碰到一些注解:

```scala
// Predef.scala
@inline def implicitly[T](implicit e: T) = e

@deprecated("Use `sys.error(message)` instead", "2.9.0")
  def error(message: String): Nothing = sys.error(message)
  
// Spark RDD.scala
abstract class RDD[T: ClassTag](
    @transient private var sc: SparkContext,
    ....)
```

当然上面只是列举了一部分，本文只介绍比较常见的注解以及定义的方法。

# 解决

<1> 注解的作用域

一般来说，注解可以作用于vals, vars, defs, classes, objects, traits, and types甚至是表达式的后面。

```scala
import scala.reflect.runtime.{ universe => ju }

def meth[A: ju.TypeTag](xs: List[A]) = xs match {
 case strList: List[String @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[String] => "list of Strings"
 case barList: List[Bar @ unchecked] 
    if ju.typeTag[A].tpe =:= ju.typeOf[Bar] => "list of Bar"
}
```

我们知道List[T]在运行时会被类型擦除，相当于变成List。@unchecked 告诉compiler不要去检查这个，否则就会报下面的warning。

```scala
non-variable type argument String in type pattern List[String] is unchecked 
since it is eliminated by erasure
```

另外, 注解实际上也是普通的类只不过Compiler对其进行特殊的支持，所以我们才能那样书写。比如说，我们常见的序列号的注解。

```scala
class SerialVersionUID(uid: Long) extends scala.annotation.StaticAnnotation

@SerialVersionUID(13567156)
class B {}

@deprecated("use newShinyMethod() instead", "since 2.3")
def bigMistake() =  //...
bigMistake

scalac -deprecation Deprecation2.scala

warning: method bigMistake in class B is deprecated: use newShinyMethod() instead
  println(bigMistake)
          ^
one warning found
```

<2> 几种常见的注解

最常见应该是上面已经提到的Deprecated，另外几种常见的如下:

(1) @volatile

实际上这个注解或是关键字，大多用于被并发访问的共享变量。虽然Scala对于并发的处理就是尽量减少共享变量，当我们不得不这样的时候就可以使用volatile， 它告诉JVM，当某个值在某个地方被更新的时候，要确保下次另一个线程访问的时候是最新的值(当然实际情况可能没有这么简单)。
volatile涉及的并发知识点还是很多的，但这并不是本文的重点。

```scala
class Person(@volatile var name: String) {
  def set(changedName: String) {
    name = changedName
  }
}
```

(2) @tailrec 

这个注解是与[尾递归优化](/scala/tail-recurison/)有关的，在Scala中如果写了一个需要递归的方法，这样我们就能够在方法的签名处添加@tailrec告诉Scala Compiler进行尾递归优化(有兴趣可以去Programming In Scala第8章看看)。

```scala
// 阶乘
def factorial(n: Int) = {
	@tailrec def go(n: Int, acc: Int): Int = {
		if (n <=0) acc
		else go(n-1, acc * n) 
		// 尾递归，顾名思义方法的调用也必须出现在返回值的位置
	}
	go(n, 1)
}
```
(3) @Unchecked

上面已经提到了，一般是在模式匹配的时候用到的，告诉Compiler有些地方不用"检查"了。如前所述，List[String @ unchecked]。

(4) @transient

这个注解一般用于序列化的时候标识某个字段不想要被序列化，这样即使它所在的对象被序列化，它也不会。

```scala
import java.io.{ FileOutputStream, FileInputStream }
import java.io.{ ObjectOutputStream, ObjectInputStream }

class Hippie(val name: String, @transient val age: Int) extends Serializable

object Serialization {

	val fos = new FileOutputStream("hippie.txt")
	val oos = new ObjectOutputStream(fos)

	val p1 = new Hippie("zml", 34)
	oos.writeObject(p1)
	oos.close()
}

object Deserialization extends App {

	val fis = new FileInputStream("hippie.txt") 
	val ois = new ObjectInputStream(fis)

	val hippy = ois.readObject.asInstanceOf[Hippie]
	println(hippy.name)
	println(hippy.age)
	ois.close()
}

运行之后的结果
zml
0
```

由于age被标记为@transient，在反序列化的时候，就获取不到原始值了所以被赋值为默认值。

(5) @inline

这个注解，在Scala.Predef中见到过一次。

> An annotation on methods that requests that the compiler should
try especially hard to inline the annotated method.

文档中的解释跟没有一样，在StackOverflow上找到几个问题，其中参考<5>中的有一段解释，个人觉得比较能说明作用。

>  Instead of a function call resulting in parameters being placed on the stack and an invoke operation occurring, 
the definition of the function is copied at compile time to where the invocation was made, 
saving the invocation overhead at runtime.

大致的意思就是@inline能够避免方法的参数被放到栈上，以及"显示的调用"，因为编译器在编译的时候会将整个方法copy到它被调用的地方。


# 参考

<1> [annotation](https://www.artima.com/pins1ed/annotations.html)

<2> [volatile](https://twitter.github.io/scala_school/zh_cn/concurrency.html#danger)

<3> [should-my-scala-actors-properties-be-marked-volatile](http://stackoverflow.com/questions/1031167/should-my-scala-actors-properties-be-marked-volatile)

<4> [concurrency](https://twitter.github.io/scala_school/zh_cn/concurrency.html#danger)

<5> [when-should-i-and-should-i-not-use-scalas-inline-annotation](http://stackoverflow.com/questions/4593710/when-should-i-and-should-i-not-use-scalas-inline-annotation)

<6> [does-the-inline-annotation-in-scala-really-help-performance](http://stackoverflow.com/questions/2709095/does-the-inline-annotation-in-scala-really-help-performance)



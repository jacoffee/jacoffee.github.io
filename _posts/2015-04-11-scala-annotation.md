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

一般来说，注解可以作用于vals, vars, defs, classes, objects, traits 和 types甚至是表达式的后面。

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

另外, 注解实际上也是普通的类只不过编译器对其进行特殊的支持，所以我们才能那样书写。比如说，我们常见的序列号的注解。

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

###  @volatile

实际上这个注解或是关键字，大多用于被并发访问的共享变量。在JVM内存模型中happens-before规则有一条就是volatile变量法则(有兴趣可以阅读Java并发编程实践 第16章Java内存模型)，对于volatile变量，同一变量的写操作总是先于读操作。

```scala
class Person(@volatile var name: String) {
  def set(changedName: String) {
    name = changedName
  }
}
```

###  @tailrec

这个注解是与[尾递归优化](/scala/tail-recurison/)有关的。

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

###  @Unchecked

一般是在模式匹配的时候用到的，告诉编译器有些地方不用"检查"了。如前所述，List[String @ unchecked]。

###  @transient

这个注解一般用于序列化的时候，标识某个字段不用被序列化。

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

###  @inline

这个注解，在Scala.Predef中见到过一次。官方文档中的解释跟没说一样，
倒是[StackOverflow上一个的答案](http://stackoverflow.com/questions/2709095/does-the-inline-annotation-in-scala-really-help-performance)，个人觉得比较能说明作用。

>  Instead of a function call resulting in parameters being placed on the stack and an invoke operation occurring, the definition of the function is copied at compile time to where the invocation was made, 
saving the invocation overhead at runtime.

大致的意思就是@inline能够避免方法的参数被放到栈上，以及"显示的调用"。因为编译器在编译的时候会将整个方法复制到它被调用的地方。


## 参考

\> [注解](https://www.artima.com/pins1ed/annotations.html)

\> [volatile](https://twitter.github.io/scala_school/zh_cn/concurrency.html#danger)

\> [什么时候使用inline注解](http://stackoverflow.com/questions/4593710/when-should-i-and-should-i-not-use-scalas-inline-annotation)


---
layout: post
category: scala
date: 2015-02-06 14:59:49 UTC
title: Scala基础之静态代理(static-forwarder)
tags: [静态代理, 单例对象]
permalink: /scala/static-forwarder/
key: ae1c4aa3fc3d7ec318c814a4093d0e29
description: "本文研究了静态代理的概念，产生方式以及Scala编译器对于静态代理的一些处理机制"
keywords: [静态代理, 单例对象]
---

在Scala类中我们并没有像Java那样显示的声明main方法，而是通过继承Trait App来实现的。今天在写下面这段代码的时候突然报了如下错误:

```scala
  trait TryCatch {}
  object TryCatch extends App {}
  // Reason: companion is a trait, which means no 
  // static forwarder can be generated.
```

提示说的很清楚TryCatch的伴生"类"是一个特质，这样的话静态代理就无法生成了。

# 问题
那么Scala中的静态代理是什么呢？为什么Trait中无法生成静态代理呢？

# 解决
Static forwarder provides easy way for java to invoke scala object's method(静态代理让Java调用Scala的方法更方便了)。

首先，我们采用一个简单的例子来查看什么是static forwarder。

eg:

```scala
object Forwarder {
	def foo = 2
}
```
翻译之后会生成两个类Forwarder.class, Forwarder$.class。

<b style="color: red">反编译出来Forwarder.class</b>

```java
public final class Forwarder
{
  public static int foo()
  {
    return Forwarder..MODULE$.foo();
  }
}
```
<b style="color:red">Scala编译器生成的**合成类**(synthetic class) Forwarder$.class</b>

```Java
public final class Forwarder$
{
    public static final  MODULE$;
    
    static
    {
     new ();
    }
    
    public int foo()
    {
     return 2; 
    } 
    private Forwarder$() { MODULE$ = this; }
}
```
我们可以看出，Forwarder中foo方法实际上引用的是Forwarder中静态实例MODULE$(通过static块，静态初始化生成的)的foo方法。

> <b style="color:red">这实际上Scala编译器的一个规则，如果你定义了一个单例对象object A但是并没有一个同名的class A, 那么编译器就会为你生成一个同名的class A并且在里面定义所有object A中方法的静态代理。</b>
 如果你已经有了一个同名的class A，这时编译器便不会再生成静态代理。此时Java代码可以通过MODULE$属性来访问单例对象。

---

**上面两条规则有一个特别需要注意的地方，什么叫已经有了一个同名的class A**。
经过验证发现这里的含义是如果你先在Desktop文件下通过

```scala
  class Test {}
```
编译出Test.class，然后通过

```scala
  object Test {
    def foo = 2
  }
```
编译出Test$.class, **由于在相同的文件夹下已经有一个Test.class**, 所以此时Test.class 并没有像之前那样出现静态代理method。

但是如果你在同一个文件中写下如下代码：

```scala
  object Test {
  	def foo = 2
  }
  class Test {
  	def myMethod = println("test")
  }
```
**此时在Test.class中会生成相应的静态代理类**, 通过打印jvm的log也可以看出

```
$ scalac -Ylog:jvm Full.scala 
[log jvm] No mirror class for module with linked class: Full
[log jvm] missingHook(package <root>, android): <none>
[log jvm] Adding static forwarders from 'class Full' to 
implementations in 'object Full'
```
--- 

我们通过Java代码来测试一下静态代理

```Java
public class MainTest {
	public static void main(String[] args) {
		int r = Forwarder$.MODULE$.foo(); 
		int r1 = Forwarder.foo();
      // 如果使用第一种方法调用foo，我们就需要了解Forwarder$.class的bytecode,
      // 这样不友好。所以Scala编译器在Forwarder.class中提供了
      // 单例对象方法的"副本"即foo以实现静态代理机制。
		System.out.println(r == r1); // true
	}
}
```

为了进一步理解这个概念，引用来自Programming In Scala中的一句话。
> Java has no exact equivalent to a singleton object, but it does have static methods.
  The Scala translation of singleton objects uses a combination of static and instance methods. For every Scala singleton object, the compiler will <b style="color:red">create a Java class for the object with a dollar sign added to the end.</b>
  
> For a singleton object named App, the compiler produces a Java class named App$.
  This class has all the methods and fields of the Scala singleton object.
The Java class also has a single static field named MODULE$ to hold the one instance of the class that is created at run time.

上面那段话很好的解释了Forwarder$出现的原因并且在Forwarder$中有一个静态的实例MODULE$以及Scala单例对象Forwarder的所有实例方法。


#结语
<b style="color:red">最后来回答开篇的问题</b>：

<1> 静态代理实际上就是Scala编译器生成的静态方法，以便能更方便Java的调用。

<2> 因为trait TryCatch(编译之后实际上变成了public abstract interface TryCatch)中无法生成
单例对象的静态代理，interface中显然不能出现静态方法(它是类层面的概念)。

#参考
<1> [scala里的静态代理(static-forwarders)](http://hongjiang.info/scala-static-forwarders/)

<2> [Singletons as Synthetic classes in Scala?](http://stackoverflow.com/questions/5721046/singletons-as-synthetic-classes-in-scala)
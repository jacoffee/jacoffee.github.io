---
layout: post
category: scala
date: 2015-02-06 14:59:49 UTC
title: Scala基础之静态代理(static forwarder)
tags: [静态代理, 单例对象]
permalink: /scala/static-forwarder/
key: ae1c4aa3fc3d7ec318c814a4093d0e29
description: "本文研究了静态代理的概念，产生方式以及Scala编译器对于静态代理的一些处理机制"
keywords: [静态代理, 单例对象]
---

在Scala中，想要通过主类运行程序，一般我们并没有声明main方法，而是通过**单例对象**继承App来实现的。如果单例对象和同名特质同时出现在同一作用域则会报如下警告: Test的伴生"类"是一个特质，无法生成静态代理，所以导致单例对象Test无法成为一个可运行的程序。

```scala
/*
Warning:(140, 8) Test has a main method with parameter type Array[String], but Test will not be a runnable program.
  Reason: companion is a trait, which means no static forwarder can be generated.
object Test extends App {}
       ^
*/
trait Test {}
object Test extends App {}
```

在解答这个问题之前，先来了解一下什么是**静态代理**，以单例对象为例来展开。

```scala 
object Test { def foo = 2 }
```

编译之后会生成两个类`Test.class`, `Test$.class`(合成类)。

```bash
allen:Desktop allen$ javap Test
Compiled from "Test.scala"
public final class Test {
  public static int foo();
}
allen:Desktop allen$ javap Test$
Compiled from "Test.scala"
public final class Test$ {
  public static final Test$ MODULE$;
  public static {};
  public int foo();
}
```

在**Programming In Scala**一书中对于单例对象有这样一段解释：

> For a singleton object named App, the compiler produces a Java class named App$.This class has all the methods and fields of the Scala singleton object.
The Java class also has a single static field named MODULE$ to hold the one instance of the class that is created at run time.

对于单例对象App，编译器会生成一个Java类App$，它包含单例对象中所有的方法和属性，同时还有一个静态的属性`MODULE$`引用了一个运行时生成的App$实例。具体到上面的代码`Test$`为编译器生成的Java类包含了单例对象中的`foo`方法和静态属性`MODULE$`。

如果只有这个类的话，那么我们在Java中我们就需要这样调用`Test$.MODULE$.foo()`, 它要求我们了解`Test$`内部结构并且也不友好。 所以`Test`出现了, 这样在Java中可以
通过`Test.foo()`来调用(Test$中的方法)，**Test.class中的静态方法实际上就是前面提到的静态代理**。

```java
public class MainTest {
  public static void main(String[] args) {
    int r = Test$.MODULE$.foo(); 
    int r1 = Test.foo();
    System.out.println(r == r1); // true
  }
}
```

当同一作用域内同时出现了伴生类A和伴生对象A时，编译时除了在`A.class`中生成静态代理之外，还会添加伴生类本来的方法。

```scala
object Test { def foo = 2 }
class Test {}

$ scalac -Ylog:jvm Test.scala 
[log jvm] missingHook(package <root>, android): <none>
[log jvm] Adding static forwarders from 'class Test' to implementations in 'object Test'
[log jvm] No mirror class for module with linked class: Test
```

回到开始的案例，当同一作用域内同时出现了特质和单例对象时，编译器会尝试在`Test.class`中添加静态代理，但此时`trait Test`**编译之后实际上是Java中的接口**，而接口是不能定义静态方法的。
所以`Test.class`中并没有对应的`main`方法，所以不能成为一个可运行的程序。

## 参考

\> [Scala里的静态代理(static-forwarders)](http://hongjiang.info/scala-static-forwarders/)

\> [Singletons as Synthetic classes in Scala?](http://stackoverflow.com/questions/5721046/singletons-as-synthetic-classes-in-scala)
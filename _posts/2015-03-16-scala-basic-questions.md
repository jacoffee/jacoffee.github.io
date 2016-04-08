---
layout: post
category: scala
date: 2015-03-16 09:51:22 UTC
title: Scala基础问题汇总
tags: [修饰符]
permalink: /scala/questions/
key: b801553a6ab218a20268697af6676ce5
description: "本文收集一些Scala基本问题的解答"
keywords: [修饰符]
---

### <1> 类和对象

(1) **作为属性修饰符，private[this]和private的区别**

```scala
class Foo {
    private var i = 0 
    private[this] var j = 0 
    def iThing = i * i 
    def jThing = j * j 
}
```

访问权限层面，前者仅限当前类中访问，后者稍微宽松一点能在类伴生对象。两者都不能在新生成的类实例中访问。

```scala
object Foo {
  (new Foo).i // Okay
  (new Foo).j // Not Okay
}
object Bar {
  (new Foo).i // Not Okay
  (new Foo).j // Not Okay
}
```

字节码层面，两者的调用机制不一样，前者是在方法访问的时候直接调用的属性，而后者调用的是编译器生成的属性对应的getter(如下所示)。个人猜想，因为private[this]属性是**当前实例才能访问的，不需要向外暴露什么**，所以没有getter，直接访问属性。但private属性在伴生对象中还可以访问，所以需要提供getter。

```scala
// 部分反编译的代码
public int iThing() { return i() * i(); } 
public int jThing() { return this.j * this.j; }
```


##参考

\> [private[this] vs private的字节码区别](https://gist.github.com/twolfe18/5767545)


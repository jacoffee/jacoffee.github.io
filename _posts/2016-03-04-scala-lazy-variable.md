---
layout: post
category: scala
date: 2016-03-04 11:29:03 UTC
title: Scala基础之懒加载变量
tags: [延迟加载，双重检验锁，语法糖]
permalink: /scala/lazy-variable/
key: cbf39c821f9351c5ac7d70065c9d793d
description: "本文研究了Scala中懒加载变量背后的实现"
keywords: [延迟加载，双重检验锁，语法糖]
---

在Scala中lazy关键字还是比较常用的，相较于方法或者函数，它是在被使用的时候第一次初始化，之后会继续使用之前计算的值而不会重新计算。
不过从本质上来讲，它修饰的是一个变量，所以需要确保该变量在线程A中被初始化之后，另一个线程B访问的时候不再初始化。而lazy关键字也确实保证了这点，实际上就是[双重检验锁](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)。


```scala
class LazyVar {
	lazy val bar = math.pow(5, 3)
	lazy val foo = "allen"
}
```

反编译的代码:

```java
public class LazyVar {
	private double bar;
	private String foo;
	// byte的默认值0
	private volatile byte bitmap$0;
    
	private double bar$lzycompute() {
		synchronized (this) {
			if ((byte) (this.bitmap$0 & 0x1) == 0) {
				this.bar = package..MODULE$.pow(5.0D, 3.0D);
				this.bitmap$0 = ((byte) (this.bitmap$0 | 0x1));
			} 
			return this.bar;
		}
	}

	private String foo$lzycompute() {
		synchronized (this) {
			if ((byte) (this.bitmap$0 & 0x2) == 0) { 
				this.foo = "allen";
				this.bitmap$0 = ((byte) (this.bitmap$0 | 0x2));
			}
			return this.foo;
		}
	}
	
	public double bar() {
		return (byte) (this.bitmap$0 & 0x1) == 0 ? bar$lzycompute() : this.bar;
	}

	public String foo() {
		return (byte) (this.bitmap$0 & 0x2) == 0 ? foo$lzycompute() : this.foo;
	}

}
```

关于上面代码实现的几点解释:

(1) **双重检验锁**

> In Scala, it use double-checked lock -- when in the synchronized block, it will check again whether the variable been protected has been initialized.
   
在`bar$lzycompute()`中，除了使用`synchronized`关键字，在初始化bar之前还需要再次检验。

(2) **lazy变量的个数与bitmap的实现策略**

当lazy变量只有1个，直接用布尔值来判断;
当lazy变量的个数小于等于8个，可以通过bitmap的对应位数上是1还是0来判断(1byte=8bit);
当lazy变量的个数超过8个的时候，一个字节的8位便不足以进行判断。需要将bitmap扩展为整形(Int)，判断也会更复杂。
    
(3) **判断lazy变量是否已经初始化的标识**

按位与 (&) -- 相同位的两个数，**如果都为1，则结果为1；其它情况均为0**; 

按位或 (|) -- 相同位的两个数，**如果其中一个为1，则结果为1；其它情况均为0**。

对于bar变量的初始化判断标准就是: 

如果bitmap的末位为0，则说明该变量没有被初始化;和0x1的按位与只有在末位为1的时候才会返回1。
**(this.bitmap$0 & 0x1) == 0**说明bitmap末位是0，所以进行初始化。

然后将bitmap的末位赋值为**(this.bitmap$0 | 0x1)**通过按位与将末位变成1，标识该变量已经被初始化。 
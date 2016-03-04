---
layout: post
category: scala
date: 2015-03-04 15:29:03 UTC
title: Scala基础之懒加载变量
tags: [延迟加载，双重检验锁]
permalink: /scala/lazy-variable/
key:
description: "本文研究了Scala中懒加载变量背后的实现"
keywords: [延迟加载，双重检验锁]
---

# 问题
在Scala中lazy关键字还是使用的比较多的, lazy变量是在调用的时候初始化一次的。 当多个线程同时访问该变量的时候，这就不可避免的要涉及到**变量共享**， 我们需要确保当该变量在一个线程中初始化之后，另一个线程访问的时候不再初始化。那么Scala到底是如何实现lazy变量的？


# 解决

下面是一个基本的测试用例:

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

### 双重检验锁

> In Scala, it use double-checked lock -- when in the synchronized block, it will check again whether the variable been protected has been initialized.
   
在调用foo方法的时候，检验bar是否被初始化；如果没有，通过synchronized获取锁。但可能在获取锁的瞬间在另一个线程访问的时候被初始化了，所以还要对标识变量再次检验以确保不会重复初始化。

### lazy变量的个数与bitmap的实现策略

- 当lazy变量只有1个，直接用布尔值来判断;
- 当lazy变量的个数小于等于8个，可以通过bitmap的对应位数上是1还是0来判断;
- 当lazy变量的个数超过8个的时候，一个字节的8个位便不足以再进行判断，需要将bitmap扩展为整形格式，判断也会更复杂，这里不再赘述。
    
### 判断lazy变量是否已经初始化的标识

> 按位与 (&) -- 相同位的两个数，**如果都为1，则结果为1；其它情况均为0**
  按位或 (|) -- 相同位的两个数，**如果其中一个为1，则结果为1；其它情况均为0** 
    
对于bar变量的初始化判断标准就是: 如果bitmap的末位为0，则说明该变量没有被初始化;和0x1的按位与只有在末位为1的时候才会返回1。
```(this.bitmap$0 & 0x1) == 0```说明bitmap末位是0，所以进行初始化，然后将bitmap的末位赋值为```(this.bitmap$0 | 0x1)```通过按位与将末位变成1。 

```bash
  # 与
  0000 0001
  0000 0000
  --------
  0000 0000
  
  # 或
  0000 0001
  0000 0000
  --------
  0000 0001
```  
  
对于foo变量的初始化判断标准就是: 如果bitmap的倒数第二位为0，则说明该变量没有被初始化;
具体流程如下:
 
```bash
  # 与
  0000 0010
  0000 0000
  --------
  0000 0000
  
  # 或
  0000 0010
  0000 0000
  --------
  0000 0010
```  

##参考
\> [Double-checked lock](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)
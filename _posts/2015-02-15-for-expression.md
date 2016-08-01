---
layout: post
category: scala
date: 2015-02-15 15:29:03 UTC
title: Scala基础之For表达式
tags: [For表达式，流程控制，语法糖]
permalink: /scala/for-expression/
key: 708d91df9e7ec1dc30e4f6af8b57183f
description: "本文研究了Scala For表达式以及背后涉及的语法糖"
keywords: [For表达式，语法糖，高阶函数]
---

在实际的开发中，当我们需要多重map和flatMap嵌套时，不妨考虑For表达式。
它基本格式为: `for ( seq | option | 实现了map或者是flatMap的数据结构) yield expr`

```scala
case class People(name: String, isMale: Boolean, children: People*)

object SugarFor extends App {
	val lara = People("Lara", false)
	val bob = People("Bob", true)
	val julie = People("Julie", false, lara, bob)
	val people = List(lara, bob, julie)

	val certainName = for {
		p <- people // generator
		n = p.name // definition
		if (n startsWith "Bob")  // filter also called guard
	} yield n
}
```

For表达式都会被编译器转换成一些高阶函数的调用(map, flatMap, withFilter)。如果For表达式中没有<b style="color:red">yield</b>的话，那么它会被转换成withFilter和foreach组合调用。上面代码反编译出来的结果：

```java
/* 
  上面的三种类型分别对应了map, withFilter, map
  不过需要注意的是上面的filter实际上在defnition前面执行的，
  如果去掉yield的话，最后一步就会变成foreach。
*/

 this.$outer.certainName_$eq(
    (List)(
    (TraversableLike)this.$outer.people().
        map(
          new SugarFor..anonfun.4(),          
          List..MODULE$.canBuildFrom()
        )
    ).withFilter(
        new SugarFor..anonfun.5()
    ).map(
        new SugarFor..anonfun.6(),        
        List..MODULE$.canBuildFrom()
    )
);
```


当同时出现多个seq以及过滤表达式的时候。对于这种情况，有一个简单的通用规则。当Generator表达式数目超过1时，每多一个就会多一个flatMap。总的flatMap个数总是比Generator数目少1。

```scala
object ForTest extends App {
    val list = List(1,2,3)
    // 两个generator
    val r1 = for {
    	l1 <- list;
    	l2 <- list if l2 > 2
    } yield {
    	(l1, l2)
    }
    
    val r2 = list.flatMap { x1 =>
    	list.withFilter(_ > 2).map { x2 =>
    		(x1, x2)
    	}
    }
    assert(r1 == r2) // true
}
```

```scala
object ForTest extends App {
	val list = List(1,2,3)
	// 三个generator
	val r1 = for {
		l1 <- list;
		l2 <- list if l2 > 1
		l3 <- list if l3 > 2
	} yield {
		(l1, l2, l3)
	}

	val r2 = list.flatMap { x1 =>
		list.withFilter(_ > 1).flatMap { x2 =>
			list.withFilter(_ > 2).map { x3 =>
				(x1, x2, x3)
			}
		}
	}

	assert(r1 == r2) // true
}
```

如果Generator过程中出现复杂的模式。一般情况下，Generator左边的表达式一般是单一变量x或者是tuple。但如果左边是一个比较复杂的模式，那么编译器会转换成如下格式:

```scala
    for (pat <- expr1) yield expr2
    
    // 只有在通过模式匹配的情况下，才会继续执行
    expr1 withFilter {
        case pat => true
        case _ => false
    } map {
        case pat => expr2
    }
```

```scala
import net.liftweb.mongodb.JsonFormats
object ForTest extends App with JsonFormats {
	implicit val jValueformate = allFormats
	val pet1 = Pet("stella", 23)
	val pet2 = Pet("dodge", 12)
	
	val petJValue = (pet1 :: pet2 :: Nil).map(Extraction.decompose)

	val r1 = for {
		JObject(
            JField("name", JString(name)) :: 
            JField("age", JInt(age)) :: 
            Nil
        ) <- petJValue
	} yield {
		(name, age.intValue)
	}

	
	val r2 = petJValue.withFilter {
		case JObject(
            JField("name", JString(name)) :: 
            JField("age", JInt(age)) :: 
        Nil) => true
		case _ => false
	}.map {
		case JObject(
            JField("name", JString(name)) :: 
            JField("age", JInt(age)) :: 
        Nil) => (name, age.intValue)
	}

	assert(r1 == r2) // true
}
```

通过上面的例子，我们可以得出如下结论:

(1) For表达式可以被转化成map, flatMap, withFilter等高阶函数的调用，反过来<b style="color:red">map, flatMap, filter也能够通过for来实现(因为withFilter并不是简单的filter所以不能直接实现)</b>


```scala
class HighOrderFuncViaFor {
    def map[A, B](list: List[A], f: A => B): 
    List[B] = for { l  <- list } yield f(l)
    
    def flatMap[A, B](list: List[A], f: A => List[B]): 
    List[B] = for { l <- list;l1 <- f(l) } yield l1
    
    def filter[A](list: List[A], f: A => Boolean): 
    List[A] = for { l <- list if f(l) } yield l
}
```

(2) 只要定义了map, flatMap, withFilter等高阶函数的数据结构都可以进行for表达式(排除没有yield的情况)。

具体的规则如下，针对给定的数据类型F:

\> 如果F只定义了map操作，那么对应的For只允许单个的Generator操作，因为前面提到过两个及以上的Generator就会涉及到flatMap。

\> 如果同时定义了map和flatMap，那么就能进行多个Generator的操作。

\> 如果定义了withFilter方法，那么就会支持以if开头的过滤表达式。

```scala
abstract class ADT[A] {
	def map[B](f: A => B): ADT[B]

	def flatMap[B](f: A => ADT[B]): ADT[B]

	def foreach(f: A => Unit): Unit

	def withFilter(f: A => Boolean): ADT[A]
}
```
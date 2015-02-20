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

最开始学习Scala的时候，公司的人告诫我不要轻易使用For表达式，因为"有点像"Java中命令式的编程与函数式编程的基因有点格格不入(<b style="color:red">尽管Scala是个面对对象和函数式编程的混血儿，但是在日常开发中它的函数式编程特点还是占据了主导地位</b>)。

后来随着学习的深入，发现在有些场景。For表达式是比map, flatMap的嵌套是更加优雅的。实际上，背后也只是map, flatMap, withFilter等高阶函数的组合调用。

# 问题
Scala For的基本语法以及背后涉及的语法糖问题？

# 解决
<h3 style="text-indent: 25px;">基本格式: for ( seq ) yield expr</h3>

> seq is a sequence of generators, definitions, and filters, with semi- colons between successive elements.
seq就是一系列生成器，定义以及过滤的组合，并且以；分隔。

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
		if (n startsWith "Bob") 
		// filter also called guard
	} yield n
}
```

For表达式其实是一种<b style="color:red">语法糖</b>。规则如下:

一般来说，For表达式都会被编译器转换成一些高阶函数的调用(map, flatMap, withFilter)。如果For表达式中没有<b style="color:red">yield</b>的话，那么它会被转换成withFilter和foreach组合调用。前者不一定有，但是后者一定有。

上面的代码的class文件反编译出来的结果：

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

<h3 style="text-indent: 25px;">重要规则展开</h3>

<1> 同时出现多个Generator以及过滤表达式的

对于这种情况，有一个简单的通用规则。当Generator表达式数目超过1时，每多一个就会多一个flatMap。总的flatMap个数总是比Generator数目少1。

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

<2> 如果Generator过程中出现复杂的Pattern

一般情况下，Generator左边的表达式一般是单一变量x或者是tuple。但如果左边是一个比较复杂的Pattern，那么编译器会转换成如下格式:

```scala
    for (pat <- expr1) yield expr2
    
    // 这样就确保只map通过withFilter的item
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

<3> 如果For表达式中出现了defition语句

```scala
 for { x <- expr1; y = expr2; seq } yield { expr3 }
 
 // translate
 
 for { 
    (x, y) <- for(x <- expr1) yield { (x, expr2)}; seq  } yield { expr3 }
```

这种情况下，x可能被计算两遍，因为expr2中可能会引用x。所以这种写法不是很提倡。

```scala
// 注意下面代码的class反编译文件

object ForTest {
	val r1 = for {
		x <- 1 to 5
		y = x * 3
	} yield (x, y)
}
```

```java
// 很明显可以发现definition语句实际上对应了一次map。
// 第一次map仅仅只是生成了(x, expr2),   第二次map才是真正的获得最后结果的(x, y)

  this.r1 = 
      ((IndexedSeq)(
      (TraversableLike)RichInt..MODULE$
.to$extension0(Predef..MODULE$.intWrapper(1), 5).map(
new AbstractFunction1() {
  public final Tuple2<Object, Object> apply(int x) { 
    int y = x * 3;
    return new Tuple2.mcII.sp(x, y);
    }
}
, IndexedSeq..MODULE$.canBuildFrom()
)).map(new AbstractFunction1() { 
 public final Tuple2<Object, Object> apply(Tuple2<Object, Object> x$1) { 
          Tuple2 localTuple2 = x$1; 

          if (localTuple2 != null) {
            int x = localTuple2._1$mcI$sp();
            int y = localTuple2._2$mcI$sp();
            Tuple2.mcII.sp localsp = new Tuple2.mcII.sp(x, y);

            return localsp; 
          }
          
          throw new MatchError(localTuple2);
      }
    }
    , IndexedSeq..MODULE$.canBuildFrom()));
  }
```

<h3 style="text-indent: 25px;">通用的规则总结</h3>

通过上面的例子，我们可以得出如下结论:

<1> For表达式可以被转化成map, flatMap, withFilter等高阶函数的调用，反过来<b style="color:red">map, flatMap, filter也能够通过for来实现(因为withFilter并不是简单的filter所以，不能直接实现)</b>


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

<2> 只要定义了map, flatMap, withFilter等高阶函数的数据结构都可以进行for表达式(排除没有yield的情况)。

具体的规则如下，针对给定的数据类型F：

<1> 如果F只定义了map操作，那么对应的For只允许单个的Generator操作，因为前面提到过两个及以上的Generator就会涉及到flatMap。

<2> 如果同时定义了map和flatMap，那么就能进行多个Generator的操作。

<3> 如果定义了withFilter方法，那么就会支持以if开头的过滤表达式。

```scala
abstract class ADT[A] {
	def map[B](f: A => B): ADT[B]

	def flatMap[B](f: A => ADT[B]): ADT[B]

	def foreach(f: A => Unit): Unit

	def withFilter(f: A => Boolean): ADT[A]
}
```

不过上面的withFilter的实现实际上是很废的，因为每次调用都会返回一个新的ADT[A]，相当于产生了中间状态。所以在Option中的实现避免了上面的情况。

```scala
def withFilter(p: A => Boolean): WithFilter = new WithFilter(p)
```

WithFilter就像是"记住"了对于类型中元素的操作，等待map, flatMap的时候再进行操作，有点lazy的意思。


# 结语
For表达式说到底，其实不是什么新语法。只是map, flatMap, withFilter等高阶函数的语法糖。至于什么时候使用，完全取决于个人偏好和编程风格。

在实际的开发中，当多个Option和List混合在一起并且需要Guard的时候，那么For无疑是个不错的选择。这样可以避免map和flatMap的嵌套。


# 参考
<1> Programming In Scala Second Edition Chapter 23

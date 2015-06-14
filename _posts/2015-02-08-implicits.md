---
layout: post
category: scala
date: 2015-02-08 14:51:31 UTC
title: Scala基础之隐式转换(implicit conversion)
tags: [隐式转换, 隐式参数, 隐式方法, 隐式类]
permalink: /scala/implicits/
key: dde9e45d61a77e15ceb5bbf9cfa1e6ac
description: "本文研究了隐式转换，隐式参数, 视界的概念以及Scala编译器寻找隐式类型的顺序"
keywords: [隐式转换, 隐式方法, 隐式类, 视界, 类型参数]
---

最初学习Scala的时候对隐式转换这个概念印象特别深，因为它可以进行自动类型转换，方法调用还有增强类的功能，还有隐式参数（就是Scala编译器自动帮你传进去的参数），所以现在想借此机会将隐式转换以及隐式参数这个知识点进行系统的梳理，同时也会总结个人在学习中的一些心得。

# 问题
隐式转换，隐式参数是什么？隐式类型分为哪些？Scala编译器是按照什么顺序去寻找隐式类型的？

# 解决
隐式转换指的是Scala编译器会帮助完成指定的"转换"，从而减少重复代码并且在一定程度上扩展现有library的功能。隐式参数就是不需要你显示传递参数(Scala编译器会帮你完成当然你也可以显示的传递)。

<1> 减少重复代码

在Java的Swing中我们会给组件注册监听事件，通常会有如下代码:

```java
JButton button = new JButton("press me");
button.addActionListener(new ActionListener() {
	@Override
	public void actionPerformed(ActionEvent e) {
		System.out.println("Click");
	}
});
```

如果以Scala的角度考虑(函数式)， 我们可以看出ActionListener中代码是非常模式化的，就是生成ActionListerner的实例并且重写actionPerformed方法。

```scala
    button.addActionListener({
		(_: ActionEvent) => println("I am pressed")
	})
	// 但是这种写法会报编译类型错误的
	
	// 增加隐式方法为我们完成这次转换
    implicit def actionEventFuncToActionListener(f: ActionEvent => Unit) =
		new ActionListener {
			override def actionPerformed(e: ActionEvent): Unit = f(e)
		}
	// 那么当编译器去编译上面的代码，开始不通过然后去搜索隐式转换，编译通过
	// 特别注意隐式方法有点先定义后使用的感觉，
    // 就是actionEventFuncToActionListener一定要定义
	// 在button.addActionListener之前，当然这个之前不仅仅指的是代码的位置。
```
所以隐式转换很好的避免了，每次注册监听事件都要重复书写ActionListener的实现。

<2> 功能扩展

比如说我们想扩展某个jar中类的功能，就可以采用此方法。

```scala
// 给String添加新方法
class StringImprovement(s: String) {
	def increment = s.map(one => (one + 1).toChar)
}
implicit def stringIncrement(s: String) = new StringImprovement(s)


// "abc"默认的是没有increment方法，这次隐式转换就会去搜索哪个类有这个方法然后，将string转换成StringImprovement
// 还有一点需要注意的，Scala编译器对于隐式方法的寻找， 似乎有点类似于“变量的先定义后使用的原则"
// 如果上面的隐转方法在下面的调用之后被定义， Scala编译器就会寻找失败
"abc".increment


//还有一个更为常见的就是
Map(1 -> "one", 2 -> "two") 

// 下面的隐转使上面的书写更简洁和达意
package scala
object Predef {
    class ArrowAssoc[A](x: A) {
      def -> [B](y: B): Tuple2[A, B] = Tuple2(x, y)
    }
    implicit def any2ArrowAssoc[A](x: A): ArrowAssoc[A] =
      new ArrowAssoc(x)
    ...
}
```

### 隐式类型
<1> 隐式转换

这个上面已经提到了，就是如果class C的某个实例o调用了方法m，即o.m, 但实际上o并没有m方法。这时候Scala编译器就会去寻找能够把o转成支持m方法调用的一个类型。
    
<2> 隐式参数

编译器会自动帮你传递这个参数，如果找不到就会报错。简单来说就是someCall(a)在实际调用的时候会变成someCall(a)(b)。

举个例子，网站用户登录之后一般都会有问候语什么的但一般是固定的。我们可以用隐式参数来模拟。

eg:

```scala
class Welcome(val msg: String)
object Greeter {
    def greet(name: String)(implicit welcome: Welcome) = {
        s"${name}, ${welcome}" 
    }
}

// Explicit Invocation
Greeter.greet("allen")(new Welcome("欢迎回来"))

// Implicit Invocation
object GreetingSetting {
    implicit val welcome = new Welcome("欢迎回来")
} 

//引入隐式参数
import GreetingSetting._
Greeter.greet("allen")
```

使用隐式参数避免了一些重复的显示调用并且使代码更简洁。另外上面那个例子我们实际上需要输出的是一个字符串，但是传进去的是一个Type，实际上这样的隐式参数灵活性也是比较好的。而这种写法也在Programming In Scala中提到了

> A style rule for implicit parameters。As a style rule, it is best to use a <b style="color:red">custom named type in the types of implicit parameters.</b> For example, the types of welcome in the previous example was not String, but Welcome。

> Thus the style rule: use at least one role-determining name within the type of an implicit parameter(这句话告诉我们隐式参数的类型命名应该是达意的，让人一下就看出来它是干什么的).

Scala的CanBuildFrom就是一个不错的例子

```scala
trait CanBuildFrom[-From, -Elem, +To] {}

def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That = {}
```

<3> 视界(View Bound)
说实话，我感觉有些技术术语翻译成中文之后总感觉怪怪的，就像"视界"第一眼看到根本不知道是干什么的。
对于O <% T, 只要O可以被"当作"T，那么我就可以随意使用O。

eg:

```scala
// 从List[T]中寻找最大的T, Ordered是Scala Library的一个用于比较元素"大小"的特质
def maxList[T](elements: List[T])(implicit orderer: T => Ordered[T]): T = {
    elements match {
        case Nil => throw new NoSuchElementException("empty list!")
        case head :: Nil => head
        case head :: tail => {
            val maxRest = maxList(tail) // maxList(tail)(orderer) 
            if (head > maxRest) head // orderer(head).>(maxRest)
            else maxRest
        }
    }
}

// 上面的orderer是一个转换的函数，
// Scala Compiler提供了一种更简单的简洁的写法: View Bound
def maxList[T <% Ordered[T]](elements: List[T]): T = {
    elements match {
        case Nil => throw new NoSuchElementException("empty list!")
        case head :: Nil => head
        case head :: tail => {
            val maxRest = maxList(tail) // maxList(tail)(orderer) 
            if (head > maxRest) head // orderer(head).>(maxRest)
            else maxRest
        }
    }
}

// 运用: 找出年龄最大的女生
class Girl(val name: String, val age: Int) extends Ordered[Girl] {
	override def compare(that: Girl): Int = age - that.age
}
val girls = Girl("zml", 27) ::  Girl("allen", 25) :: Nil
maxList(girls)
```
其实上面这种写法还不能精确的传达View Bound的含义，实际上使用的Upper Bound因为我使用的是Ordered的子类。但根据View Bound的定义，只要可以被当做Ordered类型的都可以传入，Girl当然可以被传进去。

下面举一个更贴切的例子，通过隐转满足视界:

```scala
def getIndex[T, CC](seq: CC, value: T)(implicit conv: CC => Seq[T]) 
    = seq.indexOf(value)
def getIndexViaContextBound[T, CC <% Seq[T]](seq: CC, value: T) 
    = seq.indexOf(value)

getIndexViaContextBound("abc", 'c')
```
Scala编译器会首先找到下面的隐转

```scala
implicit def wrapString(s: String): WrappedString = 
    if (s ne null) new WrappedString(s) else null

class WrappedString(val self: String) extends AbstractSeq[Char] 
with IndexedSeq[Char] with StringLike[WrappedString] {
// IndexSeq 是有indexOf方法的
// "abc" ->  wrapString("abc")
```

### 隐式类型的规则
<1> 只有使用implicit标记的定义(val, def, class)才会被编译器当作隐式类型去使用

<2> 插入的隐式转换必须以单一标识符的形式存在于作用域中，或是与源类型或目标类型相关联。

前半句的意思就是x + y可以被转换成 convert(x) + y, 但是不会被转换成SomeVariable.convert(x) + y。如果非要使用后一种必须要显示的引入进来。

后半句的意思是：

```scala
object Dollar {
	implicit def dollarToEuro(dollar: Dollar): Euro...
}
object Euro {
	implicit def dollarToEuro(dollar: Dollar): Euro...
}
class Dollar {}

def testCompanionScope(euro: Euro) = ...
```
如上面的代码，如果某个需要一个Euro作为参数方法被传入了Dollar类型的，这时Scala编译器会搜索Dollar(源类型)或Euro(目标类型)的<b style="color:red">伴生对象</b>。

<3> 需要隐式类型的地方，一次只能进行一次转换

也就是不可能有x + y被编译器隐式转换成 convert1(convert2(x)) + y, 这种写法会导致实际运行的代码跟你预期的有很大的不同，而且使代码变得不清晰。

<4> 如果<b style="color:red">通过编译器检测</b>，那么隐式类型便不会被用到

<5> 如果某个作用域有多个隐式转换，我们可以通过显示的声明来控制哪个隐式转换需要用到
    
### 隐式类型的寻找
<1> 搜索当前作用域

```scala
implicit val limit = 10
def paginateByXX(search: String)(implicit limit: Int) = ???
```

<2> 显示的引入

```scala
import scala.collection.JavaConversions.mapAsScalaMap 
val env = System.getenv
env.apply("USER")
```

<3> 某个类型的伴生对象(Dollar, Euro所描述的)

<4> 在类型参数中寻找

eg1: Scala类库中的sorted

```scala
class A(val n: Int) {}
object A {
    implicit val ord: Ordering[A] = new Ordering[A] {
        override def compare(x: A, y: A): Int = x.n - y.n
    }
}

List(new A(3), new A(5)).sorted
```
很明显上面的sorted方法需要传入一个隐式的ord参数，但是Ordering[A]根本没有这样的转换，这时候Scala编译器就会去类型参数A中去寻找即A的<b style="color:red">伴生对象</b>中定义的ord。
 	                                  
# 结语
Scala的隐式类型还是比较复杂的并且涉及到很多类型方面的知识。尽管它为我们提供了很多编程方面的便利，但是在使用时还是需要谨慎，注意作用域否则就可能带来意想不到的"转换"

# 参考
<1> [finding implicits](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html)

<2> Programming In Scala - Implicit Conversions and Parameters(Chapter 21)




---
layout: post
category: scala
date: 2015-02-08 14:51:31 UTC
title: Scala基础之隐式转换(implicit-conversion)
tags: [隐式转换, 隐式参数, 隐式方法, 隐式类]
permalink: /scala/implicits/
key: dde9e45d61a77e15ceb5bbf9cfa1e6ac
description: "本文研究了隐式转换，隐式参数, 视界的概念以及Scala编译器寻找隐式类型的顺序"
keywords: [隐式转换, 隐式方法, 隐式类, 视界, 类型参数]
---

无论是对于初学者，还是对于有经验的Scala程序员，隐式转换都是一个必须掌握的技巧。它可以进行类型转换，方法调用还有类功能增强。如果高级一点，就是在[类型类](/scala/type-class/)中的使用。

隐式转换是指Scala编译器会帮助完成指定的"转换"，从而减少重复代码并且在一定程度上扩展现有库的功能。隐式参数就是不需要你显示传递参数(Scala编译器会帮你传递当然你也可以显示的传递)。

### 基本使用

(1) **减少重复代码**

如果`class C`的某个实例o调用了方法m，即o.m, 但实际上o并没有m方法。这时候Scala编译器就会去寻找能够把o转成支持m方法调用的一个类型。
在Java Swing中我们会给组件注册监听事件，通常会有如下代码:

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
```

利用隐式转换避免每次注册监听事件都要重复实现ActionListener，当然仅仅是对于通用的逻辑。

(2) **功能扩展**

比如说我们想扩展某个jar中类的功能，就可以采用此方法。

```scala
// 给String添加新方法
class StringImprovement(s: String) {
    def increment = s.map(one => (one + 1).toChar)
}
implicit def stringIncrement(s: String) = new StringImprovement(s)

"abc".increment

//还有一个更为常见
package scala
object Predef {
    class ArrowAssoc[A](x: A) {
      def -> [B](y: B): Tuple2[A, B] = Tuple2(x, y)
    }
    implicit def any2ArrowAssoc[A](x: A): ArrowAssoc[A] =
      new ArrowAssoc(x)
    ...
}

Map(1 -> "one", 2 -> "two") 
```

### 隐式参数

编译器会自动帮你传递这个参数，如果找不到就会报错。简单来说就是`someCall(a)`在实际调用的时候会变成`someCall(a)(b)`。

```scala
class Welcome(val msg: String)
object Greeter {
    def greet(name: String)(implicit welcome: Welcome) = {
        s"${name}, ${welcome}" 
    }
}

// 显示调用
Greeter.greet("allen")(new Welcome("欢迎回来"))

object GreetingSetting {
    implicit val welcome = new Welcome("欢迎回来")
} 

//引入隐式参数
import GreetingSetting._
Greeter.greet("allen")
```

Scala内置的CanBuildFrom就利用了这一点

```scala
trait CanBuildFrom[-From, -Elem, +To] {}

def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That = {}
```

###  视界(View Bound)

对于`O <% T`, 只要O可以被隐式转换成T或者O是T的子类或者就是T类型，那么我就可以随意使用O。

```scala
def maxList[T <% Ordered[T]](elements: List[T]): T = {
    elements match {
        case Nil => throw new NoSuchElementException("empty list!")
        case head :: Nil => head
        case head :: tail => {
            val maxRest = maxList(tail) 
            if (head > maxRest) head
            else maxRest
        }
    }
}

// Int => intWrapper(隐式转换成RichInt， 而RichInt是Ordered的子类)
maxList(List(1,2,3))


// 下面两种写法等价
def getIndex[T, CC](seq: CC, value: T)(implicit conv: CC => Seq[T]) 
    = seq.indexOf(value)
def getIndexViaViewBound[T, CC <% Seq[T]](seq: CC, value: T) 
    = seq.indexOf(value)

getIndexViaContextBound("abc", 'c')
```

```scala
// 隐式转换搜索的过程
implicit def wrapString(s: String): WrappedString = 
    if (s ne null) new WrappedString(s) else null

// IndexSeq有indexOf方法的
// "abc" ->  wrapString("abc")
class WrappedString(val self: String) extends AbstractSeq[Char] 
with IndexedSeq[Char] with StringLike[WrappedString] {
```

###  隐式类型的规则

\> 只有使用implicit标记的定义(val, def, class)才会被编译器当作隐式类型去使用

\> 插入的隐式转换必须以单一标识符的形式存在于作用域中，或是与源类型或目标类型相关联。

前半句的意思就是`x + y`可以被转换成`convert(x) + y`, 但是不会被转换成`SomeVariable.convert(x) + y`。如果非要使用后一种必须要显示的引入进来。

后半句的意思是:


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
如上面的代码，如果某个需要一个Euro作为参数方法被传入了Dollar类型的，这时Scala编译器会搜索Dollar(源类型)或Euro(目标类型)的<b style="color:red">伴生对象</b>以锁定相应的隐式转换。

\> 需要隐式类型的地方，一次只能进行一次转换

也就是不可能有`x + y`被编译器隐式转换成`convert1(convert2(x)) + y`, 这种写法可能会导致实际运行的代码跟你预期的有很大的不同，而且使代码变得不清晰。

\> 如果<b style="color:red">通过编译器检测</b>，那么隐式类型便不会被用到

\> 如果某个作用域有多个隐式转换，我们可以通过显示的声明来控制哪个隐式转换需要用到
    
###  隐式类型的寻找

\> 搜索当前作用域

```scala
implicit val limit = 10
def paginateByXX(search: String)(implicit limit: Int) = ???
```

\> 显示的引入

```scala
import scala.collection.JavaConversions.mapAsScalaMap 
val env = System.getenv
env.apply("USER")
```

\> 某个类的伴生对象(Dollar, Euro所描述的)

```scala
class A(val n: Int) {}
object A {
implicit val ord: Ordering[A] = new Ordering[A] {
    def compare(x: A, y: A): Int = x.n - y.n
}
}

// def sorted[B >: A](implicit ord: Ordering[B]): Repr...
List(new A(3), new A(5)).sorted
```

很明显上面的sorted方法需要传入一个隐式的ord参数，但是Ordering[A]根本没有这样的转换，这时候Scala编译器就会去类型参数A中去寻找即A的<b style="color:red">伴生对象</b>中定义的隐式ord。

关于作用域，下面的案例也能说明问题:

```scala
trait FKTC[T] {
    def value: T
}

// companion here is for Scala Compiler to find 
object FKTC {
  implicit val defaultInt = new FKTC[Int] {
    def value = 5
  }
  
  implicit def listInt[T: FKTC] = new FKTC[List[T]] {
     def value = implicitly[FKTC[T]].value :: Nil
  }
}

object FKTCTest {

    implicit val defaultInt = new FKTC[Int] {
        def value = 43
    }
    
    def default[T: FKTC] = implicitly[FKTC[T]].value

    default[Int] 
    default[List[Int]] 
}
```

defaultInt的存在与否，将会影响最后两行的结果，如果存在则结果分别为43, List(43); 反之则为5, List(5)。

在实际开发过程中，Scala隐式转换使用起来还是非常方便的，只要注意好作用域，基本不会有太大的问题。当然也有，隐式转换的层次太多反而让人理解起来有点困难的，典型的就是[Spray Directive](/spray/directive/)。所以，有时要注意权衡一下简洁和可读性。

##  参考

\> [隐式转换的寻找规则](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html)

\> Programming In Scala - Implicit Conversions and Parameters(Chapter 21)

\> [隐式转换 vs 类型类](http://stackoverflow.com/questions/8524878/implicit-conversion-vs-type-class)





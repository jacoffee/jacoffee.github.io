---
layout: post
category: spray
date: 2015-05-21 19:51:22 UTC
title: Spray基础之Directive(路由)
tags: [路由，指令，管道，Shapeless，响应式编程]
permalink: /spray/directive/
key: 648a1d4bee718608389fbafe9c484311
description: "本文对于Spray对于Directve进行了比较详细的讲解"
keywords: [路由，指令，管道，Shapeless，响应式编程]
---

在谈论Directive之前，我们先要简单的解释一下[Routes(路由)](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/routes/), 它的类型是**```RequestContext => Unit```**，在英文中Route是路径，路线的意思。所以在Spray中可以理解为
<b style="color:red">
 Route承担了如何处理请求的任务，将请求指派到什么流程；一般来说，它有三种方式处理请求：
 + 顺利完成请求 requestContext.complete(...)
 + 因为某种原因拒绝请求 requestContext.reject(...)
 + 因为某些异常而导致 500 Server Error
</b>

在最开始接触Routes的时候的一个疑惑，Route函数(**```RequestContext => Unit```**)的返回值类型为什么是**```Unit```**。

> Contrary to what you might initially expect a route does not return anything. Rather, all response processing (i.e. everything that needs to be done after the route itself has handled a request) is performed in “continuation-style” via the **responder of the RequestContext**
  
一般来说，在B/S的架构中**请求**是和**响应**相对应的，如果将**RequestContext**视为请求上下文，那么返回值类型就应该是Response之类的。后来阅读源码之后发现，这与Spray的设计理念是有关的，它是一个基于Actor的Rest框架，相应的响应是通过**消息发送**传递的而且采用的是**Fire-And-Forget**的模式，所以返回值是**``Unit``**。后续会再写一篇文章分析Spray建立TCP连接并准备接受请求的过程，这样的话对于Spray的基本骨架会有一个清晰的认识。
  
```scala
case class RequestContext(req: HttpRequest, responder: ActorRef, 
  unmatchedPath: Uri.Path) {
  
  def complete[T](obj: T)(....): Unit =  {
    val ctx = new ToResponseMarshallingContext {
        ..
        def marshallTo(response: HttpResponse) = responder ! response
        ..
    }
  }
  
}
```
  
而[Directive](http://spray.io/documentation/1.2.2/spray-routing/key-concepts/directives/#directives)从数据结构上来讲是基于**Shapeless HList**的高度抽象，读起来比较困难，但是理解之后会感受到类型系统的奥妙; 实际上这是Scala类型系统的一个弊端同时也是隐式转换的弊端: 学习曲线高， 太多的"过程"被隐藏导致的(这一点在后面的代码中会体现的更明显)。

从功能上来讲，它提供了各种各样的**指令**来构建Routes; 它的一般结构是:

```scala
name(parameters) { L =>
  innerRoute
}
```

 PathDirective用于抽取部分URL

```scala
path(uid / "key") { uid =>
    innerRoute
}
```
 
在Restful风格的编程中，我们通常会将资源的唯一标识放在URL中，通过上面的**PathDirective**我们可以从URL中抽取相应的数据。上面的规则适用于该URL"http://localhost/534e8effe4b078a6b660f705/uid"。

SecurityDirective用于请求验证

```scala
path(uid / "key") { uid =>
    authorize { ctx =>
       // 获取URL参数 | 请求头等来进行权限验证
       boolean
    } {
      innerRoute
    }
}
```
   
接下来看看几种常见的使用Directive实现[Route的实例](http://spray.io/documentation/1.2.2/spray-routing/advanced-topics/understanding-dsl-structure/)。

#### <1> 最基本的Route实现
 
```scala
// get的类型是Directive0
get {
  // complete的类型: ToResponseMarshallable => StandRoute
  // Spray提供了很多隐式转换将不同类型转换成ToResponseMarshallable，String正是其中之一
  // complete("")的类型是 RequestContext => Unit
  complete("")
}
```

**``HttpServiceBase``**要求所有的子类调用**``runRoute``**方法的时候提供**``Route``**函数实例

```scala
 // type Route = RequestContext => Unit
 trait HttpServiceBase extends Directives {
   def runRoute(route: Route)(...) {}
 }
```

再来看一下抽象类**``Directive``**的基本结构:

```scala
abstract class Diretive[L <: HList] {
  // L的类型为HNil
  def happly(func: L => Route): Route
  // 组合操作
  def &(magnet: ConjunctionMagnet): magnet.OUT = magnet(this)
  ...
}

object Directive {
    /*
        H的类型: HNil
        hac.apply的类型: def apply(in: Route): HNil => Route
        hac.In的类型 => Route: Route => Route
        hac(f)的类型 HNil => Route
        happly(HNil => Route)的类型: Route
    */
    implicit def pimpApply[H <: HList](directive: Directive[H])
        (implicit hac: ApplyConveter[H]): hac.In => Route = {
        f => directive(happly(hac(f)))
    }
}
```
 
Directiv0实例经由Directive中的隐式方法pimpApply被转换:

```scala
abstract class ApplyConverter[L] {
    type In
    // L的类型为HNil 所以apply的返回值: HNil => Route
    // In的类型为Route
    def apply(in: In): L => Route 
}

// 这种模式在Spray中大量使用，伴生对象并没有直接继承伴生类
// 而是继承了另外一个提供了伴生类实例的类或特质
object ApplyConverter extends ApplyConverterInstances {
    implicit val hac0 = new ApplyConverter[HNil] {
        type In = Route
        
        def apply(fn: In) = {
            case HNil => fn
        }
    }
}
```

最终Get的类型为**```Route => Route```**，而complete("")类型是**```StandardRoute```**，函数调用的结果是**```Route```**，正好是**```runRoute```**方法所需要的。


#### <2> 涉及到Directive组合操作的Route实现

```scala
path("xx") & authorize(true)
```
  
上述操作是一个简单的Directive组合操作，目的是为了在路径匹配之后增加额外的验证; 比如说为App提供API的话，每一次请求除了要验证Path，还需要验证相应的权限。

```scala
// ConjunctionMagnet
trait ConjunctionMagnet[L <: HList] {
  type OUT 
  def apply(underlying: Directive[L]): OUT
}
  
// 这里伴生对象又没有直接继承伴生类(这种模式上面已经提到过了)
// prepender提供了进一步的拼接操作; prepender.Out拼接之后的结果(HList)
object ConjunctionMagnet {
  implicit def fromDirecive[L <: HList, R <: HList](other: Directive[R])
    (implicit prepender: Prepender[L, R]) = {
       
       new Directive[prepender.Out] {
         type Out = Directive[prepender.Out]
         def apply(left: Directive[L]): Out = {
           new Directive[prepender.Out] {
             def happly(f: prepender.Out => Route): Route = {
               left.happly { l =>
                 right.happly { r =>
                    f(prepender(l, r))
                 }
               }
             }
           }
        }
      }
   }
}
```

#### <3> 涉及到Future Directive组合操作的Route实现

```scala
  
val routes: spray.routing.Route = {
  get {
    // Normally T is a case class
    onSuccess(.. Future[T]..) {
        // T => Route => Route
        (t: T) => complete("t")
    }
  }
}
  
```

![Future Directive简单流程](/static/images/charts/2015-05-21/FutureDirective.png)

```scala
trait OnSuccessFutureMagnet {
  type Out <: HList
  def get: Directive[Out]
}
  
object OnSuccessFutureMagnet {
 implicit def apply[T](future: ⇒ Future[T])(implicit hl: HListable[T], ec: ExecutionContext) =
   new Directive[hl.Out] with OnSuccessFutureMagnet {
    ...
    type Out = hl.Out
    ...
   }
}
```

上面**``HListable[T]``**的实现还是挺有意思，基本规则如果**``T <: HList``**，直接返回，否则 使用``T :: HNil``构建新的**``HList``**，下面模仿它的实现思想，实现一个**``Listable``**。

```scala
trait Listable[T] {
   type Out <: List[T]
   def apply(t: T): Out
}

object Listable extends LowerpriorityListable {
   implicit def fromList[T <: List[T]] = new Listable[T] {
     override type Out = T
     override def apply(t: T): Out = t
   }
}

abstract class LowerpriorityListable {
    implicit def fromAnyRef[T] = new Listable[T] {
      import scala.collection.immutable.::
      override type Out = ::[T]
      override def apply(t: T): Out = ::(t, Nil)
    }
}
```

在HListable实现过程中，有一个地方需要注意:

```scala
abstract class LowerpriorityHListable { 
  implicit def fromAnyRef[T] = new HListable[T] {
    override type Out = T :: HNil
    override def apply(t: T): Out = t :: HNil
  }
}

final case class ::[+H, +T <: HList](head: H, tail: T) extends HList {}
```

Out类型的定义实际上用到了中置操作符(Infix Operations): **``T :: HNil => ::[T, Nil]``**，实际上就是生成了::类型。



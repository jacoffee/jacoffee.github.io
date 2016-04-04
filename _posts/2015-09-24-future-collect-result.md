---
layout: post
category: concurrency
date: 2015-09-24 09:51:22 UTC
title: 从Future产生到结果收集
tags: [异步计算，共享变量，链向根Promise，回调]
permalink: /concurrency/future/result-collect/
key: 488b299b8446fc87aa490a28629fcd61
description: "本文就Future的初始化以及结果收集的过程进行了研究"
keywords: [异步计算，共享变量，链向根Promise，回调]
---

Future在异步编程中使用非常广泛，Scala中的Actor就是建立在Future之上的，而Actor又在Akka和Spark中被广泛使用，重要性不言而喻。对于Future的语法和基本使用，网上已经有太多的资源了。本文想以源码为基础，就Future的结果收集进行剖析，不求多么深刻，但求能对结构收集的每一步有个大致的了解。

> A Future is a data structure used to retrieve the result of some concurrent operation. This result can be accessed synchronously (blocking) or asynchronously (non-blocking).

## <1> Future的结果 - 非阻塞式 vs 阻塞式收集

```scala
val sumResult = Future((1L to 100000000L).sum)
```

非阻塞式的取值(onComplete | onSuccess | onFailure)

```scala
sumResult onComplete {
  case Success(s) => println(" Future succeed " + sumResult)
  case Failure(f) => println(" Future failed")
}
```

阻塞式的取值(Await.result)

```scala
Await.result(sumResult, Duration.Inf)
```

使用Future的初衷是为了不阻塞，所以尽量不要使用阻塞式取值，除非你不得不这么做。在项目开发中，一般通过**```For ... Yield```**将多个Future组合起来，最后调用Await.result获取最终的结果。

```scala
for {
    A <- Future[A]
    B <- Future[B]
    C <- Future[C]
} yield {
    op(A, B, C)
}
```

针对阻塞式的取值，以上面的代码为例就会阻塞主线程:


```scala
package scala

// STEP1
package concurrent {
    object Await {
       def result[T](awaitable: Awaitable[T], atMost: Duration): T =
        blocking(awaitable.result(atMost)(AwaitPermission))
    }
}

// STEP2 
package object concurrent {
   def blocking(body: => T) = 
       BlockContext.current.blockOn(body)(scala.concurrent.AwaitPermission)
}


// STEP3
package scala.concurrent

object BlockContext {

    private val contextLocal = new ThreadLocal[BlockContext]()
    
    def current = contextLocal.get {
        case null => Thread.currentThread match {
            case ctx: BlockContext => ctx
            case _ => defaultBlockContext
        }
        case some => some
    }

}
```

在最初阅读**```current```**方法时候会疑惑，为什么**```Thread.currentThread```**会匹配**```BlockContext```**类型，源码中的解释是: 有些线程池实现的时候会让Thread继承BlockContext。

对于非阻塞式的取值，我们可以通过Future的相关方法来判断它的当前状态。

```scala
sumResult.isCompleted
```

显然**onComplete**，**onSuccess**，**onFailure**这些回调也是根据当前Future的状态来执行相应的操作的。那么Future内部的状态有哪几种呢？这些状态之间又是如何切换的呢？

## <2> Future的状态以及改变

要研究Future的状态变化，最好的方式就是利用**onComplete**的调用栈并且打断点来摸清整个调用过程，下面通过代码调用来一步一步解释(由于之前没有接触过UML图，所以可能存在一些纰漏，以下涉及到UML图的地方仅供参考)。

![Future的初始化和结果收集UML](/static/images/charts/2015-09-24/future_init.png)

### (1) Future的初始化 -- Future.apply

Future初始化的时候(实际上是DefaultPromise的初始化)，会调用**```AbstractPromise```**的updateState方法来设置初始状态。

```scala
class DefaultPromise[T] extends AbstractPromise { self =>
 updateState(null, Nil) // 实际上更新的是AbstractPromise中的_ref属性
}
```

所以，**Future初始化之后状态变成了Nil(类型实际上是List[CallbackRunnable])**。

### (2) Future中的任务开始执行 -- PromiseCompletingRunnable

```scala
  // executor ---> package scala.concurrent.impl.ExecutionContextImpl    
    def apply[T](body: => T)(implicit executor ExecutionContext): 
       scala.concurrent.Future[T] = {
       /*
          这里也体现了Promise的语义执行Runnable并且返回Future
          每次启动一个异步计算的时候，PromiseCompletingRunnable都会被创建一次
          body是懒加载的，所以在调用的时候才会被执行
       */
       val runnable = new PromiseCompletingRunnable(body)
       
       // 异步计算从这个地方开始然后调用PromiseCompletingRunnable的run方法
      executor.prepare.execute(runnable) 
              
      // 返回Future，实际上是DefaultPromise实例，以便后续使用，比如说Compose
      runnable.promise.future
    }
  }
```

而在**```PromiseCompletingRunnable```**执行的时候，会将**执行结果**封装在**```Try[T]```**中，所以onComplete的回调函数的参数也是**Try[T]**类型的。

```scala
class PromiseCompletingRunnable[T](body: => T) {
       val promise = new Promise.DefaultPromise[T]()
       override def run() = {
          promise complete {
              try {
                Success(T)
              } catch {
                case NonFatal(e) => Failure(e)
              }
              
          }       
       }
}
```

#### 1. DefaultPromise.complete

```scala
// 如果该Promise已经完成则报错，否则就返回DefaultPromise的当前实例
def complete(result: Try[T]): this.type = 
  if (tryComplete(result)) this else throw IllegalStateException("Promise already completed.") 
```

#### 2. DefaultPromise.tryComplete

```scala
// tryComplete返回true则说明没有完成，返回false则说明完成
def tryComplete(result: Try[T]) = {
    tryCompleteAndGetListeners(result) {
       case null => false
       case rs if rs.isEmpty => true
       case rs => rs.foreach(r => r.executeWithValue(result)); true;
    }
}
```

#### 3. DefaultPromise.tryCompleteAndGetListeners

```scala
@tailrec
private def tryCompleteAndGetListeners(v: Try[T]): List[CallbackRunnable[T]] = {
  getState match {
    case raw: List[_] =>
      val cur = raw.asInstanceOf[List[CallbackRunnable[T]]]
      if (updateState(cur, v)) cur else tryCompleteAndGetListeners(v)
      
    case _: DefaultPromise[_] =>
      compressedRoot().tryCompleteAndGetListeners(v)
    case _ => null
  }
}
```

  前面提到过，**DefaultPromise**的状态是由**AbstractPromise**的**_ref**属性维护的。状态有以下几种:
  
  + **List[CallbackRunnable[T]]**(最开始的状态是Nil)。
  则接下来会将**DefaultPromise**的状态设置成**Try[T]**。在sumFuture的那种情况中，模式匹配会进到这种情况。然后**tryComplete**返回当前**DefaultPromise**实例并且它的状态已经变成了**Try[T]**

  + **DefaultPromise[_]**(一般是调用了map, flatMap生成了新的DefaultPromise)，这里涉及到链向根DefaultPromise的过程(会在后续的更新中补充)

  + 其它情况，则返回null

### (3) Future注册回调 -- Future.onComplete

Future的异步调用体现在当Future初始化之后，一个异步计算就已经开始；一般情况，我们会为Future注册相关的回调，要么是对正常的返回值进行处理，要么是对异常进行处理。同样以源码入手。

```scala
sumResult onComplete {
  case Success(s) => println(" Future succeed " + sumResult)
  case Failure(f) => println(" Future failed")
}
```

简化版的调用链如下:

![dispatchOrAddCallback](/static/images/charts/2015-09-24/dispatchOrAddCallback.png)

#### 1. DefaultPromise.onComplete

```scala
def onComplete[U](func: Try[T] => U)(implicit executor: ExecutionContext): Unit = {
    ...
    val runnable = new CallbackRunnable(executor, func) // 1.1
    dispatchOrAddCallback(runnable) // 1.2
}
```

#### 1.1 scala.concurrent.impl.CallbackRunnable

因为Java中Runnable的实现(run方法)默认是返回空的，所以此处基于Runnable再实现了一个，主要的目的是为了注入回调，让Future完成之后执行该回调。

```scala
class CallBackRunnable(val executor: ExecutionContext, val onComplete: Try[T] => Any) 
     extends Runnable with OnCompleteRunnable {
     var value: Try[T] = null
     
     override def run() = {
        // 与executeWithValue中require相结合，执行run的时候value必须不为null
        require(value ne null) 
        
        try {
          // case Success(s) => println(" Future succeed " + sumResult)
          // case Failure(f) => println(" Future failed")
          onComplete(value)
        } catch {
          case NoFatal(e) => ....
        }
     }
     
    def executeWithValue(v: Try[T]) = {
        
        // 确保回调函数没有被多次执行
        require(value eq null) 
        value = v
        
        try executor.execute(this) 
        catch {
          case NonFatal(e) => ....
        }
     }

}
```

#### 1.2 DefaultPromise.dispatchOrAddCallback
    
这个方法的主要目的是获取Future的状态，从而决定是执行回调还是将回调转移给根Promise。注意这个过程中**```Future```**的执行会导致**```Promise```**的状态发生变化，也就是前面提到的**```PromiseCompletingRunnable中的run方法```**。 以sumFuture为例，

```
private def dispatchOrAddCallback(runnable: Runnable): Unit = {
    getState match {
      case r: Try[_]          => runnable.executeWithValue(r.asInstanceOf[Try[T]])
      
      case _: DefaultPromise[_] => compressRoot().dispatchOrAddCallback(runnable)
      
      case listeners: List[_] => 
         if (updateState(listeners, runnable :: listeners)) () 
         else dispatchOrAddCallback(runnable)
    }
  }
```

 前面提到过，**DefaultPromise**的状态有三种情况，在**执行回调函数的线程上**(Future运行和回调执行是不同的线程)通过取Future的状态来决定如何操作
 
  + **Try[T]** 
    表示已经执行完了，可以开始执行回调函数，这个比较好理解。**CallbackRunnable**的**executeWithValue**被调用，触发**run**方法，从而执行**onComplete**中注册的回调。

  + **DefaultPromise[_]**(一般是调用了map, flatMap生成了新的DefaultPromise)，
    这里涉及到链向根DefaultPromise的过程(会在后续的更新中补充)

  + **List[CallbackRunnable]**，
    Future的初始状态是**Nil(List[CallbackRunnable])**，<b style="color:red">如果执行回调函数的线程在执行的时候，Future的执行并没有结束就会将Future的状态更新为List[CallbackRunnable]， 实际上注册了一个监听器(在Future执行完成之后做什么)</b>。代码中如果状态更新成功，什么都没做，返回值是**``()``**。
   由于Future执行线程和回调函数执行线程共享Future的状态，所以  在**``tryCompleteAndGetListeners``**中也能捕获到状态的变化然后进入不同的操作。
  

至此，一个最基本的Future初始化以及注册回调并收集结果的流程就走完了(对于，状态为DefaultPromise的情况会在后续的更新中完成)。

以下是本文的一些汇总:

+ Future的执行和结果搜集发生在不同的线程上，Promise以及DefaultPromise中的一些方法分别在<b style="color:red">两个线程被调用但是同时修改同一个Future的状态</b>，所以能根据Future不同的状态进行不同的响应。

+ Future的状态由父类**AbstractPromise**中的**_ref**属性维护

+ Future在初始化之后的状态为**List[CallbackRunnable]**，执行完成之后的状态为**Try[T]**，如果通过map，flatMap与其它的Future产生了联系，则状态为**DefaultPromise**。
---
layout: post
category: spark
date: 2017-06-03 21:30:55 UTC
title: Spark基础之EventLoop实现
tags: [行为封装，回调，Lock，Condition，线程唤醒]
permalink: /spark/evenloop
key: 
description: 本文介绍了Spark中Eventloop的实现以及利用它实现一个简单的定时发送消息的KafkaProducer
keywords: [行为封装，回调，Lock，Condition，线程唤醒]
---

Spark内部的很多模块会将相关的行为抽象成一个个事件:

<ul class="item">
    <li>
        Spark Core中与DAGScheduler相关的各种行为，如JobSubmitted，JobCancelled，StageCancelled等
    </li>
    <li>
        Spark Streaming中与JobScheduler相关的行为，如JobStarted，JobCompleted等
    </li>
</ul>

这些事件产生之后，必然有后续的处理，Spark利用EventLoop来将这两种行为解耦，一般需要处理各种行为的组件内部都有一个EventLoop成员变量，并且在自身启动的时候，初始化EventLoop。组件内部也会为各种事件定义了对应的处理机制。

## EventLoop实现

EventLoop主要包含如下几个方面:

<ul class="item">
    <li>用于接受事件的双端队列(LinkedBlockingDeque) -- eventQueue。Blocking体现在当队列满了之后，如果再次插入元素的操作将会被阻塞， 如果队列是空的话，该操作同样会被阻塞，直到有了元素</li>
    <li>一个不断从队列中取出事件并且执行的守卫进程(Daemon Thread) -- eventThread</li>
    <li>控制线程是否继承执行的条件变量(condition) -- stopped，这个模型在Java Thread中使用的非常多</li>
    <li>
        取出事件之后的回调 -- onReceive，不同的事件对应不同的处理，有点类似Actor中的receive模式
    </li>
</ul>

![EventLoop](http://static.zybuluo.com/jacoffee/qemvdsfchmauxr2d2rlcq1bx/image.png)

<p align="center">图1.EventLoop的组成</p>

下面我们通过JobGenerator(Spark Streaming)中的EventLoop的执行流程来更好理解上面这幅图:

<b class="highlight">(1) 启动EventLoop </b>

在JobGenerator启动的时候，EventLoop将被真正赋值(**null -> new EventLoop**)并启动，内部eventThread开始从eventQueue中取事件，如果没有则阻塞。

<b class="highlight">(2) 在指定时间点，提交GenerateJobs事件 </b>

```scala
private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
    longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

eventThread从eventQueue获取事件，触发onReceive方法，进而调用JobGenerator的**processEvent**方法，包括各种事件的具体处理逻辑。

```scala
private def processEvent(event: JobGeneratorEvent) {
    event match {
      ...
      case GenerateJobs(time) => generateJobs(time)
      ...
    }
  }
}
```

<b class="highlight">(3) JobGenerator关闭的时候同时停止内部的EventLoop </b>

关闭EventLoop主要的任务就是停止内部的eventThread，以及子类实现的**onStop()**方法。

```scala
def stop() = {
    if (compareAndSwap(false, true)) {
        eventThread.interrupt()
        try {
            eventThread.join()
            onStop()
            ....
        } catch {
            case ie: InterruptedException =>
              // 这是为了让打断状态能被上层的方法和线程组知晓
              Thread.currentThread().interrupt()
        }
    }
}
```

## 实际应用

在开发或者是测试中，我们经常会碰到这样的需求，特别是在Spark Streaming中，我们需要Kafka能够定时发送一些消息，比如说每5秒一次，一般我们会这样实现。

```scala
def produceRecord() {}

while (true) {
    produceRecord()
    Thread.sleep(5000)
}
```

虽然能够达到目的，但是显得不够fashion，所以我们可以尝试利用**org.apache.spark.streaming.util.RecurringTimer**和**EventLoop**来实现一个定时发送Kafka消息的程序，由于这两个类在Spark中都设置了**包访问权限**，所以我们可以在自己的程序中按`package org.apache.spark.streaming`构建包名，这样就可以访问这两个类了。至于具体实现则要注意以下几点:

<ul class="item">
    <li>用于发送消息的KafkaProducer</li>
    <li>将与KafkaMessage相关的行为抽象成事件，如GenerateMessages</li>
    <li>与KafkaMessage事件相关的EventLoop负责接收事件，与事件的处理。</li>
    <li>随着应用程序一起初始化的Timer，用于定时向EventLoop提交GenerateMessages事件</li>
    <li>各种事件的实际逻辑处理方法 -- processEvent</li>
</ul>

具体实现:

```scala
package org.apache.spark.streaming

import java.util.Properties
import java.util.concurrent.locks.ReentrantLock
import scala.util.Random
import org.apache.kafka.clients.producer.{ProducerRecord, KafkaProducer}
import org.apache.spark.streaming.util.RecurringTimer
import org.apache.spark.util.{EventLoop, SystemClock}

// behavior related to KafkaMessageProducer encapsulated as case class/object Event
sealed trait KafkaMessageEvent
case class GenerateMessages(time: Time) extends KafkaMessageEvent

object KafkaMessageProducer {

  private val lock = new ReentrantLock()
  private val condition = lock.newCondition()

  var eventLoop: EventLoop[KafkaMessageEvent] = null
  var producer: KafkaProducer[String, String] = null

  def start(): Unit = {
    eventLoop = new EventLoop[KafkaMessageEvent]("MessageGenerator") {
      override protected def onReceive(event: KafkaMessageEvent): Unit = processEvent(event)
      override protected def onError(e: Throwable): Unit = throw e
    }
    eventLoop.start()
    producer = getKafkaProducer()

    val startTime = new Time(timer.getStartTime())
    timer.start(startTime.milliseconds)
  }

  def getKafkaProducer() = {
    val props = new Properties()
    props.put("bootstrap.servers", "localhost:9092")
    props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    new KafkaProducer[String, String](props)
  }

  // execution interval
  val durationInMillis = Seconds(5).milliseconds
  val timer = new RecurringTimer(
      new SystemClock, durationInMillis,
      longTime => eventLoop.post(GenerateMessages(new Time(longTime))), "MessageGenerator"
  )

  // called by EventLoop's onReceive to handle the logic
  def processEvent(kafkaMessageEvent: KafkaMessageEvent) = {
    kafkaMessageEvent match {
      case GenerateMessages(time) => sendMessage(producer, time)
    }
  }

  def sendMessage(producer: KafkaProducer[String, String], time: Time) = {
     val alphabets = List("A", "B", "C", "D")
     Random.shuffle(alphabets).take(1).foreach { alpha =>
       val producerRecord: ProducerRecord[String, String] = new ProducerRecord("alphabets", alpha)
       producer.send(producerRecord)
     }
  }

  def main(args: Array[String]): Unit = {
      start()
      waitTillEnd()
  }

  // used to block the program
  def waitTillEnd(): Unit = {
    lock.lock
    try {
      condition.await
    } finally {
      lock.unlock
    }
  }

}
```

其实代码本身并没有什么价值，不过我们可以借鉴**行为抽象成事件**以及**EventLoop**这两种思想，并且运用在我们的编程中。
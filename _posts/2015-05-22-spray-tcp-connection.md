---
layout: post
category: spray
date: 2015-05-22 19:51:22 UTC
title: Spray基础之一次请求的响应
tags: [TCP连接，管道操作，流式处理，用消息抽象事件]
permalink: /spray/request-response/
key: 98a3079f40849d37b94af15d8e3f7506
description: "本文对于Spray是如何建立TCP连接并且响应一次请求进行了研究"
keywords: [TCP连接，管道操作，流式处理，用消息抽象事件]
---

区别于Java EE的传统框架，Spray并没有使用容器(Tomcat, Jetty之类的); 不过它提供了[spray-servlet](http://spray.io/documentation/1.2.3/spray-servlet/#spray-servlet)模块来支持Servlet，它实际上是在Servlet API基础上搭建了[spray-can](http://spray.io/documentation/1.2.3/spray-can/)(spray server的核心部分)，这意味Spray是比较轻量的。 基于此决定研究一下Spray是如何建立TCP连接并且响应HTTP请求的。

在Spray中启动Server服务的基本代码如下:

```scala
class RestApi extends HttpServiceActor {
  override def receive: Receive = runRoute(
     path("test")(complete("simple test"))
  )
}

object Boot extends App with ShutdownIfNotBound {
  implicit val system = ActorSystem("req-test")
  val frontEnd = system.actorOf(Props[RestApi], "frontEnd")

  implicit val executionContext = system.dispatcher

  implicit val globalTimeout: Timeout = {
    val d = Duration("10seconds")
    FiniteDuration(d.length, d.unit)
  }

  IO(Http) ? Http.Bind(frontEnd, interface = "127.0.0.1", port = 8082)
}
```

**(1)** IO.apply返回一个用于**处理相应层的管理类Actor**，**spray.can.HttpManager**之于HTTP层，**spray.can.TcpManager**之于TCP层, Spray的层级划分如下:

![Spray的栈](/static/images/charts/2015-05-22/rs_stack.png)

**(2)** 项目启动时，HttpManager接受一个**Http.Bind**绑定信息，目的是让服务器端的Socket监听在指定的**InetSocketAddress**(IP地址+端口号)，当绑定成功之后，就可以开始建立连接了。
下面代码构建了一个简单的spray-client，向服务器发送消息然后接受服务器响应。

```scala
object SprayClient {
	(1 to 3).foreach { num =>
      (IO(Http) ? HttpRequest(uri = "http://localhost:8082/test")).mapTo[HttpResponse]
	}
}
```

具体的流程如下图:

![Spray 端口绑定和请求响应](/static/images/charts/2015-05-22/spray_request.png)

Spray使用Actor来抽象HTTP连接，每一次新的请求就会有一个新的Actor生成，这点可以从命令行打印的日志看出来。 所以官网中提到的[spray-can](http://spray.io/documentation/1.2.3/spray-can/)是完全异步的并且能同时处理千计的连接就不难理解了。

```bash
# HttpHostConnectionSlot
Dispatching GET request to URL across connection Actor[akka://spray-client/user/IO-HTTP/group-0/0]
Dispatching GET request to URL across connection Actor[akka://spray-client/user/IO-HTTP/group-0/1]
Dispatching GET request to URL across connection Actor[akka://spray-client/user/IO-HTTP/group-0/2]
```

<p style="display:none">
**(5)** Actor是通过消息传递来实现异步的，所以在实际编程中，很多实际的逻辑都被抽象成了消息。作为一个基于Akka Actor的框架自然不例外，就TCP的整个生命周期来说，在Spray中被分成了如下几个消息(部分):

```bash
Tcp.Connect
Tcp.Connected
Tcp.WriteCommand
Tcp.Close
```

所以在上图提到的管道操作，会涉及到大量消息的转变(比如说ServerFrontend将HttpMessagePart消息转换成HttpResponsePartRenderingContext，传递给管道中的下游)，实际上也就是请求中的状态改变(从请求阶段变成了响应)。
</p>

在阅读的源码时候还有一个比较重要的地方没有研究，那就是Spray的管道操作，在``**spray.can.server.HttpServerConnection**``中对管道的每一部分进行简单的描述，它主要是用于TCP连接建立之后的请求渲染和请求生成的，基本的流程就是在TCP连接管道上进行数据的写入和读取。




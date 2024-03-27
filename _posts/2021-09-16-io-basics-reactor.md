---
layout: post
category: spring
date: 2020-06-28 17:26:03 UTC
title: 【IO基础】Reactor模型
tags: [Reactor、非阻塞的同步事件处理模型、Acceptor]
permalink: /io/basics/reactor
key:
description: 本文主要介绍Reactor模型的基础知识
keywords: [Reactor、非阻塞的同步事件处理模型、Acceptor]
---

# 1. What & Why

一种**非阻塞的同步事件处理模型**，只不过很多开源框架将其运行在网络编程中，像Kafka、Netty、Redis以及Nginx都使用了这种事件处理模型， 所以现在提起Reactor，一般会和网络编程关联起来。

另外需要注意的是这种模式其实和反应堆关系不大，**更合适的叫法应该叫Dispatch模式**，IO多路复用器监听I/O事件，然后根据事件分配给某个线程/进程。

> The Reactor design pattern **handles service requests that are delivered concurrently to an application** by one or more clients.
>
> The application can register **specific handlers for processing which are called by reactor** on specific events. Dispatching of event handlers is performed by an initiation dispatcher, which manages the registered event handlers.

## 1.1 事件模型(event-driven)

将服务端与客户端交互的流程抽象成各种事件:  **接收连接(Accept)**、**读取请求(Read)**、**写入响应(Write)**

## 1.2 分而治之(Divide and Conquer)

将IO事件监听、连接建立和具体处理进行拆分，各司其职，提高`扩展性`。

Kafka中通过下面提到的三个角色的抽象，并且Processor和RequestHandler都是可以调整线程数，可以进一步提升性能

## 1.3 多路复用器(multiplexier)

程序移植性(portability): 基本每个操作系统

[这篇论文中 Reactor An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events](http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)也提到相较于

**Thread Per Connection**的几个优点

+ 效率问题: 多线程会涉及到大量的上下文切换、同步等

> Efficiency: Threading may lead to poor performance due to context switching, synchronization, and data movement

这个在实时接入中应该是很明显，如果线程数配置太大，这快开销应该也会提升

+  Programming simplicity: Threading may require complex concurrency control schemes;

+  可移植性

Portability: Threading is not available on all OS platforms.

因为线程并不是所有语言中都有对应的概念，比如说标准C语言中就没有线程概念

# 2. 三个角色和三类事件

## 2.1 三类事件

+ 连接事件

+ 读取事件

+ 写入事件

## 2.2 三个角色

+ Reactor: 主要负责监听IO事件 -- 这个过程是阻塞的，只不过后续的事件处理是非阻塞的

+ Acceptor: 主要负责建立连接

+ Handler: 真正进行IO事件的处理

# 3. 三种模式

后面的进程主要针对C来讲的，C语言实现的是`单Reactor单进程`的方案，因为C语编写完的程序，运行后就是一个独立的进程，不需要在进程中再创建线程。

下面的讲解主要结合Java NIO中相关的概念，SocketChannel、Selector等

## 3.1 单Reactor单线程/进程

+ IO事件监听(Reactor) --- 连接建立(Acceptor) --- 具体的IO事件处理(Hanlder)都在一个线程中完成

这个典型就是Redis在多线程版本之前的实现


## 3.2 单Reactor多线程/进程

第一种方式的问题很明显，由于都在一个线程上处理，一旦出现IO操作长时间阻塞后续请求将无法正常响应，因此当前模式扩展了Handler，利用线程池的方式维护多个Handler。

Kafka中使用`KafkaRequestHandlerPool`来抽象Handler池，`KafkaRequestHandler`来抽象Handler

## 3.3 主从Reactor多线程/进程

单Reactor(单Dispatcher)是一个Selector(多路复用器)监听了所有的Channel中的IO事件，为了分担压力避免大量IO事件需要分发给而造成瓶颈，我们可以将IO事件的监听进行拆分。

因为TCP中的字节流，肯定不能直接交给上层应用，需要按照指定的协议解析，如果请求量很大，势必会成为当前Reactor的瓶颈。

+ **主Reactor**: 主要负责监听Accept事件，然后交由Acceptor进行连接建立。Kafka中使用`kafka.network.Acceptor`来抽象，每一个Endpoint对应一个Acceptor，建立连接之后，将SocketChannel交由 SubReactor去监听IO读写事件。

```java
private[kafka] class Acceptor(
) {

  private val nioSelector = NSelector.open()
  
  while (true) {
  
  	def run(): Unit = {
        serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
     }
  }

}
```

+ **从Reactor**:  主要负责监听主Reactor建立了连接的Channel的IO事件，Kafka中使用`kafka.network.Processor`来抽象，具体就是每一个Processor内部维护一个Reactor，用于关注当前Channel上面的读写事件。

```java
private[kafka] class Processor() {
	
	private val selector = createSelector(..)
	
	while (true) {
		
	}
}
```

+ 工作线程池: 同上


# 4. 适用场景

一般在高性能、高并发的涉及到网络编程的开源框架中会使用

+ **服务器需要同时处理来自多个客户端的大量请求**，可以是客户端多、也可以是请求量大(A server application needs to handle concurrent service requests from multiple clients)

+ 当服务器需要在处理之前客户端请求的时候，同时**能继续接收新连接的请求**(A server application needs to be available for receiving requests from new clients even when handling older client requests)

+ 服务器需要最大的吞吐同时最大CPU性能, 并且不能阻塞，减小延时(A server must maximize throughput, minimize latency and use CPU efficiently without blocking)

# 5. 参考

\> [wiki Reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern)
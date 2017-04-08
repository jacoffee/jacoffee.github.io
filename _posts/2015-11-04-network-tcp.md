---
layout: post
category: network
date: 2015-11-04 20:51:22 UTC
title: 网络基础之TCP协议
tags: [TCP协议，三次握手，四次握手，流量控制，拥堵控制]
permalink: /network/tcp
key: 5fe47ecc064930a594384b516378be60
description: "本文阐述了TCP协议的定义，结构等基本问题"
keywords: [TCP协议，三次握手，四次握手，流量控制，拥堵控制]
---

<p style="display:none">
TCP流量控制
简介 TCP 和 UDP 区别，他们位于哪一层？
路由器和交换机的工作原理大概是什么，他们分别用到什么协议，位于哪一层？
描述TCP 协议三次握手，四次释放的过程。 
// 255 in Computer Networking || 3.5.5 Flow Control 250 Congestion 拥堵
TCP 协议是如何进行拥塞控制的？ || Http: The definitive Guide
为什么建立连接时是三次握手，两次行不行？如果第三次握手失败了怎么处理
关闭连接时，第四次握手失败怎么处理？
你怎么理解分层和协议？
HTTP 请求中的 GET 和 POST 的区别，Session 和 Cookie 的区别。
谈谈你对 HTTP 1.1，2.0 和 HTTPS 的理解。
</p>

TCP(**Transmission Control Protocol**)- 传输交换协议，位于网络模型的是传输层(见下面的示意图)。它一种可靠的基于IP Packet(报文)的交换协议，说它可靠是因为从源发出的字节流会以相同的顺序被接收。

![网络模型中TCP/IP所处的位置](/static/images/charts/2015-11-04/network_stack.png)

实际上HTTP连接从本质上来讲就是TCP连接，当客户端与服务器端进行沟通时需要先建立TCP连接，也就是一个通信的管道。建立TCP连接的过程就需要经典的三次握手。

![三次握手](/static/images/charts/2015-11-04/three_way_handshake.png)

在数据包交换的过程中，我们把数据包从客户端出发到达服务器端并且回来所花费的时间称之为往返时延(the round-trip time (RTT), which is the time it takes for a small packet to travel from client to server and then back to the client)。

三次握手的具体过程如下:

1. 首先客户端会发起一个建立TCP连接的请求，里面包含一个SYN的标识(SYN(synchronous)是握手信号)。
2. 如果服务器端接受了这个请求，那么它就会返回一个TCP Packet，里面包含SYN, ACK 标识，告知客户端，这个连接已被接受。
3. 客户端会把回执(ACK), 再次发送给服务端，告知服务器端，连接已经建立好了。
这就好像别人给你发来了请柬并且要求你发送回执一样。一般情况下，除了返回回执，这个过程实际上已经开始发送请求了。

**关于回执**，因为网络并不能保证可靠的数据包传递(<b style="color:red">网络路由, 可以在网络负载过高的时候任意的销毁它们，也就是我们常说丢包</b>)，所以TCP实现了自己的一套确认数据接收的机制。每一个TCP Segment都会有一个唯一的序列号和用于数据完整性的校验码(checksum)。每收到一个完好无缺的Segment，接收端都会返回一个回执给发送者以告知它数据已经接受了。<b style="color:red">如果发送者在指定时间内没有收到回复，则会认定数据丢失并重新发送</b>。 由于回执一般比较小，TCP会将这些回执放入一个缓冲中，等到有数据发出的时候一并发出。

**关于连接**，每一个TCP连接是都被以下四个属性唯一标识: **源IP地址 | 源端口 | 目标IP地址 | 目标端口号**。
一台服务器可以同时接受多个TCP连接，当客户端同时发送大量请求的时候，内核会给每一个请求分配一个**临时的端口号**，以确保每次都是建立的唯一的连接。每一次请求过来，服务器端会新建一个套接字(Socket)并且监听在80端口以处理这个请求。

为了提高HTTP传输数据的效率，HTTP连接可以复用之前的TCP连接。长连接一般通过设置请求头来完成,

```bash
Connection: Keep-Alive
Keep-Alive: timeout=120
```

**Keep-Alive**必须要在**Connection: Keep-Alive**请求头出现的时候才能出现，上述设置表示服务器如果在两分钟之类没有收到任何请求则会关闭TCP连接。

**关于信息传递**，当HTTP传递信息时，首先会转换成字节流，TCP接受到字节流数据之后会将字节流进一步拆分成Segments，
然后通过IP报文(IP Packets，IP Datagram)进行包裹然后传递。

![IP报文](/static/images/charts/2015-11-04/ip_packets.png)

**关于流量控制**，建立TCP连接的双方都会维护一个**接受数据的缓冲区**(Rev Buffer)，当TCP接受到完整并且按顺序到达的数据之后，就会将数据放入缓冲区等待其他应用进程来取。但是这个缓冲区的大小是有限制的。如果应用进程无法及时取走，就可能导致缓冲区溢出，所有TCP需要建立一套机制来避免这个问题，也就是TCP的流量控制(TCP
Flow Control)。

**(1)** 双发都会维持一个**Receive Window**(用于告知发送者，接受者的缓冲区还有多少空间)

![数据接收缓冲区](/static/images/charts/2015-11-04/recv_buffer.png)

**(2)** 接受方在接受数据的时候，会记两个变量**LastByteRev**(最近从网络中传输过来并且放入缓冲区的字节数), **LastByteRead**(最近一次缓冲区中读取的字节数)，它们之间应该满足如下关系:

```bash
#rwnd receive window size, 初始大小为BufferSize
LastByteRev - LastByteRead <= rwnd
```

所以，每一次接受到数据并且读取完成之后，接受方的rwnd应该为`BufferSize - (LastByteRev - LastByteRead)`，然后在回应的时候将该字段放入TCP Segment中，实际上就是上图(IP报文结构)中的Window那个属性。

**(3)** 相对应的，在发送方也需要维护两个变量**LastByteSent**(最近一次发出的字节数)和**LastByteACK**(接收方最近一次接受的字节数)，它们之间应该满足如下关系:`LastByteSent - LastByteACK <= rwnd`始终保证接收方未接受的字节数要小于rwnd。

**(4)** 有一个临界条件需要注意，就是当`rwnd=0`的时候，按照刚才的分析发送方接收到这个参数，认为接收方不能再接收数据，这时候就会停止发送数据。并且阻塞; 但接受方可能在发送``rwnd=0``之后就取走了缓冲区的数据并且此时也没有什么要返回给发送方。所以TCP规范中要求即使发送方收到了``rwnd=0``之后，也要**持续**给接收方发送一些试探性的数据(英文版中使用的是one data byte)，这样接受方会发出响应(此时缓冲区的数据也在被取走)，所以会计算新的``rwnd``并返回给发送方，这样又恢复了正常的传输。

最后推荐一个TCP调试工具，[Wireshark](http://www.cnblogs.com/TankXiao/archive/2012/10/10/2711777.html)，链接指向的博文对于Wireshark进行了非常详细的介绍，有兴趣可以阅读一下。其实对于http://roadtopro.cn/网络相关的东西都是比较抽象，如果借助一下工具将传输和数据图形化，这将会大大提升我们的学习效果。


## 参考

\> HTTP权威指南 第四章 TCP连接

\> 计算机网络自顶向下方法 第三章 传输层(讲的真好，这部书也非常不错)

\> [TCP/CP文档关于临时端口的解释](http://www.tcpipguide.com/free/t_TCPIPClientEphemeralPortsandClientServerApplicatio-2.htm)

\> [监听在80端口的Web服务器，如何同时响应众多请求](http://stackoverflow.com/questions/16952625/how-can-a-web-server-handle-multiple-users-incoming-requests-at-a-time-on-a-sin)
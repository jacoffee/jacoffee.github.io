---
layout: post
category: grpc
date: 2021-05-17 10:51:22 UTC
title: 【gPRC】负载均衡实现概述
tags: [gRPC、服务端负载均衡、客户端负载均衡]
permalink: /grpc/load-balance
key: 
description: 本文主要阐述了负载均衡中常用的实现方式以及gRPC的可用的选择
keywords: [gRPC、服务端负载均衡、客户端负载均衡]
---

# 1. 实现方法

## 1.1 代理(proxy model) -- 服务端负载均衡

在一些语境中，这种模式我们一般默认为**服务端负载均衡**。代理模式会将请求转发到后端的服务器，唯一的不足是需要对于**RPC请求和响应进行临时缓存**，这对于存储性的RPC服务是非常低效的，而且这种复制会增加响应的延时。

## 1.2 客户端模式(balancing-aware client)

这种又分为胖客户端(thick client)和lookaside负载

### 1.2.1 胖客户端(thick client)

这种将可用的服务端地址的选择放在了客户端去做，客户端在获取了一系列可用连接之后可用各种策略(round robin, hash)等方式进行选择，并且客户端需要**维护可用服务器列表的状态**、负载以及健康状况，通常会整合服务发现、域名解析和Quota分配等。比如说，Eureka就是属于这种模式。由于集成了太多的功能因此也成为胖客户端。

### 1.2.2 Lookaside balancing(external balancing)

这种方式将胖客户端中一些**功能和责任**(服务发现、域名解析和Quota分配)移除，使用外部组件来实现，这也就是**gPRC采用的策略(唯一内置的是grpclb)**。gPRC client只是做简单的获取可用服务端地址以及根据配置的负均衡算法去选择要往哪个服务器去发送请求。

## 1.3  服务端负载均衡 vs 客户端负载均衡

| 选项 | 服务端负载均衡                 | 客户端负载均衡                                               |
| ---- | ------------------------------ | ------------------------------------------------------------ |
| 优点 | 实现简单、对客户端透明   | 减少代理过程的损耗                                           |
| 不足 | 可能会增加延迟，甚至影响扩展性 | 实现复杂，需要自己定义负载逻辑、维护活跃节点，还有多语言维护成本 |

#  2. 负载的网络模型层次选择

负载均衡既可以基于应用层(HTTP)，也可以基于传输层(TCP)。

+ 基于传输层的负载均衡，会首先断开当前TCP连接然后在新开连接。一般内部服务，基于RPC调用的话，会选择这种
+ 基于应用层的负载均衡，会直接断开HTTP连接，然后再次选择服务器。一般Web对外的服务会选择这种

# 3. gRPC中的负载均衡支持

首先，gRPC中的负载均衡主要**外部负载模式(客户端负载的第二种形式)**，也就是外部服务负责可用的服务列表，然后客户端只需要实现简单的负载均衡算法。

假设我们使用Zookeeper来维护服务端地址(或者是外部负载均衡器Eureka的地址)，官网提供了实现的[示例代码](https://github.com/makdharma/grpc-zookeeper-lb)。下面主要总结下核心流程:

1. 服务端启动的时候，向指定ZNode注册自己
2. 客户端启动的时候会发送可用服务解析的请求，获取一系列服务端的地址(域名、端口)
3. 然后和每一个服务器建立连接，在gRPC中抽象成为subchannel
4. 客户端发起调用，基于内部的负载均衡算法(round robin)选择一个subchannel发送请求

![grpc-zookeeper](/static/images/charts/2021-05-17/grpc-zookeeper.png)

# 4. 参考

\> [gRPC官方博客讲解负载均衡](https://grpc.io/blog/grpc-load-balancing/)

\> [github gRPC load balancing](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)



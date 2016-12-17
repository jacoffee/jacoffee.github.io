---
layout: post
category: spark
date: 2016-12-15 09:38:55 UTC
title: Spark基础之广播(broadcast)
tags: [BitTorrent，Netty通讯，端对端通讯，BlockManager，基于消息传递的模型]
permalink: /spark/broadcast
key: 31bcb859f360cab0c42e6876ecaf6c2c
description: 本文阐述了broadcast的整个过程
keywords: [BitTorrent，Netty通讯，端对端通讯，BlockManager，基于消息传递的模型]
---
 
之前在实现一个给executor使用的线程池时，直接将它broadcast了，这种用法被同事质疑了，他认为应该在executor上初始化线程池实现类。如果从broadcast方法的定义上来讲，这种质疑是合理的，因为它的初衷是将只读的变量广播到各个节点(driver or executor)上(**Broadcast a read-only variable to the cluster**)，比如说通过数据库查询获取某种映射关系然后广播到各个节点上以减少重复查询。

加之在查看YARN Cluster Manager中的日志时，经常会看到`TorrentBroadcast`，甚是疑惑广播为什么会与Torrent产生关联，于是决定通过阅读源码来理清broadcast的实现。

broadcast的主要实现在`org.apache.spark.broadcast.TorrentBroadcast`中，之前一直以为broadcast的实现机制是在driver上将对象通过netty通讯传递到不同的节点上。但这种实现有一个问题，一旦broadcast变量比较大，并且节点比较多的时候，driver就容易成为瓶颈。

所以，Spark借鉴了BitTorrent的实现，大致流程如下图:

![broadcast流程](http://static.zybuluo.com/jacoffee/xoitcdzj93rrsfop47n2qx5z/image_1b461do6cr071liuj96p398r2p.png)

<b class="highlight">(1) TorrentBroadcast的初始化</b>

在SparkEnv初始化的时候，BroadcastManager也完成初始化并且创建了新的`TorrentBroadcastFactory`，然后通过`newBroadcast`方法创建`TorrentBroadcast`。由于每一个broadcast变量需要先存储然后再获取，那么就需要使用某种标识，也就是我们在日志中看到的**broadcast_0**，**broadcast_0_piece0**。

```scala
val mapBC = sc.broadcast(Map("a" -> 1))
// broadcastManager在SparkEnv初始化的过程中被初始化
broadcastManager.newBroadcast[T](value, isLocal)
    // uniqueBroadcastId -- nextBroadcastId.getAndIncrement()
    broadcastFactory.newBroadcast[T](value_, isLocal, uniqueBroadcastId)
    
mapBC.getValue()
```

<b class="highlight">(2) 广播对象被切分成小块(chunk)并通过BlockManager进行存储</b>

比如说刚开始在driver上进行广播操作:

首先会将对象直接以**MEMORY_AND_DISK**的storage level放到BlockManager中，方便之后的本地获取(不用反序列化，直接使用)，唯一ID为`broadcast_broadcastId`的格式。

然后按照`spark.broadcast.blockSize`指定的大小(默认是4m)将对象切分形成`Array[ByteBuffer]`，以`MEMORY_AND_DISK_SER`的storage level存储到BlockManager中，这样别的节点可以通过Netty通讯获取这些"碎片"，唯一ID为`broadcast_broadcastId_field_Index`的格式。

上面两种操作在日志中的反映就是:

```bash
INFO MemoryStore: Block broadcast_0 stored as values in memory (estimated size 5.0 MB, free 5.0 MB)
INFO MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 436.9 KB, free 5.4 MB)
```

<b style="color:red">实际上切分和存储的过程在TorrentBroadcast初始化的过程中就已经开始了。</b>

```scala
new TorrentBroadcast {
    // 获取拆分的block数目
    private val numBlocks: Int = writeBlocks(obj)
    
     // 触发变量拆分和block存储过程
     writeBlocks(obj) {
        if (!blockManager.putSingle(broadcastId, value, MEMORY_AND_DISK)) {
          ...
        }
     }
     
     val blocks: Array[ByteBuffer] = TorrentBroadcast.blockifyObject(...)
      blocks.zipWithIndex.foreach { case (block, i) =>
         ...
         blockManager.putBytes(pieceId, bytes, MEMORY_AND_DISK_SER ...)
         ...
      }
}
```

存储完成之后，会将相关的信息汇报给driver中的`BlockManagerMasterEndpoint`，它汇总集群中各个BlockManagerId中的BlockManager的情况, 也就是哪些executor上有哪些block。

```scala
Executor BlockManager ===> BlockManagerMaster ===> BlockManagerMasterEndpoint
```

关于上面的这种通讯模式在Spark其它地方也用到了，driver上的某个功能模块会有一个Endpoint和executor上对应功能模块的Endpoint进行沟通，大多数会通过NettyRPC。

<b class="highlight">(3) 通过TorrentBroadcast的getValue触发广播变量读取</b>

比如说现在在executorA上需要获取上面的广播变量，则需要调用反序列化之后的TorrentBroadcast的`getValue`方法，进而触发`readBlocks()`方法。

它首先通过`broadcast_broadcastId`去本地的BlockManager中获取，因为之前有可能已经获取过了。

如果没有的话，获取之前拆分的block数`numBlocks`，按照指定规则拼接成`pieceId`(broadcast_broadcastId_field_Index)；同样先从本地获取，然后从远端获取。
从远端获取，实际上就是给`BlockManagerMaster`发送获取Block位置的请求，<b style="color:red">但可能返回多个可供选择的位置</b>，遍历进行获取，一旦成功则返回。

成功获取之后，就会存储在自己的BlockManager中，这样避免下一次重复获取。随着时间的推移，每一个Block可供获取的位置越来越多，直至最后成功的『下载』完整个广播变量，颇有些BitTorrent的感觉。

```scala
broadcast.getValue()
    broadcast.readBroadcastBlock()
        blockManager.getLocalValues(broadcastId)
            Some(x) -> x
            None -> readBlocks()
                bm.getLocalBytes(pieceId) {
                    Some(block) -> block
                    None ->  bm.getRemoteBytes(pieceId) -> put inn blockManager
                }
                val obj = TorrentBroadcast.unBlockifyObject()
                obj
```     
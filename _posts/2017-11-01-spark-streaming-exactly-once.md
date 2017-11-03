---
layout: post
category: spark
date: 2017-10-26 06:30:55 UTC
title: Spark Streaming中关于exactly-once语义的思考
tags: [端对端一致性、状态维护、Checkpoint、幂等性]
permalink: /spark/streaming-exactly-once
key: 
description: 本文梳理了我对于Spark Streaming exactly-once语义的理解
keywords: [端对端一致性、状态维护、Checkpoint、幂等性]
---

在[Spark Streaming概述](/spark/streaming-basics)中主要介绍了Streaming的基本概念以及DirectKafkaInputDStream的执行流程。在流式计算中，还有一个比较重要的概念就是端对端exactly-once语义，在Spark Streaming中也就是说需要确保消息只被消费一次，并且只对外输出一次。

在DirectKafkaInputDStream中，我们需要考虑如下问题:

<ul class="item">
  <li>如何高效稳定的记录Kafka消费的偏置</li>
  <li>消息处理之后，保证输出的幂等性(idempotence)，即使重复操作数据状态也不会有问题</li>
</ul>

## 1.消费一致性

### 1.1 存储的数据

因为KafkaStream计算的时候才会去从Kafka中进行消费，所以Checkpoint的结构只需要保存offset即可，并且该数据结构需要能不断更新，在DirectKafkaInputDStream中，该数据结构为可变的HashMap。

```scala
class DirectKafkaInputDStreamCheckpointData {
    ...
    def batchForTime: mutable.HashMap[Time, Array[(String, Int, Long, Long)]] = {
      data.asInstanceOf[mutable.HashMap[Time, Array[OffsetRange.OffsetRangeTuple]]]
    }
    ...
}
```

OffsetRangeTuple维护的是某个话题某个分区的消费起始Offset和结束Offset。

### 1.2 Checkpoint的时机

如果需要开启Checkpoint机制，首先需要在设置相应的Checkpoint dir。

```bash
ssc.checkpoint(checkpointDirectory)
```

<ul class="item">
  <li>JobGenerator每次生成Jobs的时候，会checkpoint更新当前批次对应的Offset信息</li>
  <li>当前批次结束的时候，再次checkpoint清除元数据</li>
</ul>

通过存储Offset的机制保证了即使任务执行失败，当前batch也可以再次执行。

### 1.3 恢复的时机

在StremingContext初始化的时候，也会同时初始化StreamingGraph，这个时候就会尝试去从checkpoint中去恢复数据

```bash
cp_.graph.restoreCheckpointData()
```

## 2.输出幂等

上面提到过，如果Offset被checkpoint之后，任务执行失败了，由于Spark是分partition计算的，可能一部分数据已经落入存储了，那么batch重新计算的时候，就可能会产生重复数据，所以需要我们自己手动控制。

+ ElasticSearch

在ES中，如果ID相同，则会直接更新，借助这一特性，我们可以为每一条记录分配**唯一的ID**，这样即使batch重算，也不会出现重复数据。

+ 数据库

由于我们在批次中checkpoint了相关的信息，比如说批次时间，批次对应的offset等，所以在重算或者恢复的时候，这些信息依然可以获得，结合partitionId便可以构成当前批次的唯一标识。

```bash
dstream.foreachRDD { (rdd, time) =>
  rdd.foreachPartition { partitionIterator =>
    val partitionId = TaskContext.get.partitionId()
    val uniqueId = generateUniqueId(time.milliseconds, partitionId)
    // 带事务的操作，如果uniqueId对应的记录不存在，则插入，反之则放弃当次操作
    insertIfNotExists(batchRecords)
  }
}
```

最后的话，对于Spark Streaming宣称的exactly-once语义，从实现上来看实际上是不满足的，
首先消息会重复消费，其次消息也是会被重复处理的。所以它最多能保证at-least once语义。在分布式系统中，exactly-once delivery是一大难题，但是在[Kafka 0.11版本](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)中实现了该机制并且很好的融入到Kafka Stream中去了。


## 参考

\> [Spark Streaming消费Kafka数据的两种方案](https://mp.weixin.qq.com/s/bjlDHFLwxjej2t8iDhVb1A)

\> [You can not have exactly once delivery](http://bravenewgeek.com/you-cannot-have-exactly-once-delivery/)
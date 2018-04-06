---
layout: post
category: kafka
date: 2017-11-15 06:30:55 UTC
title: Kafka消息发送剖析
tags: [Partition，重试，回执，异步，Availability]
permalink: /kafka/producer-order-study
key: 
description: 本文对Kafka消息发送过程进行了研究以及如何最大程度的保证可靠送达
keywords: [Partition，重试，回执，异步，Availability]
---

对于消息队列来说，首先需要确保的是消息的稳定送达，Kafka(0.9.0)当然也不例外。本文将从两个方面展开: Kafka消息发送过程研究以及它是如何确保消息的可靠送达的。

## 1. 消息发送

当我们向消息队列中发送消息的时候，如下几点是我们需要考虑的:

<ul class="item">
    <li>发送的过程中，消息丢失和重复影响大嘛？</li>
    <li>对于延时和吞吐有怎样的要求？</li>
    <li>当发送失败时的重试机制是怎样的？</li>
</ul>

而不同的场景，可能会有不同的要求。这样我们在构建Kafka Producer的时候就会有不同的配置，典型的比如说ack的级别、retry次数等(对于吞吐的都会有影响)。

下图展示的是Kafka Producer发送消息的基本过程:

![Kafka消息发送过程](http://static.zybuluo.com/jacoffee/9uo57mfbanwug2rt3d2tzi7d/image.png)

**1.构建ProducerRecord**:  一般会指定topic和value，即向哪个topic发送什么消息


**2. 序列化消息**: Producer会将相应的消息序列化成字节数组(byte array)以便网络传输


**3. 消息分区选择**: 接下来，我们需要根据消息发送时是否指定分区，来决定消息的去向，即路由到什么地方。如果指定了分区，则进行校验。反之，则需要借助Partitioner根据相应的规则去分发到不同的分区: 指定了相应的Key，则按照key bytes的哈希值取模来决定分区；否则根据是否有可用的分区来选择，如果有则按可用分区随机选择一个; 没有的话，直接按照存储的nextvalue随机选择


```java
org.apache.kafka.clients.producer.internals.DefaultPartitioner

private final AtomicInteger counter = new AtomicInteger(new Random().nextInt());

if (keyBytes == null) {
   int nextValue = counter.getAndIncrement();
   ....
   if (availablePartitions.size() > 0) {
        return availablePartitions.get(part).partition();
    } else {
        return DefaultPartitioner.toPositive(nextValue) % numPartitions;
    }
} else {
    DefaultPartitioner.toPositive(Utils.murmur2(keybytes)) % numPartitions;
}
```


**4. 消息发送**: Kafka消息发送一般是异步的，所以在设计上将消息的聚合和发送分成了两部分。分别对应
**org.apache.kafka.clients.producer.internals.RecordAccumulator** 和 **org.apache.kafka.clients.producer.internals.Sender**，前者负责将消息放入某个内存缓冲区(批次消息)，而Sender(Runnable)会在<b style="color:red">后台线程</b>中不停运行将每个批次的消息发送到Kafka集群中，实际上就是处理accumulator中累计的消息，消息发送的重试也是在Sender中实现的(**ProducerConfig.RETRIES_CONFIG**)


```java
org.apache.kafka.clients.producer.KafkaProducer

// 守护线程
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
this.ioThread.start();
```

**5. 消息发送响应**: 当broker接受到消息之后，会产生相应的响应，由于一般是异步发送，所以我们会注册回调来处理响应，它会在消息被broker成功接收之后执行(acknowledged)。


```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    ....
}
```

## 2.送达保证

Kafka是可以保证单分区消息的顺序性的，但这个实际上是需要在特定的配置下才能实现的。

首先，我们来认识一下与此相关的两个Producer参数

**1. retries(消息发送重试) -- 默认值为0**

上面流程我们提到过当broker返回错误信息时，如果我们设置了重试并且是可重试错误(**RetriableException**)，则Producer会再次尝试发送相应的RecordBatch。如果我们将其设置为大于0的话，就有可能导致单分区消息的顺序无法保障，比如说前一个消息因为发送失败而重试，后面的消息一次性成功了，这样就会导致消息的位置发生改变


```java
// org.apache.kafka.clients.producer.internals.Sender
private boolean canRetry(RecordBatch batch, Errors error) {
    return batch.attempts < this.retries && error.exception() instanceof RetriableException
}
```

**2. max.in.flight.requests.per.connection -- 默认值为5**

在单个连接(**NetworkClient**)中，同时最多能有多少个没有收到响应的发送请求在执行。

按照默认的设置，是不会有问题的，因为默认情况下，如果消息发送失败了是不会重试的。
但是如果我们希望通过重试来减少数据丢失(通过多次发送)，这个时候就需要限制同时发出的请求，即设置为1。但这种情况吞吐量会下降，所以有一定的场景要求。

```bash
retries=3
max.in.flight.requests.per.connection=1
```

实际上这一切的配置都是为了更可靠的发送(做到不丢失和不重复)，而在[Kafka 0.11](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)已经实现了exactly-once的语义，简单概括就是需要引入事务来实现，本人将会在后续的文章中对于这一实现进行剖析。

## 参考

\> [KAFKA-3197 Fix producer sending records out of order](https://github.com/apache/kafka/pull/857/files)

\> [Kafka的生产者适当配置情况下 支持单分区消息的发送顺序](https://stackoverflow.com/questions/36692113/does-kafka-guarantee-message-ordering-within-a-single-partition-with-any-config)

\> [inflightRequestPerConnection和 acks对于生产者性能的影响分析](https://cwiki.apache.org/confluence/display/KAFKA/An+analysis+of+the+impact+of+max.in.flight.requests.per.connection+and+acks+on+Producer+performance)

\> [只发送一次以及事务性消息](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)

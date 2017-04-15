---
layout: post
category: spark
date: 2016-09-05 15:51:22 UTC
title: Spark基础之Checkpoint与Cache
tags: [BlockManager, Block, getOrCompute]
permalink: /spark/checkpoint-cache
key: 
description: "本文主要介绍了Checkpoint与Cache的区别以及基本实现"
keywords: [BlockManager, Block, getOrCompute]
---

在RDD的操作中，我们会用到Checkpoint和Cache两种操作，它们主要存在如下差异:

<table>
    <tr>
        <th width="100">维度\操作</th>
        <td align="center"><b>Cache</b></td>
        <td align="center"><b>Checkpoint</b></td>
    </tr>
    <tr>
        <td>存储地点</td>
        <td>一般是内存中，当然也可以使用磁盘</td>
        <td>磁盘</td>
    </tr>
    <tr>
        <td>作用时间</td>
        <td>每计算一个RDD分区就会缓存一次</td>
        <td>当RDD计算结束之后会再次运行任务将每个分区数据写入外部存储中</td>
    </tr>
    <tr>
        <td>使用场景</td>
        <td>
            <p>
                通过缓存分区来减少重复计算; 
                典型的应用就是对于会被重复计算的RDD进行缓存，当一个RDD需要同时被写入HDFS和数据库中时，就可以先缓存该RDD然后再进行后续操作。
            </p>
        </td>
        <td>
            <p>
            减少依赖维护难度，不同Job之间共享数据; 典型的应用就是在Streaming中，会定期将流相关的状态检出，一方面在出现故障时可以读取之间的状态信息用于恢复，另一方面也可以简化RDD的依赖链
            </p>
        </td>
    </tr>
    <tr>
        <td>后续影响</td>
        <td>原RDD的所有依赖和分区仍然被保留，如果计算失败可以再次进行计算</td>
        <td>原RDD的所有依赖和分区都被清除</td>
    </tr>
</table>

下面我们来了解一下上述两个操作的大致流程:

##缓存(Cache)

RDD缓存就是将分区的计算结果存放在内存或是磁盘中，下次需要计算同样分区的时候直接通过**BlockManager**去获取，大致分为如下几个流程:

<ul class="item">
    <li>
每缓存一个RDD(<b>rdd.persist(level)</b>)，都会向内存映射<b>Map[RDDID, RDD]</b>中添加相应的记录用于后续信息统计
    </li>
</ul>

```scala
// org.apache.spark.rdd.RDD
def persist(rdd: RDD[T]) {
    // persistRDDs.update(rdd.id, rdd) = 
    persistRDDs(rdd.id) = rdd
}
```

<ul class="item">
    <li>
当RDD分区计算被触发时，会先尝试从BlockManager中去获取如果不存在，则<b>进行计算然后将结果放入BlockManager中</b>，这也是真正缓存的时间点    
    </li>
</ul>

```scala
// org.apache.spark.rdd.RDD
getOrCompute
    SparkEnv.get.blockManager.getOrElseUpdate
        // 如果存在直接获取结果
        get(blockId) match {
          case Some(block) =>
            // Scala中很罕见的return使用啊
            return Left(block)
          case _ =>
            // Need to compute the block.
        }
        // 不存在，则进行计算再存储
        doPutIterator(blockId, makeIterator, ....)
```

##检出(Checkpoint)

在RDD计算结束之后会将所有分区依次写入到可靠的外部存储中(reliable system, such as HDFS)，一般是由于RDD之间的依赖链太长或是需要在不同的Job之间共享。

```scala
import org.apache.spark.rdd.RDD

private[spark] var checkPointData: Option[RDDCheckpointData[T]] = None
// RDD调用该方法的时候，给checkPointData赋值
def checkpoint() {
    ...
    checkPointData = Some(new ReliableRDDCheckpointData(this))
    ...
}

// Job执行完成之后，开始对checkPointData进行处理
def runJob[T, U: ClassTag](...): Unit = {
    dagScheduler.runJob()
    ...
    rdd.doCheckpoint()
}
```

在1.6中，<b class="highlight">Checkpoint操作会导致两次RDD计算，一次是通过runJob计算RDD，一次是计算结束后运行另外一个Job将该RDD的内容写到HDFS中</b>。具体可以参考[SPARK-8582](https://issues.apache.org/jira/browse/SPARK-8582)。所以使用时建议**先将该RDD缓存到内存中，然后再checkpoint**。

关于重复计算的问题**org.apache.spark.rdd.ReliableCheckpointRDD**的writeRDDToCheckpointDirectory方法中也有提到:

```scala
// org.apache.spark.rdd.ReliableCheckpointRDD
def writeRDDToCheckpointDirectory
    // TODO: This is expensive because it computes the RDD again unnecessarily (SPARK-8582)
    sc.runJob(originalRDD,
        writePartitionToCheckpointFile[T](checkpointDirPath.toString, broadcastedConf) _)
```

Checkpoint大致可以分为如下三个流程:

<ul class="item">
    <li><b>初始化阶段: </b>当rdd调用checkpoint的时候，会生成一个实现checkpoint的类ReliableRDDCheckpointData，然后建立相应的checkpoint path，并设置checkpoint的状态为<code>Initialized</code>。
    </li>
    <li>
        <b>检出阶段: </b> 当rdd.runJob执行完成之后，会执行rdd的doCheckpoint，此时checkpoint的状态变为<code>CheckpointingInProgress</code>，也就是正在checkpoint中
    </li>
    <li>
        <b>检出完成: </b> 通过ReliableCheckpointRDD的方法writeRDDToCheckpointDirectory，将原RDD中的数据写入磁盘，然后将checkpoint状态改<code>为Checkpointed</code>，调用原RDD的markCheckpointed方法清除原RDD的所有依赖以及分区
    </li>
</ul>

而关于checkpointed RDD的获取，在每次计算的时候都会先尝试去获取，具体的逻辑可以参考**org.apache.spark.rdd.RDD.iterator**方法。


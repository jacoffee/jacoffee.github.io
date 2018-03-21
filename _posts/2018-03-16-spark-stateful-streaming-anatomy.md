---
layout: post
category: spark
date: 2018-03-16 01:29:03 UTC
title: Spark Stateful Streaming概述
tags: []
permalink: /spark/stateful-streaming
key:
description: 本文研究Spark Stateful Streaming的基本使用，背后的实现以及容错过程中需要考虑的问题
keywords: []
---

Spark Stateful Streaming的出现就是为了将不同的批次之间的状态关联起来，就像cookie和session将独立Http请求之间的状态关联起来，关于Spark Streaming的基础使用，可参照[Spark基础之Spark Streaming概述](http://roadtopro.cn/spark/streaming-basics)，本文主要讲解mapWithState方法的使用、背后的实现以及容错过程中的一些思考。

下面以统计Kafka消息中词出现的次数来讲解整个过程，Kafka生产者不停的向Producer发送消息，Streaming定时获取并统计每个词的出现次数，如果发现出现次数高于多少则移除。

## 1.Kafka消息发送(kafka-console-producer)

```bash
/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic kafka-word-count
AAAA
BBBB
AAAA
....
```

## 2.mapWithState方法涉及的类

首先它是PairDStreamFunctions[K, V]的一个方法，也就是说原始流一定要是**DStream[K, V]**的类型，才能被隐式转换成PairDStreamFunctions，它的主要目的是对于流中的每一个Key, Value对调用相关的函数，同时<b style="color:red">维护流的状态以及生成相应的返回值</b>，这样我们在调用mapWithState之后可以根据需要得到不同的返回值。

```scala
class PairDStreamFunctions[K, V](self: DStream[(K, V)])  {
    ....
}

def mapWithState[StateType: ClassTag, MappedType: ClassTag](
    spec: StateSpec[K, V, StateType, MappedType]
): MapWithStateDStream[K, V, StateType, MappedType] = { 
    ....
}
```

**K**: 转换流的Key类型，在本例中是String，也就是Kafka中的消息
**V**: 转换流的Value类型，在本例中是Int，单个单词出现次数
**StateType**: 状态类型，在本例中是Int，也就是单词出现次数汇总
**MappedType**: 对于转换流，在本例中是(String, Int)，也就是**(单词次数，出现次数汇总)**

之所以叫转换流，是因为**DirectKafkaInputDStream[K, V]**中的K对应是Kafka消息中的key，一般为null；value对应的是Kafka消息中的value，一般是String。而这个消息中可能包含了丰富的内容，抑或是日志的一行，抑或是json string，所以一般都会经过简单处理形成新的**DStream[K, V]**(转换流)之后才会调用mapWithState方法的。

```scala
abstract class StateSpec[KeyType, ValueType, StateType, MappedType] extends Serializable {
    ...
    def initialState(rdd: RDD[(KeyType, StateType)]): this.type
    def numPartitions(numPartitions: Int): this.type
    def timeout(idleDuration: Duration): this.type
    ...    
}
```

StateSpec抽象了调用mapWithState的相关参数:

+ 是否使用初始状态
+ 调用函数之后，会生成相应的StateRDD，numPartitions设置的分区数(默认是HashPartitioner)就是对于该RDD进行的。
+ 由于State以RDD形式存在，一般在初始化流的时候都会设置StorageLevel，比如说**StorageLevel.MEMORY_ONLY_SER**，这样随着流式程序的运行，内存会越来越大，因此Spark提供一种Key过期的机制，也就如果State中的key在多长时间内没有更新的话，就会自动被移除。

```scala
private[streaming] abstract class StateMap[K, S] extends Serializable {
    ...
    def get(key: K): Option[S]
    
    def getByTime(threshUpdatedTime: Long): Iterator[(K, S, Long)]
    
    def remove(key: K): Unit
    ...
}
```

Stateful streaming的状态维护的基本数据结构就是上面的StateMap，源码中的注释也体现了这一点:
**Internal interface for defining the map that keeps track of sessions**。

+ get方法用于获取状态，而签名中的S实际上就是上面的**State**
+ getByTime获取比该时间还早的key和以及状态，目的就是了删除过期Key的
+ remove则提供了Key删除机制

## 3.基本使用

经过上面的梳理，基本上对于mapWithState涉及的相关的类有一个了解，下面通过相应的代码来实现上文提到的word count，对于过期我们选择手动清理出现次数大于10次的。

```scala
// StateSpec.mappintFunction
private def mappingFunction(
    time: Time, key: String, valueOpt: Option[Int], wordCountState: State[Int]
): Option[(String, Int)] = {
    val thresholdCount = 10
    // key在StateMap中有值, 相当于调用StateMap.getOption(keyType).nonEmpty
    // 首先判断该key对应的次数, 如果大于10, 则移除
    if (wordCountState.exists) {
      // StateMap.get(keyType)
      val existingCount = wordCountState.get
    
      // 出现次数超过10次, 则移除
      if (existingCount > thresholdCount) {
         wordCountState.remove()
         None
      } else {
        val newCount = valueOpt.getOrElse(1) + existingCount
        // 更新状态
        wordCountState.update(newCount)
        Some(key -> newCount)
      }
    } else {
      // 首次出现, 则计1
      val newCount = valueOpt.getOrElse(1)
      wordCountState.update(newCount)
      Some(key -> newCount)
    }
}
```

从控制台的来看，会发现某个单词，突然在下一次批次消失，比如下面的BBBBBBB:

```bash
Word AAAAAAAA, count 1
Word BBBBBBBBBBBBBB, count 2
Word BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB, count 1
Word BBBBBBBBBBBBBBBBBBBBBBBBBBBB, count 1
Word BBBBBBB, count 11

--------------------------- 

Word AAAAAAAA, count 1
Word BBBBBBBBBBBBBB, count 2
Word BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB, count 1
Word BBBBBBBBBBBBBBBBBBBBBBBBBBBB, count 1
```

关于本文中的案例，详见[]()

## 4.状态更新

通过mapWithStream产生的流，它的状态更新主要是在**MapWithStateRDD**的compute方法中实现，核心实现为**MapWithStateRDD.updateRecordWithData**

```scala
case class MapWithStateRDDRecord[K, S, E](var stateMap: StateMap[K, S], var mappedData: Seq[E])

object MapWithStateRDDRecord {

    def updateRecordWithData(
        prevRecord: Option[MapWithStateRDDRecord[K, S, E]],
        dataIterator: Iterator[(K, V)],
        mappingFunction: (Time, K, Option[V], State[S]) => Option[E],
        batchTime: Time,
        timeoutThresholdTime: Option[Long],
        removeTimedoutData: Boolean
    ): MapWithStateRDDRecord[K, S, E] =  {
    
        // 通过克隆之前StateMap来创建新的map
        val newStateMap = prevRecord.map { _.stateMap.copy() }. getOrElse { new EmptyStateMap[K, S]() }
        
        val mappedData = new ArrayBuffer[E]
        
        dataIterator.foreach { case (key, value) =>
        
            // mappingFunction真正的调用处
            val returned = mappingFunction(batchTime, key, Some(value), wrappedState)      
            
        }
        
        // 超时记录处理
        if (removeTimedoutData && timeoutThresholdTime.isDefined) { 
            ....
        }
        
        MapWithStateRDDRecord(newStateMap, mappedData)
    }

}
```

+ MapWithStateRDDRecord实际上是和MapWithStateRDDPartition一一对应的，也就是一个MapWithStateRDDRecord实例就对应了一个分区的数据。这样设计的原因是因为，一般状态涉及到**聚合操作**，所以会对于某一个Key的状态进行加总，比如说本例中Word出现次数，因此无法设计成一个Key-Value对应一个
MapWithStateRDDRecord实例

+ 在上面的更新程序中，我们可以看到当前批次产生的分区即dataIterator，直接对于prevRecord中的key对应的状态进行了更新，那么Spark是如何保证它们俩中的key是在同一个分区的呢？答案就在于: 确保**RDD[MapWithStateRDDRecord[K, S, E]]**(历史状态)和**RDD[(K, V)]**分区器(Partitioner)相同，这样就能保证相同Key落入相同的分区，给同分区更新提供了基础

```scala
class MapWithStateRDD[K: ClassTag, V: ClassTag, S: ClassTag, E: ClassTag]( 
    ...
    private var prevStateRDD: RDD[MapWithStateRDDRecord[K, S, E]],
    private var partitionedDataRDD: RDD[(K, V)],
    ...
) {

    require(partitionedDataRDD.partitioner == prevStateRDD.partitioner)

}
```

+ 每个分区更新完成之后返回值为MapWithStateRDDRecord，所以如果我们需要获取当前批次处理之后的结果，则获取的是每个分区mappedData的汇总; 如果要获取每个批次完成之后，流的历史状态(snapshots)，则获取的是每个分区stateMap的汇总，即Stream快照(snapshots)。

```scala
// RDD[MapppedType] 上例中对应 RDD[(String, Int)]
dstream.mapWithState.foreachRDD { rdd: RDD[MapppedType] => 

}

// RDD[(K, StateType)] 上例中对应 RDD[(String, Int)]
dstream.mapWithState.stateSnapshots().foreachRDD { rdd: RDD[(K, StateType)] =>

}
```

## 5.容错机制的一些思考以及实现

这里的容错机制主要是指: 流式程序的状态会不断积累，如何在程序宕机之后恢复到之前的状态，上例中就是如何保证之前WordCount的统计不丢失。

Spark(1.6)对于Stateful Streaming强制要求使用Checkpoint将流式状态(主要指的是底层RDD)，默认是**10倍的slideDuratio**n。但是目前这种实现有个问题:

```scala
class InternalMapWithStateDStream {
    // 默认的StorageLevel
    persist(StorageLevel.MEMORY_ONLY)
    
    override def initialize(time: Time): Unit = {
        ...
        checkpointDuration = slideDuration * DEFAULT_CHECKPOINT_DURATION_MULTIPLIER
        ...
    }
    ...
}
```

如果重新运行的时候，代码发生了改变，可能会导致相应的反序列化问题，所以之前的状态变无法恢复; 如果删除的话，之前的状态以及消息的消费也可能会丢失。这对于不断更新逻辑的项目而言是不可接受的。

所以我们可以尝试手动维护Kaka偏置，同时对于带状态的流，我们需要定期导出Snapshots作为流状态的初始值，具体实现需要考虑的一些问题:

<ul class="item">
    <li>
    <b>偏置存储介质的选择:  </b>
由于Kafka自身就依赖Zookeeper，再加上其不错的容灾能力，所以将偏置维护到Zookeeper就非常自然。由于apache curator(Zookeeper client library)提供了非常实用的方法，所以处理偏置的时候选择了这个框架。
    </li>
    <li>
    <b>存储偏置的时机:  </b> 
如果以对外提供接口的角度来看的话，我们希望存储偏置的这个过程对于<b>调用者是不可见的</b>，所以可以在创建DirectKafkaInputDStream的时候，注册一个foreachFunc去定期存储偏置，后面再注册我们的业务处理逻辑。 这个时候就需要注意存储时，到底是开始位置还是结束位置，答案是<b style="color:red">开始位置</b>，因为业务逻辑处理可能失败，这时候如果在存储偏置时直接存储了结束的偏置，那么重启的时候，就会丢失消息；存储开始位置，会导致消息重复，这时候可以通过输出时确保幂等性
    </li>
    <li>
    <b>存储流状态的时机以及频率:  </b>  在调用mapWithState之后，流的整体状态已经发生改变，所以在之后的foreachRDD操作中通过<b>rdd.saveAsObjectFile</b>进行存储，但考虑到如果太大存储也会耗时很久，所以需要控制频率，比如说每隔5倍的slideDuration存储一次，<b style="color:red">这里还是有状态丢失的风险，因为不是每个批次都存储了状态的</b>
    </li>
</ul>

```scala
class DirectKafkaInputDStream[K: ClassTag, V: ClassTag, KD <: Decoder[K]: ClassTag, VD <: Decoder[V]: ClassTag](
  ssc: StreamingContext, topic: String, kafkaParams: Map[String, String]
) extends Serializable {

  private val logger = LoggerFactory.getLogger(getClass.getName.stripSuffix("$"))

  // commit offset at every interval
  def build() = {
    require(kafkaParams.contains("group.id"), "Consumer group id should not be empty")
    require(kafkaParams.contains("zookeeper.connect"), "Zookeeper connect should not be empty")

    val consumerGroupId = kafkaParams.get("group.id").get
    val zkConnect = kafkaParams.get("zookeeper.connect").get

    val zkClient = ZookeeperClient.connect(zkConnect)
    val getFromOffsetsFunc: ZookeeperClient => Map[TopicAndPartition, Long] = _.getFromOffsets(consumerGroupId, topic)

    val directKafkaInputDStream =
      CommonUtils.safeRelease(zkClient)(getFromOffsetsFunc)() match {
        case Success(storedFromOffsets) =>
          // try to get from offset from zookeeper or create a new one
          val directKafkaInputDStream =
            if (storedFromOffsets.nonEmpty) {
              val messageHandler = (mmd: MessageAndMetadata[K, V]) => (mmd.key, mmd.message)
              KafkaUtils.createDirectStream[K, V, KD, VD, (K, V)](
                ssc, kafkaParams, storedFromOffsets, messageHandler
              )
            } else {
              logger.info(s"Create direct stream with topics: ${topic}")
              KafkaUtils.createDirectStream[K, V, KD, VD](ssc, kafkaParams, Set(topic))
            }

          directKafkaInputDStream
        case Failure(e) =>
          logger.error(s"Exception when creating DirectKafkaInputDStream", e)
          throw e
      }

    directKafkaInputDStream.foreachRDD { (rdd, time) =>

      val zkClient = ZookeeperClient.connect(zkConnect)
      CommonUtils.safeRelease(zkClient)(
        _.commitFromOffset(consumerGroupId, rdd.asInstanceOf[HasOffsetRanges].offsetRanges)
      )()
    }

    directKafkaInputDStream
  }

}
```

等到获取当前批次的状态之后，根据相应的逻辑决定是否检出，

```scala
def periodicSnapShotDump[T](
    rdd: RDD[T], startTime: Long, currentTime: Long
  ): Unit = {
    val sparkConf = rdd.context.getConf

    for {
      snapShotDir <- sparkConf.getOption(STATE_SNAPSHOT_DIR)
      snapShotDuration <- sparkConf.getOption(STATE_SNAPSHOT_CP_DURATION)
    } yield {
      val partitionNumber = sparkConf.getInt(STATE_SNAPSHOT_PARTITION, DEFAULT_CHECKPOINT_DURATION_MULTIPLIER)

      if (currentTime != startTime && (currentTime - startTime) % ScalaDuration(snapShotDuration).toMillis == 0) {
        cleanCheckpoint(rdd.context, snapShotDir)
        logger.info(s"Periodic checkpointing data into ${snapShotDir}")
        rdd.repartition(partitionNumber).saveAsObjectFile(snapShotDir)
      }
    }
}
```

本文实现的完整版，可参考[KafkaWordCounter.scala](https://github.com/jacoffee/codebase/blob/76f4dca9400587a2ef1e97aae270fafc43335bf3/src/main/scala/com/jacoffee/codebase/spark/streaming/stateful/KafkaWordCounter.scala)。不过还有两个问题没有找到很好的解决方法:

<ul class="item">
    <li>如何在不流式处理造成影响的前提下，更好的保存StateRDD</li>
    <li>流式状态更新时的去重处理</li>
</ul>

所以计划在阅读<<大数据系统构建:可扩展实时数据系统构建原理与最佳实践>>这本书之后，尝试使用Lambda架构结合Redis来解决。

## 参考

\> [Exploring Stateful Streaming](http://asyncified.io/2016/07/31/exploring-stateful-streaming-with-apache-spark/)

\> [Spark Streaming中Driver的容灾](https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html)

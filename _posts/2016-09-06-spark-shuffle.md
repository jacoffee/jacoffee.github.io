---
layout: post
category: spark
date: 2016-09-06 15:51:22 UTC
title: Spark基础之Shuffle
tags: [缓存，压缩流，shuffle write，shuffle read，Netty通讯，BlockManager]
permalink: /spark/shuffle
key: 3beaa4f402f4403835d0b4afc1ae9341
description: "本文尝试解决如下问题: Shuffle在什么时候发生，分为哪几个步骤，每个步骤有哪些重要的操作"
keywords: [缓存，压缩流，shuffle write，shuffle read，Netty通讯，BlockManager]
---

Spark中的stage划分是基于<b style="color:red">Shuffle Dependency</b>的(**Spark stages are created by breaking the RDD graph at shuffle boundaries**)。

Shuffle总体分为两个部分: **shuffle write** -- 将ShuffleMapTask中的元素写到相应的文件中; **shuffle read** -- 从Shuffle block中去读取之前存储的block，基本的流程如下图:

![shuffle](http://static.zybuluo.com/jacoffee/hfocl66r91nem7xy6c5u0g3j/image_1b3tm4rmk1bkq1u1mlsehun1vvm9.png)

下面以一个简单的例子来梳理一下Shuffle的整个过程:

```scala
val pairList = List("a" -> 1, "a" -> 2, "b" -> 3)

// ParallelCollectionRDD =====> ShuffledRDD
// ShuffleMapStage =====> ResultStage
val finalRDD = sc.parallelize(pairList).reduceByKey(_ + _)
```

上面的操作会产生两个Stage: 

(1) **Stage 0 -- ShuffleMapStage**(对应parallelize操作)，该Stage是在DAG执行流程中为shuffle产生数据的中间stage，它总是发生在每次shuffle操作(e.g. reduceByKey, join)之前并且可能会包含多个管道操作(e.g. map, filter)。但被执行之后，他们会存储map out文件(`org.apache.spark.storage.FileSegment`)以被后续的reduce任务读取。

(2) **Stage 1 -- ResultStage**(对应reduceByKey操作)，该Stage是一个Job的最后阶段。

```bash
INFO DAGScheduler: Final stage: ResultStage 1 (collect at SparkTest.scala:37)
INFO DAGScheduler: Parents of final stage: List(ShuffleMapStage 0)
INFO DAGScheduler: Missing parents: List(ShuffleMapStage 0)
```

###Shuffle write(HashShuffle)

在Stage的提交过程中，如果有父Stage，会先提交父Stage以及它的相关任务; 如果RDD之间是**`ShuffleDependency`**(ParallelCollectionRDD, ShuffledRDD)，则会产生ShuffleMapStage，进而产生ShuffleMapTask，<b style="color:red">而shuffle write正是发生在ShuffleMapTask的计算过程中的</b>。
**`ShuffleMapTask`**的主要任务就是根据ShuffleDependency中定义的Partitioner将RDD中的分区数据重新分配到不同的文件中，以让shuffle read获取(**A ShuffleMapTask divides the elements of an RDD into multiple buckets**)。

```scala
dagScheduler.submitJob()
 dagScheduler.handleJobSubmitted()
  dagScheduler.submitWaitingStages()
   dagScheduler.submitStage(stage) 
    dagScheduler.submitMissingTasks()
        case stage: ShuffleMapStage => partitionsToCompute.map { id =>new ShuffleMapTask(...) }        
```

所以接下来，我们着重来看一下**`ShuffleMapTask`**是如何实现shuffle write的，实际上就是
**HashShuffleWriter的write操作做了哪些事。**

<b class="highlight">(1) 生成HashShuffleWriter</b>

```scala
// org.apache.spark.shuffle.FileShuffleBlockResolver

private[spark] class FileShuffleBlockResolver(conf: SparkConf) {
    private val bufferSize = conf.getSizeAsKb("spark.shuffle.file.buffer", "32k").toInt * 1024
}
```

```scala
// org.apache.spark.scheduler.ShuffleMapTask

private val shuffle = shuffleBlockResolver.forMapTask(dep.shuffleId, mapId, numOutputSplits ...) {
    blockManager.diskBlockManager.getFile(blockId)
    blockManager.getDiskWriter( ... blockId ... )
}

shuffleMapTask.runTask()
    manager = SparkEnv.get.shuffleManager
    hashShuffleWriter = manager.getWriter(..partitionId..)         
```

**SparkContext**初始化的时候会生成**唯一的SparkEnv**， 而它会生成各种ShuffleManager(包括<b style="color:red">HashShuffleManager</b>， SortShuffleManager等)用于管理Shuffle的各个流程。当分区需要进行shuffle write的时候，HashShuffleManager会生成一个HashShuffleWriter，<b style="color:red">也就是说一个分区(一个Task)对应一个HashShuffleWriter</b>。在HashShuffleWriter的初始化过程中还会生成一个`FileShuffleBlockResolver` -- <b style="color:red">每一个Execuctor中只有一个</b> --
它主要负责给shuffle task分配block(后面会详细解释)。

<b class="highlight">(2) HashShuffleWriter的写入过程</b>

```scala
hashShuffleWriter.write
  for (elem <- iter) {
     val bucketId = dep.partitioner.getPartition(elem._1)
     // Array[DiskBlockObjectWriter] in ShuffleWriterGroup
     shuffle.writers(bucketId).write(elem._1, elem._2)   
     diskBlockObjectWriter.write(elem._1, elem._2)
        diskBlockObjectWriter.write(elem._1, elem._2)
        (objOut: SerializationStream).write(elem._1, elem._2)
  }

hashShuffleWriter.stop(success = true).get
    hashShuffleWriter.commitWritesAndBuildStatus()
        val sizes: Array[Long] = shuffle.writers.map { writer: DiskBlockObjectWriter =>
            writer.commitAndClose()
            // writer.fileSegment() --> FileSegment
            writer.fileSegment().length
        }
```

HashShuffleWriter只是一个逻辑概念，因为真正的写入操作是`ShuffleWriterGroup`中的writers负责的。每一个ShuffleWriterGroup都会维护<b style="color:red">numOfReducer</b>个`DiskBlockObjectWriter`, 也就是<b style="color:red">ShuffleDependency中的Partitioner中的分区数 -- ShuffledRDD的分区数</b>。然后ShuffleMapTask中的Iterator中的每一个元素都会通过key基于Partitioner计算出新的分区编号，也就是上面的bucketId，每一个bucket分配一个`DiskBlockObjectWriter`，然后将它们写入相应的`SerializationStream`中。

而`DiskBlockObjectWriter`则是由`BlockManager`(运行在每一个节点上 -- Driver和Executor -- 它提供本地或者远程，获取和存储block的接口无论是内存，磁盘还是off-heap)通过`getDiskWriter`生成的，并且关联了一个**blockId**, 它会被后续的shuffle read使用到，所以`BlockManager`提供了类似于`getBlockData`的方法。

注意在`FileShuffleBlockResolver`初始化的时候定义了一个`spark.shuffle.file.buffer`, 也就是`SerializationStream`中的buffer size，所以当流中字节超过指定的大小时，会被flush到文件中，<b style="color:red">而这个文件是和一个DiskBlockObjectWriter相对应的</b>，DiskBlockObjectWriter会不断往该文件中追加字节，直到`hashShuffleWriter.stop`调用之后，<b style="color:red">所有写入完成形成一个最终的文件，也就是`FileSegment`(一个DiskBlockObjectWriter会对应一个FileSegment)</b>。

关于这个追加过程在`DiskBlockObjectWriter`的源码中有一段解释:

```scala
/**
*  ^ 表示文件中各种位置的游标
* xxxxxxxx|--------|---       |
*         ^        ^          ^
*         |        |        最后一次追加后位置
*         |      最近一次的更新位置
*       初始位置
*
* initialPosition: 开始往文件中写入时的位置
* reportedPosition: 最近一次写入后的位置
* finalPosition: 最后一次追加结束后位置，在closeAndCommit()调用后确定
* -----: 当前写入
* xxxxx: 已经写入文件的内容
*/
```

当ShuffleMapTask结束之后，还涉及到一个问题就是map out文件的路径汇报，也就是将相关的位置信息经由executor上面的`MapOutputTrackerWorker`
汇报给Driver上的`MapOutputTracker`。

```scala
executor.run()
    execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)
     coarseGrainedExecutorBackend.statusUpdate(...)
      driverRef ! StatusUpdate(executorId, taskId, state, data)
       coarseGrainedScheduler$.backendDriverEndpoint
        taskSchedulerImpl.statusUpdate(taskId, state, data.value)    
         taskResultGetter.enqueueSuccessfulTask(taskSet, tid, serializedData)
           taskResultGetter.handleSuccessfulTask
                taskSetManager.handleSuccessfulTask
                   dagScheduler.taskEnded
                     dagScheduler.handleTaskCompletion(completion)
                          mapOutputTracker.registerMapOutputs(...)
```

再次结合文章开头的shuffle write示意图，整个流程就相对清晰了。

###Shuffle Read

在ShuffleMapStage运行完成之后，ResultStage开始运行，这里我们主要关注shuffle read是在何时被触发的。 所以，我们直接从Excecutor的任务执行开始切入。

<b class="highlight">(1) RDD分区计算的触发</b>

```scala
// org.apache.spark.executor.CoarseGrainedExecutorBackend
executor.launchTask(...taskDesc.taskId ... taskDesc.serializedTask ..)    
    threadPool.execute(...TaskRunner...)
        taskRunner.run()
        task.run(...taskId...)
        // ResultTask        
        task.runTask(context)
        // func --> (TaskContext, Iterator[(String, Int)]) => Array[(String, Int)]
        func(context, rdd.iterator(partition, context))
        rdd.getOrCompute(split, context)
        shuffledRDD.compute(split: Partition, context): Iterator[(K, C)]
        
execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)
```

从上面简化的代码来看，executor上面的任务启动之后会经过一系列调用触发Task的<b style="color:red">runTask</b>方法，从而触发我们在action中的函数，本例中就是collect， 
`(iter: Iterator[(String, Int)]) => iter.toArray`，而它最终触发了rdd的"计算"。
这里计算也许是从checkpoint处获取，也许是从cache中获取，也许是真正的计算。 本例中仅考虑真正的计算，那么我们就可以在ShuffledRDD的compute方法中看到ShuffleReader的相关逻辑。

<b class="highlight">(2) ShuffleReader的读取逻辑</b>

```bash
shuffledRDD.compute()
    shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
    BlockStoreShuffleReader(..startPartition, endPartition, mapOutputTracker...).read()
    val blockFetcherItr = new ShuffleBlockFetcherIterator {
        mapOutputTracker.getMapSizesByExecutorId(),
    }
    
    val wrappedStreams = blockFetcherItr.map { case (blockId, inputStream) =>
      //  (BlockId, InputStream) => InputStream
      //  blockFetcherItr.next() ==> (BlockId, InputStream)
      //  f(iter.next())
      //  InputStream
      serializerManager.wrapForCompression(blockId, inputStream)
    }
    
    val recordIter = wrappedStreams.flatMap { wrappedStream =>
      serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
    }

    serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
    val aggregatedIter: Iterator[Product2[K, C]] = { aggregation process }
```

首先，ShuffleReader是由`BlockStoreShuffleReader`实现的;
当计算分区中的数据时，它需要获取上一个阶段map out的文件。它会根据shuffleId, 分区的index等信息去driver上的`mapOutputTracker`获取block的信息。<b style="color:red">同样的每一个分区会对应一个BlockStoreShuffleReader</b>。

`ShuffleBlockFetcherIterator`初始化的时候，会进行将本地block获取和远程block获取进行拆分，本地获取交由本地blockManager负责；而远程block则通过发送FetchRequest来获取
。Spark限制了每一批次请求中请求字节的总大小(`spark.reducer.maxSizeInFlight`), 因此Block的请求是分批完成的，请求完成之后返回的ManagedBuffer通过相应的处理反序列化成了`Iterator[(K, V)]`。

接下来进入聚合阶段, 像上面的reduceByKey，在shuffle write的时候就已经在分区内部的进行一次聚合(**mapSideCombine**)，此时只需要按照指定的函数，比如说上面的 `value1 + value2`汇总值即可。这个逻辑是由`ExternalAppendOnlyMap`实现的，在聚合Value的过程中，map的内存占用会越来越大。
因此，会涉及到是否spill到磁盘的逻辑，简单的理解就是当超过某个阈值之后(`spark.shuffle.spill.initialMemoryThreshold`)，就会spill到磁盘。<b style="color:red">但实际过程中还会尝试申请更多内存</b>，具体逻辑可参考`org.apache.spark.util.collection.Spillable`的**maybeSpill**方法。

我们有时会在Spark UI中的Tasks视图中看到**shuffle spill(memory)**和**shuffle spill(disk)**这两个指标 -- 前者指的是spill到磁盘的所有Map的预计字节大小的总和(**total estimated byte size**), 后者指的是spill到磁盘的所有Map的实际大小。

###总结

**(1) ShuffleDependency产生ShuffleMapStage，ShuffleMapStage生成ShuffleMapTask**

**(2) Shuffle write发生在ShuffleMapTask的计算过程中，每一个ShuffleMapTask会对应一个ShuffleWriter，它会将分区中元素按照相应的规则写入不同的文件中，也就是我们通常看到的map out files或者是block files**

**(3) Shuffle read通过MapoutTracker获取相应的block file并借助各种Map对于数据进行汇总，这个过程涉及到bytes spilled到磁盘的过程**

##参考

\> [Spark Internals](https://github.com/JerryLead/SparkInternals/blob/master/markdown/english/4-shuffleDetails.md)

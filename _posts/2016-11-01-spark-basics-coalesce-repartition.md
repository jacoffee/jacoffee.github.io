---
layout: post
category: Spark
date: 2016-11-01 13:30:55 UTC
title: Spark基础之coalesce和repartition
tags: [哈希分区，默认并发度, balls-into-bin, 随机负载选择]
permalink: /spark/partitions/coalesce-repartition
key: 988da0e17e1944ec4d250c69f29cb028
description: 本文介绍了Spark RDD中的coalesce与repartition的实现
keywords: [哈希分区，默认并发度, balls-into-bin, 随机负载选择]
---

在Spark中，如果想要对分区数进行调整，我们一般会使用**coalesce**和**repartition**这两个方法。

coalesce主要是用于减小RDD中的分区(当然增加也是没有问题的)，
比较典型的使用场景: 

+ <b style="color:red">通过textFile读取HDFS文件之后</b>，发现分区太多;

+ <b style="color:red">filter操作之后的分区压缩</b>，进行筛选操作之后，某些分区中的数据可能急剧下降，这样就导致各个分区之间的数据不均衡，通过coalesce进行压缩以更好的利用集群资源(保证任务数跟集群资源有一个较好的平衡)

而repartition既可以用来增加分区也可以用来减少分区，不过一般伴随着RDD分区的shuffle。

之前对于这两个方法的理解仅限于此，但今天一个同事再做一个任务的时候，将这两种操作的区别进一步展示出来了。逻辑其实很简单对于1000个分区分别进行筛选操作，将最终结果压缩到1个分区存入HDFS系统中。他最开始使用coalesce的时候总是会显示内存不够，后来改为repartition之后才完成了。

后来在源码文档中发现了下面这句话:

>  However, if you're doing a drastic coalesce, e.g. to numPartitions = 1, this may result in your computation taking place on fewer nodes than you like (e.g. one node in the case of numPartitions = 1)

最后结合Stage的划分弄清楚了这个问题，以下面的简化实例予以说明。

```scala
// textFile --> 默认读取的是1000个分区
val rdd = sc.textFile(bigFile).filter(...).coalesce(1).saveAsTextFile(...)
val rdd = sc.textFile(bigFile).filter(...).repartition(1).saveAsTextFile(...)
```

对于coalesce，前面三个阶段都是窄依赖所以可以合并，但由于最终只有一个分区，在节点上计算父RDD(HadoopRDD)的分区时候需要从各个节点拉取数据，然后做过滤操作，最终导致计算节点因内存不足而任务失败。

而repartition因为有shuffle操作，所以被分割成为两个Stage。筛选操作发生在各个节点(筛选力度较大，导致数据量减小)，然后通过shuffle read转移到单个节点，所以内存占用会小很多。

总结一句就是，**如果通过coalesce(默认行为)骤降分区数，计算会发生在很少的节点上，可能会由于内存不足导致任务失败，所以要慎用**

后来通过阅读coalesce的源代码，对于减小分区时，它是如何构建新分区和原分区的关系(新分区以原RDD中的哪几个分区作为父分区，需不需要考虑就近原则，需不需要考虑负载等等)的又有了新的了解。

首先，这两种方法都产生了**CoalescedRDD**，只不过repartition过程产生了一个中间RDD也就是ShuffledRDD。我们知道在RDD的计算过程中，都会有一个方法是获取其中的分区，然后逐个追溯父分区，然后计算到当前分区，所以**getPartitions**便是建立分区之间关系的核心。

```scala
// balanceSlac --> 负载和就近原则之间的平衡， 1.0 is all locality, 0 is all balance
private[spark] class CoalescedRDD[T: ClassTag](
    prev: RDD[T], 
    maxPartitions: Int, 
    balanceSlack: Double = 0.1
) {
    ...
    override def getPartitions: Array[Partition] = {
        val pc = new PartitionCoalescer(maxPartitions, prev, balanceSlack)
    }
    ...
}

private class PartitionCoalescer(
    maxPartitions: Int, prev: RDD[_], balanceSlack: Double
) { 
    ...
    val groupArr = ArrayBuffer[PartitionGroup]()
    
    // 如果是true的话，父RDD中的分区就不存在preferredLocations
    var noLocality = true
    ...
}
```

CoalescedRDD构建的过程中，会生成M个PartitionGroup，然后把原来的Partition按照规则放入PartitionGroup中，实际上PartitionGroup对应的就是新生成的CoalesceRDD中的一个Partition，只不过它依赖父RDD中的多个Partition，也就是计算的时候从固定个父RDD的Partition中取数据(窄依赖)。

而这种实现包含了一个重要的算法[balls-into-bins](https://en.wikipedia.org/wiki/Balls_into_bins) -- 它解决的是将m个球放入n个盒子。每一次，我们会将一个球放入到其中一个盒子中。当所有的球都在盒子中了，汇总每个盒子中球的个数，这个数字也称为该盒子的负载，我们要得出的结论是单个盒子中最大的负载是多少？

对应来看就是现有prev.partitions.size个分区，我们产生math.min(prev.partitions.size, maxPartitions)个分区组(盒子)。然后将这些分区放进去，放的时候需要在两个原则之间进行权衡: 就近原则(locality)和负载均衡(load balance)。也就是说如果PartitionGroup在**server A**，有一个待放入的Partition也在**server A**上，如果从就近原则考虑应该放在**server A**，但如果此时**server A**的负载已经很重，我们需要找出其它负载较小的ParttitionGroup然后放进去。

首先，目标分区数**targetLen**(PartitionGroup的组数)是原RDD分区数和传入的分区数之间的最小值。

<b class="highlight">(1) 产生PartitionGroup -- CoalescedRDD.setupGroups</b>

因为分组是就近原则和负载均衡的一个权衡，所以首先会获取父RDD中每一个分区的最优计算位置。

```scala
class LocationIterator(prev: RDD[_]) extends Iterator[(String, Partition)] {
      def resetIterator(): Iterator[(String, Partition)] = {
        ...
        prev.partitions.iterator.flatMap(p => {
          if (currPrefLocs(p).size > x) Some((currPrefLocs(p)(x), p)) else None
        } )
        ...
    }
}
```

如果父RDD的分区没有优先位置，则新建**targetLen**个PartitionGroup。

```scala
(1 to targetLen).foreach { groupArr += PartititonGroup() }
```

如果父RDD的分区中有优先位置，要么创建出**targetLen**个唯一并且独立的优先位置，要么经过
`expectedCoupons2`次迭代，这种迭代涉及到另外一种算法[赠券收集问题](https://zh.wikipedia.org/wiki/%E8%B4%88%E5%88%B8%E6%94%B6%E9%9B%86%E5%95%8F%E9%A1%8C) -- 假设有n种赠券，每种赠券获取概率相同，而且赠券可以无限供应。若取赠券T张，能集齐n种赠券的概率多少？在Spark中实际上就是，通过多少次迭代能获取到所有唯一的host地址，但一般来说PartitionGroup的个数都要大于机器数，所以一个机器上通常有多个PartitionGroup。

```scala
 def setupGroups(targetLen: Int) {
    val expectedCoupons2 = 2 * (math.log(targetLen) * targetLen + targetLen + 0.5).toInt

    // 每遇到一个host地址，就新增一个PartitionGroup；如果尝试次数结束之后， 
    // PartitionGroup还是不够，则在之前的host上再增加PartitionGroup，然后将Partition放进去
    // 所以在setGroups过程有些Partition就已经提前被放进去了，所以后面才会有负载的考虑
    while (numCreated < targetLen && tries < expectedCoupons2) {
       ....     
    }
    
    while (numCreated < targetLen) {
       ....
    }
 }
```


<b class="highlight">(2) 将Partition放入PartitionGroup中 -- CoalescedRDD.throwBalls()</b>

如果父RDD的分区没有优先位置(`noLocality=true`)

```scala
// groupArr.size => math.min(prev.partitions.length, maxPartitions)
// 如果coalesce是想扩大分区，则直接将父RDD中的每一个分区放入每一个PartitionGroup
if (maxPartitions > groupArr.size) {
    for ((par, i) <- prev.partitions.zipWithIndex) {
        groupArr(i).arr += p
    } else {
        // 如果coalesce的操作是缩小分区，则以maxPartitions为步长，将原RDD分区拆成maxPartitions个组
        
        for (i <- 0 until maxPartitions) {
            // 0 / M ~ 1 / M -> 1 / M ~ 2 / M
            val rangeStart = (i * prev.partitions.size) / maxPartitions
            val rangeEnd = ((i + 1) * prev.partitions.size) / maxPartitions
           
           （rangeStart until rangeEnd).foreach { j => groupArr(i).arr += prev.partitions(j) }
        }
    
    }
}
```

如果父RDD的分区有优先位置(`noLocality=false`)，会通过**pickBin**方法，在[the power of two choices in randomized load balancing](https://brooker.co.za/blog/2012/01/17/two-random.html)原则的指导下，来将原RDD中的分区依次放入。

```scala
for (p <- prev.partitions if (!initialHash.contains(p))) { 
    pickBin(p).arr += p
}
```

上面的**initialHash**充分说明，之前已经有Partition被提前放进PartitionGroup里面了。


<b class="highlight">(3) 计算时的逻辑 </b>

因为coalesce形成的是窄依赖，新RDD分区计算的时候就需要取原RDD中对应的分区(即PartitionGroup中对应的分区)。

```scala
override def getPartitions: Array[Partition] = {
    // pc.run() ==> Array[PartitionGroup]
    pc.run().zipWithIndex.map {
      case (pg, i) =>
        val ids = pg.arr.map(_.index).toArray
        
        new CoalescedRDDPartition(i, prev, ids, pg.prefLoc)
    }
}
```

## 参考

\> [randomized load balancing](https://gist.github.com/pbailis/4964307)
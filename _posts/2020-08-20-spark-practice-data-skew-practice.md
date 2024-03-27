---
layout: post
category: spark
date: 2020-08-15 15:29:03 UTC
title: 【Spark实战】数据倾斜处理
tags: [Data Skew, Adpative Optimization]
permalink: /spark/practice/data-skew
key:
description: 本文介绍Spark中处理Data Skew的一些思路以及skew join optimize的原理
keywords: [Data Skew, Adpative Optimization]
---

Spark在进行数据处理的时候，由于数据分布的问题，有可能会造成处理时间不均匀，在Spark UI上的体现就是有的Task很快完成，但是有的任务需要很长时间才能完成，也就是我们经常说的**数据倾斜问题或者说长尾问题**。

# 1. 解决思路

## 1.1 分析数据分布然后过滤导致倾斜的key

一般倾斜是因为shuffle导致数据分布到下游算子的时候不均匀导致的，所以我们可以针对分组key 或者是join key对数据进行分布分析。



以key=user_id为例，我们可以通过如下代码去观察某个key的分组数量的中位数、标准差等，来判断skew的程度。

```scala
dataframe.groupBy("user_id").count().agg(mean("count"), stddev("count"), count("*")).show()
```

```bash
df1 = [mean=4.989209978967438, stddev=2255.654165352454, count=2400088] 
df2 = [mean=1.0, stddev=0.0, count=18408194]
```

从结果显示 df1的数据发生了明显的波动，平均值接近5， 但是标准差到达了2255，说明偏离平均值非常多，也就是不同user_id分组之后，组内之间的数量差异特别大。



我们获取了分布数据之后，可以基于这个标准将 df1 数据集 进行拆分 然后再和 df2 进行join，最后union all。

```scala
df1.groupBy("id").count().agg(mean("count").alias("mean"), stddev("count").alias("stddev"), count("*")).show()

val meanResult = 4.989209978967438
val sdResult = 2255.654165352454

val counts = df1.groupBy("id").count().persist()

// 寻找一个标准值threshold 比如说 meanResult + 2 * sdResult
val inFrequentDf = counts.filter($"count" <= threshold).join(df1, Seq("userId"))
val frequentDf = counts.filter($"count" > threshold).join(df1, Seq("userId"))
```

## 1.2 补充随机前缀 + 两阶段聚合(局部聚合 + 全局聚合)

这种比较适合聚合操作导致的data skew。 核心思路就是针对分组key，增加随机前缀或者后缀，基于重塑的key进行分组聚合，分组之后移除前缀或者后缀再次聚合，试图通过第一次的局部聚合减少分组中的数量。

用SQL的逻辑写出来大致就是这样的(ClickHouse SQL):

```sql
select 
  splitByChar('_', suffix_key)[1] as original_key,
  sum(partial_total)
from (  
  select
  concat(toString(uid), '_', toInt32(randUniform(0, 10))) as suffix_key, 
  count(*) as partial_total
  from users
  group by
  concat(toString(uid), '_', toInt32(randUniform(0, 10))) as suffix_key
) group by original_key;
```

## 1.3  借助Broadcast Join

如果内存抗的住话，将某个表广播出去，相当于转换为map side join，这个技术在很多成熟的数据库系统也有使用，但相对来说会很局限。

# 2. Apative Skewed Join Optimization实现剖析

在Spark 3.0之后支持了adaptive optimize skewed join， 也就是Spark会自动检测并自适应的解决数据倾斜问题，我们着重来看看这一部分是如何实现的？

`spark.sql.adaptive.skewJoin.enabled=true`。

## 2.1 界定Skewed Partition

这个优化中提出的标准就是：Partition大小，如果某个Partition的大小 超过 预设的值 与  median size *  skewed Factor的最大值，则视为skewed。

```scala
case class OptimizeSkewedJoin(ensureRequirements: EnsureRequirements) extends Rule[SparkPlan] {
   
 val SKEW_JOIN_SKEWED_PARTITION_THRESHOLD =
    buildConf("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes")
      .doc("A partition is considered as skewed if its size in bytes is larger than this " +
        s"threshold and also larger than '${SKEW_JOIN_SKEWED_PARTITION_FACTOR.key}' " +
        "multiplying the median partition size. Ideally this config should be set larger " +
        s"than '${ADVISORY_PARTITION_SIZE_IN_BYTES.key}'.")
      .version("3.0.0")
      .bytesConf(ByteUnit.BYTE)
      .createWithDefaultString("256MB")
    
  val SKEW_JOIN_SKEWED_PARTITION_FACTOR =
    buildConf("spark.sql.adaptive.skewJoin.skewedPartitionFactor")
      .doc("A partition is considered as skewed if its size is larger than this factor " +
        "multiplying the median partition size and also larger than " +
        "'spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes'")
      .version("3.0.0")
      .intConf
      .checkValue(_ >= 0, "The skew factor cannot be negative.")
      .createWithDefault(5)  
    
 def getSkewThreshold(medianSize: Long): Long = {
    conf.getConf(SQLConf.SKEW_JOIN_SKEWED_PARTITION_THRESHOLD).max(
      medianSize * conf.getConf(SQLConf.SKEW_JOIN_SKEWED_PARTITION_FACTOR))
  }   
    
}
```



举例 如果某个分区的大小为 1000M， 预设值为512M,  Median Size=300M，skewed Factor = 3，基于规则这个分区会视为Skewed Partition。

```scala
skewedPartitionThresholdInBytes = 1000M
median size * skewed factor = 900M
```

## 2.2  进一步拆分Skewed Partition

一句话总结这个解决方案: 针对Skewed Partition进一步拆分， 由原来的 skewed left P1 join P2 变成 skewed left P1 part 1 join P2 和 skewed left P1 part 2 join P2。

`org.apache.spark.sql.execution.adaptive.OptimizeSkewedJoin` 顶部的注释已经给了很清晰的解释了。

```scala
For example, assume the Sort-Merge join has 4 partitions:
left:  [L1, L2, L3, L4]
right: [R1, R2, R3, R4]

Let's say L2, L4 and R3, R4 are skewed, and each of them get split into 2 sub-partitions. This
is scheduled to run 4 tasks at the beginning: (L1, R1), (L2, R2), (L3, R3), (L4, R4).
This rule expands it to 9 tasks to increase parallelism:

(L1, R1),
(L2-1, R2), (L2-2, R2),
(L3, R3-1), (L3, R3-2),
(L4-1, R4-1), (L4-2, R4-1), (L4-1, R4-2), (L4-2, R4-2)
```



如果对于Spark执行计划比较了解的，应该可以猜到，这个肯定是以一种优化的Rule实现的。

````bash
			...
	ShuffledHashJoinExec
		/          \ 
     left				right
````



树形遍历的时候，检测到左边或者右边的数据集发生Skew，针对Skewed partition进行拆分，变成多个sub partition, 变向更改left plan or right plan。

切分的规则会 结合 map size  和 目标 target size(基于not skewed partition size进行计算)。

```scala
// org.apache.spark.sql.execution.adaptive.ShufflePartitionsUtil
/**
   * Splits the skewed partition based on the map size and the target partition size
   * after split, and create a list of `PartialReducerPartitionSpec`. Returns None if can't split.
   */
def createSkewPartitionSpecs(
    shuffleId: Int,
    reducerId: Int,
    targetSize: Long,
    smallPartitionFactor: Double = SMALL_PARTITION_FACTOR)
: Option[Seq[PartialReducerPartitionSpec]] = {

}
```

# 3. 参考

\> [Skewed Join Optimization Design Doc](https://issues.apache.org/jira/secure/attachment/12983701/Skewed%20Join%20Optimization%20Design%20Doc.docx)

\> [Skewed Join对应的github pull request](https://github.com/apache/spark/pull/26434)

\> [Spark性能优化指南——高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

\> [Spark final task takes 100x times longer than first 199, how to improve](https://stackoverflow.com/questions/38517835/spark-final-task-takes-100x-times-longer-than-first-199-how-to-improve)

\> [Wiki data skew](https://en.wikipedia.org/wiki/Skewness)

---
layout: post
category: bigdata
date: 2017-11-01 10:30:55 UTC
title: Hive中连表查询时的倾斜(skew)解决
tags: []
permalink: /bigdata/skew/solution
key: 
description: 本文总结了一次Hive查询中，连表倾斜解决的过程，包括在其它OLAP引擎上类似的处理
keywords: [OLAP, BroadcastJoin, MapJoin, Skew, Hive, Spark, Impala]
---

今天线上一个连表查询的批次任务运行时，通过查看MapReduce的reduce任务详情，发现某个任务运行时间明显长于其它的reduce任务并且运行较长。初步断定是连表时，某个Key数量过大，导致对应的reducer中数据量太大，所以导致运行速度变慢。

简化的连表:

```bash
select a.productid, a.publish_date, max(b.x) as x 
from A a join B b
on a.productid = b.productid
where a.publish_date > b.x
group by a.productid, a.publish_date
```

## 1.连表时的倾斜

### 1.1 使用mapjoin

前面已经提到，在reducer处发生了倾斜，对于这种场景，首先看<b class="highlight">能否转换成mapjoin</b>，即将小表放入内存中，然后广播到各个mapper中，在mapper所在容器的节点上进行本地的"连表"，这样就避免了reduce操作导致大量key聚合。

默认情况mapjoin的参数是开启的:

```bash
hive.auto.convert.join=true
```

决定是否将表放入内存由下面的参数控制默认只有23M，所以需要根据业务场景进行适当的调整。

```bash
hive.mapjoin.smalltable.filesize=25000000
```

在进行调整之后，我们可以通过explain来确认是否触发了mapjoin，这个信息可以在**Map Operator Tree**的相关部分看到:

```bash
explain select a.productid, a.publish_date, max(b.x) as x from A a join B b on
a.productid = b.productid where a.publish_date > b.x group by a.productid, a.publish_date
```

```bash
Statistics: Num rows: 31591763 Data size: 953670250 Basic stats: COMPLETE Column stats: NONE
              **Map Join Operator**
                condition map:
                     Left Outer Join0 to 1
                keys:
```

### 1.2 分离出热点数据，然后再尝试`mapjoin`

mapjoin适用于大笔与小表的连表，如果小表大到一定程度，比如说超过1G，这时候我们可以尝试从右表中剥离热点key。

```bash
select productid, count(*) total from B group by productid order by total desc;

Key1 Number1
Key2 Number2
....
```

假设Key1的数量明显高于其它的Key，则可以认定为倾斜Key，接下来分别进行处理最后进行合并(处理的原则就是拆分之后的数据大小满足mapjoin的要求)。

```bash
select a.productid, a.publish_date, max(b.x) as x from A a join B b 
on a.productid = b.productid where b.productid = 'Key1' 
and a.publish_date > b.x group by a.productid, a.publish_date
union all
select a.productid, a.publish_date, max(b.x) as x from A a join B b 
on a.productid = b.productid where b.productid <> 'Key1' 
and a.publish_date > b.x group by a.productid, a.publish_date
```

不过上面这种方式对于表进行了**重复扫描**。
## 2.Spark SQL与Impala中的mapjoin

Spark SQL(ThriftServer)也提供了类似的机制即**BroadcastHashJoin**，即将小表广播到各个Executor中，然后和各个分区中的数据进行本地聚合。

需要通过如下参数设定(广播的阈值，表的大小小于此值才进行广播):

```bash
set spark.sql.autoBroadcastJoinThreshold = 1073741824;
```

Impala提供的机制也是类似的，默认情况下会将join右边的表广播出去，所以在右边的表比较小的时候这个机制是没有问题的，<b class="highlight">当两个表大小差不多时并且都比较大的时候</b>，可以考虑使用shuffle机制来实现。

>  Typically, partitioned joins(shuffle) are more efficient for joins between large tables of similar size.

```bash
select a.productid, a.publish_date, max(b.x) as x 
from A a join /* +SHUFFLE */ B b
on a.productid = b.productid
where a.publish_date > b.x
group by a.productid, a.publish_date
```

```bash
| |                                                                                  |
| 02:HASH JOIN [LEFT OUTER JOIN, BROADCAST]                                          |
| |  hash predicates: a.productid = b.productid                             |
| |                                                                                  |
| |--03:EXCHANGE [BROADCAST]                                                         |
| |                                         
```

连表倾斜的根源是key分布不均匀，所以我们关注的重点是如何**更均匀的分散join key**。一种思路就是提前在map的时候去做，另外一种思路是将join key随机的分散在不同的reducer中，这样也可以一定程度上的避免key倾斜。


## 参考

\> Apache Hive Essentials
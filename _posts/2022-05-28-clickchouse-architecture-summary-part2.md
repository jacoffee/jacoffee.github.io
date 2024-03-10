---
layout: post
category: database
date: 2022-05-30 15:29:03 UTC
title: 【数据库综述系列】ClickHouse综述(下)
tags: [OLAP数据库研究、ClickHouse、Vectorization Processing, Distributed Read&Write]
permalink: /database/summary/clickhouse/part2
key:
description: 本文介绍从宏观层面ClickHouse
keywords: [OLAP数据库研究、ClickHouse、Vectorization Processing, Distributed Read&Write]
---

[TOC]

在[【数据库综述系列】ClickHouse综述(上)](http://roadtopro.top/database/summary/clickhouse/part1)中，我们针对ClickHouse的基础架构、MergeTree存储结构进行了展开，本篇我们将针对ClickHouse中分布式表的写入和查询流程进行剖析，由于是综述性质所以会更偏向于流程性的东西。

# 1. 关于分布式表

首先分布式表对应的是Distributed表引擎，它本身不存储数据，而是数据分片的代理，能够自动路由数据至集群中的其它节点。

```sql
ENGINE = Distributed('cluster_name',  'database', 'local_table_name',  sharding key expression);
```

# 2. 写入流程


## 2.1 选择Shard中的Replica进行数据写入

这部分主要涉及到Shard中的Replica选择算法，支持的几种算法都在[SettingsEnums.h](https://clickhouse.com/codebrowser/ClickHouse/src/Core/SettingsEnums.h.html)中定义了。

+ RANDOM -- 默认的负载均衡算法，在ClickHouse的服务节点中，拥有一个全局计数器error_count，当服务发生任何异常时，该计数累积加1。而random算法会选择error_count错误数量最少的replica，如果多个replica的errors_count计数相同，则在它们之中随机选择一个

+ ROUND_ROBIN -- 在错误数相同的Replica中轮询


## 2.2 写本地part，然后存储日志

从下面的日志可以很清晰的看出，当前Replica会先尝试写本地文件，然后将"相关指令"封装成log，写入Zookeeper，相当于变向通知其它Replica进行数据同步。

```bash
(Replicated OutputStream): Wrote block with ID '20240307_4336155613182891851_14196419963788083537', 1 rows

Renaming temporary part tmp_insert_20240307_5_5_0 to 20240307_2_2_0 with tid (1, 1, 00000000-0000-0000-0000-000000000000).

# 存储Log
(ReplicatedMergeTreeQueue): Insert entry queue-0000029888 to queue with type GET_PART
with virtual parts [20240307_2_2_0]
```

## 2.3 副本之间的同步

为了减少分布式表写入过程的压力，一般会将shard中的`internal_replication`设置为true，也就是写入Shard中的一个replica之后，如果采用ReplicatedMergeTree引擎，其它副本会自动从某个副本同步数据。这个过程中会重度依赖Zookeeper来作为协调工具，[【ClickHouse基础】Zookeeper in ClickHouse](http://roadtopro.top/database/clickhouse/zookeeper) 中的`1.3 分布式写入的数据同步`。

+ Other Replica发起同步请求

```bash
Fetching part 20240307_1_1_0 from /clickhouse/tables/xxxxx/01/replicas/cluster01-01

(90192e5c-f812-4875-be83-1e2a086f807e) (Fetcher): Downloading part 20240307_1_1_0 onto disk default.
(90192e5c-f812-4875-be83-1e2a086f807e) (Fetcher): Download of part 20240307_1_1_0 onto disk default finished.

Renaming temporary part tmp-fetch_20240307_1_1_0 to 20240307_1_1_0 with tid (1, 1, 00000000-0000-0000-0000-000000000000).
(90192e5c-f812-4875-be83-1e2a086f807e): Fetched part 20240307_1_1_0 from /clickhouse/tables/xxxxx/01/replicas/cluster01-01-1
```

+ Source replica接受Other Replica的同步请求，然后响应，将相应的Part数据传递给对应的Replica

```bash
<Trace> InterserverIOHTTPHandler: Request URI: /?endpoint=DataPartsExchang...
(90192e5c-f812-4875-be83-1e2a086f807e) (Replicated PartsService): Sending part 20240307_2_2_0

<Debug> InterserverIOHTTPHandler: Done processing query
```


## 2.4 关于写入的一些问题

### 2.4.1 一致性保障

默认是提供**最终一致性(eventual consistency)**，主要是由于Replica之间的异步复制，所以有可能读取的时候，选择的是数据还没有完全的同步的Replica。

不过可以通过Quoram NWR机制解决，假设是Repliace=2，可以通过**insert_quorum=2**来限制一定要两个Replica都收到响应，才返回给客户端成功。如果active replica小于这个配置(通过检查Zookeeper中相应路径下的活跃replica)，写入的时候则会报错 **TOO_FEW_LIVE_REPLICAS**。

```c++
// ReplicatedMergeTreeSink.cpp

template<bool async_insert>
void ReplicatedMergeTreeSinkImpl<async_insert>::consume(Chunk chunk) {
    
    
    /** If write is with quorum, then we check that the required number of replicas is now live,
      *  and also that for all previous parts for which quorum is required, this quorum is reached.
      * And also check that during the insertion, the replica was not reinitialized or disabled (by the value of `is_active` node).
      * TODO Too complex logic, you can do better.
      */
    size_t replicas_num = checkQuorumPrecondition(zookeeper);
    
}
```

以下配置主要是针对非[SharedMergeTree](https://clickhouse.com/docs/en/cloud/reference/shared-merge-tree#consistency)的一致性相关配置:

| 配置                          | 说明                                                         | 默认值        |
| ----------------------------- | ------------------------------------------------------------ | ------------- |
| insert_distributed_sync       | 写入Shard是同步还是异步的控制，同步写就是要写到所有的Shard才算成功(当然默认是replica只要一个成功就行) | 0-异步        |
| insert_quorum                 | 是否开启quoram write, 可以类比Kafka ack<br />`insert_quorum <2`  the quorum writes are disabled<br />If `insert_quorum >= 2`, the quorum writes are enabled<br />If `insert_quorum = 'auto'`, use majority number (`number_of_replicas / 2 + 1`) as quorum number. | 0 -- 关闭     |
| insert_quorum_parallel        | 允许并行写入Quoram，就是之前的写入还没有完成，可以继续发送请求，为了更高的吞吐所以默认开启了(if enabled, additional `INSERT` queries can be sent while previous insert have not yet finished) | 1 -- 默认开启 |
| insert_quorum_timeout         | 如果开启了insert_quorum，需要检验写入quoram是否在指定timeout满足了，否则就会报错 | 600000ms      |
| select_sequential_consistency | 是否开启读一致性支持(Enables or disables sequential consistency for `SELECT` queries)，需要关闭并行插入才能生效 | 0 -- 关闭     |

关于读一致性的一点说明，在ClickHouse我们可以通过如下方式实现:

+ Read-On-What-Node-You-Write，即从写的节点读
+ 通过上面的`select_sequential_consistency`参数，它的核心原理就是 检查当前part `要么在当前replica写入` 或者是`在所有的Quoram中都成功写入`

### 2.4.2 关于双主复制(mulit-master replication)

这个指的是一个Shard的副本之间，并没有所谓的主从关系，每一个都可以接收写入，然后基于Zookeeper进行数据同步。
具体的可以参照《[【ClickHouse基础】Zookeeper in ClickHouse](http://roadtopro.top/database/clickhouse/zookeeper)》1.3 分布式写入的数据同步，但是这个过程是异步的，所以会存在一定的延时。

在这种设计下，即使其中一个Replica down掉之后，写入仍然可以继续(假设insert_quoram关闭)。等待Replica恢复之后，会从基于相关日志来同步拉取落下的数据。

# 3. 查询流程

以下面这个查询为例， event_record_all为分布式表，对应的本地表为event_record_local，按照`toYYYYMMDD(event_time)`进行分区。

```sql
select
title, count(*)
from
event_record_all
where
toDate(event_time) = '2024-02-21'
group by title;
```

## 3.1 如何锁定分区Part文件

在展开查询流程之前，我们先来解决一个问题，就是ClickHouse是如何结合分区键查询条件定位到指定的data part的。分区表达式是`toYYYYMMD`这个格式的，而表达式筛选条件是'yyyy-MM-dd'这两个是如何匹配的呢？

### 3.1.1 基于分区键存储 minmax_event_time.idx

由于event_time是DateTime类型，而底层是UnixTimeStamp形式存储的，所以会Part写入的时候，存储记录中最小和最大值。 假设写入的两条数据中的event_time分别是 '2024-02-21 08:16:16'、'2024-02-21 09:17:17'，就会先尝试转换为DateTime， 然后再使用 `toInt32(event_time)=170850337`、`toInt32(event_time)=1708507037`，转换为UnixTimeStamp之后存入`minmax_event_time.idx`。

下面我们通过`od - dump files in octal and other formats`来查看`minmax_event_time.idx`中存储的内容，会发现和预期正好一样

```bash
# minmax_event_time.idx
# j skip 0 bytes
# 读取8byte 因为DateTime底层存储的是秒 toInt32(now())，正好是4byte * 2
od -i -j 0 -N 8 minmax_event_time.idx
0000000  1708503376  1708507037
0000010
```

### 3.1.2 查询时，解析分区筛选条件对应的值，形成KeyCondition

如果观察过ClickHouse Server日志，会经常看到分区表达式，会输出如下的日志：

```bash
MinMax index condition: (toDate(column 0) in [19774, +Inf)), (toDate(column 0) in (-Inf, 19774]), and
```

虽然我们SQL查询中写的是`toDate(event_time) = '2024-02-21'`,  但ClickHouse使用按照Range的方式进构建。

### 3.1.3  基于Part的minmax构建KeyRange

上面我们提到minmax会记录分区键的最小值和最大值，天然形成Part数据的左边界和右边界。

---

**下面转换部分，纯属个人基于初步看 KeyCondition.cpp源码的猜测**

+ 查询条件中 分区列筛选条件的类型是Date，所以会尝试将右边的值转换为Date，然后再次转换为UnixTimeStamp
+ 与此同时读取minmax中的数据，然后尝试转换为UnixTimeStamp，进行比较

---

通过下面的SQL ，我们可以更好的理解这个转换

```sql
select 
toDate('2024-02-21') as parseDate,
toInt32(parseDate),
toDate(1708503376) as min_date,
toInt32(min_date) as min_date_sec,
toDate(1708507037) as max_date,
toInt32(max_date) as max_date_sec;
```

```bash
┌──parseDate─┬─toInt32(toDate('2024-02-21'))─┬───min_date─┬─min_date_sec─┬───max_date─┬─max_date_sec─┐
│ 2024-02-21 │                         19774 │ 2024-02-21 │        19774 │ 2024-02-21 │        19774 │
└────────────┴───────────────────────────────┴────────────┴──────────────┴────────────┴──────────────┘
```

### 3.1.4  最后检查minmax形成的KeyRange是否在KeyCondition中

这部分的源码详情可以参照 [KeyCondition.cpp](https://clickhouse.com/codebrowser/ClickHouse/src/Storages/MergeTree/KeyCondition.cpp.html)。

```c++
BoolMask KeyCondition::checkInRange(
    size_t used_key_size,
    const FieldRef * left_keys,
    const FieldRef * right_keys,
    const DataTypes & data_types,
    BoolMask initial_mask) const
{
    
}
```

里面的涉及到 Reverse Polish notation，逆波兰表达式之类的，反正看的云里雾里的。不过我想理解到这个地步，对于个人来说已经足够了。当然还会基于`partition.dat`，也就是part数据存储的时候，分区表达式中的值，进行相关裁剪。具体可参考[PartitionPruner.h](https://clickhouse.com/codebrowser/ClickHouse/src/Storages/MergeTree/PartitionPruner.h.html)

```c++
// 当前part是不是可以被剔除掉
PartitionPruner::canBePruned(const IMergeTreeDataPart & part)
    const auto & partition_value = part.partition.value;
	std::vector<FieldRef> index_value(partition_value.begin(), partition_value.end());	
	
	 is_valid = partition_condition.mayBeTrueInRange(partition_value.size(), index_value.data(), index_value.data(), partition_key.data_types);
		KeyCondition::mayBeTrueInRange(partition_value.size(), index_value.data(), index_value.data(), partition_key.data_types);
            checkInRange(used_key_size, left_keys, right_keys, data_types, BoolMask::consider_only_can_be_true).can_be_true;
```


## 3.2 分布式表SQL查询

### 3.2.1 SQL查询的改写

ClickHouse的分布式表查询，采用**scatter-gather模式**，也就是将分布式表拆分成对于多个本地表的查询，然后针对结果进行聚合。 上面的SQL实际执行的时候，分布式表会被替换成对应的本地表，然后发送到相应的Shard。

```sql
explain
select
title, count(*)
from
event_record_local
where
toDate(event_time) = '2024-01-14'
group by title
```

通过explain命令可以看到实际执行的时候，同时查询了本地表和远程表，而`MergingAggregated`则体现针对各本地表的**聚合结果的再次合并**。

```bash
┌─explain──────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY))                  │
│   MergingAggregated                                          │
│     Union                                                    │
│       Aggregating                                            │
│         Expression (Before GROUP BY)                         │
│           Filter (WHERE)                                     │
│             ReadFromMergeTree (event_record_local) │
│       ReadFromRemote (Read from remote replica)              │
└──────────────────────────────────────────────────────────────┘
```

```bash
# 向其它节点发起了连接请求，准备执行SQL
[xxxx-host0001] yyyyy 15:46:32.496818 [ 9508 ] {245b23c0-5804-46f6-ae22-a4b065ec88f4} <Debug> Connection (xxxx-host0004:9002): Sent data for 2 scalars, total 2 rows in 4.9469e-05 sec., 39749 rows/sec., 68.00 B (1.28 MiB/sec.), compressed 0.4594594594594595 times to 148.00 B (2.79 MiB/sec.)
```



### 3.2.2 副本分片选择

这个在上面已经提到过了，此处不再赘述。

### 3.2.3 关于[Distributed Subqueries](https://clickhouse.com/docs/en/sql-reference/operators/in#select-distributed-subqueries)

在分布式表带子查询的时候，一定要注意IN / JOIN  vs  Global IN / GLOBAL  JOIN的区别。假设是3 Shard的分布式表，针对下面的查询: 

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

+ 外层是分布式表，所以转换成local_table，分发到不同的shard执行

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

+ Remote Shard接收到查询之后，发现内层还是分布式表，所以会继续上面的流程，转换为local_table之后再次发送到不同的Remote Shard

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

按照上面的3个Shard计算，就已经有9次请求了， 也就是shard number  * shard number， 如果shard数越多，这个肯定是不能接受的。



下面两种改进思路虽然是按照Join的方式展开，但是对于IN同样适用，核心是要掌握关键的思想。

#### 3.2.3.1 Global Join(broadcast join)

一种思路就是将 **右表(ClickHouse始终是将右表的数据放入内层中)**  中涉及的数据先查询出来，然后拉到Initiator node，作为临时表。最后再将该临时表发送到不同的节点，这样外层表数据和临时表就可以在对应的节点进行相关的操作，无论是IN还是Join。这种操作相当于将一部分数据广播出去了，所以在很多数据库中也称之broadcast join，最初是在Spark SQL中接触到这个概念的。

显然这种优化有一个限制，就是**尽量减少广播出去的数据大小**，不然计算是一方面，另外一方面对于带宽影响也会很大。

#### 3.2.3.2 Colocate Join

这种的话通用性会差点，因为它对于Shard数据分布有一点要求，**要求join的表都按照相同的key进行shard**，这样的话就可以直接进行本地join，而不会产生数据问题。

针对上面的SQL, 就可以直接将内层的 distributed_table ---> local_table

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```



# 4. 总结

这部分主要梳理本人在实际开发中使用ClickHouse中的一些痛点，以Clickhouse版本(21.3.15.4)为例，有些点是站在2024年的视角来看的。

+ **Join支持的局限性**

目前Global Join、Colocate Join是远远不能满足，大表join的需求的，特别是本人之前也重度使用过Spark SQL，对比之下像shuffle hash join之类的目前是还不支持的，但这个肯定后面都会支持的

+ 元数据改动方面，列调整比如说删除列

由于元数据每个part中都存储了，所以进行列删除的时候，需要扫描所有part文件，如果表较大的话，**将是一个非常漫长的过程**。这个对于埋点数据治理将会带来很多运维成本

+ 扩容或者缩容无法自动进行数据平衡

Hadoop中增减DataNode，是有相关机制进行自动数据平衡。但是ClickHouse需要手动迁移，这个如果节点多了，会非常痛苦。 而相较而言StarRocks在缩减或扩容资源时，只需一行命令，无需重启集群即可自动完成扩缩容，不会对稳定性造成影响

+ 冷热分离存储架构中，社区版的支持不够

当前版本只能在做到在**不同的磁盘中进行迁移**，比如说热数据存储在SSD，冷数据存储在HDD。比如说想要归档到OSS中(主要是指不支持迁移到阿里云、腾讯云等国内的)，目前是不支持的。这个问题就是需要不停的处理磁盘资源紧张的问题，一方面需要清理资源，另外一方面实在清理不掉就要挂载更多的磁盘

+ 存算分离的改造

Shared-Nothing，随着云原生技术的飞速发展，已经逐渐凸显出一些弊端，**典型就是计算资源和存储资源绑定**，而计算资源和存储资源的实际需求或者扩张频率可能是不同的。通过存算分离架构，将存储层与计算层解耦，使得存储资源和计算资源分别弹性扩展成为可能。一方面是扩展能力得到极大提升，另外一方面就是基于需求动态调整资源，成本也会得到一定程度的控制。

目前高峰的时候，ClickHouse资源瓶颈还是很大的，但是这个高峰在一个月中的占比又比较低，所以这个时候弹性资源就显得尤为重要。 不过ClickHouse目前已经退出了存算分离版本的引擎[SharedMergeTree Table Engine](https://clickhouse.com/docs/en/cloud/reference/shared-merge-tree)，但是只有在企业版中才能使用。



最后，最重要的一点感受就是，ClickHouse发展明显比国内的新兴OLAP比如说StarRocks、Doris等慢了很多，刚出来的以高性能著称，被国内各种公司大量使用。但最近经常会在各种社区看到类似使用StarRocks, Doris替换ClickHouse的分享，里面提到的ClickHouse的一些痛点也是实际存在的。一方面很欣慰国产OLAP做的越来越好，另外一方面也感叹ClickHouse这个早期"巨人"是不是越走越慢。



# 5. 参考

\> ClickHouse原理解析与应用

\> [ClickHouse replication](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication)

\> [C++ MergeTreeData.h](https://clickhouse.com/codebrowser/ClickHouse/src/Storages/MergeTree/MergeTreeData.h.html)

\> [ClickHouse内幕（1）数据存储与过滤机制](http://jackywoo.cn/ck-internal-data-store-and-filtering/)

\>  [github 解析Mark file](https://github.com/ClickHouse/ClickHouse/issues/6830)

\> [leetcode RPN](https://leetcode.cn/problems/evaluate-reverse-polish-notation/description/?utm_source=LCUS&utm_medium=ip_redirect&utm_campaign=transfer2china)

\> [github issue Query with Date type against table partitioned by DateTime doesn't work](https://github.com/ClickHouse/ClickHouse/issues/5131)

\> [性能全面提升！白山云基于StarRocks替换ClickHouse的数据库实践](https://zhuanlan.zhihu.com/p/665702530)

\> [ClickHouse 存算分离改造：小红书自研云原生数据仓库实践](https://zhuanlan.zhihu.com/p/655316816)

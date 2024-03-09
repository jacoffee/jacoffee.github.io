---
layout: post
category: database
date: 2022-10-20 15:29:03 UTC
title: 【ClickHouse基础】Zookeeper in ClickHouse
tags: [OLAP数据库研究、ClickHouse、Zookeper、Linearizable Write]
permalink: /database/clickhouse/zookeeper
key:
description: 本文主要总结了Zookeeper在ClickHouse中起到的作用
keywords: [OLAP数据库研究、ClickHouse、Vectorization Processing]
---

在ClickHouse Keeper出现之前，Zookeeper在ClickHouse集群中起到了非常大的作用，本文主要梳理了Zookeeper在ClickHouse中一些重要环节的的使用。

# 1. 使用场景

## 1.1 分布式表的结构信息存储

一般基于宏配置，创建分布式本地表在引擎处做如下配置:

```sql
create table test_local (
....
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
```

假设是2shard 2replica, 则会在Zookeeper中创建指定的路径：

```bash
/clickhouse/tables/table_uuid/01
/clickhouse/tables/table_uuid/02
```

同时该路径下，比如说/clickhouse/tables/table_uuid/01，下面会包括如下路径:

| 路径                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| /metadata            | 表元数据信息，索引列粒度、主键、分区键等                     |
| /columns             | 记录对应本地表的列信息，列名、字段类型                       |
| /replicas            | 保存副本名称，对应设置参数中的replica_name                   |
| ~~/leader_election~~ | ~~用于主副本的选举工作，主副本会主导MERGE和MUTATION操作（ALTER DELETE和ALTER UPDATE）。这些任务在主副本完成之后再借助ZooKeeper将消息事件分发至其他副本~~ |
| /blocks              | 记录Block数据块的Hash信息摘要以及对应的partition_id。通过Hash摘要能够判断Block数据块是否重复; partition_id，则能够找到需要同步的数据分区 |
| /quorum              | 记录quorum的数量，当至少有quorum数量的副本写入成功后，整个写操作才算成功。quorum的数量由insert_quorum参数控制，默认值为0 |
| /log                 | Shard中分片操作日志节点(INSERT、MERGE和DROP、PARTITION)，它是整个工作机制中最为重要的一环，保存了副本需要执行的任务指令。log使用了Z持久顺序型节点，每条指令的名称以log-为前缀递增，例如log-0000000000、log-0000000001等。 |

关于Shard中的leader replica, 这点从20.6已经发生了改变，或者21.12以后的版本实际上都是双主了，也就是多个Replica都可以接收写入，没有所谓的Leader Election。

```c++
// LeaderElection.h

/** Initially was used to implement leader election algorithm described here:
  * http://zookeeper.apache.org/doc/r3.4.5/recipes.html#sc_leaderElection
  *
  * But then we decided to get rid of leader election, so every replica can become leader.
  * For now, every replica can become leader if there is no leader among replicas with old version.
  */
void checkNoOldLeaders(Poco::Logger * log, ZooKeeper & zookeeper, const String path)
{
    /// Previous versions (before 21.12) used to create ephemeral sequential node path/leader_election-
    /// Replica with the lexicographically smallest node name becomes leader (before 20.6) or enables multi-leader mode (since 20.6)
    constexpr auto persistent_multiple_leaders = "leader_election-0";   /// Less than any sequential node
    constexpr auto suffix = " (multiple leaders Ok)";
    constexpr auto persistent_identifier = "all (multiple leaders Ok)";
    ....
}
```

# 1.2 分布式DDL协调

这个主要是指我们再运行一些分布式DDL(add column、drop column)的时候，需要借助Zookeeper进行一些协调工作，也就是将命令分发到不同节点，然后在节点本地执行。

分布式DDL在Zookeeper内使用的 根路径为:   `/clickhouse/task_queue/ddl`， 在config.xml 中通过 `distributed_ddl`。

```xml
<distributed_ddl>
	<!-- Path in ZooKeeper to queue with DDL queries -->
	<path>/clickhouse/task_queue/ddl</path>
</distributed_ddl>
```

比如说给分布式表加列的时候，我们通过会运行如下命令。

```sql
alter table test_all add column A on cluster xx_cluster
```

在此根路径之下，还有一些其他的监听节点，其中包括/query-[seq]，其是DDL操作日志，每执行一次分布式DDL查询，在该节点下就会新增一条操作日志，以记录相应的操作指令。

当各个节点监听到有新日志加入的时候，便会响应执行。DDL操作日志使用ZooKeeper的**持久顺序型节点(PersistentSequential)**，每条指令的名称以query-为前缀，后面的序号递增，例如query-0000000000、query-0000000001等。

大体流程就是:  接收执行命令的节点先在本地执行，然后推送DDL命令到相应的Zookeeper Path，同时需要负责监听执行结果，其它节点监听到节点改变之后，读取相应的命令执行，然后反馈执行结果，最后由执行节点负责收集结果。

# 1.3 分布式写入的数据同步

这里主要是INSERT过程中，借助Zookeeper的事件通知机制，多个副本之间会自动进行有效协同。 比如说以下面的Shard-Replica配置为例，internal_replication表示写入一个副本实例之后，剩下的数据复制由ReplicatedMergeTree引擎内部机制自行完成。

```xml
<shard>
    <!-- 由ReplicatedMergeTree复制表自己负责数据分发 -->
    <internal_replication>true</internal_replication>
	<replica>
		<host>node1.com</host>
		<port>9000</port>
	</replica>
	<replica>
		<host>node2.com</host>
		<port>9000</port>
	</replica>
</shard>
```

当执行下面的插入命令:

```sql
INSERT INTO TABLE test_all VALUES('A001')
```

首先会在执行节点插入相关的数据，写入临时文件夹中，然后往Replica对应Shard的log目录下的`/log`推送本次命令的日志，可以想象同一个Shard下面的两个Replica都会监听该目录，大致格式如下:

```bash
# Zookeeper路径
/clickhouse/tables/{uuid}/01/log/log-0000029825

format version: 4
create_time: xxxxx
source replica: xxxx
block_id: xxxx
get
partition_id
part_type: Compact
```

然后对应的副本监听了对应的目录，便会拉取相应的操作日志LogEntry，然后存放到自己的任务队列中`/clickhouse/tables/{uuid}/01/replicas/{cluster}-{shard}-{replica}/queue`，
很典型的异步解耦的操作，用于应对同时段大量的LogEntry处理。

# 2. 关于ClickHouse Keeper

由于Zookeeper使用过程中的一些痛点，官方已经在后续版本中使用Keeper -- 使用C++编写的基于RAFT的分布式协调框架且完全兼容Zookeeper的协议，也就意味着迁移的时候只需要将Zoookeeper的data snapshot以及日志导入到新的Keeper集群中，即可正常工作。

[altinity All About Zookeeper (And ClickHouse Keeper, too!](https://altinity.com/presentations/all-about-zookeeper-and-clickhouse-keeper-too-2)中提到了如下原因:

+ 项目本身发展考虑，Clickhouse本身应该包括所有它运行所需要的组件，正如Kafka 3.0移除了对于Zookeeper的依赖
+ Zookeeper这个项目社区相对没有那么活跃了(Old，not very actively developed)
+ Zookeeper本身存在一些诸如ZXID rollover(溢出问题)以及官网中提到的不支持线性读(doesn't provide linearizability guarantees for reads, because each ZooKeeper node serves reads locally)

# 3. 参考

\>  ClickHouse原理解析与应用实践

\>  [blog StorageReplicatedMergeTree源码注释 层面针对Zookeeper在CK中的作用进行了详细的描述]( https://clickhouse.com/codebrowser/ClickHouse/src/Storages/StorageReplicatedMergeTree.h.html )

\>  [Clickhouse Keeper官方文档](https://clickhouse.com/docs/en/operations/clickhouse-keeper/)


---
layout: post
category: bigdata
date: 2017-11-03 06:30:55 UTC
title: Spark Thriftserver使用总结与思考
tags: [批处理、分布式查询引擎]
permalink: /spark/thriftserver
key: 
description: 本文梳理了Spark ThriftServer的基本使用以及相关的思考
keywords: [批处理、分布式查询引擎]
---

Spark thriftserver是一个分布式查询引擎，有点类似于HiveServer2，可以利用Hive元数据以及表，借助Spark的分布式计算能力，提供较好的查询效率。

## 1.部署

本质上来讲thriftserver就是一个普通的Spark应用，不同的是，一般的我们通过spark-submit运行一个任务之后可能很快就执行完了，但是这个Spark任务是一个常驻任务，Driver会一直接受客户端的SQL查询并且在各个节点上运行最后回收结果给客户端。

部署需要注意的地方(on yarn):

<ul class="item">
    <li>
        <b>Metastore server的地址: </b>在生产部署中，metastore一般都会<a href="/hive/deployment" target="_blank">单独部署</a>
    </li>
    <li>
        <b>服务启动后的host和绑定的端口: </b>也就是为了客户端访问时使用
    </li>
    <li>
        <b>部署模式的选择: </b>一般是Yarn的client模式(<b>cluster模式暂不不支持</b>)
    </li>
    <li>
        <b>应用分配的资源: </b>driver的内存大小、executor个数以及核数、内存大小
    </li>
    <li>
        <b>应用使用队列: </b>一般是Yarn Scheduler中配置的，如果有子队列，配置只用写字队列名，不用写成A.B.C
    </li>
    <li>
        <b>额外的依赖: </b>比如读取Elasticsearch数据的elasticsearch-hadoop
    </li>
</ul>

下面是一个简单的部署脚本:

```bash
./start-thriftserver.sh -v \
    --hiveconf hive.metastore.uris=thrift://host:port \
    --hiveconf hive.server2.thrift.port=bind_port \
    --hiveconf hive.server2.thrift.bind.host=bind_host \

    --name "Thrift JDBC/ODBC Server" \
    --master yarn --deploy-mode client \
    --properties-file resource-allocation.conf \
    --queue "xxxx" \
    --jars "/xxx/elasticsearch-hadoop-2.3.4.jar"
```

```bash
# resource-allocation.conf
spark.default.parallelism=50
# join、groupby操作时的分区数
spark.sql.shuffle.partitions=10

spark.driver.memory=3g
spark.executor.instances=10
spark.executor.cores=1
spark.executor.memory=2g
# 限制SQL查询返回的数据量大小，避免撑爆Driver
spark.driver.maxResultSize=2g
```

## 2.基本使用

### 2.1 Beeline访问

这种方式和HiveServer2的beeline一样，语法基本和[Hive SQL](https://cwiki.apache.org/confluence/display/Hive/LanguageManual)差不多。

```bash
./beeline -u jdbc:hive2://bind_host:bind_port "" ""
```

### 2.2 JDBC访问

这种方式实际上将thriftserver当成了数据库，引入相应的Driver即可(**org.apache.hive.jdbc.HiveDriver**)。这样我们就可以在程序中通过代码的方式访问Hive中的表。

```bash
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version>1.2.1</version>
</dependency
```

```scala
val conn = DriverManager.getConnection("jdbc:hive2://bind_host:bind_port", "", "")
val statement = conn.prepareStatement("select * from xxx limit 2")
val rs = statement.executeQuery()
```

上述两种方式的查询详情都可以通过ResourceManager UI中的ApplicationMaster UI去查看。

## 3.经验总结

### 3.1 资源分配的选择

在Spark中，有静态资源分配(Static allocation)和动态资源分配(Dynamic allocation)两种资源分配方式。前者好理解，我们每次在启动Spark应用时，都会指定相关的资源(executor、core)。静态分配，会根据提交任务的等待时长(**schedulerBacklogTimeout**)来添加Executor，会根据Executor的闲置时间(**executorIdleTimeout**)来移除Executor。具体的实现可参照**org.apache.spark.ExecutorAllocationManager**。

静态分配需要一直占据分配的资源，比如说上面的启动脚本中使用了10个Executor，而动态分配只需要设定最小Executor和最大Executor，使用完成之后会释放，回到最小Executor。

看上去，貌似动态分配更好，但**动态分配的劣势在于Executor的计算能力不能被充分利用**，典型的表现就是后面新增的Executor分配的任务相对较少; 而静态模式下，任务的分配则相对均匀。

在实践中发现，同样的配置，静态资源分配的执行速度是优于动态资源分配的。

### 3.2 场景的选择

由于接触Spark先于Hive，一直有一个印象就是Spark速度优于Hive(在SQL on Hadoop方面，这当然是事实)，所以会在各种场景下都尽量使用Spark，但之前在生产中的一次操作让我对于它们的场景有了新的认识。

具体情况也就是[Hive中连表查询时的倾斜(skew)解决](/bigdata/skew/solution)中描述的，将连表(有倾斜情况)的结果存入HDFS中，此时Hive的优势就体现出来了。

<ul class="item">
    <li>
        <b>并行度控制: </b> Spark和Hive读取文件时，都有相应的切分规则然后形成Partition或者是Mapper。Hive可以通过<b>mapreduce.input.fileinputformat.split.maxsize</b>参数控制mapper数也就是并行数，但是thriftserver中并没有提供类似的机制来控制，只能根据<a href="/spark/hdfs/partition" target="_blank">默认切分规则</a>去生成。
    </li>
    <li>
        <b>输出文件数控制: </b>Hive在输出结果的时候可以根据输出文件大小结合相关参数来决定reducer数，也就是最终落到HDFS的文件数，这个优化可以很好的解决小文件问题，而且不需要显示的进行配置。当然如果需要显示配置，也可以通过<b>mapreduce.job.reduces</b>手动控制。而thriftserver只能通过<b>spark.sql.shuffle.partitions</b>设置join或者groupby之后的分区数来决定最终的文件数，如果没有上述的操作就无法直接控制了。
    </li>
</ul>

所以一般涉及到DML操作的(insert、update或者delete)，还是优先选择Hive。至于Spark thriftserver，还是让它安安静静的做一个分布式查询引擎吧。

## 参考

\> [HiveServer2](https://cwiki.apache.org/confluence/display/Hive/Setting+up+HiveServer2)

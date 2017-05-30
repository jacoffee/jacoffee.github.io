---
layout: post
category: spark
date: 2017-05-23 10:30:55 UTC
title: Spark基础之Spark和ES整合时的Guava版本问题解决
tags: [extraClassPath, lib priority, guava]
permalink: /spark/spark-on-yarn-es-guava
key: 
description: 本文介绍了在Spark On Yarn中使用ES时，如何解决guava版本问题
keywords: [extraClassPath, lib priority, guava]
---

在Spark(1.6.2) On Yarn模式中使用ElasticSearch 2.x版本(2.3.5)，会碰到Guava版本不兼容的问题。问题的根源是: Spark默认使用的是Guava 14.0，而ElasticSearch使用的是Guava 18.0并且使用了
`com.google.common.util.concurrent.MoreExecutors`中的**directExecutor**方法(在14.0中并没有)。

在[Spark-Yarn-Elasticsearch Guava冲突解决的帖子](https://stackoverflow.com/questions/37120174/guava-jar-conflict-when-using-elasticsearch-on-spark-job)中也提到了很多方法，但并不是每一个都有效。因此决定将自己解决的方法整理如下:

<b class="highlight">(1) 移除Spark Core的Guava依赖</b>

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_2.10</artifactId>
    <version>1.6.2</version>
    <exclusions>
    <exclusion>
      <!--exclude for elasticsearch-->
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
    </exclusion>
    </exclusions>
</dependency>
```

然后确保项目其它外部依赖不再引入Guava，整个项目中就只有ElasticSearch引入了Guava。这样在本地运行Spark任务就没有什么问题了。但是通过spark submit在Yarn上提交任务的时候，还是会发现下面的异常:

```bash
17/05/27 16:02:30 ERROR ApplicationMaster: User class threw exception: java.lang.NoSuchMethodError: com.google.common.util.concurrent.MoreExecutors.directExecutor()Ljava/util/concurrent/Executor;
java.lang.NoSuchMethodError: com.google.common.util.concurrent.MoreExecutors.directExecutor()Ljava/util/concurrent/Executor;
	at org.elasticsearch.threadpool.ThreadPool.<clinit>(ThreadPool.java:190)
	at org.elasticsearch.client.transport.TransportClient$Builder.build(TransportClient.java:131)
```

<b class="highlight">(2) 确保Spark Submit的时候Guava依赖的优先级高于Hadoop中相关的依赖</b>

Spark on Yarn模式中spark-submit启动时会读取Hadoop文件夹中的相关依赖，而这些依赖中也包括早期版本的Guava 11.0，因此在程序运行时该依赖被使用，导致上面的错误。所以我们需要做的就是"提高"Guava 18.0优先级，在**spark-defaults.conf**中提供如下参数即可:

```bash
spark.jars hdfs://localhost/user/allen/guava-18.0.jar
# Extra classpath entries to prepend to the classpath of the driver. 
spark.driver.extraClassPath guava-18.0.jar
# Extra classpath entries to prepend to the classpath of executors.
spark.executor.extraClassPath guava-18.0.jar
```

第一个属性在spark-submit中为**--jars**的形式，而在**spark-defaults.conf**需要改为**spark.jars**，它可以让某些依赖在集群中共享，在Spark源码中也有提到。然后通过extraClassPath 属性将它分别添加到driver和executor的classpth上，并且采用的是prepend(放在最前面)方式。

```bash
org.apache.spark.SparkContext

/**
Adds a JAR dependency for all tasks to be executed on this SparkContext in the future.
The `path` passed can be either a local file, a file in HDFS (or other Hadoop-supported
filesystems), an HTTP, HTTPS or FTP URI, or local:/path for a file on every worker node.

给所有在SparkContext中运行的任务添加依赖。
该路径可以是本地文件(存在于每个worker上面，通过file:///path的方式指定)，也可以HDFS路径

*/
def addJar(path: String) {}
```

通过下面的日志可以看出guava-18.0.jar被放在了**CLASSPATH的最前面**。

```bash
17/05/27 16:17:20 INFO SparkContext: Added JAR hdfs://localhost/user/allen/jars/guava-18.0.jar at hdfs://localhost/user/allen/jars/guava-18.0.jar with timestamp 1495873040984


===============================================================================
YARN executor launch context:
  env:
    CLASSPATH -> guava-18.0.jar<CPS>{{PWD}}<CPS>{{PWD}}/__spark__.jar<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/etc/hadoop<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/common/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/common/lib/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/hdfs/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/hdfs/lib/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/yarn/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/yarn/lib/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/mapreduce/*<CPS>/Users/allen/OpenSource/Hadoop-2.7.2/share/hadoop/mapreduce/lib/*
    SPARK_LOG_URL_STDERR -> http://192.168.31.228:8042/node/containerlogs/container_1495872092104_0005_01_000002/allen/stderr?start=-4096
```

另外还有一点我非常不明白，网上很多人使用[shaded jar](https://www.elastic.co/blog/to-shade-or-not-to-shade)方式来**移除Elasticsearch中有冲突的依赖**，但就本问题而言，它的关键是Spark-on-Yarn模式下引入了早期的Guava依赖，但ElasticSearch又不得不使用高版本的，所以移除有冲突的依赖并不能解决问题。

当然如果你对于升级Elasticsearch版本没有意见的话，直接升级到Elasticsearch 5.x，该系列版本已经移除了对于Guava的依赖。

## 参考

\> [elastic4s guava issue](https://github.com/sksamuel/elastic4s/issues/595)

\> [spark distributed classpath](https://community.cloudera.com/t5/Batch-Processing-and-Workflow/Spark-distributed-classpath/td-p/31320)

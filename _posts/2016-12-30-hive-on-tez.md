---
layout: post
category: hive
date: 2016-12-30 10:30:55 UTC
title: Hive on Tez配置
tags: [计算引擎，动态配置，依赖，日志输出]
permalink: /hive/tez
key: 07ac23070a7963b7e499d4ee16a222d0
description: 本文详细介绍了如何在hive中配置tez计算引擎
keywords: [计算引擎，动态配置，依赖，日志输出]
---

一般对于这种配置类的，官网都有相应的指南。但官网上的**Hive on Tez**指南太简略了(至少我配置的时候是这样的)，再加上今天在尝试的时候着实花了一番功夫，所以有些点还是值得写出来，以免大家重复踩坑。我的安装版本分别是**hive-2.1.1**，**tez-0.8.4**。

<b class="highlight">(1) 关于版本</b>

为了保证兼容性，[Hive和Tez](https://cwiki.apache.org/confluence/display/Hive/Hive-Tez+Compatibility)需要使用对应的版本。所以务必下载对应的版本，然后再开始配置。

<b class="highlight">(2) 关于Tez的安装</b>

我在网上以及[官网上](http://tez.apache.org/install.html)看到的都需要下载tez项目(**github**)然后进行Maven打包，生成`tez-version-SNAPSHOT-archive.tar.gz`。

但在实际操作中，我从官网上下载了[对应版本的tar.gz包](http://tez.apache.org/releases/apache-tez-0-8-4.html)之后就开始使用，具体流程如下:

由于Tez是新一代的计算引擎并且是构建在YARN之上的，所以一些依赖需要被集群共享，因此它包含的一些jar包需要上传到HDFS中。

解压`tez-0.8.4-tar.gz`，进入share目录，将`tez.tar.gz`放到HDFS中:

```bash
hdfs dfs -put /user/allen/ ~/OpenSource/tez-0.8.4/share/tez.tar.gz 
```

<b class="highlight">(3) 关于Hive的配置</b>

由于Hive本身是基于HDFS的，所以如果Hadoop的计算引擎替换成了Tez，那么它也可以使用，但是这种方式侵入性比较大，需要整个Hadoop集群都替换。另外一种，在Hive层面配置Tez。

首先在**${HIVE_HOME}/conf/hive-site.xml**中将计算引擎替换成**tez**，这个是全局层面的。当然也可以在进入hive console之后，配置Session层面的。

```xml
<property>
    <name>hive.execution.engine</name>
    <value>tez</value>
</property>
```

```bash
hive> set hive.execution.engine=tez;
```

然后在**${HIVE_HOME}/conf/**中新建`tez-site.xml`并配置如下属性，以让Hive使用:

```xml
# tez-site.xml
<property>
    <name>tez.lib.uris</name>
    <value>hdfs://localhost/user/allen/tez.tar.gz</value>
</property>
<property>
<property>
    <name>tez.lib.uris.classpath</name>
    <value>
        $HADOOP_CONF_DIR, 
        $HADOOP_HOME/share/hadoop/common/*, 
        $HADOOP_HOME/share/hadoop/common/lib/*, 
        $HADOOP_HOME/share/hadoop/hdfs/*, 
        $HADOOP_HOME/share/hadoop/hdfs/lib/*, 
        $HADOOP_HOME/share/hadoop/mapreduce/*, 
        $HADOOP_HOME/share/hadoop/mapreduce/lib/*, 
        $HADOOP_HOME/share/hadoop/yarn/*, 
        $HADOOP_HOME/share/hadoop/yarn/lib/*
    </value>
</property>
```

`tez.lib.uris.classpath`配置一些属性是因为Tez好像不能加载Hadoop的一些包，比较典型的是如果Hive中的查询涉及到LZO格式文件的读取，就会出现`com.hadoop.compression.lzo.LzoCodec not found`异常。

接下来这点是我目前还不能理解的，即使上面配置好了。还需要将解压后的**tez-0.8.4**中所有tez-*.jar以及**tez-0.8.4/lib**中的部分jar(我配置的时候是`commons-collections4-4.1.jar`)也放入`{HIVE_HOME}/conf/lib`下面。

运行hive命令，然后看到日志中出现如下信息就说明配置成功了。

```bash
INFO  [Tez session start thread]: impl.YarnClientImpl (:()) - Submitted application application_1483168665382_0018
INFO  [Tez session start thread]: client.TezClient (:()) - The url to track the Tez Session: http://192.168.31.65:8088/proxy/application_1483168665382_0018/
```

我在配置中遇到的问题大多集中在**NoClassDefFoundError**，像什么`org/apache/tez/dag/api/SessionNotRunning`，`org.apache.tez.dag.app.DAGAppMaster`，这类问题一般是Hive找不到相应的依赖，所以可以尝试在**${HIVE_HOME}/conf/lib**以及**tez-site.xml**中添加相应依赖来解决。

<b style="color:red">最重要的一点是</b> --  在配置的过程中，一定要时刻关注hive日志以及**Hadoop Resource Manager**(http://localhost:8088)中的任务日志，它们会输出很多有用的信息，可以帮助我们进行错误的排查。

Hive日志的位置在`${HIVE_HOME}/conf`中配置，同样如果这种开源项目你接触多了，你就会了解很多配置文件默认都是以模板(template)形式存在的，默认是`hive-log4j2.properties.template`，复制一份`hive-log4j2.properties`，配置`property.hive.log.dir`属性即可

```bash
property.hive.log.dir = /path/hive/log
```


## 参考

\> [配置Tez计算引擎](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Tez)

\> [Hive on Tez com.hadoop.compression.lzo.LzoCodec not found](http://www.inter12.org/archives/1099)

\> [Hive on Tez com.hadoop.compression.lzo.LzoCodec not found CDH5](http://stackoverflow.com/questions/23441142/class-com-hadoop-compression-lzo-lzocodec-not-found-for-spark-on-cdh-5)

\> [Tez安装](http://tez.apache.org/install.html) 

\> [Tez官网配置](https://tez.apache.org/releases/0.8.4/tez-api-javadocs/configs/TezConfiguration.html)
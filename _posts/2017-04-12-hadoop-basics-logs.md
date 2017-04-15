---
layout: post
category: hadoop
date: 2017-04-12 10:30:55 UTC
title: Hadoop基础之日志梳理
tags: [日志聚合，删除并上传]
permalink: /hadoop/logs
key: 
description: 本文介绍了Hadoop以及Yarn中各种日志的位置以及Spark应用日志的相关位置
keywords: [日志聚合，删除并上传]
---

Hadoop中有许多类型的日志，系统进程日志(NameNode, DataNode), HDFS审计日志(HDFS audit log)还有应用相关的日志(NodeManager中容器产生的日志)，下面主要介绍一下系统进程日志以及应用相关的日志。

## 系统进程日志

Hadoop中的主要进程包括NameNode、DataNode、JournalNode、ResourceManager以及NodeManager。
前三者属于Hadoop守卫进程，可以通过**hadoop-daemon.sh start xxx**启动；后两者属于Yarn守卫进程，可以通过**yarn-daemon.sh start xxx**启动。默认情况下，它们的日志位于**${HADOOP_HOME}/logs**文件夹下面，实际上每次脚本启动进程的时候都会在命令行中打印进程的日志地址。

```bash
allen:logs allen$ hadoop-daemon.sh start journalnode
starting journalnode, logging to ${HADOOP_HOME}/logs/hadoop-allen-journalnode-allen.local.out
```

另外为了更好的组织各种进程的日志，我们也可以通过在环境变量中或是**${HADOOP_CONF_DIR}/hadoop-env.sh**中分别配置`HADOOP_LOG_DIR`和`YARN_LOG_DIR`两个属性，这样对应进程的日志就会写到相应的文件夹中。

## 应用相关日志

这个主要指的是我们在ResourceManager UI中通过点击logs看到的Spark运行日志，实际上就是NodeManager中的Container日志(**注意和NodeManger本身的进程日志区分**)，默认地址为**${HADOOP_HOME}/logs/userlogs**。当然，我们也可以在yarn-site.xml中通过配置**yarn.nodemanager.log-dirs**属性来改变该地址。

```xml
<property>
    <name>yarn.nodemanager.log-dirs</name>
    <!--yarn.log.dir 指的就是配置的变量YARN_LOG_DIR-->
    <value>${yarn.log.dir}/userlogs</value>
    <description>
    应用的容器日志地址，每一个应用在运行的时候都会将日志放到该文件夹下面。
    每个应用会形成一个文件夹applicaiton_timestamp_appId，里面包含了各个运行容器的日志。
    </description>
</property>
```

在实际应用中，我们经常会配置一个属性**yarn.log-aggregation-enable**，它主要功能是决定是否开容器的日志聚合功能，作用效果表现在如下两个方面:

<ul class="item">
    <li>是否将上面的提到的Container日志上传到HDFS</li>
    <li>Spark UI的Executor Tab中各个Executor的<code>stdout、stderr</code>可否直接通过链接访问</li>
</ul>

```xml
<!--yarn-site.xml-->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
    <description>是否开启Container日志聚合功能</description>
</property>
<property>
     <name>yarn.log-aggregation.retain-seconds</name>
    <value>864000</value>
    <description>聚合日志的保留时间</description>
</property>
<property>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/yarn/logs</value>
    <description>聚合日志上传到HDFS的地址，默认为/tmp/logs</description>
</property>
```

当开启日志聚合功能后，应用结束时，NodeManager会对Container日志做如下处理。这个过程是属于NodeManager进程的行为，所以我们需要去上面提到的NodeManager日志中去查看相应的日志。具体的流程如下:

<ul class="item">
    <li>依次将节点上当前结束应用的所有容器中的stderr, stdout文件上传到<b>yarn.nodemanager.remote-app-log-dir</b>中</li>
    <li>然后依次删除节点中各个容器中的stderr, stdout</li>
    <li>在HDFS中所有的容器(包括driver)的日志都融合成了一个文件</li>
</ul>

概括来说就是如果开启了容器日志聚合功能，所有相关日志都会被上传到HDFS中。这样我们就可以通过`yarn logs -applicationId <application ID> -containerId <Container ID>`来对于各个容器中的日志进行排查和分析。

最后，我们在**spark-defaults.conf**配置的**spark.eventLog.dir**和**spark.history.fs.logDirectory**里面的存储的日志都是为Spark UI服务的(reconstruct the Web UI after the application has finished)。


## 参考

\> [HortonWorks简化YARN中的日志管理](https://hortonworks.com/blog/simplifying-user-logs-management-and-access-in-yarn/)
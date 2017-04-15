---
layout: post
category: hadoop
date: 2017-04-08 09:30:55 UTC
title: Hadoop基础之ResourceManager重启过程中的恢复机制
tags: [重启恢复，RPC通讯，Znode，Zookeeper高可用]
permalink: /hadoop/resourcemanager-restart
key: 8f7342ec2f931b0d7e7550f0625e1655
description: 本文介绍了ResourceManager重启过程的恢复机制以及具体配置
keywords: [重启恢复，RPC通讯，Znode，Zookeeper高可用]
---

周五的时候，向Hadoop集群提交了一个任务，运行的过程中ResourceManager进程由于某些原因突然中止了。通过**yarn-daemon.sh start resourcemanager**命令重启之后，发现之前的应用仍然顺利执行。猜想可能是由于ResourceManager重启恢复的某种机制才能保证了应用的继续运行。通过[官网中的ResourceManger Restart](https://hadoop.apache.org/docs/r2.7.2/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html)找到一些答案，现整理如下。

## 重启过程

为了降低ResourceManager重启带来的影响，它就必须在正常运行时去记录一些状态信息，这样在重启之后才能去获取并且构造对应状态的实例。这个过程经历了两个阶段的发展:

<ul class="item">
    <li>
        <p>
<b>不保留当前应用执行状态的重启(non-work-preserving restart): </b>当ResourceManager重启之后，它必须知道之前的应用(application)处于什么状态(ACCEPTED、RUNNING、FAILED、FINISHED等)。在Hadoop 2.4的时候，ResourceManager会存储应用的元数据(i.e ApplicationSubmissonContext)以及应用状态到某种存储介质中(Zookeeper, HDFS)等。重启之后，它会获取相应的信息，然后重新提交应用而不需要用户再次提交(如果应用在ResourceManager中止之前应用已经完成--FINISHED、FAILED、KILLED--则不会再次提交)。
        </p>
        <p>
        在ResourceManager挂掉之后，NodeManagers和客户端会持续发送心跳来检测它的状态。等到它重启之后，ResourceManager会发送重新同步的命令(re-sync)给所有的NodeManager，而当时的行为就是杀掉所有的NodeManager然后重启，并且重新在ResourceManager中注册。
        </p>
        <p>
        由此可见，之前应用不会在ResourceManager重启之后继续工作，而是重新开始了一次尝试(attempt)，所以该阶段也被称为<b class="highlight">不保留当前应用执行状态的重启</b>
        </p>
    </li>
    <li>
        <p>
        <b>保留当前应用执行状态的重启(work-preserving restart): </b>在ResourceManger重启机制的第一阶段，虽然可以做到应用的重新提交(re-kick application)，但任务的执行状态却没有得到延续。所以在2.6版本中该功能得到进一步加强。
        </p>
        <p>
        当ResourceManager重启之后，发送同步命令的时候，NodeManager中的
    Container会将自己的运行状态信息发送给ResourceManager，ApplicationMaster也会将自己的一些资源请求信息发送给ResourceManager，通过这些状态信息的收集来恢复任务的执行
        </p>
    </li>
</ul>

## 基本配置

为了存储应用元数据，我们需要使用一些可靠的外部存储系统，比如说Zookeeper、HDFS还有官网中提到的LevelDB。下面主要介绍Zoookeeper的相关配置:

<ul class="item">
    <li>启动恢复ResourceManager状态信息的机制</li>
    <li>配置具体的存储介质</li>
    <li>配置存储介质对应的地址</li>
</ul>

```xml
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.resourcemanager.store.class</name>
        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
    </property>
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <!--如果是集群则可以配置多个，以逗号分隔-->
        <value>localhost:2181</value>
    </property>
```

在应用运行的过程中，我们也可以通过查看<b>Znode</b>的结构来验证状态是否被存储。

```bash
# zk-shell localhost:2181 --run-once tree

├── rmstore
│   ├── ZKRMStateRoot
│   │   ├── RMAppRoot
│   │   │   ├── application_1491640108912_0001
│   │   │   │   ├── appattempt_1491640108912_0001_000001
...
```

通过ZKRMStateRoot键我们可以确认，ResourceManager的信息已经被存储。

## 实际操作

在验证的过程中(伪分布式集群模式 - Pseudodistributed mode，Hadoop 2.7.2)，我们可以先启动应用，等它进行RUNNING状态之后，
通过`yarn-daemon.sh stop resourcemanager`命令杀掉ResourceManager进程，这时候可以看到Client会尝试重连ResourceManager Server，然后再通过`yarn-daemon.sh start resourcemanager`，之后发现应用恢复之前的状态然后继续正常运行了。

```bash
17/04/08 16:20:09 INFO Client: Retrying connect to server: localhost/127.0.0.1:8032. Already tried 0 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
17/04/08 16:20:10 INFO Client: Retrying connect to server: localhost/127.0.0.1:8032. Already tried 1 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
```

在日志中，也能发现相关的重新同步的信息，但是任务的运行却没有被中断。

```bash
17/04/08 21:53:58 INFO TaskSetManager: Finished task 1.0 in stage 0.0 (TID 1) in 6334 ms on 192.168.31.65 (1/30)
17/04/08 21:53:58 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 6362 ms on 192.168.31.65 (2/30)
17/04/08 21:54:00 WARN AMRMClientImpl: ApplicationMaster is out of sync with ResourceManager, hence resyncing.
17/04/08 21:54:00 INFO TaskSetManager: Starting task 4.0 in stage 0.0 (TID 4, 192.168.31.65, partition 4,NODE_LOCAL, 2156 bytes)
17/04/08 21:54:00 INFO TaskSetManager: Finished task 3.0 in stage 0.0 (TID 3) in 2460 ms on 192.168.31.65 (3/30)
```

最后，如果启动恢复ResourceManager状态信息的机制之后，有两点需要注意:

ContainerId的格式会发生改变，因此就需要**spark-assembly-1.6.2-hadoopx.x.x.jar**中对应的Hadoop版本要能识别这种新的ContainerId，否则就会出现下面的异常。因为我开始在尝试的时候使用的是**spark-assembly-1.6.2-hadoop2.4.0.jar**，当ResourceManager重启之后，ContainerId使用了新的格式所以无法被之前版本的读取。

```bash
# 之前  container_集群时间戳_应用编号_尝试编号_容器编号
container_1491640108912_0001_01_000001
# 之后 container_e{RM重启次数}_集群时间戳_应用编号_尝试编号_容器编号
container_e01_1491640108912_0001_01_000001
```

```bash
java.lang.IllegalArgumentException: Invalid ContainerId: container_e01_1491640108912_0001_01_000001
...
Caused by: java.lang.NumberFormatException: For input string: "e01"
```

启动恢复ResourceManager状态信息的机制之后，如果借助Zookeeper存储状态信息，则首先需要启动Zookeeper。因为在ResourceManager启动时会尝试建立连接，如果Zookeeper没有启动则会导致ResourceManager启动失败。



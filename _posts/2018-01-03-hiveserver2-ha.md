---
layout: post
category: hive
date: 2018-01-03 10:29:03 UTC
title: Hiveserver2高可用实现以及配置
tags: [动态服务发现、服务治理、Znode节点监控]
permalink: /hive/hiveserver2-ha
key:
description: 本文介绍了Hiveserver2高可用的实现以及具体的配置
keywords: [动态服务发现、服务治理、Znode节点监控]
---

Hiveserver2(hive version 1.1.0)的高可用主要是指客户端进行连接时能被随机分配到当前活跃的实例中(server instances)，这种机制是通过动态服务发现来控制的(dynamic service discovery)。

在Hive实现中，主要借助Zookeeper来进行实例的管理以及请求的"代理"(proxy)，具体流程如下:

<ul class="item">
    <li>
    <b>实例注册: </b>Hiveserver2实例启动的时候，如果开启了动态服务发现，那么就会在Zookeeper中注册该实例并生成临时的Znode(ephemeral)，这一过程在<code>org.apache.hive.service.server.HiveServer2.startHiveServer2()实现</code>
    </li>
    <li>
    <b>连接实例: </b>
如果要使用动态服务发现，则通过beeline连接的时候的地址应该变为<b>jdbc:hive2://zookeeper-quoram/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2_zookeeper_namespace</b>，而Hiveserver2会自动解析该地址，然后从Zookeeper相应的namespace中注册的实例中随机获取server uri作为该客户端连接的服务器，这一过程在<code>org.apache.hive.jdbc.Utils.parseURL.configureConnParams实现</code>

<b style="color:red">不过当前版本， 如果客户端与Hiveserver2成功建立连接之后，HiveServer2挂掉了。客户端会话被销毁，所以不会重连到新Hiveserver2实例。</b>
</li>

<li>
<b>销毁实例: </b>当我们关闭某个实例时，与之关联的ZookeeperClient(CuratorFramework)也会被关闭，相应的znode会被删除，这样当客户端再次连接的时候，该实例就无法被连接上了。
</li>
</ul>

![](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.1/bk_hadoop-high-availability/content/figures/2/figures/Query_Ex_Path_With_ZK.png)

<p align="center">图1. Hiveserver2动态服务发现流程</p>

## 1.设计层面

由于需要监控Hiveserver2的状态以及对相应的znode操作，所以需要引用(composition)Zookeeper客户端以及Znode实例(ephemeral)

```java
private CuratorFramework zooKeeperClient;
private PersistentEphemeralNode znode;
```

节点注册地址对应的Znode需要注册相应的Watcher以应对节点的变化(如节点删除之后，对应的实例也应该被移除)

```java
public void addServerInstanceToZookeeper(HiveConf hiveConf) throws Exception {
    // set a watch on the znode
    zooKeeper.checkExists().usingWatcher(new DeRegisterWatcher()).forPath(znodePath)
}
```

## 2.具体配置

| 属性        | 默认值   |  解释  |
| --------   | :-----  | :----  |
| hive.server2.support.dynamic.service.discovery     | false |   是否支持客户端动态发现活跃实例(Hiveserver2)， 为了支持该功能，每个HiveServer2实例启动的时候会在Zookeeper中注册     |
|hive.server2.zookeeper.namespace|hiveserver2| 如果支持了动态服务发现，实例Znode的父Znode name|
|hive.zookeeper.quorum| 无 | Zookeeper集群对应的地址 host1:port1,host2:port2|

通过beeline连接时，由之前的

```bash
jdbc:hive2://host1:port1/default
```

改为

```bash
jdbc:hive2://host1:port1,host2:port2/default;serviceDiscoveryMode=zooKeeper;
zooKeeperNamespace=<hiveserver2_zookeeper_namespace>
```

## 参考

\> [IBM Hiveserver2 HA](https://www.ibm.com/support/knowledgecenter/en/SSPT3X_4.1.0/com.ibm.swg.im.infosphere.biginsights.admin.doc/doc/admin_HA_HiveS2.html)
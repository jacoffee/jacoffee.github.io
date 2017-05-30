---
layout: post
category: hive
date: 2017-05-26 10:30:55 UTC
title: Hive基础之Metastore部署
tags: [内嵌模式，服务层之间的boundary, Metastore server, JDBC]
permalink: /hive/deployment
key: 
description: 本文介绍了Hive的各种部署模式
keywords: [内嵌模式，服务层之间的boundary, Metastore server, JDBC]
---

今天在Spark Standalone集群中部署thriftserver的时候，发现自己对于HiveServer2、Metastore server等几个概念还不是很清晰，主要是对于Hive的几种部署模式还不是很清晰。它们的主要的区别在于Metastore的部署方式，它存储了表结构和分区(partition)的相关元数据。Metastore中有一张表叫**tbls**记录了所有hive表的创建时间、类型和拥有者等。

## 内嵌模式 - Embeded Mode

Metastore默认采用derby存储，这种模式一般用于单元测试并且同一时间只能有一个活跃用户(也就是启动的时候在**hive-site.xml**中配置的**hive.server2.thrift.client.user**)，还有一点就是它需要在你运行hive命令的目录下创建**Metastore_db**文件夹(初次运行的时候)。

在这种模式下，HiveCli、Metastore service和Derby都在同一进程中。

![metastor embeded mode](http://static.zybuluo.com/jacoffee/pduxxrj0knzr26bs6fxp3ttm/image.png)

## 本地模式 - Local Mode

在这种模式下，Metastore将表和分区的元数据存储在外部数据库中(MySQL, PostgreSQL等)，Metastore service通过JDBC访问数据库。

![metasotre service put data in databases](http://static.zybuluo.com/jacoffee/wy594cm7lc3ozxjrxg6hyqfd/image.png)

## 远程模式 - Remote Mode

这种模式将Metastore service抽取出来，作为单独的一层，运行在独立的JVM进程中。其它的服务借助Thrift network API(地址通过**hive-site.xml中的hive.metastore.uris属性**配置)来访问Metastore service，并不需要知道Metastore service使用的**外部数据库信息**。
这些服务包括HiveServer2、Hive Cli、Spark Thrift JDBC/ODBC Server(和HiveServer2类似，通过它来**让Spark作为Hive的执行引擎**)。
而Metastore service通过JDBC访问数据库(通过**javax.jdo.option.ConnectionURL**配置)，一般生产环境都采用这种模式。

![Metastore server as service](http://static.zybuluo.com/jacoffee/zwgcgibn450ge6fnsph4p9n2/image.png)

在本地模式中，具体的服务(如cli)和Metastore service还是运行在同一JVM中的，所以**需要在每一个节点上配置Metastore数据库连接信息**，而远程模式则很好的避免这一点，上面已经提到。

## 远程模式部署(hive 2.1.1)

<b class="highlight">(1) 启动Metastore service server</b>

```bash
hive --service metastore

# 监控在其它端口
hive --service metastore -p <port_num>
```

默认监听在9083端口，所以对于客户端暴露的地址(thrift network address)就是:

```xml
<property>  
  <name>hive.metastore.uris</name>  
  <value>thrift://hostname:9083</value>  
</property>
```

启动节点上**hive-site.xml**中的一些基本配置(<b class="highlight">注意将数据库对应的jdbc 依赖放入${HIVE_HOME}/lib中去</b>):

```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost/Metastore</value>
    <description>数据库连接URL</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>xxxx</value>
    <description>Metastore数据库(mysql)连接用户名</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>xxxx</value>
    <description>Metastore数据库(mysql)连接用户名的密码</description>
</property>

<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>JDBC Metastore连接数据库的driver</description>
</property>

<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
    <description>
        default location for Hive tables.
        Metastore中各种表数据的实际存放位置(HDFS路径)
    </description>
</property>
```

<b class="highlight">(2) 连接客户端的配置</b>

连接Metastore server的客户端(如HiveServer2)，既可以和server在同一机器上，也可以不在同一台机器。注意这个地方的角色转换，HiveServer2对于beeline来说是服务端，而对于Metastore server又变成了客户端。

```xml
<property>  
    <name>hive.metastore.warehouse.dir</name>  
    <value>/user/hive/warehouse</value>  
</property>

<property>  
    <name>hive.metastore.uris</name>  
    <value>thrift://hostname:9083</value>  
</property>

<property>
    <name>hive.server2.thrift.port</name>
    <value>10005</value>
</property>

<property>
    <name>hive.server2.thrift.bind.host</name>
    <value>hostname</value>
</property>

<property>
    <name>hive.server2.thrift.client.user</name>
    <value>xxxx</value>
    <description>连接HiveServer2的用户名</description>
</property>

<property>
    <name>hive.server2.thrift.client.password</name>
    <value>xxxx</value>
    <description>连接HiveServer2的用户名密码</description>
</property>
```

## 参考

\> [官方文档 Metastore管理](https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin)

\> [官方文档 HiveServer2](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2)

\> [Cloudera文档 Hive部署模式](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cdh_ig_hive_Metastore_configure.html) 
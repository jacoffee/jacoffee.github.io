---
layout: post
category: logging
date: 2017-07-08 06:30:55 UTC
title: Linux中用户应用的日志回滚
tags: [日志回滚，定时任务]
permalink: /logging/rotation
key: 
description: Linux系统针对用户应用日志回滚
keywords: [日志回滚，定时任务]
---

在Linux系统中，各种开源组件(httpd，mysql)和用户应用(java程序，python程序)会输出大量的日志，随着时间推移，它们会占用大量的磁盘空间，所以需要对于日志进行管理，一般是对日志进行回滚。由于开源组件一般都会提供回滚策略或者利用Linux系统自带的回滚工具，这里我们主要讨论**用户应用**的日志回滚。

## 1.日志的输出

以Java项目为例，一般都会使用使用日志框架(**log4j**)对于应用日志进行管理，要么输出到**控制台**，要么输出到配置的**日志文件**中。

对于第一种方式，我们一般会将日志**重定向**到指定的日志文件中。

```bash
# 普通java应用
java -jar xx.jar classname > xx.log
```

对于第二种方式，我们会同时在**log4j.properties**配置相应的回滚策略

```bash
# 默认放在项目根目录下面
log.dir=xxxxx.log

log4j.rootCategory=INFO, FILE
log4j.appender.FILE=org.apache.log4j.RollingFileAppender

# 通过java环境变量-Dlog.dir=xxxx.log传递，优先级高于上面的配置的默认地址
log4j.appender.FILE.File=${log.dir}

log4j.appender.FILE.ImmediateFlush=true
log4j.appender.FILE.Append=true

log4j.appender.FILE.MaxFileSize=20MB

# 保留的归档文件数，不包括当前正在被写入的日志文件
log4j.appender.FILE.MaxBackupIndex=5

log4j.appender.FILE.layout=org.apache.log4j.PatternLayout
log4j.appender.FILE.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
```

## 2.日志的回滚

上面提到的日志输出中，第二种方式实际上由日志框架接管了，从整体层面来看这种策略有如下两个不足之处:

<ul class="item">
    <li>
        当本地开发需要将日志输出到控制台时，需要额外配置。或者说<b>同时输出到控制台和日志文件中</b>，显然这不是我们想要的
    </li>
     <li>
         这种配置只在单个项目的层面生效，意味着我们需要对很多项目进行相应的回滚配置。如果使用的是开源的其它语言(Python)的项目，可能又要耗费一番功夫
     </li>
</ul>

基于以上两点再加上测试环境和生产环境一般是Linux(CentOS)系统，所以我们可以借助[logrotate](https://linux.die.net/man/8/logrotate)来统一管理用户应用的日志。

在`/etc/logrotate.d/`下面实际上已经有很多相关的日志回滚配置了:

```bash
-rw-r--r--. 1 root root  71 Jul 24  2015 cups
-rw-r--r--. 1 root root 139 Jul 24  2015 dracut
-rw-r--r--. 1 root root 185 Dec  9  2016 httpd
-rw-r--r--. 1 root root 192 May 26 23:20 jenkins
-rw-r--r--. 1 root root 267 Jul 24  2015 mcelog
-rw-r--r--. 1 root root 871 May  3 13:36 mysqld
-rw-r--r--. 1 root root 106 Jun  3  2015 numad
-rw-r--r--. 1 root root 329 Jul 17  2012 psacct
-rw-r--r--. 1 root root 121 Mar 22 11:30 setroubleshoot
-rw-r--r--. 1 root root 237 Jul 24  2015 sssd
-rw-r--r--. 1 root root 210 Dec 10  2014 syslog
-rw-r--r--. 1 root root  87 Jul 24  2015 yum
-rw-r--r--. 1 root root 132 Apr 24 02:03 zabbix-agent
-rw-r--r--. 1 root root 132 Apr 24 02:03 zabbix-server
```

我们需要考虑如下几点:

<ul class="item">
    <li>
        <code>回滚间隔</code>: 按天、按周、按月
    </li>
    <li>
        <code>文件大小阈值</code>: 文件超过多大的时候才回滚，这需要根据每天产生的日志量来决定
    </li>
    <li>
        <code>归档文件的保留个数</code>: 每次回滚都会产生一个归档文件，我们可以根据磁盘空间规划以及需要日志回溯的需求来确定保留多少个，保留个数并不包括正在被写入的文件
    </li>
    <li>
        <code>回滚策略</code>: 默认情况是将正在写入的文件xx.log重命名xx1.log，然后新建xx.log。这种方式对于通过<b>重定向控制台日志</b>形成日志文件的那种方式是有问题的，后面会提到。另外一种方式就是copytruncate，即<code>cp xx.log xx1.log</code>然后清空**xx.log**
    </li>
</ul>

**重定向控制台日志**指的是在项目中将日志文件输出到控制台，然后启动项目时将日志重定向到指定日志文件中(Java项目名为metrics-report)。

```bash
# 以hdfs用户启动
java -jar metrics-report.jar classname > /srv/logs/metrics-reporter/metrics-report.log
```

如果日志文件都统一进行`/srv/logs`文件夹的话，则提前新建相应的文件夹，同时修改用户权限(如果有必要的)，因为`/srv/`文件夹下的默认都是root组。

```bash
mkdir /srv/logs/metrics-reporter 
chown -R hdfs:hdfs metrics-reporter/
```

### 2.1 日志回滚配置

对于日志回滚的配置，我们可以每一个应用配置一个，也可以将多个应用的配置在一个文件里面。下面的案例是将所有用户的项目配置在一个文件里面:

```bash
cd /etc/logrotate.d/
touch user-application

# local的全局配置，global的全局配置在/etc/logrotate.conf中
# 配置日期扩展，而不是直接在后面拼接数字
size 20M
dateext
# 回滚的日志为 logname-2017-07-08
dateformat -%Y-%m-%d 
# 复制日志文件然后清空，通过重定向控制台日志形成日志文件的方式必须使用
copytruncate
# 如果为空则不回滚
notifempty
    
/srv/logs/metrics-reporter/metrics-reporter.log {
    daily
    size 10M
    rotate 5
}
```

像Spark程序我们可以以**文件夹**为单位来配置日志回滚:

```bash
/srv/logs/realtime-computation/*.log {
    daily
    size 10M
    rotate 5
}
```

上面的配置可以作用于所有realtime-computation文件夹下后缀名为log的文件。


### 2.2 日志回滚配置的验证

配置完成之后，我们可以通过下面的命令进行验证:

```bash
# -d 表示debug模式也就是只是测试配置而不用真正运行
logrotate -d /etc/logrotate.d/user-application

reading config file /etc/logrotate.d/user-application
reading config info for /srv/logs/metrics-reporter/metrics-reporter.log 

Handling 1 logs

rotating pattern: /srv/logs/metrics-reporter/metrics-reporter.log  10485760 bytes (5 rotations)
empty log files are not rotated, old logs are removed
considering log /srv/logs/metrics-reporter/metrics-reporter.log
  log does not need rotating

```

上面的日志输出显示logroate已经被正确配置(回滚阈值10M，保留5个归档文件)，测试的时候由于没有达到回滚要求所以并不会回滚。

但是如果想要真正运行，也可以通过下面的命令强制日志回滚(<b style="color:red">不过设置的一些阈值条件也会被忽略</b>)

```bash
# -f --force Tells logrotate to force the rotation, even if it doesn't think this is necessary. 
logrotate -f /etc/logrotate.d/user-application

# 注意回滚的日志格式
-rw-rw-r-- 1 hdfs hdfs 21228 Jul  8 16:34 metrics-reporter.log
-rw-rw-r-- 1 hdfs hdfs   348 Jul  8 15:33 metrics-reporter.log-2017-07-08
```


## 参考

\> Linux Shell Scripting Cookbook 2nd Edition

\> [man logrotate](http://linux.die.net/man/8/logrotate)

\> [测试日志回滚的配置是否正确](https://stackoverflow.com/questions/2117771/is-it-possible-to-run-one-logrotate-check-manually)
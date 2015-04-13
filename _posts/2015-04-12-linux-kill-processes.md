---
layout: post
category: scala
date: 2015-04-12 09:51:22 UTC
title: Linux基础之杀死进程(kill processes)
tags: [进程, PID, 信号补货, Trap, Trap Signal]
permalink: /linux/kill-processes/
key: c4d6d33810e11258a1d8586ce1008b27
description: "本文罗列了Linux/Unix中常见的杀死进程的方法"
keywords: [进程, PID, 信号补货, Trap, Trap Signal]
---

接触Ubuntu的时候就一直被杀死进程的各种方法困扰，后来转到Mac的时候感觉还是没有掌握那几种杀死进程的命令，所以决定系统的整理一下。

# 问题
Linux/Unix下有哪几种杀死进程的方法？

# 解决

<1> 进程号

每一个进程，简单的理解就是某个运行着的程序实例。比如说我们启动Chrome，系统就会自动分配一个unique process identification number(PID)，不过Chrome好像每开一个窗口就会启动一个进程。以Chrome为例，我们可以运行下面的命令找出当前进程所对应的进程号。

```bash
pidof Chrome # for linux 

pgrep Skim # for linux/Unix
ps aux | grep Chrome # for Unix(mac 不支持pidof)
allen  2586   0.0  0.0  2432772    636 s003  S+    5:42PM   0:00.00 grep chrome
```

上面的2586就是Chrome的进程号。

<2> 杀死进程

(1) 哪些进程你能够杀死

& 杀死所有你拥有权限的进程

& 只有root用户才能杀死系统级别的进程

& 由其它用户启动的进程只能由root用户杀死

(2) 基本语法:

```bash
kill [signal] PID
killall process_name
pkill process_name
killall -u USERNAME process_name
```

我们在terminal中输入 

```bash 
kill -l
```

会获取所有的Signal, 不过一般常用的是如下几个:

& SIGINT(2): 相当于我们按Ctrl+C时向进程发出的指令

& SIGKILL(9): 强制杀死进程

& SIGTERM(15): 默认杀死进程的方式

```bash
pgrep Skim # 2672
kill -9 2672

killall Skim

pkill Skim
```

<3> 捕获Signal并给出回应

实际上kill -l列出的所有的Singal都会有相应的Handler，当系统接受到这些Signal之后，会对进程做相应的处理。但是，我们也可以自己给Signal绑定相应的Handler。

```bash
#!/bin/bash

function handler() {
    echo Hey, received signal name: SIGINT
}

echo My Process ID $$ # $$ is used to obtain the current process name

trap 'handler' SIGINT

while true;
do 
    sleep 1;
done
```

运行如下命令:

```bash
allen:Script allen$ bash kill.sh 
My Process ID 910
^CHey, received signal name: SIGINT
```

其实上面的trap的第二个参数也可以是直接的命令，如trap 'echo Signal trapped' SIGINT。

# 结语
在验证trap signal的时候，我碰到了一个问题:

如果使用source kill.sh 或者是. kill.sh的方式启动的话，Ctrl+C是无法触发trap的第二个参数对应的handler。我知道source 或者.启动script的话是在当前的process中的，而sh或者是bash启动script则会新启动一个进程并且将执行结果返回到当前的process中。目前还没有得到解决，如果解决了，会回来补充的。

# 参考
<1> [Kill Process](http://www.cyberciti.biz/faq/kill-process-in-linux-or-terminate-a-process-in-unix-or-linux-systems/)

<2> Linux Shell Scripting CookBook

<3> [Kill -9原理](http://unix.stackexchange.com/questions/5642/what-if-kill-9-does-not-work)

<4> [source, sh, bash, . 运行Script的不同](http://superuser.com/questions/176783/what-is-the-difference-between-executing-a-bash-script-and-sourcing-a-bash-scrip)
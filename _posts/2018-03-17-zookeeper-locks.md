---
layout: post
category: distribution
date: 2018-03-17 15:29:03 UTC
title: 【分布式锁实战】Zookeeper分布式锁实现剖析
tags: [分布式锁，事务，critical section，监视器锁]
permalink: /zookeeper/distributed-locks-implementation
key:
description: 本文介绍了分布式锁的基本概念以及如何基于Zookeeper去实现
keywords: [分布式锁，事务，critical section，监视器锁]
---


在单进程应用中，如果多个线程同时访问共享变量时，在Java中我们通常会使用内置锁(synchronized)或显示锁(Lock)来协调多个线程的访问。但如果多个进程想要同时访问某个共享资源，但同一时刻只允许某个进程去访问该资源，这时候我们就需要一个**外部协调者**，来管理这些进程的操作，也就是我们下面需要提到的分布式锁。

在[Zookeeper高级应用](https://zookeeper.apache.org/doc/r3.1.2/recipes.html#sc_outOfTheBox)中就提到了分布式锁的实现，下面我们以写锁(**write lock**)实现为例，通过具体代码来讲解整个过程。

## 具体实现

首先我们需要注意的是，每一个ZookeeperClient是和对应的进程绑定的，也就是进程获取锁和释放锁的过程都是由对应的ZookeeperClient来实现的。在实际实现中，我只使用了主线程，所以也就是主线程去负责获取锁和释放锁。

写锁(write lock)一般也称为排它锁(mutex)，所以某一个时刻只能有一个进程获取该锁，而其它进程则要等待持有锁的进程释放之后才能再次尝试获取。

在Zookeeper中主要是基于Znode去实现的，总结起来就是在创建的Znode中总是index(其实就是EPHEMERAL_SEQUENTIAL源码中提到的**monotonically increasing number**)最小的那个获得锁，由于是瞬时节点，所以导致各个进程轮流获取锁。

### 获取锁的过程

+ 1.定义Lock路径的根路径以及lockname: **basePath=/\_locknode\_** 以及 **lockname=lock-**

+ 2.通过create()方法创建节点，模式为CreateMode.EPHEMERAL_SEQUENTIAL，也就是瞬时(**客户端如果连接断开会自动删除，等同于释放了锁**)且名字中带有全局递增的一个数字，创建成功之后会返回该路径的实际地址path，假设为**/\_locknode\_/lock-0000000001**

+ 3.然后获取basePath的子节点路径(**getChildren()**)数字部分，我们称之为索引，如0000000001，0000000002。这时进行对比，如果第二部分创建的路径的后缀是所有索引中最小的，则当前进程成功获取锁

+ 4.如果不是最小的，则下一个获取锁(如果没有意外)应该就是第二小的索引全路径对应的进程(**next_lowest_index_path**)，这时候我们调用**getData**并且监听在该路径上，即注册Watch(如果节点发生变化则唤醒当前等待线程)，然后将自己挂起(即调用wait())。 该环节在两种情况下会结束，地址不存在了则会抛**KeeperException.NoNodeException**或者是节点状态发生改变，然后被唤醒，无论哪种情况，我们都需要重新回到2对应的环节(<b>当然前提是一直尝试获取锁</b>)

```java
// 核心代码
public String attemptLock() throws Exception {
    boolean hasTheLock = false;
    boolean doDeletePath = false;
    
    String path = String.format("%s/%s", basePath, lockName);
    String ourPath = null;
    
    try {
      // /_locknode_/lock-0000000001
      ourPath = createsTheLock(path);
    
      // try to get the lock until we can not    
      while (!hasTheLock) {
        List<String> children = zooKeeper.getChildren(basePath, false);
        LockCheckResult lockCheckResult = getLockCheckResult(children, ourPath);
    
        if (lockCheckResult.isLocked()) {
          System.out.println(
              String.format(
                  "Thread %s in %s get lock on path",
                  Thread.currentThread().getName(), nameForTest, ourPath
              )
          );
          return ourPath;
        } else {
          String nextPathToWatch = lockCheckResult.getPathToWatch();
          System.out.println(" nextPathToWatch " + nextPathToWatch);
    
          // remember we have to get the object monitor before call wait()
          synchronized (this) {
            try {
              zooKeeper.getData(nextPathToWatch, watcher, null);
              String msg =
                  String.format(
                      "Thread %s in %s enters lock-waiting pool",
                      Thread.currentThread().getName(), nameForTest
                  );
              System.out.println(msg);
              wait();
            } catch (KeeperException.NoNodeException e) {
              // If the nextPathToWatch is gone, try next round of lock
            }
          }
        }
      }
      return ourPath;
    } catch (Exception e) {
      doDeletePath = true;
      throw e;
    } finally {
      if (doDeletePath) {
        deletePath(ourPath);
      }
    }
}
```

### 释放锁的过程

释放锁就相对简单一些，拥有锁的进程直接将**对应的路径删除**或者是对应的ZookeeperClient退出、Session过期时，路径会被Zookeeper Quoram自动删掉都会导致锁释放。无论如何，一点要确保锁被释放，以避免发生死锁。

本文实现的完整版，可参考[DistributedMutexLock.java](https://github.com/jacoffee/codebase/blob/master/src/main/java/com/jacoffee/codebase/zookeeper/DistributedMutexLock.java)

## 实现过程中需要注意的问题

1.在各种Zookeeper节点操作的时候一定要注意**org.apache.zookeeper.KeeperException**的处理

2.由于当前进程(在本例中就是主线程)需要被挂起，等待唤醒。一定要记得先获取对象监视器锁，然后再进行操作，也就是上面的**synchronized (this)**

3.上面强调过路径删除(即锁释放)的重要性，所以要确保无论是异常抛出还是人为终止(如Ctrl + C)，都需要确保删除操作执行; 异常可以通过finally来解决，人为终止可以通过注册ShutdownHook来解决，不过这种情况只能关闭ZookeeperClient，<b style="color:red">不能释放锁</b>，因为ShutdownHook是在另外的线程中执行的。

```java
private static AtomicBoolean zkClosed = new AtomicBoolean(false);

private static void registerShutdownHook(ZooKeeper zk) {
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
      System.out.println("Clean up in the shut down hook !!!");
      
      // make sure we do not run zk.close twice
      if (zkClosed.compareAndSet(false, true)) {
        if (zk != null) {
          try {
            zk.close();
          } catch (Exception e) {
            // just ignore
          }
        }
      }
    }));
}
```

本质上，自己借助Zookeeper实现分布式锁，只是为了理解官网提到的实现机制。在实际开发中，如果遇到需要使用分布式锁的场景，不妨尝试一下[apache curator](http://curator.apache.org/curator-recipes/shared-reentrant-read-write-lock.html)，Zookeeper的客户端框架，有点类似于Guava之于Java。另外需要注意，[Curator和Zookeeper版本兼容问题](https://stackoverflow.com/questions/35734590/apache-curator-unimplemented-errors-when-trying-to-create-znodes)。


## 参考

> [分布式锁实现](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

> [twitter commons 通过Zookeeper实现分布式锁](https://github.com/twitter/commons/blob/master/src/java/com/twitter/common/zookeeper/DistributedLockImpl.java)

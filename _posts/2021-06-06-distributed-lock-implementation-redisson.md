---
layout: post
category: distribution
date: 2021-06-06 15:29:03 UTC
title: 【分布式锁实战】Redisson实现剖析
tags: [分布式锁，事务，Redisson]
permalink: /distributed-lock/redisson
key:
description: 本文主要剖析Redisson分布式锁实现的原理
keywords: [分布式锁，事务，Redisson]
---

# 1. 基本使用

我们知道`java.util.concurrent.locks.Lock` 定义了Java中实现显示锁的规范，而Redisson中的Lock实现了它，因此我们可以通过lock和unlock来加锁和释放，非常简洁和方便。

```
 # Redis使用环境
 master: redis-cli -h xxxxxx -a xxxxxx
 slave: redis-cli -h xxxxxx -a xxxxxx
  public static void main(String[] args) {
         Config config = new Config();
         // use "rediss://" for SSL connection
         MasterSlaveServersConfig masterSlaveServersConfig = config.useMasterSlaveServers();
         masterSlaveServersConfig
                 .setPassword("xxxxxx")
                 .setDatabase(30)
                 .setMasterConnectionPoolSize(10)
                 .setMasterConnectionMinimumIdleSize(10)
                 .setMasterAddress("redis://xxxxxx:6379")
                 .addSlaveAddress("redis://xxxxxx:6379");
 
         RedissonClient redissonClient = Redisson.create(config);
         RLock rlock = redissonClient.getLock("jacoffee-lock");
         try {
             // 对应的数据库会创建名为jacoffee-lock，类型为none
             rlock.lock();
             System.out.println(getIpAddress() + " get the lock");
             TimeUnit.SECONDS.sleep(20);
         } catch (SocketException | InterruptedException e) {
             e.printStackTrace();
         } finally {
             rlock.unlock();
         }
     }
```

通过debug,  我们可以从上面这段简单代码得出如下信息:

+ **rlock.lock()**方法成功获取锁之后，会在对应的Redis数据库创建类型为**hash，key为jacoffee-lock**，键值对为**f9d9be78-5d47-4975-b95e-da0753d2b0c2:1 ----> 1**

```
 xxxxxx:6379[30]> type 'jacoffee-lock'
 hash
 xxxxxx:6379[30]> hgetall "jacoffee-lock"
 1) "f9d9be78-5d47-4975-b95e-da0753d2b0c2:1"
 2) "1"
```

+ 由于debug导致主进程阻塞，有可能导致锁过期，最后再去检查的时候，会报当前线程未持有锁，也就是`释放锁的时候的身份验证`

```
 Exception in thread "main" java.lang.IllegalMonitorStateException: attempt to unlock lock, not locked by current thread by node id: ecdd57d7-a505-42f4-867b-c9c35ba1f2d8 thread-id: 1
     at org.redisson.RedissonBaseLock.lambda$unlockAsync$1(RedissonBaseLock.java:312)
     at org.redisson.misc.RedissonPromise.lambda$onComplete$0(RedissonPromise.java:187)
     at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:578)
     at io.netty.util.concurrent.DefaultPromise.notifyListenersNow(DefaultPromise.java:552)
     at io.netty.util.concurrent.DefaultPromise.notifyListeners(DefaultPromise.java:491)
```

下面我们就以加锁到释放锁的流程来梳理下<<【分布式锁实战】实现思路分析>> 在Redisson中是如何体现的。下图是核心流程图:

![redisson-lock.png](/static/images/charts/2021-06-06/redisson-lock.png)

# 2. 加锁以及可重入锁

+ **定义唯一客户端标识**:  UUID + 当前获取锁定线程ID
+ **通过hash key以及hash key field**是否存在来确定当前加锁是否成功， 主要是通过lua脚本来实现，这个在Redisson中的有大量运用。

`hash key`:  jacoffee-lock

`hash key field`:  UUID:threadId

`hash key value`:  获取锁的次数，可重入的体现

首先， 判断jacoffee-lock这个可以是否存在， 如果不存在则尝试添加key-value对(UUID:threadId -- 1),   成功则获取锁； 如果jacoffee-lock这个key存在，则`尝试更新获取锁的次数(可重入实现)`，成功则获取锁，反之则失败, 返回当前hash key的ttl。

```
 -- 核心的lua脚本
 -- KEYS[1] jacoffee-lock
 -- ARGV[1] 30s
 -- ARGV[2] f9d9be78-5d47-4975-b95e-da0753d2b0c2:1
 -- 后面的是否加锁是否成功也是根据，这段脚本的返回值是否null进行判断， null成功
 if (redis.call('exists', KEYS[1]) == 0); then
     redis.call('hincrby', KEYS[1], ARGV[2], 1);
     redis.call('pexpire', KEYS[1], ARGV[1]);
     return nil; --- java中的null
 end;
 
 if (redis.call('hexists', KEYS[1], ARGV[2]) == 1)); then
     -- 可重入的体现
     redis.call('hincrby', KEYS[1], ARGV[2], 1);
     redis.call('pexpire', KEYS[1], ARGV[1]);
     return nil
 end;
 
 return redis.call('pttl', KEYS[1]);
```

# 3. 加锁过程中的故障切换

在《【分布式锁实战】实现思路分析》中提到过当Redis中**主从发生切换**的时候，可能导致重复获取锁。Redisson基于Redis中同步复制([WAIT](https://redis.io/commands/wait))机制解决了该问题，不了解Redis WAIT机制可以查看官网的相关文档。

核心机制就是:  通过lua脚本成功加锁之后，跟一个WAIT命令，获取锁之后要同时检查有多少副本响应。比如说目前是一主两备，那么就需要WAIT在指定时间内返回2，才认为加锁成功。至少目前版本[3.16.7](https://github.com/redisson/redisson/milestone/131)是不支持指定需要个副本返回的，所有配置的副本都需要"响应"。

实际上，本人在实验过程中发现了Redisson在这个过程中的一个bug。 [RLock syncedSlaves in BatchResult is not checked](https://github.com/redisson/redisson/issues/3994#issuecomment-992343065), 简单来说就是作者在实现的时候，并没有对于WAIT在指定时间内响应的副本数进行检查。

# 4. 加锁重试

分为**无限制重试**和**有一定超时的重试**，虽然暴露的接口不同，但其实核心逻辑都一样，都是第一次尝试获取锁失败之后，然后不断的调用获取锁的方法去获取锁。

+ 无限制重试

```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
    
    // 死循环
    while (true) {
        ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            break;
        }
        
        ...
    }
}
```

+ 带超时时间重试:  每次循环之后，减去对应的时间，直到超时时间到，然后退出循环。

```java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {

    // 等待时间
    long time = unit.toMillis(waitTime);
    
    while (true) {
        long currentTime = System.currentTimeMillis();
        ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            return true;
        }

        // 检出每次获取锁耗费的时间
        time -= System.currentTimeMillis() - currentTime;
        // 最终如果小于0，则放弃尝试
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
        
    }
    
}
```

# 5. 锁续期

一般加锁都是对于共享资源更改， 所以一般需要指定锁的有效期，不可能一直持有。 Redisson中同时支持设置过期时间和不设置过期时间,  所以锁续期一般是针对后者，主要为了让后者主动释放锁，避免出现锁因为超时而自动释放:

+ **设置了过期时间的(leaseTime)**： 获取锁成功之后，直接设置hash key的过期值

+ **没有设置过期时间的**： 需要引入锁续期机制，也就是我们经常听到的看门狗(watch dog机制) -- **本质上就是一个更新redis key过期时间的任务且循环调用**,  内部默认的过期时间是30s, 也就是internalLockLeaseTime

```java
private T RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    
     // 如果设置了持有锁的时间
    if (leaseTime != -1) {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
    } else {
        ttlRemainingFuture = tryLockInnerAsync(
            waitTime, internalLockLeaseTime,
            TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN
        );
    }
    
	ttlRemainingFuture((ttlRemaining, e) -> {
        if (e != null) {
            return;
        }
        
        /**
        * 其它进程去加的时候 不为null 返回的是ttl值
        */
        // lock acquired
        if (ttlRemaining == null) {
            if (leaseTime != -1) {
               // 设置了过期时间，成功获取锁之后，更新内部维护的过期时间
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                // 没有设置了过期时间，成功获取锁之后，则进行锁续期机制
                scheduleExpirationRenewal(threadId);
            }
        }
    })
    
}
```

这里主要说明下续期机制的lua脚本:  **如果当前客户端持有锁，则直接更新对应field的过期时间**。

```lua
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
	redis.call('pexpire', KEYS[1], ARGV[1]); 
    return 1; 
end;
return 0;
```

# 6. 锁释放

与获取锁相对应的，就是锁的释放，也是分布式锁设计中很重要的环节，主要分为**主动释放**和**被动释放**

## 6.1 主动释放

调用unlock方法，移除对应的hash key,  `特别要注意身份检查`，只有当前锁的持有者才有资格删除;  `可重入锁的释放`,  每调用一次unlock,  将相应的field值减1，只到为0才算彻底释放

```lua
// field不存在了，说明当前客户端已经不再持有锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
	return nil;
end

// 尝试释放，减去对应的数值
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
if (counter > 0) then 
    redis.call('pexpire', KEYS[1], ARGV[2]); 
else
    redis.call('del', KEYS[1]);
    // 涉及到pubsub通知机制，通知其它节点，当前节点已经释放锁
    redis.call('publish', KEYS[2], ARGV[1]); 
	return 1; 
end;

return nil;
```

## 6.2 被动释放

设置过期时间的情况下，Redis key自动到期然后释放锁;  没有设置过期时间的情况下，客户端down掉，续期的任务自动终止，最终key因为续期设置过期时间(internalLockLeaseTime)到了之后同样过期；

# 7. 参考

\> [Redisson官网](https://redisson.org/)

\> [Redisson实战 分布式对象](https://github.com/redisson/redisson/wiki/7.-Distributed-collections)




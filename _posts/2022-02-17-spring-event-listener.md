---
layout: post
category: spring
date: 2022-02-17 09:29:03 UTC
title: 【Spring实战】事件发布和订阅机制
tags: [事件发布、订阅机制]
permalink: /spring/event-listener
key:
description: 本文介绍了Spring中的事件发布和订阅机制
keywords: [事件发布、订阅机制]
---


# 1. 是什么

Spring内部基于**观察者模式**提供了一套事件发布和订阅机制，可以简单理解为内存消息队列，核心目的在于解耦。 它可以让我们将内部的业务逻辑抽象成事件，

同时关心该事件的使用方，会提前以`EventListener`注册自己。发布的时候，会通过内部维护Listener,  依次广播出去。

# 2. 核心组件

## 2.1 ApplicationEvent -- 事件源

应用事件的抽象，业务需要封装的对象都需要实现该类。

```java
public abstract class ApplicationEvent extends EventObject {
	
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}

}
```

## 2.2 EventListener -- 逻辑处理者

在Spring中以注解的形式出现在组件上。标记某个组件作为事件的listener，也就是进行实际的逻辑处理。

## 2.3 ApplicationEventMulticaster -- 事件广播

用于管理一系列的ApplicationListener，当收到相关事件时，广播给相应的ApplicationListener。可想而知，在Spring boot服务启动的时候，会扫描所有组件中的@EventListener以及相应的方法，
同时会维护指定事件对应哪些Listener，也就是需要触发哪些逻辑调用。


# 3. 基本案例

新用户创建之后，发送其它相关的邮件、通知之类。 新用户创建之后，发布UserCreateEvent，相关Listener订阅之后做相关处理。当然，这个肯定是适用于规模不大的，用户量大的一般会使用消息队列。

+ 指定位置发布事件

```java
// 用户Service
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Transactional
    public User createUser(User user) {
        User savedUser = userMapper.save(user);
        eventPublisher.publishEvent(new UserCreated(savedUser.id));
        return savedUser;
    }

}
```

+ 定义EventListener

```java
// Listener监听接受通知
@Component
public class UserEventNotifier {
    
    @Autowired
    private UserMapper userMapper;

    @EventListener(UserCreated.class)
    public void onUserCreate(UserCreated userCreated) {
        // 获取用户
        User user = userMapper.find(userCreated.id);
        // 基于用户信息构建邮件模板，发送邮件
    }

}
```

## 3.1 实现剖析

关于Spring事务，我们需要先明确如下原则:

+ 事务和线程绑定的，也就是**不同的线程会分配不同的事务**
+ @Transactional修饰的类，当前类方法在调用的时候，默认的传播机制是REQUIRED，也就是会延用**当前线程所处的事务**
+ 默认情况下，只有**@Transactional标注的方法执行完成之后**，事务才会提交(当然包括底层数据库事务提交)

+ 默认情况下，EventListener是在调用线程中执行的，也就是和`userMapper.save()`在同一个事务中

所以`userMapper.save(user)` 和 `onUserCreate` 在同一个事务中执行，所以后者肯定是能读取到插入的数据的。

## 3.2 扩展方案

为了明确的拆分上面两个操作，比如说用户插入完成之后(commit)了，才进行邮件发送。我们可以使用Spring提供的`TransactionalEventListener` - 它会基于事务的不同的阶段来做出不同的处理，比如说等待当前事务提交之后再执行相应的操作。

```java
// Listener监听接受通知
@Component
public class UserEventNotifier {
    
    @Autowired
    private UserMapper userMapper;

    @EventListener(UserCreated.class)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onUserCreate(UserCreated userCreated) {
        // 获取用户
        User user = userMapper.find(userCreated.id);
        // 基于用户信息构建邮件模板，发送邮件
    }

}
```

`@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`注解的作用就是在它所处的事务提交之后才会执行onUserCreate中的代码。当然底层涉及到Spring事务同步机制(**synchronization**)。

# 4. 参考

\> [Spring事务和事件机制混用的时候 需要注意事务隔离性](https://blog.pragmatists.com/spring-events-and-transactions-be-cautious-bdb64cb49a95)
---
layout: post
category: spring
date: 2020-06-28 17:26:03 UTC
title: 【Java基础】动态代理
tags: [Proxy, Dynamic Proxy, InvocationHandler]
permalink: /java/basics/dynamic-proxy
key:
description: 本文整理了Java中的动态代理基础知识以及实际运用
keywords: [Proxy, Dynamic Proxy, InvocationHandler]
---

# 1. 动态代理(Dynamic Proxies)

## 1.1 定义

基于代理模式(Proxy Pattern)的实现:

+ 隐藏被代理对象
+ 代理和被代理对象实现共同接口，**代理持有共同接口的引用**，对调用者暴露代理，调用时实际执行被代理者的方法

而动态代理则更进一步，`动态产生代理对象`和`将调用动态传递给被代理的方法`(实际上就是和动态代理对象关联的invocation handler)

> Java’s dynamic proxy takes the idea of a proxy one step further, by both creating the proxy object dynamically and handling calls to the proxied methods dynamically

实现步骤

+ 实现InvocationHandler，用于接收方法调用的转发
+ 动态代理的方法调用转到Handler的invoke方法


```java
public class DynamicProxy implements InvocationHandler {

  private WizardTower proxied;

  public DynamicProxy(WizardTower proxied) {
    this.proxied = proxied;
  }

  @Override
  public java.lang.Object invoke(java.lang.Object proxy, Method method, java.lang.Object[] args) throws Throwable {
      
      
      // 加入其它处理后，最终真正在被代理的类上 执行方法
      return method.invoke(proxied, args);
      
  }
    
}

public interface WizardTower {

    void enter(Wizard wizard);

}

public class WizardTowerImp implements WizardTower {

    @Override
    public void enter(Wizard wizard) {
        System.out.println("kill " + wizard.toString());
    }

}

/*
    + 代理类的Classloader
    + 被代理的接口类(运行时表示)
    + InvocationHandler实例
*/

WizardTower wizardTowerProxy =
            (WizardTower) Proxy.newProxyInstance(
                WizardTower.class.getClassLoader(),
                new Class[]{WizardTower.class},
                new DynamicProxy(new WizardTowerImp())
            );

// 返回代理类(接口的)的实例，用于将方法调用转发到Invocation Handler中invoke方法中      
wizardTowerProxy.enter(new Wizard("xxx"));
```

## 1.2 核心组件

### 1.2.1 java.lang.reflect.Proxy `用于创建动态代理`

Proxy提供了用于创建动态代理的对象以及实例的静态方法，与此同时也是所有由该类创建的动态代理的父类

如果我们需要为Foo Interface创建一个动态代理类的话，可以通过如下方法:

```java
 Proxy.newProxyInstance(Foo.class.getClassLoader(), new Class<?>[] {Foo.class}, handler)
```

所谓的动态代理实际就是一个类，它在创建时就实现了`运行时指定的`一系列的接口(a dynamic proxy is class that implements a list of interfaces specified at runtime when the class is created)，并且有如下行为(with behavior as described below)。

### 1.2.2 java.lang.reflect.InvocationHandler

定义了代理 `对应的handler` 应该实现的规范，每一个Proxy实例都会与一个handler相关联

```java
// java.lang.reflect.Proxy
public class Proxy implements java.io.Serializable { 

    protected InvocationHandler h;
    
    protected Proxy(InvocationHandler h) {
    	Objects.requireNonNull(h);
    	this.h = h;
	}

}
```

当我们在代理上调用某个方法的时候，实际上会转给与它关联的handler(**that it is assocaited with**)的`invoke`方法上

> Each proxy instance has an associated invocation handler, when a method is invoked on a proxy instance, it will be encoded and dispatched to invoke method of the corresponding invocation handler

## 1.3 实现剖析

当被问到，Java动态代理如何工作的时候，我们肯定会提到`java.lang.reflect.Proxy.newInstance`方法，但是我们知道正常的通过`Interface.class.newInstance`是无法创建的，那么Proxy底层又是如何实现的。

底层实际上是通过`sun.misc.ProxyGenerator`动态的进行字节码生成的，所以我们看到代理类一般都类似于这种的`$Proxy`，开启`-verbose:class`参数，可以看到动态加载了代理类`com.sun.proxy.$Proxy0`

```java
public interface DInter {
}

// 字节码生成之后，动态加载
// [Loaded com.sun.proxy.$Proxy0 from sun.misc.Launcher$AppClassLoader]
// -verbose:class
Object dInterInstances =
    Proxy.newProxyInstance(
        KdcHiveConnectTest.class.getClassLoader(), 
    	new Class[] {DInter.class}, 
    	new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                return proxy;
            }	
        }
    );
```

# 2. 参考

\> [SO Java动态代理底层是如何工作的，是如何动态的创建接口实现类的](https://stackoverflow.com/questions/781528/how-does-javas-dynamic-proxy-actually-work)

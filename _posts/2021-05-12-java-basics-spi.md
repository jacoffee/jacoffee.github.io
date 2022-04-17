---
layout: post
category: java
date: 2021-05-12 15:29:03 UTC
title: 【Java基础】SPI
tags: [类加载，SPI]
permalink: /java/oop/spi
key:
description: 本文主要剖析SPI的实现
keywords: [类加载，SPI]
---

SPI(service provider interface) -- 服务提供接口，一种扩展机制，在相应的位置**resources/META-INF/services/**配置接口的实现类，Java通过ServiceLoader去加载这些接口的实现类,  从而实现动态扩展。是一种典型的解耦思想也体现了OOP中的开闭原则(对于扩展开放，对于修改是封闭的)。实际上也是对类加载器"限制"的一种扩展，比如说定义通用规范的DriverManager,  它们在JDK核心包中，但是实现类肯定不能放里面，所以SPI也为加载提供商实现类提供了便利。

# 1. 使用流程

## 1.1 定义服务接口

```java
package org.apache.ibatis.jacoffee.spi;

public interface SQLParserProvider {
    void parse(String text);
}
```

## 1.2 定义服务接口实现类

需要无参构造方法

```java
public class DruidSQLParser implements SQLParserProvider {

    @Override
    public void parse(String text) {
        System.out.println("Druid SQL Parser parse: " + text);
    }

}

public class JacoffeeSQLParser implements SQLParserProvider {

    @Override
    public void parse(String text) {
        System.out.println("Jacoffee SQL Parser parse: " + text);
    }
    
}
```

## 1.3 配置实现类

一般是在**resources**文件下面，新建`META-INF/services/`目录，然后新建服务配置文件, **文件名为服务接口的全限定名**(带package名的) - `org.apache.ibatis.jacoffee.spi.SQLParserProvider`

```
org.apache.ibatis.jacoffee.spi.impl.JacoffeeSQLParser
org.apache.ibatis.jacoffee.spi.impl.DruidSQLParser
```

## 1.4 测试

```java
ServiceLoader<SQLParserProvider> provider = ServiceLoader.load(SQLParserProvider.class);
Iterator<SQLParserProvider> iterator = provider.iterator();
while (iterator.hasNext()) {
    iterator.next().parse("select * from `user`");
}
```

# 2. 源码剖析

### 2.1.1 ServiceLoader初始化，将Service类和线程上下文类加载器封装在LazyIterator中

ServiceLoader初始化，定义加载Service Provider的类加载器，然后将查找加载逻辑封装在LazyIterator中。初始化的核心方法就是`reload()`。

```java
public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
```

### 2.1.2  ServiceLoader的LazyIterator借助迭代器模式完成类加载

如果在自身项目和依赖包同时配置了Service Provider，`优先执行注册本项目中的`，但是依赖包中加载的顺序则不确定。

```java
// ServiceLoader
public Iterator<SQLParserProvider> iterator() {

    new Iterator<S> {
            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                // 也就是上面的LazyIterator
                return lookupIterator.next();
            }
    }

}

private class LazyIterator implements Iterator<S> {

    private boolean hasNextService() {
            ....
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        // 获取地址的核方法 ==> 
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            ....
    }

}
```

### 2.1.3 用户迭代Iterator的时候，真正触发Service类实例构造

当用户的iterator被调用，导致底层的LazyIterator的`nextService()`被调用，这个过程中会生成**类的实例(反射 + 无参构造方法)**，同时缓存下来。从这里反射初始化实例，我们可以看到ServiceLoader机制的一个限制，那就是`实现类必须定义无参数的构造函数`。

> The only requirement enforced by this facility is that **provider classes must have a zero-argument constructor** so that they can be instantiated during loading

# 3. 典型场景

## 3.1 JDBC Driver加载

JDBC操作数据库时候，有一个必要步骤就是先注册并且加载对应数据库的Driver实现。Java层面提供了统一的操作接口`java.sql.Driver`, 各个厂商各自实现，以操作MySQL为例:

```java
String url = "jdbc:mysql://localhost:3306/content_center?useSSL=false&useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&failOverReadOnly=false&useSSL=false";
String user = "xxxxx";
String passwd = "xxxxx";

// Class.forName("com.mysql.jdbc.Driver") 并没有配置但还是可以Work, 这是什么原因
Connection conn = DriverManager.getConnection(url, user, passwd);
ResultSet rs = conn.prepareStatement("select * from `jc_match` limit 1").executeQuery();
while (rs.next()) {

}
conn.close();
```

前面提到，在使用具体数据库的时候不是要进行加载吗(**Class.forName("com.mysql.jdbc.Driver")**。其实在DriverManager初始化的时候就已经进行注册加载了，实现在**静态初始化块中的loadInitialDrivers()**

```java
public class DriverManager {
    
    // 当前JVM中注册的Driver，扫描所有依赖下面的指定文件夹 META-INF/services
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
    
    static {
        loadInitialDrivers();
    }
    
    private static void loadInitialDrivers() {
        
        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
        
        Iterator<Driver> driversIterator = loadedDrivers.iterator();
        
        // 主动调用迭代器去 加载Driver，从而导致Driver实现类中的静态初始化块被执行，进而主动注册自己
        try {
            while (driversIterator.next()) {
                driversIterator.next();
            }    
        } catch (Throwable t) {
            
        }   
    }
    
}

package com.mysql.jdbc

public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    
    static {
        try {
            // 初始化的时候注册自己
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException e) {
            throw new RuntimeException("Can't register driver!");
        }
    }
    
}
```

而翻开MySQL connector的源码，可以看到有一个**java.sql.Driver**  文件，里面维护的是MySQL的Driver实现类

```bash
mysql-connector-java-5.1.47.jar
com
META-INF
    INDEX.LIST
    MANIFEST.MF
services
    java.sql.Driver
        com.mysql.jdbc.Driver
        com.mysql.fabric.jdbc.FabricMySQLDriver
```

registerDrivers:

```bash
0 = {DriverInfo@967} "driver[className=com.alibaba.druid.proxy.DruidDriver@6fdb1f78]"
1 = {DriverInfo@968} "driver[className=com.alibaba.druid.mock.MockDriver@59f99ea]"
2 = {DriverInfo@969} "driver[className=com.mysql.jdbc.Driver@239963d8]"
3 = {DriverInfo@970} "driver[className=com.mysql.fabric.jdbc.FabricMySQLDriver@598067a5]"
```

## 3.2 Spring Boot的SPI机制

严格来说Spring boot中是**思想类似**并不是真正意义上的SPI机制。它体现在进行自动装配阶段，SpringFactoriesLoader会负责扫描 `META-INF/spring.factories`中配置的EnableAutoConfiguration的实现类。

# 4.  SPI破坏了双亲委派机制嘛？

关于这个问题，随便网上搜帖子可以看到很多人回答是的。但知乎这个帖子[为什么说java spi破坏双亲委派模型？](https://www.zhihu.com/question/49667892)也有人给出了不同的解释。

## 4.1 正方观点

ServiceLoader暴露的加载方法:

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {}
```

+ 一个使用的是当前线程的**上下类加载器(并没有像正常加载那样向上寻找)**，一个使用的参数中的传递的ClassLoader(有可能加载逻辑已经完全改变)
+ 基于类加载的可见性原则， 也就是**系统类加载器加载的类对于启动类加载器是不见的**。 **SPI的调用方和接口定义方很可能都在Java的核心类库之中**，而实现类交由开发者实现，然而实现类并不会被启动类加载器所加载，**基于双亲委派的可见性原则，SPI调用方无法拿到实现类**。SPI Serviceloader通过线程上下文获取能够加载实现类的Classloader(一般情况下是系统类加载器)，绕过了这层限制，逻辑上打破了双亲委派原则

> Visibility principle allows child class loader to see all the classes loaded by parent ClassLoader, but parent class loader can not see classes loaded by child


## 4.2 反方观点

在JDBC中加载Driver获取连接的时候:

```java
// null 说明该类是由BoostrapClassLoader加载的
System.out.println(java.sql.Connection.class.getClassLoader());

Connection conn = DriverManager.getConnection("jdbc:mysql://xxxxx/xxxxx", "xxxxx", "xxxxx")
// com.mysql.jdbc.JDBC4Connection ==> sun.misc.Launcher$AppClassLoader@18b4aac2
System.out.println(conn.getClass().getClassLoader());
```

可以看到Connection是由启动类加载器加载的，JDBC4Connection这个第三方的类是由系统类加载器记载的，这个从逻辑上来看也没有什么问题。启动类加载器肯定不能加载第三方的类。

其实是与否都不重要，重要的是我们**需要掌握Java类加载机制中的双亲委派机制(parent delegation)以及SPI的用法**。

# 5. 参考

\> [mp 从源码角度，看 Java 是如何实现自己的 SPI 机制的？](https://mp.weixin.qq.com/s/20t_UtNNwXfynbzxpi7p2Q)


\> [zhihu 为什么说java spi破坏双亲委派模型？](https://www.zhihu.com/question/49667892)



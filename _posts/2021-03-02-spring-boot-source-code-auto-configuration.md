---
layout: post
category: spring
date: 2021-03-02 15:29:00 UTC
title: 【Spring boot基础】自动装配原理剖析
tags: [AutoConfiguration、Servlet&Reactive]
permalink: /spring-boot/httpmessageconverter
key:
description: 本文介绍了Spring boot是如何启动的以及自动装配是如何实现的
keywords: [AutoConfiguration、Servlet&Reactive]
---



[TOC]

# 1. Spring boot运行核心流程

## 1.1 SpringApplication初始化

### 1.1.1 推断Web应用类型

根据Classpath上是否有某些类，Servlet和Reactive的类不一样

### 1.1.2 初始化实现了ApplicationContextInitializer的类

通过扫描所有**META-INF/spring.factories**获取实现类全类名然后反射实例化;

扫描的时候，会从Classpath中的所有`META-INF/spring.factories`读取，本质上还是Properties，key-value对的形式，然后缓存下来，key就是Properties中的key

以spring-boot的META-INF/spring.factories为例

```properties
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
```

最终在内存中维护的就是 ApplicationContextInitializer有哪些实现类

### 1.1.3 初始化实现了ApplicationListener的类

通过扫描所有`META-INF/spring.factories`获取实现类全类名然后反射实例化

这个步骤同上，只不过最终获取的是实现的是`org.springframework.context.ApplicationListener`(注意这个类是Spring里面的)

### 1.1.4 推断主类

通过调用栈判断哪个类包含主方法(main)， 直接通过初始化RuntimeException来获取调用栈，非常妖。

```java
public class SpringApplication {
    
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   			
        // 推断WebApplication类型: Servlet Or Reactive
        this.webApplicationType = WebApplicationType.deduceFromClasspath();

        //  是否在这个地方进行Factories扫描 === 扫描实例 && 动态代理 动态生成字节码
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

        // 是否在这个地方进行Listener扫描
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

        // 主类推断
        this.mainApplicationClass = deduceMainApplicationClass();
        
        
    }
 
    
        /**
     *  直使用RuntimeException来获取调用栈，但实际上这个地方根本是没有异常的
     * @return
     */
    private Class<?> deduceMainApplicationClass() {
        try {
            StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
            // 基础知识: 最先调用的方法在栈底，也就是XXApplicationd的Main方法方法，调用一次方法，入栈，什么栈，大多数情况下Java虚拟机栈，有什么 主要是方法局部变量，操作

            /*
                0 = {StackTraceElement@2204} "org.springframework.boot.SpringApplication.deduceMainApplicationClass(SpringApplication.java:281)"
                1 = {StackTraceElement@2205} "org.springframework.boot.SpringApplication.<init>(SpringApplication.java:274)"
                2 = {StackTraceElement@2206} "org.springframework.boot.SpringApplication.<init>(SpringApplication.java:253)"
                3 = {StackTraceElement@2207} "org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)"
                4 = {StackTraceElement@2208} "org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)"
                5 = {StackTraceElement@2209} "XXApplication.main(XXApplication.java:17)"
            */
            for (StackTraceElement stackTraceElement : stackTrace) {
                if ("main".equals(stackTraceElement.getMethodName())) {
                    // 成功锁定 并且 加载
                    return Class.forName(stackTraceElement.getClassName());
                }
            }
        } catch (ClassNotFoundException ex) {
            // Swallow and continue
        }
        return null;
    }
   
    
}
```

## 1.2 SpringApplication.run -- 核心步骤

### 1.2.1 搜索实现了`SpringApplicationRunListener`类、初始化，然后通过广播通知这些listener执行相应的方法。`典型的生命周期和Calback结合的实现`

> 这对于我们编程设计的指导就是，如果框架某个流程设计到很多阶段，常规是注册回调`beforeHandle`、`afterHandle`,  但如果我们消费者(对这些阶段感兴趣的有很多)，可以尝试嵌入Listener， 然后发生对应事件广播给这些Listener

`这个地方重点的不是实现，而是让我们知道Spring的扩展点，后面知道在什么地方扩展`

```bash
org.springframework.boot.web.servlet.context.AnnotationConfig `ServletWebServer` ApplicationContext

org.springframework.context.annotation.AnnotationConfigApplicationContext
```

### 1.2.2 实例化Spring容器 -- AnnotationConfigServletWebServerApplicationContext

`AnnotationConfigApplicationContext`我们在Spring源码中接触，负责整个容器的初始化(**Standalone application context**)，通过前缀我们看出来主要去扫描注册类的配置，像我们常见的`@Configuration`、`@Component`等。支持通过Register方法手动去注册 Annotated Class或者事设置路径去扫描

```java
* Standalone application context, accepting annotated classes as input - in particular
* {@link Configuration @Configuration}-annotated classes, but also plain
* {@link org.springframework.stereotype.Component @Component} types and JSR-330 compliant
* classes using {@code javax.inject} annotations. 
Allows for registering classes one by one using {@link #register(Class...)} as well as for classpath scanning using {@link #scan(String...)}.
```

它是没有Web支持的，而`AnnotationConfigServletWebServerApplicationContext`则可以理解在前者的功能加了Web相关的支持，不过在UML图中它们是平级的。

### 1.2.2 依次调用注册Initializers和Listeners -- SpringApplication.prepareContext

比如说`DubboApplicationContextInitializer`就通过initialize方法注册了Dubbo相关的BeanFactoryPostProcessor，即`BeanDefinitionRegistryPostProcessor`。

### 1.2.3 通过refreshContext触发容器的初始化

包括执行BeanFactoryPostProcessor、初始化Spring内置bean、用户定义的单例Bean。

像我们比较关心的Tomcat容器、Spring MVC的相关支持DispatchServlet的初始化也是在该阶段完成。

可以说通过厘清，Tomcat容器是如何初始化的以及DispatchServlet是如何初始化并且注册到Tomcat中的(Tomcat会将请求分发给DispatchServlet)的具体流程，我们基本上就了解了Spring boot的核心: `自动装配的原理`。

# 2. 自动装配

## 2.1 什么是自动装配

自动装配指的是Spring boot会基于`EnableAutoConfiguration`去自动注册用户可能需要的Bean, 也就是引入相关功能。

核心是`EnableAutoConfiguration`注解中配置的`AutoConfigurationImportSelector`，它可以扫描加载并且初始化用户可能需要的Bean，该Selector会扫描classpath下面的`META-INF/spring.factories`中配置的AutoConfiguration类。

>  Enable auto-configuration of the Spring Application Context, attempting to guess and configure beans that you are likely to need. Auto-configuration classes are usually
>  applied based on your classpath and what beans you have defined. For example, if you have {@code tomcat-embedded.jar} on your classpath you are likely to want a
>  {@link TomcatServletWebServerFactory} (unless you have defined your own {@link ServletWebServerFactory} bean)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    
}
```

`XXAutoConfiguration`可以理解为`Configuration`的增强版本，引用的Bean的形式也多种多样:

### 2.1.1 通过@Import注解引入相关的Bean

```java
// WebServer的自动引入类
@Import({
	ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class, // ImportBeanDefinitionRegistrar实现类
	ServletWebServerFactoryConfiguration.EmbeddedTomcat.class // Configuration
})
public class ServletWebServerFactoryAutoConfiguration {

}
```

### 2.1.2 直接像普通的`@Configuration`那样引入Bean

```java
@Configuration
@EnableConfigurationProperties(JacoffeeThreadPoolProperties.class)
public class JacoffeeThreadPoolAutoConfiguration {

    public static final String DEFAULT_THREADPOOL_BEAN_NAME = "jacoffeeThreadPool";

    @Bean(name = DEFAULT_THREADPOOL_BEAN_NAME)
    public JacoffeeThreadPoolExecutor jacoffeeThreadPool(JacoffeeThreadPoolProperties jacoffeeThreadPoolProperties) {
        ThreadFactory schemaTaskThreadFactory = new ThreadFactoryBuilder().setNameFormat("schedule-task-%d").build();

        JacoffeeThreadPoolExecutor threadPoolExecutor = new JacoffeeThreadPoolExecutor(
             jacoffeeThreadPoolProperties.getCorePoolSize(),
             jacoffeeThreadPoolProperties.getMaximumPoolSize(),
             1L, TimeUnit.MINUTES,
             new LinkedBlockingDeque<>(jacoffeeThreadPoolProperties.getMaxQueueSize()),
             schemaTaskThreadFactory
         );

         return threadPoolExecutor;
    }

}
```

实际上这两种在复杂的spring-boot-starter中通常会混用。

同时会通过组合这三个`@Conditional`, `@ConditionalOnClass`以及`@ConditionalOnMissingBean`注解来实现`什么情况初始化什么Bean的逻辑`。

比如说DataSource注册，如果classpath上面有Hikira的相关依赖，就初始化HikiraDataSource; 如果有Druid相关依赖，就初始化DruidDataSource。

## 2.2 实现剖析

通过EnableAutoConfiguration的源码我们可以发现，它是利用`AutoConfigurationImportSelector`来实现具体的配置引入，本质来讲AutoConfiguration类就是`@Configuration`注解修饰的类

### 2.2.1 Spring会基于我们定义的入口类`XXApplication` (它被@EnableAutoConfiguration注解修饰),  会基于它去加载BeanDefinition

而实际搜索由会交由`AutoConfigurationImportSelector`去处理。至此**Spring的BeanDefinition注册过程便和Boot的自动装配流程产生关联**

```bash
"restartedMain@1900" prio=5 tid=0x15 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.springframework.boot.autoconfigure.AutoConfigurationImportSelector.getAutoConfigurationEntry(AutoConfigurationImportSelector.java:124)
	  at org.springframework.boot.autoconfigure.AutoConfigurationImportSelector$AutoConfigurationGroup.process(AutoConfigurationImportSelector.java:434)
	  at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorGrouping.getImports(ConfigurationClassParser.java:879)
	  at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorGroupingHandler.processGroupImports(ConfigurationClassParser.java:809)
	  at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorHandler.process(ConfigurationClassParser.java:780)
	  at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:193)
	  at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:319)
	  at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:236)
	  at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:280)
	  at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:96)
	  at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:707)
	  at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:533)
	  - locked <0x132a> (a java.lang.Object)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:143)
	  at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:758)
	  at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:750)
	  at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:405)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:315)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
```

### 2.2.2 AutoConfigurationImportSelector会扫描classpath下面`META-INF/spring.factories`中配置的EnableAutoConfiguration的实现类

SpringFactoriesLoader会具体负责这一流程

```bash
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.jacoffee.threadpool.config.JacoffeeThreadPoolAutoConfiguration
```

### 2.2.3 基于扫描出来AutoConfiguration进行过滤(比如说配置了exclude)，然后分别解析，最终目的是要存放在不同的结构，后面会通过`ConfigurationClassBeanDefinitionReader`来进行生成BeanDefinition

本质上来讲AutoConfiguration是通过Configuration机制来暴露Bean的，而它自己也可以通过@Import注解进行额外的Bean引入。

**源码层面**:  ConfigurationClassParser的`doProcessConfigurationClass`方法会根据不同的情况去处理，比如说上面的`jacoffeeThreadPool`方法就会被维护在ConfigurationClass内部的`beanMethods`中。后续ConfigurationClassBeanDefinitionReader会进行beanMethods处理，将BeanDefinition维护到`BeanDefinitionRegistry`中

通过这几个核心步骤，AutoConfiguration中的Bean就被注册进入了Spring容器中，后续在Bean初始化阶段被真正处理。

## 2.3 核心流程之Tomcat是如何被初始化

`ServletWebServerFactoryAutoConfiguration`执行流程，负责内置Web Server的相关Bean注册。

`PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors`导致`ServletWebServerFactoryAutoConfiguration`中`EmbeddedTomcat`暴露TomcatServletWebServerFactory bean， 而该工厂类是有创建Tomcat Server的能力的。

另外关于Tomcat的创建时机，AbstractApplicationContext作为Spring容器的抽象类，在refresh()方法中嵌入回调onRefresh回调，在此方法执行之前，很多Bean已经完成注册，当然包括`TomcatServletWebServerFactory`，因此`ServletWebServerApplicationContext`作为创建、初始化、并且运行在WebServer之上的Spring容器，会覆盖`onRefresh`完成Web Server的创建逻辑。

## 2.4 核心流程之Spring MVC DispatchServlet是如何被初始化

`DispatcherServletAutoConfiguration`执行流程，负责Spring WebMVC相关的功能



# 3. 实战演练

我们在基于Spring boot实际开发的过程中，会引入各种starter， 比如说`spring-boot-starter-web`, `druid-spring-boot-starter`， 它通过集成的方式引入了一揽子的依赖，比如说前面的如果我们需要引入druid相关的功能。不用说背后肯定会使用到AutoConfiguration, 下面的我们通过引入的一个定制化的threadpool，来看看如何基于AutoConfiguration开发一个简单的starter，`spring-boot-starter-threadpool`。



## 3.1 定义核心功能类 -- JacoffeeThreadPoolExecutor

继承ThreadPoolExecutor，实现自己的定制化功能

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class JacoffeeThreadPoolExecutor extends ThreadPoolExecutor {

    public JacoffeeThreadPoolExecutor(
        int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue
    ) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
    }

    public JacoffeeThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
    }

    public JacoffeeThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
    }

    public JacoffeeThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    }

    @Override
    public BlockingQueue<Runnable> getQueue() {
        // super.getQueue().getClass()
        return super.getQueue();
    }

}
```



## 3.2 通过ConfigurationProperties定义线程池所需要的配置(非必须)

核心还是为了让配置更加集中， 然后通过`EnableConfigurationProperties` 方式来引入这种形式注册的Bean。

```java
@ConfigurationProperties(prefix = "spring.jacoffee.threadpool")
public class JacoffeeThreadPoolProperties {

    private Integer corePoolSize;

    private Integer maximumPoolSize;

    private Integer maxQueueSize;

    public Integer getCorePoolSize() {
        return corePoolSize;
    }

    public void setCorePoolSize(Integer corePoolSize) {
        this.corePoolSize = corePoolSize;
    }

    public Integer getMaximumPoolSize() {
        return maximumPoolSize;
    }

    public void setMaximumPoolSize(Integer maximumPoolSize) {
        this.maximumPoolSize = maximumPoolSize;
    }

    public Integer getMaxQueueSize() {
        return maxQueueSize;
    }

    public void setMaxQueueSize(Integer maxQueueSize) {
        this.maxQueueSize = maxQueueSize;
    }

}

```

## 3.3 AutoConfiguration的前半部分 -- 在@Configuration暴露Bean

+ @Configuration注解相关的类
+ 基于方法暴露@Bean，同时辅以各种@ConditionalXX注解，以便基于不同的情况加载Bean

```java
import com.google.common.util.concurrent.ThreadFactoryBuilder;
import com.jacoffee.threadpool.JacoffeeThreadPoolExecutor;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * {@link EnableAutoConfiguration Auto-configuration} for jacoffee thread pool
 */
@Configuration
@EnableConfigurationProperties(JacoffeeThreadPoolProperties.class)
public class JacoffeeThreadPoolAutoConfiguration {

    public static final String DEFAULT_THREADPOOL_BEAN_NAME = "jacoffeeThreadPool";

    @Bean(name = DEFAULT_THREADPOOL_BEAN_NAME)
    @ConditionalOnMissingBean
    public JacoffeeThreadPoolExecutor jacoffeeThreadPool(JacoffeeThreadPoolProperties jacoffeeThreadPoolProperties) {
        ThreadFactory schemaTaskThreadFactory = new ThreadFactoryBuilder().setNameFormat("schedule-task-%d").build();

        JacoffeeThreadPoolExecutor threadPoolExecutor = new JacoffeeThreadPoolExecutor(
             jacoffeeThreadPoolProperties.getCorePoolSize(),
             jacoffeeThreadPoolProperties.getMaximumPoolSize(),
             1L, TimeUnit.MINUTES,
             new LinkedBlockingDeque<>(jacoffeeThreadPoolProperties.getMaxQueueSize()),
             schemaTaskThreadFactory
         );

         return threadPoolExecutor;
    }

}
```

## 3.4 AutoConfiguration的后半部分 -- Auto如何体现

在`resources/META-INF/spring.factories`配置EnableAutoConfiguration，以让Spring boot自动装配机制能根据 JacoffeeThreadPoolAutoConfiguration 中定义的规则进行Bean的初始化。

```bash
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.jacoffee.threadpool.config.JacoffeeThreadPoolAutoConfiguration
```

至此一个基本的starter就完成了，核心就与利用Spring boot的Auto Configuration机制。

# 4. 参考

> [Spring官网对于Auto configuration](https://docs.spring.io/spring-boot/docs/1.3.8.RELEASE/reference/html/using-boot-auto-configuration.html)

> [mp 淘宝一面："说一下 Spring Boot 自动装配原理呗？"](https://mp.weixin.qq.com/s/bSY_LdiDs1339lL9zEGl9Q)

> [blog Spring boot starter中的SPI 机制](https://juejin.cn/post/6844903890173837326)

>[How Spring Boot Autoconfiguration works?](https://medium.com/empathyco/how-spring-boot-autoconfiguration-works-6e09f911c5ce)

> [blog Autoconfiguration的魔力在哪](https://dzone.com/articles/how-springboot-autoconfiguration-magic-works)

> [bilibili 子路说之springboot源码](https://www.bilibili.com/video/BV1pa4y1v7ao)
>
> [bilibli Spring boot starter原理](https://www.bilibili.com/video/BV1s7411M7pH)

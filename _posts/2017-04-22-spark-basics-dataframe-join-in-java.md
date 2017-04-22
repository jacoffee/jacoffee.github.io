---
layout: post
category: spark
date: 2017-04-22 08:30:55 UTC
title: Spark基础之Java中使用DataFrame Join时的序列化问题
tags: [Serializable，延时计算，懒加载]
permalink: /spark/dataframe-join-in-java
key: 
description: 本文介绍了Java Spark中，使用DataFrame Join时的序列化问题
keywords: [Serializable，延时计算，懒加载]
---

在DataFrame中，join方法有很多重载的版本，大致如下:

```scala
def join(right: DataFrame, usingColumn: String): DataFrame
def join(right: DataFrame, usingColumns: Seq[String]): DataFrame
def join(right: DataFrame, joinExprs: Column, joinType: String): DataFrame
def join(right: DataFrame, usingColumns: Seq[String], joinType: String): DataFrame
```

我们重点来看一下第三个和第四个方法。第三个允许我们自己构造join表达式，并且指定连表类型，**如果两边join的Column一样，则两个都会被保留**。那么最后列名映射的时候，就会因为有两个相同的列名而抛出下面的**AnalysisException**。当然我们也可以在刚开始的时候分别重命名对应的列名。

```scala
val list1: List[(String, Int)] = List(("allen", 1001))
val dataFrame1 = sc.parallelize(list1).toDF("name", "id")

val list2: List[(String, Int)] = List(("allen", 26), ("zml", 28))
val dataFrame2 = sc.parallelize(list2).toDF("name", "age")
```

```scala
val dataFrame3 = dataFrame1.join(dataFrame2, dataFrame1.col("name") === dataFrame2.col("name"), "outer")
dataFrame3: org.apache.spark.sql.DataFrame = [name: string, id: int, name: string, age: int]

dataFrame3.show()
+-----+----+-----+---+                                                          
| name|  id| name|age|
+-----+----+-----+---+
| null|null|  zml| 28|
|allen|1001|allen| 26|
+-----+----+-----+---+

dataFrame3.select("name", "id", "age")
org.apache.spark.sql.AnalysisException: Reference 'name' is ambiguous, could be: name#12, name#16.;
```

第四个可以让我们直接指定join的列名以及连表类型，另外相较于第三个方法，**当join列名相同的时候，它只会保留其中一个**。

```scala
val dataFrame3 = dataFrame1.join(dataFrame2, List("name"), "outer")
dataFrame3: org.apache.spark.sql.DataFrame = [name: string, id: int, age: int]

dataFrame3.show()

+-----+----+---+                                                                
| name|  id|age|
+-----+----+---+
|  zml|null| 28|
|allen|1001| 26|
+-----+----+---+
```

在Scala中我们可以非常方便的使用该方法，但是Java中目前(Spark 1.6)好像不能使用该方法。因为它的方法签名需要**scala.collection.Seq**，而Java并没有该类型，不过我们可以通过相应的方法来构造。<b class="highlight">我在标题中提到的序列化问题也正是由于这次尝试导致的</b>。

```java
public static <T> scala.collection.Seq<T> asScalaSeq(T... elements) {
    // def toSeq: Seq[A] = toStream
    return collectionAsScalaIterable(Arrays.asList(elements)).toSeq();
}
```

通过上面的方法，将Java中的容器类型转换成为Scala的容器类型，但是上面的方法有一个潜在的问题 -- 将driver上不能序列化的游标类(<b class="highlight">Itr, java.util.AbstractList中的一个内部类</b>)直接传递到executor上面去了。

```java
DataFrame dataFrame3 =
    dataFrame1.join(dataFrame2, CollectionUtil.asScalaSeq("name"), "outer");
    
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
Caused by: java.io.NotSerializableException: java.util.AbstractList$Itr
```

**collectionAsScalaIterable**将Java容器类转换成Scala的Iterable类**JCollectionWrapper**，而**toSeq()**则将该它转换成为Stream(scala中的流)，这一个过程是非常自然的，因为它们两种类型的容器都有**延迟计算**的特性。关键就在这一步，转换过程中调用了**底层Java容器的iterator**方法，但恰好ArrayList(**java.util.Arrays中的内部类**，不是我们经常使用的那个)的iterator返回的是**Itr***的实例(它没有实现Serializable接口)。

```scala
// scala.collection.convert.Wrappers

case class JCollectionWrapper[A](underlying: ju.Collection[A]) 
    extends AbstractIterable[A] with Iterable[A] {
    ...
    def iterator = underlying.iterator
    ...
}
```

上面提到过程的简化调用链

```scala
Arrays.asList(elements)
    ArrayList<E> extends AbstractList<E>
    Iterator<E> iterator() { return new Itr(); } // 该类不能序列化
    
collectionAsScalaIterable(Arrays.asList(elements))
    JCollectionWrapper { def iterator = underlying.iterator }
        JCollectionWrapper.toSeq()
            JCollectionWrapper.toStream()
                iterator.toStream
                    underlying.iterator
```

对于类不能被序列化而造成Task Failure，一般有两种解法:

<ul class="item">
    <li>让该类实现Serializable接口，或者将该类转换成能序列化的类</li>
    <li>在Executor上面初始化该类</li>
</ul>

在本场景中，由于Executor使用该类的过程我们无法更改，所以可以尝试**让该类变得可以序列化**，最直接的方式就强制该流的输出，即立马拿到结果，比如说转换成**scala.collection.immutable.List**。

```java
DataFrame dataFrame3 =
    dataFrame1.join(dataFrame2, CollectionUtil.asScalaSeq("row1").toList(), "outer");
```

所以，在Spark中，在处理Scala和Java互相使用的时候一定要注意是否可以序列化的问题。
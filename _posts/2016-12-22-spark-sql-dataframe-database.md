---
layout: post
category: spark
date: 2016-12-22 11:30:55 UTC
title: Spark SQL基础之利用DataFrame进行数据库读写
tags: [JDBCType, DataType, LogicalRelation]
permalink: /spark/dataframe-database
key: "194592209f3d2e65f56756c8800fb6a3" 
description: "本文介绍DataFrame进行基本的数据库读写以及背后的基本原理"
keywords: [JDBCType, DataType, LogicalRelation]
---

之前用过Dataframe来读取MongoDB，PG，总体来说还算比较顺利。但今天在读写Oracle的时候却发现了一些问题，因此跟踪了一下源码顺便将基本的用法和背后的原理整理了一下(spark version=1.6.2)。

首先对于**DataFrame on Oracle**，要么就不用，要么就采用Spark 2.0及以上的版本，不然的话在<b style="color:red">Spark DataType到JDBCType的映射上会有问题</b>。

```scala
// spark 1.6
val sc = new SparkContext(new SparkConf().setMaster("local[*]").setAppName("Oracle Connection"))
val sqlc = new SQLContext(sc)
val dbProps: Properties = new Properties
dbProps.setProperty("driver", "oracle.jdbc.driver.OracleDriver")
dbProps.setProperty("url", "jdbc:oracle:thin:@localhost:1521:orcl")
dbProps.setProperty("user", "xxxx")
dbProps.setProperty("password", "xxxx")
import sqlc.implicits._

  
val resultDF =
    sc.parallelize(List(("allen", 21), ("zoe", 22), ("allen", 23)))
      .toDF.withColumnRenamed("_1", "name").withColumnRenamed("_2", "age")
      .groupBy("name").count()

// name VARCHAR2(255) , count BIGINT NOT NULL
JdbcUtils.schemaString(resultDF, dbProps.getProperty("url"))

resultDF.write.mode(SaveMode.Append).jdbc(dbProps.getProperty("url"), "xxxx", dbProps)
```

count的jdbc类型被映射成了bigint，而Oracle的数字类型中并没有这一类型，所以在建表的时候就会失败，<b class="highlight">根本原因还是OracleDialect的支持不完善</b>。

下面我们通过调用链来解释这个问题:

```scala
private case object OracleDialect extends JdbcDialect {
  override def getJDBCType(dt: DataType): Option[JdbcType] = dt match {
    case StringType => Some(JdbcType("VARCHAR2(255)", java.sql.Types.VARCHAR))
    case _ => None
  }
}
```
```scala
def getCommonJDBCType(dt: DataType): Option[JdbcType] = {
    dt match {
      ...
      case LongType => Option(JdbcType("BIGINT", java.sql.Types.BIGINT))
      ...
    }
}

dataFrameWriter.jdbc(url, )
    if (!tableExists) JdbcUtils.schemaString(df, url)
        val dialect = JdbcDialects.get(url)
        getJdbcType(field.dataType, dialect)
            dialect.getJDBCType(dt).orElse(getCommonJDBCType(dt))
            
```

因为Spark SQL会根据传入的url来寻找对应的数据库方言也就是确认对应的Column类型，因为OracleDialect只对于String类型进行了处理，对于LongType则因为没有对应的映射而采用了默认的即BIGINT，[Spark 2.0 OracleDialect的完整实现](https://github.com/apache/spark/blob/branch-2.1/sql/core/src/main/scala/org/apache/spark/sql/jdbc/OracleDialect.scala)。


## 关于DataFrame的数据库写入

<b class="highlight">DataFrame的数据库写入就是在`foreachPartition`中进行JDBC的数据库操作</b>，并且将每一次分区操作控制在了一个事务中同时使用了Preparement的Batch避免了多次插入，相关逻辑在`org.apache.spark.sql.DataFrameWriter`中实现。

```scala
// org.apache.spark.sql.execution.datasources.jdbc.JdbcUtils
saveTable()
 df.foreachPartition
  savePartition
    val stmt = insertStatement(conn, table, rddSchema)
    stmt.addBatch()
    stmt.executeBatch()
```

因为是通过`foreachPartition`进行数据库操作，显然我们需要控制分区数量以免crash数据库, 可以通过`df.coalesce`或者`df.repartition`，关于它们的区别可以参考[Spark基础之coalesce和repartition](/spark/partitions/coalesce-repartition)。

## 关于DataFrame的数据库读取

相较于写入，读取则显得相对复杂一点。它主要涉及到以下几个步骤，相关逻辑在`org.apache.spark.sql.DataFrameReader`中实现:

<b class="highlight">(1) 数据库记录Record ==> JDBCPartition</b>

也就是通过某种条件将Record进行分组，然后放入不同的分区。目前可以按照**某个Column的上下界(必须是整型)结合分区数**或者是提供一系列<b style="color:red">互斥的查询条件</b>进行划分。

以Column上下界为例，最简单的逻辑就是lowerBound按照步长`(upperBound - lowerBound) / numPartitions`往上累计，每累计一次作为一次查询条件。
通过下面的where查询条件，我们可以得知<b style="color:red">这种方式实际上将整表都取出来了</b>，所以在使用的时候需要注意。


```scala
sqlc.read.jdbc(url, tableName, "COLUMNNAME", 1, 96, 10, dbProps)
    JDBCPartitioningInfo(columnName, lowerBound, upperBound, numPartitions)
      JDBCRelation.columnPartition(partitioning)
        // 利用上下界结合传入的分区数拼接查询条件，直接在源码中打印出WhereClause
        // 这个过程是我自己在源码中打印的，Spark默认并不会提供这个行为
        whereClauseList = whereClause :: whereClauseList
        
List(
    COLUMNNAME >= 82, 
    COLUMNNAME >= 73 AND COLUMNNAME < 82, COLUMNNAME >= 64 AND COLUMNNAME < 73,      
    COLUMNNAME >= 55 AND COLUMNNAME < 64, COLUMNNAME >= 46 AND COLUMNNAME < 55,      
    COLUMNNAME >= 37 AND COLUMNNAME < 46, COLUMNNAME >= 28 AND COLUMNNAME < 37,      
    COLUMNNAME >= 19 AND COLUMNNAME < 28, COLUMNNAME >= 10 AND COLUMNNAME < 19,      
    COLUMNNAME < 10 or COLUMNNAME is null
)
```

它还有一个重载接口: 

```scala
def jdbc(
  url: String, table: String, 
  predicates: Array[String], connectionProperties: Properties
): DataFrame
```

由于这个查询条件是用于划分分区的，所以应该是互斥的。

<b class="highlight">(2) 数据库Column JdbcType ==> StructType</b>

既然写入的时候需要将StructType转换成JdbcType，所以读取的时候需要根据**url**, **dbProps**等信息获取表字段，然后转换成相应的StructType，从而获取了TableRelation。

```scala
jdbc(url, table, parts, connectionProperties)
    JDBCRelation(url, table, parts, props)(sqlContext)
        // 表字段类型转换成StructType
        JDBCRDD.resolveTable(url, table, properties): StructType
    sqlContext.baseRelationToDataFrame(relation)        
```

<b class="highlight">(3) TableRelation转换成Dataframe</b>

这个步骤实际上是Spark SQL中核心的实现，将各种LogicalPlan转换成为`DataFrame=DataSet[Row]`。但真正的计算是发生在各种操作的时候，比如说filter, groupBy。关于Spark SQL的几个重要组件以及背后的实现也会在接下来的文章中提到。

```scala
sqlContext.baseRelationToDataFrame(relation)        
    Dataset.ofRows(this, LogicalRelation(baseRelation))
      new Dataset[Row](sqlContext, logicalPlan, RowEncoder(qe.analyzed.schema))
```

其实`jdbc`那几个接口，个人觉得不是非常实用(或者是我并没有感受到)。需要传入上下界的那个，如果某个表是不断增长的并且没有一个合适整形Column，那么确定起来就非常困难。而需要传入predicates的那个接口，对于调用者就需要考虑一系列的互斥查询条件。

可以考虑增加一个接口，利用`主键哈希 + 分区数`来划分分区:

```scala
 def jdbc(
      url: String,
      table: String,
      column: String,
      numPartitions: Int,
      connectionProperties: Properties): DataFrame = { 
    // scan column value    
    // hash(value) %  numPartitions = partition index
}
```

目前来说，这个接口比较适合读取那些基础表，它们一般包含某种映射关系并且变动并不是很大，因此可以考虑整表当成一个分区读入，也就是默认的实现。





---
layout: post
category: mongodb
date: 2015-05-20 09:51:22 UTC
title: MongoDB基础之聚合(aggregation)
tags: [聚合，aggregation，管道，pipeline, 降维]
permalink: /mongodb/aggregation/
key: f2c79124fd492226e8102043e7f30a6c 
description: "本文简单地介绍了MongoDB的聚合的相关使用"
keywords: [聚合，aggregation，管道，pipeline, 降维]
---

MongoDB中聚合可以分为很多种，目前我接触的比较多就是pipeline, 它实际上就是将一堆collection扔进去然后按照特定条件分组并且提供一定的筛选条件，最后获取你想要的结果。

# 问题
在MongoDB中，如何使用pipeline?

# 解决
pipeline顾名思义就是管道，以大量的documents作为输入，每一阶段都会对文档进行处理，直到产生最终的结果，当然在此过程也可能产生新的文档。在MongoDB中，pipeline的基本语法如下:

```bash
collection.aggregate
```

## 常见的Stage

几个在MongoDB基本查询中也会经常用到的$limit, $sort, $skip,
两个在Aggregation中经常使用的$match, $group。第一个实际上就是普通查询的查询条件，第二个类似于MySQL中的分组。
还有几个个人接触的比较少但是觉得比较有意思的$unwind, $redact。

本文重点介绍一下$group, 其次是$unwind。

<1> $group

  该操作针对传入的Documents进行分组，然后将分好组的Documents传递给下一个阶段。
 
```bash
{
    $group: { 
        _id: <expression>, <field1>: { <accumulator1> : <expression1> }, ...     } 
}
```

引入测试数据

```bash
db.sales.insert(
[
  { 
    "_id" : 1, "item" : "abc", "price" : 10, 
    "quantity" : 2, "date" : ISODate("2014-01-01T08:00:00Z") 
  },
  { 
    "_id" : 2, "item" : "jkl", "price" : 20, 
    "quantity" : 1, "date" : ISODate("2014-02-03T09:00:00Z") 
  },
  { 
    "_id" : 3, "item" : "xyz", "price" : 5, 
    "quantity" : 5, "date" : ISODate("2014-02-03T09:05:00Z") 
  },
  { 
    "_id" : 4, "item" : "abc", "price" : 10, 
    "quantity" : 10, "date" : ISODate("2014-02-15T08:00:00Z") 
  },
  { 
    "_id" : 5, "item" : "xyz", "price" : 5, 
    "quantity" : 10, "date" : ISODate("2014-02-15T09:05:00Z") 
  },
  { 
    "_id" : 6, "item" : "xyz", "price" : 5, 
    "quantity" : 5, "date" : ISODate("2014-02-15T12:05:10Z") 
  },
  { 
    "_id" : 7, "item" : "xyz", "price" : 5, 
    "quantity" : 10, "date" : ISODate("2014-02-15T14:12:12Z") 
  }
])
```

<1> _id属性是必须的，但是也可以设置成null这样来计算所有文档的某个值。
<2> 第二个属性传入的必须是使用accumulator统计的值

```bash
"$group": { "_id": "$_id", item: "$item" } // 错误

"$group": { "_id": "$_id", total: {"$sum": "$item"} } // 正确 

"$group": { "_id": { identity: "$_id", item: "$item" }} 
// 正确 有点类似于联合主键的意思
```

其它的包括:

```bash
$sum: 统计分组中某个字段的和
$avg: 统计分组中某个字段的平均值
$first | $last: 返回分组中的第一个或最后一个文档中的某个值
$max | $min: 相对于$first | $last实际上有一个潜在的排序过程，因为要寻找的是最小和最大值
$push: 把每组指定属性值放到一个集合中
$addToSet: 相较于$push来说强调唯一性，但是不保证顺序
```

下面使用案例来验证上面的accumulator。

```bash
db.sales.aggregate([
    {"$sort": {"quantity": 1}},
    {
        "$group": 
       {
           "_id": "$item",
           "firstDate": {"$first": "$date"},
           "lastDate": {"$last": "$date"},
           "maxQ": {"$max": "$date"},
           "minQ": {"$min": "$date"},
           "q1": { $push: "$quantity"}, 
           "q2": { $addToSet: "$quantity"},
           "qAvg": {$avg: "$quantity"},
           "qSum": {$sum: "$quantity"}
       }
    },
    {"$limit": 1}
])
```

```bash
"result" : [ 
    {
        "_id" : "xyz",
        "firstDate" : 10,
        "lastDate" : ISODate("2014-02-15T14:12:12.000Z"),
        "maxQ" : ISODate("2014-02-23T09:05:00.000Z"),
        "minQ" : ISODate("2014-02-03T09:05:00.000Z"),
        "q1" : [ 
            3, 
            5, 
            5, 
            10, 
            10
        ],
        "q2" : [ 
            10, 
            5, 
            3
        ],
        "qAvg" : 6.6,
        "qSum" : 33
    }
],
"ok" : 1
}
```
<2> $unwind

这个modifier可以理解为是在降维， 如果对q2使用unwind操作，则原对象会被拆成三个对象，每一个对象的q2分别是10, 5, 3。这个操作即使针对重复值也会被当做单独的值处理。


## Pipeline中的性能优化

+ 当$sort和$match在一起的时候，不论顺序是怎样的，$match都会被移到$sort之前，这样通过$match可以大大减小需要排序的documents。

+ 当$skip和$limit在一起的时候，不论顺序是怎样的，$limit都会被放$skip前面，减少$skip的数量。因为skip需要MongoDB Server去从头开始扫描文档，直到停止的位置，然后将数据加载到内存中。所以当skip的数目比较大的时候，性能损耗就会比较大。

+ 当$limit和$limit在一起的时候，最终的结果是limit值更小的那个。

+ 当$skip和$skip在一起的时候，最终的结果是两个的skip值相加。

+ 当$match和$match在一起的时候，最终的结果就是两个查询条件的并集。


# 参考

<1> MongoDB Aggregation and Data Processing

<2> [Paging](http://stackoverflow.com/questions/7228169/slow-pagination-over-tons-of-records-in-mongo#)

<3> [Sample Code](http://media.mongodb.org/zips.json)

<4> [Aggregation Modifier First](http://docs.mongodb.org/manual/reference/operator/aggregation/first/#grp._S_first)



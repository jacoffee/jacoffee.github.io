---
layout: post
category: mongodb
date: 2016-01-27 10:12:25 UTC
title: MongoDB基础之BSON Date
tags: [协调世界时，BSON Date，UTC，ISO 8601，Decimal fraction of second]
permalink: /mongodb/bson-date/
key: cf1bc6b91fd0a95fece8eb7beaeaa341
description: "本文简单地介绍了MongoDB的日期类型以及在文档中的表示"
keywords: [协调世界时，BSON Date，UTC，ISO 8601，Decimal fraction of second]
---

接触MongoDB到现在已经有两年了，但对于日期类型还是很模糊的，印象中就只有ISODate， 因为几乎所有文档的日期类型都是用它表示的```ISODate("2014-04-16T03:39:14.874Z")```。 理解也仅限于，它好像和JS中的Date有点类似。所以决定系统的研究一下。

**(1)** Mongodb中日期的类型是使用BSON Date来表示的，而BSON Date对象是以**64bit的整形数**来存储的(实际上就是Java，Scala中的Long， 表示范围-2 ^ 63 to 2 ^ 63 - 1)，表示从Unix Epoch(1970-01-01)到某个时间点的**毫秒数**，大概可以表示从那个时间点**往前**或**往后**的2.9亿年。

计算过程如下:

```bash
scala> val ll: Long = pow(2, 63).toLong
ll: Long = 9223372036854775807

scala> ll - 1
res3: Long = 9223372036854775806

# 约为2.9亿年
scala> res3 / 1000 / 3600 / 24 / 365
res4: Long = 292471208
```

[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)(国际日期和时间规范)对于时间定义了如下几个[级别](https://www.w3.org/TR/NOTE-datetime):

```
 YYYY = 四位数表示的年 如 2016
 MM   = 两位数表示的月 如 01
 DD   = 两位数表示的日 如 27
 hh   = 两位数表示的小时 (00 through 23) 如09
 mm   = 两位数表示的分 (00 through 59) 如51
 ss   = 两位数表示的秒 (00 through 59) 如02
 s    = 一位或多位来表示秒的小数(decimal fraction)部分，基于精度的考虑 如051(下文有解释)
 TZD  = 时区表示 (Z or +hh:mm or -hh:mm)
```

另外当要表示时区差异时，也提供了如下两种方式:

> Times are expressed in UTC (Coordinated Universal Time), with a special UTC designator ("Z").

> Times are expressed in local time, together with a time zone offset in hours and minutes. A time zone offset of "+hh:mm" indicates that the date/time uses a local time zone which is "hh" hours and "mm" minutes ahead of UTC. A time zone offset of "-hh:mm" indicates that the date/time uses a local time zone which is "hh" hours and "mm" minutes behind UTC.

如果以协调世界时(UTC)表示的话，就会在最后直接加上一个Z;

如果以当地时间表示的话就需要指定时差(基于UTC)，以上海为例就应该是+08:00, 表示上海时间比协调世界时快8个小时。
而[BSON官方文档](http://bsonspec.org/#/specification)对于BSON Date的定义也是协调世界时(UTC datetime), 所以**MongoDB的日期是以UTC datetime存储的**，如下:

```bash
{
    ...
	"created_at" : ISODate("2016-01-27T09:51:02.051Z"),
	...
}
```

所以在实际编程过程中，各种语言的MongoDB Driver在反序列化ISODate时会根据当前的时区来进行转换，比如上面的`created_at`以中国为例的话，就应该是`2016-01-27 17:51:02`。

前面我们提到过BSON Date是从Unix Epoch以来的毫秒数，MongoDB Date中的**s**使用的是三位(即为s的位数)，即上面的051，也就是毫秒数，通过如下命令可以得到:

```bash
# mongodb js console
18> current
ISODate("2016-01-27T09:59:53.602Z")
19> current.getMilliseconds()
602
```

**(2)** MongoDB还有一种**供内部使用**的时间类型叫Timestamps，比如说**oplog**中有一个**ts**属性，就是使用该类型。一般我们开发的时候并不会使用这个类型。


##参考

\> [W3C对于datetime的规范](https://www.w3.org/TR/NOTE-datetime)

\> [MongoDB 时间类型概述](https://docs.mongodb.org/manual/reference/bson-types/#timestamps)

\> [MongoDB 时间类型的构造方法](https://docs.mongodb.org/manual/core/shell-types/)

\> [Decimal Time关于秒小数部分的解释](https://en.wikipedia.org/wiki/Decimal_time#China)

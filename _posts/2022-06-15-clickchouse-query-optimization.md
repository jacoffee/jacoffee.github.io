---
layout: post
category: database
date: 2022-06-16 22:53:03 UTC
title: 【ClickHouse实战】查询性能优化
tags: [OLAP数据库研究、ClickHouse、Vectorization Processing、Query Optimization]
permalink: /database/clickhouse/query-optimization
key:
description: 本文主要分享了个人在工作中的一些ClickHouse查询优化实践
keywords: [OLAP数据库研究、ClickHouse、Vectorization Processing、Query Optimization]
---

关于查询性能的优化，其实大致的方向就是: **干的少，做的快** -- 减少扫描的数量、网络IO，然后并行的处理数据。[知乎上的  关于如何进行sql优化？这个问题的回答](https://www.zhihu.com/question/637554270/answer/3346518807)我想很全面的总结了SQL优化大体思路, 而且感觉是非常OLAP方向，当然那个答主本身就是StarRocks工程师。

本文主要是分享和总结本人在运营分析平台构建业务模型SQL(事件、漏斗等)中的一些ClickHouse查询优化实践，ClickHouse版本21.3.15.4，基本都是分布式表(**3 Shard 2 Replica**)。**一些优化成果(可能与特定版本和环境有关**，所以请谨慎参考，但是思路是通用的，说明这个方向至少是可行的。


# 1. 调优实践

## 1.1 事件分析场景单group by替换多group by + union all

### 1.1.1 场景概述

事件分析主要是为了分析各种事件的指标的，比如说登录的UV、下单的UV等，还可以按照维度去统计，比如说某渠道，目前平台的事件分析支持配置多个指标

之前的做法是每一个指标 分别计算然后union all，SQL大致类似于这种:

```sql
select 
channel, count(distinct uid) as `登录的UV`, null as `下单的UV`
from event_record
where event_name = 'login'
group by channel

union all

select 
channel, null as `登录的UV`, count(distinct uid) as `下单的UV`
from event_record
where event_name = 'order'
group by channel
```

### 1.1.2 问题描述

之前采用这种做法主要是逻辑比较清晰，开发的时候也相对方便。但存在以下问题:

+ union all是并行查询的，但是同时带来的是重复数据扫描以及短时间内更大的内存使用，通过explain pipeline查看执行计划(样例SQL进行了简化，所以不能一一对应)，发现多次ReadFromPreparedSource。

```bash
ExpressionTransform                                                 │
│   (Aggregating)                                                   │
│   Resize 2 → 1                                                    │
│     AggregatingTransform × 2                                      │
│       StrictResize 2 → 2                                          │
│         (Expression)                                              │
│         ExpressionTransform × 2                                   │
│           (Union)                                                 │
│             (Expression)                                          │
│             ExpressionTransform                                   │
│               (MergingAggregated)                                 │
│               SortingAggregatedTransform 2 → 1
                  .....
│                 (ReadFromPreparedSource)                          │
│                           Remote × 2 0 → 1
│             (Expression)                                          │
│             ExpressionTransform                                   │
│               (MergingAggregated)                                 │
│               SortingAggregatedTransform 2 → 1                    │
│                 .....
│                 (ReadFromPreparedSource)                          │
│                           Remote × 2 0 → 1
```

+ 多union all时, Clickhouse偶尔会出现预计执行时间过长的报警

```bash
Estimated query execution time (654467.2455200038 seconds) is too long. Maximum: 3600. Estimated rows to process: 341736006: While executing MergeTreeThread (version 21.3.15.4 (official build))
```

经过排查这个报错，一方面是当前内存使用过高(当然union all也可以通过max_thread限制同时查询的线程数，变向的减少内存使用)，一方面是有可能是误报，因为Clickhouse预估查询机制会错误预估执行时间，[github issue 18872](https://github.com/ClickHouse/ClickHouse/issues/18872)。


### 1.1.3 优化方向

最终采用的方案是**单group by代替多union all(内存、IO 和 并行度的折衷)**，汇总所有指标的查询条件，取并集获取基础数据集。然后在聚合函数中，进一步过滤然后计算结果，也就是类似于countIf、sumIf之类的，SQL大致类似于这种:

```sql
select 
channel,
count(distinct case when event_name = 'login' and ( (1 = 1) ) then uid end) as `登录的UV`,
count(distinct case when event_name = 'order' and ( (1 = 1) ) then uid end) as `下单的UV`
from event_record
where 
-- 将指标的条件OR起来
(event_name = 'login' or event_name = 'order')
group by channel
```

通过explain pipeline查看执行计划(样例SQL进行了简化，所以不能一一对应)，只有单表相应的ReadFromPreparedSource。

```bash
(Expression)
ExpressionTransform
  (MergingSorted)
    (MergeSorting)
    MergeSortingTransform
      (PartialSorting)
      LimitsCheckingTransform
        PartialSortingTransform
          (Expression)
          ExpressionTransform
            (MergingAggregated)
            SortingAggregatedTransform 2 → 1
              MergingAggregatedBucketTransform × 2
                Resize 1 → 2
                  GroupingAggregatedTransform 3 → 1
                    (SettingQuotaAndLimits)
                      (Union)
                        (Expression)
                        ExpressionTransform
                          (Aggregating)
                          AggregatingTransform
                            (Expression)
                            ExpressionTransform
                              (Filter)
                              FilterTransform
                                (SettingQuotaAndLimits)
                                  (ReadFromPreparedSource)
                                  NullSource 0 → 1
                        (ReadFromPreparedSource)
                        Remote × 2 0 → 1
```


经过这个优化实践之后:
+ 在指标较多且有筛选条件有一定区分度的情况会有比较明显的效果，差不多是3倍
+ 如果查询条件过滤后仍然会导致很多数据被处理则group by优势不大，并行优势丢失且聚合过程开销较大

## 1.2 事件分析场景基于跳数索引优化URL查询

### 1.2.1 场景概述

某些站点事件分析模型中存在较多的URL相关的查询，比如说事件筛选条件、全局筛选条件，一般会采用like进行url查看，比如说url包含`https://host/path1/path2?cid=xxxx&templateType=yyyy`的事件量。

由于目前我们的事件表以事件时间作为分区，基于(事件名，站点，oneid)作为主键。所以使用url进行查询的时候，无法利用其它的优化，导致扫描出更多的数据，还有一定的优化空间。


### 1.2.3 实际优化

官网在[跳数索引](https://clickhouse.com/docs/en/optimize/skipping-indexes)章节提到了一个比较适用的场景就是**针对基数比较大的列而且比较离散**，像url搜索一般会针对哪些特定的活动页之类的或者详情页之类的，一般带有特殊ID，所以正好满足条件。

> Another good candidate for a skip index is for high cardinality expressions where any one value is relatively sparse in the data

经过研究决定使用跳数索引中的**布隆过滤器索引 -- tokenbf_v1**，本质上还是借助**布隆过滤器的高效存储**和**过滤功能**来辅助主键进行进一步的Granule过滤。关于布隆过滤器的基础知识，可自行查阅，此处不再赘述。

这种查询思路实际上类似于**Elasticsearch查询原理，通过分词存储数据，查询时候先进行分词然后再进行相应的匹配**。 tokenbf_v1会将存储的字串符，按照非字母的字符分割，由于url除了单词就是一些特殊字符(:、？、=)等，所以特别适合这种分词规则。

```sql
SELECT
tokens('http://googoom?ref=10&session_key=RUR&price&order][min]=200905&view/65550014031558
121-9H834Oq8j1LKFKhaj6Su-jitelstva/detay/forum/ford-of-town=1073.html&lang=ru&lr=213&textonli
ne.turken-100-prisova-baka/oborudovo/avtor_barry/regnum.ru/yandsearch?win=97e42c504fcb')

['http','googoom','ref','10','session','key','RUR','price','order','min','200905','view','65550014031558121','9
H834Oq8j1LKFKhaj6Su','jitelstva','detay','forum','ford','of','town','1073','html','lang','ru','lr','213','textonlin
e','turken','100','prisova','baka','oborudovo','avtor','barry','regnum','ru','yandsearch','win','97e42c504fcb']
```

tokenbf_v1只能运用于String、FixedString类型，tokenbf_v1由于基于BloomFilter实现，所以会有一定false positive概率，也就是在声称在里面的也可能不再里面，不过对于Clickhouse来说也只是多扫描一些底层数据，最终还是会在内存中再次筛选的。

这个从日志中也可以看出来：

```bash
xxx [ 34 ] {26b313c1-b7db-4c94-ae3b-a093381a61b6} <Debug> xxx (f1fe6faa-145f-417c-b25c-23051eb63c3a)  (SelectExecutor): Index `idx_url` has dropped 59/75 granules.
```

经过这个优化实践之后:

+ **查询效果受url基数和页面传递格式影响**， 当查询的url基数越大(High cardinality)，也就是越离散，比如说那种带id的页面，效果提升会更明显，因为可以进一步提高过滤的数据量
+ 页面查询指定的越通用的，比如如说直接写二级域名 'http://www.xxxx.com' ，效果越差，这个和离散程度实际上一一对应的

## 1.3 Join优化之事件表联表用户表时，双重过滤

### 1.3.1 场景概述

事件表记录的是埋点行为日志，当需要结合用户维表进行分析的时候，就需要联表。在我们的系统通过统一的OneId进行关联。SQL大致类似于这种:

```sql
select 
-- 指标相关SQL
from
(
	select
	oneid
	channel,
	event_name
	from event_record
	where 
	event_date = 'yyyy-MM-dd' and
	(event_name = 'login' or event_name = 'order')
) t1
left global join
(
	select 
	channel 
	from
	user_table 
	where event_date = 'yyyy-MM-dd'
) t2 on t1.oneid = t2.oneid
```

关于ClickHouse分布式表查询原理，可以参考[【数据库综述系列】ClickHouse综述(下)]()  的 关于Distributed Subqueries 部分。



### 1.3.2 问题描述

由于用户维表较大，有的站点可能达到上10亿，但是如果查询时用户维表上没有很有**筛选力度的条件的**，右表就会非常大。 在上例中，即使使用了global join，在我们的正常模型查询也是非常慢的，假设事件表选了一个周的范围。原因也很简单，当前版本的Join支持还是比较局限的再加上两个比较大的表join，磁盘、网络IO都是很夸张的。

### 1.3.3 优化方向

考虑到用户维表的主键是oneid  + 只有在事件表有行为的用户，才需要去联表，基于这点考虑，**在联表之前基于事件表的条件先捞出oneid，作为用户维表查询的一部分条件**，SQL大致类似于这种:

```sql
select 
-- 指标相关记录
from
(
	select
	oneid
	channel,
	event_name
	from event_record
	where 
	event_date = 'yyyy-MM-dd' and
	(event_name = 'login' or event_name = 'order')
) t1
left global join
(
	select 
	channel 
	from
	user_table 
	where event_date = 'yyyy-MM-dd' and oneid global in (
        -- 借助event_record的筛选条件 筛选出满足条件oneid
		select oneid from (
			select
				oneid
			from event_record
			where 
				event_date = 'yyyy-MM-dd' and
				(event_name = 'login' or event_name = 'order')
		)
	)
) t2 on t1.oneid = t2.oneid
```

经过这个优化实践之后，可以大大提高join的性能，目前**几乎所有的分析模型在用户关联的时候采用类似的方式**。核心原因就是 通过oneid的筛选条件降低 右边的数量，这样可以加快global join基于右表生成临时表的过程。


## 1.4 基于特定站点的大小表查询优化

### 1.4.1 场景概述

由于我们的埋点数据以ClickHouse作为数仓基座，然后按照部门站点进行的分表，底层形成一张大宽表。但是随着上报的字段不断增加，ClickHouse列越来越宽，即使有数据治理辅助，最大列宽几乎接近1000了。

### 1.4.2 问题描述

Altinity(一家ClickHouse专业提供商)，关于ClickHouse Wide Table 还是 Not Wide，有一篇[非常详细的博文](https://altinity.com/blog/too-wide-or-not-too-wide-that-is-the-clickhouse-question)介绍，里面有各种实验论证，甚至提供了建表时候如何选择数据结构。列太宽对于ClickHouse查询和写入都有巨大影响。 如果从原理层面来解释就是: 

ClickHouse在写入和读取的时候会按照列申请Compress Buffer，默认的是每列1MB，在[github issue 6943](https://github.com/ClickHouse/ClickHouse/issues/6943), ClickHouse CTO有专门回复过这个问题。

> It's related to compress buffer size and write buffer size. Both are 1MB by default.
> **It looks like buffers for all columns are allocated at once**, and 20 000 columns lead to about 40 GB of buffers.

当初发现这个问题就是，在监测ClickHouse写入的时候，发现某个表即使当次写入的有数据列的非常少，但是每次分配的内存也很大，后来结合Compress Buffer发现了这个问题。

关于Compress Buffer的扩展解释:

+ 为什么需要Compress Buffer

ClickHouse写入底层列数据的时候一般都会进行压缩，将内存数据从一个地方压缩到另外一个内存，这个过程可能需要一块临时内存存储中间数据进行一些额外处理，比如说计算checksum之类的， 也就是下面代码中提到的**compressed_buffer**。基于临时内存中的数据计算checksum，最后将checksum + 临时内存数据写到output buffer中。

```c++
// https://clickhouse.com/codebrowser/ClickHouse/src/Compression/CompressedWriteBuffer.cpp.html

void CompressedWriteBuffer::nextImpl() {
    
    /**
    * If output buffer does not have necessary capacity. Compress data into a temporary buffer.
      * Then we can write checksum and copy the temporary buffer into the output buffer.
    */
    // 分配 compressed_buffer 用于额外的数据处理
    compressed_buffer.resize(compressed_reserve_size);
    
    UInt32 compressed_size = codec->compress(working_buffer.begin(), decompressed_size, compressed_buffer.data());

    
    CityHash_v1_0_2::uint128 checksum = CityHash_v1_0_2::CityHash128(compressed_buffer.data(), compressed_size);
    
    writeBinaryLittleEndian(checksum.low64, out);
    writeBinaryLittleEndian(checksum.high64, out);
    out.write(compressed_buffer.data(), compressed_size);
}


```

+ 由于ClickHouse中的数据是分列写入的，所以每一个Column写入的时候都需要分配(If needed)

### 1.4.3 优化方向

问题描述部分基本描述清楚太宽的表，对于读写性能的影响。 另外从业务角度考虑，虽然底层表有几千列，但是分析模型常用的查询时间区间、使用筛选条件以及维度都是比较有限的，对应的底层表来说，就是可以基于近一个月使用的列，构建一张小表，**查询的时候通过特定的路由规则(分析平台的SQL目前都会基于Antlr4解析之后进行存储)**，来决定使用原始表还是小表。当然整个实现还包括很多细节，不过那个就不在本文的范畴内。 

经过这个优化实践之后，当查询命中小表之后， 大概有20%~30%提升。

## 1.5 函数不当使用

### 1.5.1 场景概述

在ClickHouse中，有些函数可能实现不那么高效或者使用的时候需要注意其场景。 

### 1.5.2 问题描述

当时在构建一个分析模型的时候，需要对于数组进行拆分，ClickHouse提供了arraySplit函数，关于相关语法可以参考官网，第二个参数是切分点，一般是包含1,0的数组。通常会想到使用arrayEnumerate来获取数组的索引，一般会配合arrayMap使用。

```sql
SELECT
    [100, 200, 500, 100, 150, 200] AS arr,
    arrayMap((x, index) -> if(index = 1, 1, (arr[index]) < (arr[index - 1])), arr, arrayEnumerate(arr)) AS split_rules,
    arraySplit((x, y) -> y, arr, split_rules) AS result
```



```bash
┌─arr───────────────────────┬─split_rules───┬─result────────────────────────┐
│ [100,200,500,100,150,200] │ [1,0,0,1,0,0] │ [[100,200,500],[100,150,200]] │
└───────────────────────────┴───────────────┴───────────────────────────────┘
```

在官网的github中的提供一个issue [How to reduce memory usage with arrayEnumerate? ](https://github.com/ClickHouse/ClickHouse/issues/5105), 详细解释了在这种情况内存使用"爆炸"的原因

### 1.5.3 优化方向

+ 选用其它的函数绕过，这个是我在实际开发中采用的方案
+ Github中提到的通过`max_block_size` 参数

```sql
create table X(A Array(String)) engine = TinyLog;

-- X表当前记录 1000
insert into X select arrayMap(x->toString (x) , range(1000)) from numbers(1000);

SET max_block_size = 1;

select arrayFilter(x->A[x]='777', arrayEnumerate(A)) from X format Null;
```

对于上面这个案例， 如果不加max_block_size，在我16G的Mac Pro上，直接跑失败了，就是这么夸张。

```bash
Code: 241. DB::Exception: Memory limit (total) exceeded: would use 16.04 GiB (attempt to allocate chunk of 4294967296 bytes), maximum: 14.40 GiB: while executing 'FUNCTION arrayElement(A :: 1, x :: 0) -> arrayElement(A, x) String : 3': while executing 'FUNCTION arrayFilter(__lambda :: 3, arrayEnumerate(A) :: 2) -> arrayFilter(lambda(tuple(x), equals(arrayElement(A, x), '777')), arrayEnumerate(A)) Array(UInt32) : 1'. (MEMORY_LIMIT_EXCEEDED) (version 22.6.1.823 (official build)) (from [::1]:51532) (in query: select arrayFilter(x->A[x]='777', arrayEnumerate(A)) from X format Null;), Stack trace (when copying this message, always include the lines below):
```

增加max_block_size参数之后，就能正常运行了。

```bash
<Information> executeQuery: Read 1000 rows, 11.35 MiB in 1.49055 sec., 670 rows/sec., 7.61 MiB/sec.
{ccb0adff-5836-498e-846f-acbb31eb6671} <Debug> MemoryTracker: Peak memory usage (for query): 20.56 MiB.
Ok.

0 rows in set. Elapsed: 1.492 sec. Processed 1.00 thousand rows, 11.90 MB (670.36 rows/s., 7.98 MB/s.)
```

官网关于ClickHouse配置部分，针对的[max_block_size](https://clickhouse.com/docs/en/operations/settings/settings)的解释，能非常清晰的解释为什么调小这个参数之后，能顺利跑过，简而言之就是：

ClickHouse中数据读取的单位是Block，**Block包含的读取的数据行**，减少每个Block的数据量从而减少了内存消耗，**属于时间换空间的操作**。显然这个参数不能调的太小，上面的测试案例只是为了演示而设置成了1。

经过这个优化实践之后，差不多有两倍的速度提升。




TO  BE  CONTINUED...


# 2. 调优思路分享

## 2.1 善用explain，观察执行计划

几乎所有的数据库都会提供explain来查看SQL的执行计划，通过结果我们大致可以看出SQL的执行流程、有没有使用索引之类的。通过SQL的改写然后查看执行计划，我们就知道优化是否产生了效果。

## 2.2 调整服务端日志级别，观察执行流程

ClickHouse在运行客户端的时候，可以通过`--send_logs_level='trace'`参数设置服务的日志级别，打开之后可以观察运行过程中实际发生了什么，也可以清晰的看到耗时发生在什么阶段。

+ Parts、Marks扫描数、基于索引扫描数等

```bash
Selected 20/706 parts by partition key,
17 parts by primary key,
27/554 marks by primary key,
27 marks to read from 17 ranges
```

+ 分组key对应的数量较多的时候, 会看到 AggregatingTransform 打印大量日志(多线程在执行聚合任务)且可以看到耗时

```bash
<Trace> AggregatingTransform: Aggregated. 0 to 0 rows (from 0.00 B) in 2.194896902 sec. (0.0 rows/sec., 0.00 B/sec.)
<Trace> AggregatingTransform: Aggregated. 0 to 0 rows (from 0.00 B) in 2.645799918 sec. (0.0 rows/sec., 0.00 B/sec.)
```

+ 如果执行报错，可以看到服务端调用栈打印出来，之后我们可以参照相应的源码去定位问题

```bash
22. DB::InterpreterSelectQuery::InterpreterSelectQuery(std::__1::shared_ptr<DB::IAST> const&, DB::Context const&, DB::SelectQueryOptions const&, std::__1::vector<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > > > const&) @ 0xea2fedd in /usr/bin/clickhouse
23. DB::InterpreterSelectWithUnionQuery::buildCurrentChildInterpreter(std::__1::shared_ptr<DB::IAST> const&, std::__1::vector<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > > > const&) @ 0xed69c75 in /usr/bin/clickhouse
24. DB::InterpreterSelectWithUnionQuery::InterpreterSelectWithUnionQuery(std::__1::shared_ptr<DB::IAST> const&, DB::Context const&, DB::SelectQueryOptions const&, std::__1::vector<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > > > const&) @ 0xed68570 in /usr/bin/clickhouse
25. DB::InterpreterFactory::get(std::__1::shared_ptr<DB::IAST>&, DB::Context&, DB::SelectQueryOptions const&) @ 0xe9e6d10 in /usr/bin/clickhouse
26. ? @ 0xef065a9 in /usr/bin/clickhouse
27. DB::executeQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::Context&, bool, DB::QueryProcessingStage::Enum, bool) @ 0xef05183 in /usr/bin/clickhouse
28. DB::TCPHandler::runImpl() @ 0xf69632d in /usr/bin/clickhouse
29. DB::TCPHandler::run() @ 0xf6a88c9 in /usr/bin/clickhouse
30. Poco::Net::TCPServerConnection::start() @ 0x11d5dfdf in /usr/bin/clickhouse
31. Poco::Net::TCPServerDispatcher::run() @ 0x11d5f9f1 in /usr/bin/clickhouse
```

## 2.3 多关注数据库对应社区中相关的分享

任何活跃的数据库社区都会积极推动社区发展，组织各种技术分享。这里面会有很多公司分享实战案例，有助于扩展我们的调优思维，毕竟每个开发者实际接触的场景是有限的。

就本人而言，会关注ClickHouse、StarRocks、Doris等相关的公众号，同时遇到相关问题会去Github上寻找答案，不仅能获取一些解决问题的思路更重要的是从底层原理理解为什么这样做可以。

## 2.4 技术方向固然重要，但业务场景能提供更切实的指导

本质上技术还是为业务服务的，所以在考虑优化方案的时候，一定将具体的使用场景纳入考虑，比如说上面提到的跳数索引以及大小表的方案，背后的设计都有业务场景的支撑。



# 3. 参考

\> [zhihu 如何进行sql优化？](https://www.zhihu.com/question/637554270/answer/3346518807)

\> [ClickHouse architecture](https://clickhouse.com/docs/en/development/architecture#merge-tree)

\> [ClickHouse和他的朋友们（6）MergeTree存储结构](https://bohutang.me/2020/06/26/clickhouse-and-friends-merge-tree-disk-layout/)

\> [ClickHouse sparse primary index](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes)

\> [C++ MergeTreeData.h](https://clickhouse.com/codebrowser/ClickHouse/src/Storages/MergeTree/MergeTreeData.h.html)
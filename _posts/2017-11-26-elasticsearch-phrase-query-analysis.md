---
layout: post
category: elasticsearch
date: 2017-11-26 06:30:55 UTC
title: 关于Elasticsearch中phrase query使用的一点总结
tags: []
permalink: /search/elasticsearch/phrase-query
key: 
description: 本文通过Elasticsearch sql中的A=B查询条件的延伸，阐述了phrase query中使用需要注意的地方
keywords: [Go-to search, 分词, Term, Term position]
---

今天在排查一个Elasticsearch sql查询问题时发现了phrase query使用过程中的一个问题，基本描述如下:

索引中某个属性A([ik_max_word分词](https://github.com/medcl/elasticsearch-analysis-ik))有下面一条记录:

```bash
%VIP%宣布鲁炜落马同一天 中纪委还发出另一个强烈信号
```

则下面的查询中(注意一般分词后，英文term都是小写的):

```bash
# 查询1
SELECT A FROM index_xx where A = 'vip'

# 查询2
SELECT A FROM index_xx where A = '宣布'

# 查询3
SELECT A FROM index_xx where A like 'vip'

# 查询4
SELECT A FROM index_xx where A like '宣布'
```

**除了查询2之外**，其它查询均能命中这条记录。

首先需要确认`=`和`like`在使用上的区别，Elasticsearch sql背后肯定是转换成了对应的Query。通过explain发现，前者转换成了**phrase query**，后者转换成了**wildcard query**。

```json
{
    "query": {
    	"bool": {
    		"must": {
    			"match": {
    				"A": {
    					"query": "vip",
    					"type": "phrase"
    				}
    			}
    		}
    	}
    }
}
```

```json
{
    "query": {
    	"bool": {
    		"must": {
    			"wildcard": {
    				"A": "vip"
    			}
    		}
    	}
    }
}
```

对于wildcard查询没什么好说的，肯定能匹配的。所以关键就是看phrase query在命中时做了些什么。

phrase query和match query一样首先也会将语句进行分词，不过需要注意的是，它进行的是<b style="color:red">相邻匹配</b>，也就是如果搜索**A B C**。则语句分词之后，必须在相邻的位置上出现**term A, term B, term C**，具体的解释如下:

<ul class="item">
    <li>原句分词之后，term A、B、C必须出现在索引中</li>
    <li>B的position必须比A大1</li>
    <li>C的position必须比A大2</li>
    <li>实际上就是A、B、C相邻</li>
</ul>

当同时满足上述条件的时候，该搜索才会命中文档。

通过**_analyze** rest api来查看原句分词之后的结果:

```bash
jacoffee:~ allen$ curl -XGET 'http://localhost:9200/xxx/_analyze?pretty=true' -d '
{
  "analyzer" : "ik_max_word",
  "text" : ["%VIP%宣布鲁炜落马同一天 中纪委还发出另一个强烈信号"] 
}'
{
  "tokens" : [ {
    "token" : "vip",
    "start_offset" : 1,
    "end_offset" : 4,
    "type" : "ENGLISH",
    "position" : 0
  }, {
    "token" : "宣布",
    "start_offset" : 5,
    "end_offset" : 7,
    "type" : "CN_WORD",
    "position" : 1
  }, {
    "token" : "宣",
    "start_offset" : 5,
    "end_offset" : 6,
    "type" : "CN_WORD",
    "position" : 2
  }, {
    "token" : "布鲁",
    "start_offset" : 6,
    "end_offset" : 8,
    "type" : "CN_WORD",
    "position" : 3
  }, {
    "token" : "布",
    "start_offset" : 6,
    "end_offset" : 7,
    "type" : "CN_WORD",
    "position" : 4
  }
  ...
```

由于采用最细粒度(**max_word**)的分词，所以**宣**和**布**也被拆分了，而且它们的position相差了2个，不满足上面提到命中条件，所以无法命中。

所以，phrase query从字面上可以理解为**语句匹配**，因此在使用的时候要特别注意。另外在分词的时候也要注意粒度，像宣布被拆开明显就不太合理。

## 参考

\> [phrase query](https://www.elastic.co/guide/en/elasticsearch/guide/current/phrase-matching.html)
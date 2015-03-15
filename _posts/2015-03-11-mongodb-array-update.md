---
layout: post
category: mongodb
date: 2015-03-11 22:03:22 UTC
title: MongoDB基础之数组类型元素的基本操作
tags: [文档，Document，数组，Array，WriteResult，回调，钩子，Callback]
permalink: /mongodb/array-update/
key: "484fcc7242ca50685c5fea00eb7394af"
description: "本文研究了mongodb中对于数组类型元素的相关操作: 增加，删除以及修改"
keywords: [文档，Document，数组，Array，WriteResult，回调，钩子，Callback]
---

今天，在开发一个产品功能的时候。需要对MongoDB中数组类型的元素进行相关操作，于是简单研究了一下。发现这种level的操作在某些情况下还是有些帮助的。

# 说明
本文使用的相关工具以及主要依赖的jar包：

<1> Lift框架

<2> MongoDB

<3> mongo-java-driver-2.10.1.jar

<4> lift-mongo_2.10-2.5.1.jar

# 解决

<h3 style="text-indent: 25px;">MongoDB代码</h3>


```json
"collection_name": "date_serializers"

{
	"_id" : ObjectId("54f6e3e1d4c678d4a1f7b8c9"),
	"created_at" : ISODate("2015-03-04T10:52:17.762Z"),
	"updated_at" : ISODate("2015-03-04T10:52:17.649Z"),
	"d_student" : [
		{
			"name" : "zml",
			"startYear" : "2008"
		},
		{
			"name" : "xxl",
			"startYear" : "2010"
		}
	]
}
```

<h3 style="text-indent: 25px;">Scala代码</h3>

Model层: MongoModel，MongoModelMeta分别是对于MongoRecord, MongoMetaRecord的封装。
created_at和updated_at来源于MongoModel。

````scala
class DateSerializer extends MongoModel[DateSerializer] {
 def meta = DateSerializer

 object d_student extends 
MongoJsonObjectListField[DateSerializer, Graduate](this, Graduate)
}

object DateSerializer extends DateSerializer with MongoModelMeta[DateSerializer] {
	override val collectionName = "date_serializers"

	/*
     def updateWithResult(qry: JObject, newobj: JObject, opts: UpdateOption*) = {
		useDb { db =>
			val dboOpts = opts.toList
			db.getCollection(collectionName).update(
				qry,
				newobj,
				dboOpts.find(_ == Upsert).map(x => true).getOrElse(false),
				dboOpts.find(_ == Multi).map(x => true).getOrElse(false)
			)
	    }
    }
	*/

	/*
		("_id" -> id) ~ ("d_student.name" -> name)
		注意这个查询的条件的level会决定后面的WriteResult中n的结果的"准确性"，如果("_id" -> id)能够匹配到文档，那么n永远是1，updatedExisting永远是true。这样就无法根据n来判断操作是否成功，<b style="color:red">因为我们操作不是top_level文档</b>。

		加大查询条件的力度("_id" -> id) ~ ("d_student.name" -> name), 使它到Array的element级别，这样n加上updatedExisting以及err就能判断是否成功操作了。
	*/

	def updateSegmentById(id: ObjectId, name: String, newOne: JValue) = {
		updateWithResult(("_id" -> id) ~ ("d_student.name" -> name), ("$set" -> ("d_student.$" -> newOne)))
	}

	def addSegmentById(id: ObjectId, newOne: JValue) = {
		updateWithResult(("_id" -> id), ("$addToSet" -> ("d_student" -> newOne)))
	}

	
	def deleteSegmentById(id: ObjectId, name: String) = {
		val matchedToDelete: JObject = "d_student" -> ("name" -> List(1,3))
		updateWithResult(
			("_id" -> id) ~ ("d_student.name" -> name), "$pull" -> matchedToDelete
		)
	}
}

object Graduate extends JsonObjectMeta[Graduate]
case class Graduate(name: String, startYear: String) extends JsonObject[Graduate] {
	def meta = Graduate
}
```

几点说明:

<1> 代码中updateWithResult实际上是对于mongo-java-driver-2.10.1.jar中DBCollection方法的封装。

```java
public WriteResult update( 
    DBObject q , DBObject o , 
    boolean upsert , boolean multi ) {
    return update( q , o , upsert , multi , getWriteConcern() );
}
```

所以updateWithResult实际上是对于数据库底层的操作，并没有经过Lift的封装。另外当我们需要知道<b style="color:red">Update操作的返回值的时候，我们需要使用WriteResult这个类的相关方法</b>。

<2> WriteResult的getLastError会返回一个stringify的Json格式的字符串，格式如下:

```json
{ 
  "serverUsed" : "/127.0.0.1:27017" , "connectionId" : 38 , "updatedExisting" :   true , "n" : 1 , "syncMillis" : 0 , "writtenTo" :  null  , "err" :  null  , "ok" : 1.0 
}
```

n表示的是如果当前操作是更新或者删除时，被更新或者是删除的文档(top-level)数，这里面实际上隐藏了一层意思，n是建立在匹配到文档之上的。然后结合updatedExisting和err我们可以判断，上面的操作是否成功。

<3> 前面提到了updateWithResult直接走的底层数据库，所以我们在框架层面定义的<b style="color:red">callback，俗称钩子(afterSave, afterUpdate...)</b>。所以需要手动去触发框架层面的保存以保证updated_at字段的更新。

<4> 由于前面的Model层代码已经说明了三种操作的代码，因此下面着重讲MongoDB中三种操作的使用，Scala代码层面的调用就不提了。

<h3 style="text-indent: 25px;">更新操作</h3>

对于上述这个简单的Document，如果我们要更新数组中name为zml的元素，就必须首先对该元素在数组中的位置进行<b style="color:red">定位</b>这样我们才能对它进行操作。

它的核心是用到的了MongoDB的Update Operator以及Operato $。基本语法:

> db.collection.update(
   { <array>: value ... },
   { <update operator>: { "<array>.$" : value } }
)

```shell
db.date_serializers.update({"_id": ObjectId("54f6e3e1d4c678d4a1f7b8c9"), "d_student.name": "zml"}, {<b style="color:red">"$set"</b>: {"d_student.$": {"name": "zml_new", "startYear":"2018"}}})
```

如果更新成功，mongo shell会提示如下:

WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

匹配一个文档(top-level), 修改一个文档(top-level), 没有文档被Upserted。

"d_student.\$"用于定位匹配到的Array里面的element。这种用法有一个限制，如果update第一个参数的查询条件没有找到匹配的文档，那么会报$定位错误，如下：

```scala
WriteResult({
	"nMatched" : 0,
	"nUpserted" : 0,
	"nModified" : 0,
	"writeError" : {
		"code" : 16836,
		"errmsg" : "The positional operator did not find the match needed from the query. Unexpanded update: d_student.$"
	}
})
```

<h3 style="text-indent: 25px;">新增操作</h3>

$addSet operator能向Array添加不重复的元素，实际上会遍历Array的element，如果都不相等的话，就插入。否则，什么都不做。所以，这种操作适合<b style="color:red">Array中element不太多且结构不会太复杂的</b>，否则就会有性能问题
。

```shell
db.date_serializers.update({"_id": ObjectId("54f6e3e1d4c678d4a1f7b8c9")}, {"$addToSet": {"d_student": {"name": "tuniu", "startYear":"2020"}}})

WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

注意$addToSet的后面key的结构。另外我们知道MongoDB的数据结构修改起来是非常容易的

```shell
db.date_serializers.update({"_id": ObjectId("54f6e3e1d4c678d4a1f7b8c9")}, {"$addToSet": {"d_student": {"name": "tuniu"}}})
```

上面这种操作也是Okay的，对于数据库层面没有问题但是对于业务逻辑和代码层面就有问题，所以在实际开发中我们要确保"d_student"的Json格式的数据满足我们定义的数据结构，如上面的graduate的结构，即序列化和反序列化的时候要保证数据结构的统一。

<h3 style="text-indent: 25px;">删除操作</h3>

$pull操作符可以用于删除Array中符合条件的element，如果没有匹配的什么都不做。

```shell
db.date_serializers.update({"_id": ObjectId("54f6e3e1d4c678d4a1f7b8c9"), "d_student.name": "zml"}, {"$pull": {"d_student": {"name": "xxl"}}})

db.date_serializers.update({"_id": ObjectId("54f6e3e1d4c678d4a1f7b8c9")}, {"$pull": {"d_student": {"name": "xxl"}}})

WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

对于第二个查询条件"d_student.name": "zml"在MongoDB中实际上可以省略，因为凭借WriteResult的提示，你可以看出实际操作有没有成功。加上只是为了代码层面能够很好的辨识。

上面的操作在没有数据库异常的情况下，有可能出现nModified < nMatched。

如果我选取第二种查询方式，顺利匹配到top-level document，"nMatched" : 1。但是后面的"name": "value"没有匹配到值，"nModified" : 0。因为找不到要删除的Array的element。


# 结语
MongoDB Array的element operation还有很多，本文只简单的介绍了一部分。其它的有兴趣的可以参阅官方文档，
[Array Operation](http://docs.mongodb.org/manual/reference/operator/update/positional/)














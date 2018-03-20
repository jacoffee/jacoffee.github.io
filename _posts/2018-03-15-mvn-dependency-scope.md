---
layout: post
category: maven
date: 2018-03-15 15:30:55 UTC
title: Maven项目中不同环境设置不同的依赖范围(scope)
tags: [Scope, Dependency, Process]
permalink: /mvn/dependency-scope
key: 
description: 本文介绍了如何在Spark项目中，在开发和部署的时候使用不同的jar包依赖
keywords: []
---

在Spark相关的项目中，Spark相关的依赖(spark-core, spark-sql)都可以在spark-submit提交时由spark提供，所以可以设置成provided，避免在使用maven-shade-plugin打成fat jar的时候包太大。但另外一个问题马上出现，provided依赖在编译的时候没有问题，但是在运行的时候就会出现问题。

所以，我们需要解决的问题是: 如何在本地开发时使用默认的compile依赖，然后在打包的时候变换成provided依赖。比较直观的一种做法就是 -- 打包时动态更改pom.xml，输出到build.xml中，通过mvn相关命令打包，与此同时显示指定pom文件的地址，即动态修改后并输出的pom文件。

## pom.xml文件修改

pom.xml文件修改实际上就是对于XML节点的处理，Scala提供了scala.xml.Node对其进行抽象。

```scala
abstract class Node extends NodeSeq { 

  // Node的标签
  def label: String
  
  // 命名空间
  def namespace = getNamespace(this.prefix)

  // 命名空间相关的绑定 xmlns:xx, xmlns:yy；默认是没有的，除了预定义xml
  // case class NamespaceBinding(prefix: String, uri: String, parent: NamespaceBinding)
  def scope: NamespaceBinding = TopScope   

}
```


```xml
<!-- XML node ->
<dependency
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://maven.apache.org/POM/4.0.0">
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-sql_2.10</artifactId>
  <version>1.6.2</version>
</dependency>
```

上面的XML Node中命名空间(xmlns后面不带任何东西)为**http://maven.apache.org/POM/4.0.0**，标签为**dependency**。命令空间绑定的父子关系是通过书写的顺序来形成的，比如说上面的**xmlns:xsi**就是**xmlns**的父NamespaceBinding，这一点在下面我们清除命令空间绑定的时候会使用到。

所以，我们需要锁定pom.xml中的dependency Node然后，为它的子Node添加一个Node，即`<scope>provided</scope>`，通过Scala的模式匹配，可以很容易做到这一点。

```scala
val xml = XML.load("/xx/xx/pom.xml")
val newPom =
  xml match {
    case e @ Elem(_, _, _, scope, nodes @ _*) => {
      val changedSubNodes =
        nodes.map { node =>
          if (node.label == "dependencies") {
            node match {
              case <dependencies>{ dependencies @ _* }</dependencies> => {
                val changedChild =
                  dependencies.flatMap { dependencyNode =>
                    val artifactId = (dependencyNode \ "artifactId").text
                    dependencyNode match {
                      case elem: Elem =>
                        val toBeAdded =
                          if (artifactIds.contains(artifactId)) {
                            elem.child ++ appendedNode
                          } else {
                            elem.child
                          }
                        Some(elem.copy(child = toBeAdded))
                      case other => Some(other)
                    }
                  }
                <dependencies>{ changedChild }</dependencies>
              }
            }
          } else {
            node
          }
        }
      // 清除子Node命名空间
      e.copy(child = changedSubNodes.map(clearScope))
    }
    case _ => xml
  }
newPom
```

生成之后，通过**XML.save**输出到文件中(build.xml)。

## 构建进程对象并执行

由于更改后的pom文件(build.xml)已经产生，所以可以直接运行**mvn clean package -f /xx/build.xml**，但既然已经在程序中将pom文件生成了何不直接通过Process对象执行相关的命令，有如下几点需要注意:

<ul class="item">
    <li>
mvn命令以空格分隔，然后构建成数组，作为ProcessBuilder的构造参数，<b>mvn clean package -f /xx/build.xml</b> ---> <b>Array("mvn", "clean", "package", "-f", "/xx/build.xml")</b>
    </li>
    <li>
创建的进程为当前进程的子进程(subprocess)并且没有自己的终端，它所有和I/O相关的东西(i.e. stdin, stdout, stderr)都被重定向到了父进程，我们可以通过getInputStream()、getOutputStream()、getErrorStream()等方法获取
    </li>
</ul>

```scala
def mvnPackage(pomPath: String) = {
  val cmds = List("mvn", "clean", "-DskipTests", "package", "-f", pomPath)
  val processBuilder = new ProcessBuilder(cmds.toArray: _*).redirectErrorStream(true)
  
  var process: Process = null
  try {
    process = processBuilder.start()
    // mvn package命令运行之后的日志输出
    val bufferedSource = Source.fromInputStream(process.getInputStream())
    bufferedSource.getLines().foreach { line =>
      println(line)
    }
  } catch {
    case ex: Exception =>
      throw ex
  } finally {
    if (process != null) {
      process.destroyForcibly()
    }
  }
}
```

虽然，本文中提到的方法实用性并不是太高，但这种解决问题的方式个人觉得还是挺有意思的，在此过程中，进一步熟悉了Scala XML操作、Java中代码中操作Process等知识点。完整版代码，请参照[Build.scala](https://github.com/jacoffee/codebase/blob/master/src/main/scala/com/jacoffee/codebase/Build.scala)

## 参考

\> [W3C Xml教程](http://www.w3school.com.cn/x.asp)

\> [Maven依赖详解](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)

\> [Maven打包时指定输出文件夹路径](https://stackoverflow.com/questions/4757426/maven-specify-the-outputdirectory-only-for-packaging-a-jar)
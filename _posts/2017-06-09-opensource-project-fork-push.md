---
layout: post
category: opensource
date: 2017-06-09 06:38:55 UTC
title: 获取开源项目并发布到Maven私库的流程
tags: [Credentials，Maven私库，Git Tag]
permalink: /opensource/fork-and-push
key: 
description: 本文介绍如何拉取开源项目，修改并且发布到Maven私库
keywords: [Credentials，Maven私库，Git Tag]
---

在实际开发中，我们会使用很多开源项目，有时我们需要对其中的一些逻辑进行修改或者添加一些功能，这时候就需要获取源码并且做相应改动。为便于团队成员共享，需要将修改好的代码再次发布，基本流程如下，以修改[elastic4s](https://github.com/sksamuel/elastic4s)并发布为例。

## 获取项目源码

通过**git clone**获取源码之后，我们需要考虑基于那个版本进行改动。基于git的项目，每发布一个版本就会对应一个tag，我们需要做的就是<b class="highlight">基于tag切出一个分支然后进行改动</b>。

```bash
# 列出tag
git tag

v1.1.1.0
v2.2.0
v2.2.1
v2.3.0
v2.3.1
v2.3.2

# 基于v2.3.2切出开发分支
git checkout -b branch_name v2.3.2
```

## 发布到远程仓库

修改完成之后，接下来我们需要将项目编译并部署到远程Maven私库中。这里涉及到的一个重点就是: <b class="highlight">配置相应的认证信息(credentials)</b>

<b class="highlight">(1) Maven中配置</b>

Maven的部署主要借助[deploy插件](http://maven.apache.org/plugins/maven-deploy-plugin/)

```bash
# 部署命令
mvn deploy
```

相应的crentials配置: **${HOME}/.m2/settings.xml**

```xml
<servers>
    <server>
      <id>nexus-releases</id>
      <username>username</username>
      <password>passwd</password>
    </server>

    <server>
      <id>snapshots</id>
      <username>username</username>
      <password>passwd</password>
    </server>
</servers>
```

pom.xml中的repository配置，**pom.xml中的repository优先级高于settings.xml中的，而pom.xml中的repository又是按照顺序来依次尝试的**。

```xml
<repositories>
    <repository>
      <id>releases</id>
      <name>Private Releases Repository</name>
      <url>http://hostname/nexus/content/repositories/releases/</url>
    </repository>
    <repository>
      <id>snapshots</id>
      <name>Private Snapshots Repository</name>
      <url>http://hostname/nexus/content/repositories/snapshots/</url>
    </repository>
    <repository>
      <id>aliyun</id>
      <name>Aliyun Maven Repository</name>
      <url>http://Maven.aliyun.com/nexus/content/groups/public/</url>
    </repository>
    <repository>
      <id>central</id>
      <name>Maven Central Repository</name>
      <url>http://repo1.Maven.org/Maven2/</url>
    </repository>
</repositories>
```

<b class="highlight">[(2) sbt中配置](http://www.scala-sbt.org/0.13/docs/Publishing.html)</b>

主要包括如下几个部分:

<ul class="item">
    <li>域(Realm)一般是: <code>Sonatype Nexus Repository Manager</code></li>
    <li>主机名: com.xxx.xxx 或者是 ip地址(不需要端口)</li>
    <li>Nexus repository用户名</li>
    <li>Nexus repository密码</li>
</ul>

接下来我们需要在Build.scala或者build.sbt中配置如下信息:

```scala
// 确保生成相应的pom文件
publishMavenStyle := true

credentials += Credentials("Sonatype Nexus Repository Manager", hostname, username, passwd)

// 根据版本发布到不同的仓库中去
publishTo <<= version {
  (v: String) =>
    // val repo = "http://hostname"
    val repo = "http://ip:port/"
    if (v.trim.endsWith("SNAPSHOT"))
      Some("snapshots" at repo + "nexus/content/repositories/snapshots")
    else
      Some("releases" at repo + "nexus/content/repositories/releases")
}
```

之后便可以通过**sbt publish**进行发布，不过即使上面的credentials信息无误，也会[报权限错误](https://stackoverflow.com/questions/16425639/sbt-publish-to-corporate-nexus-repository-unauthorized)，目前需要通过[sbt-aether-deploy](https://github.com/arktekk/sbt-aether-deploy)插件解决，在**plugins.sbt**中添加**addSbtPlugin("no.arktekk.sbt" % "aether-deploy" % "0.19.0")**，
将发布命令改为**sbt aether-deploy**即可。

对于多模块项目，如果想控制发布到私库的项目，可以通过aggregate来定义需要发布的子模块。

```scala
// build.sbt
lazy val root = Project("elastic4s", file("."))
  // 不发布package生成的main jar
  .settings(publishArtifact := false)
  .settings(name := "elastic4s")
  // 控制需要发布的子模块
  .aggregate(core, streams)
```

## 参考

\> [sbt发布](http://www.scala-sbt.org/0.13/docs/Publishing.html)




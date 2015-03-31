---
layout: post
category: scala
date: 2015-03-31 09:51:22 UTC
title: Scala基础之三引号(""")
tags: [三引号，转义字符，原生字符串]
permalink: /scala/quote/
key: "67c0f6abd80ace3b9ddd7984da650b52"
description: "本文简单的介绍了三引号在Scala中的作用以及运用"
keywords: [三引号，转义字符，原生字符串]
---

今天在Scala(version 2.11.4 )中写一个正则表达式的时候写了很多的转义字符如\\d, \\s，不过印象中三引号包裹的正则表达式是不用写转义字符的，于是研究了一下，记录于此。

# 问题
Scala中三引号(三个双引号)的作用是什么？一般用在什么地方？

# 解决

<1>  输出多行字符串

Scala Reference中的定义:
> A multi-line string literal is a sequence of characters enclosed in triple quotes """ ... """

三引号可以用来输出多行字符串。

```scala
val foo = """this is triple double quotes test,
       when you see the result, you will know that clearly,
       LOL
"""
// result1
foo: String =
"this is triple double quotes test,
       when you see the result, you will know that clearly,
       LOL"
       
val foo1 = """this is triple double quotes test,
     | when you see the result, you will know that clearly,
     |LOL
""".stripMargin

// result2 = 
foo1: String = """this is triple double quotes test,
 when you see the result, it will know that clearly, // | 和when之间的还是留下了
LOL      
"
```

注意result1中第二，三行前面的空格也被保留了。 Scala library 提供了一个 stripMargin方法，<b style="color:red">只需要在每一行除了第一行开头加上|</b>即可，
结果如 result2。

<2> 直接输出原生字符串(换行符，双引号，单引号等)

```scala
// 省略多余的转义字符
val ipPattern = """(?:(?:\d{1,3}\.){3}\d{1,3})"""

// 直接输出单引号，双引号
val mixedPattern = """"(?:(?:\d{1,3}\.){3}'\d{1,3})""""
```

虽然要多写引号，但是去除掉那些多余的转义字符还是不错的。


# 参考

<1>  Scala Reference

<2> [why-does-intellij-idea-offer-to-enclose-a-string-literal-to-triple-double-quotes](http://stackoverflow.com/questions/8024245/why-does-intellij-idea-offer-to-enclose-a-string-literal-to-triple-double-quotes)



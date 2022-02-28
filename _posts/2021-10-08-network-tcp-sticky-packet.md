---
layout: post
category: network
date: 2021-10-08 20:51:22 UTC
title: 【网络】TCP基础之粘包和拆包
tags: [TCP协议，粘包拆包]
permalink: /network/tcp/sticky-packet
key: 
description: 本文主要阐述了TCP拆包和粘包产生的原因以及解决方案
keywords: [TCP协议，粘包拆包]
---

# 1. 粘包与拆包(sticky packet)

首先外文文档中是没有拆包这个概念，只有粘包的概念。 本质上是由于TCP是**流式协议(stream protocol)**，可以将TCP协议类比成一根水管。A向B分两次分别运输了2升水, TCP协议保障：

+ 水一定是按照顺序抵达B
+ 到达B的水的容量一定是等于A发送出去的

但发送过程中可能有延迟， 运输的可能会将水管的中的水进行合并(The pipe may **merge multiple segments of water streams** because of the nature of water pipe)。

正是由于对TCP Segment的合并，所以导致发送出去的数据被合并，这就是所谓的**粘包**。 显著的影响就是小文件系统中多个客户端同时向一个server发送 上传图片的请求，不同图片的字节可能被合并后发送过去了，所以解析时需要做额外处理。相较于UDP是基于packet的协议，数据总是packet为单位，统一发送。

# 2. 如何解决

粘包本质上来讲是TCP的特性，并不是什么问题。只是我们需要额外处理，通用的思路就是`传递一种标识，告诉接收者什么时候消息发送完了`

## 2.1 利用特殊分割符

当接受者读取到特殊分隔符之后，标识任务消息已经全部传递完毕。典型的如HTTP/1.1，**利用两个换行符来标识消息完成传递**

## 2.2 传递消息长度

在传递的消息中存储当次消息总长度，这样接收者就依据此来判断是否处理完消息

```bash
【content length】【content】
````

# 3. Java NIO中如何解决

默认没有解决，需要我们自己基于ByteBuffer去解决，典型的就是**定义ByteBuffer消息的格式**：每个部分存储什么、总长度等，这样解析的时候就可以读取指定长度来作为一个消息。以小文件系统中，上传图片的格式为例:

+ 协议格式

| 属性         | 长度  | 字段名            |
| ------------ | ----- | ----------------- |
| 请求类型     | 4字节 | requestType       |
| 文件名长度   | 4字节 | fileNameLength    |
| 文件名       | N字节 | fileName          |
| 文件内容长度 | 4字节 | fileContentLength |
| 文件内容     | N字节 | fileContent       |

+ 构造上传内容的ByteBuffer

```java
///////////// 协议相关的长度定义
public static final int REQUEST_TYPE_LENGTH = 4;

public static final int FILE_NAME_LENGTH = 4;

public static final int CLIENT_RESPONSE_LENGTH = 4;

public static final int FILE_CONTENT_LENGTH = 4;


//////////// 计算总长度

/**
* 注意中文转换UTF-8 一个中文转换成UTF-8编码的时候是三个字节
* @return
*/
private int capacityForUpload() {
	byte[] utf8Bytes = fileInfo.getFileName().getBytes(StandardCharsets.UTF_8);
	return REQUEST_TYPE_LENGTH + FILE_NAME_LENGTH + utf8Bytes.length + FILE_CONTENT_LENGTH + fileInfo.getContent().length;
}

//////////// 构造ByteBuffer

// 按照总长度分配容量，每次消费的时候，如果ByteBuffer为空，这说明已经结束
ByteBuffer byteBuffer = ByteBuffer.allocate(capacityForUpload());

// 设置类型
byteBuffer.putInt(type);

// 设置文件名长度
byte[] fileNameBytes = fileInfo.getFileName().getBytes(StandardCharsets.UTF_8);
byteBuffer.putInt(fileNameBytes.length);
// 设置文件名长度
byteBuffer.put(fileNameBytes);

// 设置文件内容长度
byteBuffer.putInt(fileInfo.getFileSize());
// 设置文件内容
byteBuffer.put(fileInfo.getContent());
```

# 4. 参考

\> [mp TCP拆包与粘包](https://mp.weixin.qq.com/s/cdJ7LbH-_uVWz9BZgOuMtg)
---
layout: post
category: spring
date: 2020-10-13 19:55:03 UTC
title: 【网络基础】Https
tags: [网络基础, 4层网络协议，SSL/TLS, 数据链路层，、网络层、传输层、应用层]
permalink: /network/https
key:
description: 本文介绍了应用层协议Https的基础知识以及实际应用
keywords: [网络基础, 4层网络协议，SSL/TLS, 数据链路层，、网络层、传输层、应用层]
---

[TOC]

#  1. 基础概念

## 1.1 对称加密(**Symmetric Cryptography**) vs 非对称加密(Asymmetric Cryptography)

前者，通讯双方会约定好**加解密算法以及密钥**，核心点是**如何管理密钥**，像我们在API签名时，一般会提前约定好密钥。

后者，一般是使用公钥和私钥两种密钥来进行通讯，一般是公钥加密然后私钥解密

## 1.2 session key

通讯双方在完成SSL/TLS握手之后，双方完成身份确认、加密算法确认之后，会生成**session key**。实际上就是一个对称加密key, 可用于后续通讯的加密，后面就不用再使用公钥和私钥了，每个会话都会生成唯一的session key。

阿里云开发社区中的[使用HTTPS防止流量劫持](http://yq.aliyun.com/articles/2666)一文中提到如何通过加密的请求来防止发出的请求由于**DNS劫持(将请求的IP解析到中间人的网站IP)或是内容篡改**而出现安全问题或是请求的

页面被加上了一些原本没有的内容。于是对于HTTPS请求建立过程中的消息加密过程以及安全沟通的连接产生了兴趣，本文也将从以上的两个方面展开。

# 2. [公钥加密算法(public-key cryptography)](https://en.wikipedia.org/wiki/Public-key_cryptography)

也称为**非对称加密算法**(顾名思义就是使用一把key加密，使用另一把key解密， 如下图所示)。密钥有两把，一把是公开的公钥，还有一把是不公开的私钥。双钥加

密遵循如下原则:

+ 公钥和私钥是一一对应的关系，有一把公钥就必然有一把与之对应的、独一无二的私钥，反之亦成
+ 所有的(公钥, 私钥)对都是不同的
+ **用公钥可以解开私钥加密的信息，反之亦成立**
+ 同时生成公钥和私钥应该相对比较容易，但是从公钥推算出私钥，应该是很困难或者是不可能的
+ **公钥用来加密信息，私钥用来数字签名**

![public-key-encrypt-private-key-decrypt](/static/images/charts/2020-10-13/public-key-encrypt-private-key-decrypt.png)

下面以**用户向目标站点发起付款请求**，其实也就是输入用户支付密码付款的流程来综合解释对称加密、非对称加密以及HTTPS在互联网通讯中的作用。

HTTP请求就不用说了，消息并没有加密，所以中间人很容易截获我们支付密码。

**对称加密**: 这一过程直接**使用对称加密**的问题是没有一个很好的方案**保证密钥的安全传输**，如果在网络上传输，很容易被中间人截获然后获取我们的私密信息。

**非对称加密**: 我们使用目标站点的公钥加密信息，然后目标站点使用私钥进行解密。这样的中间人即使截获我们的私密信息，但由于没有目标站点的私钥所以无法解密。

但是这个引入另外一个问题，我们怎么获取目标站点的公钥以及如何确保获取的公钥的就是目标站点的(也就是公钥的权威性如何确定)？

+ 针对获取的话，肯定还是需要通过网络传输
+ 另外验证的话，由于每个人都可以生成公钥和私钥，所以**公钥的合法性就需要一个权威机构进行背书**，为目标站点生成证书，以证明其身份，这样就能确保我们在和正确的人在交互

# 3. 数字证书(Digital Certificates) -- 确保证书合法性以及和正确的人通讯

上面我们提到了网站公钥的权威性问题，需要专业机构来背书，这时候就需要引入数字证书: 它一般是由一个权威的第三方机构认证颁发的，就像一个网站的身份证，由如下几个部分组成:

+ 申请认证机构的名字
+ 申请认证机构的公钥 -- 目标站点的公钥
+ 过期时间
+ 证书的发布者
+ 证书颁发者的数字签名

承接上例，**目标站点如果向CA(certificate authority)申请数字证书成功之后**，CA会使用自己的私钥对用户的一些信息包括公钥等进行加密，生成数字证书。

我们在浏览器(自身有一个信任的**认证机构的公钥列表**)中进行通讯的时候，浏览器会验证对方网站的证书进行验证(**注意这并非是SSL要求而是现代浏览器一般会做证书检查**):

+ 日期检查(**date check**): 确保证书在有效内

+ 证书颁发者身份验证(**signer trust check**): 浏览器会默认维护一系列受认证机构的公钥，当收到访问网站发来的证书时，就会尝试用这些公钥去解密(**签发时，认证机构使用自己的私钥加密，所以这个地方如果某个机构颁发过肯定能解开**)

+ 签名校验(**Signature check**): 浏览器获取到相应的CA公钥之后，对于signature进行处理然后和checksum进行比较，以确认是否合法

下面是浏览器展示的Baidu的CA样式

![浏览器展示Baidu的CA](/static/images/charts/2020-10-13/baidu_ca.png)

# 4. 数字签名(Digital Signature) -- 确保信息没有被篡改

附着在消息上面的加密checksum(**cryptograpic checksum**), 确保信息没有被篡改

+ **验证消息发送者的身份** -- 一般发送者使用自己的私钥对于内容摘要(**message digest**)进行加密生成checksum

+ **验证消息的完整性确保没有被篡改** -- 接受者拥有发送者的公钥，通过非对称加密获取解密后的摘要(**message digest**)，然后自己根据约定的算法计算出发送内容的摘要, 两者进行对比则可判断内容有没有篡改(**tamper**)

![Https加密流程](/static/images/charts/2020-10-13/Https加密流程.png)

# 5. HTTP的不安全体现

+ 明文传输，容易暴露隐私信息
+ 无法验证对方的身份信息，这也是中间人攻击的根源
+ 基于第一点，消息容易被篡改，典型的如**DNS流量劫持、页面被强塞了广告**等

# 6. HTTPS -- HTTP Over SSL

**Secure socket layer / transport layer security**

在HTTP权威指南中对于HTTPS有如下一段定义:

> HTTPS combines the HTTP protocol with a powerful set of symmetric, asymmetric, and certificate-based cryptographic techniques

HTTPS将HTTP协议以及一系列强大的对称的(**SSL Handshake结束之后双方后续使用约定的session key进行对称加密通讯**)，非对称的(**通讯双方协调生成通讯密钥的过程使用了该种加密算法**)以及基于证书的加密技术进行整合，以提供更加安全的通讯。

当使用HTTPS协议的时候，所有HTTP请求和响应都会先进行加密和解密(**在TLS/SSL层做，使用session key**)，然后进行后续流程。

>  Once the TCP connection is established, the client and server initialize the SSL layer, **negotiating cryptography parameters** and **exchanging keys**. When the handshake completes, the SSL initialization is done, and the client can send request messages to the security layer. These messages are encrypted before being sent to TCP

客户端发起HTTPS请求，服务器端监听在443(the default port for secure HTTP)。一旦TCP请求建立以后，客户端和服务器端开始建立SSL层，磋商加密参数以及Session Key。一旦SSL Layer建立以后，双方就可以使用Session Key加密要传递的内容，从而进行安全的传输。

> Server and Browser now encrypt and decrypt all transmitted data **with the symmetric session key**. This allows for a secure channel because **only the browser and the server know the symmetric session key**, and the session key is only used for that session. If the browser was to connect to the same server the next day, a new session key would be created.

到此总结下HTTPS是如何解决上面提到的HTTP的不安全问题的

+ 通过CA证书机制，验证了对方的身份
+ 通过约定的加密算法以及三个随机数，对于消息进行加密，从而确保了消息的安全性和无法篡改；另外比较关键的是**解决了对称加密保存和传输的问题，双方都按照约定的规则，基于三个随机数生成的session key，而这个加密key并没有传输，所以中间人即使拿到消息也不知道如何解密**

## 6.1 SSL/TLS Handshake流程

以向目标站点发起支付请求为例，核心流程图如下:

![SSL_Handshake_Process.png](/static/images/charts/2020-10-13/SSL_Handshake_Process.png)

+ --broswer to server--  浏览器进行证书验证并**获取目标站点的公钥**、生成另外一个随机数(pre-master secret)使用目标站点公钥加密
+ ---- 目标站点通过私钥解密得到pre-master secret, 至此双方均有client random、server random、pre-master secret三个随机数
+ --broswer-- 基于约定的加密算法(ciper suite)对于这三者加密生成session key(对称密钥)用于后续的通讯，比如说发送支付密码用于支付
+ --server-- 接收到支付密码，使用同样流程生成session key然后进行解密，进入后续流程返回成功收到支付请求

## 6.2 [调试SSL/TLS Handshake流程](https://stackoverflow.com/questions/17742003/how-to-debug-ssl-handshake-using-curl)

主要为了近一步加强对于上面流程的理解 [openssl 输出结构解释](https://www.cnblogs.com/popsuper1982/p/3843325.html)

```bash
openssl s_client -connect www.baidu.com:443 -prexit
```

```bash
SSL handshake has read 5835 bytes and written 444 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 806581806EFACAD3C2CC3E8211E943ECD16C8BC69B39DCBF6F1A51F934534123
    Session-ID-ctx: 
    Master-Key: EDB21DC142521A05B0FB339D083EF0E47909DAC9C0058F55F6A75542AFB0D0933832B72A31DAC7587F83F3978801EC2C
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (s
```

# 7. 实际案例(与Github交互)

在使用Github的过程中，我们是通过**SSH(Secure Shell)协议**来建立安全的通话渠道的，SSH提供两种级别的安全验证。

## 7.1 基于用户名和密码的

默认情况下，当我们向Repo提交代码的时候，如果第一次通过SSH登录的话，会弹出如下信息:

```bash
ssh user@host

The authenticity of host 'host (22.18.439.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)?
```

上面的意思就是说`22.18.439.21`这个地址的安全性未知，只知道它的指纹(对公钥进行MD5得出一个128位的指纹 -- 16 * 8)。 然后问你是否接受。这种情况你就需要和远程网站公布的指纹进行比对以决定是否信任(一般情况下都是直接接受)，这个地方存在潜在的风险。

```bash
# && ampersands 当第一个成功了才执行第二个
ssh user@host 'mkdir -p .ssh && cat >> ~/.ssh/authorized_keys < ~/.ssh/id_rsa.pub
```

虽然所有传输的数据都会被加密，但是不能保证你正在连接的服务器就是你想连接的服务器。**可能会有别的服务器在冒充真正的服务器，也就是"中间人攻击"**。

如果选择接受，**服务器的公钥(公用密匙)**就会被放入`~/.ssh/known_hosts`中，下次连接的时候服务器检查到自己的公钥已被加入到`known_hosts`中，便不会再进行询问。

基本的过程就是:

+ 用户使用服务器传递过来的公钥**对服务器密码进行加密**，然后传递给服务器
+ 服务器端收到消息之后，使用私钥进行解密然后检查密码是否一致，以决定是否让用户登录

## 7.2 基于密匙的安全验证(借助它可以实现无密码登录)

首先你必须为自己创建一对密匙`ssh-keygen -t rsa`，其中id_rsa为私钥， is_rsa.pub为公钥。并把**公钥放在需要访问的服务器上**，可以通过**ssh-copy-id -i ~/.ssh/id_rsa.pub user@host**实现(Mac的话默认情况下是没有这个工具，使用brew install ssh-copy-id安装即可)。

默认情况下，该命令把`~/.ssh/`下面的`id_rsa.pub`传递给服务器端，把里面的内容追加到远程服务器的`~/.ssh/authorized_keys`上。

之后，当你连接服务器时，就会向服务器发出请求，请求用你的密匙进行安全验证。服务器收到请求之后，先在`authorized_keys`中寻找你的公钥，然后把它和你发送过来的公钥进行比较。

如果一致，服务器就用公钥加密 **质询"(challenge)** 并把它发送给客户端。客户端收到"质询"之后，使用私钥加密，然后发送给服务器端，如果能使用你的公钥解开则成功建立连接。

# 8. Q&A

## 8.1 为什么向[github上无密提交](https://stackoverflow.com/questions/28479567/why-does-github-only-need-my-public-key-in-order-to-push)的代码的时候，需要我们的公钥

身份认证机制的一种，基于公私钥的加解密原则，当我们传递了私钥加密的消息之后，Github如果能使用我们提供的公钥进行解密，则能认证我们的身份(verify you are the one you claim to be)

## 8.2 HTTPs协议最终的传输会基于session key进行加密，那么如果中间人获取session key怎么办

上面已经提到了，**基于三个随机数生成的session key,  而这个加密key并没有传输**。 另外pre-master secret传输过程是采用了对称加密了，所以中间人是拿不到的。

## 8.3 HTTPS(TLS/SSL over http)在通讯过程中同时用到了对称加密和非对称加密，请问它们分别体现在什么过程

+ 非对称加密体现在 验证对方的身份的时候，目标站点会发送自己的证书给到浏览器、APP之类的，颁发的证书是由**CA的私钥进行加密**的，所以浏览器/APP维护的权威的CA公钥如果能解开，说明是授权过的。这就是非对称加解密的体现

+ 对称加密体现上双方约定好三个随机数之后，基于约定的算法计算出密钥，此后双方使用该密钥加解密信息

# 9. 参考

\> Http The Definitive Guide Chapter 14

\> [blog 图解数字签名(English)](http://www.youdzone.com/signature.html)

\> [blog 图解SSL/TLS协议](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)

\> [blog 密码学基本概念](http://www.ruanyifeng.com/blog/2006/12/notes_on_cryptography.html)

\> [blog Https连接的建立过程](http://limboy.me/tech/2011/02/19/https-workflow.html)

\> [blog 关于Session Key的建立](https://www.digicert.com/ssl-cryptography.htm)

\> [百度Https接入安全](http://www.infoq.com/cn/presentations/baidu-https-access-security?utm_campaign=infoq_content&utm_source=infoq&utm_medium=feed&utm_term=presentations)

\> [blog https process 英文版](https://www.cloudflare.com/en-gb/learning/ssl/keyless-ssl/)

\> 极客时间 第15讲 | HTTPS协议：点外卖的过程原来这么复杂

---
layout: post
category: http
date: 2015-11-10 11:51:22 UTC
title: 从前到后看跨域访问
tags: [跨域访问，同源策略，Cookie共享]
permalink: /http/cross-origin/
key: 89b2e67da05a561abc1a12317ba4dd25
description: "本文对浏览器跨域访问进行了研究"
keywords: [跨域访问，同源策略，Cookie共享]
---

当一个项目逐渐演化成SOA之后，必然要涉及到不同服务之间提供服务。出于安全考虑，**浏览器**会限制从脚本中发起的跨域请求。
XMLHttpRequest遵循同源策略(决定了一个域的文档和脚本能否与另外一个域的资源进行交互），也就是默认情况下XMLHttpRequest只能请求自己域名的东西。
**这里要注意一点，服务器还是会正常响应请求，只不过浏览器检测到没有相应的东西会不进行渲染(For security reasons, browsers restrict cross-origin HTTP requests initiated from within scripts)。**


**(1) 怎样才叫同源**

> Two pages have the same origin if the protocol, port (if one is specified), and host are the same for both page

如果两个页面的**协议**，**服务器域名**，**端口**都相同的话，那么他们就是同源的。前面两个区分的标准应该没有什么问题，所以需要着重注意一下端口。

如果一个网站的资源地址是```http://store.company.com/dir/page.html```，那么下面哪些网址指向的资源与它同源: 

<table class="standard-table">
 <tbody>
  <tr>
   <th>URL</th>
   <th>Outcome</th>
   <th>Reason</th>
  </tr>
  <tr>
   <td><code>http://store.company.com/dir2/other.html</code></td>
   <td>同源</td>
   <td>&nbsp;</td>
  </tr>
  <tr>
   <td><code>http://store.company.com/dir/inner/another.html</code></td>
   <td>同源</td>
   <td>&nbsp;</td>
  </tr>
  <tr>
   <td><code>https://store.company.com/secure.html</code></td>
   <td>不同源</td>
   <td>协议不同</td>
  </tr>
  <tr>
   <td><code>http://store.company.com:81/dir/etc.html</code></td>
   <td>不同源</td>
   <td>端口不同</td>
  </tr>
  <tr>
   <td><code>http://news.company.com/dir/other.html</code></td>
   <td>不同源</td>
   <td>域名不同</td>
  </tr>
 </tbody>
</table>

而对于Cookie，如果设置了Domain为**```xxx.com```**, 则该域名和子域名**```yyy.xxx.com```**都可以进行Cookie共享。


**(2) 如何跨域访问**

跨越访问实际上指的就是跨域资源共享(i.e CORS)，
默认情况下，使用XmlHttpRequest请求某网站的数据是禁止的(比如说开发时，在一个域下面向另外一个域发送Ajax请求)。

```js
$.get('http://xxx.com/api/xxx.json', function(data) { console.log("xx") })
```

```console
XMLHttpRequest cannot load http://xxx.com/api/xxx.json. 
No 'Access-Control-Allow-Origin' header is present on the requested resource. 
Origin 'http://localhost:8080' is therefore not allowed access.
```

如果需要进行跨域访问，涉及到三个方面的工作:

**基于Java EE的后端框架的配置**

一般的Java Web框架中，都有一个web.xml(一般是webapp/WEB-INF/web.xml)。它支持我们配置**Filter**元素(关于这部分的操作，相关的帖子有很多，由于不是本文的重点，所以便不再赘述)

**定义CrossOriginFilter**

```scala
class CrossOriginFilter extends Filter {
  override def destroy(): Unit = {}
  override def init(filterConfig: FilterConfig): Unit = {}

  override def doFilter(request: ServletRequest, response: ServletResponse,
    chain: FilterChain): Unit = {
    (request, response) match {
      case (req: HttpServletRequest, resp: HttpServletResponse) =>
          Option(req.getHeader("Origin")).filter(_.trim.nonEmpty).foreach { origin =>
             Option( (new URI(origin)).getHost ).find { host => // host假设是www.google.com
                 // 域名白名单
                val whiteDomainList = "google.com"
                host.endsWith(whiteDomainList)
             }.foreach { trustedOrigin =>
                resp.setHeader("Access-Control-Allow-Origin", origin)
                resp.setHeader("Access-Control-Allow-Credentials", "true") 
                resp.setHeader("Access-Control-Allow-Headers", "Content-Type")
             }
          }
      case _  =>
    }
    // continue the chain flow
    chain.doFilter(request, response)
  }
}
```
**Web.xml**

```xml
<filter>
	<filter-name>CrossOriginFilter</filter-name>
	<filter-class>xxx.CrossOriginFilter</filter-class>
	<async-supported>true</async-supported><!-- 异步支持 -->
</filter>
```

**前端发送请求时添加相应的参数**

关于上面的请求头参数，**```Access-Control-Allow-Headers```**指定发起异步请求的域能自主设置的请求头。**```Access-Control-Allow-Credentials```**指定发起异步请求的域在发送请求的时候能否携带Cookie； <b style="color:red">浏览器在检测到是跨域请求时，会默认添加一个请求头Origin: domain</b>。
   
针对请求头，比如说我们上面设置的是允许添加**```Content-Type```**。如果我们发送Ajax请求的时候，设置了请求头。

```js
// jQuery
$.ajax(
    ...
    xhrFields: {
      // 默认值为false
      withCredentials: true
    },
    "headers": {
      "Content-Type":"text/plain; charset=utf-8"
    },
    ...
)
```

这时候浏览器会首先发送一个**```OPTION```**请求，以确定被请求的域是否允许请求域添加请求头以及请求头的内容是否正确(如果你发送的是普通文本数据，但Content-Type设置成了application/json)。如果出错，则会看到如下信息:

```bash
OPTIONS http://xxxx.com?param=value

XMLHttpRequest cannot load http://xxxx.com?param=value. 
Response for preflight has invalid HTTP status code 404
```

一般在下面几种情况下，浏览器会发送预检表请求(preflight request):

如果发表请求的时候，请求的方法并不是GET, HEAD, POST三者之一; 此外如果发送的是POST请求，但是`Content-type`不属于以下三个:
`x-www-form-urlencoded`, `multipart/form-data`, `text/plain`(当然这个好像与浏览器有关，Mozilla官方文档说的是从Gecko 2.0起，这三个也不会发了，不过以防意外还是要做好发预检表请求的准备);

如果发送请求的时候，设置了自定义的请求头。

针对Allow-Credential，首先我们要明确的一点就是，虽然我们可以通过设置某些属性来发送跨域请求，但是Cookie还是遵从浏览器的同源策略的，也是就同源的资源才能写和读取Cookie。
设置成```withCredentials: true```，是为了让**被请求的域名**设置的Cookie能够在**当前域**发送请求的时候带回给被**请求的域名**；另外就是响应头中必须携带```Access-Control-Allow-Credentials: True```并且```Access-Control-Allow-Credentials```不能被设置成通配符```Access-Control-Allow-Credentials: *```。

    
```js
// A: xx.roadtopro.me
// B: yy.roadtopro.me

    cross-domain
 A ==============> B
```
假设A和B共享Cookie，在B中登录之后会设置相关的Cookie。A跨域请求B的某些资源，如果设置了```withCredentials: false```，则相应的Cookie在发送请求的时候将会被忽略。这样请求B的资源((需要从Cookie中获取某些信息)时就会因信息丢失而失败。
      
    
##参考

\> [Mozilla官方文档中对于跨域的详细介绍](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)

\> [也谈跨域数据交互解决方案](https://imququ.com/post/cross-origin-resource-sharing.html)

\> [跨越表单提交](http://stackoverflow.com/questions/11423682/cross-domain-form-posting)

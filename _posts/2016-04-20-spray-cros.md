---
layout: post
category: spray
date: 2016-04-20 19:51:22 UTC
title: Spray基础之跨域请求处理
tags: [跨域请求，预检表请求]
permalink: /spray/cros/
key: 9d4a67106588420deb413d781e0f804a
description: "本文解决Spray跨域请求的处理"
keywords: [跨域请求，预检表请求]
---


Spray的定位是一个Rest-Http服务器，在实际使用过程中大多数是为别的服务器提供服务的，所以注定有很多地方是通过跨域访问来获取相关资源的。但是Spray好像并没有提供相关的模块来实现这个功能(至少1.3.3版本是这样的)，虽然解决跨域访问的核心是**几个响应头的设置**，但在实际处理过程中还是遇到了不少的麻烦，主要是关于Option请求的响应。

在[从前到后看跨域访问](/http/cross-origin/)一文中提到了在哪些情况下发送跨域请求的时候，会先发送预检表请求。所以我们首先要解决这个问题。很自然的会想到[Spray基础之路由(Directive)](/spray/directive/), 所以我们可以考虑构建自己的路由，基本思路如下:

将此路由置于最外层，发送Option请求的时候，由于一般不会提供Option请求的响应，所以代码中MethodDirective相关的方法都会被拒绝，但同时也意味着这些方法需要在跨域的时候能够访问，所以<b style="color:red">需要将这些方法加入到响应头`Access-Control-Allow-Methods`中去</b>。

如果Option请求已成功响应，告知浏览器哪些请求方法，哪些请求头可以添加。这时浏览器会正式发起请求，所以要在正式请求的响应中也加上相应的请求头。

以下为代码实现:

```scala
trait CrossOriginSupport { this: HttpService =>

  def addedHeaders(ctx: RequestContext): List[HttpHeader] = {
    (for {
    // 跨域请求 浏览器会自动添加Origin请求头
      originStr <- ctx.request.headers.find(_.name == "Origin").map(_.value)
      origin <- Option(HttpOrigin(originStr)) if {
        val originUrlWithoutPort = origin.host.host
        originUrlWithoutPort.endsWith("xxxxx") // 域名白名单
      }
    } yield {
      `Access-Control-Allow-Credentials`(true) ::
      `Access-Control-Allow-Origin`(SomeOrigins(origin :: Nil)) ::
      `Access-Control-Allow-Headers`("Content-Type") :: Nil
    }).getOrElse(Nil)
  }

  def cors = {
    mapRequestContext(ctx => {
      lazy val extraHeaders = addedHeaders(ctx)

      ctx.withRouteResponseHandling({
        case Rejected(rejections) if ctx.request.method.name == OPTIONS.name && 
          rejections.exists(_.isInstanceOf[MethodRejection]) =>

          // 预检表请求的响应中加上能接受哪些方法的响应头
          ctx.complete(
            HttpResponse().withHeaders(
              `Access-Control-Allow-Methods`(
                OPTIONS, (rejections collect { case m: MethodRejection => m.supported }): _*
               ) :: extraHeaders
              )
           )
      }).withHttpResponseHeadersMapped(extraHeaders ::: _) // 正常响应时需要携带的响应头
    })
  }
}
```

上面的代码需要注意**`withHttpResponseHeadersMapped`**的调用，由于**`cors`**相当于重新实现了`mapRequestContext`，在非Option请求中，响应的时候需要有相应的响应头(比如说Spray中默认的)。
然后使用`cors`方法嵌套正常的路由(route)即可:

```scala
cors {
  get {
    path("/") {
        ....
    }
  }
}
```

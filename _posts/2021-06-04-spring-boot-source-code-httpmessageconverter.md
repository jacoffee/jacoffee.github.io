---
layout: post
category: spring
date: 2021-06-04 22:43:03 UTC
title: 【Spring boot源码】HttpMessageConverter的加载和使用
tags: [AutoConfiguration、HttpMessageConvertersAutoConfiguration、MappingJackson2HttpMessageConverter]
permalink: /spring-boot/httpmessageconverter
key:
description: 本文介绍了Spring boot是如何加载和使用HttpMessageConverter的
keywords: [AutoConfiguration、HttpMessageConvertersAutoConfiguration、MappingJackson2HttpMessageConverter]
---

HttpMessageConverter主要是用于Http请求和响应的转换:

+ `请求参数的反序列化`: Controller中定义的接口参数类型是T, 但是前端传递的是json格式数据，需要借助它反序列化 
+ `响应的序列化`: Controller中定义的接口返回类型是T，但是最终请求响应是json格式(`content-type: application/json`)，需要借助它序列化
+ `调用远程请求响应的反序列化`: 远程调用接口返回的响应body是json string类型的，但是我们最终能得到T，需要借助它反序列化

# 1. 扫描阶段

这部分主要是借助Spring boot的自动装配特性来进行MessageConverter注册，具体而言就是`org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration`。从顶部@Import可以看出，默认会尝试加载Jackson、Gson等转换器。

```java
@Import({ JacksonHttpMessageConvertersConfiguration.class, GsonHttpMessageConvertersConfiguration.class,
		JsonbHttpMessageConvertersConfiguration.class })
public class HttpMessageConvertersAutoConfiguration {
    
@Bean
	@ConditionalOnMissingBean
	public HttpMessageConverters messageConverters(ObjectProvider<HttpMessageConverter<?>> converters) {
		return new HttpMessageConverters(converters.orderedStream().collect(Collectors.toList()));
	}
}

@Configuration(proxyBeanMethods = false)
class JacksonHttpMessageConvertersConfiguration {

    @Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(ObjectMapper.class)
	@ConditionalOnBean(ObjectMapper.class)
	@ConditionalOnProperty(name = HttpMessageConvertersAutoConfiguration.PREFERRED_MAPPER_PROPERTY,
			havingValue = "jackson", matchIfMissing = true)
	static class MappingJackson2HttpMessageConverterConfiguration {

		@Bean
		@ConditionalOnMissingBean(value = MappingJackson2HttpMessageConverter.class,
				ignoredType = {
						"org.springframework.hateoas.server.mvc.TypeConstrainedMappingJackson2HttpMessageConverter",
						"org.springframework.data.rest.webmvc.alps.AlpsJsonHttpMessageConverter" })
		MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter(ObjectMapper objectMapper) {
			return new MappingJackson2HttpMessageConverter(objectMapper);
		}

	}
    
    
}
```

在初始化阶段会去扫描`HttpMessageConverter实现类Bean`，如StringHttpMessageConverter、MappingJackson2HttpMessageConverter等。

这里更为关键的是自动装配会暴露很多HttpMessageConverter类型的Bean，那么Spring boot是如何整合它们以及确保调用的顺序的呢？

```java
public class HttpMessageConvertersAutoConfiguration {
 
	@Bean
	@ConditionalOnMissingBean
    // class org.springframework.beans.factory.support.DefaultListableBeanFactory$DependencyObjectProvider
    // 初始化的并没有实际 去处理 HttpMessageConverter， 注意底下的stream
	public HttpMessageConverters messageConverters(ObjectProvider<HttpMessageConverter<?>> converters) {
		return new HttpMessageConverters(converters.orderedStream().collect(Collectors.toList()));
	}
    
}
```

答案就是包装在HttpMessageConverters中的时候，会通过ObjectProvider进行排序，也就是上面的orderedStream对应的逻辑。 如果是PriorityOrdered排在最前面，然后是Ordered Bean，优先级相同任意排列，最后是没有设置优先级的。具体逻辑参考`org.springframework.core.OrderComparator`


# 2. HttpMessageConverter的调用

```java
@RequestMapping(value = "/api", method = RequestMethod.POST)
public BaseResultData<List<JavaModel>> queryModelList(@RequestBody JavaModel request) throws Exception {
        
}
```

```json

{
  "id": "1234",
  "modelSource": "web"
}
```

## 2.1 请求Body的反序列化，典型的如json string to T

针对上面的示例接口以及示例请求Body，在`org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters`方法中，会去按照注册的HttpMessageConverter尝试去反解响应，如果可以，则转换相应的Model，也就是上面的JavaModel。

```java
/**
  源码注释的这个地方也解释的很清楚了，就是基于InputMessage解析成对应类型的 方法参数
* Create the method argument value of the expected parameter type by reading from the given HttpInputMessage.
* @param <T> the expected type of the argument value to be created
* @param inputMessage the HTTP input message representing the current request

*/
@Nullable
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
                                               Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        
    Object body = NO_VALUE;
    
    message = new EmptyBodyCheckingHttpInputMessage(inputMessage);
    
    for (HttpMessageConverter<?> converter : this.messageConverters) {
     
        // 是否可以解析
        if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
            
            if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
			}
            
        }
        
    }
        
}
```

是否可以解析的核心就是，支持的MediaType是否包括**当前请求header中对应的content-type**。


## 2.2 响应的序列化 典型的如T to json string

当要返回响应的时候，需要经历一个反向过程，也就是序列化过程。`org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters`方法中，会去按照注册的HttpMessageConverter尝试去序列化相应的model，也就是上面的JavaModel - `BaseResultData<List<JavaModel>>`。

```java
/**
Writes the given return type to the given output message.
* @param value the value to write to the output message
* @param returnType the type of the value
* @param inputMessage the input messages. Used to inspect the {@code Accept} header.
* @param outputMessage the output message to write to

*/
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
			
			

	for (HttpMessageConverter<?> converter : this.messageConverters) {
	
			if (genericConverter != null ?
						((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {
						
					genericConverter.write(body, targetType, selectedMediaType, outputMessage);			
			}
	}

}
```


## 2.3 远程调用的响应转换

`org.springframework.web.client.HttpMessageConverterExtractor`实现了该功能，底层还是借助注册的HttpMessageConverter实现。

```java
/*
 * Response extractor that uses the given {@linkplain HttpMessageConverter entity converters}
 * to convert the response into a type {@code T}.
 */
public class HttpMessageConverterExtractor<T> implements ResponseExtractor<T> {
    
    public T extractData(ClientHttpResponse response) throws IOException {
        
        for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
        	
            ...
			if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
	             genericMessageConverter.read       
            }    
			...
        }
        
    }
    
}
```

# 3. 参考

\> [SO 自定义HttpMessageConverter](https://stackoverflow.com/questions/67182922/how-to-implement-a-custom-spring-http-message-converter-for-writing-a-typed-coll)




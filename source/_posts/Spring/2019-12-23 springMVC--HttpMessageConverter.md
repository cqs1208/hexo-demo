---
layout: post
title: springMVC--HttpMessageConverter
tags:
- SpringCore
categories: SpringCore
description: SpringCore
---

我们使用**@RequestBody**可以将请求体中的JSON字符串绑定到相应的bean，使用**@ResponseBody**可以使返回结果不会被解析为跳转路径，而是直接写入 HTTP response body 中，而整个数据绑定的过程其实是HttpMessageConverter在起作用

<!-- more --> 

## 1 简介

`org.springframework.http.converter.HttpMessageConverter` 是一个策略接口

![Spring_Mesage_09](/Users/admin/Desktop/note/images/Spring/Spring_Mesage_09.png)

### 接口签名

```java
public interface HttpMessageConverter<T> {

    boolean canRead(Class<?> clazz, MediaType mediaType);
    boolean canWrite(Class<?> clazz, MediaType mediaType);
    List<MediaType> getSupportedMediaTypes();
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException;
    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException;
}
```

| 方法                   | 作用                       |
| ---------------------- | -------------------------- |
| getSupportedMediaTypes | 获取支持的MediaType        |
| read                   | 读取request的body          |
| write                  | 把数据写到response的body中 |

### 缺省配置

我们写 Demo 没有配置任何 MessageConverter，但是数据前后传递依旧好用，是因为 SpringMVC 启动时会自动配置一些HttpMessageConverter，在 WebMvcConfigurationSupport 类中添加了缺省 MessageConverter：

```java
protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
		StringHttpMessageConverter stringConverter = new StringHttpMessageConverter();
		stringConverter.setWriteAcceptCharset(false);

		messageConverters.add(new ByteArrayHttpMessageConverter());
		messageConverters.add(stringConverter);
		messageConverters.add(new ResourceHttpMessageConverter());
		messageConverters.add(new SourceHttpMessageConverter<Source>());
		messageConverters.add(new AllEncompassingFormHttpMessageConverter());

		if (romePresent) {
			messageConverters.add(new AtomFeedHttpMessageConverter());
			messageConverters.add(new RssChannelHttpMessageConverter());
		}

		if (jackson2XmlPresent) {
			ObjectMapper objectMapper = Jackson2ObjectMapperBuilder.xml().applicationContext(this.applicationContext).build();
			messageConverters.add(new MappingJackson2XmlHttpMessageConverter(objectMapper));
		}
		else if (jaxb2Present) {
			messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
		}

		if (jackson2Present) {
			ObjectMapper objectMapper = Jackson2ObjectMapperBuilder.json().applicationContext(this.applicationContext).build();
			messageConverters.add(new MappingJackson2HttpMessageConverter(objectMapper));
		}
		else if (gsonPresent) {
			messageConverters.add(new GsonHttpMessageConverter());
		}
	}
```

### 自定义HttpMessageConverter

编写Converter类，需要实现**HttpMessageConverter**，或者继承已经存在的实现类，并重写上文中的关键方法

编写WebConfig（extends WebMvcConfigurerAdapter）

```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    /**
     * 自定义message_convert
     */
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
       // 把converter添加到converters的最后（SpringBoot会使用第一个匹配到的Converter）
       converters.add(new XxxConverter());
       // 把converter添加到converters的最前面
       // converters.add(0, new XxxConverter());
    }
}
```

自定义**MediaType**

```java
虽然我们已经编写Converter，但是我们一般会为自定义的Converter指定可以处理的媒体类型，可以指定自定义的媒体类型
```

在自定义的Converter中新增自定义的**MediaType**，并且根据需要修改**canRead**，**canWrite**；

```java
public class XxxConverter implements HttpMessageConverter<Serializable> {

    public static final String CUSTOM_MEDIA = "application/custom-media";

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return true;
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return true;
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return Lists.newArrayList(MediaType.parseMediaType(CUSTOM_MEDIA));
    }

    @Override
    public Serializable read(Class<? extends Serializable> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return null;
    }

    @Override
    public void write(Serializable serializable, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {

    }
}
```

这里一定要修改**getSupportedMediaTypes**方法，SpringBoot是根据这个方法的返回，以及Controller---@RequestMapping中指定的**MediaType**，判断是否可用于当前请求/返回

测试：

```java
@RestController
@RequestMapping(produces = CUSTOM_MEDIA, consumes = CUSTOM_MEDIA)
@Validated
public class HomeController {

    @GetMapping(HOME)
    JsonResult info(@RequestHeader("userId") Long userId) {
        return JsonResult.ok();
    }
}
```

### 数据流转解析

数据的请求和响应都要经过 `DispatcherServlet` 类的 `doDispatch(HttpServletRequest request, HttpServletResponse response)` 方法的处理

断点 AbstractMessageConverterMethodArgumentResolver 类的 readWithMessageConverters 方法，查看调用栈：

![Spring_Mesage_10](/Users/admin/Desktop/note/images/Spring/Spring_Mesage_10.png)

readWithMessageConverters方法

```java
@Nullable
    protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        boolean noContentType = false;

        MediaType contentType;
        try {
            contentType = inputMessage.getHeaders().getContentType();
        } catch (InvalidMediaTypeException var16) {
            throw new HttpMediaTypeNotSupportedException(var16.getMessage());
        }

        if (contentType == null) {
            noContentType = true;
            contentType = MediaType.APPLICATION_OCTET_STREAM;
        }

        Class<?> contextClass = parameter.getContainingClass();
        Class<T> targetClass = targetType instanceof Class ? (Class)targetType : null;
        if (targetClass == null) {
            ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
            targetClass = resolvableType.resolve();
        }

        HttpMethod httpMethod = inputMessage instanceof HttpRequest ? ((HttpRequest)inputMessage).getMethod() : null;
        Object body = NO_VALUE;

        AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage message;
        try {
            label98: {
                message = new AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage(inputMessage);
                Iterator var11 = this.messageConverters.iterator();

                HttpMessageConverter converter;
                Class converterType;
                GenericHttpMessageConverter genericConverter;
                while(true) {
                    if (!var11.hasNext()) {
                        break label98;
                    }

                    converter = (HttpMessageConverter)var11.next();
                    converterType = converter.getClass();
                    genericConverter = converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter)converter : null;
                    if (genericConverter != null) {
                        if (genericConverter.canRead(targetType, contextClass, contentType)) {
                            break;
                        }
                    } else if (targetClass != null && converter.canRead(targetClass, contentType)) {
                        break;
                    }
                }

                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
                }

                if (message.hasBody()) {
                    HttpInputMessage msgToUse = this.getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
                    body = genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) : converter.read(targetClass, msgToUse);
                    body = this.getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
                } else {
                    body = this.getAdvice().handleEmptyBody((Object)null, message, parameter, targetType, converterType);
                }
            }
        } catch (IOException var17) {
            throw new HttpMessageNotReadableException("I/O error while reading input message", var17);
        }

        if (body != NO_VALUE) {
            return body;
        } else if (httpMethod != null && SUPPORTED_METHODS.contains(httpMethod) && (!noContentType || message.hasBody())) {
            throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
        } else {
            return null;
        }
    }
```

### HttpMessageConverter 对应

常用 HttpMessageConverter 支持的MediaType 和 JavaType 以及对应关系

| 类名                                    | 支持的javaType  | 支持的MediaType                                        |
| --------------------------------------- | --------------- | ------------------------------------------------------ |
| ByteArrayHttpMessageConverter           | byte[]          | application/octet-stream, */*                          |
| StringHttpMessageConverter              | String          | text/plain, */*                                        |
| MappingJackson2HttpMessageConverter     | Object          | application/json, application/*+json                   |
| AllEncompassingFormHttpMessageConverter | Map<K, List<?>> | application/x-www-form-urlencoded, multipart/form-data |
| SourceHttpMessageConverter              | Source          | application/xml, text/xml, application/*+xml           |



[参考链接](<https://www.jianshu.com/p/09fcd27a8206>)


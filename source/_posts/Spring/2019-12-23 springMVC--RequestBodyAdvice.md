---
layout: post
title: springMVC--RequestBodyAdvice
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

在实际项目中，我们常常需要在请求前后进行一些操作，比如：参数解密/返回结果加密，打印请求参数和返回结果的日志等。这些与业务无关的东西，我们不希望写在controller方法中，造成代码重复可读性变差。这里，我们讲讲使用@ControllerAdvice和RequestBodyAdvice

<!-- more --> 

## 1 示例代码

```shell
@Slf4j
@ControllerAdvice
// # 可加@Order注解确定处理
public class LogRequestBodyAdvice implements RequestBodyAdvice {
	// #是否进行传递
    @Override
    public boolean supports(MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        return true;
    }
 
    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) throws IOException {
        return httpInputMessage;
    }
 
    @Override
    public Object afterBodyRead(Object o, HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        Method method=methodParameter.getMethod();
        log.info("{}.{}:{}",method.getDeclaringClass().getSimpleName(),method.getName(),JSON.toJSONString(o));
        return o;
    }
 
    @Override
    public Object handleEmptyBody(Object o, HttpInputMessage httpInputMessage, MethodParameter methodParameter, Type type, Class<? extends HttpMessageConverter<?>> aClass) {
        Method method=methodParameter.getMethod();
        log.info("{}.{}",method.getDeclaringClass().getSimpleName(),method.getName());
        return o;
    }
}
```

注意：RequestBodyAdvice，针对所有以**@RequestBody**的参数，在读取请求body之前或者在body转换成对象之前可以做相应的增强。这里要加上**@ControllerAdvice**请求才能增强。

## 2 设计思路

**1 从容器中获取所有bean的全类名**，找到标有`@ControllerAdvice`注解的类集合

```java
public static List<ControllerAdviceBean> findAnnotatedBeans(ApplicationContext applicationContext) {
        List<ControllerAdviceBean> beans = new ArrayList();
    // 获取所有bean的全类名
        String[] var2 = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class);
        int var3 = var2.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            String name = var2[var4];
            //找到含有@ControllerAdvice注解的类
            if (applicationContext.findAnnotationOnBean(name, ControllerAdvice.class) != null) {
                // 封装在集合List<ControllerAdviceBean>中
                beans.add(new ControllerAdviceBean(name, applicationContext));
            }
        }

        return beans;
    }
```

**2 定义请求/返回处理链**

```java
class RequestResponseBodyAdviceChain implements RequestBodyAdvice, ResponseBodyAdvice<Object> {
    // 请求参数处理集合
    private final List<Object> requestBodyAdvice = new ArrayList(4);
    // 返回结果处理集合
    private final List<Object> responseBodyAdvice = new ArrayList(4);

    public RequestResponseBodyAdviceChain(@Nullable List<Object> requestResponseBodyAdvice) {
        if (requestResponseBodyAdvice != null) {
            Iterator var2 = requestResponseBodyAdvice.iterator();

            while(var2.hasNext()) {
                Object advice = var2.next();
                Class<?> beanType = advice instanceof ControllerAdviceBean ? ((ControllerAdviceBean)advice).getBeanType() : advice.getClass();
                if (beanType != null) {
                    // 判断是否是 RequestBodyAdvice 的子类型
                    if (RequestBodyAdvice.class.isAssignableFrom(beanType)) {
                        this.requestBodyAdvice.add(advice);
                    } else if (ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
                        this.responseBodyAdvice.add(advice);
                    }
                }
            }
        }

    }
```

## 3 总结

由上可知，前后增强操作需要满足两个条件。1，要有`@ControllerAdvice`注解。 2，要实现`RequestBodyAdvice`接口。这样可通过容器先找出所有bean，再从bean中找出含有注解`@ControllerAdvice`的类，再从中过滤是否实现`RequestBodyAdvice`来确处理链中处理器集合








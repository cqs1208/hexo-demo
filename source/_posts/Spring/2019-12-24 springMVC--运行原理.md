---
layout: post
title: springMVC--运行原理
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

SpringMVC框架是以请求为驱动，围绕Servlet设计，将请求发给控制器，然后通过模型对象，分派器来展示请求结果视图。其中核心类是DispatcherServlet，它是一个Servlet，顶层是实现的Servlet接口。

<!-- more --> 

## 1 xml配置说明

```shell
        <servlet>
		<servlet-name>springmvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
                <!-- 如果不设置init-param标签，则必须在/WEB-INF/下创建xxx-servlet.xml文件，其中xxx是servlet-name中配置的名称。  -->
                <init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/springmvc-servlet.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springmvc</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
```

容器初始化web.xml中配置的servlet，为其初始化自己的上下文信息servletContext，并加载其设置的配置信息到该上下文中。将WebApplicationContext（即spring ioc容器）设置为它的父容器。其中便有SpringMVC（假设配置了SpringMVC），**这也是为什么spring ioc是springmvc ioc的父容器的原因**

## 2  springMVC运行流程

![SpringMVC_01](/Users/admin/Desktop/note/images/Spring/SpringMVC_01.png)

流程说明：

（1）客户端（浏览器）发送请求，直接请求到DispatcherServlet。

（2）DispatcherServlet根据请求信息调用HandlerMapping，解析请求对应的Handler。

（3）解析到对应的Handler后，开始由HandlerAdapter适配器处理。

（4）HandlerAdapter会根据Handler来调用真正的处理器开处理请求，并处理相应的业务逻辑。

（5）处理器处理完业务后，会返回一个ModelAndView对象，Model是返回的数据对象，View是个逻辑上的View。

（6）ViewResolver会根据逻辑View查找实际的View。

（7）DispaterServlet把返回的Model传给View。

（8）通过View返回给请求者（浏览器）

## 3 SpringMVC初始化过程

### init

HttpServlet中的init方法

```java
public final void init() throws ServletException {
    /*加载web.xml文件中的servlet标签中的init-param，其中含有springMVC的配置文件的名字和路径
     *若没有，则默认为（servlet-name）-servlet.xml，
     *默认路径为WEF—INF下
     */
			//解析web.xml中init-param 并封装至pvs中。
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
			//将当前的Servlet类转为Spring可操作的Wrapper类
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			//该处理为空实现，留给子类，使用模板模式
			initBeanWrapper(bw);
			//把init-param中的参数设置到DispatcherServlet里面去
			bw.setPropertyValues(pvs, true);
		}

		// Let subclasses do whatever initialization they like.
		//模板模式留给子类扩展
		initServletBean();
		...
	}
```

DispatcherServlet的初始化过程主要是通过当前的servlet类型实例转换为BeanWrapper类型实例

### initServletBean

该过程由子类完成。在FrameworkServlet中覆盖了HttpServletBean中的initServletBean方法，如下：

```java
 protected final void initServletBean() throws ServletException {
     ...
        try {
            ////创建springmvc的ioc容器实例
            this.webApplicationContext = this.initWebApplicationContext();
            //子类覆盖
            this.initFrameworkServlet();
        } catch (RuntimeException | ServletException var5) {
            this.logger.error("Context initialization failed", var5);
            throw var5;
        }
     ...
    }
```

进入关键方法initWebApplicationContext()

### initWebApplicationContext

initWebApplicationContext方法的主要工作就是创建或者是刷新WebApplicationContext实例并对servlet功能所使用的变量进行初始化。

```java
protected WebApplicationContext initWebApplicationContext() {
    //首先通过ServletContext获得spring容器，因为子容器springMVC要和父容器spring容器进行关联
        //这就是为什么要在ServletContext中注册spring ioc容器的原因
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    //定义springMVC容器wac
		WebApplicationContext wac = null;
//判断容器是否由编程式传入（即是否已经存在了容器实例），存在的话直接赋值给wac，给springMVC容器设置父容器
        //最后调用刷新函数configureAndRefreshWebApplicationContext(wac)，作用是把springMVC的配置信息加载到容器中去（之前已经将配置信息的路径设置到了bw中）
		if (this.webApplicationContext != null) {
			// webApplicationContext 实例在构造函数中被注入
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                        //将spring ioc设置为springMVC ioc的父容器
                        cwac.setParent(rootContext);
                    }
					//刷新上下文环境
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			            // 在ServletContext中寻找是否有springMVC容器，初次运行是没有的，springMVC初始化完毕ServletContext就有了springMVC容器
			wac = findWebApplicationContext();
		}
            //当wac既没有没被编程式注册到容器中的，也没在ServletContext找得到，此时就要新建一个springMVC容器
		if (wac == null) {
			// 创建springMVC容器			
            wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			            //到这里mvc的容器已经创建完毕，接着才是真正调用DispatcherServlet的初始化方法onRefresh(wac)
			onRefresh(wac);
		}

		if (this.publishContext) {
			//将springMVC容器存放到ServletContext中去，方便下次取出来
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}

		return wac;
	}
```

### createWebApplicationContext

FrameworkServlet中的createWebApplicationContext(WebApplicationContext parent)方法

```java
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
        Class<?> contextClass = getContextClass();
       ...
        //实例化空白的ioc容器
        ConfigurableWebApplicationContext wac =
                (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
        //给容器设置环境
        wac.setEnvironment(getEnvironment());
        //给容器设置父容器(就是spring容器)，两个ioc容器关联在一起了
        wac.setParent(parent);
        //给容器加载springMVC的配置信息，之前已经通过bw将配置文件路径写入到了DispatcherServlet中
        wac.setConfigLocation(getContextConfigLocation());
        //上面提到过这方法，刷新容器，根据springMVC配置文件完成初始化操作，此时springMVC容器创建完成
        configureAndRefreshWebApplicationContext(wac);
 
        return wac;
    }
```

### onRefresh

DispatcherServlet的onRefresh(ApplicationContext context)方法

```java
@Override
    protected void onRefresh(ApplicationContext context) {
        initStrategies(context);
    }
```

### initStrategies

DispatcherServlet的initStrategies（ApplicationContext context）方法

```java
protected void initStrategies(ApplicationContext context) {
        initMultipartResolver(context);//文件上传解析
        initLocaleResolver(context);//本地解析
        initThemeResolver(context);//主题解析
        initHandlerMappings(context);//url请求映射
        initHandlerAdapters(context);//初始化真正调用controloler方法的类
        initHandlerExceptionResolvers(context);//异常解析
        initRequestToViewNameTranslator(context);
        initViewResolvers(context);//视图解析
        initFlashMapManager(context);
    }
```

## 4  DispatcherServlet相关类作用概述

| 类及接口          | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| HttpServlet       | 实现了init方法，完成web,xml中与DispatcherServlet有关的参数的读入，初始化DispatcherServlet |
| FrameworkServlet  | 完成了springMVC ioc 容器的创建，并且将spring ioc容器设置为springMVC ioc容器的父容器，将springMVC ioc容器注册到ServletContext中 |
| HandlerMapping    | 根据请求匹配到对应的 `Handler`，然后将找到的 `Handler` 和所有匹配的 `HandlerInterceptor`（拦截器）绑定到创建的 `HandlerExecutionChain` 对象上并返回 |
| DispatcherServlet | 完成策略组件的初始化                                         |

## 5 springMVC 使用姿势

### @RequestMapping注解

在对 SpringMVC 进行的配置的时候, 需要我们指定请求与处理方法之间的映射关系。 指定映射关系，就需要我们用上 `@RequestMapping` 注解。
 `@RequestMapping` 是 Spring Web 应用程序中最常被用到的注解之一，这个注解会将 HTTP 请求映射到控制器（Controller类）的处理方法上。

#### value 和 method 属性

```java
@RequestMapping("rm")
@Controller
public class RequestMappingController {

    @RequestMapping(value = {"home", "/", ""}, method = RequestMethod.GET)
    public String goRMHome() {
        System.out.println("访问了 Test RequestMapping 首页");
        return "1-rm";
    }
}
```

最终访问路径是 `.../rm/home`，通过该方法返回视图名字和SpringMVC视图解析器加工，最终会转发请求到 `.../WEB-INF/jsp/1-rm.jsp` 页面。
 如果没有类名上面的 `@RequestMapping("rm")`，则访问路径为 `.../home`。
 `method` 指定方法请求类型，取值有 GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE。
 `value` 为数组字符串，指定访问路径与方法对应，指定的地址可以是URI。URI值可以是中：普通的具体值、包含某变量、包含正则表达式。

#### consumes 属性

指定处理请求的提交内容类型（Content-Type）

![SpringMVC_05](/Users/admin/Desktop/note/images/Spring/SpringMVC_05.png)

```java
@RequestMapping(value = "testConsumes", method = RequestMethod.POST, consumes = "application/x-www-form-urlencoded")
public String testConsumes() {
    System.out.println("访问了 Test Consumes 方法");
    return "success";
}
```

如果请求里面的 Content-Type 对不上会报错

![SpringMVC_03](/Users/admin/Desktop/note/images/Spring/SpringMVC_03.png)

#### produces 属性

指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回

![SpringMVC_04](/Users/admin/Desktop/note/images/Spring/SpringMVC_04.png)

其中 `*/*;q=0.8` 表明可以接收任何类型的，权重系数0.8表明如果前面几种类型不能正常接收则使用该项进行自动分析。

```java
@RequestMapping(value = "testProduces", method = RequestMethod.POST, produces = "text/html")
public String testProduces() {
    return "success";
}
```

#### params 属性

指定request中必须包含某些参数值，才让该方法处理

```java
@RequestMapping(value = "testParams", method = RequestMethod.POST, params = {"username!=Tom", "password"})
public String testParams() {
    return "success";
}
```

`params = {"username!=Tom", "password"}` 表示请求参数里面 `username !=Tom` 且有包含 `password`，二者有一个不满足则会报错

![SpringMVC_06](/Users/admin/Desktop/note/images/Spring/SpringMVC_06.png)

#### headers 属性

指定 request 中必须包含某些指定的 header 值，才能让该方法处理请求

```java
@RequestMapping(value = "testHeaders", method = RequestMethod.GET, headers = "Accept-Language=zh-CN,zh;q=0.9")
public String testHeaders() {
    return "success";
}
```

如果跟设定头里面对不上会报404错误

![SpringMVC_07](/Users/admin/Desktop/note/images/Spring/SpringMVC_07.png)

### @RequestParam注解

表单元素的name名字和控制器里的方法的形参名一致，此时可以省略

```java
@RequestMapping(value = "testGetOneParam", method = RequestMethod.GET)
public String testGetOneParam(String username) {
    System.out.println("访问了 单参数 Get 请求方法 username: " + username);
    return "success";
}
```

不省略时的写法

```java
@RequestMapping(value = "testPostOneParam", method = RequestMethod.POST)
public String testPostOneParam(@RequestParam String username) {
    System.out.println("username: " + name);
    return "success";
}
```

参数名字不一致时

```jaa
@RequestMapping(value = "testPostOneParam", method = RequestMethod.POST)
public String testPostOneParam(@RequestParam(value = "username", required = false, defaultValue = "") String name) {
    System.out.println("username: " + name);
    return "success";
}
```

`value` 属性指定传过来的参数名，跟方法里的形参名字对应上
 `required` 指定该参数是否是必须携带的
 `defaultValue` 没有或者为 `null` 时，指定默认值

**注：省略和不省略 @RequestParam 注解，最终SpringMVC内部都是使用 RequestParamMethodArgumentResolver 参数解析器进行参数解析的。如果省略 @RequestParam 注解或省略 @RequestParam 注解的 value 属性则最终则以形参的名字作为 key 去 HttpServletRequest 中取值。

### @RequestHeader和@CookieValue 

@RequestHeader 注解：可以把 Request 请求 header 部分的值绑定到方法的参数上

```java
@RequestMapping(value = "rh")
@Controller
public class RequestHeaderController {

    @RequestMapping(value = "testRHAccept", method = RequestMethod.GET)
    public String testRHAccept(@RequestHeader(value = "Accept") String accept) {
        System.out.println(accept);
        return "success";
    }

    @RequestMapping(value = "testRHAcceptEncoding", method = RequestMethod.GET)
    public String testRHAcceptEncoding(@RequestHeader(value = "Accept-Encoding") String acceptEncoding) {
        System.out.println(acceptEncoding);
        return "success";
    }
}
```

@CookieValue 注解：可以把Request header中关于cookie的值绑定到方法的参数上

![SpringMVC_08](/Users/admin/Desktop/note/images/Spring/SpringMVC_08.png)

```java
@RequestMapping(value = "cv")
@Controller
public class CookieValueController {
    @RequestMapping(value = "testGetCookieValue", method = RequestMethod.GET)
    public String testGetCookieValue(@CookieValue(value = "JSESSIONID") String cookie) {
        System.out.println("获取到Cookie里面 JSESSIONID 的值 " + cookie);
        return "success";
    }
}
```








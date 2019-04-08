---
layout: post
title: 04 springboot ApplicationContext
tags:
- Springboot
categories: Springboot
description: springboot
---

容器上下文ApplicationContext 生命周期详解

<!-- more --> 

# Spring Boot中的ApplicationContext 

前面已经对Spring Boot启动过程进行过源码分析，对于代表容器上下文的关键字段ApplicationContext只是一笔带过。实际上，它的生命周期才应该是重点关注的。

## Spring Boot使用的ApplicationContext

分两种场景，常规应用和Web应用使用的上下文类型不一样：

- 常规应用：org.springframework.context.annotation.AnnotationConfigApplicationContext
- Web应用：org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext

从类型层次来看的话：

### AnnotationConfigApplicationContext(由Spring Context定义)

![Annot](/images/SpringBoot/SpringBoot_annotationConfigEmbeddedWebApplicationContext.png)

从类图中可以发现，它们都有共同的父类GenericApplicationContext，在这地方出现了分支。

## 再谈ApplicationContextInitializer

在创建好了ApplicationContext之后，会进行prepare操作：

```
private void prepareContext(ConfigurableApplicationContext context,
            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // Add boot specific singleton beans
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // Load the sources
    Set<Object> sources = getSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[sources.size()]));
    listeners.contextLoaded(context);
}12345678910111213141516171819202122232425
```

这里面比较重要的就是调用注册的ApplicationContextInitializer实现。

```
/**
 * Callback interface for initializing a Spring {@link ConfigurableApplicationContext}
 * prior to being {@linkplain ConfigurableApplicationContext#refresh() refreshed}.
 *
 * <p>Typically used within web applications that require some programmatic initialization
 * of the application context. For example, registering property sources or activating
 * profiles against the {@linkplain ConfigurableApplicationContext#getEnvironment()
 * context's environment}. See {@code ContextLoader} and {@code FrameworkServlet} support
 * for declaring a "contextInitializerClasses" context-param and init-param, respectively.
 *
 * <p>{@code ApplicationContextInitializer} processors are encouraged to detect
 * whether Spring's {@link org.springframework.core.Ordered Ordered} interface has been
 * implemented or if the @{@link org.springframework.core.annotation.Order Order}
 * annotation is present and to sort instances accordingly if so prior to invocation.
 *
 * @author Chris Beams
 * @since 3.1
 * @see org.springframework.web.context.ContextLoader#customizeContext
 * @see org.springframework.web.context.ContextLoader#CONTEXT_INITIALIZER_CLASSES_PARAM
 * @see org.springframework.web.servlet.FrameworkServlet#setContextInitializerClasses
 * @see org.springframework.web.servlet.FrameworkServlet#applyInitializers
 */
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    /**
     * Initialize the given application context.
     * @param applicationContext the application to configure
     */
    void initialize(C applicationContext);

}
```

Javadoc比较长，提炼一下几个关键点：

- 本质上是一个回调接口，用于在ConfigurableApplicationContext执行refresh操作之前对它进行一些初始化操作
- 然后举例说明使用场景，比如Web应用中需要注册属性或者激活Profiles
- 它支持Spring中的Ordered接口以及@Order注解来对多个ApplicationContextInitializer实例进行排序，按照排序后的顺序依次执行回调(这一点后面代码中会看到)

### Spring Boot中定义的6个ApplicationContextInitializer实现类

#### DelegatingApplicationContextInitializer

顾名思义，这个初始化器实际上将初始化的工作委托给context.initializer.classes环境变量指定的初始化器(通过类名)：

```
private static final String PROPERTY_NAME = "context.initializer.classes";

@Override
public void initialize(ConfigurableApplicationContext context) {
    ConfigurableEnvironment environment = context.getEnvironment();
    List<Class<?>> initializerClasses = getInitializerClasses(environment);
    if (!initializerClasses.isEmpty()) {
        applyInitializerClasses(context, initializerClasses);
    }
}

private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
    String classNames = env.getProperty(PROPERTY_NAME);
    List<Class<?>> classes = new ArrayList<Class<?>>();
    if (StringUtils.hasLength(classNames)) {
        for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
            classes.add(getInitializerClass(className));
        }
    }
    return classes;
}
```

这个初始化器的优先级是Spring Boot定义的6个初始化器中优先级别最高的，因此会被第一个执行。

#### ContextIdApplicationContextInitializer

它的作用是给ApplicationContext设置一个ID，这个ID的生成规则是尝试读取以下几个属性：

- spring.application.name
- vcap.application.name
- spring.config.name

如果不存在则使用application作为值。除此之外，还会在上面得到的结果后附加一个port或者index，同样也是读取属性：

- vcap.application.instance_index
- spring.application.index
- server.port
- PORT

所以一个可能的Context ID就是application:10001。

#### ConfigurationWarningsApplicationContextInitializer

这个实现的作用是报告出常见的配置错误。

它的实现方式：

```
@Override
public void initialize(ConfigurableApplicationContext context) {
    context.addBeanFactoryPostProcessor(
            new ConfigurationWarningsPostProcessor(getChecks()));
}12345
```

通过向context上下文对象中添加一个BeanFactoryPostProcessor，然后在refresh ApplicationContext的时候BeanFactoryPostProcessor会被调用到：

![pos](/images/SpringBoot/SpringBoot_postProcessBeanDefinitionRegistry.png)

#### ServerPortInfoApplicationContextInitializer

```
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
    applicationContext.addApplicationListener(
            new ApplicationListener<EmbeddedServletContainerInitializedEvent>() {

                @Override
                public void onApplicationEvent(
                        EmbeddedServletContainerInitializedEvent event) {
                    ServerPortInfoApplicationContextInitializer.this
                            .onApplicationEvent(event);
                }

            });
}1234567891011121314
```

它的作用是监听EmbeddedServletContainerInitializedEvent类型的事件。然后将内嵌的Web服务器使用的端口给设置到ApplicationContext中。

同样地，是在ApplicationContext的refresh阶段，会触发上面的Listener：

![getPost](/images/SpringBoot/SpringBoot_getPost.png)

#### SharedMetadataReaderFactoryContextInitializer

它会创建一个用于在ConfigurationClassPostProcessor和Spring Boot间共享的CachingMetadataReaderFactory。

```
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
    applicationContext.addBeanFactoryPostProcessor(
            new CachingMetadataReaderFactoryPostProcessor());
}
```

同样也是通过增加BeanFactoryPostProcessor来实现。

#### AutoConfigurationReportLoggingInitializer

功能是将ConditionEvaluationReport写入到log，一般的日志的级别是DEBUG，出问题的话使用INFO级别。通过增加ApplicationListener的方式实现。

```
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    applicationContext.addApplicationListener(new AutoConfigurationReportListener());
    if (applicationContext instanceof GenericApplicationContext) {
        // Get the report early in case the context fails to load
        this.report = ConditionEvaluationReport
                .get(this.applicationContext.getBeanFactory());
    }
}

// 响应事件的逻辑
// 只处理：ContextRefreshedEvent以及ApplicationFailedEvent两种类型的事件
protected void onApplicationEvent(ApplicationEvent event) {
    ConfigurableApplicationContext initializerApplicationContext = AutoConfigurationReportLoggingInitializer.this.applicationContext;
    if (event instanceof ContextRefreshedEvent) {
        if (((ApplicationContextEvent) event)
                .getApplicationContext() == initializerApplicationContext) {
            logAutoConfigurationReport();
        }
    }
    else if (event instanceof ApplicationFailedEvent) {
        if (((ApplicationFailedEvent) event)
                .getApplicationContext() == initializerApplicationContext) {
            logAutoConfigurationReport(true);
        }
    }
}
```

### Spring Boot定义的ApplicationContextInitializer如何排定执行顺序

在SpringApplication中有这么一个方法：

```
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
            Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}1234567891011
```

里面有一行代码：`AnnotationAwareOrderComparator.sort(instances);`

这个sort方法会利用AnnotationAwareOrderComparator进行排序。

![Arator](/images/SpringBoot/SpringBoot_annotationAwareOrderComparator.png)

具体的实现可以去看源代码，就不贴在这里了。AnnotationAwareOrderComparator扩展自OrderComparator，从而能够支持@Order注解以及javax.annotation.Priority注解。OrderComparator已经可以支持Ordered接口了。

后者的排序规则如下：

1. 基于Order值升序排序，反应的就是优先级的从高到底
2. 对于拥有相同Order值的对象，任意顺序
3. 对于不能排序的对象(没有实现Ordered接口，没有@Order注解或者@Priority注解)，会排在最后，因为这类对象的优先级是最低的
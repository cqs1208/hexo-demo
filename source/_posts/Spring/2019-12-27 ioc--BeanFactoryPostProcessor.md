---
layout: post
title: ioc--BeanFactoryPostProcessor
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

`BeanFactoryPostProcessor`接口是Spring中一个非常重要的接口，可以对还没有初始化的bean的属性进行修改或添加

<!-- more --> 

## 介绍

接口定义如下

```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

BeanFactoryPostProcessor

与`BeanPostProcessor`的统一注册不同，`BeanFactoryPostProcessor`的注册是留给具体的业务实现的。它的维护是在`AbstractApplicationContext`类中

```
    private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = 
new ArrayList<>();

    public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
        Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
        this.beanFactoryPostProcessors.add(postProcessor);
    }
```

## 执行原理

调用逻辑在`AbstractApplicationContext.invokeBeanFactoryPostProcessors`方法中
这个方法比较长，可以重点关注我添加注释的地方

```
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
    }

public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
 
    Set<String> processedBeans = new HashSet<String>();
 
    // 1.判断beanFactory是否为BeanDefinitionRegistry，在这里普通的beanFactory是DefaultListableBeanFactory,而DefaultListableBeanFactory实现了BeanDefinitionRegistry接口，因此这边为true
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();
 
        // 2.处理入参beanFactoryPostProcessors
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
               // 如果是BeanDefinitionRegistryPostProcessor则直接执行BeanDefinitionRegistryPostProcessor接口的postProcessBeanDefinitionRegistry方法
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            } else {
                regularPostProcessors.add(postProcessor);
            }
        }
 
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
 
        // 3找出所有实现BeanDefinitionRegistryPostProcessor接口的Bean的beanName
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            // 校验是否实现了PriorityOrdered接口
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                //  获取对应的bean实例, 添加到currentRegistryProcessors中,
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序(根据是否实现PriorityOrdered、Ordered接口和order值来排序)
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        // 遍历currentRegistryProcessors, 执行postProcessBeanDefinitionRegistry方法
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        // 清空currentRegistryProcessors
        currentRegistryProcessors.clear();
 
        // 4.与上边3的流程差不多，这是这里处理的是实现Ordered接口
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
 
        // 5.调用所有剩下的BeanDefinitionRegistryPostProcessors
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            // 找出所有实现BeanDefinitionRegistryPostProcessor接口的类
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                // 跳过已经执行过的
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            // 5遍历currentRegistryProcessors, 执行postProcessBeanDefinitionRegistry方法
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }
 
        // 6.调用所有BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        // 7.最后, 调用入参beanFactoryPostProcessors中的普通BeanFactoryPostProcessor的postProcessBeanFactory方法
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    } else {
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }
 
    // 到这里 , 入参beanFactoryPostProcessors和容器中的所有BeanDefinitionRegistryPostProcessor已经全部处理完毕,
    // 下面开始处理容器中的所有BeanFactoryPostProcessor
 
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // 8.找出所有实现BeanFactoryPostProcessor接口的类
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
 
    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    // 用于存放实现了PriorityOrdered接口的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    // 用于存放实现了Ordered接口的BeanFactoryPostProcessor的beanName
    List<String> orderedPostProcessorNames = new ArrayList<String>();
    // 用于存放普通BeanFactoryPostProcessor的beanName
    List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
    // 8.1 遍历postProcessorNames, 将BeanFactoryPostProcessor按实现PriorityOrdered、实现Ordered接口、普通三种区分开
    for (String ppName : postProcessorNames) {
        // 8.2 跳过已经执行过的
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        } else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 8.3 添加实现了PriorityOrdered接口的BeanFactoryPostProcessor
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            // 8.4 添加实现了Ordered接口的BeanFactoryPostProcessor的beanName
            orderedPostProcessorNames.add(ppName);
        } else {
            // 8.5 添加剩下的普通BeanFactoryPostProcessor的beanName
            nonOrderedPostProcessorNames.add(ppName);
        }
    }
 
    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    // 9.调用所有实现PriorityOrdered接口的BeanFactoryPostProcessor
    // 9.1 对priorityOrderedPostProcessors排序
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    // 9.2 遍历priorityOrderedPostProcessors, 执行postProcessBeanFactory方法
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);
 
    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    // 10.调用所有实现Ordered接口的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    for (String postProcessorName : orderedPostProcessorNames) {
        // 10.1 获取postProcessorName对应的bean实例, 添加到orderedPostProcessors, 准备执行
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    // 10.2 对orderedPostProcessors排序
    sortPostProcessors(orderedPostProcessors, beanFactory);
    // 10.3 遍历orderedPostProcessors, 执行postProcessBeanFactory方法
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
 
    // Finally, invoke all other BeanFactoryPostProcessors.
    // 11.调用所有剩下的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        // 11.1 获取postProcessorName对应的bean实例, 添加到nonOrderedPostProcessors, 准备执行
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    // 11.2 遍历nonOrderedPostProcessors, 执行postProcessBeanFactory方法
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
 
    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    // 12.清除元数据缓存（mergedBeanDefinitions、allBeanNamesByType、singletonBeanNamesByType），
    // 因为后处理器可能已经修改了原始元数据，例如， 替换值中的占位符...
    beanFactory.clearMetadataCache();
}
```

## 高阶示例

描述：BenzCar类（奔驰汽车类）有成员属性Engine（发动机）， Engine是接口，无具体的实现类。本代码例子，通过BeanFactoryPostProcessor ，FactoryBean,动态代理三项技术实现给BenzCar装配上Engine。

首先是 SpringBoot的 App类，如下：

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
        try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Bean(initMethod = "start")
    BenzCar benzCar(Engine engine) {
        BenzCar car = new BenzCar();
        car.engine = engine;
        return car;
    }
}
```

从上面代码可以知道 benzCar 依赖 Engine对象，

以下是BenzCar 代码：

```java
public class BenzCar implements InitializingBean {

    public Engine engine;

    public BenzCar(){
        System.out.println("BenzCar Constructor");
        if(engine==null){
            System.out.println("BenzCar's engine not setting");
        }else{
            System.out.println("BenzCar's engine installed");
        }
    }

    void start(){
        System.out.println("BenzCar start");
        engine.fire();
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("BenzCar initializingBean after propertieSet");
        if(engine==null){
            System.out.println("BenzCar's engine not setting, in initializingBean ");
        }else{
            System.out.println("BenzCar's engine installed, in initializingBean");
            engine.fire();
        }
    }

    @PostConstruct
    public void postConstruct(){
        System.out.println("BenzCar postConstruct");
        if(engine==null){
            System.out.println("BenzCar's engine not setting, in postConstruct");
        }else{
            System.out.println("BenzCar's engine installed, in postConstruct");
        }
    }
}
```

BenzCar类中有一个Engine对象成员， 在start方法中调用Engine的fire方法。

Engine接口代码如下：

```java
public interface Engine {
    void fire();
}
```

Engine是一个接口，一般情况下，需要在App类中配置一个Engine的实现类bean才行，否则因为缺少Engine实例，spring启动时会报错。通过FactoryBean和动态代理，可以生成Engine接口的代理对象；结合BeanFactoryPostProcessor 接口，将FactoryBean动态添加到BeanFactory中，即可以给BenzCar配置上Engine接口代理对象。

为此新增一个 SpecialBeanForEngine类， 代码如下：

```java
public class SpecialBeanForEngine implements BeanFactoryPostProcessor, BeanNameAware {

    String name;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        BeanDefinitionRegistry bdr = (BeanDefinitionRegistry) beanFactory;
        GenericBeanDefinition gbd = new GenericBeanDefinition();
        gbd.setBeanClass(EngineFactory.class);
        gbd.setScope(BeanDefinition.SCOPE_SINGLETON);
        gbd.setAutowireCandidate(true);
        bdr.registerBeanDefinition("engine01-gbd", gbd);
    }

    public static class EngineFactory implements FactoryBean<Engine>, BeanNameAware, InvocationHandler {

        String name;

        @Override
        public Engine getObject() throws Exception {
            System.out.println("EngineFactory  to build Engine01 , EngineFactory :" + name);
            Engine prox = (Engine) Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[]{Engine.class}, this);
            return prox;
        }

        @Override
        public Class<?> getObjectType() {
            return Engine.class;
        }

        @Override
        public boolean isSingleton() {
            return true;
        }

        @Override
        public void setBeanName(String name) {
            this.name = name;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("here is invoke  engine:" + method.getName());
            return null;
        }
    }

    @Override
    public void setBeanName(String name) {
        this.name = name;
    }
}
```

在postProcessBeanFactory方法中添加了 EngineFactory.class类的Bean。 EngineFactory 是一个FactoryBean，代码在getObject()方法中，使用动态代理生产Engine接口的代理对象。

在App类中增加SpecialBeanForEngine  Bean， 如下

```java
 @Bean
    SpecialBeanForEngine specialBeanForEngine(){
        return new SpecialBeanForEngine();
    }
```

程序运行结果如下：

```java
EngineFactory  to build Engine01 , EngineFactory :engine01-gbd
BenzCar Constructor
BenzCar's engine not setting
BenzCar postConstruct
BenzCar's engine installed, in postConstruct
BenzCar initializingBean after propertieSet
BenzCar's engine installed, in initializingBean
here is invoke  engine:fire
BenzCar start
here is invoke  engine:fire
```







[参考链接](https://www.cnblogs.com/piepie/p/9061076.html)










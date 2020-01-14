---
layout: post
title: ioc--AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

在分析Spring IOC容器启动流程的时候，在加载Bean定义信息BeanDefinition的时候，用到了两个非常关键的类：AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner。它俩完成对Bean信息的加载。

因此为了更加顺畅的去理解Bean的加载的一个过程，本文主要介绍Spring的这两员大将的一个初始化过程，以及它俩扮演的重要角色

<!-- more --> 

## 加载Bean定义信息分析

`AnnotationConfigApplicationContext`（spring-context包下）的继承图谱如下：

>需要注意的是，我们在Tomcat等web环境下的容器类为：AnnotationConfigWebApplicationContext，它在spring-web包下

![Spring_beanDefinition05](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition05.png)

Spring容器里通过BeanDefinition对象来表示Bean，BeanDefinition描述了Bean的配置信息。
而BeanDefinitionRegistry接口提供了向容器注册，删除，获取BeanDefinition对象的方法。

简单来说，BeanDefinitionRegistry可以用来管理BeanDefinition，所以理解AnnotationConfigApplicationContext很关键，它是spring加载bean，管理bean的最重要的类。

**AnnotationConfigApplicationContext 构造方法**

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		// 把该配置类（们）注册进来
		register(annotatedClasses);
		// 容器启动核心方法
		refresh();
	}
```

this()如下：

```java
//AnnotatedBeanDefinitionReader是一个读取注解的Bean读取器，这里将this传了进去。
	public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
	
	//从上面这个构造函数可以顺便提一句：如果你仅仅是这样ApplicationContext applicationContext = new AnnotationConfigApplicationContext()
	// 容器是不会启动的（也就是不会执行refresh()的），这时候需要自己之后再手动启动容器
```

进而再看看`register()`方法：

```java
public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}

	public void register(Class<?>... annotatedClasses) {
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
	}

	public void registerBean(Class<?> annotatedClass) {
		doRegisterBean(annotatedClass, null, null, null);
	}
```

实际逻辑在`doRegisterBean()`此方法上：

```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
		
		// 先把此实体类型转换为一个BeanDefinition
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		//abd.getMetadata() 元数据包括注解信息、是否内部类、类Class基本信息等等
		// 此处由conditionEvaluator#shouldSkip去过滤，此Class是否是配置类。
		// 大体逻辑为：必须有@Configuration修饰。然后解析一些Condition注解，看是否排除~
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		abd.setInstanceSupplier(instanceSupplier);
		// 解析Scope
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
		// 得到Bean的名称 一般为首字母小写（此处为AnnotationBeanNameGenerator）
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

		// 设定一些注解默认值，如lazy、Primary等等
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		// 解析qualifiers，若有此注解  则primary都成为true了
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
		// 自定义定制信息(一般都不需要)
		for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
			customizer.customize(abd);
		}

		// 下面位解析Scope是否需要代理，最后把这个Bean注册进去
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```

## AnnotatedBeanDefinitionReader

AnnotatedBeanDefinitionReader初始化，构造器如下

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
			
		//ConditionEvaluator完成条件注解的判断，在后面的Spring Boot中有大量的应用
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		//这句会把一些自动注解处理器加入到AnnotationConfigApplicationContext下的BeanFactory的BeanDefinitions中  具体见下面
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

**这里将AnnotationConfigApplicationContext注册为管理BeanDefinition的BeanDefinitionRegistry，也就是说，spring中bean的管理完全交给了AnnotationConfigApplicationContext**

AnnotationConfigUtils#registerAnnotationConfigProcessors()
我们要用一些注解比如：@Autowired/@Required/@Resource都依赖于各种各样的BeanPostProcessor来解析（AutowiredAnnotation、RequiredAnnotation、CommonAnnotationBeanPostProcessor等等）

但是向这种非常常用的，让调用者自己去申明，显然使用起来就过重了。所以Spring为我们提供了一种极为方便注册这些BeanPostProcessor的方式（若是xml方式，配置<context:annotation- config/>，若是全注解驱动的ApplicationContext，就默认会执行）

```java
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
		registerAnnotationConfigProcessors(registry, null);
	}

	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {
		
		// 把我们的beanFactory从registry里解析出来
		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			// 相当于如果没有这个AutowireCandidateResolver，就给设置一份ContextAnnotationAutowireCandidateResolver
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}
		
		//这里初始长度放4  是因为大多数情况下，我们只会注册4个BeanPostProcessor 如下(不多说了)
		// BeanDefinitionHolder解释：持有name和aliases,为注册做准备
		// Spring 4.2之后这个改成6我觉得更准确点
		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(4);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		// 支持JSR-250的一些注解：@Resource、@PostConstruct、@PreDestroy等
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		// 若导入了对JPA的支持，那就注册JPA相关注解的处理器
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
        ...
		return beanDefs;
	}
```

### ConfigurationClassPostProcessor

`ConfigurationClassPostProcessor`是一个`BeanFactoryPostProcessor`和`BeanDefinitionRegistryPostProcessor`处理器，是一个工厂后置处理器`BeanDefinitionRegistryPostProcessor`的处理方法能处理@Configuration等注解。`ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry()`方法内部处理@Configuration，@Import，@ImportResource和类内部的@Bean。

### AutowiredAnnotationBeanPostProcessor

`AutowiredAnnotationBeanPostProcessor`是用来处理`@Autowired`注解和`@Value`注解的，在bean的属性注入的时候会用到

### RequiredAnnotationBeanPostProcessor

RequiredAnnotationBeanPostProcessor这是用来处理@Required注解，在bean的属性注入的时候会用到

### CommonAnnotationBeanPostProcessor

`CommonAnnotationBeanPostProcessor`提供对JSR-250规范注解的支持@javax.annotation.Resource、@javax.annotation.PostConstruct和@javax.annotation.PreDestroy等的支持。

### EventListenerMethodProcessor

`EventListenerMethodProcessor`提供`@PersistenceContext`的支持。

### EventListenerMethodProcessor

`EventListenerMethodProcessor`提供@ EventListener 的支持。@ EventListener实在spring4.2之后出现的，可以在一个Bean的方法上使用@EventListener注解来自动注册一个ApplicationListener。

到此`AnnotatedBeanDefinitionReader`初始化完毕。可以看一下`beanDefinitionMap`中大小

![Spring_beanDefinition06](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition06.png)

总结一下，`AnnotatedBeanDefinitionReade`读取器用来加载class类型的配置，在它初始化的时候，会预先注册一些`BeanPostProcessor`和`BeanFactoryPostProcessor`，这些处理器会在接下来的spring初始化流程中被调用

## ClassPathBeanDefinitionScanner

### 作用

ClassPathBeanDefinitionScanner作用就是将指定包下的类通过一定规则过滤后 将Class 信息包装成 BeanDefinition 的形式注册到IOC容器中。

**根据指定扫描报名 生成匹配规则**

- 例如：classpath*:com.example.demo/**/*.class

**resourcePatternResolver（资源加载器）根据匹配规则 获取 Resource[] **

- Resource数组中每一个 对象 都是对应一个 Class 文件，Spring 用Resource定位资源， 封装了资源的IO操作。

- 这里的 Resource 实际类型是 FileSystemResource.

- 资源加载器 其实就是 容器 本身。

**meteDataFactory根据 Resouce 获取到 MetadataReader 对象**

- MetadataReader 提供了 获取 一个Class 文件的 ClassMetadata 和 AnnotationMetadata 的 操作。

**根据过滤器规则 匹配 MetadataReader中的类 进行过滤，比如 是否是Componet 注解标注的类**

**转换 MetadataReader 为 BeanDefinition**

**将BeanDefinition 注册到 BeanFactory**

### 默认的过滤器注册

过滤器用来过滤 从指定包下面查找到的 Class ,如果能通过过滤器，那么这个class 就会被转换成BeanDefinition 注册到容器。

如果在实例化ClassPathBeanDefinitionScanner时，没有说明要使用用户自定义的过滤器的话，那么就会采用下面的默认的过滤器规则。

注册了`@Component` 过滤器到 includeFiters ,相当于 同时注册了所有被`@Component`注释的注解，包括`@Service` ，`@Repository`,`@Controller`，同时也支持java EE6 的`javax.annotation.ManagedBean` 和 JSR-330 的 `@Named` 注解。

```java
protected void registerDefaultFilters() {
    // 添加Component 注解过滤器
    //这就是为什么 @Service @Controller @Repostory @Component 能够起作用的原因。
        this.includeFilters.add(new AnnotationTypeFilter(Component.class));
        ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
        try {
            // 添加ManagedBean 注解过滤器
            this.includeFilters.add(new AnnotationTypeFilter(
                    ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
            logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
        }
        try {
            // 添加Named 注解过滤器
            this.includeFilters.add(new AnnotationTypeFilter(
                    ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
            logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-330 API not available - simply skip.
        }
    }
```

### 执行扫描（doScan）

实际执行包扫描,进行封装的函数是findCandidateComponents,findCandidateComponents定义在父类中。ClassPathBeanDefinitionScanner的主要功能实现都在这个函数中。

![Spring_beanDefinition07](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition07.png)

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
        try {
            // 1.根据指定包名 生成包搜索路径
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                    resolveBasePackage(basePackage) + '/' + this.resourcePattern;
            //2. 资源加载器 加载搜索路径下的 所有class 转换为 Resource[]
            Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
            // 3. 循环 处理每一个 resource 
            for (Resource resource : resources) {
            
                if (resource.isReadable()) {
                    try {
                        // 读取类的 注解信息 和 类信息 ，信息储存到  MetadataReader
                        // 
                        MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
                     // 执行判断是否符合 过滤器规则，函数内部用过滤器 对metadataReader 过滤  
                        if (isCandidateComponent(metadataReader)) {
                            //把符合条件的 类转换成 BeanDefinition
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            // 再次判断 如果是实体类 返回true,如果是抽象类，但是抽象方法 被 @Lookup 注解注释返回true 
                            if (isCandidateComponent(sbd)) {
                                if (debugEnabled) {
                                    logger.debug("Identified candidate component class: " + resource);
                                }
                                candidates.add(sbd);
                            }
                //省略了 部分代码
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
        }
        return candidates;
    }
```

### 自定义扫描器

通过自定义的扫描器,扫描指定包下所有被@MyBean 注释的类。

#### 定义一个注解，并注释一个类

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyBean {

}

@MyBean
public class TestScannerBean {

}
```

#### 编写扫描器

```java
class MyClassPathDefinitonScanner extends ClassPathBeanDefinitionScanner{
        private Class type;
       public MyClassPathDefinitonScanner(BeanDefinitionRegistry registry,Class<? extends Annotation> type){
            super(registry,false);
            this.type = type;
        }
        /**
         * 注册 过滤器
         */
        public void registerTypeFilter(){
           addIncludeFilter(new AnnotationTypeFilter(type));
        }
    }
```

#### 测试自定义扫描器

```java
@Test
    public void testSimpleScan() {
        String BASE_PACKAGE = "com.example.demo";
        GenericApplicationContext context = new GenericApplicationContext();
        MyClassPathDefinitonScanner myClassPathDefinitonScanner = new MyClassPathDefinitonScanner(context, MyBean.class);
// 注册过滤器
        myClassPathDefinitonScanner.registerTypeFilter();
        int beanCount = myClassPathDefinitonScanner.scan(BASE_PACKAGE);
        context.refresh();
        String[] beanDefinitionNames = context.getBeanDefinitionNames();
        System.out.println(beanCount);
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }
    }

测试结果：
7
//这个就是我们扫描到的bean 
testScannerBean
//下面这些 是 父类扫描器 注册的 beanFactory后置处理器 
org.springframework.context.annotation.internalConfigurationAnnotationProcessor
org.springframework.context.annotation.internalAutowiredAnnotationProcessor
org.springframework.context.annotation.internalRequiredAnnotationProcessor
org.springframework.context.annotation.internalCommonAnnotationProcessor
org.springframework.context.event.internalEventListenerProcessor
org.springframework.context.event.internalEventListenerFactory
```

通过对ClassPathBeanDefinitionScanner的分析,终于揭开了Spring 的类扫描的神秘面纱，其实，就是对指定路径下的 所有class 文件进行逐一排查，对符合条件的 class ,封装成 BeanDefinition注册到IOC 容器。

[参考链接](<https://blog.csdn.net/f641385712/article/details/88059145>)





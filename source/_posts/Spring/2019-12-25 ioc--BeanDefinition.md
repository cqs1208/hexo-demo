---
layout: post
title: ioc--BeanDefinition
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

用过spring的人都知道，我们将对象注入到spring容器中，交给spring来帮我们管理。这种对象我们称之为bean对象。但是这些bean对象在spring容器中，到底是以什么形式存在，具有哪些属性、行为呢？今天我们进入到spring源码来一探究竟

<!-- more --> 

## 1 简述

bean的创建工厂BeanFactory有个默认实现类DefaultListableBeanFactory，内部有个存放所有注入bean对象信息的Map

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

### BeanDefinition基本信息

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    ......
   // 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录的详细介绍
   // 一句话就是：继承父 Bean 的配置信息而已
   void setParentName(String parentName);
   // 获取父 Bean
   String getParentName();
   // 设置 Bean 的类名称，将来是要通过反射来生成实例的
   void setBeanClassName(String beanClassName);
   // 获取 Bean 的类名称
   String getBeanClassName();
   // 设置 bean 的 scope
   void setScope(String scope);
    ......
}
```

**【AttributeAccessor接口】**

类似于map，具有保存和访问name/value属性的能力。

```java
public interface AttributeAccessor {
	//设置类属性
    void setAttribute(String var1, @Nullable Object var2);
    @Nullable
    Object getAttribute(String var1);
    @Nullable
    Object removeAttribute(String var1);
	//是否拥有类属性
    boolean hasAttribute(String var1);
	//获取所有类属性名
    String[] attributeNames();
}
```

**【BeanMetadataElement接口】**

具有访问source（配置源）的能力。

```java
public class BeanDefinitionHolder implements BeanMetadataElement {
    private final BeanDefinition beanDefinition;
    private final String beanName;//beanID
    @Nullable
    private final String[] aliases;//Bean的别名数组
}
```

测试示例

假设有以下两个bean，Java代码如下

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Description;

public class MyTestBean {
    @Autowired
    private AutowireCandidateBean autowireCandidateBean;

    public void init() {
        System.out.println("inti MyTestBean");
    }

    public AutowireCandidateBean getAutowireCandidateBean() {
        return autowireCandidateBean;
    }
    public void setAutowireCandidateBean(AutowireCandidateBean bean) {
        this.autowireCandidateBean = bean;
    }
}
```

```java
public class AutowireCandidateBean {
    public void initBean() {
        System.out.println("init AutowireCandidateBean");
    }

    public void destroyBean() {
        System.out.println("destroy AutowireCandidateBean");
    }
}
```

在配置文件applicationContext.xml来进行配置

```java
<bean id="myTestBean" class="com.yuanweiquan.learn.bean.MyTestBean" 
         depends-on="autowireCandidateBean" 
         init-method="init"/>
         
   <bean id="autowireCandidateBean" 
         class="com.yuanweiquan.learn.bean.AutowireCandidateBean"
         init-method="initBean"
         autowire-candidate="true"
         destroy-method="destroyBean"
         scope="singleton"
         parent="myTestBean"
         lazy-init="default"
         primary="true">
      <description>autowireCandidateBean description</description>
   </bean>
```

测试：

```java
FileSystemXmlApplicationContext factory = 
        new FileSystemXmlApplicationContext("classpath:applicationContext.xml");
BeanDefinition myTestBeanDefinition = 
        factory.getBeanFactory().getBeanDefinition("autowireCandidateBean");
//输出
System.out.println("bean description:" + myTestBeanDefinition.getDescription());
System.out.println("bean class name:" + myTestBeanDefinition.getBeanClassName());
System.out.println("parent name:" + myTestBeanDefinition.getParentName());
System.out.println("scope:" + myTestBeanDefinition.getScope());
System.out.println("is lazyinit:" + myTestBeanDefinition.isLazyInit());
System.out.println("depends On:" + myTestBeanDefinition.getDependsOn());
System.out.println("is autowireCandidate:" + myTestBeanDefinition.isAutowireCandidate());
System.out.println("is primary:" + myTestBeanDefinition.isPrimary());
System.out.println("factory bean name:"+myTestBeanDefinition.getFactoryBeanName());
System.out.println("factory bean method name:" + myTestBeanDefinition.getFactoryMethodName());
System.out.println("init method name:" + myTestBeanDefinition.getInitMethodName());
System.out.println("destory method name:" + myTestBeanDefinition.getDestroyMethodName());
System.out.println("role:" + myTestBeanDefinition.getRole());
//关闭context，否则不会调用bean的销毁方法
factory.close();
```

控制台输出如下

```java
init AutowireCandidateBean
inti MyTestBean
bean description:autowireCandidateBean description
bean class name:com.yuanweiquan.learn.bean.AutowireCandidateBean
parent name:myTestBean
scope:singleton
is lazyinit:false
depends On:null
is autowireCandidate:true
is primary:true
factory bean name:null
factory bean method name:null
init method name:initBean
destory method name:destroyBean
role:0
destroy AutowireCandidateBean
```

通过上面的信息，我们能清晰的看到各属性对应的值。上述测试代码的目的是让我们大家能看到bean在spring容器中，以什么样子的形式存在，具体有哪些属性，属性的值以及默认值是多少

## 2 BeanDefinition相关类

### BeanDefinition子接口和实现类

![Spring_beanDefinition04](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition04.png)

#### AnnotatedBeanDefinition

这个接口可以获取BeanDefinition注解相关数据

```java
public interface AnnotatedBeanDefinition extends BeanDefinition { 
    AnnotationMetadata getMetadata(); 
    MethodMetadata getFactoryMethodMetadata(); 
}
```

#### AbstractBeanDefinition

这个抽象类的构造方法设置了BeanDefinition的默认属性，重写了equals，hashCode，toString方法。

#### ChildBeanDefinition

可以从父BeanDefinition中集成构造方法，属性等

#### RootBeanDefinition

代表一个从配置源（XML，Java Config等）中生成的BeanDefinition

#### GenericBeanDefinition

GenericBeanDefinition是自2.5以后新加入的bean文件配置属性定义类，是ChildBeanDefinition和RootBeanDefinition更好的替代者

#### AnnotatedGenericBeanDefinition

对应注解@Bean

```java
@Component("t")
public class Tester {
    public static void main(String[] args) throws Exception {
        AnnotatedGenericBeanDefinition beanDefinition=new AnnotatedGenericBeanDefinition(Tester.class);
        System.out.println(beanDefinition.getMetadata().getAnnotationTypes());
        System.out.println(beanDefinition.isSingleton());
        System.out.println(beanDefinition.getBeanClassName()); 
    }
}

==========输出==============
[org.springframework.stereotype.Component]
true
Tester
```

### BeanDefinitionRegistry

这个接口定义了‘**注册/获取BeanDefinition**’的方法

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
    //注册一个BeanDefinition  
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException; 
    //根据name,从自己持有的多个BeanDefinition 中 移除一个 
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
    // 获取某个BeanDefinition
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException; 
    boolean containsBeanDefinition(String beanName);//是否包含 
    String[] getBeanDefinitionNames();//获取所有名称 
    int getBeanDefinitionCount();//获取持有的BeanDefinition数量 
    boolean isBeanNameInUse(String beanName); //判断某个BeanDefinition是否在使用
}
```

实现类一般使用Map保存多个BeanDefinition,如下：

```java
Map<String, BeanDefinition> beanDefinitionMap = new HashMap<String, BeanDefinition>();
```

实现类有SimpleBeanDefinitionRegistry，**DefaultListableBeanFactory**，**GenericApplicationContext**等。

#### SimpleBeanDefinitionRegistry

SimpleBeanDefinitionRegistry是最基本的实现类。

```java
public static void main(String[] args) throws Exception {

    //实例化SimpleBeanDefinitionRegistry
    SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();

    //注册两个BeanDefinition
    BeanDefinition definition_1 = new GenericBeanDefinition();
    registry.registerBeanDefinition("d1", definition_1);

    BeanDefinition definition_2 = new RootBeanDefinition();
    registry.registerBeanDefinition("d2", definition_2);

    //方法测试
    System.out.println(registry.containsBeanDefinition("d1"));//true
    System.out.println(registry.getBeanDefinitionCount());//2
    System.out.println(Arrays.toString(registry.getBeanDefinitionNames()));//[d1, d2] 
}

================结果==================
true
2
[d1, d2]
```







### AnnotationConfigApplicationContext

```java
public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
	public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
		super(beanFactory);
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	//直接将注解bean注册到容器,通过将涉及到的配置类传递给该构造函数，以实现将相应配置类中的Bean自动注册到容器中
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
}
```

从源码中可以看出，空参的构造方法会先初始化sacner和reader，为进行bean的解析注册做准备,而容器里不包含任何Bean信息，从注释可以看出，如果调用空参构造，需要手动调用其register()方法注册配置类，并调用refresh()方法刷新容器，触发容器对注解Bean的载入、解析和注册过程。

###  BeanFactory初始化入口

从AnnotationConfigApplicationContext父类中，可以找到beanFactory的初始化。
**GenericApplicationContext#new**

```java
public GenericApplicationContext() {
	this.beanFactory = new DefaultListableBeanFactory();
}
```

## 3 解析注册BeanDefinition

### registerBean

Spring 提供两种bean的注册方式，一种时通过扫描路径来加载，另一种则是通过class文件来注册。先来看下class文件注册。

```java
@Test
public void registerAndRefresh() {
	AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
	context.register(AutowiredConfig.class);
	context.refresh();

	context.getBean("testBean");
	context.getBean("name");
	Map<String, Object> beans = context.getBeansWithAnnotation(Configuration.class);
	assertEquals(2, beans.size());
}
```

![Spring_beanDefinition01](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition01.png)

**AnnotationBeanDefinitionReader#doRegisterBean**

```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
							@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
		//根据指定的注解Bean定义类，创建Spring容器中对注解Bean的封装的数据结构
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		//用来解析注释了@condition注解的类，不满足条件跳过
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}
		//设置创建bean实例的回调
		abd.setInstanceSupplier(instanceSupplier);
		//解析注解Bean定义的作用域
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		//为注解Bean定义设置作用域
		abd.setScope(scopeMetadata.getScopeName());
		//为注解Bean生成Bean名称
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		//处理注解Bean定义中的通用注解
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		//如果在向容器注册注解Bean定义时，使用了额外的限定符注解，则解析限定符注解。
		//主要是配置的关于autowiring自动依赖注入装配的限定条件，即@Qualifier注解，Spring自动依赖注入装配默认是按类型装配，如果使用@Qualifier则按名称
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				//如果配置了@Primary注解，设置该Bean为autowiring自动依赖注入装配时的首选
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				//如果配置了@Lazy注解，则设置该Bean为非延迟初始化，如果没有配置，则该Bean为预实例化
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				//如果使用了除@Primary和@Lazy以外的其他注解，则为该Bean添加一
				//个autowiring自动依赖注入装配限定符，该Bean在进autowiring
				//自动依赖注入装配时，根据名称装配限定符指定的Bean
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
		/**
		 * @see AnnotationConfigApplicationContextTests#individualBeanWithSupplier()
		 */
		for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
			customizer.customize(abd);
		}
		//创建一个指定Bean名称的Bean定义对象，封装注解Bean定义类数据
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		//根据注解Bean定义类中配置的作用域，创建相应的代理对象
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		//向IoC容器注册注解Bean类定义对象
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

可以看出，就是将annotatedClass解析成BeanDefinition，并将bean的属性例如@Lazy等封装进去，接着调用beanFactory的注册:

**DefaultListableBeanFactory#registerBeanDefinition**

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		//从缓存中取出当前bean的 beanDefinition
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		//如果存在
		if (existingDefinition != null) {
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		//如果不存在
		else {
			//检查该bean是否已开始创建
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					//如果单例模式的bean名单中有该bean的name，那么移除掉它。
					//也就是说着，将一个原本是单例模式的bean重新注册成一个普通的bean
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}
		
}
```

BeanDefinition的注册分为步：

(1) 从缓存中取出当前bean的BeanDefinition
(2) 如果存在，则用当前BeanDefinition覆盖原有的
(3) 如果不存在，判断当前bean是否以及开始创建
(4) 如果没有开始创建，则将当前BeanDefinition，以及beanName放入缓存
(5) 如果已经开始创建，将当前BeanDefinition和beanName放入缓存后，如果当前bean是manual singleton bean，则将当前beanName从manual singleton Bean Name中移出，也就是变成了普通的bean,这里的manual singleton Bean指的是以下几种bean：

![Spring_beanDefinition02](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition02.png)

### scanBean

```java
@Test
	public void scanAndRefresh() {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		context.scan("org.springframework.context.annotation6");
		context.refresh();

		context.getBean(uncapitalize(ConfigForScanning.class.getSimpleName()));
		context.getBean("testBean"); // contributed by ConfigForScanning
		context.getBean(uncapitalize(ComponentForScanning.class.getSimpleName()));
		context.getBean(uncapitalize(Jsr330NamedForScanning.class.getSimpleName()));
		Map<String, Object> beans = context.getBeansWithAnnotation(Configuration.class);
		assertEquals(1, beans.size());
	}
```

还是先看context.scan()方法。主要流程如下：

![Spring_beanDefinition03](/Users/admin/Desktop/note/images/Spring/Spring_beanDefinition03.png)

**ClassPathBeanDefinitionScanner#doScan**

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			//扫描给定类路径，获取符合条件的Bean定义
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				//获取Bean定义类中@Scope注解的值，即获取Bean的作用域
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				//生成bean name
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				//如果扫描到的Bean不是Spring的注解Bean，则为Bean设置默认值，
				//设置Bean的自动依赖注入装配属性等
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				//如果扫描到的Bean是Spring的注解Bean，则处理其通用的Spring注解
				if (candidate instanceof AnnotatedBeanDefinition) {
					//处理注解Bean中通用的注解
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//根据Bean名称检查指定的Bean是否需要在容器中注册，或者在容器中冲突
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					//根据注解中配置的作用域，为Bean应用相应的代理模式
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

这个其实和registerBean差不多，只是多了一步从扫描路径里获取符合的bean并封装到beanDefinition中，然后就是registerBean的流程

**ClassPathScanningCandidateComponentProvider#scanCandidateComponents**

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
			//解析给定的包路径，this.resourcePattern=” **/*.class”，
			//ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX=“classpath:”
			//resolveBasePackage方法将包名中的”.”转换为文件系统的”/”
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			//将给定的包路径解析为Spring资源对象
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			for (Resource resource : resources) {
				
				if (resource.isReadable()) {
						//为指定资源获取元数据读取器，元信息读取器通过汇编(ASM)读取资源元信息
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						//如果扫描到的类符合容器配置的过滤规则
						if (isCandidateComponent(metadataReader)) {
							//通过汇编(ASM)读取资源字节码中的Bean定义元信息
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							//设置Bean定义来源于resource
							sbd.setResource(resource);
							//为元数据元素设置配置资源对象
							sbd.setSource(resource);
							//检查Bean是否是一个可实例化的对象
							if (isCandidateComponent(sbd)) {
								
								candidates.add(sbd);
							}
				}
		}
		return candidates;
	}
```

## 4 手动注册beanDefinition

手动注册bean的两种方式：

- 实现ImportBeanDefinitionRegistrar
- 实现BeanDefinitionRegistryPostProcessor

### ImportBeanDefinitionRegistrar

#### 手动创建BeanDefinition

```java
public class Foo {

    public void foo() {
        System.out.println("Foo.foo() invoked!");
    }
}

public class FooImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(Foo.class);
        BeanDefinition beanDefinition = builder.getBeanDefinition();

        registry.registerBeanDefinition("foo",beanDefinition);
    }
}

@SpringBootApplication
@Import({FooImportBeanDefinitionRegistrar.class})
public class Demo implements CommandLineRunner {

    @Autowired
    private Foo foo;

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Demo.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        foo.foo();
    }
}
```

#### ClassPathBeanDefinitionScanner

借助spring类ClassPathBeanDefinitionScanner来扫描Bean并注册

```java
public class Foo {

    public void foo() {
        System.out.println("Foo.foo() invoked!");
    }
}

public class FooImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry);
        scanner.addIncludeFilter(new AssignableTypeFilter(Foo.class));
        scanner.scan("com.study.demo.domain");
        //这里也可以扫描自定义注解并生成BeanDefinition并注册到Spring上下文中
    }
}

@SpringBootApplication
@Import({FooImportBeanDefinitionRegistrar.class})
public class Demo implements CommandLineRunner {

    @Autowired
    private Foo foo;

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Demo.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        foo.foo();
    }
}
```

### BeanDefinitionRegistryPostProcessor

实现BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法。在spring源码中@Configuration @Import等配置类的注解就是通过这种方式实现的，具体实现方式可以参考spring源码

org.springframework.context.annotation.ConfigurationClassPostProcessor

```java
@Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        int registryId = System.identityHashCode(registry);
        if (this.registriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
        }
        if (this.factoriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanFactory already called on this post-processor against " + registry);
        }
        this.registriesPostProcessed.add(registryId);

        processConfigBeanDefinitions(registry);
    }
```

另外一个可参考的样例是Mybatis，在Mybatis中MapperScannerConfigurer实现了BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法，在这个方法中注册bean的方式同1.3

org.mybatis.spring.mapper.MapperScannerConfigurer

```java
@Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```



[参考链接](https://blog.csdn.net/u011179993/article/details/51598567)

[参考链接](https://blog.csdn.net/sinat_29899265/article/details/89022765)






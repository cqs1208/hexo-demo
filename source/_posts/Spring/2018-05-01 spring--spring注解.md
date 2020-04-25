---
layout: post
title: spring--spring注解
tags:
- SpringCore
categories: SpringCore
description: spring
---

Spring中提供了各种Aware接口，比较常见的。。。

<!-- more --> 

## 1 简介

Spring中的注解大概可以分为两大类：

1. spring的bean容器相关的注解，或者说bean工厂相关的注解；
2. springmvc相关的注解。

spring的bean容器相关的注解，先后有：@Required， @Autowired, @PostConstruct, @PreDestory，还有Spring3.0开始支持的JSR-330标准javax.inject.*中的注解(@Inject, @Named, @Qualifier, @Provider, @Scope, @Singleton).``

springmvc相关的注解有：@Controller, @RequestMapping, @RequestParam， @ResponseBody等等。

要理解Spring中的注解，先要理解Java中的注解。

## 2 Java中的注解

### 1 @Override

Java中1.5中开始引入注解，我们最熟悉的应该是：@Override, 它的定义如下：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

	从注释，我们可以看出，@Override的作用是，提示编译器，使用了@Override注解的方法必须override父类或者java.lang.Object中的一个同名方法。我们看到@Override的定义中使用到了 @Target, @Retention，它们就是所谓的“**元注解**”——就是定义注解的注解，

### 2 @Retention

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    /**
     * Returns the retention policy.
     * @return the retention policy
     */
    RetentionPolicy value();
}
```

@Retention用于提示注解被保留多长时间，有三种取值：

```java
public enum RetentionPolicy {
    SOURCE,    //保留在源码级别，被编译器抛弃(@Override就是此类)； 
    CLASS,     //被编译器保留在编译后的类文件级别，但是被虚拟机丢弃
    RUNTIME    //保留至运行时，可以被反射读取。
}
```

### 3 @Target

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    /**
     * Returns an array of the kinds of elements an annotation type
     * can be applied to.
     * @return an array of the kinds of elements an annotation type
     * can be applied to
     */
    ElementType[] value();
}
```

@Target用于提示该注解使用的地方，取值有：

```java
public enum ElementType {
    TYPE,          // 1)类,接口, 注解，enum;
    FIELD,         // 2)属性域；
    METHOD,        // 3）方法；
    PARAMETER,     // 4）参数；
    CONSTRUCTOR,   // 5）构造函数；
    LOCAL_VARIABLE,// 6）局部变量；
    ANNOTATION_TYPE,// 7）注解类型；
    PACKAGE,        // 8）包
    /**
     * Type parameter declaration
     * @since 1.8
     */
    TYPE_PARAMETER, 
    /**
     * Use of a type
     * @since 1.8
     */
    TYPE_USE        
}
```

所以：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

表示 @Override 只能使用在方法上，保留在源码级别，被编译器处理，然后抛弃掉。

### 4 **@Documented**

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Documented {
}
```

表示注解是否能被 javadoc 处理并保留在文档中。

## 3 自定义注解

	有了元注解，那么我就可以使用它来自定义我们需要的注解。结合自定义注解和AOP或者过滤器，是一种十分强大的武器。比如可以使用注解来实现权限的细粒度的控制——在类或者方法上使用权限注解，然后在AOP或者过滤器中进行拦截处理。下面是一个关于登录的权限的注解的实现：

```java
/**
 * 不需要登录注解
 */
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface NoLogin {
}
```

	我们自定义了一个注解 @NoLogin, 可以被用于 方法 和 类 上，注解一直保留到运行期，可以被反射读取到。该注解的含义是：被 @NoLogin 注解的类或者方法，即使用户没有登录，也是可以访问的。下面就是对注解进行处理了：

```java
/**
 * 检查登录拦截器
 * 如不需要检查登录可在方法或者controller上加上@NoLogin
 */
public class CheckLoginInterceptor implements HandlerInterceptor {
    private static final Logger logger = Logger.getLogger(CheckLoginInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) throws Exception {
        if (!(handler instanceof HandlerMethod)) {
            logger.warn("当前操作handler不为HandlerMethod=" + handler.getClass().getName() + ",req="
                        + request.getQueryString());
            return true;
        }
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        String methodName = handlerMethod.getMethod().getName();
        // 判断是否需要检查登录
        NoLogin noLogin = handlerMethod.getMethod().getAnnotation(NoLogin.class);
        if (null != noLogin) {
            if (logger.isDebugEnabled()) {
                logger.debug("当前操作methodName=" + methodName + "不需要检查登录情况");
            }
            return true;
        }
        noLogin = handlerMethod.getMethod().getDeclaringClass().getAnnotation(NoLogin.class);
        if (null != noLogin) {
            if (logger.isDebugEnabled()) {
                logger.debug("当前操作methodName=" + methodName + "不需要检查登录情况");
            }
            return true;
        }
        if (null == request.getSession().getAttribute(CommonConstants.SESSION_KEY_USER)) {
            logger.warn("当前操作" + methodName + "用户未登录,ip=" + request.getRemoteAddr());
            response.getWriter().write(JsonConvertor.convertFailResult(ErrorCodeEnum.NOT_LOGIN).toString()); // 返回错误信息
            return false;
        }
        return true;
    }
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
    }
}
```

上面我们定义了一个登录拦截器，首先**使用反射**来判断方法上是否被 @NoLogin 注解：

NoLogin noLogin = handlerMethod.getMethod().getAnnotation(NoLogin.class);

然后判断类是否被 @NoLogin 注解：

noLogin = handlerMethod.getMethod().getDeclaringClass().getAnnotation(NoLogin.class); 

如果被注解了，就返回 true，如果没有被注解，就判断是否已经登录，没有登录则返回错误信息给前台和false. 这是一个简单的使用 注解 和 过滤器 来进行权限处理的例子。扩展开来，那么我们就可以使用注解，来表示某方法或者类，只能被具有某种角色，或者具有某种权限的用户所访问，然后在过滤器中进行判断处理。

## 4 spring常用注解

### @Configuration @Bean

```java
@Configuration
public class MainConfig {
    @Bean
    public Person person(){
        return new Person(); 
    }
}
```

**注意:** 通过@Bean的形式是使用的话， bean的默认名称是方法名，若@Bean(value="bean的名称")
那么bean的名称是指定的

去容器中读取Bean的信息(传入配置类)

```java
public static void main( String[] args ) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
    System.out.println(ctx.getBean("person")); 
}
```

### @CompentScan

在配置类上写@CompentScan注解来进行包扫描

```java
@Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"}) 
public class MainConfig {
}
```

1:排除用法 excludeFilters(排除@Controller注解的,和TulingService的)

```java
@Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"},excludeFilters = {
@ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class}),
@ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,value = {TulingService.class}) })
public class MainConfig { }
```

2:包含用法 includeFilters ,注意，若使用包含的用法，需要把useDefaultFilters属性设置为false(true表 示扫描全部的) 

```java
 @Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"},includeFilters = {
@ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class, Service.class}) },useDefaultFilters = false)
public class MainConfig {
}
```

### @Scope

配置Bean的作用域对象

1:在不指定@Scope的情况下，所有的bean都是单实例的bean,而且是饿汉加载(容器启动实例就创建 好了) 

```java
@Bean
public Person person() {
	return new Person(); 
}
```

2:指定@Scope为 prototype 表示为多实例的，而且还是懒汉模式加载(IOC容器启动的时候，并不会创建对象，是
在第一次使用的时候才会创建)

```java
@Bean
@Scope(value = "prototype") public Person person() {
	return new Person(); 
}
```

3:@Scope指定的作用域方法取值

a) singleton 单实例的(默认)
b) prototype 多实例的
c) request 同一次请求
d) session 同一个会话级别

### @Lazy

Bean的懒加载@Lazy(主要针对单实例的bean 容器启动的时候，不创建对象，在第一次使用的时候才会创建该象)

```java
@Bean
@Lazy
public Person person() {
	return new Person(); 
}
```

### @Conditional

@Conditional进行条件判断等.

场景,有二个组件TulingAspect 和TulingLog ，我的TulingLog组件是依赖于TulingAspect的组件

应用:自己创建一个TulingCondition的类 实现Condition接口

```java
public class TulingCondition implements Condition {
/**
* @param context
* @param metadata * @return
*/
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    //判断容器中是否有tulingAspect的组件 	
    if(context.getBeanFactory().containsBean("tulingAspect")) {
        return true; 
    }
	return false; 
	}
}


public class MainConfig {
    @Bean
    public TulingAspect tulingAspect() {
    	return new TulingAspect(); 
    }
    
    //当切 容器中有tulingAspect的组件，那么tulingLog才会被实例化. @Bean
    @Conditional(value = TulingCondition.class)
    public TulingLog tulingLog() {
    	return new TulingLog(); 
    }
}
```

### 往IOC 容器中添加组件

1:通过@CompentScan +@Controller @Service @Respository @compent 

​	适用场景: 针对我们自己写的组件可以通过该方式来进行加载到容器中。 

2:通过@Bean的方式来导入组件(适用于导入第三方组件的类) 

3:通过@Import来导入组件 (导入组件的id为全类名路径) 

```java
@Configuration
@Import(value = {Person.class, Car.class}) 
public class MainConfig {
}
```

通过@Import 的ImportSeletor类实现组件的导入 (导入组件的id为全类名路径)

```java
public class TulingImportSelector implements ImportSelector { 
  //可以获取导入类的注解信息
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    	return new String[]{"com.tuling.testimport.compent.Dog"}; 
    }
} 

@Configuration
@Import(value = {Person.class, Car.class, TulingImportSelector.class}) public class 	MainConfig {
}
```

通过@Import的 ImportBeanDefinitionRegister导入组件 (可以指定bean的名称)

```java
public class TulingBeanDefinitionRegister implements ImportBeanDefinitionRegistrar { @Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) { 
    //创建一个bean定义对象
    RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Cat.class);
    //把bean定义对象导入到容器中
    registry.registerBeanDefinition("cat",rootBeanDefinition); 
    }
}

@Configuration
//@Import(value = {Person.class, Car.class})
//@Import(value = {Person.class, Car.class, TulingImportSelector.class})
@Import(value = {Person.class, Car.class, TulingImportSelector.class, TulingBeanDefinitionRegister.class}) public class MainConfig {
}
```

4:通过实现FacotryBean接口来实现注册 组件

```java
public class CarFactoryBean implements FactoryBean<Car> {
    //返回bean的对象
    @Override
    public Car getObject() throws Exception {
        return new Car(); 
    }
    //返回bean的类型
    @Override
    public Class<?> getObjectType() {
        return Car.class; 
    }
    //是否为单例
    @Override
    public boolean isSingleton() {
        return true; 
    }
}
```

### Bean的初始化和销毁

bean的创建----->初始化----->销毁方法
由容器管理Bean的生命周期，我们可以通过自己指定bean的初始化方法和bean的销毁方法

```java
@Configuration
public class MainConfig {
    //指定了bean的生命周期的初始化方法和销毁方法.
    @Bean(initMethod = "init",destroyMethod = "destroy") 
    public Car car() {
		return new Car(); 
    }
}
```

针对单实例bean的话，容器启动的时候，bean的对象就创建了，而且容器销毁的时候，也会调用Bean的销毁方法 

针对多实例bean的话,容器启动的时候，bean是不会被创建的而是在获取bean的时候被创建，而且bean的销毁不受 IOC容器的管理. 

2:通过 InitializingBean和DisposableBean 的二个接口实现bean的初始化以及销毁方法

```java
@Component
public class Person implements InitializingBean,DisposableBean {
    public Person() { 
        System.out.println("Person的构造方法");
    }
    @Override
    public void destroy() throws Exception {
    	System.out.println("DisposableBean的destroy()方法 "); 
    }
    @Override
    public void afterPropertiesSet() throws Exception {
    	System.out.println("InitializingBean的 afterPropertiesSet方法"); 
    }
}
```

3:通过JSR250规范 提供的注解@PostConstruct 和@ProDestory标注的方法

```java
@Component 
public class Book {
    public Book() {
    	System.out.println("book 的构造方法");
    }
    @PostConstruct 
    public void init() {
    	System.out.println("book 的PostConstruct标志的方法"); 
    }
    @PreDestroy
    public void destory() {
    	System.out.println("book 的PreDestory标注的方法"); 
    }
}
```

4:通过Spring的BeanPostProcessor的 bean的后置处理器会拦截所有bean创建过程

postProcessBeforeInitialization 在init方法之前调用
postProcessAfterInitialization 在init方法之后调用

```java
@Component
public class TulingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {  			System.out.println("postProcessBeforeInitialization:"+beanName);
    	return bean; 
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
 		System.out.println("postProcessAfterInitialization:"+beanName);
    	return bean; 
    }
}
```

BeanPostProcessor的执行时机

```java
populateBean(beanName, mbd, instanceWrapper) 
initializeBean{
	applyBeanPostProcessorsBeforeInitialization() invokeInitMethods{
	isInitializingBean.afterPropertiesSet
	自定义的init方法 
    }
applyBeanPostProcessorsAfterInitialization()方法 
}
```

### @Value @PropertySource

注入普通字符

```java
@Value("Michael Jackson")
String name;
```

注入操作系统属性

```java
@Value("#{systemProperties['os.name']}")
String osName;
```

注入表达式结果

```java
@Value("#{ T(java.lang.Math).random() * 100}")
String randomNumber;
```

注入其它bean属性

```java
@Value("#{domeClass.name}")
String name;
```

注入文件资源

```java
@Value("#{classpath:com/hgs/hello/test.txt}")
String Resoutce file;
```

注入网站资源

```java
@Value("http://www.cznovel.com")
Resource url;
```

注入配置文件

```java
@Value("${book.name}")
Resource url;
```

注入配置使用方法：

```java
@PropertySource 加载配置文件(类上)
@PropertySource("classpath:com/hgs/hello/test.properties")
```

使用示例：

```java
public class Person {
//通过普通的方式 @Value("司马")
private String firstName;
    //spel方式来赋值 
    @Value("#{28-8}")
    private Integer age; 通过读取外部配置文件的值 
    @Value("${person.lastName}") 
    private String lastName;
}

```

```java
@Configuration
@PropertySource(value = {"classpath:person.properties"}) 
    //指定外部文件的位置 public class MainConfig {
    @Bean
    public Person person() {
   	 	return new Person(); 
    }
}
```

### @Profile

通过@Profile注解 来根据环境来激活标识不同的Bean

@Profile标识在类上，那么只有当前环境匹配，整个配置类才会生效
@Profile标识在Bean上 ，那么只有当前环境的Bean才会被激活,没有标志为@Profile的bean 不管在什么环境都可以被激活

```java
@Configuration
@PropertySource(value = {"classpath:ds.properties"})
public class MainConfig implements EmbeddedValueResolverAware {
	@Value("${ds.username}") 
    private String userName;
	@Value("${ds.password}") 
    private String password;
	private String jdbcUrl; 
    private String classDriver;
    
    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
   		this.jdbcUrl = resolver.resolveStringValue("${ds.jdbcUrl}");
    	this.classDriver = resolver.resolveStringValue("${ds.classDriver}"); 
    }
    //标识为测试环境才会被装配 @Bean
    @Profile(value = "test") public DataSource testDs() {
    	return buliderDataSource(new DruidDataSource()); 
     }
    //标识开发环境才会被激活 @Bean
    @Profile(value = "dev") public DataSource devDs() {
    	return buliderDataSource(new DruidDataSource()); 
    }
    
    //标识生产环境才会被激活 @Bean
    @Profile(value = "prod") 
    public DataSource prodDs() {
    	return buliderDataSource(new DruidDataSource()); 
    }
    private DataSource buliderDataSource(DruidDataSource dataSource) {
        dataSource.setUsername(userName); 
        dataSource.setPassword(password);
        dataSource.setDriverClassName(classDriver); 
        dataSource.setUrl(jdbcUrl);
    	return dataSource; 
    }
}
```

激活切换环境的方法

方法一:通过运行时jvm参数来切换 -Dspring.profiles.active=test|dev|prod
方法二:通过代码的方式来激活

```java
 public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(); 		ctx.getEnvironment().setActiveProfiles("test","dev");
	ctx.register(MainConfig.class); ctx.refresh(); 
    printBeanName(ctx);
}
```

### 4 切面AOP相关注解

@Aspect 声明一个切面（类上）

使用@After、@Before、@Around定义建言（advice），可直接将拦截规则（切点）作为参数。

@After 在方法执行之后执行（方法上）

@Before 在方法执行之前执行（方法上）

@Around 在方法执行之前与之后执行（方法上）

@PointCut 声明切点

在java配置类中使用@EnableAspectJAutoProxy注解开启Spring对AspectJ代理的支持（类上）

###  @Enable*注解说明

这些注解主要用来开启对xxx的支持。

@EnableAspectJAutoProxy 开启对AspectJ自动代理的支持

@EnableAsync 开启异步方法的支持

@EnableScheduling 开启计划任务的支持

@EnableWebMvc 开启Web MVC的配置支持

@EnableConfigurationProperties 开启对@ConfigurationProperties注解配置Bean的支持

@EnableJpaRepositories 开启对SpringData JPA Repository的支持

@EnableTransactionManagement 开启注解式事务的支持

@EnableTransactionManagement 开启注解式事务的支持

@EnableCaching 开启注解式的缓存支持

### 11 测试相关注解

@RunWith 运行器，Spring中通常用于对JUnit的支持

```java
@RunWith(SpringJUnit4ClassRunner.class)
```

@ContextConfiguration 用来加载配置ApplicationContext，其中classes属性用来加载配置类

```java
@ContextConfiguration(classes={TestConfig.class})
```

## 5 SpringMVC常用注解

@EnableWebMvc 在配置类中开启Web MVC的配置支持，如一些ViewResolver或者MessageConverter等，若无此句，重写WebMvcConfigurerAdapter方法（用于对SpringMVC的配置）。

@Controller 声明该类为SpringMVC中的Controller

@RequestMapping 用于映射Web请求，包括访问路径和参数（类或方法上）

@ResponseBody 支持将返回值放在response内，而不是一个页面，通常用户返回json数据（返回值旁或方法上）

@RequestBody 允许request的参数在request体中，而不是在直接连接在地址后面。（放在参数前）

@PathVariable 用于接收路径参数，比如@RequestMapping(“/hello/{name}”)申明的路径，将注解放在参数中前，即可获取该值，通常作为Restful的接口实现方法。

@RestController 该注解为一个组合注解，相当于@Controller和@ResponseBody的组合，注解在类上，意味着，该Controller的所有方法都默认加上了@ResponseBody。

@ControllerAdvice 通过该注解，我们可以将对于控制器的全局配置放置在同一个位置，注解了@Controller的类的方法可使用@ExceptionHandler、@InitBinder、@ModelAttribute注解到方法上，

这对所有注解了 @RequestMapping的控制器内的方法有效。

@ExceptionHandler 用于全局处理控制器里的异常

@InitBinder 用来设置WebDataBinder，WebDataBinder用来自动绑定前台请求参数到Model中。

@ModelAttribute 本来的作用是绑定键值对到Model里，在@ControllerAdvice中是让全局的@RequestMapping都能获得在此处设置的键值对。



[参考链接](<https://www.cnblogs.com/digdeep/p/4525567.html>)
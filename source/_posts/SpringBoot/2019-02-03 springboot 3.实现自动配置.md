---
layout: post
title: 03 spring 实现自动配置
tags:
- Springboot
categories: Springboot
description: springboot
---

Spring中纷繁复杂的配置文件一直让广大开发人员颇有微词，但是随着业务的复杂度越来越高，这也是在所难免的事情。

<!-- more --> 

# 1.Spring Boot实现自动配置的基础

## 引子

### 问题是什么

Spring中纷繁复杂的配置文件一直让广大开发人员颇有微词，但是随着业务的复杂度越来越高，这也是在所难免的事情。

哪怕是要创建一个简单的Web应用，需要配置的东西也是一坨坨的，这个过程虽然不复杂但是很繁琐，而且非常容易出错。所以聪明的开发人员想出了很多办法来解决这个问题，比如Maven的Archetype创建，又或者各个公司内部的脚手架小工具。但是这些方案总是有这样那样的问题，比如维护不方便，不好定制等等。

在云计算，弹性计算以及微服务越来越普及的今天，急需要一种自动配置的方式，从而方便地按需部署。所以Spring Boot应用而生，而自动配置作为Spring Boot的闪光点之一，自然非常受人关注。

Spring Boot的自动配置功能，其实从本质上说并没有引入什么新功能，它只是将Spring现存的能力做了一次组合和封装。那么在深入了解Spring Boot的自动配置原理之前，可以先了解一下Spring的这些已知能力，打下良好地基础。

## 和配置相关的Spring已有能力

### @Profile

很多时候，我们开发的Spring应用，需要根据所在环境的不同注册相应的Bean到上下文中。

比如本地开发环境中，数据库连接对象往往指向的是开发数据库；在测试环境中，又会指向测试数据库；而到了线上，指向的自然是生产数据库。

为了满足这个常见需求，Spring 3.1中引入了Profile的概念。比如在下面的代码中，配置类会根据所在环境(Profile)的不同，向上下文中注入对应的Bean实例。

```java
@Configuration
public class DataSourceConfiguration
{
    @Bean
    @Profile("DEV")
    public DataSource devDataSource() {
        // ...
    }

    @Bean
    @Profile("TEST")
    public DataSource testDataSource() {
        // ...
    }

    @Bean
    @Profile("PROD")
    public DataSource prodDataSource() {
        // ...
    }
}
```

那么，如何声明应用所处的Profile呢？还是有几种选择：

1. 配置文件，比如在application.properties中声明`spring.profiles.active=DEV`
2. 启动参数，比如`-Dspring.profiles.active=DEV`

这个Profile的概念很直观，但是由于它仅仅是依赖一个字符串的值作出决策，所以不够灵活和强大。因此就有了下面@Conditional注解和Condition接口的诞生。

### @Conditional以及Condition接口

它们是Spring 4中引入的新功能。

Condition接口和@Conditional接口通常会一起配合使用。

Condition接口的定义如下：

```java
public interface Condition {

    /**
     * 决定Condition是否被满足
     * @param context 上下文信息
     * @param metadata 类或方法上的元数据注解信息
     */
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

下面是一个该接口的实现类：

```java
public class CustomCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 通过一系列计算决定Condition是否被满足，如果满足返回true，反之返回false
        return true;
    }
}
```

@Conditional注解的定义如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

    /**
     * 该注解接受的默认参数类型是一个Class数组，这些Class都应该实现Condition接口，并且当其中的matches都返回true的时候，对应的Bean才会被注入
     */
    Class<? extends Condition>[] value();

}
```

然后一般在Java Config类中将它们结合在一起使用：

```java
@Configuration
public class CustomConfiguration {
    // 当CustomCondition中的matches返回true的时候才会注入CustomClass这个类的实例Bean。
    @Bean
    @Conditional(CustomCondition.class)
    public CustomClass customClass() {
        return new CustomClass();
    }
}
```

因此通过将@Conditional注解和Condition接口的能力进行结合，就可以实现根据外界条件来决定一个Bean是不是应该被加载到Spring上下文中。对比一下@Profile注解，可以发现这种方式提供了更高的灵活度。

在@Profile注解中，唯一决定是否加载一个Bean的是一个字符串，即所在环境的标识符例如DEV，TEST，PRODUCTION这类。但是在@Conditional的方案中，决定是否加载的逻辑被抽象到了Condition接口中，这样能够做的判断就非常多了。

### @ConfigurationProperties以及@EnableConfigurationProperties

要实现自动配置，还有一个比较关键的环节。我们都知道Bean之所以有意义是因为每个Bean中都有自己独特的字段，而很多情况下，Bean中的字段都是可配置的。因此，要实现自动配置，我们首先需要对一些关键字段给予一个默认值，当然也需要提供一个机制让用户能够对这些字段进行覆盖。在Spring Boot中，提供了一个新的机制叫做Configuration Properties(可配置的属性)，它有几个特点：

1. 类型安全的配置方式(基于Java类)，基于Spring Conversion API完成类型转换
2. 可以通过配置文件，启动参数等多种方式完成覆盖
3. 和JavaConfig类无缝集成

这个机制有两个关键的注解类型，即@ConfigurationProperties和@EnableConfigurationProperties。下面我们就来看看它们的定义以及如何应用：

#### 定义方式

**@ConfigurationProperties**

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {

    @AliasFor("prefix")
    String value() default "";

    @AliasFor("value")
    String prefix() default "";

    // ...
}
```

只捡主要的属性介绍，这个注解中比较关键的就是value/prefix属性。这两个属性是一个意思，互相设置了@AliasFor。

**@EnableConfigurationProperties**

```java
/**
 * 用于支持@ConfigurationProperties标注的Beans。
 * @ConfigurationProperties注解的Bean也可以通过标准方式(比如使用@Bean方法)，但是这个注解提供了一个更方便的方法。
 *
 * @author Dave Syer
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesImportSelector.class)
public @interface EnableConfigurationProperties {

    Class<?>[] value() default {};

}
```

#### 应用方式

以Spring Web应用中需要的HttpEncodingFilter这个Filter的自动配置为例，看看如何应用这两注解：

```java
@ConfigurationProperties(prefix = "spring.http.encoding")
public class HttpEncodingProperties {

    public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

    private Charset charset = DEFAULT_CHARSET;

    private Boolean force;

    // ...

}
```

这是一个配置类，它设置了前缀spring.http.encoding。因此，这个配置类中的字段都会对应某一个具体的配置KEY，比如charset字段对应的配置KEY为spring.http.encoding.charse; force字段对应的是spring.http.encoding.force。同时可以给每个字段设置一个默认值，比如上述的charset字段就有一个默认值DEFAULT_CHARSET，对应的是UTF-8的字符集。

如果我们不想用默认的UTF-8编码方式，想换成GBK，那么在配置文件(如application.properties中)，可以进行如下覆盖：

```java
spring.http.encoding.charset=GBK1
```

这里有个地方值得一提，在配置文件中的GBK明明是字符串类型的值，而在对应的配置属性类这个属性是Charset类型的，那么肯定是有一个步骤完成了从字符串到Charset类型的转换工作。完成这个步骤的就是Spring框架中的Conversion API。这里只贴一张相关的图，不进行深入分析，感兴趣的同学可以在配置属性类的setCharset方法上打个断点然后深挖一下。![genericConverter](../images/SpringBoot/SpringBoot_genericConverter.png)

从这张图可以看出，Spring Conversion API中维护了一系列Converters，它实际上是一个从源类型到目标类型的Map对象。从String类型到Charset类型的Converter也被注册到了这个Map对象中。因此，当发现配置文件中的值类型和配置Java类中的字段类型不匹配时，就会去尝试从这个Map中找到相应的Converter，然后进行转换。

### 自定义注解实现自动配置

下面我们尝试来创建一个自定义的注解，实现Bean的按需加载。假设我们要创建的注解名为@DatabaseType。

具体的需求是：当启动参数中的dbType=MYSQL的时候，创建一个数据源为MySQL的UserDAO对象；当启动参数中的dbType=ORACLE的时候，创建一个数据源为Oracle的UserDAO对象。

最终配置类的代码可以是这样的：

```java
@Configuration
public class DatabaseConfiguration
{
  @Bean
  @DatabaseType("MYSQL")
  public UserDAO mysqlUserDAO() {
      return new MySQLUserDAO();
  }

  @Bean
  @DatabaseType("ORACLE")
  public UserDAO oracleUserDAO() {
      return new OracleUserDAO();
  }
}
```

可以知道，@DatabaseType这个注解能够接受一个字符串作为参数。然后将该参数和启动参数中的dbType值进行比对，如果相等的话，就会进行对应Bean的注入工作。@DatabaseType的实现如下：

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Conditional(DatabaseTypeCondition.class)
public @interface DatabaseType {
    String value();
}
```

其中比较重要的是`@Conditional(DatabaseTypeCondition.class)`，表明它的觉得实际上也是委托给了DatabaseTypeCondition这个类：

```java
public class DatabaseTypeCondition implements Condition {
    // 这里也展示了AnnotatedTypeMetadata这个参数的用法
    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
        Map<String, Object> attributes = metadata.getAnnotationAttributes(DatabaseType.class.getName());
        String type = (String) attributes.get("value");
        String enabledDBType = System.getProperty("dbType", "MYSQL");
        return (enabledDBType != null && type != null && enabledDBType.equalsIgnoreCase(type));
    }
}
```

通过matches方法的第二个参数AnnotatedTypeMetadata，可以得到指定注解类型的所有属性，本例中就是@DatabaseType这个注解。然后将注解中的value值和启动参数dbType(不存在时使用MYSQL作为默认值)进行比对，然后返回相应的值来决定是否需要注入Bean。

所以，通过这个例子我们也可以发现。通常而言为了程序的可读性，可以将@Conditional和Condition接口的实现类给封装到一个业务含义更加明确的注解类型中，比如上面的@DatabaseType类型。它的意义就很明确，当类型是MySQL的该如何如何，当类型为Oracle的时候又当如何如何。

在后面深究Spring Boot的自动配置机制时，可以发现它也在@Conditional和Condition接口的基础上，定义了很多相关类型，用于更好地定义自动配置行为。

------

## 总结

本文介绍了Spring中和自动配置相关的两个概念：

1. @Profile
2. @Conditional以及Condition接口



# 2.Spring Boot实现自动配置的原理

## 入口注解类@EnableAutoConfiguration

@SpringBootApplication注解中包含了自动配置的入口注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
  // ...
}
```

```java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
  // ...
}
```

这个注解的Javadoc内容还是不少，所有就不贴在文章里面了，概括一下：

1. 自动配置基于应用的类路径以及你定义了什么Beans
2. 如果使用了@SpringBootApplication注解，那么自动就启用了自动配置
3. 可以通过设置注解的excludeName属性或者通过spring.autoconfigure.exclude配置项来指定不需要自动配置的项目
4. 自动配置的发生时机在用户定义的Beans被注册之后
5. 如果没有和@SpringBootApplication一同使用，最好将@EnableAutoConfiguration注解放在root package的类上，这样就能够搜索到所有子packages中的类了
6. 自动配置类就是普通的Spring @Configuration类，通过SpringFactoriesLoader机制完成加载，实现上通常使用@Conditional(比如@ConditionalOnClass或者@ConditionalOnMissingBean)

### @AutoConfigurationPackage

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

这个注解的职责就是引入了另外一个配置类：AutoConfigurationPackages.Registrar。

```java
/**
 * ImportBeanDefinitionRegistrar用来从导入的Config中保存base package
 */
@Order(Ordered.HIGHEST_PRECEDENCE)
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.<Object>singleton(new PackageImport(metadata));
    }

}
```

这个注解实现的功能已经比较底层了，调试看看上面的register方法什么会被调用![AutoConfigurationPackages](/images/SpringBoot/SpringBoot_autoConfigurationPackages.png)

调用参数中的packageNames数组中仅包含一个值：com.example.demo，也就是项目的root package名。

从调用栈来看的话，调用register方法的时间在容器刷新期间：

refresh -> invokeBeanFactoryPostProcessors -> invokeBeanDefinitionRegistryPostProcessors -> postProcessBeanDefinitionRegistry -> processConfigBeanDefinitions(开始处理配置Bean的定义) -> loadBeanDefinitions -> loadBeanDefinitionsForConfigurationClass(读取配置Class中的Bean定义) -> loadBeanDefinitionsFromRegistrars(这里开始准备进入上面的register方法) -> registerBeanDefinitions(即上述方法)

这个过程已经比较复杂了，目前暂且不深入研究了。它的功能简单说就是将应用的root package给注册到Spring容器中，供后续使用。

相比而言，下面要讨论的几个类型才是实现自动配置的关键。

### @Import(EnableAutoConfigurationImportSelector.class)

@EnableAutoConfiguration注解的另外一个作用就是引入了EnableAutoConfigurationImportSelector：

它的类图如下所示：

 ![En](/images/SpringBoot/SpringBoot_enableAutoConfigurationImportSelector.png)

可以发现它除了实现几个Aware类接口外，最关键的就是实现了DeferredImportSelector(继承自ImportSelector)接口。

所以我们先来看看ImportSelector以及DeferredImportSelector接口的定义：

```java
public interface ImportSelector {

    /**
     * 基于被引入的Configuration类的AnnotationMetadata信息选择并返回需要引入的类名列表
     */
    String[] selectImports(AnnotationMetadata importingClassMetadata);

}
```

这个接口的Javadoc比较长，还是捡重点说明一下：

1. 主要功能通过selectImports方法实现，用于筛选需要引入的类名
2. 实现了ImportSelector的类也可以实现一系列Aware接口，这些Aware接口中的相应方法会在selectImports方法之前被调用(这一点通过上面的类图也可以佐证，EnableAutoConfigurationImportSelector确实实现了四个Aware类型的接口)
3. ImportSelector的实现和通常的@Import在处理方式上是一致的，然而还是可以在所有@Configuration类都被处理后再进行引入筛选(具体看下面即将介绍的DeferredImportSelector)

```java
public interface DeferredImportSelector extends ImportSelector {

}
```

这个接口是一个标记接口，它本身没有定义任何方法。那么这个接口的含义是什么呢：

1. 它是ImportSelector接口的一个变体，在所有的@Configuration被处理之后才会执行。在需要筛选的引入类型具备@Conditional注解的时候非常有用
2. 实现类同样也可以实现Ordered接口，来定义多个DeferredImportSelector的优先级别(同样地，EnableAutoConfigurationImportSelector也实现了Ordered接口)

明确了这两个接口的意义，下面来看看是如何实现的：

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    try {
      // Step1: 得到注解信息
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        // Step2: 得到注解中的所有属性信息
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // Step3: 得到候选配置列表
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        // Step4: 去重
        configurations = removeDuplicates(configurations);
        // Step5: 排序
        configurations = sort(configurations, autoConfigurationMetadata);
        // Step6: 根据注解中的exclude信息去除不需要的
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        // Step7: 派发事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return configurations.toArray(new String[configurations.size()]);
    }
    catch (IOException ex) {
        throw new IllegalStateException(ex);
    }
}
```

很明显，核心就在于上面的步骤3：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
        AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

它将实现委托给了SpringFactoriesLoader的loadFactoryNames方法：

```java
// 传入的factoryClass：org.springframework.boot.autoconfigure.EnableAutoConfiguration
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    try {
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
                "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}

// 相关常量
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
```

这段代码的意图很明确，在第一篇文章讨论Spring Boot启动过程的时候就已经接触到了。它会从类路径中拿到所有名为META-INF/spring.factories的配置文件，然后按照factoryClass的名称取到对应的值。那么我们就来找一个META-INF/spring.factories配置文件看看。

#### META-INF/spring.factories

比如spring-boot-autoconfigure包：

```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
# 省略了很多12345678910
```

列举了非常多的自动配置候选项，挑一个AOP相关的AopAutoConfiguration看看究竟：

```java
// 如果设置了spring.aop.auto=false，那么AOP不会被配置
// 需要检测到@EnableAspectJAutoProxy注解存在才会生效
// 默认使用JdkDynamicAutoProxyConfiguration，如果设置了spring.aop.proxy-target-class=true，那么使用CglibAutoProxyConfiguration
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = false)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = true)
    public static class JdkDynamicAutoProxyConfiguration {

    }

    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = true)
    @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = false)
    public static class CglibAutoProxyConfiguration {

    }

}
```

这个自动配置类的作用是判断是否存在配置项：

```
spring.aop.proxy-target-class=true1
```

如果存在并且值为true的话使用基于CGLIB字节码操作的动态代理方案，否则使用JDK自带的动态代理机制。

在这个配置类中，使用到了两个全新的注解：

- @ConditionalOnClass
- @ConditionalOnProperty

从这两个注解的名称，就大概能够猜出它们的功能了：

**@ConditionalOnClass**

当类路径上存在指定的类时，满足条件。

**@ConditionalOnProperty**

当配置中存在指定的属性时，满足条件。

其实除了这两个注解之外，还有几个类似的，它们都在org.springframework.boot.autoconfigure.condition这个包下，在具体介绍实现之前，下面先来看看Spring Boot对于@Conditional的扩展。=

## Spring Boot对于@Conditional的扩展

Spring Boot提供了一个实现了Condition接口的抽象类SpringBootCondition。

这个类的主要作用是打印一些用于诊断的日志，告诉用户哪些类型被自动配置了。

它实现Condition接口的方法：

```java
@Override
public final boolean matches(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
    String classOrMethodName = getClassOrMethodName(metadata);
    try {
        ConditionOutcome outcome = getMatchOutcome(context, metadata);
        logOutcome(classOrMethodName, outcome);
        recordEvaluation(context, classOrMethodName, outcome);
        return outcome.isMatch();
    }
    catch (NoClassDefFoundError ex) {
        throw new IllegalStateException(
                "Could not evaluate condition on " + classOrMethodName + " due to "
                        + ex.getMessage() + " not "
                        + "found. Make sure your own configuration does not rely on "
                        + "that class. This can also happen if you are "
                        + "@ComponentScanning a springframework package (e.g. if you "
                        + "put a @ComponentScan in the default package by mistake)",
                ex);
    }
    catch (RuntimeException ex) {
        throw new IllegalStateException(
                "Error processing condition on " + getName(metadata), ex);
    }
}

/**
 * Determine the outcome of the match along with suitable log output.
 * @param context the condition context
 * @param metadata the annotation metadata
 * @return the condition outcome
 */
public abstract ConditionOutcome getMatchOutcome(ConditionContext context,
        AnnotatedTypeMetadata metadata);
```

SpringBootCondition已经提供了基本的实现，将内部的匹配细节定义成抽象方法getMatchOutcome，交给其子类去完成。

另外，还提供了两个可能会被子类使用到的方法：

```java
/**
 * 如果指定的conditions中有任意一个匹配，那么就返回true
 * @param context the context
 * @param metadata the annotation meta-data
 * @param conditions conditions to test
 * @return {@code true} if any condition matches.
 */
protected final boolean anyMatches(ConditionContext context,
        AnnotatedTypeMetadata metadata, Condition... conditions) {
    for (Condition condition : conditions) {
        if (matches(context, metadata, condition)) {
            return true;
        }
    }
    return false;
}

/**
 * 检查指定的condition是否匹配
 * @param context the context
 * @param metadata the annotation meta-data
 * @param condition condition to test
 * @return {@code true} if the condition matches.
 */
protected final boolean matches(ConditionContext context,
        AnnotatedTypeMetadata metadata, Condition condition) {
    if (condition instanceof SpringBootCondition) {
        return ((SpringBootCondition) condition).getMatchOutcome(context, metadata)
                .isMatch();
    }
    return condition.matches(context, metadata);
}
```

### org.springframework.boot.autoconfigure.condition包

除了上面已经遇到的@ConditionalOnClass和@ConditionalOnProperty，这个包中还定义了很多条件实现类，下面简单列举几个：

#### @ConditionalOnExpression - 基于SpEL的条件判断

```java
/**
 * Configuration annotation for a conditional element that depends on the value of a SpEL
 * expression.
 *
 * @author Dave Syer
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnExpressionCondition.class)
public @interface ConditionalOnExpression {

    /**
     * The SpEL expression to evaluate. Expression should return {@code true} if the
     * condition passes or {@code false} if it fails.
     * @return the SpEL expression
     */
    String value() default "true";123456789101112131415161718
```

然后相应的实现类是OnExpressionCondition，它继承自SpringBootCondition。

#### @ConditionalOnMissingClass - 基于类不存在与classpath的条件判断

这一个条件实现正好和@ConditionalOnClass条件相反。

------

下面列举所有由Spring Boot提供的条件注解：

- @ConditionalOnBean
- @ConditionalOnClass
- @ConditionalOnCloudPlatform
- @ConditionalOnExpression
- @ConditionalOnJava
- @ConditionalOnJndi
- @ConditionalOnMissingBean
- @ConditionalOnMissingClass
- @ConditionalOnNotWebApplication
- @ConditionalOnProperty
- @ConditionalOnResource
- @ConditionalOnSingleCandidate
- @ConditionalOnWebApplication

一般的模式，就是一个条件注解对应一个继承自SpringBootCondition的具体实现类。
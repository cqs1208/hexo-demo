---
layout: post
title: aop--springAop详解
tags:
- SpringCore
categories: SpringCore
description: spring
---

AOP是Spring Core中几大重要能力之一，我们可以使用AOP实现很多功能，比如我们常用的日志处理与Spring中的声明式事务。

<!-- more --> 

## springAop基本使用

### 概念说明

#### 切面（Aspect）

官方的抽象定义为“一个关注点的模块化，这个关注点可能会横切多个对象”。“切面”在ApplicationContext 中<aop:aspect>来配置。连接点（Joinpoint） ：程序执行过程中的某一行为，例如，MemberService.get 的调用或者MemberService.delete 抛出异常等行为。

#### 通知（Advice）

“切面”对于某个“连接点”所产生的动作。其中，一个“切面”可以包含多个“Advice”。

#### 切入点（Pointcut）

匹配连接点的断言，在 AOP 中通知和一个切入点表达式关联。切面中的所有通知所关注的连接点，都由切入点表达式来决定。

#### 目标对象（Target Object）

被一个或者多个切面所通知的对象。当然在实际运行时，SpringAOP 采用代理实现，实际 AOP 操作的是 TargetObject 的代理对象。

#### AOP 代理（AOP Proxy）

在 Spring AOP 中有两种代理方式，JDK 动态代理和 CGLib 代理。默认情况下，TargetObject 实现了接口时，则采用 JDK 动态代理，反之，采用 CGLib 代理。强制使用 CGLib 代理需要将 <aop:config>的 proxy-target-class 属性设为 true。

### 通知类型：

#### Before Advice

在某连接点（JoinPoint）之前执行的通知，但这个通知不能阻止连接点前的执行。ApplicationContext中在<aop:aspect>里面使用<aop:before>元素进行声明。

#### After Advice

当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。ApplicationContext 中在<aop:aspect>里面使用<aop:after>元素进行声明。

#### After Return Advice

在某连接点正常完成后执行的通知，不包括抛出异常的情况。ApplicationContext 中在<aop:aspect>里面使用<after-returning>元素进行声明。

#### Around Advice

包围一个连接点的通知，类似 Web 中 Servlet 规范中的 Filter 的 doFilter 方法。可以在方法的调用前后完成自定义的行为，也可以选择不执行。ApplicationContext 中在<aop:aspect>里面使用<aop:around>元素进行声明。例如，ServiceAspect 中的 around 方法。

#### After Throwing Advice

在 方 法 抛 出 异 常 退 出 时 执 行 的 通 知 。 ApplicationContext 中 在 <aop:aspect> 里 面 使 用<aop:after-throwing>元素进行声明。

### 使用示例

```java
// 配置扫描包 并开启aop
@ComponentScan("com.dependency")
@EnableAspectJAutoProxy
public class AppConfig {

}

// 目标对象(类)
@Component
public class MathService {
    public int division(int x, int y){
        System.out.println("*******执行方法********");
        return x / y;
    }
}

// 目标对象(接口)
public interface TestService {
     void sayHello();
}

// 接口实现类
@Component
public class TestServiceImpl implements TestService {
    @Override
    public void sayHello() {
        System.out.println("*******执行方法********");
    }
}

// aop拦截配置
@Component
@Aspect
public class MyAspects {

    @Pointcut("execution(* com.dependency.service.TestService.*(..))")
    public void jdkPointCut(){}

    @Pointcut("execution(* com.dependency.service.MathService.*(..))")
    public void cglibPointCut(){}

    //前置通知
    @Before("jdkPointCut() || cglibPointCut()")
    public void logBefote(){
        System.out.println("前置通知Before");
    }
    //后置通知
    @After("jdkPointCut() || cglibPointCut()")
    public void logAfter(){
        System.out.println("后置通知After");
    }
    //异常通知
    @AfterThrowing("jdkPointCut() || cglibPointCut()")
    public void logAfterThrowing(){
        System.out.println("异常通知AfterThrowing");
    }
    //返回通知
    @AfterReturning("jdkPointCut() || cglibPointCut()")
    public void logAfterReturning(){
        System.out.println("返回通知AfterReturning");
    }
}

// 
public class TestDemo {
    public static void main(String[] args) throws Throwable {

        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();

        context.register(AppConfig.class);
        context.refresh();
        TestService testService = context.getBean(TestService.class);
        MathService mathService = context.getBean(MathService.class);

        System.out.println("接口代理（testService）： " + testService.getClass().getName());
        testService.sayHello();

        System.out.println("===============================================");
        System.out.println("类代理（mathService）： " + mathService.getClass().getName());
        mathService.division(10 , 2);
    }
}

// 返回结果
接口代理（testService）： com.sun.proxy.$Proxy19
前置通知Before
*******执行方法********
后置通知After
返回通知AfterReturning
===============================================
类代理（mathService）： com.dependency.service.MathService$$EnhancerBySpringCGLIB$$e604edf9
前置通知Before
*******执行方法********
后置通知After
返回通知AfterReturning
```

## 手动aop设计示例

springAop使用的是动态代理和责任链设计模式，本示例使用的是cglib代理

### 扫描类

```java
@ComponentScan("com.dependency")
@EnableAspectJAutoProxy
public class AppConfig {

}
```

### 实体bean(目标类)

```java
public class UserService {
    public Object insert(String args){
        System.out.println("==插入用户成功==");
        return "user insert ok";
    }
}
```

### 代理工厂

```java
public class ProxyFactory2 implements MethodInterceptor {

    private Object target;
    List<BaseAdvice> baseAdviceList;

    public ProxyFactory2 () {
        baseAdviceList = new ArrayList<>();
        //异常
        baseAdviceList.add(new ThrowAdvice());
        //返回
        baseAdviceList.add(new ReturnAdvice());
        //后置
        baseAdviceList.add(new AfterAdvice());
        //前置
        baseAdviceList.add(new BeforeAdvice());
    }

    public UserService getProxy (Object target ){
        this.target = target;
        //CGLIB生成代理对象
        Enhancer enhancer = new Enhancer();
        //设置父类
        enhancer.setSuperclass(target.getClass());
        //设置回调
        enhancer.setCallback(this);
        //设置类加载器
        enhancer.setClassLoader(target.getClass().getClassLoader());
        //生成代理对象
        return (UserService)enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        Chain chain = new Chain(baseAdviceList, target, method, objects);
        return chain.proceed();
    }
}
```

### 通知责任链

```java
public class Chain {

    private List<BaseAdvice> adviceList;

    //记录一下index下标
    private int index = -1;

    //目标方法
    private Method method;
    //方法参数
    private Object[] args;
    //目标对象
    private Object target;

    public Chain(List<BaseAdvice> adviceList, Object o, Method method, Object[] objects) {
        this.adviceList = adviceList;
        this.target = o;
        this.method = method;
        this.args = objects;
    }

    public Object proceed() throws Throwable{
        //依次调用集合中的所有通知
        if(index == adviceList.size() - 1){
            //执行我们的目标方法
            return method.invoke(target, args);
        }
       return adviceList.get(++index).execute(this);
    }
}
```

### 基础通知接口

```java
public abstract class BaseAdvice {
    public abstract Object execute(Chain chain) throws Throwable;
}
```

### 各种通知实现

```java
// 前置通知
public class BeforeAdvice extends BaseAdvice {
    @Override
    public Object execute(Chain chain) throws Throwable {
        System.out.println("前置通知========1");
        return chain.proceed();
    }
}

// 后置通知
public class AfterAdvice extends BaseAdvice {
    @Override
    public Object execute(Chain chain) throws Throwable {
        try{
            return chain.proceed();
        }finally {
            System.out.println("后置通知");
        }
    }
}

// 返回通知
public class ReturnAdvice extends BaseAdvice {
    @Override
    public Object execute(Chain chain) throws Throwable {
        Object value = chain.proceed();
        //如果没有异常执行返回通知
        System.out.println("返回通知");
        return value;
    }
}

// 异常通知
public class ThrowAdvice extends BaseAdvice {
    @Override
    public Object execute(Chain chain) throws Throwable {
        try{
            return chain.proceed();
        }catch (Throwable e){
            System.out.println("异常通知");
            throw e;
        }
    }
}
```

### 测试

```java
public class TestDemo {
    public static void main(String[] args) throws Throwable {

        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(AppConfig.class);
        context.refresh();

        System.out.println("================================");
        UserService userService = new UserService();
        UserService userService1 = new ProxyFactory2().getProxy(userService);
        userService1.insert("canshu");
    }
}
```

## springAop分析

### 基础对象介绍

**AopProxyFactory**

AopProxy代理工厂类，用于生成代理对象AopProxy。

```java
public interface AopProxyFactory {
    AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
```

**AopProxy**

代表一个AopProxy代理对象，可以通过这个对象构造代理对象实例。

```java
public interface AopProxy {
    Object getProxy();
    Object getProxy(ClassLoader classLoader);
}
```

**Advised**

代表被Advice增强的对象，包括添加advisor的方法、添加advice等的方法。

**ProxyConfig**

一个代理对象的配置信息，包括代理的各种属性，如基于接口还是基于类构造代理

**AdvisedSupport**

对Advised的构建提供支持，Advised的实现类以及ProxyConfig的子类。

**ProxyCreatorSupport**

AdvisedSupport的子类，创建代理对象的支持类，内部包含AopProxyFactory工厂成员，可直接使用工厂成员创建Proxy。

**ProxyFactory**

ProxyCreatorSupport的子类，用于生成代理对象实例的工厂类

**Advisor**

代表一个增强器提供者的对象，内部包含getAdvice方法获取增强器。

**AdvisorChainFactory**

获取增强器链的工厂接口。提供方法返回所有增强器，以数组返回

**Pointcut**

切入点，用于匹配类与方法，满足切入点的条件是才插入advice。相关接口：ClassFilter、MethodMatcher。

### 入口

从获取代理对象的实例入口ProxyFactory.getProxy()开始：

```java
public Object getProxy() {
    // 创建AopProxy对象再获取代理对象实例
	return createAopProxy().getProxy();
}
// createAopProxy方法在父类ProxyCreatorSupport中
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	// 先获取AopProxy的工厂对象，再把自己作为createAopProxy的参数AdvisedSupport传进去，用自己作为代理对象的配置
	return getAopProxyFactory().createAopProxy(this);
}
```

代理对象实例最终是使用AopProxy.getProxy()得到的，他的调用是在AopProxyFactory.createAopProxy(AdvisedSupport config，createAopProxy有两个结果。一个是基于接口的JDK动态代理JdkDynamicAopProxy，一个是基于CGLib的生成类代理ObjenesisCglibAopProxy。源码如下：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		// 如果是需要优化的代理，或者标记代理目标类，或者代理配置中没有需要代理的接口
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			    // 如果目标类是接口，或者已经是Jdk的动态代理类，则创建jdk动态代理
				return new JdkDynamicAopProxy(config);
			}
			// 否则创建Cglib动态代理
			return new ObjenesisCglibAopProxy(config);
		}
		else {
		    // 如果声明创建Jdk动态代理则返回Jdk动态代理
		    return new JdkDynamicAopProxy(config);
		}
	}
}
```

传入的AdvisedSupport config中包含了需要注册的Method拦截器，AopProxy会保存这个config为advised对象。

### JDK的动态代理

JdkDynamicAopProxy中getProxy会返回：

```java
public Object getProxy(ClassLoader classLoader) {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
	}
	// 获取所有需要代理的接口
	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	// 返回代理对象的实例
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

自己作为InvocationHandler注册，看他的invoke方法

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class<?> targetClass = null;
	Object target = null;

	try {
		...
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// 如果调用的方法是DecoratingProxy中的方法，因为其中只有一个getDecoratedClass方法，这里直接返回被装饰的Class即可
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// 代理不是不透明的，且是接口中声明的方法，且是Advised或其父接口的方法，则直接调用构造时传入的advised对象的相应方法
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
		    // 如果暴露代理，则用AopContext保存当前代理对象。用于多级代理时获取当前的代理对象，一个有效应用是同类中调用方法，代理拦截器会无效。可以使用AopContext.currentProxy()获得代理对象并调用。
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}

		// 这里是关键，获得拦截链chain，是通过advised对象，即config对象获得的。
		method.
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		if (chain.isEmpty()) {
		    // 如果链是空，则直接调用被代理对象的方法
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// 否则创建一个MethodInvocation对象，用于链式调用拦截器链chain中的拦截器。
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// 开始执行链式调用，得到返回结果
			retVal = invocation.proceed();
		}

		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// 处理返回值
			// 如果返回结果是this，即原始对象，且方法所在类没有标记为RawTargetAccess(不是RawTargetAccess的实现类或者子接口)，则返回代理对象。
			retVal = proxy;
		}
        ...
		}
		return retVal;
	}
}
```

注册的Method拦截器都是通过AdvisedSupport这个config对象的addAdvice或者addAdvisor注册进去的。

```java
public void addAdvice(int pos, Advice advice) throws AopConfigException {
	Assert.notNull(advice, "Advice must not be null");
	if (advice instanceof IntroductionInfo) {
		// 如果是引介，则加入引介advisor。(新增功能)
		addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
	}
	else if (advice instanceof DynamicIntroductionAdvice) {
		// jdk动态代理不支持动态引介
		throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
	}
	else {
	    // 把advice转换为advisor并添加，目标是DefaultPointcutAdvisor。
		addAdvisor(pos, new DefaultPointcutAdvisor(advice));
	}
}
```

其实也是把advice转成了advisor注册的。 看下最上面invoke方法中有一个方法调用：

List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
	    // 其实是通过advisorChainFactory工厂对象获得的
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```

是通过AdvisorChainFactory的getInterceptorsAndDynamicInterceptionAdvice方法获取的，也把config对象传入了，且加的有缓存。其实是通过method获取该method对应的advisor。下面是他的唯一实现：

```java
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {

		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

		for (Advisor advisor : config.getAdvisors()) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}

	private static boolean hasMatchingIntroductions(Advised config, Class<?> actualClass) {
		for (int i = 0; i < config.getAdvisors().length; i++) {
			Advisor advisor = config.getAdvisors()[i];
			if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (ia.getClassFilter().matches(actualClass)) {
					return true;
				}
			}
		}
		return false;
	}

}
```

上面包括了各种对Advisor包装，通过Pointcut等的判断把Advisor中的Advice包装成MethodInterceptor、InterceptorAndDynamicMethodMatcher或者Interceptor。

之后在调用方法前，又把chain转换为了aopalliance体系的的MethodInvocation。

invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);

最终执行的是retVal = invocation.proceed()。 在ReflectiveMethodInvocation的proceed方法中，有整个拦截器链的责任链模式的执行过程，可以仔细看看，通过责任链序号方式执行的。

```java
public Object proceed() throws Throwable {
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
	    // 链全部执行完，再次调用proceed时，返回原始对象方法调用执行结果。递归的终止。
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	// 用currentInterceptorIndex记录当前的interceptor位置，初值-1，先++再获取。当再拦截器中调用invocation.proceed()时，递归进入此方法，索引向下移位，获取下一个拦截器。
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
	    // 如果是InterceptorAndDynamicMethodMatcher则再执行一次动态匹配
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
		    // 匹配成功，执行
			return dm.interceptor.invoke(this);
		}
		else {
			// 匹配失败，跳过该拦截器，递归调用本方法，执行下一个拦截器。
			return proceed();
		}
	}
	else {
		// 如果是interceptor，则直接调用invoke。把自己作为invocation，以便在invoke方法中，调用invocation.proceed()来执行递归。或者invoke中也可以不执行invocation.proceed()，强制结束递归，返回指定对象作为结果。
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```

cglib的动态代理代码省略......

### 结论

Spring使用了Jdk动态代理和Cglib做代理，但是会把两种代理的拦截器转换为aopalliance这种标准形式进行处理。但是在公开给外部时，其实使用的是advisor这种形式，都注册为advisor或者advised即可。


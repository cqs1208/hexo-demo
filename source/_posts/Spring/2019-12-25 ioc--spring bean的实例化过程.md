---
layout: post
title: ioc--spring bean的实例化过程
tags:
- SpringCore
categories: SpringCore
description: spring源码
---

spring bean 的实例化过程

<!-- more --> 

## 调用链路图

简单链路图

![Spring_springbean01](/Users/admin/Desktop/note/images/Spring/Spring_springbean01.png)

详细链路图：

![Spring_bean03](/Users/admin/Desktop/note/images/Spring/Spring_bean03.png)

## 调用链

比如我们容器中 A a = tcx.getBean(A.class); 容器中的过程是什么?

>I1...org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)
>
>I2...org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
>
>> I2.1...org.springframework.beans.factory.support.AbstractBeanFactory#transformedBeanName转换
>> beanName
>>
>> I2.2...org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton去缓存
>> 中获取bean
>>
>> I2.3...org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance 对缓存中的获取的bean进行后续处理
>>
>> I2.4...org.springframework.beans.factory.support.AbstractBeanFactory#isPrototypeCurrentlyInCreation 判断原型bean的依赖注入
>>
>> I2,5...org.springframework.beans.factory.support.AbstractBeanFactory#getParentBeanFactory 检查父容器加载bean
>>
>> I2.6...org.springframework.beans.factory.support.AbstractBeanFactory#getMergedLocalBeanDefinition 将bean定义转为RootBeanDifination
>>
>> I2.7...org.springframework.beans.factory.support.AbstractBeanFactory#checkMergedBeanDefinition 检查bean的依赖(bean加载顺序的依赖)
>>
>> I2.8...org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton 根据 
>>
>> scope 的添加来创建bean
>
>i3...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean 创建 bean的方法
>
>i4...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean 真正的创建bean的逻辑
>
>> i4.1...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance 调用构造函数创建对象
>>
>> i4.2...判断是否需要提早暴露对象(mbd.isSingleton() && this.allowCircularReferences && i 
>>
>> sSingletonCurrentlyInCreation(beanName));
>>
>> i4.3...org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory  暴露对象解决循环依赖
>>
>> i4.4...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean  给创建的bean进行赋值
>>
>> i4.5...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean  对象进行初始化
>>
>> > i4.5.1...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeAwareMethods  调用XXAware接口
>> >
>> > i4.5.2...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization  调用bean的后置处理器进行对处理
>> >
>> > i4.5.3...org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods  对象的初始化方法
>> >
>> > > i4.5.3.1...org.springframework.beans.factory.InitializingBean#afterPropertiesSet 调用
>> > > InitializingBean的方法
>> > >
>> > > i4.5.3.2...String initMethodName = mbd.getInitMethodName(); 自定义的初始化方法
>> > >
>> > >
>
>i5...org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton 把创建好的实 例化好的bean加载缓存中 
>
>I6...org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance   对创建的bean进行后续的加工 

## 源码分析

### i1 getBean
i1 org.springframework.beans.factory.support.AbstractBeanFactory#getBean

```java
public Object getBean(String name) throws BeansException { 
    return doGetBean(name, null, null, false);
}
```

### I2 doGetBean

I2 org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

```java
protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
       /**
        * 转换对应的beanName 你们可能认为传入进来的name 不就是beanName么?
        * 传入进来的可能是别名,也有可能是是factoryBean
        * 1)去除factoryBean的修饰符 name="&instA"=====>instA
        * 2)取指定的alias所表示的最终beanName 比如传入的是别名为ia---->指向为instA的bean，
           那么就返回	instA 
        **/
        final String beanName = this.transformedBeanName(name);
        Object bean;
       /**
        * 设计的精髓
        * 检查实例缓存中对象工厂缓存中是包含对象(从这里返回的可能是实例话好的,
          也有可能是没有实例化好的)
        * 为什么要这段代码?
        * 因为单实例bean创建可能存主依赖注入的情况，而为了解决循环依赖问题，
          在对象刚刚创建好(属性还没有赋值)
        * 的时候，就会把对象包装为一个对象工厂暴露出去(加入到对象工厂缓存中),
          一但下一个bean要依赖他，就直接可以从缓存中获取. 
         **/
        //直接从缓存中获取或者从对象工厂缓存去取。
        Object sharedInstance = this.getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            ...
            /**
			* 若从缓存中的sharedInstance是原始的bean(属性还没有进行实例化,那么在这里进行处理)
			* 或者是factoryBean 返回的是工厂bean的而不是我们想要的getObject()返回的bean ,
			 就会在这里处理 
			 **/
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
            /**
            * 为什么spring对原型对象就不能解决循环依赖的问题了?
            * 因为spring ioc对原型对象不进行缓存,所以无法提前暴露对象,每次调用都会创建新的对象. *
            * 比如对象A中的属性对象B,对象B中有属性A, 在创建A的时候 检查到依赖对象B，
               那么就会返过来创建对象B，在创建B的过
            * 又发现依赖对象A,由于是原型对象的，ioc容器是不会对实例进行缓存的 
               所以无法解决循环依赖的问题 *
            * */
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            //获取父容器
            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            //如果beanDefinitionMap中所有以及加载的bean不包含 本次加载的beanName，
            //那么尝试取父容器取检测
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (args != null) {
                    //父容器递归查询
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
			//如果这里不是做类型检查，而是创建bean,这里需要标记一下.
            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
                /**
                合并父 BeanDefinition 与子 BeanDefinition，后面会单独分析这个方法 *
                * */
             final RootBeanDefinition mbd=this.getMergedLocalBeanDefinition(beanName);
                this.checkMergedBeanDefinition(mbd, beanName, args);
                //用来处理bean加载的顺序依赖 比如要创建instA 的情况下 必须需要先创建instB 
                /**
                <bean id="beanA" class="BeanA" depends-on="beanB"> 
                <bean id="beanB" class="BeanB" depends-on="beanA">
				创建A之前 需要创建B 创建B之前需要创建A 就会抛出异常
				* */
                String[] dependsOn = mbd.getDependsOn();
                String[] var11;
                if (dependsOn != null) {
                    var11 = dependsOn;
                    int var12 = dependsOn.length;

                    for(int var13 = 0; var13 < var12; ++var13) {
                        String dep = var11[var13];
						//注册依赖
                        this.registerDependentBean(dep, beanName);

                        try {
                            //优先创建依赖的对象
                            this.getBean(dep);
                        } catch (NoSuchBeanDefinitionException var24) {
                            throw new BeanCreationException(...);
                        }
                    }
                }
				//创建bean(单例的 )
                if (mbd.isSingleton()) {
                    //创建单实例bean
                    sharedInstance = this.getSingleton(beanName, new ObjectFactory<Object>() {
                        //在getSingleton房中进行回调用的
                        public Object getObject() throws BeansException {
                            try {
                                return AbstractBeanFactory.this.createBean(beanName, mbd, args);
                            } catch (BeansException var2) {
                                AbstractBeanFactory.this.destroySingleton(beanName);
                                throw var2;
                            }
                        }
                    });
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                    //创建非单实例bean
                } else if (mbd.isPrototype()) {
                    var11 = null;

                    Object prototypeInstance;
                    try {
                        this.beforePrototypeCreation(beanName);
                        prototypeInstance = this.createBean(beanName, mbd, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
                    String scopeName = mbd.getScope();
                    Scope scope = (Scope)this.scopes.get(scopeName);
                    try {
                        Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                            public Object getObject() throws BeansException {
                                AbstractBeanFactory.this.beforePrototypeCreation(beanName);

                                Object var1;
                                try {
                                    var1 = AbstractBeanFactory.this.createBean(beanName, mbd, args);
                                } finally {
                                    AbstractBeanFactory.this.afterPrototypeCreation(beanName);
                                }

                                return var1;
                            }
                        });
                        bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    } catch (IllegalStateException var23) {
                        throw new BeanCreationException(...);
                    }
                }
            } catch (BeansException var26) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var26;
            }
        }

        if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
            try {
                return this.getTypeConverter().convertIfNecessary(bean, requiredType);
            } catch (TypeMismatchException var25) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        } else {
            return bean;
        }
    }
```

#### i2.2 getSingleton

i2.2 org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton 去缓存中
获取bean源码分析

```java
 protected Object getSingleton(String beanName, boolean allowEarlyReference) {
     //去缓存map中获取以及实例化好的bean对象
        Object singletonObject = this.singletonObjects.get(beanName);
     //缓存中没有获取到,并且当前bean是否在正在创建
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            Map var4 = this.singletonObjects;
            //加锁，防止并发创建
            synchronized(this.singletonObjects) {
                //保存早期对象缓存中是否有该对象
                singletonObject = this.earlySingletonObjects.get(beanName);
				//早期对象缓存没有
                if (singletonObject == null && allowEarlyReference) {
                    //早期对象暴露工厂缓存(用来解决循环依赖的)
                    ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                         //调用方法获早期对象
                        singletonObject = singletonFactory.getObject();
                        //放入到早期对象缓存中
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }

        return singletonObject != NULL_OBJECT ? singletonObject : null;
    }
```

#### i2.3 getObjectForBeanInstance

i2.3 org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance
在Bean的生命周期中，

getObjectForBeanInstance方法是频繁使用的方法，无论是从缓存中获取出来的bean还是根据scope创建出来的bean,都要通过该方法进行检查。

1:检查当前bean是否为factoryBean,如果是就需要调用该对象的getObject()方法来返回我们需要的bean对象

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
    //判断name为以 &开头的但是 又不是factoryBean类型的 就抛出异常
        if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(this.transformedBeanName(name), beanInstance.getClass());
        } 
    /**
    * 现在我们有了这个bean，它可能是一个普通bean 也有可能是工厂bean
    * 1)若是工厂bean，我们使用他来创建实例，当如果想要获取的是工厂实例而不是工厂bean的getObject()
      对应的bean,我们应该 * */
    else if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
        //加载factoryBean
            Object object = null;
            if (mbd == null) {
                /*
            * 如果 mbd 为空，则从缓存中加载 bean。FactoryBean 生成的单例 bean 会被缓存 
            *   在 factoryBeanObjectCache 集合中，不用每次都创建
            */
                object = this.getCachedObjectForFactoryBean(beanName);
            }

            if (object == null) {
                // 经过前面的判断，到这里可以保证 beanInstance 是 FactoryBean 类型的，
                //所以可以进行类型转换
                FactoryBean<?> factory = (FactoryBean)beanInstance;
                // 如果 mbd 为空，则判断是否存在名字为 beanName 的 BeanDefinition
                if (mbd == null && this.containsBeanDefinition(beanName)) {
                    //合并我们的bean定义
                    mbd = this.getMergedLocalBeanDefinition(beanName);
                }

                boolean synthetic = mbd != null && mbd.isSynthetic();
                // 调用 getObjectFromFactoryBean 方法继续获取实例
                object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
            }

            return object;
        } else {
            return beanInstance;
        }
    }
```

======核心方法===========getObjectFromFactoryBean()方法==

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    /*
    * FactoryBean 也有单例和非单例之分，针对不同类型的 FactoryBean，这里有两种处理方式:
    * 1. 单例 FactoryBean 生成的 bean 实例也认为是单例类型。需放入缓存中，供后续重复使用
    * 2. 非单例 FactoryBean 生成的 bean 实例则不会被放入缓存中，每次都会创建新的实例 
    */
        if (factory.isSingleton() && this.containsSingleton(beanName)) {
            //加锁，防止重复创建 可以使用缓存提高性能
            synchronized(this.getSingletonMutex()) {
                //从缓存中获取
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    //没有获取到，使用factoryBean的getObject()方法去获取对象
                    object = this.doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    } else {
                        if (object != null && shouldPostProcess) {
                            if (this.isSingletonCurrentlyInCreation(beanName)) {
                                return object;
                            }

                            this.beforeSingletonCreation(beanName);

                            try {
                                //调用ObjectFactory的后置处理器
                                object = this.postProcessObjectFromFactoryBean(object, beanName);
                            } catch (Throwable var14) {
                                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);
                            } finally {
                                this.afterSingletonCreation(beanName);
                            }
                        }

                        if (this.containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object != null ? object : NULL_OBJECT);
                        }
                    }
                }

                return object != NULL_OBJECT ? object : null;
            }
        } else {
            Object object = this.doGetObjectFromFactoryBean(factory, beanName);
            if (object != null && shouldPostProcess) {
                try {
                    object = this.postProcessObjectFromFactoryBean(object, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var17);
                }
            }

            return object;
        }
    }
```

=====================================doGetObjectFromFactoryBean()作用====================

```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, String beanName) throws BeanCreationException {
        Object object;
        try {
            //安全检查
            if (System.getSecurityManager() != null) {
                AccessControlContext acc = this.getAccessControlContext();

                try {
                    object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        public Object run() throws Exception {
                            //调用工厂bean的getObject()方法
                            return factory.getObject();
                        }
                    }, acc);
                } catch (PrivilegedActionException var6) {
                    throw var6.getException();
                }
            } else {
                //调用工厂bean的getObject()方法
                object = factory.getObject();
            }
        } catch (FactoryBeanNotInitializedException var7) {
            throw new BeanCurrentlyInCreationException(beanName, var7.toString());
        } catch (Throwable var8) {
            throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", var8);
        }

        if (object == null && this.isSingletonCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName, "FactoryBean which is currently in creation returned null from getObject");
        } else {
            return object;
        }
    }
```

#### i2.6 getMergedLocalBeanDefinition

i2.6 org.springframework.beans.factory.support.AbstractBeanFactory#getMergedLocalBeanDefinition 将
bean定义转为RootBeanDifination 

合并父子bean定义

```java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
    //去合并的bean定义缓存中 判断当前的bean是否合并过
        RootBeanDefinition mbd = (RootBeanDefinition)this.mergedBeanDefinitions.get(beanName);
    //没有合并，调用合并分方法
        return mbd != null ? mbd : this.getMergedBeanDefinition(beanName, this.getBeanDefinition(beanName));
    }
```

======================getMergedBeanDefinition

```java
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd, BeanDefinition containingBd) throws BeanDefinitionStoreException {
        Map var4 = this.mergedBeanDefinitions;
        synchronized(this.mergedBeanDefinitions) {
            RootBeanDefinition mbd = null;
            //去缓存中获取一次bean定义
            if (containingBd == null) {
                mbd = (RootBeanDefinition)this.mergedBeanDefinitions.get(beanName);
            }
			//尝试没有获取到
            if (mbd == null) {
                //当前bean定义是否有父bean
                if (bd.getParentName() == null) { //没有
                    //转为rootBeanDefinaition 然后深度克隆返回
                    if (bd instanceof RootBeanDefinition) {
                        mbd = ((RootBeanDefinition)bd).cloneBeanDefinition();
                    } else {
                        mbd = new RootBeanDefinition(bd);
                    }
                } else { //有父bean
                    //定义一个父的bean定义
                    BeanDefinition pbd;
                    try {
                     //获取父bena的名称
                    String parentBeanName = transformedBeanName(bd.getParentName());
                    /** 判断父类 beanName 与子类 beanName 名称是否相同。若相同，
                    则父类 bean 一定 * 在父容器中。原因也很简单，容器底层是用 Map 
                    缓存 <beanName, bean> 键值对
                    * 的。同一个容器下，使用同一个 beanName 映射两个 bean 实例显然是不合适的。
                    * 有的朋友可能会觉得可以这样存储:<beanName, [bean1, bean2]> ，似乎解决了
                    * 一对多的问题。但是也有问题，调用 getName(beanName) 时，到底返回哪个 bean
                    * 实例好呢?
                    */
                        if (!beanName.equals(parentBeanName)) {
                            /*
                        * 这里再次调用 getMergedBeanDefinition，只不过参数值变为了 
                        * parentBeanName，用于合并父 BeanDefinition 和爷爷辈的
                        * BeanDefinition。如果爷爷辈的 BeanDefinition 仍有父
                        * BeanDefinition，则继续合并
                        */
                            pbd = this.getMergedBeanDefinition(parentBeanName);
                        } else {
                            //获取父容器
                            BeanFactory parent = this.getParentBeanFactory();
                            if (!(parent instanceof ConfigurableBeanFactory)) {
                                //从父容器获取父bean的定义 //若父bean中有父bean 存储递归合并
                                throw new NoSuchBeanDefinitionException(...);
                            }

                            pbd = ((ConfigurableBeanFactory)parent).getMergedBeanDefinition(parentBeanName);
                        }
                    } catch (NoSuchBeanDefinitionException var10) {
                        throw new BeanDefinitionStoreException(...);
                    }
					//以父 BeanDefinition 的配置信息为蓝本创建 RootBeanDefinition，
                    //也就是“已合并的 BeanDefinition”
                    mbd = new RootBeanDefinition(pbd);
                    //用子 BeanDefinition 中的属性覆盖父 BeanDefinition 中的属性
                    mbd.overrideFrom(bd);
                }
				//若之前没有定义,就把当前的设置为单例的
                if (!StringUtils.hasLength(mbd.getScope())) {
                    mbd.setScope("singleton");
                }
				// 缓存合并后的 BeanDefinition
                if (containingBd == null && this.isCacheBeanMetadata()) {
                    this.mergedBeanDefinitions.put(beanName, mbd);
                }
            }

            return mbd;
        }
    }
```

#### i2.8 getSingleton

i2.8 org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton根据scope
的添加来创建bean

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        Map var3 = this.singletonObjects;
        synchronized(this.singletonObjects) {
            //从缓存中获取对象
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                if (this.singletonsCurrentlyInDestruction) {
                    throw new BeanCreationNotAllowedException(beanName, "Singleton bean creation not allowed while singletons of this factory are in destruction (Do not request a bean from a BeanFactory in a destroy method implementation!)");
                }

                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                }
               //打标.....把正在创建的bean 的标识设置为ture singletonsCurrentlyInDestruction
                this.beforeSingletonCreation(beanName);
                boolean newSingleton = false;
                boolean recordSuppressedExceptions = this.suppressedExceptions == null;
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = new LinkedHashSet();
                }

                try {
                    //调用单实例bean的创建
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                } catch (IllegalStateException var16) {
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        throw var16;
                    }
                } catch (BeanCreationException var17) {
                    BeanCreationException ex = var17;
                    if (recordSuppressedExceptions) {
                        Iterator var8 = this.suppressedExceptions.iterator();

                        while(var8.hasNext()) {
                            Exception suppressedException = (Exception)var8.next();
                            ex.addRelatedCause(suppressedException);
                        }
                    }

                    throw ex;
                } finally {
                    if (recordSuppressedExceptions) {
                        this.suppressedExceptions = null;
                    }

                    this.afterSingletonCreation(beanName);
                }

                if (newSingleton) {
                    //加载到缓存中
                    this.addSingleton(beanName, singletonObject);
                }
            }

            return singletonObject != NULL_OBJECT ? singletonObject : null;
        }
    }
```

==============================================addSingleton(beanName, singletonObject);======

```java
 protected void addSingleton(String beanName, Object singletonObject) {
        Map var3 = this.singletonObjects;
        synchronized(this.singletonObjects) {
            //加入到缓存
            this.singletonObjects.put(beanName, singletonObject != null ? singletonObject : NULL_OBJECT);
            //从早期对象缓存和解决依赖缓存中移除..................
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
```

### i3 createBean

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Creating instance of bean '" + beanName + "'");
        }

        RootBeanDefinition mbdToUse = mbd;
    //根据bean定义和beanName解析class
        Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        try {
            mbdToUse.prepareMethodOverrides();
        } catch (BeanDefinitionValidationException var7) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var7);
        }

        Object beanInstance;
        try {
            //给bean的后置处理器一个机会来生成一个代理对象返回,在aop模块进行详细讲解
            beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
            if (beanInstance != null) {
                return beanInstance;
            }
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var8);
        }
		//真正进行主要的业务逻辑方法来进行创建bean
        beanInstance = this.doCreateBean(beanName, mbdToUse, args);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Finished creating instance of bean '" + beanName + "'");
        }

        return beanInstance;
    }
```

### i4 doCreateBean

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean 真正
的创建bean的逻辑

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }
		//调用构造方法创建bean的实例()
        if (instanceWrapper == null) {
            /**
            * 如果存在工厂方法则使用工厂方法进行初始化
            * 一个类有多个构造函数，每个构造函数都有不同的参数，所以需要根据参数锁定构造 
            函数并进行初始化。 
            * 如果既不存在工厂方法也不存在带有参数的构造函数，则使用默认的构造函数进行 bean 的实例化
            * */
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        final Object bean = instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null;
        Class<?> beanType = instanceWrapper != null ? instanceWrapper.getWrappedClass() : null;
        mbd.resolvedTargetType = beanType;
        Object var7 = mbd.postProcessingLock;
        synchronized(mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    /*bean的后置处理器
                    *bean 合并后的处理， Autowired 注解正是通过此方法实现诸如类型的预解析。 
                    **/
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(...);
                }

                mbd.postProcessed = true;
            }
        }
		//判断当前bean是否需要暴露到 缓存对象中
        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug(...);
            }
			//暴露早期对象到缓存中用于解决依赖的。
            this.addSingletonFactory(beanName, new ObjectFactory<Object>() {
                public Object getObject() throws BeansException {
                    return AbstractAutowireCapableBeanFactory.this.getEarlyBeanReference(beanName, mbd, bean);
                }
            });
        }

        Object exposedObject = bean;

        try {
            //为当前的bean 填充属性，发现依赖等....解决循环依赖就是在这个地方
            this.populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
                //调用bean的后置处理器以及 initionalBean和自己自定义的方法进行初始化
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(...);
        }

        if (earlySingletonExposure) {
            //去缓存中获取对象 只有bean 没有循环依赖 earlySingletonReference才会为空
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                //检查当前的Bean 在初始化方法中没有被增强过(代理过)
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(...);
                    }
                }
            }
        }

        try {
            //注册 DisposableBean。 如果配置了 destroy-method，
            //这里需要注册以便于在销毁时候调用。
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(...);
        }
    }
```

#### i4.1 createBeanInstance

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance 

调用构造函数创建对象 

```java
 /**因为一个类可能有多个构造函数，所以需要根据配置文件中配置的参数或者传入的参数确定最终调用的
       构造函数。	
       因为判断过程会比较消耗性,所以Spring会将解析、确定好的构造函数缓存到BeanDefinition
       中的  resolvedConstructorOrFactoryMethod字段中。在下次创建相同bean
       会直接从RootBeanDefinition中的属性resolvedConstructorOrFactoryMethod缓存的值获取，
       避免再次解析
       */
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
        Class<?> beanClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
        } 
    ///工厂方法不为空则使工厂方法初始化策略 也就是bean的配置过程中设置了factory-method方法
    else if (mbd.getFactoryMethodName() != null) {
            return this.instantiateUsingFactoryMethod(beanName, mbd, args);
        } else {
            boolean resolved = false;
            boolean autowireNecessary = false;
            if (args == null) {
                 // 如果已缓存的解析的构造函数或者工厂方法不为空，则可以利用构造函数解析
				// 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，
                //所有采用缓存机制(缓存到bean定义中)
                Object var7 = mbd.constructorArgumentLock;
                synchronized(mbd.constructorArgumentLock) {
                    if (mbd.resolvedConstructorOrFactoryMethod != null) {
                        resolved = true;
                        //从bean定义中解析出对应的构造函数
                        autowireNecessary = mbd.constructorArgumentsResolved;
                    }
                }
            }
			//已经解析好了，直接注入即可
            if (resolved) {
                return autowireNecessary ? 
                     //autowire 自动注入，调用构造函数自动注入
         this.autowireConstructor(beanName, mbd, (Constructor[])null, (Object[])null) : 
                //使用默认的构造函数
                this.instantiateBean(beanName, mbd);
            } else {
                //根据beanClass和beanName去bean的后置处理器中获取构造方法(SmartInstantiationAwareBeanPostProcessor ----Auto
                Constructor<?>[] ctors = this.determineConstructorsFromBeanPostProcessors(beanClass, beanName);
                return ctors == null && mbd.getResolvedAutowireMode() != 3 && !mbd.hasConstructorArgumentValues() && ObjectUtils.isEmpty(args) ? this.instantiateBean(beanName, mbd) : this.autowireConstructor(beanName, mbd, ctors, args);
            }
        }
    }
```

=======================================提前暴露对象================================

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    Map var3 = this.singletonObjects;
    synchronized(this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            //把bean 作为objectFactory暴露出来..........
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }

    }
}
```

i4.2>:判断是否需要提早暴露对象(mbd.isSingleton() && this.allowCircularReferences && i
sSingletonCurrentlyInCreation(beanName));

#### i4.3 addSingletonFactory

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory 

暴露对象解决循环依赖 

```java
 //判断当前bean是否需要暴露到 缓存对象中
boolean earlySingletonExposure = (mbd.isSingleton() && 
                this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName)); 
	if (earlySingletonExposure) {
	//暴露早期对象到缓存中用于解决依赖的。 
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean); 
            }
		}); 
    }

//暴露早期对象到缓存中用于解决依赖的。
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        Map var3 = this.singletonObjects;
        synchronized(this.singletonObjects) {
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory);
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }

        }
    }
```

#### i4.4 populateBean

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean 给
创建的bean进行赋值

```java
 protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
     //从bean定义中获取属性列表
        PropertyValues pvs = mbd.getPropertyValues();
        if (bw == null) {
            if (!((PropertyValues)pvs).isEmpty()) {
                throw new BeanCreationException(...);
            }
        } else {
            /*
			* 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改
			 bean 状态的机会。官方的解释是:让用户可以自定义属性注入。比如用户实现一
            * 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过
            * postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。当然，
            如果无 * 特殊需求，直接使用配置中的信息注入即可。
            */
            boolean continueWithPropertyPopulation = true;
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Iterator var6 = this.getBeanPostProcessors().iterator();

                while(var6.hasNext()) {
                    BeanPostProcessor bp = (BeanPostProcessor)var6.next();
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                            continueWithPropertyPopulation = false;
                            break;
                        }
                    }
                }
            }

               // 判断注入模型是不是byName 或者是byType的
            if (continueWithPropertyPopulation) {
                if (mbd.getResolvedAutowireMode() == 1 ||
                    mbd.getResolvedAutowireMode() == 2) {
                    //封装属性列表
                    MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                    // 若是基于byName自动转入的
                    if (mbd.getResolvedAutowireMode() == 1) {
                        this.autowireByName(beanName, mbd, bw, newPvs);
                    }
					//计入byType自动注入的
                    if (mbd.getResolvedAutowireMode() == 2) {
                        this.autowireByType(beanName, mbd, bw, newPvs);
                    }
					//把处理过的 属性覆盖原来的
                    pvs = newPvs;
                }
			//判断有没有InstantiationAwareBeanPostProcessors类型的处理器
                boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
                boolean needsDepCheck = mbd.getDependencyCheck() != 0;
                /*
           * 这里又是一种后置处理，用于在 Spring 填充属性到 bean 对象前，对属性的值进行相应的处理， 
                * 比如可以修改某些属性的值。这时注入到 bean 中的值就不是配置文件中的内容了，
                * 而是经过后置处理器修改后的内容
                */
                if (hasInstAwareBpps || needsDepCheck) {
                    //过滤出所有需要进行依赖检查的属性编辑器 并且进行缓存起来
                    PropertyDescriptor[] filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    //通过后置处理器来修改属性
                    if (hasInstAwareBpps) {
                        Iterator var9 = this.getBeanPostProcessors().iterator();

                        while(var9.hasNext()) {
                            BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                                pvs = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
                                if (pvs == null) {
                                    return;
                                }
                            }
                        }
                    }
					//需要检查的化 ，那么需要检查依赖
                    if (needsDepCheck) {
                        this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
                    }
                }
				//设置属性到beanWapper中
                this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
            }
        }
    }
//上诉代码的作用
1)获取了bw的属性列表 
2)在属性列表中被填充的之前，通过InstantiationAwareBeanPostProcessor 对bw的属性进行修改 
3)判断自动装配模型来判断是调用byTypeh还是byName 
4)再次应用后置处理，用于动态修改属性列表 pvs 的内容
5)把属性设置到bw中
```

#### i4.5 initializeBean

i4.5>org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean对
bean进行初始化

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                public Object run() {
                    AbstractAutowireCapableBeanFactory.this.invokeAwareMethods(beanName, bean);
                    return null;
                }
            }, this.getAccessControlContext());
        } else {
            //调用bean实现的 XXXAware接口
            this.invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //调用bean的后置处理器的before方法
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            //调用initianlBean的方法和自定义的init方法
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            //调用Bean的后置处理器的post方法
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
```

##### i4.5.1invokeAwareMethods

i4.5.1>:org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeAwareMethods 调用XXAware接

```java
 private void invokeAwareMethods(String beanName, Object bean) {
        if (bean instanceof Aware) { //判断bean是否实现了Aware接口
            //实现了BeanNameAware接口
            if (bean instanceof BeanNameAware) { 
                ((BeanNameAware)bean).setBeanName(beanName);
            }
            //实现了BeanClassLoaderAware接口
           if (bean instanceof BeanClassLoaderAware) {
          ((BeanClassLoaderAware)bean).setBeanClassLoader(this.getBeanClassLoader());
            }
			//实现了BeanFactoryAware接口
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware)bean).setBeanFactory(this);
            }
        }

    }
```

##### i4.5.2 applyBeanPostProcessorsBeforeInitialization

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization    调用bean的后置处理器进行对处理

```java
  public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;
        Iterator var4 = this.getBeanPostProcessors().iterator();

        do {
            if (!var4.hasNext()) {
                return result;
            }
			//调用所有的后置处理器的before的方法
            BeanPostProcessor processor = (BeanPostProcessor)var4.next();
            result = processor.postProcessBeforeInitialization(result, beanName);
        } while(result != null);

        return result;
    }
```

##### i4.5.3 invokeInitMethods

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods  对象的实例化

```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
    //判断你的bean 是否实现了 InitializingBean接口
        boolean isInitializingBean = bean instanceof InitializingBean;
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug(...);
            }

            if (System.getSecurityManager() != null) {
                try {
                    AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        public Object run() throws Exception {
                            ((InitializingBean)bean).afterPropertiesSet();
                            return null;
                        }
                    }, this.getAccessControlContext());
                } catch (PrivilegedActionException var6) {
                    throw var6.getException();
                }
            } else {
                //调用了InitializingBean的afterPropertiesSet()方法
                ((InitializingBean)bean).afterPropertiesSet();
            }
        }DefaultSingletonBeanRegistry
//调用自己在配置bean的时候指定的初始化方法
        if (mbd != null) {
            String initMethodName = mbd.getInitMethodName();
            if (initMethodName != null && (!isInitializingBean || !"afterPropertiesSet".equals(initMethodName)) && !mbd.isExternallyManagedInitMethod(initMethodName)) {
                this.invokeCustomInitMethod(beanName, bean, mbd);
            }
        }

    }
```

### i5 addSingleton

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton 把创建好的实
例化好的bean加载缓存中

```java
 protected void addSingleton(String beanName, Object singletonObject) {
        Map var3 = this.singletonObjects;
        synchronized(this.singletonObjects) {
            this.singletonObjects.put(beanName, singletonObject != null ? singletonObject : NULL_OBJECT);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
```

## 循环依赖解决链路图

![Spring_bean04](/Users/admin/Desktop/note/images/Spring/Spring_bean04.png)


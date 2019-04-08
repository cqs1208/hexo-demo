---
layout: post
title: 01 ThreadLocal详解
tags:
- JavaCore
categories: JavaCore
description: JavaCore
---

ThreadLocal 可以存储任何类型的变量对象， get返回的是一个Object对象，但是我们可以通过泛型来制定存储对象的类型。

<!-- more --> 

## ThreadLocal详解

### 1，ThreadLocal 方法

ThreadLocal 的几个方法： ThreadLocal 可以存储任何类型的变量对象， get返回的是一个Object对象，但是我们可以通过泛型来制定存储对象的类型。 

```java
public T get() { } // 用来获取ThreadLocal在当前线程中保存的变量副本
public void set(T value) { } //set()用来设置当前线程中变量的副本
public void remove() { } //remove()用来移除当前线程中变量的副本
protected T initialValue() { } //initialValue()是一个protected方法，一般是用来在使用时进行重写的
```

**Thread 在内部是通过ThreadLocalMap来维护ThreadLocal变量表**， 在Thread类中有一个threadLocals 变量，是ThreadLocalMap类型的，它就是为每一个线程来存储自身的ThreadLocal变量的， ThreadLocalMap是ThreadLocal类的一个内部类，这个Map里面的最小的存储单位是一个Entry， 它使用ThreadLocal作为key， 变量作为 value，这是因为在每一个线程里面，可能存在着多个ThreadLocal变量

**初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals**。 
**然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找**

ThreadLocal是如何为每一个线程创建一个变量副本的，下面举一个例子来看一看。

```java
public class ThreadLocalTest {
    public static void main(String[] args) throws InterruptedException {
        final ThreadLocalTest test = new ThreadLocalTest();

        test.set();
        System.out.println(test.getLong());
        System.out.println(test.getString());
        // 在这里新建了一个线程
        Thread thread1 = new Thread() {
            public void run() {
                test.set(); // 当这里调用了set方法，进一步调用了ThreadLocal的set方法是，会将ThreadLocal变量存储到该线程的ThreadLocalMap类型的成员变量threadLocals中，注意的是这个threadLocals变量是Thread线程的一个变量，而不是ThreadLocal类的变量。
                System.out.println(test.getLong());
                System.out.println(test.getString());
            };
        };
        thread1.start();
        thread1.join();

        System.out.println(test.getLong());
        System.out.println(test.getString());
    }

    ThreadLocal<Long> longLocal = new ThreadLocal<Long>();
    ThreadLocal<String> stringLocal = new ThreadLocal<String>();

    public void set() {
        longLocal.set(Thread.currentThread().getId());
        stringLocal.set(Thread.currentThread().getName());
    }

    public long getLong() {
        return longLocal.get();
    }

    public String getString() {
        return stringLocal.get();
    }

}
```

代码的输出结果：  1  main  9  Thread-0  1  main 

### 2,**ThreadLocal 的应用场景**

最常见的ThreadLocal使用场景为  用来解决 **数据库连接**、**Session管理**等。 

数据库连接： 

```java
Class A implements Runnable{
    private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
        public Connection initialValue() {
                return DriverManager.getConnection(DB_URL);
        }
    };

    public static Connection getConnection() {
           return connectionHolder.get();
    }
}
```

Session管理 :

```java
private static final ThreadLocal threadSession = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    Session s = (Session) threadSession.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
}
```

### 3,**ThreadLocal与Connection** 

**疑问1：有一个用户请求就会启动一个线程。而如果ThreadLocal用的是变量副本，那我们把connection放在Threadlocal里的话，那么我们的程序只需要一个connection连接数据库就行了，每个线程都是用的connection的一个副本，那为什么还有必要要数据库连接池呢？** 

ThreadLocal使得各线程能够保持各自独立的一个对象，并不是通过ThreadLocal.set()来实现的，而是通过每个线程中的new 对象 的操作来创建的对象，每个线程创建一个，不是什么对象的拷贝或副本。通过ThreadLocal.set()将这个新创建的对象的引用保存到各线程的自己的一个map中，每个线程都有这样一个map，执行ThreadLocal.get()时，各线程从自己的map中取出放进去的对象，因此取出来的是各自自己线程中的对象，ThreadLocal实例是作为map的key来使用的。

**疑问2：既然ThreadLocal当当前线程中没有时去新建一个新的，有的话就用当前线程中的，那数据库连接池已经有了这种功能啊，还要ThreadLocal干什么？**

由于请求中的一个事务涉及多个 DAO 操作，而这些 DAO 中的 Connection 不能从连接池中获得，如果是从连接池获得的话，两个 DAO 就用到了两个Connection，这样的话是没有办法完成一个事务的。DAO 中的 Connection 如果是从 ThreadLocal 中获得 Connection 的话那么这些 DAO 就会被纳入到同一个 Connection 之下。当然了，这样的话，DAO 中就不能把 Connection 给关了，关掉的话，下一个使用者就不能用了。

### 4.**在自己的项目中使用ThreadLocal解决线程安全问题** 

这里已request对象为例，首先我们新建一个常量类用来新建ThreadLocal对象，代码如下： 

```java
public class CommonConstant {
	public static final ThreadLocal<HttpServletRequest> requestTL = new ThreadLocal<HttpServletRequest>(); // 保存request的threadlocal
}
```

接着新建一个Request工具类，用来获得当前的request或取得request里面的属性等，代码如下： 

```java
public class RequestUtil {
	public static Object getAttribute(String name) {
		return CommonConstant.requestTL.get().getAttribute(name);
	}
	public static void setAttribute(String name, Object value) {
		CommonConstant.requestTL.get().setAttribute(name, value);
	}
	public static void removeAttribute(String name) {
		CommonConstant.requestTL.get().removeAttribute(name);
	}
	public static boolean containsKey(String name) {
		Object value = getAttribute(name);
		if (value != null) {
			return true;
		}
		return false;
	}
	public static boolean notContainsKey(String name) {
		Object value = getAttribute(name);
		if (value == null) {
			return true;
		}
		return false;
	}
	public static HttpServletRequest getRequest() {
		return CommonConstant.requestTL.get();
	}
	public static String getParameter(String name) {
		return CommonConstant.requestTL.get().getParameter(name);
	}
}
```

接着最重要的就是什么时候将这个request放入当前线程，比如在Servlet中当然可以在dosomething(HttpServletRequest request, HttpServletResponse response){}这样的方法里，当然也可以再一些拦截器，过滤器的时候进行设置，如Spring的preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)，代码如下： 

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		CommonConstant.requestTL.set(request);
	}
```

那么接下来在需要的地方如Service层中就可以这样获取到当前线程的request了，代码如下： 

```java

RequestUtil.getRequest();
RequestUtil.getAttribute(name);
RequestUtil.getParameter(name);
//如果没有RequestUtil，那么就先取得request
HttpServletRequest request = CommonConstant.requestTL.get();
```

### 5.**ThreadLocal源代码实现** 

```java
public class ThreadLocal<T> {
    protected T initialValue() {
        return null;
    }
    public ThreadLocal() {
    }
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

从public void set(T value){}可以看出在设置当前线程的线程局部变量时，首先获得当前的线程，接着获取一个与当前线程关联的ThreadLocalMap对象（ThreadLocalMap是ThreadLocal类的一个静态内部类，它实现了键值对的设置和获取，可以用常用的Map对象来理解），如果该ThreadLocalMap对象存在，则将当前线程局部变量和其值放进map，否则的话新建一个map，并使线程的threadLocals为新的map的引用。而当我们通过public T get(){}获取当前线程的线程局部变量时，如果当前线程有相应的ThreadLocalMap对象，则已线程局部变量为键，取出其值，如果map不存在，则通过setInitialValue();将初始化值也就是null设置到map中，并返回该初始化值。

这里我想强调的是每一个Thread对象中有一个ThreadLocal.ThreadLocalMap threadLocals对象引用，指向的是一个ThreadLocalMap对象，该map用来存储一些键值对，如<requestTL,request>;<reponseTL,reponse>，键值对中前一个为定义的线程局部变量，后一个为具体保存的变量。

### 6.ThreadLocal的内存泄露问题 

根据上面Entry方法的源码，我们知道**ThreadLocalMap是使用ThreadLocal的弱引用作为Key的**。下图是本文介绍到的一些对象之间的引用关系图，实线表示强引用，虚线表示弱引用： 

![ThreadLocal](/images/JavaCore/JavaCore_threadLocal.png)

**首先来说，如果把ThreadLocal置为null，那么意味着Heap中的ThreadLocal实例不在有强引用指向，只有弱引用存在，因此GC是可以回收这部分空间的，也就是key是可以回收的。但是value却存在一条从Current Thread过来的强引用链。因此只有当Current Thread销毁时，value才能得到释放。**

**因此，只要这个线程对象被gc回收，就不会出现内存泄露，但在threadLocal设为null和线程结束这段时间内不会被回收的，就发生了我们认为的内存泄露。最要命的是线程对象不被回收的情况，比如使用线程池的时候，线程结束是不会销毁的，再次使用的，就可能出现内存泄露。**

**那么如何有效的避免呢？**

**事实上，在ThreadLocalMap中的set/getEntry方法中，会对key为null（也即是ThreadLocal为null）进行判断，如果为null的话，那么是会对value置为null的。我们也可以通过调用ThreadLocal的remove方法进行释放！**

#### 6.1。线程池中使用ThreadLocal导致的内存泄露

下面先看线程池中使用ThreadLocal的例子： 

```java
public class ThreadPoolTest {

    static class LocalVariable {
        private Long[] a = new Long[1024*1024];
    }

    // (1)
    final static ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());
    // (2)
    final static ThreadLocal<LocalVariable> localVariable = new ThreadLocal<LocalVariable>();

    public static void main(String[] args) throws InterruptedException {
        // (3)
        for (int i = 0; i < 50; ++i) {
            poolExecutor.execute(new Runnable() {
                public void run() {
                    // (4)
                    localVariable.set(new LocalVariable());
                    // (5)
                    System.out.println("use local varaible");
                    //localVariable.remove();

                }
            });

            Thread.sleep(1000);
        }
        // (6)
        System.out.println("pool execute over");
    }
```

- 代码（1）创建了一个核心线程数和最大线程数为5的线程池，这个保证了线程池里面随时都有5个线程在运行。
- 代码（2）创建了一个ThreadLocal的变量，泛型参数为LocalVariable，LocalVariable内部是一个Long数组。
- 代码（3）向线程池里面放入50个任务
- 代码（4）设置当前线程的localVariable变量，也就是把new的LocalVariable变量放入当前线程的threadLocals变量。
- 由于没有调用线程池的shutdown或者shutdownNow方法所以线程池里面的用户线程不会退出，进而JVM进程也不会退出。

运行当前代码，使用jconsole监控堆内存变化如下图：
![image.png](/images/JavaCore/JavaCore_jconsole.png)

然后解开localVariable.remove()注释，然后在运行，观察堆内存变化如下:
![image.png](/images/JavaCore/JavaCore_jconsole2.png)

从运行结果一可知，当主线程处于休眠时候进程占用了大概77M内存，运行结果二则占用了大概25M内存，可知运行代码一时候内存发生了泄露，下面分析下泄露的原因。 

运行结果一的代码，在设置线程的localVariable变量后没有调用`localVariable.remove()`
方法，导致线程池里面的5个线程的threadLocals变量里面的`new LocalVariable()`实例没有被释放，虽然线程池里面的任务执行完毕了，但是线程池里面的5个线程会一直存在直到JVM退出。这里需要注意的是由于localVariable被声明了static，虽然线程的ThreadLocalMap里面是对localVariable的弱引用，localVariable也不会被回收。运行结果二的代码由于线程在设置localVariable变量后即使调用了`localVariable.remove()`方法进行了清理，所以不会存在内存泄露。

总结：线程池里面设置了ThreadLocal变量一定要记得及时清理，因为线程池里面的核心线程是一直存在的，如果不清理，那么线程池的核心线程的threadLocals变量一直会持有ThreadLocal变量。



参考博客：

https://blog.csdn.net/Sup_Heaven/article/details/30094187

http://www.cnblogs.com/dolphin0520/p/3920407.html
---
layout: post
title: LockSupport
tags:
- JUC
categories: JUC
description: 并发编程
---

LockSupport类可以阻塞当前线程以及唤醒指定被阻塞的线程

<!-- more --> 

### 1 LockSupport类介绍

LockSupport类可以阻塞当前线程以及唤醒指定被阻塞的线程。主要是通过park()和unpark(thread)方法来实现阻塞和唤醒线程的操作的。 

> 每个线程都有一个许可(permit)，permit只有两个值1和0,默认是0。
>
> 1. 当调用unpark(thread)方法，就会将thread线程的许可permit设置成1(注意多次调用unpark方法，不会累加，permit值还是1)。
> 2. 当调用park()方法，如果当前线程的permit是1，那么将permit设置为0，并立即返回。如果当前线程的permit是0，那么当前线程就会阻塞，直到别的线程将当前线程的permit设置为1.park方法会将permit再次设置为0，并返回。
>
> 注意：因为permit默认是0，所以一开始调用park()方法，线程必定会被阻塞。调用unpark(thread)方法后，会自动唤醒thread线程，即park方法立即返回。

LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。 

LockSupport中的park() 和 unpark() 的作用分别是阻塞线程和解除阻塞线程，而且park()和unpark()不会遇到“Thread.suspend 和 Thread.resume所可能引发的死锁”问题。 

因为park() 和 unpark()有许可的存在；调用 park() 的线程和另一个试图将其 unpark() 的线程之间的竞争将保持活性。 

### 2 **基本用法** 

LockSupport 很类似于二元信号量(只有1个许可证可供使用)，如果这个许可还没有被占用，当前线程获取许可并继 续 执行；如果许可已经被占用，当前线 程阻塞，等待获取许可。 

```java
public static void main(String[] args)
{
   LockSupport.park();
   System.out.println("block.");
}
```

运行该代码，可以发现主线程一直处于阻塞状态。因为 许可默认是被占用的 ，调用park()时获取不到许可，所以进入阻塞状态。 

如下代码：先释放许可，再获取许可，主线程能够正常终止。LockSupport许可的获取和释放，一般来说是对应的，如果多次unpark，只有一次park也不会出现什么问题，结果是许可处于可用状态。 

```java
public static void main(String[] args)
{
   Thread thread = Thread.currentThread();
   LockSupport.unpark(thread);//释放许可
   LockSupport.park();// 获取许可
   System.out.println("b");
}
```

LockSupport是可不重入 的，如果一个线程连续2次调用 LockSupport .park()，那么该线程一定会一直阻塞下去。 

```java
public static void main(String[] args) throws Exception
{
 Thread thread = Thread.currentThread();
  
 LockSupport.unpark(thread);
  
 System.out.println("a");
 LockSupport.park();
 System.out.println("b");
 LockSupport.park();
 System.out.println("c");
}
```

这段代码打印出a和b，不会打印c，因为第二次调用park的时候，线程无法获取许可出现死锁。

下面我们来看下LockSupport对应中断的响应性

```java
public static void t2() throws Exception
{
 Thread t = new Thread(new Runnable()
 {
  private int count = 0;
 
  @Override
  public void run()
  {
   long start = System.currentTimeMillis();
   long end = 0;
 
   while ((end - start) <= 1000)
   {
    count++;
    end = System.currentTimeMillis();
   }
 
   System.out.println("after 1 second.count=" + count);
 
  //等待或许许可
   LockSupport.park();
   System.out.println("thread over." + Thread.currentThread().isInterrupted());
 
  }
 });
 
 t.start();
 
 Thread.sleep(2000);
 
 // 中断线程
 t.interrupt();

 System.out.println("main over");
}
```

最终线程会打印出thread over.true。这说明 线程如果因为调用park而阻塞的话，能够响应中断请求(中断状态被设置成true)，但是不会抛出InterruptedException 。 

### 3 LockSupport函数列表*

```java
// 返回提供给最近一次尚未解除阻塞的 park 方法调用的 blocker 对象，如果该调用不受阻塞，则返回 null。
static Object getBlocker(Thread t)
// 为了线程调度，禁用当前线程，除非许可可用。
static void park()
// 为了线程调度，在许可可用之前禁用当前线程。
static void park(Object blocker)
// 为了线程调度禁用当前线程，最多等待指定的等待时间，除非许可可用。
static void parkNanos(long nanos)
// 为了线程调度，在许可可用前禁用当前线程，并最多等待指定的等待时间。
static void parkNanos(Object blocker, long nanos)
// 为了线程调度，在指定的时限前禁用当前线程，除非许可可用。
static void parkUntil(long deadline)
// 为了线程调度，在指定的时限前禁用当前线程，除非许可可用。
static void parkUntil(Object blocker, long deadline)
// 如果给定线程的许可尚不可用，则使其可用。
static void unpark(Thread thread)
```

### 4 **LockSupport示例** 

对比下面的“示例1”和“示例2”可以更清晰的了解LockSupport的用法。 

**示例1** 

```java
public class WaitTest1 {
 
  public static void main(String[] args) {
 
    ThreadA ta = new ThreadA("ta");
 
    synchronized(ta) { // 通过synchronized(ta)获取“对象ta的同步锁”
      try {
        System.out.println(Thread.currentThread().getName()+" start ta");
        ta.start();
 
        System.out.println(Thread.currentThread().getName()+" block");
        // 主线程等待
        ta.wait();
 
        System.out.println(Thread.currentThread().getName()+" continue");
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
 
  static class ThreadA extends Thread{
 
    public ThreadA(String name) {
      super(name);
    }
 
    public void run() {
      synchronized (this) { // 通过synchronized(this)获取“当前对象的同步锁”
        System.out.println(Thread.currentThread().getName()+" wakup others");
        notify();  // 唤醒“当前对象上的等待线程”
      }
    }
  }
}
```

**示例2** 

```java
import java.util.concurrent.locks.LockSupport;
 
public class LockSupportTest1 {
 
  private static Thread mainThread;
 
  public static void main(String[] args) {
 
    ThreadA ta = new ThreadA("ta");
    // 获取主线程
    mainThread = Thread.currentThread();
 
    System.out.println(Thread.currentThread().getName()+" start ta");
    ta.start();
 
    System.out.println(Thread.currentThread().getName()+" block");
    // 主线程阻塞
    LockSupport.park(mainThread);
 
    System.out.println(Thread.currentThread().getName()+" continue");
  }
 
  static class ThreadA extends Thread{
 
    public ThreadA(String name) {
      super(name);
    }
 
    public void run() {
      System.out.println(Thread.currentThread().getName()+" wakup others");
      // 唤醒“主线程”
      LockSupport.unpark(mainThread);
    }
  }
}
```

运行结果

```java
main start ta
main block
ta wakup others
main continue
```

说明：park和wait的区别。wait让线程阻塞前，必须通过synchronized获取同步锁。 

### 5 LockSupport源码注释

```java
package java.util.concurrent.locks;
import sun.misc.Unsafe;

import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadLocalRandom;

/**
 * 提供阻塞线程和唤醒线程的方法。
 */
public class LockSupport {
    // 构造函数是私有的，所以不能在外部实例化
    private LockSupport() {}

    // 用来设置线程t的parkBlocker属性。此对象在线程受阻塞时被记录，以允许监视工具和诊断工具确定线程受阻塞的原因。
    private static void setBlocker(Thread t, Object arg) {
        UNSAFE.putObject(t, parkBlockerOffset, arg);
    }

    // 唤醒处于阻塞状态下的thread线程
    public static void unpark(Thread thread) {
        // 当线程不为null时调用
        if (thread != null)
            // 通过UNSAFE的unpark唤醒被阻塞的线程
            UNSAFE.unpark(thread);
    }

    // 阻塞当前线程
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        // 设置线程t的parkBlocker属性，用于记录线程阻塞情况
        setBlocker(t, blocker);
        // 通过UNSAFE的park方法阻塞线程
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }

    // 阻塞当前线程nanos纳秒时间，超出时间线程就会被唤醒返回
    public static void parkNanos(Object blocker, long nanos) {
        if (nanos > 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            UNSAFE.park(false, nanos);
            setBlocker(t, null);
        }
    }
    // 阻塞当前线程，超过deadline日期线程就会被唤醒返回
    public static void parkUntil(Object blocker, long deadline) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(true, deadline);
        setBlocker(t, null);
    }

    // 获取线程t的parkBlocker属性
    public static Object getBlocker(Thread t) {
        if (t == null)
            throw new NullPointerException();
        return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
    }

    // 阻塞当前线程，不设置parkBlocker属性
    public static void park() {
        UNSAFE.park(false, 0L);
    }

    public static void parkNanos(long nanos) {
        if (nanos > 0)
            UNSAFE.park(false, nanos);
    }

    public static void parkUntil(long deadline) {
        UNSAFE.park(true, deadline);
    }

    static final int nextSecondarySeed() {
        int r;
        Thread t = Thread.currentThread();
        if ((r = UNSAFE.getInt(t, SECONDARY)) != 0) {
            r ^= r << 13;   // xorshift
            r ^= r >>> 17;
            r ^= r << 5;
        }
        else if ((r = ThreadLocalRandom.current().nextInt()) == 0)
            r = 1; // avoid zero
        UNSAFE.putInt(t, SECONDARY, r);
        return r;
    }

    // Hotspot implementation via intrinsics API
    private static final Unsafe UNSAFE;
    private static final long parkBlockerOffset;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    static {
        try {
            UNSAFE = Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception ex) { throw new Error(ex); }
    }
}
```

> 


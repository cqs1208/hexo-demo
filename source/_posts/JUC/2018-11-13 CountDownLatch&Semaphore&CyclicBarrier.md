---
layout: post
title: CountDownLatch&Semaphore&CyclicBarrier
tags:
- JUC
categories: JUC
description: 并发编程
---

本文介绍Semaphore、CountDownLatch和CyclicBarrier的基本使用和原理概述

<!-- more --> 

CountDownLatch的一个非常典型的应用场景是：有一个任务想要往下执行，但必须要等到其他的任务执行完毕后才可以继续往下执行。假如我们这个想要继续往下执行的任务调用一个CountDownLatch对象的await()方法，其他的任务执行完自己的任务后调用同一个CountDownLatch对象上的countDown()方法，这个调用await()方法的任务将一直阻塞等待，直到这个CountDownLatch对象的计数值减到0为止。

## CountDownLatch

CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

### 工作原理

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

API 

```
CountDownLatch.countDown()
CountDownLatch.await();
```

### 简单示例

```java
public class CountDownLatchLearn {
 
	private static int LATCH_SIZE = 5;
	private static CountDownLatch doneSignal;
	public static void main(String[] args) {
		try {
			doneSignal = new CountDownLatch(LATCH_SIZE);
			// 新建5个任务
			for(int i=0; i<LATCH_SIZE; i++)
				new InnerThread().start();
 
			System.out.println("main await begin.");
			// "主线程"等待线程池中5个任务的完成
			doneSignal.await();
 
			System.out.println("main await finished.");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
	static class InnerThread extends Thread{
		public void run() {
			try {
				Thread.sleep(1000);
				System.out.println(Thread.currentThread().getName() + " sleep 1000ms.");
				// 将CountDownLatch的数值减1
				doneSignal.countDown();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

说明：创建等待线程数为5，当主线程Main运行到doneSignal.wait()时会阻塞当前线程，直到另外5个线程执行完成之后主线程才会继续执行。 

**构造函数：设置锁标识state的值**

```java
CountDownLatch countDownLatch = new  CountDownLatch(5);//实现的操作是设置锁标识state的值为5
```

**countDownLatch.await()**

调用await的实现是如果state的值不等于0，表示还有其他线程没有执行完（其他线程执行完之后会将state减一操作），此时主线程处于阻塞状态 

```java

public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

这里acquireSharedInterruptibly会进行state状态判断 

```java
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0) //tryAcquireShared函数用来判断state的值是否等于0
            doAcquireSharedInterruptibly(arg);
    }
```

tryAcquireShared中的操作是判断锁标识位state是否等于0，如果不等于0，则调用doAcquireSharedInterruptibly函数，阻塞线程。 

```java
//会将线程添加到FIFO队列中，并阻塞
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
		//将线程添加到FIFO队列中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
				//parkAndCheckInterrupt完成线程的阻塞操作
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

**countDownLatch.countDown()**

**操作是将锁标识位state进行减一操作，如果state此时减一之后为0时则唤起被阻塞线程。**

```java
public void countDown() {
        sync.releaseShared(1); //将state值进行减一操作
    }
```

releaseShared中完成的操作是将锁标识位state进行减一操作，如果state此时减一之后为0时则唤起被阻塞线程 

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {//将锁标识位state进行减arg操作
            doReleaseShared();//唤起阻塞线程操作
            return true;
        }
        return false;
    }
```

在tryReleaseShared中会完成state的减值操作。 

```java
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
				//获取state值
                int c = getState();
                if (c == 0)
                    return false;
				//进行减一操作
                int nextc = c-1;
				//cas操作完成state值的修改
                if (compareAndSetState(c, nextc))
					//如果nextc等于0则返回
                    return nextc == 0;
            }
        }
```

doReleaseShared完成阻塞线程的唤起操作 

```java
private void doReleaseShared() {
 
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
					//完成阻塞线程的唤起操作
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

总结：通过CountDownLatch完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待  

###  CountDownLatch源码： 

```java
public class CountDownLatch {
  
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
 
		//设置state值
        Sync(int count) {
            setState(count);
        }
 
        int getCount() {
            return getState();
        }
	//判断state值是否等于0
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
	//将state锁标识位进行减一操作
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
 
    private final Sync sync;
 
   //count值代表执行等待的次数
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
 
   //当count小于1时，wait线程才会继续执行，不然阻塞
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
 
	//每调用一次count值减1
    public void countDown() {
        sync.releaseShared(1);
    }
 
    public long getCount() {
        return sync.getCount();
    }
 
   
    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```

## Semaphore

Semaphore 字面意思是信号量的意思，它的作用是控制访问特定资源的线程数目。

**构造方法**

```java
public Semaphore(int permits)
public Semaphore(int permits, boolean fair)
- permits 表示许可线程的数量 
- fair 表示公平性，如果这个设为 true 的话，下次执行的线程会是等待最久的线 程 
```

**重要方法**

```java
public void acquire() throws InterruptedException 
public void release()
tryAcquire(long timeout, TimeUnit unit)
- acquire() 表示阻塞并获取许可 
- release() 表示释放许可
```

需求场景：资源访问，服务限流

示例：

```java
public class SemaphoreSample {

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);
        for (int i=0;i<5;i++){
            new Thread(new Task(semaphore,"yangguo+"+i)).start();
        }
    }

    static class Task extends Thread{
        Semaphore semaphore;

        public Task(Semaphore semaphore,String tname){
            this.semaphore = semaphore;
            this.setName(tname);
        }

        public void run() {
            try {
                semaphore.acquire();
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
                Thread.sleep(1000);
                semaphore.release();
                System.out.println(Thread.currentThread().getName()+":aquire() at time:"+System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

打印结果:

```java
Thread-3:aquire() at time:1563096128901
Thread-1:aquire() at time:1563096128901
Thread-1:aquire() at time:1563096129903
Thread-7:aquire() at time:1563096129903
Thread-5:aquire() at time:1563096129903
Thread-3:aquire() at time:1563096129903
Thread-7:aquire() at time:1563096130903
Thread-5:aquire() at time:1563096130903
Thread-9:aquire() at time:1563096130903
Thread-9:aquire() at time:1563096131903
```

从打印结果可以看出，一次只有两个线程执行 acquire()，只有线程进行 release() 方法后
才会有别的线程执行 acquire()。

## CyclicBarrier

栅栏屏障，让一组线程到达一个屏障(也可以叫同步点)时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线 程数量，每个线程调用await方法告CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。 

API 

```java
cyclicBarrier.await();
```

**应用场景**

可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。

```java
public class CyclicBarrierTest implements Runnable {
    private CyclicBarrier cyclicBarrier;
    private int index ;

    public CyclicBarrierTest(CyclicBarrier cyclicBarrier, int
            index) {
        this.cyclicBarrier = cyclicBarrier;
        this.index = index;
    }

    public void run() {
        try {
            System.out.println("index: " + index);
            index--;
            cyclicBarrier.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(11, new
                Runnable() {
            public void run(){
                System.out.println("所有特工到达屏障，准备开始执行秘密任");
            }
    });

    for (int i = 0; i < 10; i++) {
        new Thread(new CyclicBarrierTest(cyclicBarrier,i)).start();
    }
    cyclicBarrier.await(); System.out.println("全部到达屏障....");
    }
}
```


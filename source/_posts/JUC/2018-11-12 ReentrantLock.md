---
layout: post
title: ReentrantLock
tags:
- JUC
categories: JUC
description: 并发编程
---

ReentrantLock

<!-- more --> 

### 1 概括

​	采用synchronized进行加锁，是由jvm内部实现的 称为：内置锁。从java1.5开始，jdk api引入了新锁API 他们都继承自Lock，称为：显式锁 。比如今天的主题ReentrantLock。之所以称做显式锁，主要有两点原因：

1、相对于内置锁是有jvm内部实现的，显式锁是在使用java api实现的，确切的说是基于AQS实现的；

2、使用Lock加锁，需要显式的加锁以及释放锁，相对于内置锁使用synchronized而言，要麻烦些。 

简单来说ReentrantLock提供了两个方法： 

获取锁：

```java
public void lock() {
        sync.lock();
    }
```

释放锁：

```java
public void unlock() {
        sync.release(1);
    }
```

​	ReentrantLock实现机制：首先提供了一个volatile修饰的state，当state=0时表明没有线程获取锁，一个线程想要获取锁时，如果state=0，则将state=1，此时这个线程获取到了锁，其他线程来获取锁时会判断state，此时state不等于0，将此线程添加到一个FIFO队列中，并将线程阻塞，当一个线程释放锁时会通知阻塞的线程，使阻塞线程变成可执行线程来竞争锁（判断state是否是0），当一个线程已经获得锁并再次获取锁时，虽然此时state=1，此时只需要将state++就好，这样这个线程就获取了多次锁，实现机制在AbstractQueuedSynchronizer中接下来我们会对源码简单分析

​	简单来说锁的实现机制是通过一个volatile变量state和线程FIFO来实现的，state为0时锁可获取，线程执行；当state不为0且线程不是当前可执行线程时将线程放到FIFO队列中并阻塞，线程执行完之后再唤醒阻塞线程来进行竞争锁。 

#### 1.1 显示锁和内置锁

​	简单的讲，显式锁的内置锁的补充：显式锁Lock提供了中断锁、定时锁、可轮询等实现，这些都是内置锁synchronized不具备的；显式锁可以指定为公平锁或非公平锁，而内置锁synchronized锁是非公平的。使用synchronized加锁，有些情况下很容易导致死锁，在这种情况下改用显式锁定时功能 在一段时间没有获取到锁，就放弃获取锁 就可以避免死锁。 

​	另外与内置锁还有一个明显不同的地方是，内置锁可以用在方法上，而显式锁只能用在代码块上，也就是说强制使用更细粒度的加锁。 

​	可以说内置锁和显式锁是互补关系：显式锁不能用在方法上，而且容易忘记释放锁；内置锁不可中断，有些情况下容易产生死锁，另外内置锁无法实现公平锁。根据这些差异在自己的实际业务场景中选择性使用即可，需要说明下的是显式锁的性能比内置锁性能稍微好些。 

### 2 ReentrantLock提供的接口 

> // 创建一个 ReentrantLock ，默认是“非公平锁”。
> ReentrantLock()
> // 创建策略是fair的 ReentrantLock。fair为true表示是公平锁，fair为false表示是非公平锁。
> ReentrantLock(boolean fair)
> // 查询当前线程保持此锁的次数。
> int getHoldCount()
> // 返回目前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null。
> protected Thread getOwner()
> // 返回一个 collection，它包含可能正等待获取此锁的线程。
> protected Collection<Thread> getQueuedThreads()
> // 返回正等待获取此锁的线程估计数。
> int getQueueLength()
> // 返回一个 collection，它包含可能正在等待与此锁相关给定条件的那些线程。
> protected Collection<Thread> getWaitingThreads(Condition condition)
> // 返回等待与此锁相关的给定条件的线程估计数。
> int getWaitQueueLength(Condition condition)
> // 查询给定线程是否正在等待获取此锁。
> boolean hasQueuedThread(Thread thread)
> // 查询是否有些线程正在等待获取此锁。
> boolean hasQueuedThreads()
> // 查询是否有些线程正在等待与此锁有关的给定条件。
> boolean hasWaiters(Condition condition)
> // 如果是“公平锁”返回true，否则返回false。
> boolean isFair()
> // 查询当前线程是否保持此锁。
> boolean isHeldByCurrentThread()
> // 查询此锁是否由任意线程保持。
> boolean isLocked()
> // 获取锁。
> void lock()
> // 如果当前线程未被中断，则获取锁。
> void lockInterruptibly()
> // 返回用来与此 Lock 实例一起使用的 Condition 实例。
> Condition newCondition()
> // 仅在调用时锁未被另一个线程保持的情况下，才获取该锁。
> boolean tryLock()
> // 如果锁在给定等待时间内没有被另一个线程保持，且当前线程未被中断，则获取该锁。
> boolean tryLock(long timeout, TimeUnit unit)
> // 试图释放此锁。
>
> void unlock()

#### 2.1 tryLock和lock和lockInterruptibly的区别

- tryLock能获得锁就返回true，不能就立即返回false，tryLock(long timeout,TimeUnit unit)，可以增加时间限制，如果超过该时间段还没获得锁，返回false
- lock能获得锁就返回true，不能的话一直等待获得锁
- lock和lockInterruptibly，如果两个线程分别执行这两个方法，但此时中断这两个线程，前者不会抛出异常，而后者会抛出异常

#### 2.2 Lock的公平锁和非公平锁 

```java
Lock lock=new ReentrantLock(true);//公平锁
Lock lock=new ReentrantLock(false);//非公平锁
```



### 3 ReentrantLock源码 

​	首先需要说明的ReentrantLock与内置锁一样都是排它锁，从名字上开是与内置锁一样是可重入的。 ReentrantLock是排它锁实现，也就是说ReentrantLock的内部类在实现AQS时是对tryAcquire、tryRelease这两个方法进行扩展的。 

#### 3.1 内部类Sync对AQS的实现

```java
abstract static class Sync extends AbstractQueuedSynchronizer {  
    private static final long serialVersionUID = -5179523762034025860L;  
   
    //交给公平和非公平的子类去实现  
    abstract void lock();  
   
    //非公平的排它尝试获取锁实现  
    final boolean nonfairTryAcquire(int acquires) {  
        final Thread current = Thread.currentThread();  
        int c = getState();  
        //如果AQS的state为0说明获得锁，并且对state加1，其他线程获取锁时被阻塞  
        if (c == 0) {  
            if (compareAndSetState(0, acquires)) {  
                setExclusiveOwnerThread(current);  
                return true;  
            }  
        }  
         
        //判断线程是不是重新获取锁，如果是 无需排队，对AQS的state+1处理，这就是重入锁的实现  
        else if (current == getExclusiveOwnerThread()) {  
            int nextc = c + acquires;  
            if (nextc < 0) // overflow  
                throw new Error("Maximum lock count exceeded");  
            setState(nextc);  
            return true;  
        }  
        return false;//都不满足获取锁失败，进入AQS队列阻塞  
    }  
   
    //公平的排它尝试释放锁实现  
    protected final boolean tryRelease(int releases) {  
        int c = getState() - releases;//对应重入锁而言，在释放锁时对AQS的state字段减1  
        if (Thread.currentThread() != getExclusiveOwnerThread())  
            throw new IllegalMonitorStateException();  
        boolean free = false;  
        if (c == 0) {//如果AQS的状态字段已变为0，说明该锁被释放  
            free = true;  
            setExclusiveOwnerThread(null);  
        }  
        setState(c);  
        return free;  
    }  
   
    //判断是否是当前线程持有锁  
    protected final boolean isHeldExclusively() {  
        // While we must in general read state before owner,  
        // we don't need to do so to check if current thread is owner  
        return getExclusiveOwnerThread() == Thread.currentThread();  
    }  
   
    //获取条件队列  
    final ConditionObject newCondition() {  
        return new ConditionObject();  
    }  
   
    // Methods relayed from outer class  
   
    final Thread getOwner() {  
        return getState() == 0 ? null : getExclusiveOwnerThread();  
    }  
   
    final int getHoldCount() {  
        return isHeldExclusively() ? getState() : 0;  
    }  
   
    final boolean isLocked() {  
        return getState() != 0;  
    }  
   
    /** 
     * 说明ReentrantLock是可序列化的 
     */  
    private void readObject(java.io.ObjectInputStream s)  
            throws java.io.IOException, ClassNotFoundException {  
        s.defaultReadObject();  
        setState(0); // reset to unlocked state  
    }  
} 
```

#### 3.2 Sync子类 -- 非公平锁：

```java
static final class NonfairSync extends Sync {  
    private static final long serialVersionUID = 7316153563782823691L;  
   
    /** 
     * Performs lock.  Try immediate barge, backing up to normal 
     * acquire on failure. 
     */  
    final void lock() {  
        //判断当前state是否为0，如果为0直接通过cas修改状态，并获取锁  
        if (compareAndSetState(0, 1))  
            setExclusiveOwnerThread(Thread.currentThread());  
        else  
            acquire(1);//否则进行排队  
    }  
   
    //调用父类的的非公平尝试获取锁  
    protected final boolean tryAcquire(int acquires) {  
        return nonfairTryAcquire(acquires);  
    }  
}  
```

非公平锁的实现很简单，在lock获取锁时首先判断判断当前锁是否可以用（AQS的state状态值是否为0），如果是 直接“插队”获取锁，否则进入排队队列，并阻塞当前线程。 

#### 3.3 Sync子类 -- 公平锁：

```java
static final class FairSync extends Sync {  
    private static final long serialVersionUID = -3000897897090466540L;  
   
    final void lock() {  
        acquire(1);//获取公平，每次都需要进入队列排队  
    }  
   
    /** 
     * 公平锁实现 尝试获取实现方法， 
     * Fair version of tryAcquire.  Don't grant access unless 
     * recursive call or no waiters or is first. 
     */  
    protected final boolean tryAcquire(int acquires) {  
        final Thread current = Thread.currentThread();  
        int c = getState();  
        if (c == 0) {  
            //AQS队列为空，或者当前线程是头节点 即可获的锁  
            if (!hasQueuedPredecessors() &&  
                    compareAndSetState(0, acquires)) {  
                setExclusiveOwnerThread(current);  
                return true;  
            }  
        }  
        //重入锁实现  
        else if (current == getExclusiveOwnerThread()) {  
            int nextc = c + acquires;  
            if (nextc < 0)  
                throw new Error("Maximum lock count exceeded");  
            setState(nextc);  
            return true;  
        }  
        return false;  
    }  
}
```

公平锁的实现，通过hasQueuedPredecessors方法判断当前线程是否是AQS队列中的头结点或者AQS队列为空，并且当前锁状态可用 可以直接获取锁，否则需要排队。 

### 4 ReentrantLock示例： 

```java
public class Operation {
	private Lock lock;
	
	int value = 1;
	
	public Operation(){
		this.lock = new ReentrantLock(false);
	}
	
	public void print1(){
		lock.lock();
		value = value +1;
		System.err.println(value);
		lock.unlock();
	}
	public void print2(){
		lock.lock();
		value = value +2;
		System.err.println(value);
		lock.unlock();
	}
	
}
```

```java
public class LockMain {
 
 
	public static void main(String[] args) {
		
		final Operation operation = new Operation();
		Runnable runnable = new Runnable() {
			
			@Override
			public void run() {
					operation.print1();
				
			}
		};
		Runnable runnable2 = new Runnable() {
			
			@Override
			public void run() {
				operation.print2();
			}
		};
		Runnable runnable3 = new Runnable() {
			
			@Override
			public void run() {
					operation.print1();
				
			}
		};
		
		Thread thread = new Thread(runnable);
		Thread thread2 =  new Thread(runnable2);
		Thread thread3 = new Thread(runnable3);
		
		thread.start();
		thread2.start();
		thread3.start();
	}
 
}
```


---
layout: post
title: condition
tags:
- JUC
categories: JUC
description: 并发编程
---

condition

<!-- more --> 

### 1 概述

我们有时会遇到这样的场景：线程A执行到某个点的时候，因为某个条件condition不满足，需要线程A暂停；等到线程B修改了条件condition，使condition满足了线程A的要求时，A再继续执行。 

### 2 自旋实现的等待通知

最简单的实现方法就是将condition设为一个volatile的变量，当A线程检测到条件不满足时就自旋，类似下面： 

```java
public class Test {
    private static volatile int condition = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread A = new Thread(new Runnable() {
            @Override
            public void run() {
                while (!(condition == 1)) {
                    // 条件不满足，自旋
                }
                System.out.println("a executed");
            }
        });

        A.start();
        Thread.sleep(2000);
        condition = 1;
    }
}
```

这种方式的问题在于自旋非常耗费CPU资源，当然如果在自旋的代码块里加入Thread.sleep(time)将会减轻CPU资源的消耗，但是如果time设的太大，A线程就不能及时响应condition的变化，如果设的太小，依然会造成CPU的消耗。 

### 3 Object提供的等待通知

因此，java在Object类里提供了wait()和notify()方法，使用方法如下： 

```java
class Test1 {
    private static volatile int condition = 0;
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread A = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (lock) {
                    while (!(condition == 1)) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    }
                    System.out.println("a executed by notify");
                }
            }
        });
        A.start();
        Thread.sleep(2000);
        condition = 1;
        synchronized (lock) {
            lock.notify();
        }
    }
}
```

通过代码可以看出，在使用一个对象的wait()、notify()方法前必须要获取这个对象的锁。 

当线程A调用了lock对象的wait()方法后，线程A将释放持有的lock对象的锁，然后将自己挂起，直到有其他线程调用notify()/notifyAll()方法或被中断。可以看到在lock.wait()前面检测condition条件的时候使用了一个while循环而不是if，那是因为当有其他线程把condition修改为满足A线程的要求并调用notify()后，A线程会重新等待获取锁，获取到锁后才从lock.wait()方法返回，而在A线程等待锁的过程中，condition是有可能再次变化的。

因为wait()、notify()是和synchronized配合使用的，因此如果使用了.显示锁Lock就不能用了。所以显示锁要提供自己的等待/通知机制，Condition应运而生。

### 4 显示锁提供的等待通知

我们用Condition实现上面的例子： 

```java
class Test2 {
    private static volatile int condition = 0;
    private static Lock lock = new ReentrantLock();
    private static Condition lockCondition = lock.newCondition();

    public static void main(String[] args) throws InterruptedException {
        Thread A = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();
                try {
                    while (!(condition == 1)) {
                        lockCondition.await();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    lock.unlock();
                }
                System.out.println("a executed by condition");
            }
        });
        A.start();
        Thread.sleep(2000);
        condition = 1;
        lock.lock();
        try {
            lockCondition.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

### 5 应用举例

上面我们看到了Condition实现的等待通知和Object的等待通知是非常类似的，而Condition提供的等待通知功能更强大，最重要的一点是，一个lock对象可以通过多次调用 lock.newCondition() 获取多个Condition对象，也就是说，在一个lock对象上，可以有多个等待队列，而Object的等待通知在一个Object上，只能有一个等待队列。用下面的例子说明，下面的代码实现了一个阻塞队列，当队列已满时，add操作被阻塞有其他线程通过remove方法删除元素；当队列已空时，remove操作被阻塞直到有其他线程通过add方法添加元素。 

```java
public class BoundedQueue1<T> {
    public List<T> q; //这个列表用来存队列的元素
    private int maxSize; //队列的最大长度
    private Lock lock = new ReentrantLock();
    private Condition addConditoin = lock.newCondition();
    private Condition removeConditoin = lock.newCondition();

    public BoundedQueue1(int size) {
        q = new ArrayList<>(size);
        maxSize = size;
    }

    public void add(T e) {
        lock.lock();
        try {
            while (q.size() == maxSize) {
                addConditoin.await();
            }
            q.add(e);
            removeConditoin.signal(); //执行了添加操作后唤醒因队列空被阻塞的删除操作
        } catch (InterruptedException e1) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
        }
    }

    public T remove() {
        lock.lock();
        try {
            while (q.size() == 0) {
                removeConditoin.await();
            }
            T e = q.remove(0);
            addConditoin.signal(); //执行删除操作后唤醒因队列满而被阻塞的添加操作
            return e;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        } finally {
            lock.unlock();
        }
    }

}
```

### 6 源码分析

#### 6.1 概述

之前我们介绍AQS的时候说过，AQS的同步排队用了一个隐式的双向队列，同步队列的每个节点是一个AbstractQueuedSynchronizer.Node实例。 

Node的主要字段有： 

1. waitStatus：等待状态，所有的状态见下面的表格。
2. prev：前驱节点
3. next：后继节点
4. thread：当前节点代表的线程
5. nextWaiter：Node既可以作为同步队列节点使用，也可以作为Condition的等待队列节点使用(将会在后面讲Condition时讲到)。在作为同步队列节点时，nextWaiter可能有两个值：EXCLUSIVE、SHARED标识当前节点是独占模式还是共享模式；在作为等待队列节点使用时，nextWaiter保存后继节点。

Condition实现等待的时候内部也有一个等待队列，等待队列是一个隐式的单向队列，等待队列中的每一个节点也是一个AbstractQueuedSynchronizer.Node实例。

每个Condition对象中保存了firstWaiter和lastWaiter作为队列首节点和尾节点，每个节点使用Node.nextWaiter保存下一个节点的引用，因此等待队列是一个单向队列。

每当一个线程调用Condition.await()方法，那么该线程会释放锁，构造成一个Node节点加入到等待队列的队尾。

#### 6.2 等待

Condition.await()方法的源码如下： 

```java
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter(); //构造一个新的等待队列Node加入到队尾
            int savedState = fullyRelease(node); //释放当前线程的独占锁，不管重入几次，都把state释放为0
            int interruptMode = 0;
            //如果当前节点没有在同步队列上，即还没有被signal，则将当前线程阻塞
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                //后面的蓝色代码都是和中断相关的，主要是区分两种中断：是在被signal前中断还是在被signal后中断，如果是被signal前就被中断则抛出 InterruptedException，否则执行 Thread.currentThread().interrupt();
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)  //被中断则直接退出自旋
                    break;
            }
            //退出了上面自旋说明当前节点已经在同步队列上，但是当前节点不一定在同步队列队首。acquireQueued将阻塞直到当前节点成为队首，即当前线程获得了锁。然后await()方法就可以退出了，让线程继续执行await()后的代码。
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }


    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }

    final boolean isOnSyncQueue(Node node) {
        //如果当前节点状态是CONDITION或node.prev是null，则证明当前节点在等待队列上而不是同步队列上。之所以可以用node.prev来判断，是因为一个节点如果要加入同步队列，在加入前就会设置好prev字段。
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //如果node.next不为null，则一定在同步队列上，因为node.next是在节点加入同步队列后设置的
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node); //前面的两个判断没有返回的话，就从同步队列队尾遍历一个一个看是不是当前节点。
    }

    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    } 
```

```java
 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

#### 6.3 通知

Condition.signal() 方法的源码如下： 

```java
public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException(); //如果同步状态不是被当前线程独占，直接抛出异常。从这里也能看出来，Condition只能配合独占类同步组件使用。
            Node first = firstWaiter;
            if (first != null)
                doSignal(first); //通知等待队列队首的节点。
        }
```

```java
private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&   //transferForSignal方法尝试唤醒当前节点，如果唤醒失败，则继续尝试唤醒当前节点的后继节点。
                     (first = firstWaiter) != null);
        }

    final boolean transferForSignal(Node node) {
        //如果当前节点状态为CONDITION，则将状态改为0准备加入同步队列；如果当前状态不为CONDITION，说明该节点等待已被中断，则该方法返回false，doSignal()方法会继续尝试唤醒当前节点的后继节点
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);  //将节点加入同步队列，返回的p是节点在同步队列中的先驱节点
        int ws = p.waitStatus;
        //如果先驱节点的状态为CANCELLED(>0) 或设置先驱节点的状态为SIGNAL失败，那么就立即唤醒当前节点对应的线程，线程被唤醒后会执行acquireQueued方法，该方法会重新尝试将节点的先驱状态设为SIGNAL并再次park线程；如果当前设置前驱节点状态为SIGNAL成功，那么就不需要马上唤醒线程了，当它的前驱节点成为同步队列的首节点且释放同步状态后，会自动唤醒它。
        //其实笔者认为这里不加这个判断条件应该也是可以的。只是对于CAS修改前驱节点状态为SIGNAL成功这种情况来说，如果不加这个判断条件，提前唤醒了线程，等进入acquireQueued方法了节点发现自己的前驱不是首节点，还要再阻塞，等到其前驱节点成为首节点并释放锁时再唤醒一次；而如果加了这个条件，线程被唤醒的时候它的前驱节点肯定是首节点了，线程就有机会直接获取同步状态从而避免二次阻塞，节省了硬件资源。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

### 7 Condition等待通知的本质

总的来说，Condition的本质就是等待队列和同步队列的交互：

当一个持有锁的线程调用Condition.await()时，它会执行以下步骤：

1. 构造一个新的等待队列节点加入到等待队列队尾
2. 释放锁，也就是将它的同步队列节点从同步队列队首移除
3. 自旋，直到它在等待队列上的节点移动到了同步队列（通过其他线程调用signal()）或被中断
4. 阻塞当前节点，直到它获取到了锁，也就是它在同步队列上的节点排队排到了队首。

当一个持有锁的线程调用Condition.signal()时，它会执行以下操作：

从等待队列的队首开始，尝试对队首节点执行唤醒操作；如果节点CANCELLED，就尝试唤醒下一个节点；如果再CANCELLED则继续迭代。

对每个节点执行唤醒操作时，首先将节点加入同步队列，此时await()操作的步骤3的解锁条件就已经开启了。然后分两种情况讨论：

1. 如果先驱节点的状态为CANCELLED(>0) 或设置先驱节点的状态为SIGNAL失败，那么就立即唤醒当前节点对应的线程，此时await()方法就会完成步骤3，进入步骤4.
2. 如果成功把先驱节点的状态设置为了SIGNAL，那么就不立即唤醒了。等到先驱节点成为同步队列首节点并释放了同步状态后，会自动唤醒当前节点对应线程的，这时候await()的步骤3才执行完成，而且有很大概率快速完成步骤4.

### 8 总结

如果知道Object的等待通知机制，Condition的使用是比较容易掌握的，因为和Object等待通知的使用基本一致。

对Condition的源码理解，主要就是理解等待队列，等待队列可以类比同步队列，而且等待队列比同步队列要简单，因为等待队列是单向队列，同步队列是双向队列。

以下是笔者对等待队列是单向队列、同步队列是双向队列的一些思考，欢迎提出不同意见：

*之所以同步队列要设计成双向的，是因为在同步队列中，节点唤醒是接力式的，由每一个节点唤醒它的下一个节点，如果是由next指针获取下一个节点，是有可能获取失败的，因为虚拟队列每添加一个节点，是先用CAS把tail设置为新节点，然后才修改原tail的next指针到新节点的。因此用next向后遍历是不安全的，但是如果在设置新节点为tail前，为新节点设置prev，则可以保证从tail往前遍历是安全的。因此要安全的获取一个节点Node的下一个节点，先要看next是不是null，如果是null，还要从tail往前遍历看看能不能遍历到Node。*

*而等待队列就简单多了，等待的线程就是等待者，只负责等待，唤醒的线程就是唤醒者，只负责唤醒，因此每次要执行唤醒操作的时候，直接唤醒等待队列的首节点就行了。等待队列的实现中不需要遍历队列，因此也不需要prev指针。*



参考链接：[Java显式锁学习总结之六：Condition源码分析](https://www.cnblogs.com/sheeva/p/6484224.html)
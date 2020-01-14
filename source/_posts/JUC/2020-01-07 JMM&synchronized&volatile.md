---
layout: post
title: 01 JMM&synchronized&volatile
tags:
- JUC
categories: JUC
description: 并发编程
---

本文介绍内容：现代计算机理论模型与工作原理、什么是线程、为什么用到并发，并发的优缺点、JMM模型、volatile、synchronized

<!-- more --> 

## 现代计算机理论模型与工作方式

现代计算机模型是基于-冯诺依曼计算机模型

![juc_jmm01](/Users/admin/Desktop/note/images/JUC/juc_jmm01.png)

CPU内部结构划分:控制单元、运算单元、存储单元

![juc_jmm02](/Users/admin/Desktop/note/images/JUC/juc_jmm02.png)

**控制单元**

控制单元是整个CPU的指挥控制中心，由指令寄存器IR(Instruction Register)、指令 译码器ID(Instruction Decoder)和 操作控制器OC(Operation Controller) 等组成， 对协调整个电脑有序工作极为重要。它根据用户预先编好的程序，依次从存储器中取出各条指 令，放在指令寄存器IR中，通过指令译码(分析)确定应该进行什么操作，然后通过操作控制 器OC，按确定的时序，向相应的部件发出微操作控制信号。操作控制器OC中主要包括:节拍 脉冲发生器、控制矩阵、时钟脉冲发生器、复位电路和启停电路等控制逻辑。 

**运算单元**

运算单元是运算器的核心。可以执行算术运算(包括加减乘数等基本运算及其附加运算)和逻辑运算(包括移位、逻辑测试或两个值比较)。相对控制单元而言，运算器接受控制单元的命令而进行动作，即运算单元所进行的全部操作都是由控制单元发出的控制信号来指挥的，所以它是执行部件。

**存储单元**

存储单元包括 CPU 片内缓存Cache和寄存器组，是 CPU 中暂时存放数据的地方，里面保存着那些等待处理的数据，或已经处理过的数据，CPU 访问寄存器所用的时间要比访问内存的时间短。 寄存器是CPU内部的元件，寄存器拥有非常高的读写速度，所以在寄存器之间的数据传送非常快。采用寄存器，可以减少 CPU 访问内存的次数，从而提高了 CPU 的工作速度。寄存器组可分为专用寄存器和通用寄存器。专用寄存器的作用是固定的，分别寄存相应的数据;而通用寄存器用途广泛并可由程序员规定其用途。

计算机硬件多CPU架构:

![juc_jmm03](/Users/admin/Desktop/note/images/JUC/juc_jmm03.png)

**多CPU**

一个现代计算机通常由两个或者多个CPU，如果要运行多个程序(进程)的话，假如只有一个CPU的话，就意味着要经常进行进程上下文切换，因为单CPU即便是多核的，也只是多个处理器核心，其他设备都是共用的，所以 多个进程就必然要经常进行进程上下文切换，这个代价是很高的。

**CPU多核**

一个现代CPU除了处理器核心之外还包括寄存器、L1L2L3缓存这些存储设备、浮点运算单元、整数运算单元等一些辅助运算设备以及内部总线等。一个多核的CPU也就是一个CPU上有多个处理器核心，这样有什么好处呢?比如说现在我们要在一台计算机上跑一个多线程的程序，因为是一个进程里的线程，所以需要一些共享一些存储变量，如果这台计算机都是单核单线程CPU的话，就意味着这个程序的不同线程需要经常在CPU之间的外部总线上通信，同时还要处理不同CPU之间不同缓存导致数据不一致的问题，所以在这种场景下多核单CPU的架构就能发挥很大的优势，通信都在内部总线，共用同一个缓存。

**CPU寄存器** 

每个CPU都包含一系列的寄存器，它们是CPU内内存的基础。CPU在寄存器上执行操作的 速度远大于在主存上执行的速度。这是因为CPU访问寄存器的速度远大于主存。 

**CPU缓存** 

即高速缓冲存储器，是位于CPU与主内存间的一种容量较小但速度很高的存储器。由于 CPU的速度远高于主存，CPU直接从内存中存取数据要等待一定时间周期，Cache中保存着 CPU刚用过或循环使用的一部分数据，当CPU再次使用该部分数据时可从Cache中直接调用,减少CPU的等待时间，提高了系统的效率。

**内存**

一个计算机还包含一个主存。所有的CPU都可以访问主存。主存通常比CPU中的缓存大得多

**CPU读取存储器数据过程**

- CPU要取寄存器XX的值，只需要一步:直接读取。
- CPU要取L1 cache的某个值，需要1-3步(或者更多):把cache行锁住，把某个数据拿 来，解锁，如果没锁住就慢了。
- CPU要取L2 cache的某个值，先要到L1 cache里取，L1当中不存在，在L2里，L2开始加 锁，加锁以后，把L2里的数据复制到L1，再执行读L1的过程，上面的3步，再解锁。
- CPU取L3 cache的也是一样，只不过先由L3复制到L2，从L2复制到L1，从L1到CPU。 
- CPU取内存则最复杂:通知内存控制器占用总线带宽，通知内存加锁，发起内存读请求， 等待回应，回应数据保存到L3(如果没有就到L2)，再从L3/2到L1，再从L1到CPU，之后解 除总线锁定。 

**缓存一致性问题**

在多处理器系统中，每个处理器都有自己的高速缓存，而它们又共享同一主内存 (MainMemory)。基于高速缓存的存储交互很好地解决了处理器与内存的速度矛盾，但是 也引入了新的问题:缓存一致性(CacheCoherence)。当多个处理器的运算任务都涉及同一 块主内存区域时，将可能导致各自的缓存数据不一致的情况，如果真的发生这种情况，那同步 回到主内存时以谁的缓存数据为准呢?为了解决一致性的问题，需要各个处理器访问缓存时都 遵循一些协议，在读写时要根据协议来进行操作，这类协议有MSI、 MESI(IllinoisProtocol)、MOSI、Synapse、Firefly及DragonProtocol，等等 

![juc_jmm04](/Users/admin/Desktop/note/images/JUC/juc_jmm04.png)

**指令重排序问题**

为了使得处理器内部的运算单元能尽量被充分利用，处理器可能会对输入代码进行乱序执行(Out-Of-Order Execution)优化，处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致。因此，如果存在一个计算任务依赖另一个计算任务的中间结果，那么其顺序性并不能靠代码的先后顺序来保证。与处理器的乱序执行优化类似，Java虚拟机的即时编译器中也有类似的指令重排序(Instruction Reorder)优化

## 什么是线程

现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个Java程序，操作 系统就会创建一个Java进程。现代操作系统调度CPU的最小单元是线程，也叫轻量级进程 (Light Weight Process)，在一个进程里可以创建多个线程，这些线程都拥有各自的计数 器、堆栈和局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。

线程的实现可以分为两类:**用户级线程(User-Level Thread)、内核线线程(Kernel-Level Thread)**

在理解线程分类之前我们需要先了解系统的用户空间与内核空间两个概念，以4G大小的内 存空间为例 

![juc_jmm05](/Users/admin/Desktop/note/images/JUC/juc_jmm05.png)

Linux为内核代码和数据结构预留了几个页框，这些页永远不会被转出到磁盘上。从0x00000000 到 0xc0000000(PAGE_OFFSET) 的线性地址可由用户代码 和 内核代码进行引用(即用户空间)。从0xc0000000(PAGE_OFFSET)到 0xFFFFFFFFF的线性地址只能由内核代码进行访问(即内核空间)。内核代码及其数据结构都必须位于这 1 GB的地址空间中，但是对于此地址空间而言，更大的消费者是物理地址的虚拟映射。

这意味着在 4 GB 的内存空间中，只有 3 GB 可以用于用户应用程序。一个进程只能运行在用户方式(usermode)或内核方式(kernelmode)下。用户程序运行在用户方式下，而系统调用运行在内核方式下。在这两种方式下所用的堆栈不一样:用户方式下用的是一般的堆栈，而内核方式下用的是固定大小的堆栈(一般为一个内存页的大小)

每个进程都有自己的 3 G 用户空间，它们共享1GB的内核空间。当一个进程从用户空间进入内核空间时，它就不再有自己的进程空间了。**这也就是为什么我们经常说线程上下文切换会涉及到用户态到内核态的切换原因所在**

**用户线程**

指不需要内核支持而在用户程序中实现的线程，其不依赖于操作系统核心，应用进程利用线程库提供创建、同步、调度和管理线程的函数来控制用户线程。另外，用户线程是由应用进程利用线程库创建和管理，不依赖于操作系统核心。不需要用户态/核心态切换，速度快。操作系统内核不知道多线程的存在，因此一个线程阻塞将使得整个进程(包括它的所有线程)阻塞。由于这里的处理器时间片分配是以进程为基本单位，所以每个线程执行的时间相对减少。

**内核线程**

线程的所有管理操作都是由操作系统内核完成的。内核保存线程的状态和上下文信息，当一个线程执行了引起阻塞的系统调用时，内核可以调度该进程的其他线程执行。在多处理器系统上，内核可以分派属于同一进程的多个线程在多个处理器上运行，提高进程执行的并行度。由于需要内核完成线程的创建、调度和管理，所以和用户级线程相比这些操作要慢得多，但是仍然比进程的创建和管理操作要快。大多数市场上的操作系统，如Windows，Linux等都支持内核级线程。

原理区别如下图所示

![juc_jmm06](/Users/admin/Desktop/note/images/JUC/juc_jmm06.png)

**Java线程与系统内核线程关系**

![juc_jmm07](/Users/admin/Desktop/note/images/JUC/juc_jmm07.png)

**new java.lang.Thread().start()生命周期步骤**

1. 创建对应的JavaThread的instance

2. 创建对应的OSThread的instance

3. 创建实际的底层操作系统的native thread

4. 准备相应的JVM状态，比如ThreadLocal存储空间分配等

5. 底层的native thread开始运行，调用java.lang.Thread生成的Object的run()方法

6. 当java.lang.Thread生成的Object的run()方法执行完毕返回后,或者抛出异常终止后，终止native thread

7. 释放JVM相关的thread的资源，清除对应的JavaThread和OSThread 

## JMM模型

### 介绍

Java内存模型(Java Memory Model简称JMM)是一种抽象的概念，并不真实存在，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量(包括实例字段，静态字段和构成数组对象的元素)的访问方式。JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(有些地方称为栈空间)，用于存储线程私有的数据，而Java内存模型中规定所有变量都存储在主内存，主内存是共享内存区域，所有线程都可以访问，但线程对变量的操作(读取赋值等)必须在工作内存中进行，首先要将变量从主内存拷贝的自己的工作内存空间，然后对变量进行操作，操作完成后再将变量写回主内存，不能直接操作主内存中的变量，工作内存中存储着主内存中的变量副本拷贝，前面说过，工作内存是每个线程的私有数据区域，因此不同的线程间无法访问对方的工作内存，线程间的通信(传值)必须通过主内存来完成。

**线程，工作内存，主内存工作交互图(基于JMM规范):**

![juc_jmm08](/Users/admin/Desktop/note/images/JUC/juc_jmm08.png)

**主内存**

主要存储的是Java实例对象，所有线程创建的实例对象都存放在主内存中，不管该**实例对象是成员变量还是方法中的本地变量(也称局部变量)**，当然也包括了共享的类信息、常量、静态变量。由于是共享数据区域，多条线程对同一个变量进行访问可能会发生线程安全问题。

**工作内存**

主要存储当前方法的所有本地变量信息(工作内存中存储着主内存中的变量副本拷贝)，每个线程只能访问自己的工作内存，即线程中的本地变量对其它线程是不可见的，就算是两个线程执行的是同一段代码，它们也会各自在自己的工作内存中创建属于当前线程的本地变量，当然也包括了字节码行号指示器、相关Native方法的信息。注意由于工作内存是每个线程的私有数据，线程间无法相互访问工作内存，因此存储在工作内存的数据不存在线程安全问题。

**根据JVM虚拟机规范**主内存与工作内存的数据存储类型以及操作方式，对于一个实例对象中的成员方法而言，如果方法中包含本地变量是基本数据类型(boolean,byte,short,char,int,long,float,double)，将直接存储在工作内存的帧栈结构中，但倘若本地变量是引用类型，那么该变量的引用会存储在功能内存的帧栈中，而对象实例将存储在主内存(共享数据区域，堆)中。但对于实例对象的成员变量，不管它是基本数据类型或者包装类型(Integer、Double等)还是引用类型，都会被存储到堆区。至于static变量以及类本身相关信息将会存储在主内存中。需要注意的是，在主内存中的实例对象可以被多线程共享，倘若两个线程同时调用了同一个对象的同一个方法，那么两条线程会将要操作的数据拷贝一份到自己的工作内存中，执行完成操作后才刷新到主内存

模型如下图所示

![juc_jmm09](/Users/admin/Desktop/note/images/JUC/juc_jmm09.png)

### JMM与硬件内存架构的关系

通过对前面的硬件内存架构、Java内存模型以及Java多线程的实现原理的了解，我们应该已经意识到，多线程的执行最终都会映射到硬件处理器上进行执行，但Java内存模型和硬件内存架构并不完全一致。对于硬件内存来说只有寄存器、缓存内存、主内存的概念，并没有工作内存(线程私有数据区域)和主内存(堆内存)之分，也就是说Java内存模型对内存的划分对硬件内存并没有任何影响，因为JMM只是一种抽象的概念，是一组规则，并不实际存在，不管是工作内存的数据还是主内存的数据，对于计算机硬件来说都会存储在计算机主内存中，当然也有可能存储到CPU缓存或者寄存器中，因此总体上来说，Java内存模型和计算机硬件内存架构是一个相互交叉的关系，是一种抽象概念划分与真实物理硬件的交叉。(注意对于Java内存区域划分也是同样的道理)

![juc_jmm10](/Users/admin/Desktop/note/images/JUC/juc_jmm10.png)

### JMM-同步八种操作介绍

**JMM存在的必要性**

在明白了Java内存区域划分、硬件内存架构、Java多线程的实现原理与Java内存模型的具体关系后，接着来谈谈Java内存模型存在的必要性。由于JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(有些地方称为栈空间)，用于存储线程私有的数据，线程与主内存中的变量操作必须通过工作内存间接完成，主要过程是将变量从主内存拷贝的每个线程各自的工作内存空间，然后对变量进行操作，操作完成后再将变量写回主内存，如果存在两个线程同时对一个主内存中的实例对象的变量进行操作就有可能诱发线程安全问题。

![juc_jmm11](/Users/admin/Desktop/note/images/JUC/juc_jmm11.png)

以上关于主内存与工作内存之间的具体交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步到主内存之间的实现细节，Java内存模型定义了以下八种操作来完成。

**JMM-同步八种操作介绍**

**lock(锁定)**:作用于主内存的变量，把一个变量标记为一条线程独占状态
**unlock(解锁)**:作用于主内存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
**read(读取)**:作用于主内存的变量，把一个变量值从主内存传输到线程的工作内存中，以便随后的load动作使用
**load(载入)**:作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中
**use(使用)**:作用于工作内存的变量，把工作内存中的一个变量值传递给执行引擎
**assign(赋值)**:作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量
**store(存储)**:作用于工作内存的变量，把工作内存中的一个变量的值传送到主内存中，以便随后的write的操作
**write(写入)**:作用于工作内存的变量，它把store操作从工作内存中的一个变量的值传送到主内存的变量中

如果要把一个变量从主内存中复制到工作内存中，就需要按顺序地执行read和load操作，如果把变量从工作内存中同步到主内存中，就需要按顺序地执行store和write操作。但Java内存模型只要求上述操作必须按顺序执行，而没有保证必须是连续执行。

![juc_jmm12](/Users/admin/Desktop/note/images/JUC/juc_jmm12.png)

**同步规则分析**

1): 不允许一个线程无原因地(没有发生过任何assign操作)把数据从工作内存同步回主内存中
2): 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化(load或者assign)的变量。即就是对一个变量实施use和store操作之前，必须先自行assign和load操作。
3): 一个变量在同一时刻只允许一条线程对其进行lock操作，但lock操作可以被同一线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。lock和unlock必须成对出现。
4): 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量之前需要重新执行load或assign操作初始化变量的值。
5): 如果一个变量事先没有被lock操作锁定，则不允许对它执行unlock操作;也不允许去unlock一个被其他线程锁定的变量。
6): 对一个变量执行unlock操作之前，必须先把此变量同步到主内存中(执行store和write操作)





## 并发编程中的三个问题

### 可见性

可见性(Visibility):是指一个线程对共享变量进行修改，另一个先立即得到修改后的最新值。

**演示**

```java
/**
案例演示: 一个线程对共享变量的修改,另一个线程不能立即得到最新值
*/
public class Test01Visibility {
// 多个线程都会访问的数据，我们称为线程的共享数据
private static boolean run = true;
public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            while (run) {
} });
t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(() -> {
            run = false;
System.out.println("时间到，线程2设置为false"); });
        t2.start();
    }
}
```

**小结**

并发编程时，会出现可见性问题，当一个线程对共享变量进行了修改，另外的线程并没有立即看到修改后的最新值。 

### 原子性

原子性(Atomicity):在一次或多次操作中，要么所有的操作都执行并且不会受其他因素干扰而中
断，要么所有的操作都不执行。

**演示**

```java
/*
    目标:演示原子性问题
        1.定义一个共享变量number
        2.对number进行1000的++操作
        3.使用5个线程来进行
 */
public class Test02Atomicity {
    // 1.定义一个共享变量number
    private static int number = 0;
    public static void main(String[] args) throws InterruptedException {
        // 2.对number进行1000的++操作
        Runnable increment = () -> {
            for (int i = 0; i < 1000; i++) {
                number++;
            }
        };

        List<Thread> list = new ArrayList<>();
        // 3.使用5个线程来进行
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(increment);
            t.start();
            list.add(t);
        }

        for (Thread t : list) {
            t.join();
        }

        System.out.println("number = " + number);
    }
}
```

使用javap反汇编class文件，得到下面的字节码指令:`javap -p -v Test02Atomicity.class`

```java
  ...
private static void lambda$main$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=1, args_size=0
         0: iconst_0
         1: istore_0
         2: iload_0
         3: sipush        1000
         6: if_icmpge     23
         9: getstatic     #18                 // Field number:I
        12: iconst_1
        13: iadd
        14: putstatic     #18                 // Field number:I
        17: iinc          0, 1
        20: goto          2
        23: return
      LineNumberTable:
...
```

其中，对于 number++ 而言(number 为静态变量)，实际会产生如下的 JVM 字节码指令:

```java
 9: getstatic     #18                 // Field number:I
 12: iconst_1
 13: iadd
 14: putstatic     #18                 // Field number:I
```

由此可见number++是由多条语句组成，以上多条指令在一个线程的情况下是不会出问题的，但是在多
线程情况下就可能会出现问题。比如一个线程在执行13: iadd时，另一个线程又执行9: getstatic。会导
致两次number++，实际上只加了1。

**小结**

并发编程时，会出现原子性问题，当一个线程对共享变量操作到一半时，另外的线程也有可能来操作共
享变量，干扰了前一个线程的操作。

### 有序性

有序性(Ordering):是指程序中代码的执行顺序，Java在编译时和运行时会对代码进行优化，会导致
程序最终的执行顺序不一定就是我们编写代码时的顺序。

```java
public static void main(String[] args) {
    int a = 10;
	int b = 20; }
```

**演示**

```java
public class Test03Orderliness {
int num = 0;
boolean ready = false;
// 线程一执行的代码
 Runnable increment = () -> {
    if(ready) {
        System.out.println(num + num);
    } else {
        System.out.println(1);
    }
 };
    
 Runnable increment = () -> {
    num = 2;
    ready = true;
 };
 // 如果结果为0,则可知指令进行了重排序
```

**小结** 

程序代码在执行过程中的先后顺序，由于Java在编译期以及运行期的优化，导致了代码的执行顺序未必 就是开发者编写代码时的顺序。 

## 指令重排序&happens-before

**指令重排序**:java语言规范规定JVM线程内部维持顺序化语义。即只要程序的最终结果与它顺序化情况的结果相等，那么指令的执行顺序可以与代码顺序不一致，此过程叫指令的重排序。指令重排序的意义是什么?JVM能根据处理器特性(CPU多级缓存系统、多核处理器等)适当的对机器指令进行重排序，使机器指令能更符合CPU的执行特性，最大限度的发挥机器性能。

下图为从源码到最终执行的指令序列示意图

![juc_jmm13](/Users/admin/Desktop/note/images/JUC/juc_jmm13.png)

**as-if-serial语义**

as-if-serial语义的意思是:不管怎么重排序(编译器和处理器为了提高并行度)，(单线 程)程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义。 

为了遵守as-if-serial语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因 为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被 编译器和处理器重排序。 

**happens-before 原则**

只靠sychronized和volatile关键字来保证原子性、可见性以及有序性，那么编写并发程序可能会显得十分麻烦，幸运的是，从JDK 5开始，Java使用新的JSR-133内存模型，提供了happens-before 原则来辅助保证程序执行的原子性、可见性以及有序性的问题，它是判断数据是否存在竞争、线程是否安全的依据，happens-before 原则内容如下

1): 程序顺序原则，即在一个线程内必须保证语义串行性，也就是说按照代码顺序执行。 

2):  锁规则 解锁(unlock)操作必然发生在后续的同一个锁的加锁(lock)之前，也就是说， 如果对于一个锁解锁后，再加锁，那么加锁的动作必须在解锁动作之后(同一个锁)。
3):  volatile规则 volatile变量的写，先发生于读，这保证了volatile变量的可见性，简单 的理解就是，volatile变量在每次被线程访问时，都强迫从主内存中读该变量的值，而当 该变量发生变化时，又会强迫将最新的值刷新到主内存，任何时刻，不同的线程总是能 够看到该变量的最新值。 

4): 线程启动规则 线程的start()方法先于它的每一个动作，即如果线程A在执行线程B的 start方法之前修改了共享变量的值，那么当线程B执行start方法时，线程A对共享变量 的修改对线程B可见
5): 传递性 A先于B ，B先于C 那么A必然先于C 

6): 线程终止规则 线程的所有操作先于线程的终结，Thread.join()方法的作用是等待当前 执行的线程终止。假设在线程B终止之前，修改了共享变量，线程A从线程B的join方法 成功返回后，线程B对共享变量的修改将对线程A可见。
7):  线程中断规则 对线程 interrupt()方法的调用先行发生于被中断线程的代码检测到中 断事件的发生，可以通过Thread.interrupted()方法检测线程是否中断。 

8):对象终结规则 对象的构造函数执行，结束先于finalize()方法 

## volatile

### volatile内存语义

volatile是Java虚拟机提供的轻量级的同步机制。volatile关键字有如下两个作用

- 保证被volatile修饰的共享变量对所有线程总数可见的，也就是当一个线程修改了一 个被volatile修饰共享变量的值，新值总是可以被其他线程立即得知。 
- 禁止指令重排序优化

### volatile的可见性

关于volatile的可见性作用，我们必须意识到被volatile修饰的变量对所有线程总数立即可 见的，对volatile变量的所有写操作总是能立刻反应到其他线程中 

示例：

```JAVA
public class VolatileVisibilitySample {
    volatile boolean initFlag = false;

    public void save(){
        this.initFlag = true;
        String threadname = Thread.currentThread().getName();
        System.out.println("线程:"+threadname+":修改共享变量initFlag");
    }

    public void load(){
        String threadname = Thread.currentThread().getName();
        while(!initFlag){
            //线程在此处空跑，等待initFlag状态改变
        }
        System.out.println("线程:"+threadname+"当前线程嗅探到initFlag的状态的改变");
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileVisibilitySample  sample  = new VolatileVisibilitySample();
        Thread threadA = new Thread(() -> {sample.save();}, "thireadA");
        Thread threadB = new Thread(() -> {sample.load();}, "thireadB");
        threadB.start();
        Thread.sleep(2000);
        threadA.start();
    }
}
```

线程A改变initFlag属性之后，线程B马上感知到

### volatile无法保证原子性

示例：

```java
public class VolatileVisibility {
    public static volatile int i = 0;

    public static void increase(){
        i++;
    }
}
```

在并发场景下，i变量的任何改变都会立马反应到其他线程中，但是如此存在多条线程同时调用increase()方法的话，就会出现线程安全问题，毕竟i++;操作并不具备原子性，该操作是先读取值，然后写回一个新值，相当于原来的值加上1，分两步完成，如果第二个线程在第一个线程读取旧值和写回新值期间读取i的域值，那么第二个线程就会与第一个线程一起看到同一个值，并执行相同值的加1操作，这也就造成了线程安全失败，因此对于increase方法必须使用synchronized修饰，以便保证线程安全，需要注意的是一旦使用synchronized修饰方法后，由于synchronized本身也具备与volatile相同的特性，即可见性，因此这样种情况下就完全可以省去volatile修饰变量。

### volatile禁止重排优化

volatile关键字另一个作用就是禁止指令重排优化，从而避免多线程环境下程序出现乱序 执行的现象，关于指令重排优化前面已详细分析过，这里主要简单说明一下volatile是如何实 现禁止指令重排优化的。先了解一个概念，内存屏障(Memory Barrier)。 

内存屏障，又称内存栅栏，是一个CPU指令，它的作用有两个，一是保证特定操作的执行顺序，二是保证某些变量的内存可见性(利用该特性实现volatile的内存可见性)。由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条Memory Barrier则会告诉编译器和CPU，不管什么指令都不能和这条Memory Barrier指令重排序，也就是说通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化。Memory Barrier的另外一个作用是强制刷出
各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。总之，volatile变量正是通过内存屏障实现其在内存中的语义，即可见性和禁止重排优化。下面看一个非常典型的禁止重排优化的例子DCL，如下:

```java
public class DoubleCheckLock {
    DoubleCheckLock  instance;
    DoubleCheckLock(){}

    public  DoubleCheckLock  getInstance(){
        // 第一次检测
        if(instance == null){
            synchronized (DoubleCheckLock.class){
                if(instance == null){
                    //
                    instance = new DoubleCheckLock();
                }
            }
        }
        return instance;
    }
}
```

上述代码一个经典的单例的双重检测的代码，这段代码在单线程环境下并没有什么问题，但如果在多线程环境下就可以出现线程安全问题。原因在于某一个线程执行到第一次检测，读取到的instance不为null时，instance的引用对象可能没有完成初始化。

因为instance = new DoubleCheckLock();可以分为以下3步完成(伪代码)

	memory = allocate();//1.分配对象内存空间
	instance(memory);//2.初始化对象
	instance = memory;//3.设置instance指向刚分配的内存地址，此时instance!=null

由于步骤1和步骤2间可能会重排序，如下:

	memory=allocate();//1.分配对象内存空间
	instance=memory;//3.设置instance指向刚分配的内存地址，此时instance!=null，但对象还没有初始化完成!
	instance(memory);//2.初始化对象

由于步骤2和步骤3不存在数据依赖关系，而且无论重排前还是重排后程序的执行结果在单线程中并没有改变，因此这种重排优化是允许的。但是指令重排只会保证串行语义的执行的一致性(单线程)，但并不会关心多线程间的语义一致性。所以当一条线程访问instance不为null时，由于instance实例未必已初始化完成，自然而然在使用instance时会出错，也就造成了线程安全问题。那么该如何解决呢，很简单，我们使用volatile禁止instance变量被执行指令重排优化即可。

```java
//禁止指令重排优化
public volatile DoubleCheckLock  instance;
```

### volatile内存语义的实现

前面提到过重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM 会分别限制这两种类型的重排序类型。 

下图是JMM针对编译器制定的volatile重排序规则表。

![juc_jmm14](/Users/admin/Desktop/note/images/JUC/juc_jmm14.png)

举例来说，第三行最后一个单元格的意思是:在程序中，当第一个操作为普通变量的读或 写时，如果第二个操作为volatile写，则编译器不能重排序这两个操作。 

从上图可以看出: 

- 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保 volatile写之前的操作不会被编译器重排序到volatile写之后。 
- 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保 volatile读之后的操作不会被编译器重排序到volatile读之前。 
- 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。 

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来 禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数 几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略。 

- 在每个volatile写操作的前面插入一个StoreStore屏障。 

- 在每个volatile写操作的后面插入一个StoreLoad屏障。

- 在每个volatile读操作的后面插入一个LoadLoad屏障。 

- 在每个volatile读操作的后面插入一个LoadStore屏障。 

上述内存屏障插入策略非常保守，但它可保证在任意处理器平台，任意程序中都能得 到正确的volatile内存语义。 

下面是保守策略下，volatile写插入内存屏障后生成的指令序列示意图

![juc_jmm15](/Users/admin/Desktop/note/images/JUC/juc_jmm15.png)

上图中StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

这里比较有意思的是，volatile写后面的StoreLoad屏障。此屏障的作用是避免volatile写与 后面可能有的volatile读/写操作重排序。因为编译器常常无法准确判断在一个volatile写的后面 是否需要插入一个StoreLoad屏障(比如，一个volatile写之后方法立即return)。为了保证能正确 实现volatile的内存语义，JMM在采取了保守策略:在每个volatile写的后面，或者在每个volatile 读的前面插入一个StoreLoad屏障。从整体执行效率的角度考虑，JMM最终选择了在每个 volatile写的后面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是:一个 写线程写volatile变量，多个读线程读同一个volatile变量。当读线程的数量大大超过写线程时，选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。从这里可以看到JMM 在实现上的一个特点:首先确保正确性，然后再去追求执行效率。

下图是在保守策略下，volatile读插入内存屏障后生成的指令序列示意图

![juc_jmm16](/Users/admin/Desktop/note/images/JUC/juc_jmm16.png)

上图中LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

上述volatile写和volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变 volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。下面通过具体的示例

```java
public class VolatileBarrierExample {
    int a ;
    volatile int v1 = 1;
    volatile int v2 = 2;

    void readAndWrite(){
        int i = v1; // 第一个volatile读
        int j = v2; // 第二个volatile读
        a = i + j;  // 普通写
        v1 = i + 1; // 第一个volatile写
        v2 = j * 2; // 第二个 volatile写
    }
}
```

针对readAndWrite()方法，编译器在生成字节码时可以做如下的优化。

![juc_jmm17](/Users/admin/Desktop/note/images/JUC/juc_jmm17.png)

**注意**，最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即return。此时编 译器可能无法准确断定后面是否会有volatile读或写，为了安全起见，编译器通常会在这里插 入一个StoreLoad屏障。

上面的优化针对任意处理器平台，由于不同的处理器有不同“松紧度”的处理器内存模 型，内存屏障的插入还可以根据具体的处理器内存模型继续优化。以X86处理器为例，图3-21 中除最后的StoreLoad屏障外，其他的屏障都会被省略。

![juc_jmm18](/Users/admin/Desktop/note/images/JUC/juc_jmm18.png)

## synchronized 

### 基本使用

java中的同步块在某些对象上是同步的。在一个对象上的所有同步块，在同一时间只能有一个线程在执行。其他的线程尝试进入同步块将会被阻塞  ，直到同步块中的线程退出该代码块。  synchronized可以作用于不同类型块。 

**1：实例方法上。 2：实例方法上的代码块。 3：静态方法上的代码块**

**synchronized实例方法**

实例方法中的同步块，代码如下 

```java
public synchronized void add(int value){
      this.count += value;
  }
```

synchronized作用于一个方法上，我们说这个方法是同步方法。一个同步实例方法在java中是同步与对象实例的方法上。因此每个实例有它自己的同步方法同步在不同的对象上：他们自己的实例对象。 
只有一个线程可以在同步实例方法上执行。 

如果超过一个实例存在（这个类的对象有多个），在每个实例的同步方法中只能有一个线程，也就是一个线程一个实例。如下代码:

```java
package synchironized_ex;

/**
 * Created by fang on 2017/11/23.
 * 类的每个实例同步示例.
 */
public class InstanceSynchronized {
    public synchronized void instanceMethod(){
        System.out.println("执行实例同步方法开始......");
         try{
            Thread.sleep(1000);
        }catch (InterruptedException e){
            e.printStackTrace();
        }
        System.out.println("执行实例同步方法结束......");

    }

}
package synchironized_ex;

/**
 * Created by fang on 2017/11/23.
 * 线程类.
 */
public class InstanceThread extends Thread {
    public void run(){
        InstanceSynchronized instanceSynchronized = new InstanceSynchronized();
        instanceSynchronized.instanceMethod();
    }
}
package synchironized_ex;

/**
 * Created by fang on 2017/11/23.
 * main方法
 */
public class ThreadClient {
    public static void main(String[] args) {
        for(int i = 0;i<3; i++){
            Thread myThread = new InstanceThread();
            myThread.start();
        }
    }
}
```

返回结果：

```java
执行实例同步方法开始......
执行实例同步方法开始......
执行实例同步方法开始......
执行实例同步方法结束......
执行实例同步方法结束......
执行实例同步方法结束......   
```

从上面结果可以看出来，synchronized作用于类的多个实例上，每个实例保证一次只有一个线程，并不是多个实例保证一次只有一个线程。实际上是“对象锁” 

**synchronized静态方法**

类中的静态方法被关键字synchronized标记，如下 

```java
public static synchronized void add(int value){
      count += value;
  }
```

同样，这里synchronized关键字告诉java这个方法是同步的。  在java虚拟机中一个类只能对应一个类对象（注意不是实例对象），所以同时只允许一个线程执行类对象的静态方法。  我们把上面的代码中线程执行的方法中改为静态方法： 

```java
package synchironized_ex;

/**
 * Created by fang on 2017/11/23.
 * 类的每个实例同步示例.
 */
public class InstanceSynchronized {
    public static synchronized void instanceMethod(){//加static和不加static的区别,锁类和锁对象.
        System.out.println("执行实例同步方法开始......");
         try{
            Thread.sleep(1000);
        }catch (InterruptedException e){
            e.printStackTrace();
        }
        System.out.println("执行实例同步方法结束......");

    }

}
```

返回结果：

```java
执行实例同步方法开始......
执行实例同步方法结束......
执行实例同步方法开始......
执行实例同步方法结束......
执行实例同步方法开始......
执行实例同步方法结束......
```

**注：非静态方法时对象持有锁，而对于静态方法来说是类持有锁。 **

**实例方法中的同步块**

有时候我们不需要同步整个方法,可能需要同步的仅仅是方法中的一块代码,java可以在方法中写同步代码块,如下所示: 

```java
public void add(int value){
    synchronized(this){
       this.count += value;   
    }
  }
```

上述中使用了”this”,调用的是add方法本身的实例. 在同步构造其中用括号括起来的对象叫做”监视器对象”. 上述代码使用监视器对象同步, 同步方法使用调用方法本身的实例作为监视器对象. 
一次只能有一个线程能够同步于同一个监视器对象的方法内执行. 

下面的例子都同步在他们所调用的对象的实例上, 因此调用效果是等效的

```java
public class MyClass {

  public synchronized void log1(String msg1, String msg2){
     log.writeln(msg1);
     log.writeln(msg2);
  }
   public void log2(String msg1, String msg2){
     synchronized(this){  
            log.writeln(msg1);
         log.writeln(msg2);
      }
   }
 }
```

在上述例子中, 每次只有一个线程能够在同步块中的任意一个方法内执行.  如果第二个方法块不是同步在this实例对象上, 那么两个方法可以被线程同时执行. 

**静态方法中的同步块**

下面是两个静态方法的同步块 

```java
 public class MyClass {
    public static synchronized void log1(String msg1, String msg2){
      log.writeln(msg1);
      log.writeln(msg2);
    }

   public static void log2(String msg1, String msg2){
       synchronized(MyClass.class){
          log.writeln(msg1);
          log.writeln(msg2);
       }
    }
  }
```

这两个方法不允许同时被线程访问, 如果第二个同步块不是同步在MyClass.class这个类上, 那么可以同时被线程访问. 

使用同步方法注意项

1. 线程同步的目的是为了保护多个线程访问一个资源时对资源的破坏。 
2. 线程同步方法是通过锁来实现，每个对象都有且仅有一个锁，这个锁与一个特定的对象关联，线程一旦获取了对象锁，其他访问该对象的线程就无法再访问该对象的其他同步方法。 
3. 对于静态同步方法，锁是针对这个类的，锁对象是该类的Class对象。静态和非静态方法的锁互不干预。一个线程获得锁，当在一个同步方法中访问另外对象上的同步方法时，会获取这两个对象锁。 
4. 对于同步，要时刻清醒在哪个对象上同步，这是关键。 
5. 编写线程安全的类，需要时刻注意对多个线程竞争访问资源的逻辑和安全做出正确的判断，对“原子”操作做出分析，并保证原子操作期间别的线程无法访问竞争资源。 
6. 当多个线程等待一个对象锁时，没有获取到锁的线程将发生阻塞。 
7. 死锁是线程间相互等待对方释放锁造成的，在实际中发生的概率非常的小

### synchronized 特性

**可重入特性**

```java
/*
    目标:演示synchronized可重入
        1.自定义一个线程类
        2.在线程类的run方法中使用嵌套的同步代码块
        3.使用两个线程来执行
 */
public class Demo01 {
    public static void main(String[] args) {
        new MyThread().start();
        new MyThread().start();
    }

    public static void test01() {
        synchronized (MyThread.class) {
            String name = Thread.currentThread().getName();
            System.out.println(name + "进入了同步代码块2");
        }
    }
}

// 1.自定义一个线程类
class MyThread extends Thread {
    @Override
    public void run() {
        synchronized (MyThread.class) {
            System.out.println(getName() + "进入了同步代码块1");

            Demo01.test01();
        }
    }
}
```

注：synchronized的锁对象中有一个计数器(recursions变量)会记录线程获得几次锁.

**小结** 

synchronized是可重入锁，内部锁对象中会有一个计数器记录线程获取几次锁啦，在执行完同步代码块 

时，计数器的数量会-1，知道计数器的数量为0，就释放这个锁。 

**不可中断特性**

一个线程获得锁后，另一个线程想要获得锁，必须处于阻塞或等待状态，如果第一个线程不释放锁，第
二个线程会一直阻塞或等待，不可被中断。

```java
/*
    目标:演示synchronized不可中断
        1.定义一个Runnable
        2.在Runnable定义同步代码块
        3.先开启一个线程来执行同步代码块,保证不退出同步代码块
        4.后开启一个线程来执行同步代码块(阻塞状态)
        5.停止第二个线程
 */
public class Demo02_Uninterruptible {
    private static Object obj = new Object();
    public static void main(String[] args) throws InterruptedException {
        // 1.定义一个Runnable
        Runnable run = () -> {
            // 2.在Runnable定义同步代码块
            synchronized (obj) {
                String name = Thread.currentThread().getName();
                System.out.println(name + "进入同步代码块");
                // 保证不退出同步代码块
                try {
                    Thread.sleep(888888);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        // 3.先开启一个线程来执行同步代码块
        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        // 4.后开启一个线程来执行同步代码块(阻塞状态)
        Thread t2 = new Thread(run);
        t2.start();

        // 5.停止第二个线程
        System.out.println("停止线程前");
        t2.interrupt();
        System.out.println("停止线程后");

        System.out.println(t1.getState());
        System.out.println(t2.getState());
    }
}
```

执行结果：

```java
Thread-0进入同步代码块
停止线程前
停止线程后
TIMED_WAITING
BLOCKED      // 阻塞状态，并没有停止
```

**ReentrantLock可中断**

```java
/*
    目标:演示Lock不可中断和可中断
 */
public class Demo03_Interruptible {
    private static Lock lock = new ReentrantLock();
    public static void main(String[] args) throws InterruptedException {
        // test01();
        test02();
    }

    // 演示Lock可中断
    public static void test02() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            boolean b = false;
            try {
                b = lock.tryLock(3, TimeUnit.SECONDS);
                if (b) {
                    System.out.println(name + "获得锁,进入锁执行");
                    Thread.sleep(88888);
                } else {
                    System.out.println(name + "在指定时间没有得到锁做其他操作");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (b) {
                    lock.unlock();
                    System.out.println(name + "释放锁");
                }
            }
        };

        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(run);
        t2.start();

        // System.out.println("停止t2线程前");
        // t2.interrupt();
        // System.out.println("停止t2线程后");
        //
        // Thread.sleep(1000);
        // System.out.println(t1.getState());
        // System.out.println(t2.getState());
    }

    // 演示Lock不可中断
    public static void test01() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            try {
                lock.lock();
                System.out.println(name + "获得锁,进入锁执行");
                Thread.sleep(88888);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println(name + "释放锁");
            }
        };

        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
        Thread t2 = new Thread(run);
        t2.start();

        System.out.println("停止t2线程前");
        t2.interrupt();
        System.out.println("停止t2线程后");

        Thread.sleep(1000);
        System.out.println(t1.getState());
        System.out.println(t2.getState());
    }
}
```

**小结** 

不可中断是指，当一个线程获得锁后，另一个线程一直处于阻塞或等待状态，前一个线程不释放锁，后
一个线程会一直阻塞或等待，不可被中断。

synchronized属于不可被中断 Lock的lock方法是不可中断的 Lock的tryLock方法是可中断的 

### synchronized的实现原理

javap反编译

```java
...
public class Demo01 {
    private static Object obj = new Object();

    public static void main(String[] args) {
        synchronized (obj) {
            System.out.println("1");
        }
    }

    public synchronized void test() {
        System.out.println("a");
    }
}
```

反编译： `javap -p -v -c Demo01.class `

```java
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // Field obj:Ljava/lang/Object;
         3: dup
         4: astore_1
         5: monitorenter
         6: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         9: ldc           #4                  // String 1
        11: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        14: aload_1
        15: monitorexit
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             6    16    19   any
            19    22    19   any
      LineNumberTable:
        line 7: 0
        line 8: 6
        line 9: 14
        line 10: 24
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      25     0  args   [Ljava/lang/String;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  public synchronized void test();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String a
         5: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 13: 0
        line 14: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/itheima/demo04_synchronized_monitor/Demo01;
...
}
```

#### monitorenter

每一个对象都会和一个监视器monitor关联。监视器被占用时会被锁住，其他线程无法来获 取该monitor。 当JVM执行某个线程的某个方法内部的monitorenter时，它会尝试去获取当前对象对应 的monitor的所有权。其过程如下: 

1. 若monior的进入数为0，线程可以进入monitor，并将monitor的进入数置为1。当前线程成为 monitor的owner(所有者) 

2. 若线程已拥有monitor的所有权，允许它重入monitor，则进入monitor的进入数加1 

3. 若其他线程已经占有monitor的所有权，那么当前尝试获取monitor的所有权的线程会被阻塞，直 

   到monitor的进入数变为0，才能重新尝试获取monitor的所有权。 

**小结** 

synchronized的锁对象会关联一个monitor,这个monitor不是我们主动创建的,是JVM的线程执行到这个 同步代码块,发现锁对象没有monitor就会创建monitor,monitor内部有两个重要的成员变量owner:拥有 这把锁的线程,recursions会记录线程拥有锁的次数,当一个线程拥有monitor后其他线程只能等待 

#### monitorexit

翻译过来: 

1. 能执行monitorexit指令的线程一定是拥有当前对象的monitor的所有权的线程。 

2. 执行monitorexit时会将monitor的进入数减1。当monitor的进入数减为0时，当前线程退出 

   monitor，不再拥有monitor的所有权，此时其他被这个monitor阻塞的线程可以尝试去获取这个 monitor的所有权 

monitorexit释放锁。 monitorexit插入在方法结束处和异常处，JVM保证每个monitorenter必须有对应的monitorexit。 

#### synchronized修饰同步方法

可以看到同步方法在反汇编后，会增加 ACC_SYNCHRONIZED 修饰。会隐式调用monitorenter和
monitorexit。在执行同步方法前会调用monitorenter，在执行完同步方法后会调用monitorexit。

**小结** 

通过javap反汇编我们看到synchronized使用编程了monitorentor和monitorexit两个指令.每个锁对象 都会关联一个monitor(监视器,它才是真正的锁对象),它内部有两个重要的成员变量owner会保存获得锁 的线程,recursions会保存线程获得锁的次数,当执行到monitorexit时,recursions会-1,当计数器减到0时 这个线程就会释放锁 

#### 日常问题

**synchroznied出现异常会释放锁吗**

答：会释放锁

**synchronized与Lock的区别**

1. synchronized是关键字，而Lock是一个接口。
2. synchronized会自动释放锁，而Lock必须手动释放锁。
3. synchronized是不可中断的，Lock可以中断也可以不中断。
4. 通过Lock可以知道线程有没有拿到锁，而synchronized不能。
5. synchronized能锁住方法和代码块，而Lock只能锁住代码块。
6. Lock可以使用读锁提高多线程读效率。
7. synchronized是非公平锁，ReentrantLock可以控制是否是公平锁。

#### 深入JVM源码

**JVM源码下载**

http://openjdk.java.net/ --> Mercurial --> jdk8 --> hotspot --> zip 

**IDE(Clion )下载 **

https://www.jetbrains.com/ 

**monitor监视器锁**

可以看出无论是synchronized代码块还是synchronized方法，其线程安全的语义实现最终依赖一个叫 

monitor的东西，那么这个神秘的东西是什么呢?下面让我们来详细介绍一下。 

在HotSpot虚拟机中，monitor是由ObjectMonitor实现的。其源码是用c++来实现的，位于HotSpot虚
拟机源码ObjectMonitor.hpp文件中(src/share/vm/runtime/objectMonitor.hpp)。ObjectMonitor主
要数据结构如下:

```java
ObjectMonitor() {
    _header      = NULL;
    _count       = 0
    _waiters     = 0
    _recursions  = 0; // 线程的重入次数
    _object      = NULL; //存储该monitor的对象
    _owner       = NULL; // 标识拥有该monitor的线程
    _WaitSet     = NULL; // 处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock = 0; 
    _Responsible = NULL;
    _succ        = NULL;
    _cxq         = NULL; // 多线程竞争锁时的单向列表
    FreeNext     = NULL; 
    _EntryList   = NULL; // 处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq    = 0;
    _SpinClock   = 0;
    OwnerIsThread= 0;
}
```

1. _owner:初始时为NULL。当有线程占有该monitor时，owner标记为该线程的唯一标识。当线程 释放monitor时，owner又恢复为NULL。owner是一个临界资源，JVM是通过CAS操作来保证其线 程安全的。 

2. _cxq:竞争队列，所有请求锁的线程首先会被放在这个队列中(单向链接)。_cxq是一个临界资 源，JVM通过CAS原子指令来修改_cxq队列。修改前_cxq的旧值填入了node的next字段，_cxq指 向新值(新线程)。因此_cxq是一个后进先出的stack(栈)。 

3. _EntryList:_cxq队列中有资格成为候选资源的线程会被移动到该队列中。 
4. _WaitSet:因为调用wait方法而被阻塞的线程会被放在该队列中。 

每一个Java对象都可以与一个监视器monitor关联，我们可以把它理解成为一把锁，当一个线程想要执 行一段被synchronized圈起来的同步方法或者代码块时，该线程得先获取到synchronized修饰的对象 对应的monitor。 

我们的Java代码里不会显示地去创造这么一个monitor对象，我们也无需创建，事实上可以这么理解: monitor并不是随着对象创建而创建的。我们是通过synchronized修饰符告诉JVM需要为我们的某个对 象创建关联的monitor对象。每个线程都存在两个ObjectMonitor对象列表，分别为free和used列表。 同时JVM中也维护着global locklist。当线程需要ObjectMonitor对象时，首先从线程自身的free表中申 请，若存在则使用，若不存在则从global list中申请。 

ObjectMonitor的数据结构中包含:_owner、_WaitSet和_EntryList，它们之间的关系转换可以用下图
表示:

![juc_monitor01](/Users/admin/Desktop/note/images/JUC/juc_monitor01.png)

**monitor竞争**

1. 执行monitorenter时，会调用InterpreterRuntime.cpp

(位于:src/share/vm/interpreter/interpreterRuntime.cpp) 的 InterpreterRuntime::monitorenter函
数。具体代码可参见HotSpot源码

```java
IRT_ENTRY_NO_ASYNC(void， InterpreterRuntime::monitorenter(JavaThread* thread， BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  if (PrintBiasedLockingStatistics) {
    Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
}
Handle h_obj(thread， elem->obj()); assert(Universe::heap()->is_in_reserved_or_null(h_obj())，
         "must be NULL or an object");
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
ObjectSynchronizer::fast_enter(h_obj， elem->lock()， true， CHECK); } else {
ObjectSynchronizer::slow_enter(h_obj， elem->lock()， CHECK); }
assert(Universe::heap()->is_in_reserved_or_null(elem->obj())， "must be NULL or an object");
```

2.对于重量级锁，monitorenter函数中会调用 ObjectSynchronizer::slow_enter

3.最终调用 ObjectMonitor::enter(位于:src/share/vm/runtime/objectMonitor.cpp)，源码如下:

```java
void ATTR ObjectMonitor::enter(TRAPS) {
  // The following code is ordered to check the most common cases first
  // and to reduce RTS->RTO cache line upgrades on SPARC and IA32 processors.
  Thread * const Self = THREAD ;
  void * cur ;
// 通过CAS操作尝试把monitor的_owner字段设置为当前线程 cur = Atomic::cmpxchg_ptr (Self， &_owner， NULL) ; if (cur == NULL) {
     // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
assert (_recursions == 0 ， "invariant") ; assert (_owner == Self， "invariant") ; // CONSIDER: set or assert OwnerIsThread == 1 return ;
}
// 线程重入，recursions++ if (cur == Self) {
     // TODO-FIXME: check for integer overflow!  BUGID 6557169.
     _recursions ++ ;
return ; }
// 如果当前线程是第一次进入该monitor，设置_recursions为1，_owner为当前线程
if (Self->is_lock_owned ((address)cur)) {
assert (_recursions == 0， "internal state error");
_recursions = 1 ;
// Commute owner from a thread-specific on-stack BasicLockObject address to // a full-fledged "Thread *".
_owner = Self ;
OwnerIsThread = 1 ;
return ;
}
// 省略一些代码 for (;;) {
    jt->set_suspend_equivalent();
    // cleared by handle_special_suspend_equivalent_condition()
    // or java_suspend_self()
// 如果获取锁失败，则等待锁的释放; EnterI (THREAD) ;
    if (!ExitSuspendEquivalent(jt)) break ;
//
// We have acquired the contended monitor， but while we were // waiting another thread suspended us. We don't want to enter // the monitor while suspended because that would surprise the // thread that suspended us.
//
        _recursions = 0 ;
    _succ = NULL ;
exit (false， Self) ; jt->java_suspend_self();
}
  Self->set_current_pending_monitor(NULL);
}
```

此处省略锁的自旋优化等操作，统一放在后面synchronzied优化中说。

以上代码的具体流程概括如下:

1. 通过CAS尝试把monitor的owner字段设置为当前线程。
2. 如果设置之前的owner指向当前线程，说明当前线程再次进入monitor，即重入锁，执行 recursions ++ ，记录重入的次数。
3. 如果当前线程是第一次进入该monitor，设置recursions为1，_owner为当前线程，该线程成功获 得锁并返回。
4. 如果获取锁失败，则等待锁的释放。 

**monitor等待**

竞争失败等待调用的是ObjectMonitor对象的EnterI方法(位于:src/share/vm/runtime/objectMonitor.cpp)，源码如下所示:

```java
void ATTR ObjectMonitor::EnterI (TRAPS) {
    Thread * Self = THREAD ;
    // Try the lock - TATAS
    if (TryLock (Self) > 0) {
        assert (_succ != Self
        assert (_owner == Self
        assert (_Responsible != Self
        return ;
    }
    if (TrySpin (Self) > 0) {
        assert (_owner == Self
        assert (_succ != Self
        assert (_Responsible != Self  , "invariant") ;
        return ;
    }
     // 省略部分代码
    // 当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ; ObjectWaiter node(Self) ;
    Self->_ParkEvent->reset() ;
    node._prev = (ObjectWaiter *) 0xBAD ;
    node.TState  = ObjectWaiter::TS_CXQ ;
    // 通过CAS把node节点push到_cxq列表中 ObjectWaiter * nxt ;
    for (;;) {
    node._next = nxt = _cxq ;
    if (Atomic::cmpxchg_ptr (&node， &_cxq， nxt) == nxt) break ;
        // Interference - the CAS failed because _cxq changed.  Just retry.
        // As an optional optimization we retry the lock.
        if (TryLock (Self) > 0) {
            assert (_succ != Self
            assert (_owner == Self
            assert (_Responsible != Self
            return ;
    } }   
    // 省略部分代码 
    for (;;) { 
        // 线程在被挂起前做一下挣扎，看能不能获取到锁 if (TryLock (Self) > 0) break ;
        assert (_owner != Self， "invariant") ;
        if ((SyncFlags & 2) && _Responsible == NULL) { Atomic::cmpxchg_ptr (Self， &_Responsible， NULL) ;
        }
        // park self
        if (_Responsible == Self || (SyncFlags & 1)) {
        TEVENT (Inflated enter - park TIMED) ; Self->_ParkEvent->park ((jlong) RecheckInterval) ;
        // Increase the RecheckInterval， but clamp the value. RecheckInterval *= 8 ;
            if (RecheckInterval > 1000) RecheckInterval = 1000 ;
        } else {
        TEVENT (Inflated enter - park UNTIMED) ;
        // 通过park将当前线程挂起，等待被唤醒
            Self->_ParkEvent->park() ;
        }
        if (TryLock(Self) > 0) break ;
        // 省略部分代码 }
        // 省略部分代码 }    
```

当该线程被唤醒时，会从挂起的点继续执行，通过 ObjectMonitor::TryLock 尝试获取锁，TryLock方
法实现如下:

```java
int ObjectMonitor::TryLock (Thread * Self) {
   for (;;) {
        void * own = _owner ;
        if (own != NULL) return 0 ;
        if (Atomic::cmpxchg_ptr (Self， &_owner， NULL) == NULL) {
                 // Either guarantee _recursions == 0 or set _recursions = 0.
        assert (_recursions == 0， "invariant") ;
        assert (_owner == Self， "invariant") ;
        // CONSIDER: set or assert that OwnerIsThread == 1 return 1 ;
        }
        // The lock had been free momentarily， but we lost the race to the lock. // Interference -- the CAS failed.
        // We can either return -1 or retry.
        // Retry doesn't make as much sense because the lock was just acquired. if (true) return -1 ;
	} }
```

以上代码的具体流程概括如下:

1. 当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ。 
2. 在for循环中，通过CAS把node节点push到_cxq列表中，同一时刻可能有多个线程把自己的node 节点push到_cxq列表中。 
3. node节点push到_cxq列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当   前线程挂起，等待被唤醒。
4. 当该线程被唤醒时，会从挂起的点继续执行，通过 ObjectMonitor::TryLock 尝试获取锁。 

**monitor释放**

当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在
HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于
ObjectMonitor的exit方法中。(位于:src/share/vm/runtime/objectMonitor.cpp)，源码如下所
示:

```java
void ATTR ObjectMonitor::exit(bool not_suspended， TRAPS) { Thread * Self = THREAD ;
    // 省略部分代码
    if (_recursions != 0) {
    _recursions--;        // this is simple recursive enter
    TEVENT (Inflated exit - recursive) ;
    return ; }
    // 省略部分代码 ObjectWaiter * w = NULL ; int QMode = Knob_QMode ;
    // qmode = 2:直接绕过EntryList队列，从cxq队列中获取线程用于竞争锁 if (QMode == 2 && _cxq != NULL) {
    w = _cxq ;
    assert (w != NULL， "invariant") ;
    assert (w->TState == ObjectWaiter::TS_CXQ， "Invariant") ; ExitEpilog (Self， w) ;
    return ;
    }
// qmode =3:cxq队列插入EntryList尾部; if (QMode == 3 && _cxq != NULL) {
     w = _cxq ;
     for (;;) {
        assert (w != NULL， "Invariant") ;
        ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL，
        ， "invariant") ;
        &_cxq， w) ; w=u;
            if (u == w) break ;
        }
        assert (w != NULL
                ObjectWaiter * q = NULL ;
                ObjectWaiter * p ;
                for (p = w ; p != NULL ; p = p->_next) {
        guarantee (p->TState == ObjectWaiter::TS_CXQ， "Invariant") ; p->TState = ObjectWaiter::TS_ENTER ;
        p->_prev = q ;
        q=p;
        }
                ObjectWaiter * Tail ;
                for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail =
        Tail->_next) ;
                if (Tail == NULL) {
                    _EntryList = w ;
                } else {
                    Tail->_next = w ;
                    w->_prev = Tail ;
        } }
        // qmode =4:cxq队列插入到_EntryList头部 if (QMode == 4 && _cxq != NULL) {
         w = _cxq ;
         for (;;) {
            assert (w != NULL， "Invariant") ;
            ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL，
            &_cxq， w) ; w=u;
            }
        assert (w != NULL ， "invariant") ;
            ObjectWaiter * q = NULL ;
            ObjectWaiter * p ;
            for (p = w ; p != NULL ; p = p->_next) {
        guarantee (p->TState == ObjectWaiter::TS_CXQ， "Invariant") ; p->TState = ObjectWaiter::TS_ENTER ;
        p->_prev = q ;
        q=p;
        }
            if (_EntryList != NULL) {
                q->_next = _EntryList ;
                _EntryList->_prev = q ;
        }
            _EntryList = w ;
        }
        w = _EntryList  ;
        if (w != NULL) {
        assert (w->TState == ObjectWaiter::TS_ENTER， "invariant") ; ExitEpilog (Self， w) ;
        return ;
        } 
                w = _cxq ;
        if (w == NULL) continue ;
        for (;;) {
        assert (w != NULL， "Invariant") ;
        ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL， &_cxq，
            if (u == w) break ;
        w=u; }
        TEVENT (Inflated exit - drain cxq into EntryList) ; assert (w != NULL ， "invariant") ;
        assert (_EntryList == NULL ， "invariant") ;
        if (QMode == 1) {
            // QMode == 1 : drain cxq to EntryList， reversing order // We also reverse the order of the list.
            ObjectWaiter * s = NULL ;
            ObjectWaiter * t = w ;
            ObjectWaiter * u = NULL ;
            while (t != NULL) {
                guarantee (t->TState == ObjectWaiter::TS_CXQ， "invariant") ; t->TState = ObjectWaiter::TS_ENTER ;
                u = t->_next ;
                t->_prev = u ;
                t->_next = s ; s = t;
                t=u;
            }
            _EntryList = s ;
            assert (s != NULL， "invariant") ;     
            } else {
        // QMode == 0 or QMode == 2
        _EntryList = w ;
        ObjectWaiter * q = NULL ;
        ObjectWaiter * p ;
        for (p = w ; p != NULL ; p = p->_next) {
        guarantee (p->TState == ObjectWaiter::TS_CXQ， "Invariant") ; p->TState = ObjectWaiter::TS_ENTER ;
        p->_prev = q ;
        q=p;
        } }
        w = _EntryList  ;
     if (w != NULL) {
    guarantee (w->TState == ObjectWaiter::TS_ENTER， "invariant") ; ExitEpilog (Self， w) ;
    return ; }}}                                                       
```

1. 退出同步代码块时会让_recursions减1，当_recursions的值减为0时，说明线程释放了锁。 
2.  根据不同的策略(由QMode指定)，从cxq或EntryList中获取头节点，通过 ObjectMonitor::ExitEpilog 方法唤醒该节点封装的线程，唤醒操作最终由unpark完成，实现 如下: 

```java
    void ObjectMonitor::ExitEpilog (Thread * Self， ObjectWaiter * Wakee) { assert (_owner == Self， "invariant") ;
       _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
       ParkEvent * Trigger = Wakee->_event ;
    Wakee = NULL ;
       // Drop the lock
    OrderAccess::release_store_ptr (&_owner， NULL) ;
       OrderAccess::fence() ;
    unpark()
       if (SafepointSynchronize::do_call_back()) {
          TEVENT (unpark before SAFEPOINT) ;
    // ST _owner vs LD in
    }
    DTRACE_MONITOR_PROBE(contended__exit， this， object()， Self);
    Trigger->unpark() ; // 唤醒之前被pack()挂起的线程.
       // Maintain stats and report events to JVMTI
       if (ObjectMonitor::_sync_Parks != NULL) {
          ObjectMonitor::_sync_Parks->inc() ;
    } }
```

被唤醒的线程，会回到void ATTR ObjectNonitor::ENTER(TRAPS) 的第600行，继续执行monitor的竞争。

```java
// park self
if (_Responsible == Self || (SyncFlags & 1)) {
TEVENT (Inflated enter - park TIMED) ; Self->_ParkEvent->park ((jlong) RecheckInterval) ;
// Increase the RecheckInterval， but clamp the value. RecheckInterval *= 8 ;
    if (RecheckInterval > 1000) RecheckInterval = 1000 ;
} else {
    TEVENT (Inflated enter - park UNTIMED) ;
    Self->_ParkEvent->park() ;
}
if (TryLock(Self) > 0) break ;
```

monitor是重量级锁

可以看到ObjectMonitor的函数调用中会涉及到Atomic::cmpxchg_ptr，Atomic::inc_ptr等内核函数， 执行同步代码块，没有竞争到锁的对象会park()被挂起，竞争到锁的线程会unpark()唤醒。这个时候就 会存在操作系统用户态和内核态的转换，这种切换会消耗大量的系统资源。所以synchronized是Java语 言中是一个重量级(Heavyweight)的操作。 

用户态和和内核态是什么东西呢?要想了解用户态和内核态还需要先了解一下Linux系统的体系架构: 

![juc_monitor02](/Users/admin/Desktop/note/images/JUC/juc_monitor02.png)

从上图可以看出，Linux操作系统的体系架构分为:用户空间(应用程序的活动空间)和内核。 内核:本质上可以理解为一种软件，控制计算机的硬件资源，并提供上层应用程序运行的环境。 

用户空间:上层应用程序活动的空间。应用程序的执行必须依托于内核提供的资源，包括CPU资源、存 储资源、I/O资源等。 

系统调用:为了使上层应用能够访问到这些资源，内核必须为上层应用提供访问的接口:即系统调用。所有进程初始都运行于用户空间，此时即为用户运行状态(简称:用户态);但是当它调用系统调用执行某些操作时，例如 I/O调用，此时需要陷入内核中运行，我们就称进程处于内核运行态(或简称为内 核态)。 系统调用的过程可以简单理解为: 

1. 用户态程序将一些数据值放在寄存器中， 或者使用参数创建一个堆栈， 以此表明需要操作系统提 供的服务。 
2. 用户态程序执行系统调用。 
3. CPU切换到内核态，并跳到位于内存指定位置的指令。 
4. 系统调用处理器(system call handler)会读取程序放入内存的数据参数，并执行程序请求的服务。 
5. 系统调用完成后，操作系统会重置CPU为用户态并返回系统调用的结果。 

由此可见用户态切换至内核态需要传递许多变量，同时内核还需要保护好用户态在切换时的一些寄存器
值、变量等，以备内核态切换回用户态。这种切换就带来了大量的系统资源消耗，这就是在
synchronized未优化之前，效率低的原因。

### JDK6 synchronized优化

#### CAS

CAS的全成是: Compare And Swap(比较相同再交换)。是现代CPU广泛支持的一种对内存中的共享数
据进行操作的一种特殊指令。

CAS的作用:CAS可以将比较和交换转换为原子操作，这个原子操作直接由CPU保证。CAS可以保证共
享变量赋值时的原子操作。CAS操作依赖3个值:内存中的值V，旧的预估值X，要修改的新值B，如果旧
的预估值X等于内存中的值V，就将新的值B保存到内存中。

**CAS和volatile实现无锁并发**

```java
/*
    目标:演示原子性问题
        1.定义一个共享变量number
        2.对number进行1000的++操作
        3.使用5个线程来进行
 */
public class Demo01 {
    // 1.定义一个共享变量number
    private static AtomicInteger atomicInteger = new AtomicInteger();
    public static void main(String[] args) throws InterruptedException {
        // 2.对number进行1000的++操作
        Runnable increment = () -> {
            for (int i = 0; i < 1000; i++) {
                atomicInteger.incrementAndGet(); // 变量赋值的原子性
            }
        };

        List<Thread> list = new ArrayList<>();
        // 3.使用5个线程来进行
        for (int i = 0; i < 5; i++) {
            Thread t = new Thread(increment);
            t.start();
            list.add(t);
        }

        for (Thread t : list) {
            t.join();
        }

        System.out.println("atomicInteger = " + atomicInteger.get());
    }
}
```

通过刚才AtomicInteger的源码我们可以看到，Unsafe类提供了原子操作。

**Unsafe类介绍**

Unsafe类使Java拥有了像C语言的指针一样操作内存空间的能力，同时也带来了指针的问题。过度的使
用Unsafe类会使得出错的几率变大，因此Java官方并不建议使用的，官方文档也几乎没有。Unsafe对
象不能直接调用，只能通过反射获得。

![juc_monitor03](/Users/admin/Desktop/note/images/JUC/juc_monitor03.png)

![juc_monitor04](/Users/admin/Desktop/note/images/JUC/juc_monitor04.png)

**乐观锁和悲观锁**

**悲观锁**从悲观的角度出发:

总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这 样别人想拿这个数据就会阻塞。因此synchronized我们也将其称之为悲观锁。JDK中的ReentrantLock 也是一种悲观锁。性能较差! 

**乐观锁**从乐观的角度出发: 

总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，就算改了也没关系，再重试即可。所
以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去修改这个数据，如何没有人修改则更
新，如果有人修改则重试。

CAS这种机制我们也可以将其称之为乐观锁。综合性能较好!

>CAS获取共享变量时，为了保证该变量的可见性，需要使用volatile修饰。结合CAS和volatile可以 
>
>实现无锁并发，适用于竞争不激烈、多核 CPU 的场景下。
>
>1. 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一。 
>
>2. 但如果竞争激烈，可以想到重试必然频繁发生，反而效率会受影响。 

**小结**

CAS的作用? Compare And Swap，CAS可以将比较和交换转换为原子操作，这个原子操作直接由处理 器保证。 

CAS的原理?CAS需要3个值:内存地址V，旧的预期值A，要修改的新值B，如果内存地址V和旧的预期值 A相等就修改内存地址值为B 

#### java 对象的布局

在JVM中，对象在内存中的布局分为三块区域:对象头、实例数据和对齐填充。如下图所示:

![juc_monitor05](/Users/admin/Desktop/note/images/JUC/juc_monitor05.png)

一个Java对象在JVM中是由一个对应角色的`oop`对象来描述的，

比如`instanceOopDesc`用来描述普通实例对象，`arrayOopDesc`用来描述数组对象，而这些类型的oop对象均是继承自`oopDesc`。

```java
class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  // 对象头  
  volatile markOop _mark;
  // 元数据
  union _metadata {
    // 对应的Klass对象  
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
```

oopDesc主要包含两部分，一部分是`_mark`，一部分是`_metadata`，

- `_mark` _mark是一个`markOop`实例，它描述了一个对象的头信息，用于存储对象的运行时记录信息，如哈希值、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等：
- `_metadata` 包含一个普通`_klass`和一个压缩后的`_compressed_klass`，

对象头的位格式：

```java
32 bits:
  --------
             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
             size:32 ------------------------------------------>| (CMS free block)
             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
 
  64 bits:
  --------
  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
  size:64 ----------------------------------------------------->| (CMS free block)
 
  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```

![juc_lock01](/Users/admin/Desktop/note/images/JUC/juc_lock01.png)

```shell
hash： 保存对象的哈希码
age： 保存对象的分代年龄
biased_lock： 偏向锁标识位
lock： 锁状态标识位
JavaThread：* 保存持有偏向锁的线程ID
epoch： 保存偏向时间戳

markOop中不同的锁标识位，代表着不同的锁状态：
```

![juc_lock01](/Users/admin/Desktop/note/images/JUC/juc_lock02.png)

**klass pointer**

这一部分用于存储对象的类型指针，该指针指向它的类元数据，JVM通过这个指针确定对象是哪个类的 实例。该指针的位长度为JVM的一个字大小，即32位的JVM为32位，64位的JVM为64位。 如果应用的对 象过多，使用64位的指针将浪费大量内存，统计而言，64位的JVM将会比32位的JVM多耗费50%的内 存。为了节约内存可以使用选项 -XX:+UseCompressedOops 开启指针压缩，其中，oop即ordinary object pointer普通对象指针。开启该选项后，下列指针将压缩至32位: 

1. 每个Class的属性指针(即静态变量) 
2. 每个对象的属性指针(即对象变量) 
3.  普通对象数组的每个元素指针 

当然，也不是所有的指针都会压缩，一些特殊类型的指针JVM不会优化，比如指向PermGen的Class对 象指针( JDK8中指向元空间的Class对象指针)、本地变量、堆栈元素、入参、返回值和NULL指针等。

对象头 = Mark Word + 类型指针(未开启指针压缩的情况下)
在32位系统中，Mark Word = 4 bytes，类型指针 = 4bytes，对象头 = 8 bytes = 64 bits;

**实例对象**

就是类中定义的成员变量。

**对齐填充**

对齐填充并不是必然存在的，也没有什么特别的意义，他仅仅起着占位符的作用，由于HotSpot VM的
自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是8字节的
整数倍。而对象头正好是8字节的倍数，因此，当对象实例数据部分没有对齐时，就需要通过对齐填充
来补全。

**小结**

 Java对象由3部分组成，对象头，实例数据，对齐数据 

对象头分成两部分:Mark World + Klass pointer 

#### 偏向锁

偏向锁是JDK 6中的重要引进，因为HotSpot作者经过研究实践发现，在大多数情况下，锁不仅不存在多 线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低，引进了偏向锁。 

偏向锁的“偏”，就是偏心的“偏”、偏袒的“偏”，它的意思是这个锁会偏向于第一个获得它的线程，会在对 象头存储锁偏向的线程ID，以后该线程进入和退出同步块时只需要检查是否为偏向锁、锁标志位以及 ThreadID即可。 

不过一旦出现多个线程竞争时必须撤销偏向锁，所以撤销偏向锁消耗的性能必须小于之前节省下来的
CAS原子操作的性能消耗，不然就得不偿失了。

在64位系统中，Mark Word = 8 bytes，类型指针 = 8bytes，对象头 = 16 bytes = 128bits;

**偏向锁原理**

当线程第一次访问同步块并获取锁时，偏向锁处理流程如下:

>1. 虚拟机将会把对象头中的标志位设为“01”，即偏向模式。
>
>2. 同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中 ，如果CAS操作 
>
>   成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何
>       同步操作，偏向锁的效率高。

持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作，偏向锁
的效率高。

**偏向锁的撤销**

1. 偏向锁的撤销动作必须等待全局安全点
2. 暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态
3. 撤销偏向锁，恢复到无锁(标志位为 01)或轻量级锁(标志位为 00)的状态

偏向锁在Java 6之后是默认启用的，但在应用程序启动几秒钟之后才激活，可以使用 -
XX:BiasedLockingStartupDelay=0 参数关闭延迟，如果确定应用程序中所有锁通常情况下处于竞争
状态，可以通过 XX:-UseBiasedLocking=false 参数关闭偏向锁。

**偏向锁好处**

偏向锁是在只有一个线程执行同步块时进一步提高性能，适用于一个线程反复获得同一锁的情况。偏向
锁可以提高带有同步但无竞争的程序性能。
它同样是一个带有效益权衡性质的优化，也就是说，它并不一定总是对程序运行有利，如果程序中大多
数的锁总是被多个不同的线程访问比如线程池，那偏向模式就是多余的。

在JDK5中偏向锁默认是关闭的，而到了JDK6中偏向锁已经默认开启。但在应用程序启动几秒钟之后才 激活，可以使用 -XX:BiasedLockingStartupDelay=0 参数关闭延迟，如果确定应用程序中所有锁通常 情况下处于竞争状态，可以通过 XX:-UseBiasedLocking=false 参数关闭偏向锁。 

**小结**

偏向锁的原理是什么?

```java
当锁对象第一次被线程获取的时候，虚拟机将会把对象头中的标志位设为“01”，即偏向模式。同时使用CAS操 作把获取到这个锁的线程的ID记录在对象的Mark Word之中 ，如果CAS操作成功，持有偏向锁的线程以后每 次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作，偏向锁的效率高。
```

偏向锁的好处是什么?

```
偏向锁是在只有一个线程执行同步块时进一步提高性能，适用于一个线程反复获得同一锁的情况。偏向锁可以
提高带有同步但无竞争的程序性能。
```

#### 轻量级锁

轻量级锁是JDK 6之中加入的新型锁机制，它名字中的“轻量级”是相对于使用monitor的传统锁而言的， 

因此传统的锁机制就称为“重量级”锁。首先需要强调一点的是，轻量级锁并不是用来代替重量级锁的。 

```
引入轻量级锁的目的:在多线程交替执行同步块的情况下，尽量避免重量级锁引起的性能消耗，但是如
果多个线程在同一时刻进入临界区，会导致轻量级锁膨胀升级重量级锁，所以轻量级锁的出现并非是要
替代重量级锁。
```

**轻量级锁原理**

当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，其步
骤如下: 获取锁

1. 判断当前对象是否处于无锁状态(hashcode、0、01)，如果是，则JVM首先将在当前线程的栈帧 中建立一个名为锁记录(Lock Record)的空间，用于存储锁对象目前的Mark Word的拷贝(官方 把这份拷贝加了一个Displaced前缀，即Displaced Mark Word)，将对象的Mark Word复制到栈 帧中的Lock Record中，将Lock Reocrd中的owner指向当前对象。 
2. JVM利用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，如果成功表示竞争到 锁，则将锁标志位变成00，执行同步操作。 
3. 如果失败则判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持 有当前对象的锁，则直接执行同步代码块;否则只能说明该锁对象已经被其他线程抢占了，这时轻 量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态。 

![juc_monitor06](/Users/admin/Desktop/note/images/JUC/juc_monitor06.png)

**轻量级锁的释放**

轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下:

1. 取出在获取轻量级锁保存在Displaced Mark Word中的数据。
2. 用CAS操作将取出的数据替换当前对象的Mark Word中，如果成功，则说明释放锁成功。
3. 如果CAS操作替换失败，说明有其他线程尝试获取该锁，则需要将轻量级锁需要膨胀升级为重量级
   锁。

对于轻量级锁，其性能提升的依据是“对于绝大部分的锁，在整个生命周期内都是不会存在竞争的”，如
果打破这个依据则除了互斥的开销外，还有额外的CAS操作，因此在有多线程竞争的情况下，轻量级锁
比重量级锁更慢。

**轻量级锁好处**

在多线程交替执行同步块的情况下，可以避免重量级锁引起的性能消耗。

**小结**

轻量级锁的原理是什么?

```java
将对象的Mark Word复制到栈帧中的Lock Recod中。Mark Word更新为指向Lock Record的指针。
```

轻量级锁好处是什么?

```java
在多线程交替执行同步块的情况下，可以避免重量级锁引起的性能消耗。								
```

#### 锁状态变化示例

对象头查看工具：JOL

![juc_lock01](/Users/admin/Desktop/note/images/JUC/juc_lock03.png)

偏向锁设置：

```java
启用参数: 
-XX:+UseBiasedLocking
关闭延迟: 
-XX:BiasedLockingStartupDelay=0 
禁用参数: 
-XX:-UseBiasedLocking
```

代码准备：

```java
// 引入jol包
<dependency>
	<groupId>org.openjdk.jol</groupId>
	<artifactId>jol-core</artifactId>
	<version>0.8</version>
</dependency>

// 关闭偏向锁延迟
idea中配置 VM options 项加： -XX:BiasedLockingStartupDelay=0

// 声明对象
public class L {
}
```

**测试偏向锁**

```java
public class Layout1 {
    static L l = new L();

    public static void main(String[] args) {
        System.out.println("static");
        System.out.println(ClassLayout.parseInstance(l).toPrintable());
        synchronized (l){
            System.out.println("lock ing1");
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
        synchronized (l){
            System.out.println("lock ing2");
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
        System.out.println("end");
    }
}

// 打印结果
static    // 开始
// 未加synchronized前打印对象信息
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           05 c4 00 f8 (00000101 11000100 00000000 11111000) (-134167547)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 锁状态为： 101----表示偏向锁  后面为空表示没有没有偏向任何对象

lock ing1  // 准备加锁
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 30 82 19 (00000101 00110000 10000010 00011001) (427962373)
      4     4        (object header)                           c9 7f 00 00 (11001001 01111111 00000000 00000000) (32713)
      8     4        (object header)                           05 c4 00 f8 (00000101 11000100 00000000 11111000) (-134167547)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 加synchronized锁住L对象实例后打印对象信息。锁状态为： 101----表示偏向锁  427962373:指向当前线程id

lock ing2 // 准备再加synchronized锁住相同实例
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 30 82 19 (00000101 00110000 10000010 00011001) (427962373)
      4     4        (object header)                           c9 7f 00 00 (11001001 01111111 00000000 00000000) (32713)
      8     4        (object header)                           05 c4 00 f8 (00000101 11000100 00000000 11111000) (-134167547)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 再加synchronized锁住L对象实例后打印对象信息。锁状态为： 101----表示偏向锁  427962373:指向当前线程id
end //结束
```

**测试偏向锁升级轻量级锁**

```java
// 测试
public class Layout1 {
    static L l = new L();

    public static void main(String[] args) {
        System.out.println("static");
        synchronized (l){
            System.out.println("lock ing1");
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
        Thread thread1 = new Thread(){
            @Override
            public void run(){
                test();
            }
        };
        thread1.start();
        System.out.println("end");
    }

    public static void  test(){
        synchronized (l){
            System.out.println("lock ing3");
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
    }

}

// 执行结果
static
lock ing1
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 80 00 84 (00000101 10000000 00000000 10000100) (-2080342011)
      4     4        (object header)                           a8 7f 00 00 (10101000 01111111 00000000 00000000) (32680)
      8     4        (object header)                           61 c4 00 f8 (01100001 11000100 00000000 11111000) (-134167455)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 加synchronized锁住L对象实例后打印对象信息。锁状态为： 101----表示偏向锁  -2080342011:指向当前线程id

end
lock ing3
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           c8 08 7d 0f (11001000 00001000 01111101 00001111) (259852488)
      4     4        (object header)                           00 70 00 00 (00000000 01110000 00000000 00000000) (28672)
      8     4        (object header)                           61 c4 00 f8 (01100001 11000100 00000000 11111000) (-134167455)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 另起一子线程 执行test方法，方法内加synchronized锁住L对象实例后打印对象信息。
//锁状态为： 00----表示轻量级锁  259852488:指向子线程id
```

**偏向锁—轻量级锁—重量级锁**

```java
// 测试
public class Layout1 {
    static L l = new L();
    static  int num = 3;

    public static void main(String[] args) {
        System.out.println("static");
        synchronized (l){
            System.out.println("lock ing1");
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
        Thread thread1 = new Thread(){
            @Override
            public void run(){
                test();
                try {
                    sleep(400000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        thread1.start();

        Thread thread2 = new Thread(){
            @Override
            public void run(){
                test();
            }
        };
        thread2.start();
    }

    public static void  test(){
        synchronized (l){
            System.out.println("lock ing" + num++);
            System.out.println(ClassLayout.parseInstance(l).toPrintable());
        }
    }
}

// 测试结果
static
lock ing1
# WARNING: Unable to attach Serviceability Agent. You can try again with escalated privileges. Two options: a) use -Djol.tryWithSudo=true to try with sudo; b) echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 08 01 23 (00000101 00001000 00000001 00100011) (587270149)
      4     4        (object header)                           a9 7f 00 00 (10101001 01111111 00000000 00000000) (32681)
      8     4        (object header)                           bd c4 00 f8 (10111101 11000100 00000000 11111000) (-134167363)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 加synchronized锁住L对象实例后打印对象信息。锁状态为： 101----表示偏向锁  587270149:指向当前线程id

lock ing3
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           7a d3 03 24 (01111010 11010011 00000011 00100100) (604230522)
      4     4        (object header)                           a9 7f 00 00 (10101001 01111111 00000000 00000000) (32681)
      8     4        (object header)                           bd c4 00 f8 (10111101 11000100 00000000 11111000) (-134167363)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
// 另起一子线程 执行test方法，方法内加synchronized锁住L对象实例后打印对象信息。
//锁状态为： 00----表示轻量级锁  604230522:指向子线程id

lock ing4
com.juc.classheader.L object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           7a d3 03 24 (01111010 11010011 00000011 00100100) (604230522)
      4     4        (object header)                           a9 7f 00 00 (10101001 01111111 00000000 00000000) (32681)
      8     4        (object header)                           bd c4 00 f8 (10111101 11000100 00000000 11111000) (-134167363)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes tota
```

#### 锁的优缺点对比

| 锁       | 优点                                                         | 缺点                                           | 适用场景                         |
| -------- | ------------------------------------------------------------ | ---------------------------------------------- | -------------------------------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度                     | 如果始终得不到锁竞争的线程使用自旋会消耗CPU    | 追求响应时间,锁占用时间很短      |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU                              | 线程阻塞，响应时间缓慢                         | 追求吞吐量,锁占用时间较长        |


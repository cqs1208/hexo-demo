---
layout: post
title: java8 Stream流
tags:
- Java8
categories: Java8
description: java8
---
Stream 使用一种类似用 SQL 语句从数据库查询数据的直观方式来提供一种对Java集合运算和表达的高阶抽象。 

<!-- more --> 

​	这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如筛选， 排序，聚合等。元素流在管道中经过中间操作（intermediate operation）的处理，最后由最终操作(terminal operation)得到前面处理的结果。 

### 1 什么是 Stream？

Stream（流）是一个来自数据源的元素队列并支持聚合操作 

- 元素是特定类型的对象，形成一个队列。 Java中的Stream并不会存储元素，而是按需计算。
- 数据源流的来源。可以是集合，数组，I/O channel， 产生器generator等 
- 聚合操作类似SQL语句一样的操作，比如filter, map, reduce, find, match, sorted等。

Stream操作还有两个基础的特征：  

- **Pipelining:** 中间操作都会返回流对象本身。这样多个操作可以串联成一个管道，如同流式风格（fluent style）。 这样做可以对操作进行优化，比如延迟执行(laziness)和短路( short-circuiting)。 
- **内部迭代：**以前对集合遍历都是通过Iterator或者For-Each的方式, 显式的在集合外部进行迭代，这叫做外部迭代。 Stream提供了内部迭代的方式，通过访问者模式(Visitor)实现。 

### 2 Stream流操作API分类

在 Java 8 中, 集合接口有两个方法来生成流：  

- stream() − 为集合创建串行流。 
- parallelStream() − 为集合创建并行流。 

流的操作其实可以分为两类：**处理操作、聚合操作**。 

- 处理操作(中间操作)：诸如filter、map等处理操作将Stream一层一层的进行抽离，返回一个流给下一层使用。

  ```java
  有状态 sorted(),必须等上一步操作完拿到全部元素后才可操作
  无状态 filter(),该操作的元素不受上一步操作的影响 
  ```

- 聚合操作(终端操作)：从最后一次流中生成一个结果给调用方，foreach只做处理不做返回。

  ```java
  短路操作findFirst(),找到一个则返回,也就是break当前的循环
  非短路操作forEach(),遍历全部元素
  ```

 **中间操作:** 中间操作只是一种标记，只有结束操作才会触发实际计算。中间操作又可以分为无状态的(Stateless)和有状态的(Stateful)，无状态中间操作是指元素的处理不受前面元素的影响，而有状态的中间操作必须等到所有元素处理之后才知道最终结果，比如排序是有状态操作，在读取所有元素之前并不能确定排序结果； 

**结束操作:**  结束操作又可以分为短路操作和非短路操作，短路操作是指不用处理全部元素就可以返回结果，比如找到第一个满足条件的元素。之所以要进行如此精细的划分，是因为底层对每一种情况的处理方式不同。

### 3 Stream流水线执行原理分析

问题：

1. 操作是如何记录下来的?
2. 操作是如何叠加的?
3. 叠加完如何执行的?
4. 执行完如何收集结果的?

Stream相关类和接口的继承 关系图：

![stream继承关系图](/images/Java8/Java8_streamInherit.png)

图中Head用于表示第一个Stage，即调用调用诸如Collection.stream()方法产生的Stage，很显然这个Stage里不包含任何操作；StatelessOp和StatefulOp分别表示无状态和有状态的Stage

![stream流水线组织结构示意图](/images/Java8/Java8_stream.png)  

操作记录过程：

- Head记录Stream起始操作 ，
-  StatelessOp记录中间操作，  
-  StatefulOp记录有状态的中间操作 

这三个操作实例化会指向其父类AbstractPipeline,也就是在AbstractPipeline中建立了双向链表。

#### 3.1 记录操作--流源构建分析

 首先看下面一段代码，下面将以这一段代码来进行分析： 

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);
list.stream().filter(i -> i % 2 == 0)
        .map(String::valueOf)
        .forEach(System.out::println);
```

首先`list.stream()`会返回一个Stream对象。我们可以跟进去，看看返回的到底是个什么对象。 

```java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
```

继续跟到`StreamSupport.stream`: 

```java
 public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        return new ReferencePipeline.Head<>(spliterator,
                                            StreamOpFlag.fromCharacteristics(spliterator),
                                            parallel);
    }
```

**head是ReferencePipeline的一个源阶段** 

#### 3.2 记录操作--AbstractPipeline 

​	AbstractPipeline表示流管道的初始部分，封装了流源和零个或多个中间操作。每个 AbstractPipeline对象通常被称为*stage* ，其中每个*stage*都描述流源或中间操作，到这里大家就可以想象出流管道的本质就是双向链表。下面来看看它的几个属性： 

```java
/**官方的定义：反向链接到管道链的头部，流的源头（如果本身是源stage,这里就为self）*/
private final AbstractPipeline sourceStage;
/** 流管道的上游，若是源stage则为null,其实就是双向链表的前指针*/
private final AbstractPipeline previousStage;
/** 流管道的下游，若是最后一个中间操作，则为null*/
private AbstractPipeline nextStage;
/** 深度 */
private int depth;
/** 如果此管道已连接或使用，则为真*/
private boolean linkedOrConsumed;
/** 数据源的Spliterator,只对管道头有效*/
private Spliterator<?> sourceSpliterator;
/** 与sourceSpliterator相反，两者有一个为空 */
private Spliterator<?> sourceSpliterator;
```

构造方法一(ReferencePipeline的源阶段)：

```java
AbstractPipeline(Spliterator<?> source, int sourceFlags, boolean parallel) {
        this.previousStage = null;
        this.sourceSpliterator = source;
        this.sourceStage = this;
        this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;
        this.combinedFlags = (~(sourceOrOpFlags << 1)) & StreamOpFlag.INITIAL_OPS_VALUE;
        this.depth = 0;
        this.parallel = parallel;
    }
```

构造方法二：

```java
AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
        previousStage.linkedOrConsumed = true;
        previousStage.nextStage = this;
        this.previousStage = previousStage;
        this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
        this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
        this.sourceStage = previousStage.sourceStage;
        if (opIsStateful()) sourceStage.sourceAnyStateful = true;
        this.depth = previousStage.depth + 1;
    }
```

#### 3.3 操作叠加--sink

​	stage中记录了每一步操作，并没有执行。但是stage只是保存了当前的操作，并不能确定下一个stage需要何种操作，何种数据 。

​	**sink就是每个操作具体的行为操作,也可以叫做回调** 。Sink用于协调相邻的stage之间的数据调用

​	通过begin end accept方法 以及cancellationRequested短路标志位来控制处理流程,对数据进行管控

| 方法名                          | 作用                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| void begin(long size)           | 开始遍历元素之前调用该方法，通知Sink做好准备。               |
| void end()                      | 所有元素遍历完成之后调用，通知Sink没有更多的元素了。         |
| boolean cancellationRequested() | 是否可以结束操作，可以让短路操作尽早结束。                   |
| void accept(T t)                | 遍历元素时调用，接受一个待处理元素，并对元素进行处理。Stage把自己包含的操作和回调方法封装到该方法里，前一个Stage只需要调用当前Stage.accept(T t)方法就行了 |

​	每个Stage都会将自己的操作封装到一个Sink里，前一个Stage只需调用后一个Stage的accept()方法即可，并不需要知道其内部是如何处理的。当然对于有状态的操作，Sink的begin()和end()方法也是必须实现的。比如Stream.sorted()是一个有状态的中间操作，其对应的Sink.begin()方法可能创建一个存放结果的容器，而accept()方法负责将元素添加到该容器，最后end()负责对容器进行排序。对于短路操作，Sink.cancellationRequested()也是必须实现的，比如Stream.findFirst()是短路操作，只要找到一个元素，cancellationRequested()就应该返回true，以便调用者尽快结束查找。Sink的四个接口方法常常相互协作，共同完成计算任务。实际上Stream API内部实现的的本质，就是如何重载Sink的这四个接口方法。

​	有了Sink对操作的包装，Stage之间的调用问题就解决了，执行时只需要从流水线的head开始对数据源依次调用每个Stage对应的Sink.{begin(), accept(), cancellationRequested(), end()}方法就可以了。一种可能的Sink.accept()方法流程是这样的

```java
void accept(U u){
    1. 使用当前Sink包装的回调函数处理u
    2. 将处理结果传递给流水线下游的Sink
}
```

Sink接口的其他几个方法也是按照这种 **[处理->转发]** 的模型实现。 

Sink.ChainedReference源码：

```java
protected final Sink<? super E_OUT> downstream;

public ChainedReference(Sink<? super E_OUT> downstream) {
    this.downstream = Objects.requireNonNull(downstream);
}
@Override
public void begin(long size) {
    downstream.begin(size);
}
@Override
public void end() {
    downstream.end();
}
@Override
public boolean cancellationRequested() {
    return downstream.cancellationRequested();
}
```

查看Stream.map()源码:

```java
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    ...
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                 StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        @Override /*opWripSink()方法返回由回调函数包装而成Sink*/
        Sink<P_OUT> opWrapSink(int flags, Sink<R> downstream) {
            return new Sink.ChainedReference<P_OUT, R>(downstream) {
                @Override
                public void accept(P_OUT u) {
                    R r = mapper.apply(u);// 1. 使用当前Sink包装的回调函数mapper处理u
                    downstream.accept(r);// 2. 将处理结果传递给流水线下游的Sink
                }
            };
        }
    };
}
```

​	上述代码看似复杂，其实逻辑很简单，就是将回调函数mapper包装到一个Sink当中。由于Stream.map()是一个无状态的中间操作，所以map()方法返回了一个StatelessOp内部类对象（一个新的Stream），调用这个新Stream的opWripSink()方法将得到一个包装了当前回调函数的Sink。

![stage与sink之间的关系1](/images/Java8/Java8_streamSink.jpg)

Stream.sorted()方法将对Stream中的元素进行排序，显然这是一个有状态的中间操作，因为读取所有元素之前是没法得到最终顺序的。抛开模板代码直接进入问题本质，sorted()方法是如何将操作封装成Sink的呢？

```java
// Stream.sort()方法用到的Sink实现
class RefSortingSink<T> extends AbstractRefSortingSink<T> {
    private ArrayList<T> list;// 存放用于排序的元素
    RefSortingSink(Sink<? super T> downstream, Comparator<? super T> comparator) {
        super(downstream, comparator);
    }
    @Override
    public void begin(long size) {
        ...
        // 创建一个存放排序元素的列表
        list = (size >= 0) ? new ArrayList<T>((int) size) : new ArrayList<T>();
    }
    @Override
    public void end() {
        list.sort(comparator);// 只有元素全部接收之后才能开始排序
        downstream.begin(list.size());
        if (!cancellationWasRequested) {// 下游Sink不包含短路操作
            list.forEach(downstream::accept);// 2. 将处理结果传递给流水线下游的Sink
        }
        else {// 下游Sink包含短路操作
            for (T t : list) {// 每次都调用cancellationRequested()询问是否可以结束处理。
                if (downstream.cancellationRequested()) break;
                downstream.accept(t);// 2. 将处理结果传递给流水线下游的Sink
            }
        }
        downstream.end();
        list = null;
    }
    @Override
    public void accept(T t) {
        list.add(t);// 1. 使用当前Sink包装动作处理t，只是简单的将元素添加到中间列表当中
    }
}
```

sorted的end方法中,其依赖上一次操作的结果集,按照调用链来说结果集必须在accept()调用完才会产生.那也就说明sorted操作需要在end中,然后再重新开启调用链.那么就相当于sorted给原有操作断路了一次,然后又重新接上,再次遍历

![stage与sink之间的关系2](/images/Java8/Java8_streamSink2.jpg)

上述代码完美的展现了Sink的四个接口方法是如何协同工作的：

1. 首先beging()方法告诉Sink参与排序的元素个数，方便确定中间结果容器的的大小；
2. 之后通过accept()方法将元素添加到中间结果当中，最终执行时调用者会不断调用该方法，直到遍历所有元素；
3. 最后end()方法告诉Sink所有元素遍历完毕，启动排序步骤，排序完成后将结果传递给下游的Sink；
4. 如果下游的Sink是短路操作，将结果传递给下游时不断询问下游cancellationRequested()是否可以结束处理

#### 3.4 执行操作--终端操作调用链

​	Sink完美封装了Stream每一步操作，并给出了[处理->转发]的模式来叠加操作。这一连串的齿轮已经咬合，就差最后一步拨动齿轮启动执行。是什么启动这一连串的操作呢？也许你已经想到了启动的原始动力就是结束操作(Terminal Operation)，一旦调用某个结束操作，就会触发整个流水线的执行。 

​	结束操作之后不能再有别的操作，所以结束操作不会创建新的流水线阶段(Stage)，直观的说就是流水线的链表不会在往后延伸了。结束操作会创建一个包装了自己操作的Sink，这也是流水线中最后一个Sink，这个Sink只需要处理数据而不需要将结果传递给下游的Sink（因为没有下游）。对于Sink的[处理->转发]模型，结束操作的Sink就是调用链的出口。 

​	我们再来考察一下上游的Sink是如何找到下游Sink的。一种可选的方案是在*PipelineHelper*中设置一个Sink字段，在流水线中找到下游Stage并访问Sink字段即可。但Stream类库的设计者没有这么做，而是设置了一个`Sink AbstractPipeline.opWrapSink(int flags, Sink downstream)`方法来得到Sink，该方法的作用是返回一个新的包含了当前Stage代表的操作以及能够将结果传递给downstream的Sink对象。为什么要产生一个新对象而不是返回一个Sink字段？这是因为使用opWrapSink()可以将当前操作与下游Sink（上文中的downstream参数）结合成新Sink。试想只要从流水线的最后一个Stage开始，不断调用上一个Stage的opWrapSink()方法直到最开始（不包括stage0，因为stage0代表数据源，不包含操作），就可以得到一个代表了流水线上所有操作的Sink，用代码表示就是这样： 

```java
// AbstractPipeline.wrapSink()
// 从下游向上游不断包装Sink。如果最初传入的sink代表结束操作，
// 函数返回时就可以得到一个代表了流水线上所有操作的Sink。
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    ...
    for (AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    return (Sink<P_IN>) sink;
}
```

现在流水线上从开始到结束的所有的操作都被包装到了一个Sink里，执行这个Sink就相当于执行整个流水线，执行Sink的代码如下： 

```java
// AbstractPipeline.copyInto(), 对spliterator代表的数据执行wrappedSink代表的操作。
final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
    ...
    if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
        wrappedSink.begin(spliterator.getExactSizeIfKnown());// 通知开始遍历
        spliterator.forEachRemaining(wrappedSink);// 迭代
        wrappedSink.end();// 通知遍历结束
    }
    ...
}
```

上述代码首先调用wrappedSink.begin()方法告诉Sink数据即将到来，然后调用spliterator.forEachRemaining()方法对数据进行迭代（Spliterator是容器的一种迭代器，[参阅](https://github.com/CarpenterLee/JavaLambdaInternals/blob/master/3-Lambda%20and%20Collections.md#spliterator)），最后调用wrappedSink.end()方法通知Sink数据处理结束。

#### 3.5 Stream forEach源码调用链

```java
public void forEach(Consumer<? super P_OUT> action) {
        evaluate(ForEachOps.makeRef(action, false));  //执行开关
    }
```

```java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        assert getOutputShape() == terminalOp.inputShape();
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        linkedOrConsumed = true;

        return isParallel()  //区分串行和并行，执行相应函数
               ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
               : terminalOp.evaluateSequential(this,sourceSpliterator(terminalOp.getOpFlags()));
    }
```

```java
final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
        copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
        return sink;
     // wrapSink(Objects.requireNonNull(sink) 得到一个代表了流水线上所有操作的Sink
    }
```



### 4 Stream运行流程原理总结

```java
Stream体系是一组接口家族,AbstractPipeline 是接口的实现,PipelineHelper 是管道的辅助类,StreamSupport是流的低级工具类
 
使用stage来抽象流水线上的每个操作
其实每个stage就是一个stream 也就是AbstractPipeline几个子类的  内部子类  Head StatelessOp statefulOp
StreamSupport用于创建生成Stream 对应的是Head类
其他的中间操作分为有状态和无状态的,中间操作通过方法比如 filter map 等返回的是StatelessOp  或者 statefulOp 
多个stage组合称为双向链表的形式 从而成了整个流水线
 
有了流水线,相邻两个操作阶段之间如何协调运算?
于是又有了sink的概念,又来协调相邻的stage之间计算运行
他的模式是begin  accept end 还有短路标记
他的accept就是封装了回调方法
 
所以说每个操作stage, StatelessOp  或者 statefulOp中又封装了Sink
通过AbstractPipeline提供的opWrapSink方法可以获取这个sink
调用这个sink的accept方法就可以调用当前操作的方法
 
那么如何串联起来呢?关键点在于opWrapSink方法 ,他接收一个Sink作为参数
在调用accept方法中  可以调用这个入参sink的accept方法
这样子从当前就能调用下一个,也就是说有了推动的动作
那么只需要找到开始,每个处理了之后都推动下一个,就顺序完成了所有的操作了
```




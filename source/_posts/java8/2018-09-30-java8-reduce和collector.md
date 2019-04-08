---
layout: post
title: reduce规约和Collector搜集器
tags:
- Java8
categories: Java8
description: java8
---
reduce规约和collector搜集器的理解

<!-- more --> 


### 1  理解collector的几个函数

collector接口定义的几个函数：

```java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    BinaryOperator<A> combiner();
    Function<A, R> finisher();
    Set<Characteristics> characteristics();
}
```

#### 1.1 建立新的结果容器 supplier方法

supplier方法必须返回一个结果为空的Supplier，也就是一个无参数函数，在调用时，它会创建一个空的累加器实例，供数据收集过程使用。就个人通俗的理解来说，这个方法定义你如何收集数据，之所以提炼出来就是为了让你可以传lambda表达式来指定收集器。对于toList, 我们直接返回一个空list就好。 

```java
public Supplier<List<T>> supplier() {
    return ArrayList::new;
}
```

#### 1.2 累加器执行累加的具体实现 accumulator方法

accumulator方法会返回执行归约操作的函数，该函数将返回void。当遍历到流中第n个元素时，这个函数就会执行。函数有两个参数，第一个参数是累计值，第二参数是第n个元素。累加值与元素n如何做运算就是accumulator做的事情了。比如toList, 累加值就是一个List，对于元素n，当然就是add。 

```java
public BiConsumer<List<T>, T> accumulator() {
    return List::add;
}
```

#### 1.3 对结果容器应用最终转换 finisher方法

当遍历完流之后，我们需要对结果做一个处理，返回一个我们想要的结果。这就是finisher方法所定义的事情。finisher方法必须返回在累积过程的最后要调用的一个函数，以便将累加器对象转换为整个集合操作的最终结果， 这个返回的函数在执行时，会有个参数，该参数就是累积值，会有一个返回值，返回值就是我们最终要返回的东西。对于toList, 我最后就只要拿到那个收集的List就好，所以直接返回List。 

```java
public Function<List<T>, List<T>> finisher() {
    return (i) -> i;
}
```

对于接收一个参数，返回一个value，我们可以想到Function函数，正如finisher()的返回值。对于这个返回参数本身的做法，Function有个静态方法 

```java
static <T> Function<T, T> identity() {
    return t -> t;
}
```

可以用`Function.identity()`代替上述lambda表达式。 

#### 1.4 合并两个结果容器 combiner

上面看起来似乎已经可以工作了，这是针对顺序执行的情况。我们知道Stream天然支持并行，但并行却不是毫无代价的。想要并行首先就必然要把任务分段，然后才能并行执行，最后还要合并。虽然Stream底层对我们透明的执行了并行，但如何并行还是需要取决于我们自己。这就是combiner要做的事情。combiner方法会返回一个供归约操作使用的函数，它定义了对流的各个子部分并行处理时，各个字部分归约所得的累加器要如何合并。对于toList而言，Stream会把流自动的分成几个并行的部分，每个部分都执行上述的归约，汇集成一个List。当全部完成后再合并成一个List。 

```java
public BinaryOperator<List<T>> combiner() {
    return (list1, list2) -> {
        list1.addAll(list2);
        return list1;
    };
}
```

这样，就可以对流并行归约了。它会用到Java7引入的分支/合并框架和Spliterator抽象。大概如下所示， 

1. 原始流会以递归方式拆分为子流，直到定义流是否进一步拆分的一个条件为非(如果分布式工作单位太小，并行计算往往比顺序计算要慢，而且要是生成的并行任务比处理器内核数多很多的话就毫无意义了)。
2. 现在，所有的子流都可以并行处理，即对每个子流应用顺序归约算法。
3. 最后，使用收集器combiner方法返回的函数，将所有的部分结果两两合并。这时，会把原始流每次拆分得到的子流对应的结果合并起来。

#### 1.5 characteristics方法

Characteristics是一个包含三个项目的枚举： 包括是否有明确的finisher，是否需要同步，是否有序

1. UNORDERED--(集合是无序的)：表示流中的元素无序。 
2. CONCURRENT--(集合的操作需要同步)：表示中间结果只有一个，即使在并行流的情况下。所以只有在并行流且收集器不具备CONCURRENT特性时，combiner方法返回的lambda表达式才会执行（中间结果容器只有一个就无需合并） 
3. IDENTITY_FINISH--(不用finisher)：表示中间结果容器类型与最终结果类型一致，此时finiser方法不会被调用 

#### 1.6 说明

​	我们迄今开发的ToListCollector是IDENTITY_FINISH的，因为用来累积流中元素
List已经是我们要的最终结果，用不着进一步转换了，但它并不是UNORDERED，因为用在有序
流上的时候，我们还是希望顺序能够保留在得到的List中。最后，它是CONCURRENT的，但我们
刚才说过了，仅仅在背后的数据源无序时才会并发处理

//通过collect代码认识一下:

```java
if (isParallel()
 && (collector.characteristics().contains(Collector.Characteristics.CONCURRENT))
 && (!isOrdered() ||collector.characteristics().contains(Collector.Characteristics.UNORDERED))){//并发
}
```

即并发条件:
在Collector.Characteristics.CONCURRENT的基础上,收集器Collector具有Characteristics.UNORDERED特性或者原始数据是无序的时才应用并发;

### 2 reduce 规约

Reduce中文含义为：减少、缩小；而Stream中的Reduce方法干的正是这样的活：根据一定的规则将Stream中的元素进行计算后返回一个唯一的值。  

它有三个变种，输入参数分别是一个参数、二个参数以及三个参数； 

#### 2.1 reduce 涉及的接口

1. BiFunction  它是一个函数式接口，包含的函数式方法定义如下： 

   ```java
   R apply(T t, U u);
   ```

2. BinaryOperator  它实际上就是继承自BiFunction的一个接口；它的定义：

   ```java
   public interface BinaryOperator<T> extends BiFunction<T,T,T>
   ```

3. BiConsumer 也是一个函数式接口，它的定义如下 

   ```java
   void accept(T t, U u);
   ```

#### 2.2 一个参数的Reduce

```java
Optional<T> reduce(BinaryOperator<T> accumulator) 
```

reduce在求和、求最大最小值等方面都可以很方便的实现，代码如下，注意其返回的结果是一个Optional对象： 

```java
Stream<Integer> s = Stream.of(1, 2, 3, 4, 5, 6);
/**
 * 求和，也可以写成Lambda语法：
 * Integer sum = s.reduce((a, b) -> a + b).get();
 */
Integer sum = s.reduce(new BinaryOperator<Integer>() {
    @Override
    public Integer apply(Integer integer, Integer integer2) {
        return integer + integer2;
    }
}).get();

/**
 * 求最大值，也可以写成Lambda语法：
 * Integer max = s.reduce((a, b) -> a >= b ? a : b).get();
 */
Integer max = s.reduce(new BinaryOperator<Integer>() {
    @Override
    public Integer apply(Integer integer, Integer integer2) {
        return integer >= integer2 ? integer : integer2;
    }
}).get(); 
```

#### 2.3 两个参数的Reduce

```java
T reduce(T identity, BinaryOperator<T> accumulator)
```

如要将一个String类型的Stream中的所有元素连接到一起并在最前面添加[value]后返回： 

```java
Stream<String> s = Stream.of("test", "t1", "t2", "teeeee", "aaaa", "taaa");
// 以下结果将会是：　[value]testt1t2teeeeeaaaataaa
 System.out.println(s.reduce("[value]", (s1, s2) -> s1.concat(s2)));
 
```

#### 2.4 三个参数的Reduce

```java
<U> U reduce(U identity,
                 BiFunction<U, ? super T, U> accumulator,
                 BinaryOperator<U> combiner)
```

参数分析：

- identity: 一个初始化的值；这个初始化的值其类型是泛型U，与Reduce方法返回的类型一致；注意此时Stream中元素的类型是T，与U可以不一样也可以一样，这样的话操作空间就大了；不管Stream中存储的元素是什么类型，U都可以是任何类型，如U可以是一些基本数据类型的包装类型Integer、Long等；或者是String，又或者是一些集合类型ArrayList等；后面会说到这些用法。
- accumulator: 其类型是BiFunction，输入是U与T两个类型的数据，而返回的是U类型；也就是说返回的类型与输入的第一个参数类型是一样的，而输入的第二个参数类型与Stream中元素类型是一样的。
- combiner: 其类型是BinaryOperator，支持的是对U类型的对象进行操作

第三个参数combiner主要是使用在并行计算的场景下；如果Stream是非并行时，第三个参数实际上是不生效的。 

##### 2.4.1 非并行

```java
//以下reduce生成的List将会是[aa, ab, c, ad]
 Stream<String> s1 = Stream.of("aa", "ab", "c", "ad");
  System.out.println(s1.reduce(new ArrayList<String>(), 
                               (r, t) -> {r.add(t); return r; }, (r1, r2) -> r1));
```

也可以进行元素过滤，即模拟Stream中的Filter函数： 

```java
 Stream<String> s1 = Stream.of("aa", "ab", "c", "ad");
        Predicate<String> predicate = t -> t.contains("a");
        // 模拟Filter查找其中含有字母a的所有元素，打印结果将是aa ab ad
         s1.reduce(new ArrayList<String>(),
             (r, t) -> {
                 if (predicate.test(t)) {
                     r.add(t);
                 }
                 return r;
             },
            (r1, r2) -> r1).stream().forEach(System.out::println);
```

##### 2.4.2 并行

示例一：

```java
Stream<Integer> stream = Stream.of(1, 2, 3).parallel();
     System.out.println(stream.reduce(4,
             (s1, s2) -> s1 + s2,
             (s1, s2) -> s1 + s2)
     );
```

并行时的计算结果是18，而非并行时的计算结果是10！ 
为什么会这样？ 
先分析下非并行时的计算过程；第一步计算4 + 1 = 5，第二步是5 + 2 = 7，第三步是7 + 3 = 10。按1.3.1中所述来理解没有什么疑问。 
那问题就是非并行的情况与理解有不一致的地方了！ 
先分析下它可能是通过什么方式来并行的？按非并行的方式来看它是分了三步的，每一步都要依赖前一步的运算结果！那应该是没有办法进行并行计算的啊！可实际上现在并行计算出了结果并且关键其结果与非并行时是不一致的！ 
那要不就是理解上有问题，要不就是这种方式在并行计算上存在BUG。 
暂且认为其不存在BUG，先来看下它是怎么样出这个结果的。猜测初始值4是存储在一个变量result中的；并行计算时，线程之间没有影响，因此每个线程在调用第二个参数BiFunction进行计算时，直接都是使用result值当其第一个参数（由于Stream计算的延迟性，在调用最终方法前，都不会进行实际的运算，因此每个线程取到的result值都是原始的4），因此计算过程现在是这样的：线程1：1 + 4 = 5；线程2：2 + 4 = 6；线程3：3 + 4 = 7；Combiner函数： 5 + 6 + 7 = 18！ 

示例二：

```java
System.out.println(Stream.of(1, 2, 3).parallel().reduce(4, (s1, s2) -> s1 + s2
                , (s1, s2) -> s1 * s2));
```

以上示例输出的结果是210！ 
它表示的是，使用4与1、2、3中的所有元素按(s1,s2) -> s1 + s2(accumulator)的方式进行第一次计算，得到结果序列4+1, 4+2, 4+3，即5、6、7；然后将5、6、7按combiner即(s1, s2) -> s1 * s2的方式进行汇总，也就是5 * 6 * 7 = 210。 

使用函数表示就是：(4+1) * (4+2) * (4+3) = 210;

下面来理解并行三个参数时的场景，实际上就是第一步使用accumulator进行转换（它的两个输入参数一个是identity, 一个是序列中的每一个元素），由N个元素得到N个结果；第二步是使用combiner对第一步的N个结果做汇总。

但这里需要注意的是，如果第一个参数的类型是ArrayList等对象而非基本数据类型的包装类或者String，第三个函数的处理上可能容易引起误解，如以下示例：

```java
Stream<String> s1 = Stream.of("aa", "ab", "c", "ad");
List<String> listStr = new ArrayList<>();
        for(int i = 0; i< 1000; i++){
            listStr.add("a" + i);
        }
        Stream<String>  s1 = listStr.stream();
        Predicate<String> predicate = t -> t.contains("a");
        // Vector 模拟Filter查找其中含有字母a的所有元素，打印结果将是aa ab ad
        s1.parallel().reduce(new ArrayList<String>(),
              (r, t) -> {
                if (predicate.test(t)) {
                    r.add(t);
                }
                return r;
             },
             (r1, r2) -> {
                 System.out.println(r1==r2);
                 return r2;
             }).stream().forEach(System.out::println);
```

其中System.out.println(r1==r2)这句打印的结果是什么呢？经过运行后发现是True！ 

为什么会这样？这是因为每次第二个参数也就是accumulator返回的都是第一个参数中New的ArrayList对象！因此combiner中传入的永远都会是这个对象，这样r1与r2就必然是同一样对象！

因此如果按理解的，combiner是将不同线程操作的结果汇总起来，那么一般情况下上述代码就会这样写:

```java
 //模拟Filter查找其中含有字母a的所有元素，由于使用了r1.addAll(r2)，其打印结果将不会是预期的aa ab ad
        Stream<String> s1 = Stream.of("aa", "ab", "c", "ad");
        Predicate<String> predicate = t -> t.contains("a");
        s1.parallel().reduce(new ArrayList<String>(), 
            (r, t) -> {
                if (predicate.test(t)) { 
                    r.add(t);
                }  
                return r; 
            },
            (r1, r2) -> {
                r1.addAll(r2); 
                return r1; 
             }).stream().forEach(System.out::println);
```

### 3 collect 方法

collect含义与Reduce有点相似；  

```java
<R> R collect(Supplier<R> supplier,
              BiConsumer<R, ? super T> accumulator,
              BiConsumer<R, R> combiner);
```

使用上面Reduce模拟Filter的示例进行演示 

```java
// 模拟Filter查找其中含有字母a的所有元素，打印结果将是aa ab ad
Stream<String> s1 = Stream.of("aa", "ab", "c", "ad");
Predicate<String> predicate = t -> t.contains("a");
System.out.println(s1.parallel().collect(() -> new ArrayList<String>(),
        (array, s) -> {if (predicate.test(s)) array.add(s); },
        (array1, array2) -> array1.addAll(array2)));
```

根据以上分析，这边理解起来就很容易了：每个线程都创建了一个结果容器ArrayList，假设每个线程处理一个元素，那么处理的结果将会是[aa],[ab],[],[ad]四个结果容器（ArrayList）；最终再调用第三个BiConsumer参数将结果全部Put到第一个List中

### 4 collector 中Characteristics特性分析

```java
public class MyFromSetToMap<T> implements Collector<T, Set<T>, Map<T, T>> {

    @Override
    public Supplier<Set<T>> supplier() {
        return () -> {
            System.out.println("supplier start");
            return new HashSet<>();
        };
    }

    @Override
    public BiConsumer<Set<T>, T> accumulator() {
        return (set, item) -> {
            System.out.println("accumulator start");
            set.add(item);
        };
    }

    @Override
    public BinaryOperator<Set<T>> combiner() {
        return (set1, set2) -> {
            System.out.println("combiner start");
            set1.addAll(set2);
            return set1;
        };
    }

    @Override
    public Function<Set<T>, Map<T, T>> finisher() {
        return (set) -> {
            System.out.println("finisher start");
            Map<T, T> map = new HashMap<>();
            set.stream().forEach(item -> map.put(item, item));
            return map;
        };
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of( Characteristics.UNORDERED));
    }
}
```

```java
 @Test
    public void test3(){
        List<String> listStr = Arrays.asList("zhangsan", "lisi", "wangwu", "aaaa", "bbbb", "cccc", null);
        Set<String> set = new HashSet<>();
        listStr.stream().forEach(item -> set.add(item));
        System.out.println(set);

       // Map<String, String> map = set.stream().collect(new MyFromSetToMap<>());
        Map<String, String> map = set.stream().collect(Collectors.toMap(String::toString, String::toString));
       // System.out.println(map);

        Map<String, String> map2 = new HashMap<>();
        map2.put(null, null);

    }
```








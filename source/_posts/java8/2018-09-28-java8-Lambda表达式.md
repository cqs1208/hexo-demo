---
layout: post
title: java8 Lambda表达式
tags:
- Java8
categories: Java8
description: java8
---
Lambda表达式，维基百科上的解释是一种用于表示匿名函数和闭包的运算符 

<!-- more --> 

Lambda 函数是一个可以接收任意多个参数（包括可选参数）并且返回单个表达式值的匿名函数。

### 1 Lambda详细介绍

​	可以把Lambda表达式理解为简洁地表示可传递的匿名函数的一种方式：它没有名称，但它有参数列表、函数主体、返回类型

#### 1.1 Lambda表达式的语法

(parameters) -> expression 或（请注意语句的花括号） (parameters) -> { statements; }

示例：利用Lambda表达式定义一个Comparator对象 (javalambda_1)

```java
@Data
public class Apple {
    private BigDecimal weight;
    private String color;
}


class test {
    public static void main(String[] args) {
        //先前----匿名内部类
        Comparator<Apple> byWeight = new Comparator<Apple>() {
            @Override
            public int compare(Apple o1, Apple o2) {
                return o1.getWeight().compareTo(o2.getWeight());
            }
        };

        //lambda表达式
        Comparator<Apple> byWeight2 = (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());

    }
}
```

Lambda表达式的组成：

- 参数列表——这里它采用了Comparator中compare方法的参数，两个Apple。
- 箭头——箭头->把参数列表与Lambda主体分隔开。
- Lambda主体——比较两个Apple的重量。表达式就是Lambda的返回值了。

测试Lambda语法：

```java
1. () -> {}
2. () -> "Raoul"
3. () -> {return "Mario";}
4. (Integer i) -> return "Alan" + i;
5. (String s) -> {"IronMan";}
```

### 2 函数式接口

在函数式编程语言中，Lambda表达式的类型是函数。**而在Java中，Lambda表达式是对象，它们必须依附于一类特别的对象类型——函数式接口(Functional Interface)**。 

**定义：**如果一个接口中，有且只有一个抽象的方法（Object类中的方法不包括在内），那这个接口就可以被看做是函数式接口。 

示例（Runnable接口）：

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

来看下Runnable接口的声明，在Java8后，Runnable接口多了一个FunctionalInterface注解，表示该接口是一个函数式接口。但是如果我们不添加FunctionalInterface注解的话，如果接口中有且只有一个抽象方法时，编译器也会把该接口当做函数式接口看待。 

**注：** 使用Lambda表达式可以创建函数式接口的实例。即Lambda表达式返回的是函数式接口类型 

#### 2.1 函数接口说明

函数接口可以（可以而不是必须）采用@FunctionalInterface 注解， 

**可以**包涵以下成员： 

1. 用default 定义的默认方法，默认方法不是抽象的，其包涵一个默认实现。（也就是有方法体）
2. 用static 定义的静态方法，静态方法是一个已经实现了的方法，不是抽象方法。
3. Object类自带的方法,如toString,equals等方法。

**必须**包含以下 

```java
必须有一个抽象方法。因为函数接口本质是接口，所以不需要abstract修饰。
```

测试函数式接口语法：(javalambda_3)

```java
public interface Adder{
	int add(int a, int b);
}
public interface SmartAdder extends Adder{
	int add(double a, double b);
}
public interface Nothing{
}
public interface TestInterface {
    int add (int a, int b);
    boolean equals(Object val1);
}

```

#### 2.2 常用函数式接口介绍和使用

java.util.function 包 (javalambda_4)

- Predicate （谓词， 断言）

  Predicate<T>接口定义了一个名叫test的抽象方法，它接受泛型T对象，并返回一个boolean

  ```java
   public static <T> List<T> filter(List<T> list, Predicate<T> p) {
          List<T> results = new ArrayList<>();
          for(T s: list){
              if(p.test(s)){
                  results.add(s);
              }
          }
          return results;
      }
  
      public static void main(String[] args) {
          List<String> listOfStrings = Arrays.asList("nihao", "hello", "world", "welcome");
          Predicate<String> nonEmptyStringPredicate = (String s) -> s.contains("h");
          List<String> nonEmpty = filter(listOfStrings, nonEmptyStringPredicate);
  
         // nonEmpty.stream().forEach(System.out::println);
      }
  ```

- Consumer （消费） 

  Consumer<T>定义了一个名叫accept的抽象方法，它接受泛型T的对象，没有返回（void）

  ```java
  
    public static void main(String[] args) {
          List<String> listOfStrings = Arrays.asList("nihao", "hello", "world", "welcome");
  
          Consumer consumer = (item)-> System.out.println(item);
  
          listOfStrings.stream().forEach(consumer);
      }
  ```

- Function（函数） 

  Function<T, R>接口定义了一个叫作apply的方法，它接受一个泛型T的对象，并返回一个泛型R的对象。

  ```java
  public static void main(String[] args) {
          List<String> listOfStrings = Arrays.asList("nihao", "hello", "world", "welcome");
  
          Function<String, String> function = (item) -> item + "qingsong";
          Stream<String> stream = listOfStrings.stream().map(function);
  
          Consumer consumer = (item)-> System.out.println(item);
          stream.forEach(consumer);
      }
  ```

#### 2.3 原始类型特化

​        Java类型要么是引用类型（比如Byte、Integer、Object、List），要么是原始类型（比如int、double、byte、char）。但是泛型（比如Consumer<T>中的T）只能绑定到引用类型。这是由泛型内部的实现方式造成的。①因此，在Java里有一个将原始类型转换为对应的引用类型的机制。这个机制叫作装箱（boxing）。相反的操作，也就是将引用类型转换为对应的原始类型，叫作拆箱（unboxing）

```java
IntPredicate evenNumbers = (int i) -> i % 2 == 0;
evenNumbers.test(1000);

Predicate<Integer> oddNumbers = (Integer i) -> i % 2 == 1;
oddNumbers.test(1000);
```

#### 2.4 函数式接口异常

```java
    /*  (javalambda_5)
      当一个lambda表达式被转换成一个函数式接口的实例时，请注意处理检查期异常。
      如果你需要Lambda表达式来抛出异常
      有两种办法：定义一个自己的函数式接口，并声明受检异常，或者把Lambda包在一个try/catch块中。
      */
    BufferedReaderProcessor p = (BufferedReader br) -> br.readLine();

    Function<BufferedReader, String> f = (BufferedReader b) -> {
        try {
            return b.readLine();
        }
        catch(IOException e) {
            throw new RuntimeException(e);
        }
    };

    @FunctionalInterface
    interface BufferedReaderProcessor {
        String process(BufferedReader b) throws IOException;
    }
```

### 3 Lambda类型推断

示例代码：(javalambda_6)

```java
List<Integer> list = Arrays.asList(3,2,4,5,6,7);
list.forEach(
    (Integer item) -> {
        System.out.println(item);
    }
);
```

这里参数的类型程序可以根据上下文进行推断，但是并不是所有的类型都可以推断出来，此时就需要我们显示的声明参数类型，当只有一个参数时小括号可以省略。当只有一行代码时，外边的大括号可以省略 

Collections类的sort()方法。 

```java
 public static <T> void sort(List<T> list, Comparator<? super T> c) {
        list.sort(c);
    }
```

​	假设有这么一个Student类Student implements A, B, C ,因为他实现了A,B,C三个接口，所以我们可以按接口A, B或C中任何一个的特性去排序，但是我比较完之后最终返回的一定是List<Student>这种类型的。我们定义Comparator<A> c，就是按A类型排序；定义Comparator<C> c，就是按C类型排序。也就是我们可以按自己的类型(Student)，或者是他的父类型(A, B, C)这种方式来去进行比较，比较的点或者参照物是不一样的，但我们最终比较完之后返回回来的一定List<Student>这种类型的。 所以<? super T>比<T>从语义上看更宽泛一些

`public static <T> void sort(List<T> list, Comparator<? super T> c)`里的泛型`Comparator<? super T> c`语义上更宽泛一些，但实际上比较的就是T类型**本身**。 

下面这lambda表达式可以直接推断出参数item的类型为String。 

```java
 List<String> list = Arrays.asList("nihao", "hello", "world", "welcome");
 //按字符串长度排序
 Collections.sort(list, (item1, item2) -> item1.length() - item2.length() );
 list.forEach(System.out::println);
```

推断不出类型参数的 示例

```java
Collections.sort(list, Comparator.comparing(item -> item.length()).reversed());
list.forEach(System.out::println);
```

​	这里要注意的是，我们这里的比较器是多层的，comparingInt返回一个比较器在调用reversed再返回一个比较器，最终我们需要的比较器是reversed返回的。第一个例子的lambda表达式就是一个比较器，我们直接就传入，java编译器很容易就推断出这个比较器里的类型参数。而第二个例子的lambda表达式仅仅作为第一层比较器comparingInt的参数传进去，它离这个上下文已经很远了，所以它对上下文的感知是很弱很远的，不能感知到集合list里的元素类型是什么！它就只能推断成Object类型

我们可以验证一下，很简单，现在它是2层，lambda表达式在第一层，我们把reversed去掉，不在第一层了吗？按理java就能感知到它的类型了。  

```java
Collections.sort(list, Comparator.comparing(item -> item.length()));
list.forEach(System.out::println)
```

​	的确如此，现在编译器就可以推断出类型参数了。说明java编译器它能感知离上下文最近的，第二个reversed离上下文最近，我们可以推断它返回的比较器的类型参数，没问题，但是comparingInt(ToIntFunction<? super T> keyExtractor)里的参数ToIntFunction<? super T> keyExtractor离上下文很远了，无法推断其参数的类型参数

### 4 lambda使用局部变量

局部变量

局部变量是存储在栈上的，而栈上的内容在当前线程执行完成之后就会被GC回收掉。 

lambda表达式

lambda表达式最终被处理为一个额外的线程去执行。绝对不是上面提到的线程。如果上面的线程执行完了，而这个线程又使用到了上面提到的局部变量会出现错误。

**使用局部变量限制说明：**实例变量存在堆中，而局部变量是在栈上分配，Lambda 表达(匿名类) 会在另一个线程中执行。如果在线程中要直接访问一个局部变量，可能线程执行时该局部变量已经被销毁了，而 final 类型的局部变量在 Lambda 表达式(匿名类) 中其实是局部变量的一个拷贝。 

### 5 方法引用

​	方法引用可以被看作仅仅调用特定方法的**Lambda的一种快捷写法**。它的基本思想是，如果一个Lambda代表的只是“直接调用这个方法”，那最好还是用名称来调用它，而不是去描述如何调用它。事实上，方法引用就是让你根据已有的方法实现来创建Lambda表达式。但是，**显式地指明方法的名称，你的代码的可读性会更好。**

#### 5.1 方法引用语法

使用方法引用时，目标引用放在分隔符::前，方法的名称放在后面

示例：(javatypelambda_7)

```java
 // lambda表达式
Comparator<Apple> byWeight3 = Comparator.comparing(item -> item.getWeight());

// 方法引用
Comparator<Apple> byWeight4 = Comparator.comparing(Apple::getWeight);
```

**说明：**Apple::getWeight就是引用了Apple类中定义的方法getWeight。请记住，不需要括号，因为你没有实际调用这个方法。方法引用就是Lambda表达式(Apple a) -> a.getWeight()的快捷写法

#### 5.2 构建方法引用的方式

方法引用的标准形式是：`类名::方法名`。

| 类型                             | 示例                                 |
| -------------------------------- | ------------------------------------ |
| 引用静态方法                     | ContainingClass::staticMethodName    |
| 引用某个对象的实例方法           | containingObject::instanceMethodName |
| 引用某个类型的任意对象的实例方法 | ContainingType::methodName           |
| 引用构造方法                     | ClassName::new                       |

1. **静态方法引用** 

   String::valueOf   等价于lambda表达式 (s) -> String.valueOf(s) 

   ```java
   List<Integer> listArr = Arrays.asList(3,5,6,7,8,2,0);
   // lambda 表示
   listArr.stream().map(item -> String.valueOf(item));
   // 静态方法引用表示
   listArr.stream().map(String::valueOf);
   ```

2. **特定实例对象的方法引用** 

   String::toString 等价于lambda表达式 (s) -> s.toString() 

   实例方法要通过对象来调用，方法引用对应Lambda，Lambda的第一个参数会成为调用实例方法的对象。 

   ```java
   //特定实例对象的方法引用
   List<String> listStr = Arrays.asList("a","d","t","h","m","S","G");
   // lambda 表示
   listStr.stream().map(item -> item.toString());
   listStr.stream().map(String::toString);
   // 特定实例对象的方法引用
   listStr.stream().map(String::toUpperCase);
   ```

3. **任意对象（属于同一个类）的实例方法引用** 

   这里引用的是字符串数组中任意一个对象的compareToIgnoreCase方法。 

   ```java
   //任意对象（属于同一个类）的实例方法引用
   String[] stringArray = { "Barbara", "James", "Mary", "John", "Patricia" };
   // lambda 表示
   Arrays.sort(stringArray, (item, item2) -> item.compareToIgnoreCase(item2));
   //任意对象（属于同一个类）的实例方法引用
   Arrays.sort(stringArray, String::compareToIgnoreCase);
   ```

4. **构造方法引用**

   **组成语法格式：**Class::**new** 

   String::new， 等价于lambda表达式 () -> new String()  

   ```java
   //构造方法引用
   //lambda 表达式
   Supplier<String> supplier = () -> new String();
   //构造方法引用
   Supplier<String> supplier2 = String::new;
   String aa = supplier2.get();
   ```

### 6 java对象和lambda表达式的类型转换

```java
public static void main(String[] args) {
        Consumer<Integer> consumer = (item) -> System.out.println(item);
        IntConsumer intConsumer = (item) -> System.out.println(item);

        test(consumer);
        test((Consumer) intConsumer);
        test(consumer::accept);
        test(intConsumer::accept);
    }

    static void test(Consumer<Integer> consumer){
        if(consumer instanceof Consumer){
            System.out.println("hello");
        }
    }
```

可以把lambda表达式看作函数式接口对象传递，而不用考虑函数接口本身的类型


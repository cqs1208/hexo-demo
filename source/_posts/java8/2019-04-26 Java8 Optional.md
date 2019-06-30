---
layout: post
title: java8 Optional
tags:
- Java8
categories: Java8
description: Java8
---

本文介绍optional

<!-- more --> 

### 1 Optional 类包含的方法：

#### 1 of

为非null的值创建一个Optional。of方法通过工厂方法创建Optional类。需要注意的是，创建对象时传入的参数不能为null。如果传入参数为null，则抛出NullPointerException 。

```java
//调用工厂方法创建Optional实例
Optional<String> name = Optional.of("Sanaulla");
//传入参数为null，抛出NullPointerException.
Optional<String> someNull = Optional.of(null);
```

#### 2 **ofNullable**

为指定的值创建一个Optional，如果指定的值为null，则返回一个空的Optional。ofNullable与of方法相似，唯一的区别是可以接受参数为null的情况。

```java
//下面创建了一个不包含任何值的Optional实例
//例如，值为'null'
Optional empty = Optional.ofNullable(null);
```

#### 3 **isPresent**

如果值存在返回true，否则返回false。

```java
//isPresent方法用来检查Optional实例中是否包含值
if (name.isPresent()) {
    //在Optional实例内调用get()返回已存在的值
    System.out.println(name.get());//输出Sanaulla
}
```

#### 4 get

如果Optional有值则将其返回，否则抛出NoSuchElementException。

```java
//执行下面的代码会输出：No value present 
try {
    //在空的Optional实例上调用get()，抛出NoSuchElementException
    System.out.println(empty.get());
} catch (NoSuchElementException ex) {
    System.out.println(ex.getMessage());
}
```

#### 5 ifPresent

如果Optional实例有值则为其调用consumer，否则不做处理。

```java
//ifPresent方法接受lambda表达式作为参数。
//lambda表达式对Optional的值调用consumer进行处理。
name.ifPresent((value) -> {
    System.out.println("The length of the value is: " + value.length());
});
```

#### 6 orElse

如果Optional实例有值则将其返回，否则返回orElse方法传入的参数。

```java
//如果值不为null，orElse方法返回Optional实例的值。
//如果为null，返回传入的消息。
//输出：There is no value present!
System.out.println(empty.orElse("There is no value present!"));
//输出：Sanaulla
System.out.println(name.orElse("There is some value!"));
```

#### 7 orElseGet

orElseGet与orElse方法类似，区别在于得到的默认值。orElse方法将传入的字符串作为默认值，orElseGet方法可以接受Supplier接口的实现用来生成默认值。

```java
//orElseGet与orElse方法类似，区别在于orElse传入的是默认值，
//orElseGet可以接受一个lambda表达式生成默认值。
//输出：Default Value
System.out.println(empty.orElseGet(() -> "Default Value"));
//输出：Sanaulla
System.out.println(name.orElseGet(() -> "Default Value"));
```

#### 8 orElseThrow

如果有值则将其返回，否则抛出supplier接口创建的异常。

```java
try {
    //orElseThrow与orElse方法类似。与返回默认值不同，
    //orElseThrow会抛出lambda表达式或方法生成的异常 
    empty.orElseThrow(ValueAbsentException::new);
} catch (Throwable ex) {
    //输出: No value present in the Optional instance
    System.out.println(ex.getMessage());
}
```

#### 9 map

如果有值，则对其执行调用mapping函数得到返回值。如果返回值不为null，则创建包含mapping返回值的Optional作为map方法返回值，否则返回空Optional。

```java
//map方法执行传入的lambda表达式参数对Optional实例的值进行修改。
//为lambda表达式的返回值创建新的Optional实例作为map方法的返回值。
Optional<String> upperName = name.map((value) -> value.toUpperCase());
System.out.println(upperName.orElse("No value found"));
```

#### 10 flatMap

如果有值，为其执行mapping函数返回Optional类型返回值，否则返回空Optional。flatMap与map（Funtion）方法类似，区别在于flatMap中的mapper返回值必须是Optional。调用结束时，flatMap不会对结果用Optional封装。而map方法的mapping函数返回值可以是任何类型T，调用结束时，map一定会对结果用Optional封装,如果mapper返回值是Optional，那么map就会将结果封装成Optional<Optional>类型。

```java
Optional<String> name = Optional.ofNullable("Walker");
Optional<String> upperName = name.flatMap((value) -> Optional.of(value.toUpperCase()));
System.out.println(upperName.orElse("No value found"));//输出WALKER
```

#### 11 filter

filter个方法通过传入限定条件对Optional实例的值进行过滤。如果有值并且满足断言条件返回包含该值的Optional，否则返回空Optional。

```java
//filter方法检查给定的Option值是否满足某些条件。
//如果满足则返回同一个Option实例，否则返回空Optional。
Optional<String> longName = name.filter((value) -> value.length() > 6);
System.out.println(longName.orElse("The name is less than 6 characters"));//输出Sanaulla
 
//另一个例子是Optional值不满足filter指定的条件。
Optional<String> anotherName = Optional.of("Sana");
Optional<String> shortName = anotherName.filter((value) -> value.length() > 6);
//输出：name长度不足6字符
System.out.println(shortName.orElse("The name is less than 6 characters"));
```

### 2 使用Optional的正确姿势

**错误的姿势：**

```java
User user;
Optional<User> optional = Optional.of(user);
if (optional.isPresent()) {
    return optional.get().getOrders();
} else {
    return Collections.emptyList();;
}
```

**这和之前的写法没有任何区别**

```java
if (user != null) {
    return user.getOrders();
} else {
    return Collections.emptyList();;
}
```

当我们还在以如下几种方式使用 Optional 时, 就得开始检视自己了

1. 调用 `isPresent()` 方法时
2. 调用 `get()` 方法时
3. Optional 类型作为类/实例属性时
4. Optional 类型作为方法参数时

isPresent() 与 obj != null 无任何分别, 我们的生活依然在步步惊心. 而没有 isPresent() 作铺垫的 get() 调用在 IntelliJ IDEA 中会收到告警。把 Optional 类型用作属性或是方法参数在 IntelliJ IDEA 中更是强力不推荐的。

所以 Optional 中我们真正可依赖的应该是除了 `isPresent()` 和 `get()` 的其他方法。

**正确的姿势**

创建Optional对象:

```java
Optional<Soundcard> sc = Optional.empty(); 
SoundCard soundcard = new Soundcard();
Optional<Soundcard> sc = Optional.of(soundcard); 
Optional<Soundcard> sc = Optional.ofNullable(soundcard);
```

存在即返回, 无则提供默认值：

```java
return user.orElse(null);  //而不是 return user.isPresent() ? user.get() : null;
return user.orElse(UNKNOWN_USER);
```

存在即返回, 无则由函数来产生：

```java
return user.orElseGet(() -> fetchAUserFromDatabase()); 
//而不要 return user.isPresent() ? user: fetchAUserFromDatabase();
```

存在才对它做点什么：

```java
user.ifPresent(System.out::println);
 
//而不要下边那样
if (user.isPresent()) {
  System.out.println(user.get());
}
```

使用map抽取特定的值或者做值的转换：

```java
return user.map(u -> u.getOrders()).orElse(Collections.emptyList())
 
//上面避免了我们类似 Java 8 之前的做法
if(user.isPresent()) {
  return user.get().getOrders();
} else {
  return Collections.emptyList();
}
```

级联使用map:

```java
return user.map(u -> u.getUsername())
           .map(name -> name.toUpperCase())
           .orElse(null);
```

这样避免了连续的空值判断：

```java
User user = .....
if(user != null) {
    String name = user.getUsername();
    if(name != null) {
        return name.toUpperCase();
    } else {
        return null;
    }
} else {
    return null;
}
```

级联的Optional对象使用flatMap：

```java
String version = computer.flatMap(Computer::getSoundcard)
                   .flatMap(Soundcard::getUSB)
                   .map(USB::getVersion)
                   .orElse("UNKNOWN");
```

使用filter拒绝特定的值:

```java
Optional<USB> maybeUSB = ...;
maybeUSB.filter(usb -> "3.0".equals(usb.getVersion()).ifPresent(() -> System.out.println("ok"));
```

用了 isPresent() 处理 NullPointerException 不叫优雅, 有了  orElse, orElseGet 等, 特别是 map 方法才叫优雅.
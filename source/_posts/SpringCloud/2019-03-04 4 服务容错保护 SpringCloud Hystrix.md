---
layout: post
title: 04 服务容错保护
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

在微服务架构中，我们将系统拆分成了很多服务单元，各单元的应用间通过服务注册与订阅的方式互相依赖

<!-- more --> 

# 服务容错保护

​	在微服务架构中，我们将系统拆分成了很多服务单元，各单元的应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进程中运行，依赖通过远程调用的方式 执行，这样就有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问 题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会因 等待出现故障的依赖方响应形成任务积压，最终导致自身服务的瘫痪。
	举个例子，在一个电商网站中，我们可能会将系统拆分成用户、订单、库存、积分、评论等一系列服务单元。用户创建一个订单的时候，客户端将调用订单服务的创建订单接口，此时创建订单接口又会向库存服务来请求出货（判断是否有足够库存来出货〉。此时若库存服务因自身处理逻辑等原因造成响应缓慢，会直接导致创建订单服务的线程被挂起，以等待库存申请服务的响应，在漫长的等待之后用户会因为请求库存失败而得到创建订单 失败的结果。如果在高并发情况之下，因这些挂起的线程在等待库存服务的响应而未能释放，使得后续到来的创建订单请求被阻塞，最终导致订单服务也不可用。

​	在微服务架构中，存在着那么多的服务单元，若一个单元出现故障，就很容易因依赖关系而引发故障的蔓延，最终导致整个系统的瘫痪，这样的架构相较传统架构更加不稳定。 为了解决这样的问题，产生了断路器等一系列的服务保护机制。

​	断路器模式源于MartinFowler的CircuitBreaker一文。“断路器”本身是一种开关装置，用于在电路上保护线路过载，当线路中有电器发生短路时，“断路器”能够及时切断故障电路，防止发生过载、发热甚至起火等严重后果。

​	在分布式架构中，断路器模式的作用也是类似的，当某个服务单元发生故障〈类似用电器发生短路〉之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放， 避免了故障在分布式系统中的蔓延。

​	针对上述问题，SpringCloud Hystrix实现了断路器、线程隔离等一系列服务保护功能。它也是基于Netflix的开源框架Hys位仅实现的，该框架的目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备服 务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等强大功能。

​	接下来， 我们就从一个简单示例开始对Spring Cloud Hystrix的学习与使用。

## 1 快速入门

​	在开始使用SpringCloud Hystrix实现断路器之前，我们先用之前实现的一些内容作为基础，构建一个如下图架构所示的服务调用关系。

![hystrxi](/images/SpringCloud/SpringCloud_hystrxi.png)

​	我们在这里需要启动的工程有如下一些。

- eureka-server工程： 服务注册中心， 端口为1111。
- hello…service工程：HELLO-SERVICE的服务单元，两个实例启动端口分别为8081和80820
- ribbon-consume工程： 使用Ribbon实现的服务消费者， 端口为 9000。

在未加入断路器之前， 关闭 8081的实例， 发送 GET 请求到http://localhost: 9000/ribbon-consumer， 可以获得下面的输出：

![hystrix2](/images/SpringCloud/SpringCloud_hystrix2.png)

下面我们开始引入Spring Cloud Hystrix。

- 在ribbon-consumer 工程的pom.xml的dependency节点中引入spring­-cloud-starter-hystrix依赖：

  ```java
  <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-hystrix</artifactId>
  </dependency>
  ```

- 在ribbon-consumer 工程的主类ConsumerApplication中使用＠Enable­Circui tBreaker注解开启断路器功能：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public class ApplicationRibbonClient {
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }


    public static void main(String[] args) {
        SpringApplication.run(ApplicationRibbonClient.class);
    }
}
```

**注意：**  这里还可以使用 Spring Cloud 应用中的＠SpringCloudApplication 注解来修饰应用主类， 该注解的具体定义如下所示。 可以看到， 该注解中包含了上述我们所引用的

```java

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```

- 改造服务消费方式， 新增 HelloService 类， 注入 RestTemplate 实例。 然后，将在 ConsumerController 中对 RestTemplate 的使用迁移到 helloService函数中， 最后， 在 helloService 函数上增加＠ HystrixCommand 注解来指定回调方法：

  ```java
  @Service
  public class HelloService {
      @Autowired
      private RestTemplate restTemplate;
  
      @HystrixCommand(fallbackMethod = "helloFallback")
      public String helloService(){
          Map<String, String> map = new HashMap<>();
          map.put("name", "aa");
          return restTemplate.getForEntity("http://spring-cloud-producer/hello?name={name}", String.class, map).getBody();
      }
  
      public String helloFallback(){
          return "error";
      }
  }
  ```

  ​	 此时我们继续断开 8081 的 HELLO-SERVICE,
  然后访问 http://localhost:9000/ribbon-consumer，当轮询到 8081 服务端时，输出内容为 error，不再是之前的错误内容，Hystrix 的服务回调生效。除了通过断开具体的服务实例来模拟某个节点无法访问的情况之外，我们还可以模拟服务阻塞（长时间未响应）的情况。我们对HELLO-SERVICE 的／hello 接口做一些修改，具体如下：

  ```java
  public String hello(@RequestParam String name) throws  Exception{
          //让处理线程等待几分钟
          int sleepTime = new Random().nextInt(3000);
          Thread.sleep(sleepTime);
          
          ServiceInstance instance = client.getLocalServiceInstance();
          return "hello "+name+"，is return message and port is: " + instance.getHost()  +"  server_id is: "+ instance.getServiceId() ;
      }
  ```

  ```properties
  通过 Thread. sleep （）函数可让／hello 接口的处理线程不是马上返回内容，而是在阻塞几秒之后才返回内容。 由于 Hystrix 默认超时时间为 2000 毫秒，所以这里采用了。至3000 的随机数以让处理过程有的触发，在消费者调用函数中做一些记录，具体如下：
  ```

```java
public String hello(@RequestParam String name) throws  Exception{
        log start = System.currentTimeMillis();
    //消费服务的逻辑
    。。。。。
      	long end = System.currentTimeMillis();
    }
```

​	重新启动HELLO-SERVICE和RIBBON-CONSUMER的实例，连续访问http://localhost:9000/ribbo口－consumer几次，我们可以观察到，当RIBBON-CONSUMER 的控制台中输出的Spendtime大于2000的时候，就会返回error，即服务消费者因调用的服务超时从而触发熔断请求，并调用回调逻辑返回结果。

## 2 原理分析

​	通过上面的快速入门示例，我们对Hystrix的使用场景和使用方法已经有了一个基础的认识。接下来我们通过解读Netflix hystrix 官方的流程图来详细了解一下：当一个请求调用了相关服务依赖之后 Hystrix是如何工作的（即如上例中所示，当访问了http://localhost: 9000/ribbon-consumer请求之后，在RIBBON-CONSUMER中是如何处理的）

### 2.1 工作流程

下面我们根据图中标记的数字顺序来解释每一个环节的详细内容

![断路器时序图](/images/SpringCloud/SpringCloud_hystrix4.png)

#### 2.1.1 创建HystrixCommand或HystrixObservableCommand对象

​	首先，构建一个HystrixCommand或是HystrixObservableCommand对象，用来表示对依赖服务的操作请求， 同时传递所有需要的参数。 从其命名中我们就能知道它采用了 “ 命令模式” 来实现对服务调用操作的封装。 而这两个Command对象分别针对不同的应用场景。

- HystrixCommand: 用在依赖的服务返回单个操作结果的时候
- HystrixObservableCommand:用在依赖的服务返回多个操作结果的时候。

​        命令模式， 将来自客户端的请求封装成一个对象， 从而让你可以使用不同的请求对客户端进行参数化。 它可以被用于实现 “行为请求者” 与 “行为实现者” 的解稿， 以便使两 者可以适应变化。 下面的示例是对命令模式的简单实现：

```java
//接收者
public class Receiver {
    public void action() {
        //真正的业务逻辑
    }
}

//抽象命令
interface Command {
    void execute();
}

//具体命令实现
public class concreteCommand implements Command {
    private Receiver receiver;
    public ConcreteCommand(Receiver receiver){
        this.receiver = receiver;
    }
}

//客户端调用者
public class invoken{
    private Command command;
    public void setCommand(Command command) {
        this.command = command;
    }
    public void action() {
        this.command.execute();
    }
}

public class Client {
    public static void main(Stirng[] args){
        Receiver recerver = new Receiver();
        Command command = new ConcreteCommand(receiver);
        Invoker invoker = new Invoken();
        invoker.action(); //客户端通过调用者来执行命令
    }
}
```

从代码中， 我们可以看到这样几个对象。

- Receiver：接收者， 它知道如何处理具体的业务逻辑。
- Command：抽象命令， 它定义了一个命令对象 应具备的 一系列命令操作， 比如execute （）、undo （）、redo （）等。 当命令操作被调用的时候就会触发接收者去做具体命令对应的业务逻辑。
- ConcreteCommand：具体的命令实现， 在这里它绑定了命令操作与接收者之间的关系， execute（）命令的实现委托给了 Receiver的 action（）函数。
- Invoker：调用者， 它持有一个命令对象， 并且可以在需要的时候通过命令对象完成具体的业务逻辑。

​        从上面的示例中，我们可以看到，调用者Invoker与操作者Receiver通过 Command 命令接口实现了解耦。 对于调用者来说， 我们可以为其注入多个命令操作， 比如新建文件、复制文件、删除文件这样三个操作， 调用者只需在需要的时候直接调用即可， 而不需要知道这些操作命令实际是如何实现的。 而在这里所提到的 HystrixCommand 和 HystrixObservableCommand则是在Hys胁中对Command的进一步抽象定义。 在后续的内容中， 会逐步展开介绍它的部分内容来帮助理解其运作原理。

​         从上面的示例中我们也可以发现， Invoker和Receiver的关系非常类似于 “请求－响应 ” 模式， 所以它比较适用 于实现记录日志、撤销操作、队列请求等。

​	在下面这些情况下 应考虑使用命令模式。

- 使用命令模式作为 “ 回调（CallBack） ” 在面向对象系统中的替代。 “CallBack” 讲的便是先将一个函数登记上， 然后在以后调用此函数。
	 ·	需要在不同的时间指定请求、将请求排队。 一个命令对象和原先的请求发出者可以有不同的生命期。 换言之， 原先的请求发出者可能己经不在了， 而命令对象本身仍然是活动的。 这时命令的接收者可以是在本地， 也可以在网络的另外一个地址。命令对象可以在序列化之后传送到另外一台机器上去。
- 系统需要支持命令的撤销。 命令对象可以把状态存储起来， 等到 客户端需要撤销命令所产生的效果时， 可以调用 undo（）方法， 把命令所产生的效果撤销掉。 命令对象还可以提供redo（）方法， 以供客户端在需要时再重新实施命令效果。
- 如果要将系统中所有的数据更新到日志里，以便在系统崩溃时，可以根据日志读回所有的数据更新命令，重新调用Execute（）方法一条一条执行这些命令，从而恢复系统在崩溃前所做的数据更新。

#### 2.1.2 命令执行

​        从图中我们可以看到一共存在4种命令的执行方式，而Hystrix在执行时会根据创建的Command对象以及具体的情况来选择一个执行。其中HystrixCommand实现了下面两个 执行方式。

- execute （）：同步执行，从依赖的服务返回一个单一的结果对象，或是在发生错误的时候抛出异常。

- queue （）：异步执行，直接返回一个Future对象，其中包含了服务执行结束时要返回的单一结果对象。

  ```java
  R value = command.execute();
  Future<R> fValue = command.queue();
  ```

而HystrixObservableCommand实现了另外两种执行方式。

- observe （）：返回Observable对象，它代表了操作的多个结果，它是一个Hot Observable 
- toObservable（）：同样会返回Observable对象，也代表了操作的多个结果，但它返回的是一个ColdObservable

```java
Observable<R> ohValue = command.observe();
Observable<R> ocValue = command.toObservable();
```

​        在Hys位ix的底层实现中大量地使用了RxJava，为了更容易地理解后续内容，在这里对RxJava的观察者一订阅者模式做一个简单的入门介绍。

​       上面我们所提到的Observable对象就是RxJava中的核心内容之一，可以把它理解为“事件源”或是“被观察者”，与其对应的Subscriber对象，可以理解为“订阅者”或是“观察者”。者两个对象是RxJava响应式编程的重要组成部分。

- Observable用来向订阅者Subscriber对象发布事件，Subscriber对象则在接收到事件后对其进行处理，而在这里所指的事件通常就是对依赖服务的调用。
- 一个Observable可以发出多个事件，直到结束或是发生异常。
- Observable对象每发出一个事件，就会调用对应观察者Subscriber对象的 onNext （）方法。
- 每一个Observable的执行，最后一定会通过调用Subscriber.onCornpleted()或者Subscriber.onError（）来结束该事件的操作流。

下面我们通过一个简单的例子来直观理解一下Observable与Subscribers

```java
//创建事件源observable
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>(){
    @Override
    public void call(Subscriber<? super String> subscriber){
        subscriber.onNext("Hello RxJava");
        subscriber.onNext("I am 程序猿DD")；
        subscriber.onCompleted();
    }
});

//创建订阅者subscriber
Subscriber<String> subscriber = new Subscriber<String>(){
    @Override
    public void onCompleted(){
    }
    @Override
    public void onError(Throwable e){}
    @Override
    public void onNext(String s){
        System.out.pringln("Subscriber : " + s)
    }
};

//订阅
observable.subscribe(subscriber);
```

​	在该示例中， 创建了一个简单的事件源 observable，一个对事件传递内容输出的订阅者 subscriber， 通过 observable.subscribe (subs,criber）来触发事件的发布。

​	在这里我们对于事件源observable提到了两个不同的概念：HotObservable和Cold Observable，分别对应了上面command.observe（）和cornmand.toObservable（）的返回对象。其中HotObservable，它不论“事件源”是否有“订阅者气都会在创建后对事件进行发布，所以对于HotObservable的每一个“订阅者”都有可能是从“事件源”的中途开始的，并可能只是看到了整个操作的局部过程。而ColdObservable在没有“订阅者”的时候并不会发布事件，而是进行等待，直到有“订阅者”之后才发布事件，所以对于Cold Observable的订阅者，它可以保证从一开始看到整个操作的全部过程。

​	大家从表面上可能会认为只是在HystrixObservableCommand中使用了Rx.Java,而实际上execute（）、queue（）也都使用了RxJava来实现。从下面的源码中我们可以看到：

- execute （）是通过queue（）返回的异步对象Future<R＞的get（）方法来实现同步执行的。该方法会等待任务执行结束，然后获得R类型的结果进行返回。
- queue()则是通过toObservable()来获取一个Cold Observable,并且通过toBlocking （）将该Observable转换成BlockingObservable，它可以把数据以阻塞的方式发射出来。而toFuture方法则是把BlockingObservable转换为个Future，该方法只是创建个Future返回，并不会阻塞，这使得消费者可以自己决定如何处理异步操作。而execute（）就是直接使用了queue（）返回的Future中的阻塞方法get（）来实现同步操作的。同时通过这种方式转换的Future要求Observable只发射个数据，所以这两个实现都只能返回单一结果。

```java
public R execute(){
    try {
        return queue().get();
    }catch(Exception e){
        
    }
}

public Future<R> queue(){
    final Observable<R> o = toObservable();
    final Future<R> f = o.toBlocking().toFuture();
    
    if (f.isDone()){
        //处理立即抛出的错误
        。。。。
    }
    return f;
}
```

#### 2.1.3 结果是否被缓存

若当前命令的请求缓存功能是被启用的，并且该命令缓存命中，那么缓存的结果会立即以Observable对象的形式返回。

#### 2.1.4 断路器是否打开

在命令结果没有缓存命中的时候，Hystrix在执行命令前需要检查断路器是否为打开状态：

- 如果断路器是打开的，那么Hystrix不会执行命令，而是转接到fallback处理逻辑（对应下面第8步）
- 如果断路器是关闭的，那么Hystrix跳到第5步，检查是否有可用资源来执行命令。

#### 2.1.5 线程池/请求队列/信号量是否占满

​	如果与命令相关的线程池和请求队列，或者信号量（不使用线程池的时候〉已经被占满，那么Hystrix也不会执行命令，而是转接到fallback处理逻辑（对应下面第8步〉。

​	需要注意的是，这里Hystrix所判断的线程池并非容器的线程池，而是每个依赖服务的专有线程池。Hystrix为了保证不会因为某个依赖服务的问题影响到其他依赖服务而采用了“舱壁模式”（BulkheadPattern）来隔离每个依赖的服务。关于依赖服务的隔离与线程池相关的内容见后续详细介绍。

#### 2.1.6 HystrixObservableCommand.co口struct（）或HystrixCommand.run()

Hystrix会根据我们编写的方法来决定采取什么样的方式去请求依赖服务。

- HystrixCommand.run（）：返回一个单一的结果，或者抛出异常。
- HystrixObservableCommand.construct （）：返回一个Observable对象来发射多个结果，或通过onError发送错误通知。

​        如果run（）或construct（）方法的执行时间超过了命令设置的超时阙值，当前处理 线程将会抛出一个TimeoutException（如果该命令不在其自身的线程中执行，则会通 过单独的计时线程来抛出〉。在这种情况下，Hystrix会转接到fallback处理逻辑〈第8步〉。同时，如果当前命令没有被取消或中断，那么它最终会忽略run（）或者construct（）方法的返回。

​        如果命令没有抛出异常并返回了结果，那么Hys仕ix在记录一些日志并采集监控报告之后将该结果返回。在使用run（）的情况下，Hystrix会返回一个Observable，它发射单个结果并产生onCornpleted的结束通知；而在使用construct（）的情况下，Rys的x会直接返回该方法产生的Observable对象。

#### 2.1.7 计算断路器的健康度

​       Rys位ix会将“成功”、“失败”、“拒绝”、“超时”等信息报告给断路器，而断路器会维 护一组计数器来统计这些数据。

​        断路器会使用这些统计数据来决定是否要将断路器打开，来对某个依赖服务的请求进行“熔断／短路飞直到恢复期结束。若在恢复期结束后，根据统计数据判断如果还是未达到健康指标，就再次“熔断／短路”。

#### 2.1.8 fallback 处理

​	当命令执行失败的时候，Hystrix会进入fallback尝试回退处理，我们通常也称该操作为 服务降级气而能够引起服务降级处理的情况有下面几种：

- 第4步，当前命令处于“熔断／短路”状态，断路器是打开的时候。
- 第5步，当前命令的线程池、请求队列或者信号量被占满的时候。
- 第6步，HystrixObservableConunand.construct（）或HystrixCommand.ru口（）抛出异常的时候。

​        在服务降级逻辑中，我们需要实现一个通用的响应结果，并且该结果的处理逻辑应当是从缓存或是根据一些静态逻辑来获取，而不是依赖网络请求获取。如果一定要在降级逻辑中包含网络请求，那么该请求也必须被包装在HystrixCommand或是HystrixObservableCommand 中，从而形成级联的降级策略，而最终的降级逻辑一定不是一个依赖网络请求的处理，而是一个能够稳定地返回结果的处理逻辑。

​        在HystrixComma口d和HystrixObservableCommand中实现降级逻辑时还略有不同：

- 当使用HystrixCommand的时候，通过实现HystrixCommand.getFallback()来实现服务降级逻辑。
- 当使用Hys℃rixObser飞rableComand 的时候，通过HystrixObservable-Command.resumeWithFallback()实现服务降级逻辑，该方法会返回一个Observable对象来发射一个或多个降级结果。

​        当命令的降级逻辑返回结果之后，Hystrix就将该结果返回给调用者。当使用 HystrixCommand.getFallback（）的时候，它会返回一个Observable对象，该对象会发射getFallback（）的处理结果。而使用HystrixObservableGommand. resumeWi thFallback （）实现的时候，它会将Observable对象直接返回。

​       如果我们没有为命令实现降级逻辑或者降级处理逻辑中抛出了异常，Hystrix依然会返回一个Observable对象，但是它不会发射任何结果数据，而是通过onError方法通知命令立即中断请求，并通过onError（）方法将引起命令失败的异常发送给调用者。实现一 个有可能失败的降级逻辑是一种非常糟糕的做法，我们应该在实现降级策略时尽可能避免失败的情况。

​        当然完全不可能出现失败的完美策略是不存在的，如果降级执行发现失败的时候， Hystrix会根据不同的执行方法做出不同的处理。

- execute （）： 抛出异常。
- queue （）：正常返回Future对象，但是当调用get（）来获取结果的时候会抛出异常
- observe （）： 正常返回Observable对象， 当订阅它的时候， 将立即通过调用订阅者的onError方法来通知中止请求。
- toObservable（）： 正常返回Observable对象， 当订阅它的时候， 将通过调用订阅者的onError方法来通知中止请求。

#### 2.1.9 返回成功的响应

​      当Hystrix命令执行成功之后， 它会将处理结果直接返回或是以Observable 的形式返回。 而具体以哪种方式返回取决于之前第2步中我们所提到的对命令的4种不同执行方式， 下图中总结了这4种调用方式之间的依赖关系。 我们可以将此图与在第2步中对前两者源码的分析联系起来 ， 并且从源头toObservable（）来开始分析。![hystrix3](/images/hystrix3.png)

- toObservable（）： 返回最原始的 Observable ， 必须通过订阅它才会真正触发命令的执行流程。
- observe （）：在toObservable（）产生原始Observable 之后立即订阅它， 让命令能够马上开始异步执行，并返回一个Observable对象，当调用它的subscribe时，将重新产生结果和通知给订阅者。
- queue （）： 将 toObservable（）产生的原始Observable通过 toBlocking ()方法转换成BlockingObservable 对象， 并调用它的toFuture（）方法 返回异步的Future对象
- execute （）：在queue（）产生异步结果Future对象之后，通过调用get（）方法阻塞并等待结果的返回。

## 3 断路器原理

​       断路器在 HystrixCommand 和 HystrixObservableCornmand 执行过程中起到了举足轻重的作用，它是 Hystrix 的核心部件。 那么断路器是如何决策熔断和记录信息的呢？

我们先来看看断路器HystrixCircuitBreaker的定义：

```java
public interface HystrixCiruitBreaker {
    static class Factory {....}
    static class HystrixCircuitBreaqkerImpl implements HystrixcircuitBreaker {...}
    static class NoOpcircuitBreaker implements HystrixCircuiBreaker {...}
    public boolean allowRequest();
    public boolean isOpen();
    void markSuccess();
}
```

可以看到它的接口定义并不复杂， 主要定义了三个断路器的抽象方法。

- allowRequest （）： 每个 Hys往以命令的请求都通过它判断是否被执行。
- isOpen （）： 返回当前断路器是否打开。
- markSuccess （）： 用来闭合断路器。

另外还有三个静态类。

- 静态类 Factory 中维护了 个 Hystrix命令与 HystrixCircuitBreaker 的关系集合：ConcurrentHashMap<String, HystrixCircui tBreaker> c;ircui t­BreakersByCornmand，其中 String 类型的 key 通过 HystrixCornmandKey 定义，每一个 Hystrix 命令需要有一个 key 来标识， 同时一个 Hystrix 命令也会在该集合中
  找到它对应的断路器 HystrixCircuitBreaker 实例。
- 静态类 NoOpCircuitBreaker 定义了一个什么都不做的断路器实现，它允许所有请求， 并且断路器状态始终闭合。
- 静态类 HystrixCircuitBreakerimpl 是断路器接口 HystrixCircuitBreaker的实现类， 在该类中定义了断路器的 4 个核心对象。
  - HystrixCornma口dProperties properties：断路器对应 HystrixCornmand实例的属性对象， 它的详细内容我们将在后续章节做具体的介绍。
  - HystrixComrnandMetrics metrics ：用来让HystrixComrnand记录各类度量指标 的对象。
  - AtomicBoolean circuitOpen：断路器是否打开的 标志， 默认 为false
  - AtomicLong circui tOpenedOrLastTestedTime：断路器打开或是上 测试的时间戳。

HystrixCircuitBreakerimpl对HystrixCircuitBreaker接口的各个方法实现如下所示。

### 3.1 isOpen（）：判断断路器的 打开／关闭状态。 

- 如果断路器打开标识为true，贝。直接返回true，表示断路器处于打开状态。否则，就从度量指标对象metrics中获取healthCounts 统计对象做进一步判断（该对象记录了一个滚动时间窗内的请求信息快照，默认时间窗为10秒）

  - 如果它的请求总数（QPS ）在预设的阑值范围内就返回false，表示断路器处于未打开状态。该阑值的配置参数为circuitBreakerRequestVolumeThreshold,
    默认值为20。
  - 如果错误百分比在阀值范围内就返回false，表示断路器处于未打开状态。该阙
    值的配置参数为circuitBreakerErrorThresholdPercentage，默认值
    为 50 。
  - 如果上面的两个条件都不满足，则将断路器设置为打开状态（熔断／短路〉。 同时，如果是从关闭状态切换到打开状态的话，就将当前时间记录到上面提到的circuitOpenedOrLastTestedTime对象中。

  ```java
  public boolean isOpen (){
      if (circuitOpen.get()){
          return true;
      }
      HealthCounts health = metrics.getHealthCounts();
      if(health.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()){
          return false;
      }
      if (health.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()){
          return false;
      }else{
          if(circuitOpen.compareAndSer(false, true)){
              circuitOpenedOrLasTestedTime.set(System.currentTimeMillis());
              return true;
          }else{
              return true;
          }
      }
  }
  ```

#### 3.2 allowRequest（）：判断请求是否被允许

​	这个 实现非常简单。先根据配置对象properties中的断路器判断强制打开或关闭属性是否被设置 。如果强制打开，就直接返回false，拒绝请求。如果强制关闭，它会允许所有请求，但是同时也会调用isOpen （）来执行断路器的计算逻辑，用来模拟断路器打开／关闭的行为。在默认情况下，断路器并不会进入这两个强制打开或关闭的分支中去，而是通过！isOpen () || allowSingleTest（）来判断是否允许请求访问。！isOpen （）之前已经介绍过，用来判断和计算当前断路器是否打开， 如果是断开状态就允许请求。那么allowSingleTest（）是用来做什么的呢？

```java
@Override
public boolean allowRequest(){
    if(properties.circuitBreakerForceOpen().get()){
        return false;
    }
    if(properties.circuitBreakerForceClosed().get()){
        isOpen();
        return true;
    }
    return !isOpen() || allowSingLetest();
}
```

​	从allowSingleTest（） 的实现中我们可以看到，这里使用了在isOpen（）函数中当断路器从闭合到打开时候所记录的时间戳。当断路器在打开状态的时候，这里会判断断开时的时间戳＋配置中的circuitBreakerSleepWi 口dowinMilliseconds 时间是否小于当前时间，是的话，就 将当前时间更新到记录断路器打开的时间对象ircuitOpenedOrLastTestedTime 中，并且允许此次请求。简单地说， 通过 circuitBreakerSleepWindowinMilliseconds 属性设置了 一个断路器打开之后的休眠时间（默认为5秒〉，在该休眠时间到达之后，将再次允许请求尝试访问此时断路器处于 “ 半开” 状态， 若此时请求继续失败， 断路器又进入打开状态， 并继续等待下一个休眠窗口过去之后再次尝试；若请求成功， 则将断路器重新置于关闭状态。所以通过 allowSingleTest （）与 isOpen （）方法的配合，实现了断路器打开和关闭状态的切换。







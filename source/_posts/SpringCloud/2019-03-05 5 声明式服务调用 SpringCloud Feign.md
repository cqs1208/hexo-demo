---
layout: post
title: 05 声明式服务调用
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

通过前两章对 Spring Cloud Ribbon 和 Spring Cloud Hystrix 的介绍

<!-- more --> 

# 声明式服务调用 feign

​	通过前两章对 Spring Cloud Ribbon 和 Spring Cloud Hystrix 的介绍，我们已经掌握了开发微服务应用时的两个重磅武器， 学会了如何在微服务架构中实现客户端负载均衡的服务调用以及如何通过断路器来保护我们的微服务应用。这两者将被作为基础工具类框架广泛地应用在各个微服务的实现中，不仅包括我们自身的业务类微服务， 也包括 些基础设施类微服务（比如网关）。此外，在实践过程中，我们会发现对这两个框架的使用几乎是同时出现的。 既然如此，那么是否有更高层次的封装来整合这两个基础工具以简化开发呢？本章我们即将介绍的 Spring Cloud Feign 就是这样 个工具。它基于 Netflix Feign 实现，整合了Spring Cloud Ribbon与Spring Cloud Hystrix，除了提供这两者强大功能之外，它还提供了一种声明式的web服务客户端定义方式。

​	我们在使用 Spring Cloud Ribbon 时，通常都会利用它对 RestTemplate 的请求拦截来实现对依赖服务的接口调用， 而 RestTemplate 己经实现了对 HTTP 请求的封装处理， 形成了一套模板化的调用方法。在之前的例子中，我们只是简单介绍了 RestTemplate 调用的实现，但是在实际开发中，由于对服务依赖的调用可能不止于 个接口会被多处调用，所以我们通常都会针对各个微服务自行封装一些客户端类来包装这些依赖服务的调用。这
个时候我们会发现，由于 RestTemplate 的封装，几乎每一个调用都是简单的模板化内容。综合上述这些情况，Spring Cloud Feign 在此基础上做了进一步封装， 由它来帮助我们定义和实现依赖服务接口的定义。在SpringCloudFeign的实现下，我们只需要创建一个接口并用注解的方式来配置它， 即可完成对服务提供方的接口绑定，简化了在使用 Spring Cloud Ribbon 时自行封装服务调用客户端的开发量。Spring Cloud Feign 具备可插拔的注解支持，包括 Feign 注解和 JAX-RS 注解。同时，为了适应 Spring 的广大用户，它在 Netflix Feign的基础上扩展了对SpringMVC的注解支持。 这对于习惯于SpringMVC的开发者来说， 无疑是一个好消息， 因为这样可以大大减少学习使用它的成本。 另外， 对于Feign自身的一些主要组件， 比如编码器和解码器等， 它也以可插拔的方式提供， 在有需求的时候我们可以方便地扩展和替换它们。

## 1 快速入门

​	在本节中，我们将通过一个简单的示例来展现Spring Cloud Feign在服务客户端定义上 所带来的便利。 下面的示例将继续使用之前我们实现的hello-service服务， 这里我们 会通过Spring Cloud Feign提供的声明式服务绑定功能来实现对该服务接口的调用。

- 首先，创建一个Spring Boot基础工程， 取名为feign-consumer， 并在pom.xml 中引入spring-cloud-starter-eureka和spring-cloud-starter-feig口依赖。 具体内容如下所示：

  ```java
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
   </dependency>
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
   </dependency>
  ```

- 创建应用主类ApplicationFeign，并通过＠EnableFeignClients注解开启SpringCloud Feign的支持功能。

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
  @EnableFeignClients
  public class ApplicationFeign {
      public static void main(String[] args) {
          SpringApplication.run(ApplicationFeign.class);
      }
  }
  ```

- 定义HelloService接口，通过＠FeignClient注解指定服务名来绑定服务，然后再使用SpringMVC的注解来绑定具体该服务提供的REST接口。

  ```java
  @FeignClient(name= "hello-service")
  public interface ServiceClient {
      @RequestMapping(value = "/hello")
      public String hello();
  }
  ```

  **注意：** 这里服务名不区分大小写，所以使用hello-service和HELLO-SERVICE都是可以的。另外，在Brixton. SR5版本中，原有的serviceId属性已经被废弃，若要写属性名，可以使用name或value。

- 接着，创建一个ConsurnerController来实现对Feign客户端的调用。使用 @Autowired直接注入上面定义的ServiceClient实例，并在helloConsurner函数中调用这个绑定了hello-service服务接口的客户端来向该服务发起 /hello接口的调用。

  ```java
  @RestController
  public class UserController {
  
      @Autowired
      ServiceClient serviceClient;
  
      @RequestMapping("/feign-customer")
      public String hello() {
          return serviceClient.hello();
      }
  }
  ```

- 最后，同Ribbon实现的服务消费者一样，需要在application.properties中指定服务注册中心，并定义自身的服务名为feign-client，为了方便本地调 试与之前的Ribbon消费者区分，端口使用9001。

  ```java
  spring.application.name=feign-client
  server.port=9001
  eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
  ```

  ​	发送几次GET请求到http://localhost:9001/feign-consumer，可以得到如之前Ribbon实现时一样的效果，正确返回了“HelloWorld气并且根据控制台的输出，我们可以看到Feign实现的消费者，依然是利用Ribbon维护了针对HELLO-SERVICE的服务列表信息，并且通过轮询实现了客户端负载均衡。而与Ribbon不同的是，通过Feign我们
  只需定义服务绑定接口，以声明式的方法，优雅而简单地实现了服务调用。

## 2 参数绑定

​	在上一节的示例中，我们使用SpringCloud Feign实现的是一个不带参数的REST服务绑定。然而现实系统中的各种业务接口要比它复杂得多，我们会在HTTP的各个位置传入各种不同类型的参数，并且在返回请求响应的时候也可能是一个复杂的对象结构。在本节中，我们将详细介绍Feign中对几种不同形式参数的绑定方法

​	在开始介绍 Spring Cloud Feign 的参数绑定之前，我们先扩展一下服务提供方hello-service。增加下面这些接口定义，其中包含带有 Request参数的请求、带有 Header信息的请求、带有 RequestBody 的请求以及请求响应体中是一个对象的请求

```java
 @RequestMapping("/hello1")
    public String hello1(@RequestParam String name) throws  Exception{
        return "hello " +  name;
    }

    @RequestMapping("/hello2")
    public User hello2(@RequestHeader String name, @RequestHeader Integer age) throws  Exception{
        return new User(name, age);
    }
    @RequestMapping("/hello3")
    public String hello3(@RequestBody User user) throws  Exception{
       return "hello " + user.getName() + " , " + user.getAge();
    }
```

​	User 对象的定义如下，这里省略了 get 和 set 函数，需要注意的是，这里必须要有 User 的默认构造函数。不然， Spring Cloud Feign根据JSON 字符串转换 User 对象时会
抛出异常。

```java
public class User {
    private String name;
    private Integer age;

    public User() {
    }

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    
    //省略get和set方法
    。。。
        
     @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

​	完成了对 hello-service 的改造之后， 下面我们开始在快速入门示例的 feign-consumer 应用中实现这些新增的请求的绑定。

- 首先， 在 feign-consumer 中创建与上面一样的User类
- 然后， 在 HelloService 接口中增加对上述三个新增接口的绑定声明， 修改后，完成的 HelloService 接口如下所示：

```java
@RequestMapping(value = "/hello1")
    public String hello(@RequestParam(value = "name") String name);

    @RequestMapping("/hello2")
    public User hello2(@RequestHeader("name") String name, @RequestHeader("age") Integer age);

    @RequestMapping("/hello3")
    public String hello3(@RequestBody User user) ;
```

​	这里 定要注意， 在定义各参数绑定时，＠RequestPararn、＠RequestHeader等可以指定参数名称的注解， 它们的 value 千万不能少。 在 SpringMVC 程序中， 这些注解会根据参数名来作为默认值，但是在Feign中绑定参数必须通过 value 属性来指明具体的参数名，不然会抛出 IllegalStateExceptio口异常， value 属性不能为空。

- 最后，在ConsumerController中新增一个/feign-consumer2接口，来对本届新增的声明接口进行调用，修改后的完整代码如下所示：

```java
@RequestMapping("/feign/customer1")
    public String customer1() {
        StringBuilder sb = new StringBuilder();
        sb.append(serviceClient.hello()).append("\n");
        sb.append(serviceClient.hello1("DIDI")).append("\n");
        sb.append(serviceClient.hello2("DIDI", 30)).append("\n");
        sb.append(serviceClient.hello3(new User("DIDI", 30))).append("\n");
        return sb.toString();
    }
```

测试验证
在完成上述改造之后，启动服务注册中心、两个hello-service服务以及我们改造过的feign-consumero通过发送GET请求到http://localhost: 9001/feign-consumer2，触发HelloService对新增接口的调用。最终，我们会获得如下输出，代表接口绑定和调用成功。

```java
hello word
hello DIDI
User{name='DIDI', age=30}
hello DIDI , 30
```

## 3 继承特性

​	通过“快速入门”以及“参数绑定”小节中的示例实践，相信很多读者已经观察到，当使用SpringMVC的注解来绑定服务接口时，我们几乎完全可以从服务提供方的 Controller中依靠复制操作，构建出相应的服务客户端绑定接口。既然存在这么多复制操作， 我们自然需要考虑这部分内容是否可以得到进一步的抽象呢？在SpringCloud Feign中，针 对该问题提供了继承特性来帮助我们解决这些复制操作，以进一步减少编码量。下面，我 们详细看看如何通过SpringCloud Feign的继承特性来实现REST接口定义的复用。

- 为了能够复用DTO与接口定义， 我们先创建一个基础的Maven工程， 命名为hello-service-api。
- 由于在hello-service-api中需要定义可同时复用于服务端与客户端的接口，我们要使用到SpringMVC的注解， 所以在pom.xml 中引入spring-bootstarter-web依赖， 
- 将上一节中实现的User 对象复制到hello-service-api 工程中， 比如保存到com.didispace.dto.User 。
- 在hello-service-api 工程中创建com.didispace.service.HelloService接口，内容如下，该接口中的User 对象为本项目中的com.didispace.dto.User 。
- 将上一节中实现的User 对象复制到com.didispace.dto.User 。创建com.didispace.service.HelloService 接口， 内容如下， 该接口中的User对象为本项目中的com.didispace.dto.User 。

```java
//@RequestMapping("/refactor")
public interface RefactorServiceClient {

    @RequestMapping(value = "/hello4")
    public String hello4(@RequestParam("name") String name);

    @RequestMapping("/hello5")
    public User hello5(@RequestHeader("name") String name, @RequestHeader("age") Integer age);

    @RequestMapping("/hello6")
    public String hello6(@RequestBody User user) ;
}
```

​	因为后续还会通过之前的hello-service和feign-consumer 来重构， 所以为了避免接口混淆， 在这里定义HelloService 时， 除了头部定义了／rafactor 前缀之外，同时将提供服务的三个接口更名为／ hello4 、／ hello5 、／hello6。

- 下面对hello-service 进行重构， 在pom.xml 的dependency 节点中， 新增对hello-service-api 的依赖。

  ```java\
  <dependency>
              <groupId>com.springcloud-feign</groupId>
              <artifactId>hello-service-api</artifactId>
              <version>1.0-SNAPSHOT</version>
          </dependency>
  ```

- 创建RefactorHelloController 类继承hello-service-api 中定义的HelloService 接口， 并参考之前的HelloController 来实现这三个接口， 具体内容如下所示：

  ```java
  @RestController
  public class UserController implements RefactorServiceClient {
      Logger logger = LoggerFactory.getLogger(UserController.class);
  
      @Override
      public String hello4( String name){
          return "hello " +  name;
      }
  
      @Override
      public User hello5(@RequestHeader("name") String name, @RequestHeader("age")  Integer age){
          return new User(name, age);
      }
  ```

  ​	我们可以看到通过继承的方式， 在Controller 中不再包含以往会定义的请求映射注解@RequestMappi 均， 而参数的注解定义在重写的时候会自动带过来。在这个类中， 除了要实现接口逻辑之外， 只需再增加＠ RestController 注解使该类成为一个REST 接口类就大功告成了

- 完成了服务提供者的重构， 接下来在服务消费者feign-consumer 的pom.xml文件中， 如在服务提供者中一样， 新增对hello-service-api 的依赖。

- 创建RefactorHelloService 接口， 并继承hello-service-api 包中的HelloService 接口， 然后添加＠ FeignClient 注解来绑定服务。

  ```java
  @FeignClient(name= "spring-cloud-producer")
  public interface ServiceClient extends  RefactorServiceClient{
  }
  ```

- 最后， 在ConsumerController 中， 注入RefactorHelloService 的实例，并新增一个请求／ feign-consumer3 来触发对RefactorHelloService的实例的调用。

  ```java
   @RequestMapping("/feign/customer1")
      public String customer1() {
          StringBuilder sb = new StringBuilder();
          sb.append(serviceClient.hello4("DIDI")).append("\n");
          sb.append(serviceClient.hello5("DIDI", 30)).append("\n");
          sb.append(serviceClient.hello6(new User("DIDI", 30))).append("\n");
          return sb.toString();
      }
  ```

测试验证：

这次的验证过程需要注意几个工程的构建顺序， 由于hello－ 民主vice 和feign-consumer 都依赖hello-service-api 工程中的接口和DTO 定义， 所以必须先构建hello-service-api 工程，然后再构建hello-service 和feig口－ consumer。接着我们分别启动服务注册中心， hello-service 和feign一consumer， 并访问http://localhost:9001/feign-consumer3， 调用成功后可以获得如下输出

```java
hello DIDI
User{name='DIDI', age=30}
hello DIDI , 30
```

优点和缺点：

​	使用Spring Cloud Feign 继承特性的优点很明显， 可以将接口的定义从Controller 中剥离， 同时配合 Maven 私有仓库就可以轻易地实现接口定义的共享， 实现在构建期的接口绑定， 从而有效减少服务客户端的绑定配置。 这么做虽然可以很方便地实现接口定义和依赖的共享， 不用再复制粘贴接口进行绑定， 但是这样的做法使用不当的话会带来副作用。 由于接口在构建期间就建立起了依赖， 那么接口变动就会对项目构建造成影响， 可能服务提供方修改了 个接口定义， 那么会直接导致客户端工程的构建失败。所以， 如果开发团队通过此方法来实现接口共享的话， 建议在开发评审期间严格遵守面向对象的开闭原则， 尽可能地做好前后版本的兼容，防止牵一发而动全身的后果，增加团队不必要的维护工作量。

## 4 Ribbon配置

​	由于 Spring Cloud Feign 的客户端负载均衡是通过 Spring Cloud Ribbon 实现的，所以我们可以直接通过配置 Ribbon 客户端的方式来自定义各个服务客户端调用的参数。那么我们 如何在使用 Spring Cloud Feign 的工程中使用 Ribbon 的配置呢？

### 4.1 全局配置

​	全局配置的方法非常简单， 我们可以直接使用 ribbon.<key>=<value＞的方式来设置且bbon 的各项默认参数。 比如， 修改默认的客户端调用超时时间：

```java
ribbon.ConnectTimeout=500
ribbon.ReadTimeout=500
```

### 4.2 指定服务配置

​	大多数情况下，我们对于服务调用的超时时间可能会根据实际服务的特性做一些调整，所以仅仅依靠默认的全局配置是不行的。在使用 Spring Cloud Feign 的时候，针对各个服务客户端进行个性化配置的方式与使用 Spring Cloud Ribbon 时的配置方式是 样的， 都采用<client>.ribbon.key=value 的格式进行设置。但是，这里就有一个疑问了，<client>所指定的Ribbon客户端在哪里呢？

​	回想 下， 在定义 Feign 客户端的时候， 我们使用了＠ FeignClient 注解。在初始化过程中， Spring Cloud Feign 会根据该注解的 name 属性或 value 属性指定的服务名，自动创建一个同名的 Ribbon 客户端。也就是说，在之前的示例中，使用＠ FeignClient(value=“HELLO-SERVICE”)来创建 Feign 客户端的时候 ， 同时 也创建了 个名为HELLO-SERVICE 的 Ribbon 客户端。既然如此， 我们就可以使用＠ Feig口Client 注解中的 name 或 value 属性值来设置对应的 Ribbon 参数， 比如：

```java
HELLO-SERVICE.ribbon.ConnectTimeout=500
HELLO-SERVICE.ribbon.ReadTimeout=2000
HELLO-SERVICE.ribbon.OkToRetryOnAllOperations=true
HELLO-SERVICE.ribbon.MaxAutoRetryesNextServer=2
HELLO-SERVICE.ribbon.MaxAutoRetries=1
```

### 4.3 重试机制

​	在 Spring Cloud Feign 中默认实 现了请求的重试机制， 而上面我 们对 于 HELLO-SERVICE 客户端的配置内容就是对于请求超时以及重试机制配置的详情， 具体内一节关于 Spring Cloud Ribbon 重试机制的介绍。 我们可以通过修改之前的示例做一些验证。

- 在hello-service 应用的／hello 接口实现中， 增加一些随机延迟，比如：

```java
  @Override
    public String hello4( String name) {
        ServiceInstance serviceInstance = discoveryClient.getLocalServiceInstance();
        int port = serviceInstance.getPort();
        Long threadId = Thread.currentThread().getId();
        logger.info("hello4 sleepTime *********** start  " + threadId + " ***********" );
        int sleepTime = 0;
        try {
            sleepTime = new Random().nextInt(3000);
            logger.info("hello4 sleepTime " + sleepTime);
            Thread.sleep(sleepTime);
        } catch (Exception e) {
            e.printStackTrace();
        }
        logger.info("hello4 sleepTime ********* end  " + threadId + "  ***********" );
        return "hello " +  name;
    }
```

- 在feign-consumer应用中增加上文中提到的重试配置参数。其中，由于 HELLO­SERVICE.ribbon.MaxAutoRetries 设置为 1，所以重试策略先尝试访问首选实例一次，失败后才更换实例访问， 而更换实例访问的次数通过 HELLO-SERVICE.ribbon.MaxAutoRetriesNextServer 参数设置为 2，所以会尝试更换两次实例进行重试。

- 最后， 启动这些应用， 并尝试访问几次 http://localhost:9001/feign­consumer 接口。 当请求发生超时的时候，我们在 hello-service 的控制台中可 一定每次相同）：能会获得如下输出内容（由于 sleepTime 的随机性， 并不一定每次相同

  ![ribbon重试机制](/images/SpringCloud/SpringCloud_ribbon1.png)

​         从控制台输出中，我们可以看到这次访问的第一次请求延迟时间为 2766毫秒，由于超时时间设置为 2000 毫秒，Feign 客户端发起了重试，第二次请求的延迟为 624 秒，没有超时。Feign客户端在进行服务调用时，虽然经历了一次失败，但是通过重试机制，最终还是获得了请求结果。所以，对于重试机制的实现，对于构建高可用的服 务集群来说非常重要，而SpringCloud Feign也为其提供了足够的支持。

​         这里需要注意一点，Ribbon的超时与Hystrix的超时是两个概念。为了让上述实现有效，我们需要让Hystrix的超时时间大于Ribbon的超时时间，否则Hystrix命令超时后，该命令直接熔断，重试机制就没有任何意义了。

## 5 Hystrix配置

​	在SpringCloud Feign中，除了引入了用于客户端负载均衡的SpringCloud Ribbon之外，还引入了服务保护与容错的工具Hystrix。默认情况下，SpringCloud Feign会为将所有Feign 客户端的方法都封装到Hystrix命令中进行服务保护。在上一节末尾，我们介绍重试机制的配置时，也提到了关于Hystrix的超时时间配置。那么在本节中，我们就来详细介绍一下，
如何在使用SpringCloud Feign时配置Hys仕ix属性以及如何实现服务降级。

### 5.1 全局配置

​	对于Hystrix的全局配置同SpringCloud Ribbon的全局配置一样，直接使用它的默认配置前缀hystrix.command.default就可以进行设置，比如设置全局的超时时间：

```java
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=5000
```

​	另外，在对Hystrix进行配置之前，我们需要确认feign.hystrix.enabled参数没有被设置为false，否则该参数设置会关闭Feign客户端的Hystrix支持。而对于我们之前测试重试机制时，对于Hystrix的超时时间控制除了可以使用上面的配置来增加熔断超时时 间，也可以通过feign.hystrix.enabled=false来关闭Hystrix功能，或者使用 hystrix.command.default.execution.timeout.enabled=false来关闭熔断功能

### 5.2 禁用Hystrix

​	上文我们提到了，在SpringCloud Feign中，可以通过feign.hystrix. enabled=false来关闭Hystrix功能。另外，如果不想全局地关闭Hystrix支持，而只想针对某个服务客户端关闭Hystrix支持时，需要通过使用＠Scope（”prototype”）注解为指定的客户端配置Feign.Builder实例，详细实现步骤如下所示。

- 构建一个关闭Hystrix的配置类。

  ```java
  public class DisableHystrixConfiguration {
      @Bean
      @Scope("prototype")
      public Feign.Builder feignBuilder(){
          return Feign.builder();
      }
  }
  ```

- 在HelloService的@FeignClient注解中，通过configuration参数引入上面实现的配置。

  ```java
  @FeignClient(name= "spring-cloud-producer", configuration = DisableHystrixConfiguration.class)
  public interface ServiceClient extends  RefactorServiceClient{
      //
  }
  ```

### 5.3 指定命令配置

​	对于Hys位ix命令的配置，在实际应用时往往也会根据实际业务情况制定出不同的配置方案。配置方法也跟传统的Hystrix命令的参数配置相似，采用hystrix.cornmand. <cornmandKey＞作为前缀。而＜cornmandKey＞默认情况下会采用Feign客户端中的方法名作为标识，所以，针对上一节介绍的尝试机制中对／hello接口的熔断超时时间的配置可以通过其方法名作为＜commandKey＞来进行配置，具体如下：

```java
hystrix.command.hello4.execution.isolation.thread.timeoutInMilliseconds=5000
```

​	在使用指定命令配置的时候，需要注意，由于方法名很有可能重复，这个时候相同方法名的Hystrix配置会共用，所以在进行方法定义与配置的时候需要做好一定的规划。当然，也可以重写Feign.Builder的实现，并在应用主类中创建它的实例来覆盖自动化配置的 HystrixFeign.Builder实现。

### 5.4 服务降级配置

​	Hystrix提供的服务降级是服务容错的重要功能，由于SpringCloud Feign在定义服务客户端的时候与SpringCloud Ribbon有很大差别，HystrixCommand定义被封装了起来，我们无法像之前介绍SpringCloud Hystrix时，通过＠HystrixCornmand注解的fallback参数那样来指定具体的服务降级处理方法。但是，SpringCloud Feign提供了另外一种简单的定义
方式，下面我们在之前创建的feign-consumer工程中进行改造。

- 服务降级逻辑的实现只需要为Feign客户端的定义接口编写一个具体的接口实现类。比如为HelloService接口实现一个服务降级类helloServiceFallback,其中每个重写方法的实现逻辑都可以用来定义相应的服务降级逻辑，具体如下：

```java
@component
public class HelloServiceFallback implements ServiceClient {
    @Override
    public String hello() {
        return "error";
    }
    @Override
    public User hello2(String name, Integer age) {
        return new User("未知", 0);
    }
}
```

- 在服务绑定接口HelloService中，通过＠FeignClient注解的fallback属性来指定对应的服务降级实现类。

  ```java
  @FeignClient(name= "spring-cloud-producer", fallback = HelloServiceFallback.class)
  public interface ServiceClient extends  RefactorServiceClient{
      //
      @RequestMapping(value = "/hello")
      public String hello();
  }
  ```

测试验证：

​	下面我们来验证一下服务降级逻辑的实现。启动服务注册中心和feign-consumer, 但是不启动hello-service服务。发送GET请求到http://localhost:9001/feign-co口sumer2，该接口会分别调用HelloService中的4个绑定接口，但因为hello-service服务没有启动，会直接触发服务降级，并获得下面的输出内容：

```java
error
error
name=未知，age=0
error
```

​	正如我们在HelloServiceFallback类中实现的内容，每一个服务接口的断路器实 际就是实现类中的重写函数的实现。

注意：在Brixton.SR5版本中，fallback的实现函数中不再支持返回com.netflix.hystrix.HystrixCommand和rx.Observable类型的异步执行和响应式执行方式。

## 6 其他配置

### 6.1 请求压缩

​	Spring Cloud Feign支持对请求与响应进行GZIP压缩，以减少通信过程中的性能损耗。 我们只需通过下面两个参数设置，就能开启请求与响应的压缩功能：

```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

​	同时，我们还能对请求压缩做一些更细致的设置，比如下面的配置内容指定了压缩的请求数据类型，并设置了请求压缩的大小下限，只有超过这个大小的请求才会对其进行压缩。

```java
feign.compression.request.enabled=true
feign.compression.reauest.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

​	上述配置的feign.compression.request.mime-types和feign.compression. request.min-request-size均为默认值。

## 7 日志配置

​	Spring Cloud Feign在构建被＠Feig口Client注解修饰的服务客户端时，会为每个客户端都创建一个feign.Logger实例，我们可以利用该日志对象的DEBUG模式来帮助分析Feign的请求细节。可以在applicatio口.properties文件中使用logging.level. <FeignClient＞的参数配置格式来开启指定Feign客户端的DEBUG日志，其中<FeignClient＞为Feign客户端定义接口的完整路径，比如针对本章中我们实现的HelloService可以按如下配置开启：

```java
logging.level.com.springcloud.service.ServiceClient=DEBUG
```

​	但是，只是添加了如上配置，还无法实现对DEBUG日志的输出。这时由于Feign客 户端默认的Logger.Level对象定义为NONE级别，该级别不会记录任何Feign调用过程中的信息，所以我们需要调整它的级别，针对全局的日志级别，可以在应用主类中直接加
入Logger.Level的Bean创建，具体如下：

```java

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ApplicationFeign {
    
    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
    
    public static void main(String[] args) {
       SpringApplication.run(ApplicationFeign.class);
    }
}
```

​	当然也可以通过实现配置类，然后在具体的Feign客户端来指定配置类以实现是否要调整不同的日志级别，比如下面的实现：

```java
public class DisableHystrixConfiguration {
    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}


@FeignClient(name= "spring-cloud-producer", configuration = DisableHystrixConfiguration.class)
public interface ServiceClient extends  RefactorServiceClient{
}
```

​	在调整日志级别为FULL之后，我们可以再访问下之前的http://localhost: 9001/feign-consurner接口，这时我们在feign-co口sumer的控制台中就可以看到类似下面的请求详细日志：

![feignLogger](/images/SpringCloud/SpringCloud_feignLogger1.png)

​	对于Feign的Logger级别主要有下面4类，可根据实际需要进行调整使用。

- NONE：不记录任何信息。
- BASIC：仅记录请求方法、URL以及响应状态码和执行时间。
- HEADERS：除了记录BASIC级别的信息之外，还会记录请求和响应的头信息。
- FULL：记录所有请求与响应的明细，包括头信息、请求体、元数据等。
---
layout: post
title: 02 服务治理
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件中的 部分

<!-- more --> 

# 服务治理 Spring Cloud Eureka

​	Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件中的 部分， 它基于 Netflix Eureka 做了二次封装， 主要负责完成微服务架构中的服务治理功能。 Spring Cloud 通过为Eureka 增加了 Spring Boot 风格的自动化配置，我们只需通过简单引入依赖和注解配置就能让 Spring Boot 构建的微服务应用轻松地与 Eureka 服务治理体系进行整合。

​	在本章中， 我们将指引读者学习下面这些核心内容， 并构建起用于服务治理的基础设施

- 构建服务注册中心
- 服务注册与服务发现
- Eureka 的基础架构
- Eureka 的服务治理机制
- Eureka 的配置

## 1 服务治理

​	服务治理可以说是微服务架构中最为核心和基础的模块， 它主要用来实现各个微服务 实例的自动化注册与发现。 为什么我们在微服务架构中那么需要服务治理模块呢？微服务系统没有它会有什么不好的地方吗？

​	在最初开始构建微服务系统的时候可能服务并不多， 我们可以通过做 些静态配置来完成服务的调用。 比如，有两个服务A和B，其中服务A需要调用服务B来完成 业务
操作时， 为了实现服务B的高可用， 不论采用服务端负载均衡还是客户端负载均衡， 都需
要手工维护服务B的具体实例清单。 但是随着业务的发展， 系统功能越来越复杂， 相应的微服务应用也不断增加， 我们的静态配置就会变得越来越难以维护。 并且面对不断发展的业务， 我们的集群规模、 服务的位置 、 服务的命名等都有可能发生变化， 如果还是通过手工维护的方 式， 那么极易发生错误或是命名冲突等问题。 同时， 对于这类静态内容的维护也必将消耗大量的人力。
	为了解决微服务架构中的服务实例维护问题， 产生了大量的服务治理框架和产品。 这些框架和产品的实现都围绕着服务注册与服务发现机制来完成对微服务应用实例的自动化管理。

- **服务注册：** 在服务治理框架中， 通常都会构建 个注册中心， 每个服务单元向注册一
  中心登记自己提供的服务， 将主机与端口号、 版本号、 通信协议等 些附加信息告知注册中心， 注册中心按服务名分类组织服务清单。 比如， 我们有两个提供服务A的进程分别运行于192.168.0.100:8000和192.168.0.101:8000位置上，另外还有三个提供服务B的进程分别运行于 192.168.0.100:9000 、 192.168.0.101:9000、 192.168.0.102:9000位置上。 当 这些进程均启动，并向注册中心注册自己的服务之后， 注册中心就会维护类似下面的 个服务清单。另外， 服务注册中心还需要以心跳的方式去监测清单中的服务是否可用， 若不可用需要从服务清单中剔除， 达到排除故障服务的效果。

  ![eurekaServerList](/images/SpringCloud/SpringCloud_eurekaServerList.png)

- **服务发现：** 由于在服务治理框架下运作， 服务间的调用不再通过指定具体的实例地址来实现，而是通过向服务名发起请求调用实现。 所以， 服务调用方在调用服务提供方接口的时候， 并不知道具体的服务实例位置。 因此， 调用方需要向服务注册中心咨询服务， 并获取所有服务的实例清单， 以实现对具体服务实例的访问。 比如，现有服务 C 希望调用服务 A，服务 C就需要向注册中心发起咨询服务请求，服务注册中心就会将服务 A的位置清单返回给服务 C，如按上例服务 A 的情况， C便获得 了服务A的两个可用位置 192.168.0.100:8000和192.168.0.101:8000。当服务C要发起调用的时候，便从清单中已某种轮询策略取出一个位置来进行服务调用，这就是后续我们将会介绍的客户端负载均衡。这里我们只是列举了一种简单的服务治理逻辑，以方便理解服务治理框架的基本运行思路。 实际的框架为了性能等因素， 不会采用每次都向服务注册中心获取服务的方式， 并且不同的应用场景在缓存和服务剔除等机制上也会有一些不同的实现策略。

## 2 Netflix Eureka

​	Spring Cloud Eureka， 使用 Netflix Eureka 来实现服务注册与发现， 它既包含了服务端组件，也包含了客户端组件，并且服务端与客户端均采用Java编写，所以Eureka主要适用于通过Java实现的分布式系统，或是与NM兼容语言构建的系统。但是，由于Eureka服 务端的服务治理机制提供了完备的RESTfulAPI，所以它也支持将非Java语言构建的微服 务应用纳入Eureka的服务治理体系中来。只是在使用其他语言平台的时候，需要自己来实现Eureka的客户端程序。不过庆幸的是，在目前几个较为流行的开发平台上，都己经有了一些针对Eureka注册中心的客户端实现框架，比如.NET平台的Steeltoe、Node.的eureka-js-client等。
	Eureka服务端，我们也称为服务注册中心。它同其他服务注册中心一样，支持高可用配置。它依托于强一致性提供良好的服务实例可用性，可以应对多种不同的故障场景。如果Eureka以集群模式部署，当集群中有分片出现故障时，那么Eureka就转入自我保护模式。它允许在分片故障期间继续提供服务的发现和注册，当故障分片恢复运行时，集群中的其他分片会把它们的状态再次同步回来。以在AWS上的实践为例，Netflix推荐每个可用的区域运行一个Eureka服务端，通过它来形成集群。不同可用区域的服务注册中心通过异步模式互相复制各自的状态，这意味着在任意给定的时间点每个实例关于所有服务的状态是有细微差别的。

​	Eureka客户端，主要处理服务的注册与发现。客户端服务通过注解和参数配置的方式，嵌入在客户端应用程序的代码中，在应用程序运行时，Eureka客户端向注册中心注册自身提供的服务并周期性地发送心跳来更新它的服务租约。同时，它也能从服务端查询当前注册的服务信息并把它们缓存到本地并周期性地刷新服务状态。

​	下面我们来构建一些简单示例，学习如何使用Eureka构建注册中心以及进行注册与发 现服务。

## 3 创建服务注册中心

​	首先，创建一个基础的SpringBoot工程，命名为eureka-server，并在pom.xml 中引入必要的依赖内容，代码如下：

```java
 <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
    </dependencies>
    <dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```

通过@EnableEurekaServer注解启动一个服务注册中心提供给其他应用进行对话。

```java 
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
 
	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

​	在默认设置下， 该服务注册中心也会将自己作为客户端来尝试注册它自己， 所以我们需要禁用它的客户端注册行为，只需在application.properties中增加如下配置：

```java
server:
  port: 1111
eureka:
  instance:
    hostname: localhost
  client:
    # 表示是否注册自身到eureka服务器
    # register-with-eureka: false
    # 是否从eureka上获取注册信息
    # fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

​	由于后续内容也都会在本地运行， 为了与后续要进行注册的服务区分， 这里将服务注册中心的端口通过 server.port 属性设置为 1111。

- eureka.client.register-with-eureka：由于该应用为注册中心，所以设置 为false，代表不向注册中心注册自己。
- eureka.client.fetch-registry： 由于注册中心的职责就是维护服务实例，它并不需要去检索服务，所以也设置为false

在完成了上面的配置后，启动应用并访问http://localhost:1111／。可以看到如
下图所示的Eureka信息面板，其中Instancescurrently registered with Eureka栏是空的，说
明该注册中心还没有注册任何服务。

## 4 注册服务提供者

​	在完成了服务注册中心的搭建之后，接下来我们尝试将一个既有的Spring Boot应用加
入Eureka的服务治理体系中去。
	可以使用上一章中实现的快速入门工程来进行改造， 将其作为一个微服务应用向服务
注册中心发布自己。首先， 修改pom.xml， 增加Spring Cloud Eureka模块的依赖， 具体代
码如下所示：

```java
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
    	<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    
        <dependency>
			<groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
    </dependencies>
    <dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```

​	接着， 改造／hello请求处理接口， 通过注入DiscoveryClient对象， 在日志中打印出服务的相关内容。

```java
@RestController
public class UserController {

    @Autowired
    private DiscoveryClient client;

    @RequestMapping("/hello")
    public String index(@RequestParam String name) {
        ServiceInstance instance = client.getLocalServiceInstance();
        return "hello "+name+"，is return message and port is: " + instance.getHost()  +"  server_id is: "+ instance.getServiceId() ;
    }
}
```

​	然后， 在主类中通过加上＠EnableDiscoveryClient注解， 激活Eureka中的
DiscoveryClient实现〈自动化配置， 创建DiscoveryClient接口针对Eureka客户
端的EurekaDiscoveryClient实例〉， 才能实现上述Con位oller中对服务信息的

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ApplicationServerProvider {
    public static void main(String[] args) {
        SpringApplication.run(ApplicationServerProvider.class, args);
    }
}
```

​	最后， 我们需要在 application.properties 配置文件中， 通过 spring. application ．口ame 属性来为服务命名， 比如命名为 hello-service。再通过 eureka .client.serviceUrl.defaultZone 属性来指定服务注册中心的地址， 这里我们指定为之前构建的服务注册中心地址， 完整配置如下所示：

```java
spring.application.name=hello-service
eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka
```

​	下面我们分别启动服务注册中心以及这里改造后的 hello-service 服务。 在 hello-service服务控制台中，Torr则启动之后，com.netflix.discovery.DiscoveryClient 对象打印了该服务的注册信息，表示服务注册成功。

​	而此时在服务注册中心的控制台中，可以看到类似下面的输出，名为hello-service 的服务被注册成功了。

## 5 高可用注册中心

​	在微服务架构这样的分布式环境中，我们需要充分考虑发生故障的情况，所以在生产环境中必须对各个组件进行高可用部署，对于微服务如此，对于服务注册中心也一样。但 是到本节为止，我们一直都在使用单节点的服务注册中心，这在生产环境中显然并不合适， 我们需要构建高可用的服务注册中心以增强系统的可用性。

​	Eureka Server的设计一开始就考虑了高可用问题，在Eureka的服务治理设计中，所有 节点即是服务提供方，也是服务消费方，服务注册中心也不例外。是否还记得在单节点的配置中，我们设置过下面这两个参数，让服务注册中心不注册自己：

```java
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

​	Eureka Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这 样就可以形成一组互相注册的服务注册中心，以实现服务清单的互相同步，达到高可用的效果。下面我们就来尝试搭建高可用服务注册中心的集群。可以在本章第1节中实现的服 务注册中心的基础之上进行扩展，构建一个双节点的服务注册中心集群。

```java
eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/   # 相互注册
```

## 6 服务发现与消费

​	通过上面的内容介绍与实践， 我们已经搭建起微服务架构中的核心组件 服务注册 中心〈包括单节点模式和高可用模式〉。 同时， 还对上一章中实现的 Spring Boot 入门程序做了改造。 通过简单的 配置，使该程序注册到 Eureka注册中心上，成为该服务治理体系下的一个服务。命名为hello-service。 现在我们已经有了服务注册中心和服务提供者，下面就来尝试构建一个服务消费者，它主要完成两个目标，发现服务以及消费服务。其中服务发现的任务由Eureka的客户端完成， 而服务消费的任务由Ribbon完成。Ribbon是一个基于HTTP 和 TCP 的客户端负载均衡器，它可以在通过客户端中配置的 ribbonServerList 服务端列表去轮询访问以达到均衡负载 的作用。 当Ribbon与Eureka联合使用时，Ribbon 的服务实例清单 RibbonServerList 会被DiscoveryEnabledNIWSServerList重写， 扩展成 从Eureka注册中心中获取服务端列表。 同时它也会用 NIWSDiscoveryPing
来取代IPing，它将职责委托给Eureka来确定服务端是否己经启动。 在本章中， 我们对Ribbon不做详细的介绍，读者只需要理解它在Eureka服务发现的基础上，实现了 一套对 服
务实例的选择策略，从而实现对服务的 消费。下一章我们会对Ribbon做详细的介绍和分析。

​	下面我们通过构建一 个简单的示例，看看在Eureka的服务治理体系下如何实现服务的 发现与消费。

- 启动两个生产方

  ```java
  --server.port=9091
  --server.port=9092   
  ```

- 启动ribbon消费方

  ```java
   <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-ribbon</artifactId>
   </dependency>
  ```

- 创建应用主类ConsumerApplication， 通过＠ EnableDiscoveryClient 注解让该应用注册为Eureka 客户端应用， 以获得服务发现的能力。同时， 在该主类中创
  建RestTemplate 的Spring Bean 实例，并通过＠ LoadBalanced 注解开启客户端
  负载均衡

  ```java
  @SpringBootApplication
  @EnableDiscoveryClient
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

- 创建ConsumerController类并实现／ribbon-consumer接口。在该接口中，
  通过在上面创建的RestTemplate 来实现对HELLO-SERVICE服务提供的
  /hello接口进行调用。 可以看到这里访问的址地 是服务名EH LL-O SEVR IC，E而
  不是一个具体的地址在服， 务治理框架中这， 是一个非常重要的性特 也符， 合在本
  章一开始对服务治理的解释。

  ```java
  @RestController
  public class ConsumerController {
  
      @Autowired
      RestTemplate restTemplate;
  
      @RequestMapping(value = "/hello/{name}", method = RequestMethod.GET)
      public String hello(@PathVariable("name") String name) {
          Map<String, String> map = new HashMap<>();
          map.put("name", name);
        //  return restTemplate.getForEntity("http://spring-cloud-producer/hello",String.class , map).getBody();
          return restTemplate.getForEntity("http://spring-cloud-producer/hello?name={name}", String.class, map).getBody();
      }
  
  ```

- 在application.properties中配置Eureka服务注册中心的需位置要与之前，
  的HELLO-SERVICE一样不， 然是发现不了该 服务的同， 时设置该消费者的口端 为
  9000 ，不能与之前启动的应用端口冲突。

  ```java
  spring.application.name=ribbon-client
  server.port=8030
  eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
  ```

## 7 Eureka详解

​	在上一节中， 我们通过一个简单的服务注册与发现示例，构建了Eureka 服务治理体系中的三个核心角色： 服务注册中心、服务提供者以及服务消费者。通过上述示例， 相信读
者对于Eureka 的服务治理机制已经有了一些初步的认识。至此， 我们已经学会了如何构建
服务注册中心（包括单节点和高可用部署）， 也知道了如何使用Eureka 的注解和配置将
Spring Boot 应用纳入Eureka 的服务治理体系，成为服务提供者或是服务消费者。同时，对
于客户端负载均衡的服务消费也有了一些简单的接触。但是， 在实践中， 我们的系统结构
往往都要比上述示例复杂得多， 如果仅仅依靠之前构建的服务治理内容， 大多数情况是无
法完全直接满足业务系统需求的， 我们还需要根据实际情况来做一些配置、调整和扩展。
所以， 在本节中， 我们将详细介绍Eureka 的基础架构、节点间的通信机制以及一些进阶的
配置等。

### 7.1 基础架构

​	在“ 服务治理”一节中， 我们所讲解的示例虽然简单， 但是麻雀虽小、五脏俱全。它
己经包含了整个Eureka 服务治理基础架构的三个核心要素。

- **服务注册中心：**Eureka 提供的服务端， 提供服务注册与发现的功能， 也就是在上节中我们实现的eureka-servero
- **服务提供者：** 提供服务的应用，可以是Spring Boot 应用，也可以是其他技术平台且遵循Eureka 通信机制的应用。它将自己提供的服务注册到Eureka，以供其他应用发现， 也就是在上一节中我们实现的HELLO-SERVICE应用。

- **服务消费者：** 消费者应用从服务注册中心获取服务列表， 从而使消费者可以知道去何处调用其所需要的服务，在上一节中使用了Ribbon 来实现服务消费，另外后续还会介绍使用Feign 的消费方式。

### 7.2 服务治理机制

​	在体验了Spring Cloud Eureka 通过简单的注解配置就能实现强大的服务治理功能之后， 我们来进一步了解一下Eureka 基础架构中各个元素的一些通信行为，以此来理解基于Eureka 实现的服务治理体系是如何运作起来的。以下图为例， 其中有这样几个重要元素：

-  “服务注册中心－1 ＇＇和“ 服务注册中心－2飞它们互相注册组成了高可用集群。
- “ 服务提供者” 启动了两个实例， 一个注册到“ 服务注册中心，1 ” 上， 另外一个注册到“ 服务注册中心－2 ” 上。

- 还有两个“ 服务消费者气它们也都分别只指向了一个注册中心。

![eurekaServer](/images/SpringCloud/SpringCloud_eurekaServer.png)

​	根据上面的结构， 下面我们来详细了解一下， 从服务注册开始到服务调用， 及各个元素所涉及的一些重要通信行为。

### 7.3 服务提供者

#### 7.31 服务注册

​	“服务提供者” 在启动的时候会通过发送REST请求的方式将自己注册到Eureka Server上， 同时带上了自身服务的一些元数据信息。Eureka Server接收到这个REST请求之后，将元数据信息存储在一个双层结构Map中， 其中第一层的key是服务名， 第二层的key是具体服务的实例名。（我们可以回想一下之前在实现Ribbon负载均衡的例子中， Eureka信息面板中一个服务有多个实例的情况， 这些内容就是以这样的双层Map形式存储的。）

​	在服务注册时， 需要确认一下eureka.client.register-with-eureka=true参数是否正确， 该值默认为true。若设置为fals e将不会启动注册操作。

#### 7.3.2  服务同步

​	如架构图中所示， 这里的两个服务提供者分别注册到了两个不同的服务注册中心上，也就是说， 它们的信息分别被两个服务注册中心所维护。此时， 由于服务注册中心之间因互相注册为服务， 当服务提供者发送注册请求到一个服务注册中心时， 它会将该请求转发给集群中相连的其他注册中心， 从而实现注册中心之间的服务同步。通过服务同步， 两个服务提供者的服务信息就可以通过这两台服务注册中心中的任意一台获取到

#### 7.3.3 服务续约

​	在注册完服务之后，服务提供者会维护一个心跳用来持续告诉EurekaServer： “ 我还活着” ，以防止Eureka Server 的“ 剔除任务” 将该服务实例从服务列表中排除出去， 我们称该操作为服务续约（Renew）。

​	关于服务续约有两个重要属性， 我们可以关注并根据需要来进行调整：

```java
eureka.instance.lease-renewal-interval-in-seconds=30
eureka.instance.lease-expiration-duration-in-seconds=90
```

- eureka.instance.lease-renewal-i川erval-in-seconds : 服务续约任务的调用间隔时间，默认为30秒。
- eureka.instance.lease-expirationduration-in-seconds: 服务失效的时间， 默认为90秒。

### 7.4 服务消费者

#### 7.4.1 获取服务

​	到这里， 在服务注册中心已经注册了一个服务， 并且该服务有两个实例。当我们启动服务消费者的时候， 它会发送一个REST请求给服务注册中心， 来获取上面注册的服务清单。为了’性能考虑， Eureka Server会维护一份只读的服务清单来返回给客户端， 同时该缓存清单会每隔30秒更新一次。

​	获取服务是服务消费者的基础，所以必须确保eureka.client.fetch-registry=true参数没有被修改成false，该值默认为true。若希望修改缓存清单的更新时间， 可以通过eureka.client.registry-fetch-interval-seconds=30参数进行修改，该参数默认值为30， 单位为秒。

#### 7.4.2 服务调用

​	服务消费者在获取服务清单后， 通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。因为有这些服务实例的详细信息， 所以客户端可以根据自己的需要决定具体调用哪个实例，在Ribbon中会默认采用轮询的方式进行调用，从而实现客户端的负载均衡。

​	对于访问实例的选择， Eureka中有Region和Zone的概念， 一个Region中可以包含多个Zone， 每个服务客户端需要被注册到一个Zone中， 所以每个客户端对应一个Region和一个Zone。在进行服务调用的时候， 优先访问同处一个Zone 中的服务提供方，若访问不到， 就访问其他的Zone， 更多关于Region和Zone的知识， 我们会在后续的源码解读中介绍。

#### 7.4.3 服务下线

​	在系统运行过程中必然会面临关闭或重启服务的某个实例的情况， 在服务关闭期间，我们自然不希望客户端会继续调用关闭了的实例。所以在客户端程序中， 当服务实例进行正常的关闭操作时， 它会触发一个服务下线的阻ST请求给Eureka Server，告诉服务注册中心： “ 我要下线了” 。服务端在接收到请求之后， 将该服务状态置为下线（DOWN）， 并把该下线事件传播出去。

### 7.5 服务注册中心

#### 7.5.1 失效剔除

​	有些时候， 我们的服务实例并不一定会正常下线，可能由于内存溢出、网络故障等原因使得服务不能正常工作， 而服务注册中心并未收到“ 服务下线” 的请求。为了从服务列表中将这些无法提供服务的实例剔除， Eureka Server在启动的时候会创建一个定时任务，默认每隔一段时间（默认为60秒） 将当前清单中超时（默认为90秒〉没有续约的服务剔除出去。

#### 7.5.2 自我保护

​	当我们在本地调试基于Eureka的程序时， 基本上都会碰到这样一个问题， 在服务注册中心的信息面板中出现类似下面的红色警告信息：

​	实际上， 该警告就是触发了EurekaServer的自我保护机制。之前我们介绍过， 服务注册到EurekaServer 之后， 会维护一个心跳连接，告诉EurekaServer自己还活着。EurekaServer在运行期间，会统计心跳失败的比例在15分钟之内是否低于85 %，如果出现低于的情况〈在单机调试的时候很容易满足， 实际在生产环境上通常是由于网络不稳定导致）, EurekaServer会将当前的实例注册信息保护起来，让这些实例不会过期， 尽可能保护这些注册信息。但是， 在这段保护期间内实例若出现问题， 那么客户端很容易拿到实际己经不存在的服务实例， 会出现调用失败的情况， 所以客户端必须要有容错机制， 比如可以使用请求重试、断路器等机制。

​	由于本地调试很容易触发注册中心的保护机制， 这会使得注册中心维护的服务实例不那么准确。所以， 我们在本地进行开发的时候， 可以使用eureka.server.enableself-preservatio口＝ false参数来关闭保护机制，以确保注册中心可以将不可用的实例正确剔除。

## 8 源码分析

​	上面， 我们对Eureka 中各个核心元素的通信行为做了详细的介绍， 相信大家已经对Eureka 的运行机制有了一定的了解。为了更深入地理解它的运作和配置， 下面我们结合源码来分别看看各个通信行为是如何实现的。

​	在看具体源码之前， 我们先回顾 下之前所实现的内容， 从而找到 个合适的切入口去分析。首先， 对于服务注册中心、服务提供者、服务消费者这三个主要元素来说， 后两者（也就是Eureka 客户端〉在整个运行机制中是大部分通信行为的主动发起者， 而注册中心主要是处理请求的接收者。所以，我们可以从Eureka的客户端作为入口看看它是如何完成这些主动通信行为的。

​	我们在将一个普通的Spring Boot 应用注册到Eureka Server 或是从Eureka Server中获取服务列表时， 主要就做了两件事：

- 在应用主类中配置了＠EnableDiscoveryClient注解。
- 在application.properties中用eureka.client.serviceUrl.defaultZone参数指定了服务注册中心的位置。

顺着上面的线索， 我们来看看＠EnableDiscoveryClient的源码， 具体如下：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({EnableDiscoveryClientImportSelector.class})
public @interface EnableDiscoveryClient {
    boolean autoRegister() default true;
}
```

​	从该注解的注释中我们可以知道，它主要用来开启DiscoveryClient的实例。通过搜索DiscoveryClient，我们可以发现有一个类和一个接口。通过梳理可以得到如下图所示的关系：

![DiscoveryClient](/images/SpringCloud/SpringCloud_discoveryClient.png)

​	其中， 左边的org.springframework.cloud.client.discovery.DiscoveryClient是Spring Cloud的接口，它定义了用来发现服务的常用抽象方法， 通过该接口可以有效地屏蔽服务治理的实现细节，所以使用Spring Cloud构建的微服务应用可以方便地切换不同服务治理框架， 而不改动程序代码，只需要另外添加一些针对服务治理框架的配置即可。org.springframework.cloud. netflix.eureka .EurekaDiscoveryClient 是对该接口的实现，从命名来判断， 它实现的是对Eureka 发现服务的封装。所以EurekaDiscoveryClient依赖了Netflix Eureka的com. net flix.discovery.EurekaClient接口，EurekaClient继承了LookupService接口，它们都是Netflix开源包中的内容，主要定义了针对Eureka的发现服务的抽象方法，而真正实现发现服务的则是Netflix包中的com. netflix.discovery. DiscoveryClient类。

​	接下来，我们就来详细看看DiscoveryClient类吧。先解读一下该类头部的注释，注释的大致内容如下所示：![DiscoveryClient-class](/images/SpringCloud/SpringCloud_discoveryClient-class.png)
	

```java
/** @deprecated */  // DiscoveryClient
    @Deprecated
    public List<String> getServiceUrlsFromConfig(String instanceZone, boolean preferSameZone) {
        return EndpointUtils.getServiceUrlsFromConfig(this.clientConfig, instanceZone, preferSameZone);
    }
```

​	在具体研究EurekaClient负责完成的任务之前，我们先看看在哪里对EurekaServer的URL列表进行配置。根据我们配置的属性名eureka.client.serviceUrl .defaultZone，通过serviceUrl可以找到该属性相关的加载属性，但是在SR5版本中它们都被@Deprecated 标注为不再建议使用，并＠link到了替代类com.netflix.discovery.endpoi nt.EndpointUtils ，所以我们可以在该类中找到下面这个函数：

```java
public static Map<String, List<String>> getServiceUrlsMapFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
        Map<String, List<String>> orderedUrls = new LinkedHashMap();
        String region = getRegion(clientConfig);
        String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
        if (availZones == null || availZones.length == 0) {
            availZones = new String[]{"default"};
        }

        logger.debug("The availability zone for the given region {} are {}", region, Arrays.toString(availZones));
        int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);
        String zone = availZones[myZoneOffset];
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }

        int currentOffset = myZoneOffset == availZones.length - 1 ? 0 : myZoneOffset + 1;

        while(currentOffset != myZoneOffset) {
            zone = availZones[currentOffset];
            serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        }

     
    }
```

### 8.1 Region、Zone

​	在上面的函数中， 可以发现， 客户端依次加载了两个内容， 第一个是Region， 第二个是Zone， 从其加载逻辑上我们可以判断它们之间的关系：

- 通过getRegion函数， 我们可以看到它从配置中读取了一个Region 返回，所以一个微服务应用只可以属于一个Region， 如果不特别配置，默认为default。若我们要自己设置， 可以通过eureka.client.region 属性来定义。

  ```java
  public static String getRegion(EurekaClientConfig clientConfig) {
          String region = clientConfig.getRegion();
          if (region == null) {
              region = "default";
          }
  
          region = region.trim().toLowerCase();
          return region;
      }
  ```

- 通过getAvailabilityZones 函数， 可以知道当我们没有特别为Region 配置Zone 的时候， 将默认采用defaultZone， 这也是我们之前配置参数eureka.client.serviceUrl.defaultZone 的由来。若要为应用指定Zone， 可以通过eureka.client.availability-zones 属性来进行设置。从该函数的return内容， 我们可以知道Zone 能够设置多个， 并且通过逗号分隔来配置。由此， 我们可以判断Region与Zone 是一对多的关系。

  ```java
  public String[] getAvailabilityZones(String region) {
          String value = (String)this.availabilityZones.get(region);
          if (value == null) {
              value = "defaultZone";
          }
  
          return value.split(",");
      }
  ```

### 8.2 serviceUris

​	在获取了Region 和Zone 的信息之后， 才开始真正加载Eureka Server 的具体地址。它根据传入的参数按一定算法确定加载位于哪一个Zone 配置的serviceUrls

```java
int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);
        String zone = availZones[myZoneOffset];
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
```

​	具体获取serviceUrls 的实现， 我们可以详细查看getEurekaServerServiceUrls函数的具体实现类EurekaClientConfigBean， 该类是EurekaClientConfig 和EurekaConstants 接口的实现，用来加载配置文件中的内容，这里有非常多有用的信息，我们先说一下此处我们关心的， 关于defaultZone 的信息。通过搜索defaultZone,我们可以很容易找到下面这个函数， 它具体实现了如何解析该参数的过程， 通过此内容，我们就可以知道， eureka.client.serviceUrl.defaultZone 属性可以配置多个，并且需要通过逗号分隔。

```java
public List<String> getEurekaServerServiceUrls(String myZone) {
        String serviceUrls = (String)this.serviceUrl.get(myZone);
        if (serviceUrls == null || serviceUrls.isEmpty()) {
            serviceUrls = (String)this.serviceUrl.get("defaultZone");
        }

        if (!StringUtils.isEmpty(serviceUrls)) {
            String[] serviceUrlsSplit = StringUtils.commaDelimitedListToStringArray(serviceUrls);
            List<String> eurekaServiceUrls = new ArrayList(serviceUrlsSplit.length);
            String[] var5 = serviceUrlsSplit;
            int var6 = serviceUrlsSplit.length;

            for(int var7 = 0; var7 < var6; ++var7) {
                String eurekaServiceUrl = var5[var7];
                if (!this.endsWithSlash(eurekaServiceUrl)) {
                    eurekaServiceUrl = eurekaServiceUrl + "/";
                }

                eurekaServiceUrls.add(eurekaServiceUrl);
            }

            return eurekaServiceUrls;
        } else {
            return new ArrayList();
        }
    }
```

​	当我们在微服务应用中使用Ribbon 来实现服务调用时， 对于Zone 的设置可以在负载均衡时实现区域亲和特性： Ribbon 的默认策略会优先访问同客户端处于一个Zone 中的服务端实例，只有当同一个Zone 中没有可用服务端实例的时候才会访问其他Zone 中的实例。所以通过Zone 属性的定义，配合实际部署的物理结构， 我们就可以有效地设计出对区域性故障的容错集群。

### 8.3 服务注册

​	在理解了多个服务注册中心信息的加载后，我们再回头看看DiscoveryClient 类是如何实现“ 服务注册” 行为的， 通过查看它的构造类， 可以找到它调用了下面这个函数：

```java
 private void initScheduledTasks() {
        if (this.clientConfig.shouldRegisterWithEureka()) {
            //创建InstanceinfoReplicator实例
            this.instanceInfoReplicator = new InstanceInfoReplicator(this, this.instanceInfo, this.clientConfig.getInstanceInfoReplicationIntervalSeconds(), 2);
            this.statusChangeListener = new StatusChangeListener() {
                public String getId() {
                    return "statusChangeListener";
                }
            };

           //InstanceinfoReplicator实例执行定时任务 this.instanceInfoReplicator.start(this.clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } 

    }
```

​	从上面的函数中， 可以看到一个与服务注册相关的判断语句i f (c lientConfig.shouldRegisterWithEureka（））。在该分支内， 创建了一个InstanceinfoReplicator类的实例， 它会执行一个定时任务， 而这个定时任务的具体工作可以查看该类的run（）函数， 具体如下所示：

```java
 public void run() {
        boolean var6 = false;

        ScheduledFuture next;
        label53: {
            try {
                var6 = true;
                this.discoveryClient.refreshInstanceInfo();
                Long dirtyTimestamp = this.instanceInfo.isDirtyWithTime();
                if (dirtyTimestamp != null) {
                    this.discoveryClient.register();
                    this.instanceInfo.unsetIsDirty(dirtyTimestamp.longValue());
                    var6 = false;
                } else {
                    var6 = false;
                }
                break label53;
            } catch (Throwable var7) {
            } 
    }
```

​	相信大家都发现了discoveryClient.register （）；这一行，真正触发调用注册的地方就在这里。继续查看register （） 的实现内容， 如下所示：

```java
boolean register() throws Throwable {
        logger.info("DiscoveryClient_" + this.appPathIdentifier + ": registering service...");

        EurekaHttpResponse httpResponse;
        try {
            httpResponse = this.eurekaTransport.registrationClient.register(this.instanceInfo);
        } 

        return httpResponse.getStatusCode() == 204;
    }
```

​	通过属性命名， 大家基本也能猜出来， 注册操作也是通过REST请求的方式进行的。同时， 我们能看到发起注册请求的时候， 传入了一个corn.netflix.appinfo.Instance Info 对象， 该对象就是注册时客户端给服务端的服务的元数据。

### 8.4 服务获取与服务续约

​	顺着上面的思路， 我们继续来看DiscoveryClient 的initScheduledTasks 函数， 不难发现在其中还有两个定时任务， 分别是“ 服务获取” 和“ 服务续续约

```java
private void initScheduledTasks() {
        if (this.clientConfig.shouldFetchRegistry()) {
            //服务获取
            this.scheduler.schedule(new TimedSupervisorTask("cacheRefresh", this.scheduler, this.cacheRefreshExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.CacheRefreshThread()), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
        }

        if (this.clientConfig.shouldRegisterWithEureka()) {
            //服务续约
            this.scheduler.schedule(new TimedSupervisorTask("heartbeat", this.scheduler, this.heartbeatExecutor, renewalIntervalInSecs, TimeUnit.SECONDS, expBackOffBound, new DiscoveryClient.HeartbeatThread(null)), (long)renewalIntervalInSecs, TimeUnit.SECONDS);
            };
    }
```

从源码中我们可以发现， “服务获取 ” 任务相对于 “服务续约” 和 “服务注册” 任务更为独立。 “服务续约” 与 “服务注册” 在同一个 if逻辑中，这个不难理解，服务注册到 Eureka Server 后，自然需要一个心跳去续约， 防止被剔除， 所以它们肯定是成对出现的。 从源码中， 我们更清楚地看到了之前所提到的， 对于服务续约相关的时间控制参数：

```java
eureka.instance.lease-renewal-interval-in-seconds=30
eureka.instance.lease-expiration-duration-in-seconds=90
```

​	而“服务获取 ” 的逻辑在独立的一个 if 判断中， 其判断依据就是我们之前所提到的eureka.client.fetch-registry=true 参数，它默认为 true，大部分情况下我们不需要关心。 为了定期更新客户端的服务清单， 以保证客户端能够访问确实健康的服务实 例，“服务获取 ” 的请求不会只限于服务启动， 而是一个定时执行的任务， 从源码中我们可以看到任务运行中的 registryFetchintervalSeconds 参数对应的就是之前所提到的 eureka.client.registry-fetch-interval-seconds= 30 配置参数，它默认为 30 秒。
	继续向下深入， 我们能分别发现实现 “服务获取 ” 和 “服务续约” 的具体方法， 其中 服务续约” 的实现较为简单， 直接以REST请求的方式进行续约：

```java
boolean renew() {
        try {
            //直接以REST请求的方式进行续约
            EurekaHttpResponse<InstanceInfo> httpResponse = this.eurekaTransport.registrationClient.sendHeartBeat(this.instanceInfo.getAppName(), this.instanceInfo.getId(), this.instanceInfo, (InstanceStatus)null);
          
            if (httpResponse.getStatusCode() == 404) {
                this.REREGISTER_COUNTER.increment();
                return this.register();
            } 
        } 
    }
```

​	而“服务获取” 则复杂一些， 会根据是否是第一次获取发起不同的 REST 请求和相应的处理。 具体的实现逻辑跟之前类似， 有兴趣的读者可以继续查看服务客户端的其他具体内容， 以了解更多细节。

### 8.5 服务注册中心处理

​	通过上面的源码分析， 可以看到所有的交互都是通过 REST 请求来发起的。 下面我们来看看服务注册中心对这些请求的处理。 Eureka Server 对于各类 REST 请求的定义都位于 com.netflix.eureka.resources 包下。

​	以“服务注册服务为例”  

```java
	//ApplicationResource类 	
	@POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info, @HeaderParam("x-netflix-discovery-replication") String isReplication) {
        if (this.isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else {
            DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
            
           //服务注册
            this.registry.register(info, "true".equals(isReplication));
            return Response.status(204).build();
        }
    }
```

​	在对注册信息进行了 一堆校验之后， 会调用 org.springframework.cloud. netflix.eureka.server.InstanceRegistry 对象中的 register(Instanceinfo info, int leaseDuration, boolean isReplication）函数来进行服务注册：

```java
 public void register(InstanceInfo info, int leaseDuration, boolean isReplication) {
     	//注册前传播注册事件
        this.handleRegistration(info, leaseDuration, isReplication);
         //注册
        super.register(info, leaseDuration, isReplication);
    }
```

​	在注册函数中， 先调用 publishEvent 函数， 将该新服务注册的事件传播出去， 然后调用 com.netflix.eureka.registry.AbstractinstanceRegistry 父类中的注册实现，将 InstanceInfo 中的元数据信息存储在 一个 ConcurrentHashMap对象中。正如我们之前所说的， 注册中心存储了两层 Map 结构， 第一 层的 key 存储服务名：InstanceInfo中的appName 属性， 第二层的key 存储实例名：InstanceInfo中的instanceId属性。

​	服务端的请求和接收非常类似，对于其他的服务端处理， 这里不再展开叙述， 读者可以根据上面的脉络来自己查看其内容（这里包含很多细节内容〉来帮助和加深理解。

## 9 配置详解

​	在分析了Eureka的部分源码 之后， 相信大家对Eureka的服务治理机制已经有了进一步的理解。 在本节中， 我们从使用的角度对Eureka中一些常用配置内容进行详细的介绍，以帮助我们根据自身环境与业务特点来进行个性化的配置调整。

​	在Eureka的服务治理体系中， 主要分为服务端与客户端两个不同的角色， 服务端为服务注册中心， 而客户端为各个提供接口的微服务应用。当我们构建了高可用的注册中心之后， 该集群中所有的微服务应用和后续将要介绍的一些基础类应用（如配置中心、API网关等〉都可以视作该体系下的一个微服务（Eureka客户端〉。服务注册中心也一样， 只是高 可用环境下的服务注册中心除了作为客户端之外， 还为集群中的其他客户端提供了服务注 册的特殊功能。所以，Eureka客户端的配置对象存在于所有Eureka服务治理体系下的应用实例中。在实际使用Spring Cloud Eureka的过程中，我们所做的配置内容几乎都是对Eureka 客户端配置进行的操作， 所以了解这部分的配置内容， 对于用好Eureka非常有帮助。

​	Eureka客户端的配置主要分为以下两个方面。

- 服务注册相关的配置信息， 包括服务注册中心的地址、服务获取的问隔时间、可用区域等。
- 服务实例相关的配置信息， 包括服务实例的名称、IP地址、端口号、健康检查路径等。

​         而Eureka服务端更多地类似于一个现成产品， 大多数情况下， 我们不需要修改它的配置信息。 所以在 本书中， 我们对此不进行过多的介绍， 有兴趣的读者可以查看 org.springframework.cloud 。netflix.eureka.server.EurekaServerConfig-Bean 类的定义来做进一步的学习， 这些参数均以eureka.server 作为前缀。另外值得一提的是，我们在学习本书内容进行本地调试的时候， 可以通过设置该类中的 enableSelfPreservation参数来关闭注册中心的 “ 自我保护 ”功能， 以防止关闭的实例无法被服务注册中心剔除的问题

## 10 服务注册类配置

​	注册类配置关于服务注册类的配置信息， 我们可以通过查看 org.springframework.cloud.关于服务注册类的配置信息， 我们可以通过查看 org.springframework.cloud. netflix.eureka.EurekaClientConfigBean 的源码来获得比官方文档中更为详尽的内容，这些 配置信息都以eureka.client为前缀。 下面我们针对一些常用的配置信息做进一步的介绍和说明。

### 10.1 指定注册中心

​	在本章第1节的示 例中， 我们演示了如何将一个Spring Boot应用纳入Eureka的服务治理体系， 除了引入 Eureka 的依赖之外， 就是在配置文件中指定注册中心， 主要通过eureka.clie川.serviceUrl 参数实现。 该参数的定义 如下所 示， 它的配置值存储在Has灿1ap类型中， 并且设置有一组默认值， 默认值的key为 defaultZone、value为http://localhost:8761/eureka／。

```java
@ConfigurationProperties("eureka.client")
public class EurekaClientConfigBean implements EurekaClientConfig {
    public static final String PREFIX = "eureka.client";
    public static final String DEFAULT_URL = "http://localhost:8761/eureka/";
    public static final String DEFAULT_ZONE = "defaultZone";
```

​	由于之前实现的服务注册中心使用了1111端口， 所以我们做了如下配置， 来将应用 注册到对应的Eureka 服务端中。

```java
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```

​	当构建了高可用的服务注册中心集群时， 我们可以为参数的value 值 配置多个注册中心的地址〈通过逗号分隔〉。 比如下面的例子：

```java
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/,http://localhost:8000/eureka/
```

​	另外，为了 服务注册中心的安全考虑， 很多时候 我们都会为 服务注册中心加入安全校验。 这个时候，在配置 serviceUrl时， 需要在value值的URL中加入相应的安全校验 信息， 比如http://<user口ame>:<password>@localhost:1111/eureka。 其中， <username＞为 安全校验信息的用户名，＜password＞为该用户的密码。

### 10.2  其他配置

​	下面整理了 org.springframework.cloud.netflix.eureka.EurekaClient­ConfigBean中 定义的常用 配置参数以及对应的说明和默认值， 这些参数均以 eureka. client为前缀。

| 参数名                                        | 说明                                                         | 默认值 |
| --------------------------------------------- | ------------------------------------------------------------ | ------ |
| enabled                                       | 启用 Eureka 客户端                                           | true   |
| registryFetchlntervalSeconds                  | 从 Eureka 服务端获取注册信息的间隔时间，                     | 30     |
| instancelnfoReplicationintervalSeconds        | 更新实例信息的变化到 Eureka 服务端的问隔时间， 单位为秒      | 30     |
| initiallnstancelnfoReplicationlntervalSeconds | 初始化实例信息到 Eureka 服务端的间隔时间， 单位为秒          | 40     |
| eurekaServiceUrlPolllntervalSeconds           | 轮询Eureka 服务端地址更改的间隔时间， 单位为秒。 当我们与 Spring Cloud Config 配合，动态刷新Eureka 的 serviceURL 地址时需要关注该参数 | 300    |
| eurekaServerReadTimeoutSeconds                | 读取 Eureka Server信息的超时时间， 单位为秒                  | 8      |
| eurekaServerConnectTimeoutSeconds             | 连接 Eureka Server 的超时时间， 单位为秒                     | 5      |
| eurekaServerTotalConnections                  | 从 Eureka 客户端到所有 Eureka 服务端的连接总数               | 200    |
| eurekaServerTotalConnectionsPerHost           | 从 Eureka 客户端到每个 Eureka 服务端主机的连接总数           | 50     |
| eurekaConnectionldle TimeoutSeconds           | Eureka 服务端连接的空闲关闭时间， 单位为秒                   | 30     |
| heartbeatExecutorThreadPoolSize               | 心跳连接池的初始化线程数                                     | 2      |
| heartbeatExecutorExponentialBackOffBound      | 心跳超时重试延迟时间的最大乘数值                             | 10     |
| cacheRe台eshExecutorThreadPoolSize            | 缓存刷新线程池的初始化线程数                                 | 2      |
| cacheRefreshExecutorExponentialBackO任Bound   | 缓存刷新重试延迟时间的最大乘数值                             | 10     |
| useDnsF orF etching Service U rls             | 使用 DNS 来获取 Eureka 服务端的 serviceUrl                   | false  |
| registerWithEureka                            | 是否要将自身的实例信息注册到 Eureka 服务端                   | true   |
| preferSameZoneEureka                          | 是否偏好使用处于相同 Zone 的 Eureka 服务端                   | true   |
| filterOnly Up Instances                       | 获取实例时是否过滤， 仅保留UP状态的实例                      | true   |
| fetchRegis仕y                                 | 是否从 Eureka 服务端获取注册信息                             | true   |

## 11 服务实例类配置

​	关于服务实例类的配置信息， 我们可以通过查看org.springfrarnework.cloud.netflix.eureka.EurekainstanceConfigBean的源码来获取详细内容， 这些配置信息都以eureka.instance为前缀，下面我们针对一些常用的配置信息做一些详细的说明。

### 11.1 元数据

​	在org.springframework.cloud.netflix.eureka.EurekainstanceConfigBean 的配置信息中，有一大部分内容都是对服务实例元数据的配置，那么什么是服务实例的元 数据呢？它是Eureka客户端在向服务注册中心发送注册请求时，用来描述自身服务信息的对象，其中包含了一些标准化的元数据，比如服务名称、实例名称、实例E、实例端口等用于服务治理的重要信息；以及一些用于负载均衡策略或是其他特殊用途的自定义元数据 信息。

​	在使用SpringCloud Eureka的时候，所有的配置信息都通过org.springframework.
cloud.netflix.eureka.EurekainstanceConfigBean进行加载，但在真正进行服务注册的时候，还是会包装成com.netflix.appinfo.Instanceinfo对象发送给 Eureka服务端。这两个类的定义非常相似，我们可以直接查看com.netflix.appinfo.Instanceinfo类中的详细定义来了解原生Eureka对元数据的定义。其中，Map<String, String> metadata = newConcurrentHashMap<String, String> () 是自定义的元数据信息，而其他成员变量则是标准化的元数据信息。SpringCloud的
EurekainstanceConfigBean对原生元数据对象做了一些配置优化处理，在后续的介绍中，我们也会提到这些内容。

​	我们可以通过eureka.instance.<properties>=<value＞的格式对标准化元数据直接进行配置，其中＜properties＞就是EurekainstanceConfigBean对象中的成员变量名。而对于自定义元数据，可以通过eureka.instance.metadataMap. <key>=<value＞的格式来进行配置，比如：

```java
eureka.instance.metadataMap.zone=shanghai
```

接下来，我们将针对一些常用的元数据配置做进一步的介绍和说明。

### 11.2 实例名配置

​	实例名，即Instanceinfo中的instanceId参数，它是区分同一服务中不同实例的唯一标识。在Netflix Eureka的原生实现中，实例名采用主机名作为默认值，这样的设置使得在同一主机上无法启动多个相同的服务实例。所以，在SpringCloud Eureka的配置中，针对同一主机中启动多实例的情况，对实例名的默认命名做了更为合理的扩展，它采用了如下默认规则：

```java
${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}
```

对于实例名的命名规则，我们可以通过eureka.instance.instanceid参数来进行配置。比如，在本地进行客户端负载均衡调试时，需要启动同一服务的多个实例，如果我们直接启动同一个应用必然会产生端口冲突。虽然可以在命令行中指定不同的server. port 来启动， 但是这样还是略显麻烦。 实际上， 我们可以直接通过设置 server.port=O 或者使用随机数 server.port=${randorn.int[10000,19999 ］｝来让 Tomcat 启动的时候采用随机端口。 但是这个时候我们会发现注册到 Eureka Server 的实例名都是相同的，这会使得只有一个服务实例能够正常提供服务。 对于这个问题， 我们就可以通过设置实例名规则来轻松解决：

```java
eureka.instance.instanceId=${spring.application.name}:${random.int}
```

​	通过上面的配置，利用应用名加随机数的方式来区分不同的实例，从而实现在同一主机上，不指定端口就能轻松启动多个实例的效果。

### 11.3 端点配置

​	在 InstanceInfo 中， 我们可以看到一些 URL 的配置信息，比如 homePageUrl、 statusPageUrl、healthCheckU址，它们分别代表了应用主页的 URL、状态页的 URL 、健康检查的 URL。 其中，状态页和健康检查的 URL 在 Spring Cloud Eureka 中默认使用了 spring-boot-actuator 模块提供的/info 端点和／health 端点。 虽然我们在之前的示例中并没有对这些端点做具体的设置，但是实际上这些 URL 地址的配置非常重要。 为了 服务的正常运作， 我们必须确保 Eureka 客户端的／health 端点在发送元数据的时候，是一个能够被注册中心访问到的地址，否则服务注册中心不会根据应用的健康检查来更改状态（仅当开启了 healthcheck 功能时，以该端点信息作为健康检查标准〉。 而／info端 点如果不正确的话，会导致在 Eureka 面板中单击服务实例时，无法访问到服务实例提供的信息接口。

​	大多数情况下，我们并不需要修改这几个 URL 的配置，但是在一些特殊情况下，比如，为应用设置了 context-path，这时，所有 spring-boot-actuator 模块的监控端点 都会增加一个前缀。 所以， 我们就需要做类似如下的配置，为／info和／health 端点也加上类似的前缀信息：

```java
management.context-path=/hello

eureka.instance.statusPageUrlPath=${management.context-path}/info
eureka.instance.healthCheckUrlPath=${management.context-path}/health
```

​	另外，有时候为了安全考虑，也有可能会修改／info 和／health 端点的原始路径。这个时候，我们也需要做一些特殊的配置， 比如像下面这样：

```java
endpoints.info.path=/appInfo
endpoints.health.path=/checkHealth

eureka.instance.statusPageUtlPath=/${endpoints.into.path}
eureka.instance.healthCheckUrlPath=/${endpoints.health.path}
```

在上面所举的两个示例中， 我们使用了 eureka.instance.statusPageUrlPath
和eureka.instance.healthCheckUrlPath参数，这两个配置值有 个共同特点，它们都使用 相对路径来进行配置 。由于Eureka的服务注册中心 默认会以HTTP的方式来访问和暴露这些端点，因此当客户端应用以HTTPS的方式来 暴露服务和监控端点时，相对路径的配置方式就无法满足需求了。 所以，Spring Cloud Eureka还提供了绝对路径的配置参数，具体示例如下所示：

```java
eureka.instance.statusPageUtl=https://${eureka.instance.hostname}/info
eureka.instance.healthCheckUrl=https://${eureka.instance.hostname}/health
eureka.instance.homePageUrl=https://${eureka.instance.hostname}
```

### 11.4 健康检查

​	默认情况下，Eureka中各个服务实例的健康检测并不是通过spring-boot-actuator模块的／health 端点来实现的，而是依靠客户端心跳的方式来保持服务实例的存活。 在Eureka 的服务续约与剔除机制下，客户端的健康状态从注册到注册中心开始都会处于UP状态， 除非心跳终止 段时间之后，服务注册中心将其剔除 。默认的心跳实现方式可以有数微服务应用都会有 些其他的夕阳日资源依赖， 比如数据库、 缓存、 消息代理等，如果我们的应用与这些外部资源无法联通的时候，实际上已经不能提供正常的对外服务了，但是因为客户端心跳依然在运行，所以它还是会被服务消费者调用，而这样的调用实际上并不能获得预期的结果。

​	在Spring Cloud Eureka中，我们可以通过简单的配置，把Eureka客户端的健康检测交给spring-boot-actuator模块的／health端点，以实现更加全面的健康状态维护。详细的配置步骤如下所示：

- 在pom.xml中引入spring-boot-starter-actuator模块的依赖
- 在application.properties中增加参数配置 eureka.client.healthcheck.enabled=true。
- 如果客户端的/health 端点路径做了一些特殊处理，请参考前文介绍端点配置时的方法进行配置，让服务注册中心可以正确访问到健康监测端点

### 11.5 其他配置

​	除了上面介绍的配置参数外，下面整理了一些org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean中定义的配置参数以及对应的说明和默认值，这些参数均以eureka.instance为前缀

| 参数名                           | 说明                                                         | 默认值      |
| -------------------------------- | ------------------------------------------------------------ | ----------- |
| preferlpAddress                  | 是否优先使用IP地址作为主机名的标识                           | 默认值false |
| leaseRenewallntervallnSeconds    | Eureka客户端向服务端发送心跳的时间间隅，单位为秒             | 30          |
| leaseExpirationDurationlnSeconds | Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒。超过该时间之后服务端会将该服务实例从服务清单中剔除，从而禁止服务调用请求被发送到该实例上 | 90          |
| nonSecurePort                    | 非安全的通信端口号                                           | 80          |
| securePort                       | 安全的通信端口号                                             | 443         |
| nonSecurePortEnabled             | 是否启用非安全的通信端口号                                   | 位ue        |
| securePortEnab led               | 是否启用安全的通信端口号                                     |             |
| appname                          | 服务名，默认取spring.application.name的配置值，如果没有则为unknown |             |
| hostname                         | 主机名，不配置的时候将根据操作系统的主机名来获取             |             |

​	在上面的这些配置中，除了前三个配置参数在需要的时候可以做一些调整，其他的参数配置大多数情况下不需要进行配置，使用默认值即可。

## 11.6 跨平台支持

​	在“Eureka详解”一节中，我们对SpringCloud Eureka的源码做了较为详细的分析，在分析过程中相信大家己经发现，Eureka的通信机制使用了HTTP的阻ST接口实现，这也是Eureka同其他服务注册工具的一个关键不同点。由于HTTP的平台无关性，虽然Eureka Server通过Java实现，但是在其下的微服务应用并不限于使用Java来进行开发。

​	跨平台本身就是微服务架构的九大特性之一，只有实现了对技术平台的透明，才能更好地发挥不同语言对不同业务处理能力的优势，从而打造更为强大的大型系统。

​	目前除了Eureka的Java客户端之外，还有很多其他语言平台对其的支持，比如eureka-js-client、python-eureka等。若我们有志于自己为一门语言来开发客户端程序，也并非特别复杂，只需要根据上面提到的那些用户服务协调的通信请求实现就能实现服务的注册与发现，需要了解更多关于阳eka的RESTAPI内容可以查看Eureka宫方WiKi中的Eurekcz-REST-operations一文htφs://github.com/Netflix/eureka/wiki/Eureka-REST-operationso

### 11.6.1 通信协议

​	默认情况下，Eureka使用Jersey和XStream配合JSON作为Server与Client之间的通信协议。你也可以选择实现自己的协议来代替。

- Jersey是JAX-RS的参考实现，它包含三个主要部分。

  - **核心服务器CCore Server）:** 通过提供JSR311中标准化的注释和API标准化，你可以用直观的方式开发RESTfulWeb服务。
  - **核心客户端CCore Client)** : Jersey客户端API帮助你与REST服务轻松通信。
  - **集成CIntegration): ** Jersey还提供可以轻松集成Spring、Guice、ApacheAbdera的库

- XS位earn是用来将对象序列化成泊位（JSON）或反序列化为对象的一个Java类库。 XStream在运行时使用Java反射机制对要进行序列化的对象树的结构进行探索，并不需要对对象做出修改XStream可以序列化内部字段，包括private和final字段并且支持非公开类以及内部类。默认情况下，XS位earn不需要配置映射关系，对象和字段将映射为同名XML元素。但是当对象和字段名与XML中的元素名不同时， XStream支持指定别名。XStream支持以方法调用的方式，或是Java标注的方式指定别名。XStream在进行数据类型转换时，使用系统默认的类型转换器。同时，也支持用户自定义的类型转换器

  ​	JAX-RS是将在JavaEE 6中引入的一种新技术。JAX-RS即JavaAPI for 
  RESTful Web Services，是一个Java编程语言的应用程序接口，支持按照表述性状
  态转移（REST）架构风格创建Web服务。只X-RS使用了JavaSE 5引入的Java 标注来简化Web服务的客户端和服务端的开发和部署。包括：

  - @Path，标注资源类或者方法的相对路径。
  - @GET、＠PUT、＠POST、＠DELETE，标注方法是HTTP请求的类型。
  - @Produces，标注返回的M肌1E媒体类型。
  - @Consumes，标注可接受请求的MIME媒体类型。
  - @PathParam、＠QueryParam、＠HeaderParam、＠CookieParam、＠MatrixPaiam @FormParam，标注方法的参数来自HTTP请求的不同位置，例如，＠PathParam来自URL的路径，＠QueryParam来自URL的查询参数，＠HeaderParam来自 HTTP请求的头信息，＠CookieParam来自HTTP请求的Cookieo

  之前在分析EurekaServer端源码的时候，查看请求处理时可以看到很多上面这些注解的定义。






























































---
layout: post
title: 06 api网关服务
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

通过前几章的介绍， 我们对于 Spring Cloud N etflix 下的核心组件己经了解了 大半

<!-- more --> 

# API网关服务

​	通过前几章的介绍， 我们对于 Spring Cloud N etflix 下的核心组件己经了解了 大半。这些组件基本涵盖了微服务架构中最为基础的几个核心设施， 利用这些组件我们已经可以构建起 个简单的微服务架构系统， 比如， 通过使用 Spring Cloud Eureka 实现高可用的服务注册中心以及实现微服务的注册与发现；通过 Spring Cloud Ribbon 或 Feign 实现服务间负载均衡的接口调用；同时，为了使分布式系统更为健壮，对于依赖的服务调用使用 Spring Cloud Hystrix 来进行包装， 实现线程隔离并加入熔断机制， 以避免在微服务架构中因个别服务出现异常而引起级联故障蔓延。 通过上述思路， 我们可以设计出类似下图的基础系统架构。
![zuul](/images/SpringCloud/SpringCloud_zuul.png)

​	在该架构中，我们的服务集群包含内部服务ServiceA和ServiceB，它们都会向Eureka Server集群进行注册与订阅服务，而OpenService是个对外的RESTfulAPI服务，它通过日、Nginx等网络设备或工具软件实现对各个微服务的路由与负载均衡，并公开给外部的客户端调用。
	在本章中，我们将把视线聚焦在对外服务这块内容，通常也称为边缘服务。首先需要肯定的是，上面的架构实现系统功能是完全没有问题的，但是我们还是可以进步思考下，这样的架构是否还有不足的地方会使运维人员或开发人员感到痛苦。

​	首先，我们从运维人员的角度来看看，他们平时都需要做一些什么工作来支持这样的架构。当客户端应用单击某个功能的时候往往会发出一些对微服务获取资源的请求到后端，这些请求通过F5、Nginx等设施的路由和负载均衡分配后，被转发到各个不同的服务实例上。而为了让这些设施能够正确路由与分发请求，运维人员需要手工维护这些路由规则与 服务实例列表，当有实例增减或是IP地址变动等情况发生的时候，也需要手工地去同步修改这些信息以保持示例信息与中间件配置内容的一致性。在系统规模不大的时候，维护这些信息的工作还不会太过复杂，但是如果当系统规模不断增大，那么这些看似简单的维护任务会变得越来越难，并且出现配置错误的概率也会逐渐增加。很显然，这样的做法并不可取，所以我们需要一套机制来有效降低维护路由规则与服务实例列表的难度。

​	其次，我们再从开发人员的角度来看看，在这样的架构下，会产生一些怎样的问题呢？大多数情况下，为了保证对外服务的安全性，我们在服务端实现的微服务接口，往往都会有一定的权限校验机制，比如对用户登录状态的校验等；同时为了防止客户端在发起请求时被篡改等安全方面的考虑，还会有一些签名校验的机制存在。这时候，由于使用了微服务架构的理念，我们将原本处于一个应用中的多个模块拆成了多个应用，但是这些应用提供的接口都需要这些校验逻辑，我们不得不在这些应用中都实现这样一套校验逻辑。随着微服务规模的扩大，这些校验逻辑的冗余变得越来越多，突然有一天我们发现这套校验逻辑有个BUG需要修复，或者需要对其做一些扩展和优化，此时我们就不得不去每个应用里修改这些逻辑，而这样的修改不仅会引起开发人员的抱怨，更会加重测试人员的负担。所以，我们也需要套机制能够很好地解决微服务架构中，对于微服务接口访问时各前置校验的冗余问题。

​	为了解决上面这些常见的架构问题，API网关的概念应运而生。API网关是个更为智能的应用服务器，它的定义类似于面向对象设计模式中的Facade模式，它的存在就像是整个微服务架构系统的门面一样，所有的外部客户端访问都需要经过它来进行调度和过滤。它除了要实现请求路由、负载均衡、校验过滤等功能之外，还需要更多能力，比如与服务治理框架的结合、请求转发时的熔断机制、服务的聚合等系列高级功能。
	既然API网关对于微服务架构这么重要，那么在SpringCloud中是否有相应的解决方案呢？答案是很肯定的， 在 Spring Cloud 中了提供了基于 Netflix Zuul 实现的 API 网关组件 Spring Cloud Zuul。 那么， 它是如何解决上面这两个普遍问题的呢？

​	首先， 对于路由规则与服务实例的维护问题。 Spring Cloud Zuul 通过与 Spring Cloud Eureka 进行整合， 将自身注册为 Eureka 服务治理下的应用， 同时从 Eureka 中获得了所有其他微服务的实例信息。 这样的设计非常巧妙地将服务治理体系中维护的实例信息利用起来， 使得将维护服务实例的工作交给了服务治理框架自动完成， 不再需要人工介入。 而对 于路由规则的维护， Zuul 默认会将通过以服务名作为 ContextPath 的方式来创建路由映射，大部分情况下，这样的默认设置己经可以实现我们大部分的路由需求， 除了一些特殊情况比如兼容一些老的URL，还需要做一些特别的配置。但是相比于之前架构下的运维工作量，通过引入Spring Cloud Zuul实现Api网关后，已经能够大大减少了。

​	其次， 对于类似签名校验、 登录校验在微服务架构中的冗余问题。 理论上来说， 这些校验逻辑在本质上与微服务应用自身的业务并没有多大的关系， 所以它们完全可以独立成一个单独的服务存在，只是他们被剥离和独立出来之后，并不是给各个微服务调用，而是在API网关上进行统一调用来对微服务接口作前置过滤，在 API 网关服务上进行统 调用来对微服务接口做前置过滤， 以实现对微服务接口的拦截和校验。 Spring Cloud Zuul 提供了 套过滤器机制， 它可以很好地支持这样的任务。 开发者可以通过使用Zuul来创建各种校验过滤器，然后指定哪些规则的请求需要执行校验逻辑，只有通过校验的才会被路由到具体的微服务接口，不然就返回错误提示。 通过这样的改造，各个业务层的微服务应用就不再需要非业务性质的校验逻辑了， 这使得我们的微服务应用可以更专注于业务逻辑的开发， 同时微服务的自动化测试也变得更容易实现。
	微服务架构虽然可以将我们的开发单元拆分得更为细致， 有效降低了开发难度， 但是它所引出的各种问题如果处理不当会成为实施过程中的不稳定因素， 甚至掩盖掉原本实施微服务带来的优势。所以，在微服务架构的实施方案中，API网关服务的使用几乎成为了必然的选择。

​	下面我们将详细介绍 Spring Cloud Zuul 的使用方法、 配置属性以及 要进行的思考。

## 1 快速入门

​	介绍了这么多关于 API 网关服务的概念和作用， 在这 节中， 我们不妨用实际的示例来直观地体验一下 Spring Cloud Zuul 中封装的 API 网关是如何使用和运作， 并应用到微服 务架构中去的。

## 2 构建网关

首先，在实现各种API网关服务的高级功能之前，我们需要做些准备工作，比如，构建起最基本的API网关服务，并且搭建几个用于路由和过滤使用的微服务应用等。对于微服务应用，我们可以直接使用之前章节实现的hello-service和feign-consumer。虽然之前我们直将feign-consumer视为消费者，但是在Eureka的服务注册与发现体系中，每个服务既是提供者也是消费者，所以feign-consumer实质上也是个服务提供者。之前我们访问的http://localhost:9001/feign-consumer等一系列接口就是它提供的服务。读者也可以使用自己实现的微服务应用，因为这部分不是本章的重点
任何微服务应用都可以被用来进行后续的试验。这里，我们详细介绍下API网关服务的构建过程。

- 创建一个基础的Spring Boot工程，命名为api-gateway，并在pom.xml中引入spring-cloud-starter-zuul依赖，具体如下：

  ```java
  <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-zuul</artifactId>
          </dependency>
  ```

​        对于spring-cloud-starter-zuul依赖，可以通过查看它的依赖内容了解到：该模块中不仅包含了NetflixZuul的核心依赖zuul-core，它还包含了下面这些网 关服务需要的重要依赖。

1. spring－cloud-starter-hystrix：该依赖用来在网关服务中实现对微服务 转发时候的保护机制，通过线程隔离和断路器，防止微服务的故障引发API网关资源无法释放，从而影响其他应用的对外服务。
2. spring-cloud-starter-ribbon：该依赖用来实现在网关服务进行路由转发时候的客户端负载均衡以及请求重试。
3. spring-boot-starter-actuator：该依赖用来提供常规的微服务管理端点。另外，在SpringCloud Zuul中还特别提供了／routes端点来返回当前的所有路由规则。

	 创建应用主类，使用＠EnableZuulProxy注解开启Zuul的API网关服务功能	

  ```java
  @SpringCloudApplication
  @EnableZuulProxy
  public class ZuulApplication {
      public static void main(String[] args) {
          new SpringApplicationBuilder(ZuulApplication.class).web(true).run(args);
      }
  }
  ```

- 在application.properties中配置Zuul应用的基础信息，如应用名、服务端口号等，具体内容如下：

  ```java
  server.port=5555
  spring.application.name=api-gateway
  ```

  完成上面的工作后，通过Zuul实现的API网关服务就构建完毕了。

## 3 请求路由

​	下面，我们将通过个简单的示例来为上面构建的网关服务增加请求路由的功能。为了演示请求路由的功能，我们先将之前准备的Eureka服务注册中心和微服务应用都启动起来。

### 3.1 传统路由方式

​	使用SpringCloud Zuul实现路由功能非常简单，只需要对api-gateway服务增加一些关于路由规则的配置，就能实现传统的路由转发功能，比如：

```java
zuul.routes.api-a-url.path=/api-a-url/**
zuul.routes.api-a-url.url=http://localhost:8080/
```

​	该配置定义了发往API网关服务的请求中，所有符合／api-a-url／六六规则的访问都将被路由转发到http://localhost:8080／地址上，也就是说，当我们访问 http://localhost:5555/api-a-url/hello的时候，API网关服务会将该请求路由到http://localhost:8080/hello提供的微服务接口上。其中，配置属性 zuul.routes.api-a-url.path中的api一a-url部分为路由的名字，可以任意定义，但是一组path和url映射关系的路由名要相同，下面将要介绍的面向服务的映射方式也是如此

### 3.2 面向服务的路由

​	很显然，传统路由的配置方式对于我们来说并不友好，它同样需要运维人员花费大量的时间来维护各个路由path与url的关系。为了解决这个问题，SpringCloud Zuul实现了与Spring Cloud Eureka的无缝整合，我们可以让路由的path不是映射具体的url，而是让它映射到某个具体的服务，而具体的url则交给Eureka的服务发现机制去自动维护，我们称这类路由为面向服务的路由。在Zuul中使用服务路由也同样简单，只需做下面这些配置。

- 为了与Eureka整合，我们需要在api-gateway的pom.xml中引入spring­cloud-starter-eureka依赖，具体如下

  ```java
   <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-eureka</artifactId>
   </dependency>
  ```

- 在api-gateway的application.properties配置文件中指定Eureka注册中心的位置，并且配置服务路由。具体如下

  ```java
  server.port=5555
  spring.application.name=api-gateway
  
  zuul.routes.api-b.path=/api-b/**
  zuul.routes..api-b.serviceId=feign-consumer
  
  eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka
  ```

​        针对我们之前准备的两个微服务应用hello-service和feign-consumer，在上面的配置中分别定义了两个名为api-a和api-b的路由来映射它们。另外，通过指定 Eureka Server服务注册中心的位置，除了将自己注册成服务之外，同时也让Zuul能够获取feign-client服务的实例清单，以实现path 映射服务， 再从 服务中挑选实例来进行请求转发的完整路由机制。

​	在完成了上面的服务路由配置之后，我们可以将eureka-server、feign-consumer以及这里用SpringCloud Zuul构建的api-gateway都启动起来。启动完毕，在eureka-server的信息面板中，我们可以看到，除了hello-service和 feign-consumer之外，多了一个网关服务API-GATEWAY。

![zuul3](/images/SpringCloud/SpringCloud_zuul3.png)

​	通过上面的搭建工作，我们已经可以通过服务网关来访问hello-service和 feign-consumer这两个服务了。根据配置的映射关系，分别向网关发起下面这些请求。

- http://localhost:5555/api-b/feign/customer1： 该 url 符合／api-a／**规则， 由 api-a 路由负责转发，该路由映射的serviceid为feign-consumer ，

​        通过面向服务的路由配置方式， 我们不需要再为各个路由维护微服务应用的具体实例的位置， 而是通过简单的path与serviceId的映射组合 ， 使得维护工作变得非常简单。 这完全归功于Spring Cloud Eureka的服务发现机制， 它使得API网关服务可以自动化完成服务 实例清单的维护， 完美地解决了对路由映射实例的维护问题。

### 3.3 请求过滤

​	在实现了请求路由功能之后， 我们的微服务应用提供的接口就可以通过统一的API网关入口被客户端 访问到了。 但是， 每个客户端用户请求微服务应用提供的接口时， 它们的访问权限往往都有 一定的限制， 系统并不会将所有 的微服务接口都对它们开放。 然而， 目前的服务路由并没有 限制 权限这样的功能， 所有请求都会被毫无保留地转发 到具体的应用并返回结果， 为了实现对客户端请求的安全校验和权限控制， 最简单和粗暴的方法就是为每个微服务应用都实现一套用于校验签名和鉴别权限的过滤器或拦截器。 不过， 这样的做 法并不可取， 它会增加日后系统的维护难度， 因为同一个系统中的各种校验逻辑很多情况下都是大致相同或类似的，这样的实现方式会使得相似的校验逻辑代码被分散到了各个微 服务中去，冗余代码的出现是我们不希望看到的。 所以， 比较好的做法是将这些校验逻辑剥离出去， 构建出 个独立的鉴权服务。 在完成了剥离之后，有不少开发者会直接在微服 务应用中通过调用鉴权服务来实现校验，但是这样的做法仅仅只是解决了鉴权逻辑的分离，并没有在本质上将这部分不属于冗余的逻辑从原有的微服务应用中拆分出， 冗余的拦截器或过滤器依然会存在。

​	对于这样的问题， 更好的做法是通过前置的网关服务来完成这些非业务性质的校验。由于网关服务的加入， 外部客户端访问我们的系统已经有了统 入口， 既然这些校验与具体业务无关， 那何不在请求到达的时候就完成校验和过滤， 而不是转发后再过滤而导致更 长的请求延迟。 同时， 通过在网关中完成校验和过滤，微服务应用端就可以去除各种复杂的过滤器和拦截器了，这使得微服务应用接口的开发和测试复杂度也得到了相应降低。为了在API网关中实现对客户端请求的校验，我们将继续介绍Spring Cloud Zuul的另外一个核心功能：请求过滤Zuul允许开发者在API网关上通过定义过滤器来实现对请求的拦截与过滤，实现的方法非常简单，我们只需要继承ZuulFilter抽象类并实现它定义 的4个抽象函数就可以完成对请求的拦截和过滤了。

​	下面的代码定义了 个简单的Zuul过滤器， 它实现了在请求被路由之前检查HttpServletRequest中是否有accessToken参数，若有就进行路由， 若没有就拒绝访问， 返回401 Unauthorized错误。

```java
public class AccessFilter extends ZuulFilter {
    Logger logger = LoggerFactory.getLogger(AccessFilter.class);

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        HttpServletResponse response = ctx.getResponse();
        response.setContentType("application/json; charset=utf-8");
        logger.info("send {} request to {}", request.getMethod(),
                request.getRequestURL().toString());
        Object accessToken = request.getParameter("accessToken");
        if(null == accessToken ){
            logger.warn("access token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            Map<String, Object> map = new HashMap<>();
            map.put("code", 401);
            map.put("message", "access token is empty");
            try {
                PrintWriter out = response.getWriter();
                String jsonStr = JSONObject.toJSONString(map);
                out.print(jsonStr);
                out.flush();
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            return null;
        }
        logger.info("access token ok");
        return null;
    }
}
```

​	在上面实现的过滤器代码中，我们通过继承 ZuulFilter 抽象类并重写下面 4 个方法来实现自定义的过滤器。 这 4 个方法分别定义了如下内容。

- filterType： 过滤器的类型， 它决定过滤器在请求的哪个生命周期中执行。 这里定义为 pre，代表会在请求被路由之前执行。
- filterOrder：过滤器的执行顺序。当请求在一个阶段中存在多个过滤器时，需要根据该方法返回的值来依次执行。
- shouldFilter：判断该过滤器是否需要被执行。这里我们直接返回了true，因此该过滤器对所有请求都会生效。实际运用中我们可以利用该函数来指定过滤器的 有效范围。
- run：过滤器的具体逻辑。这里我们通过ctx.setSendZuulResponse(false) 令zuul过滤该请求，不对其进行路由，然后通过ctx.setResponseStatus­-Code(401)设置了其返回的错误码，当然也可以进一步优化我们的返回，比如通过过ctx.setResponseBody(body）对返回的body内容进行编辑等。

​        在实现了自定义过滤器之后，它并不会直接生效，我们还需要为其创建具体的Bean才能启动该过滤器，

```java
@Bean
public AccessFilter accessFilter(){
    return new AccessFilter();
}
```

​	在对 api-gateway服务完成了上面的改造之后， 我们可以重新启动它， 并发起下面的请求， 对上面定义的过滤器做一个验证。

- http://localhost:5555/api-b/feign/customer1： 返回401错误。
- http://localhost:5555/api-b/feign/customer1?accessToken= token： 正确路由到hello-service的／hello接口， 并返回hello DIDI

​        到这里， 对于API网关服务的快速入门示例就完成了。 通过对 Spring Cloud Zuul 两个核心功能的介绍， 相信读者己经能够体会到API网关服务对微服务架构的重要性了， 就目前掌握的API网关知识， 我们可以将具体原因总结如下 ：

1. 它作为系统的统一入口， 屏蔽了系统内部各个微服务的细节。
2. 它可以与服务治理框架结合，实现自动化的服务实例维护以及负载均衡的路由转发。 ．它可以实现接口权限校验与微服务业务逻辑的解祸。
3. 通过服务网关中的过滤器， 在各生命周期中去校验请求的内容， 将原本在对外服务 层做的校验前移， 保证了微服务的无状态性， 同时降低了微服务的测试难度， 让服 务本身更集中关注业务逻辑的处理。

​       实际上， 基于 Spring Cloud Zuul 实现的API网关服务除了上面所示的优点之外， 它还有一些更加强大的功能， 我们将在后续的章节对其进行更深入的介绍。 通过本节的内容，我们只是希望以一个简单的例子带领读者先来简单认识一下API网关服务提供的基础功能以及它在微服务架构中的重要地位。

## 4 路由详解

​	在 “快速入门”一节的请求路由示例中， 我们对 Spring Cloud Zuul 中的两类路由功能己经做了简单的使用介绍。 在本节中， 我们将进一步详细介绍关于 Spring Cloud Zuul 的路由功能， 以帮助读者更好地理解和使用它。

### 4.1 传统路由配置

​	所谓的传统路由配置方式就是在不依赖于服务发现机制的情况下， 通过在配置文件中具体指定每个路由表达式与服务实例的映射关系来实现API网关对外部请求的路由。

​	没有 Eureka 等服务治理框架的帮助，我们 需要根据服务实例的数量采用不同方式的配 置来实现路由规则。

- **单实例配置：** 通过zuul.routes.<route>.path 与 zuul.routes.<route>.url参数对的方式进行配置， 比如

  ```java
  zuul.routes.user-service.path=/user-service/**
  zuul.routes.user-service.url=http://localhost:8080/
  ```

  该配置实现了对符合／user-service／川规则的请求路径转发到 http:// localhost:8080／地址的路由 规则。 比如，当有一个请求http://localhost: 5555/user-service/hello 被发送到API网关上，由于／user-service/ hello 能够被上述配置的 path 规则匹配，所以API网关会转发请求到 http://localhost:8080/hello 地址。

- **多实例配置：** 通过 zuul.routes.<route>.path 与 zuul.routes.<route>. service Id 参数对的方式进行配置，比如：

  ```java
  zuul.routes.user-service.path=/user-service/**
  zuul.routes.user-service.serviceId=/user-service/**
  ribbon.eureka.enabled=false
  user-service.ribbon.listOfServers=http://localhost:8080/,http://localhost:8081/
  ```

​       该配 置实现了对符合／user-service／川规 则的请求路径转发到http://localhost:8080／和 http://localhost:8081／两个实例地址的路 由规则。 它的配置方式与服务路由的配置方式一样，都采用了 zuul.routes.<route>.path 与 zuul.routes.<route>.serviceid 参数对的映射方式，只是这里的 serviceId 是由用户手工命名 的服务名称，配合 ribbon.listOfServers 参数实现服务与实例的维护。由于存在多个实例，API 网关在进行路由转发时需要实现负载均衡策略，于是这里还需要 Spring Cloud Ribbon 的配合。由于在 Spring Cloud Zuul 中自带了对 Ribbon 的依赖，所以我们只需做一些配置即可，比如上面示例中关于 Ribbon 的各个配置，它们的具体作用如下所示。

- ribbon.eureka.enabled： 由于 zuul.routes.<route>.serviceid 指定的是服务名称，默认情况下 Ribbon 会根据服务发现机制来获取配置服务名对 应的实例清单。 但是，该示例并没有整合类似 Eureka 之类的服务治理框架，所以需要将该参数设置为 false，否则配置的service Id获取不到对应实例的清单
- user-service.ribbon.listOfServers ： 该 参数内容与 zuul.routes. <route>.serviceid 的配置相对应， 开头的 user-service 对应了service Id 的值， 这两个参数的配置相当于在该应用内部手工维护了服务与实例的对应关系。

​        不论是单实例还是多实例的配置方式，我们都需要为每一对映射关系指定一个名称，也就是上面配置中的＜route＞，每一个＜route＞对应了 条路由规则。每条路由规则都需要通过path属性来定义一个用来匹配客户端请求的路径表达式，并通过url或serviceID属性来指定请求表达式映射具体实例地址或服务名。

### 4.2 服务路由配置

​	对于服务路由，我们在快速入门示例中己经有过基础的介绍和体验，Spring Cloud Zuul通过与Spring Cloud Eureka的整合， 实现了对 服务实例的自动化维护， 所以在使用服务路由配置的时候，我们不需要向传统路由配置方式那样为 service Id 指定具体的服务实例地址，只需要通过zuul.routes.<route>.path与zuul.routes.<route>.serviceld参数对的方式进行配置即可。

​	比如下面的示例， 它实现了对符合／user-service ／川规则的请求路径转发到名为 user-service 的服务实例上去的路由规则。 其中＜route＞可以指定为 任意的路由名称。

```java
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceId=user-service
```

对于面向服务的路由配置， 除了使用path与service Id 映射的配置方式之外， 还有一种更简洁的配置方式：zuul.routes.<serviceld>=<path＞，其中＜service Id> 用来指定路白的具体服务名， ＜path＞用来配置匹配的请求表达式。 比如下面的例子，它的路由规则等价于上面通过path与service Id组合使用的配置方式。

```java
zuul.routes.user-service=/user-service/**
```

​	传统路由的映射方式比较直观且容易理解，API 网关直接根据请求的 URL 路径找到最匹配的path 表达式， 直接转发给该 表达式对应的url或对应service Id下配置的实例地址， 以实现外部请求的路由。 那么当采用path与service Id 以服务路由的方式实现时， 在没有配置任何实例地址的情况下， 外部请求经过 API 网关的时候，它是如何被解析 并转发到服务具体实例的呢？

​	在本章一开始，我们就提到了Zuul巧妙地整合了Eureka来实现面向服务的路由。实际上，我们可以直接将Api网关看作Eureka服务治理下的一个普通微服务应用。它除了会将自己注册到Eureka服务注册中心上之外，也会从注册中心获取所有服务以及他们的实力清单。所以，在Eureka的帮助下，API 网关服务本身就己经维护了系统中所有service Id 与实例地址的映射关系。当有外部请求到达API网关的时候，根据请求的URL路径找到最佳匹配的path规则， API 网关就可以知道 要将该请求路由到哪个具体的serviceId上去。 由于在 API 网关中己经知道serviceId对应服务实例的地址清单， 那么只需要通过Ribbon的负载均衡策略， 直接在这些清单中选择一个具体的实例进行转发就能完成路由工作了。

### 4.3 服务路由的默认规则

​	虽然通过Eureka与Zuul的整合已经为我们省去了维护服务实例清单的大量配置工作，剩下只需要再维护请求路径的匹配表达式与服务名的映射关系即可。 但是在实际的运用过程中会发现， 大部分的路由配置规则几乎都会采用服务名作为外部请求的前缀， 比如下面的例子， 其中 path路径的前缀使用了user-service，而对应的服务名称也是user-service

```java
zuul.routes.user-service.path=/user-service/**
zuul.routes.user-service.serviceId=user-service
```

​	对于这样具有规则性的 配置内容， 我们总是希望可以自动化地完成。 非常庆幸， Zuul 默认实现了这样的贴心功能， 当我们为Spring Cloud Zuul构建的 API 网关服务引入Spring Cloud Eureka之后， 它为Eureka中的每个服务都自动创建一个默认路由规则， 这些默认规则的path会使用serv_iceid配置的服务名作为请求前缀， 就如上面的例子那样。

​	由于默认情况下所有Eureka 上的服务都会被Zuul自动地创建映射关系来进行路由，这会使得一些我们不希望对外开放的服务也可能被外部访问到。 这个时候， 我们可以使用 zuul.ignored-services参数来设置一个服务名匹配表达式来定义不自动创建路由的规则。Zuul在自动创建服务路由的时候会根据该表达式来进行判断， 如果服务名匹配表达式， 那么 Zuul将跳过该 服务， 不为其创 建路由规则。 比如，设置为 zuul.ignored-services＝女的时候，Zuul 将对所有的服务都不自动创建路由规则。 在这种情况下，我们就要在配置文件中逐个为需要路由的服务添加映射规则（可以使用path与serviceId组合的配置方式， 也可使用更简洁的 zuul.routes.<serviceid>= < path＞配置方式〉，只有在配置文件中出现的映射规则会被创建路由，而从Eureka中获取的其他服务， Zuul将不会再为它们创建路由规则。

### 4.4 自定义路由映射规则

​	我们在构建微服务系统进行业务逻辑开发的时候， 为了兼容外部不同版本的客户端程序（尽量不强迫用户升级客户端〉，一般都会采用开闭原则来进行设计与开发。这使得系统在迭代过程中， 有时候会需要我们为一组互相配合的 微服务定义一个版本标识来方便管理它们的版本关系，根据这个标识我们可以很容易地知道这些服务需要一起启动并配合使用。比如可以采用类似这样的命名：userservice-vl、userservice-v2、 orderservice-vl, orderservice-v2。默认情况下，Zuul自动为服务创建的路由表达式会采用服务名作为前缀，比如针对上面的userservice-vl和userservice-v2, 它会产生／userservice-vl和／userservice-v2两个路径表达式来映射，但是这样生成出来的表达式规则较为单一，不利于通过路径规则来进行管理。通常的做法是为这些不同版本的微服务应用生成以版本代号作为路由前缀定义的路由规则，比如/vl/userservice／。这时候，通过这样具有版本号前缀的URL路径，我们就可以很容 易地通过路径表达式来归类和管理这些具有版本信息的微服务了。

​	针对上面所述的需求，如果我们的各个微服务应用都遵循了类似userservice-vl 这样的命名规则，通过－分隔的规范来定义服务名和服务版本标识的话，那么，我们可以使用Zuul中自定义服务与路由映射关系的功能，来实现为符合上述规则的微服务自动化地创 建类似／vl/userservice／六六的路由匹配规则。实现步骤非常简单，只需在API网关程序中，增加如下Bean的创建即可：


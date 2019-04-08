---
layout: post
title: 03 客户端负载均衡
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

Spring Cloud Ribbon 是 个基于 HTTP 和 TCP 的客户端负载均衡工具

<!-- more --> 

# 客户端负载均衡：Spring Cloud Ribbon

​	Spring Cloud Ribbon 是 个基于 HTTP 和 TCP 的客户端负载均衡工具， 它基于 Netflix Ribbon 实现。 通过 Spring Cloud 的封装， 可以让我们轻松地将面向服务的 REST 模板请求自动转换成客户端负载均衡的服务调用。 Spring Cloud Ribbon 虽然只是 个工具类框架，它不像服务注册中心、 配置中心、 API 网关那样需要独立部署， 但是它几乎存在于每 个Spring Cloud 构建的微服务和基础设施中。因为微服务间的调用， API 网关的请求转发等内容，实际上都是通过 Ribbon 来实现的，包括后续我们将要介绍的 Feign，它也是基于 Ribbon实现的工具。 所以， 对 Spring Cloud Ribbon 的理解和使用， 对于我们使用 Spring Cloud 来构建微服务非常重要。

​	在这一章中，我们将具体具体介绍如何使用 Ribbon 来实现客户端的负载均衡， 并且通过源码分析来了解 Ribbon 实现客户端负载均衡的基本原理。

## 1 客户端负载均衡

​	负载均衡在系统架构中是 个非常重要， 并且是不得不去实施的内容。因为负载均衡是对系统的高可用、 网络压力的缓解和处理能力扩容的重要手段之 。我们通常所说的负载均衡都指的是服务端负载均衡， 其中分为硬件负载均衡和软件负载均衡。 硬件负载均衡主要通过在服务器节点之间安装专门用于负载均衡的设备，比如日等；而软件负载均衡则是通过在服务器上安装 些具有均衡负载功能或模块的软件来完成请求分发工作， 比如Nginx 等。 不论采用硬件负载均衡还是软件负载均衡， 只要是服务端负载均衡都能以类似下图的架构方式构建起来：

​	硬件负载均衡的设备或是软件负载均衡的软件模块都会维护一个下挂可用的服务端清单，通过心跳检测来剔除故障的服务端节点以保证清单中都是可以正常访问的服务端节点。当客户端发送请求到负载均衡设备的时候， 该设备按某种算法（比如线性轮询、 按权重负载、 按流量负载等〉从维护的可用服务端清单中取出一台服务端的地址， 然后进行转发。

​	而客户端负载均衡和服务端负载均衡最大的不同点在于上面所提到的服务清单所存储的位置。 在客户端负载均衡中， 所有客户端节点都维护着自己要访问的服务端清单， 而这些 服务端的清单来自于服务注册中心，比如上一章我们介绍的Eureka服务端。同服务端负载均衡的架构类似， 在客户端负载均衡中也需要心跳去维护服务端清单的健康性， 只是这个步骤需要与服务注册中心配合完成。 在Spring Cloud实现的服务治理框架中， 默认会创建针对各个服务治理框架的Ribbon自动化整合配置，比如Eureka 中的org.springframework. cloud. netf 1 ix.ribbon. eureka. RibbonEurekaAutoConf igura tion , Consul 中的org.springframework.cloud.consul.discovery. RibbonConsulAuto-Configuratio口。 在实际使用的时候， 我们可以通过查看这两个类的实现， 以找到它们的配置详情来帮助我们更好地使用它。
	通过Spring Cloud Ribbon的封装， 我们在微服务架构中使用客户端负载均衡调用非常简单， 只需要如下两步：

- 服务提供者只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心。
- 服务消费者直接通过调用被＠LoadBalanced注解修饰过的RestTemplate来实现面向服务的接口调用。

## 2 RestTemplate详解

​	在上一章中，我们己经通过引入Ribbon实现了服务消费者的客户端负载均衡功能，读 者可以通过查看第3章中的“服务发现与消费”一节来获取实验示例。其中，我们使用了一个非常有用的对象RestTemplate。该对象会使用Ribbon的自动化配置，同时通过配置＠LoadBalanced还能够开启客户端负载均衡。之前我们演示了通过RestTemplate 实现了最简单的服务访问，下面我们将详细介绍RestTemplate针对几种不同请求类型和参数类型的服务调用实现

### 2.1 GET请求

​	在RestTemplate中，对GET请求可以通过如下两个方法进行调用实现。

#### 2.1.1 getForEntity函数

​	第一种：getForEntity函数。该方法返回的是ResponseEntity，该对象是Spring对HTTP请求响应的封装，其中主要存储了HTTP的几个重要元素，比如HTTP请求状态码的枚举对象HttpStatusC也就是我们常说的404、50。这些错误码）、在它的父类 HttpEntity中还存储着HTTP请求的头信息对象HttpHeaders以及泛型类型的请求体对象。比如下面的例子，就是访问USER-SERVER服务的／user请求，同时最后一个参数 didi会替换url中的｛1 ｝占位符，而返回的ResponseEntity对象中的body内容类型会根据第二个参数转换为String类型。

```java
RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://spring-cloud-producer/hello?name={1}", String.class, "name");
        String body = responseEntity.getBody();
```

若我们希望返回的body是一个User对象也可以这样实现

```java
RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://spring-cloud-producer/hello?name={1}", User.class, "name");
        String body = responseEntity.getBody();
```

​	上面的例子是比较常用的方法，getForEntity函数实际上提供了以下三种不同的重 载实现。

- getForEntity(String url, Class responseType, Object ... urlVariables):该方法提供了三个参数，其中url为请求的地址，responseType为请求响应体 body的包装类型，urlVariables为url中的参数绑定。GET请求的参数绑定通常使用url中拼接的方式，比如http://USER-SERVICE/user?name=didi, 我们可以像这样自己将参数拼接到url中，但更好的方法是在url中使用占位符并配合urlVariables参数实现GET请求的参数绑定，比如url定义为http://USER-SERVICE/user?name={l｝，然后可以这样来调用：getForEntity（”http://USER-SERVICE/user?name={l｝”，String.class ，"didi")，其中第三个参数didi会替换url中的｛1 ｝占位符。这里需要注意的是，由于urlVariables参数是一个数组，所以它的顺序会对应url中占位符定义的数字顺序。

- getForEntity(String url, Class responseType,Map urlVariables):该方法提供的参数中，只有urlVariables的参数类型与上面的方法不同。这里使用了Map类型，所以使用该方法进行参数绑定时需要在占位符中指定Map中参数的key值，比如url定义为http://USER-SERVICE/user?name={name}, 在Map类型的urlVariables中，我们就需要put一个key为口ame的参数来绑定url中｛name｝占位符的值，比如：

  ```java
  RestTemplate restTemplate = new RestTemplate();
   Map<String, String> map = new HashMap<>();
          map.put("name", name);
          ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://spring-cloud-producer/hello?name={name}", String.class, map);
  ```

#### 2.1.2 getForObject函数

​	第二种：getForObject函数。该方法可以理解为对getForEntity的进一步封装，它通过HttpMessageConverterExtractor对HTTP的请求响应体body内容进行对象转换，实现请求直接返回包装好的对象内容。比如：

```java
RestTemplate restTemplate = new RestTemplate();
        String result =  restTemplate.getForObject(url, String.class);
```

当body是一个User对象时，可以直接这样实现：

```java
RestTemplate restTemplate = new RestTemplate();
        User result =  restTemplate.getForObject(url, User.class);
```

当不需要关注请求响应除body外的其他内容时，该函数就非常好用，可以少一个从Response中获取body的步骤。它与getForEntity函数类似，也提供了三种不同的重载 实现。

- getForOb〕ect(String url, Class responseType, Object ... url Variables):与getForE口tity的方法类似，url参数指定访问的地址，respo口seType参数 定义该方法的返回类型，urlVariables参数为url中占位符对应的参数。
- getForObject(String url, Class respo口seType,Map urlVariables):在该函数中，使用Map类型的urlVariables替代上面数组形式的ur1 Variables,因此使用时在url中需要将占位符的名称与Map类型中的key对应设置。

- getForObject(URI url, Class responseType）：该方法使用URI对象来替代之前的url和urlVariables参数使用。

### 2.2 POST请求

​	在RestTemplate中，对POST请求时可以通过如下三个方法进行调用实现。

#### 2.2.1 postForEntity函数

​	第一种：postForEntity函数，该方法同GET请求中的getForE口tity类似，会在调用后返回ResponseEntity<T＞对象，其中T为请求响应的body类型。比如下面这个例子，使用postForEntity提交POST请求到USER-SERVICE服务的／user接口， 提交的body内容为user对象，请求响应返回的body类型为String。

```java
RestTemplate restTemplate = new RestTemplate();
User user = new User("didi", 30);
      ResponseEntity<String> responseEntity = restTemplate.postForEntity("http://spring-cloud-producer/hello?name={name}", user, String.class);
String body = responseEntity.getBody();
```

postForEntity函数也实现了三种不同的重载方法。

- postForEnt工ty(Str工ngurl, Object request, Class responseType, Object ... uriVariables)
- postForEntity(String url, Object request, Class responseType, Map uriVariables)
- postForEntity(URI url, Object request, Class responseType)

​        这些函数中的参数用法大部分与 getForEntity 致， 比如， 第 个重载函数和第二个重载函数中的 uriVariables 参数都用来对 url 中的参数进行绑定使用；res ponseType 参数是对请求响应的 body 内容的类型定义。 这里需要注意的是新增加的request 参数， 该参数可以是一个普通对象， 也可以是 个 HttpE口tity 对象。 如果是个普通对象， 而非 HttpEntity 对象的时候， RestTemplate 会将请求对象转换为个 HttpEntity 对象来处理， 其中 Object 就是 request 的类型， request 内容会被视作完整的 body 来处理；而如果 request 是 个 HttpEntity 对象， 那么就会被当作个完成的 HTTP请求对象来处理，这个request中不仅包含了body的内容，也包含了header的内容。
           第二种：postForObject函数。该方法也跟getForObject的类型类似，它的作用是简化postForEntity的后续处理。通过直接将请求响应的body内容包装成对象来返回使用，比如下面的例子：

   ```java
RestTemplate restTemplate = new RestTemplate();
User user = new User("didi", 30);
      ResponseEntity<String> responseEntity = restTemplate.postForEntity("http://spring-cloud-producer/hello?name={name}", user, String.class);
   ```

postForObject 函数也实现了三种不同的重载方法：

- postForObject(String url, Object request, Class respo口seType, Object ... uriVariables)
- postForObject(String url, Object request, Class responseType, Map uriVariables)
- postForObject(URI url, Object request, Class responseType)

这三个函数除了返回的对象类型不同， 函数的传入参数均与 postForEntity 一致，

​	第三种： postForLocation 函数。 该方法实现了以 POST 请求提交资源， 并返回新资源的 URI，比如下面的例子：

```java

User user = new User("didi", 30);
URI responseURI = restTemplate.postForLocation("http://spring-cloud-producer/hello?name={name}", user);
```

## 3 Ribbon 负载均衡大概流程

#### 3.1 LoadBalancerClient

其中LoadBalancerClient接口，有如下三个方法，

```java
public interface LoadBalancerClient {

    ServiceInstance choose(String serviceId);

    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    URI reconstructURI(ServiceInstance instance, URI original);

}
```

- `ServiceInstance choose(String serviceId)`：根据传入的服务名`serviceId`，从负载均衡器中挑选一个对应服务的实例。

- `T execute(String serviceId, LoadBalancerRequest request) throws IOException`：使用从负载均衡器中挑选出的服务实例来执行请求内容。

- `URI reconstructURI(ServiceInstance instance, URI original)`：为系统构建一个合适的“host:port”形式的URI。在分布式系统中，我们使用逻辑上的服务名称作为host来构建URI（替代服务实例的“host:port”形式）进行请求，比如：`http://myservice/path/to/service`。在该操作的定义中，前者`ServiceInstance`对象是带有host和port的具体服务实例，而后者URI对象则是使用逻辑服务名定义为host的URI，而返回的URI内容则是通过`ServiceInstance`的服务实例详情拼接出的具体“host:post”形式的请求地址。

  

`LoadBalancerClient`接口的所属包`org.springframework.cloud.client.loadbalancer`，对其内容进行整理，可以得出如下图的关系： 

![loaion](/images/SpringCloud/SpringCloud_loadBalanceAutoConfiguration.png)

从类的命名上我们知道`LoadBalancerAutoConfiguration`为实现客户端负载均衡器的自动化配置类。通过查看源码，我们可以验证这一点假设： 

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
public class LoadBalancerAutoConfiguration {

    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
            final List<RestTemplateCustomizer> customizers) {
        return new SmartInitializingSingleton() {
            @Override
            public void afterSingletonsInstantiated() {
                for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                    for (RestTemplateCustomizer customizer : customizers) {
                        customizer.customize(restTemplate);
                    }
                }
            }
        };
    }

    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(
            final LoadBalancerInterceptor loadBalancerInterceptor) {
        return new RestTemplateCustomizer() {
            @Override
            public void customize(RestTemplate restTemplate) {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            }
        };
    }

    @Bean
    public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerInterceptor(loadBalancerClient);
    }

}
```

从`LoadBalancerAutoConfiguration`类头上的注解可以知道Ribbon实现的负载均衡自动化配置需要满足下面两个条件：

- `@ConditionalOnClass(RestTemplate.class)`：`RestTemplate`类必须存在于当前工程的环境中。
- `@ConditionalOnBean(LoadBalancerClient.class)`：在Spring的Bean工程中有必须有`LoadBalancerClient`的实现Bean。



在该自动化配置类中，主要做了下面三件事：

- 创建了一个`LoadBalancerInterceptor`的Bean，用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡。
- 创建了一个`RestTemplateCustomizer`的Bean，用于给`RestTemplate`增加`LoadBalancerInterceptor`拦截器。
- 维护了一个被`@LoadBalanced`注解修饰的`RestTemplate`对象列表，并在这里进行初始化，通过调用`RestTemplateCustomizer`的实例来给需要客户端负载均衡的`RestTemplate`增加`LoadBalancerInterceptor`拦截器。

#### 3.2 通过以上我们可以知道：

- 通过注解`@RestTemplate` 和`@LoadBalanced`组织条件开启自动化配置
- 自动化配置类`LoadBalancerAutoConfiguration`向`RestTemplate`中添加`LoadBalancerInterceptor`拦截器
- 当`RestTemplate`发送请求时进入拦截器，
- 拦截器中注入`LoadBalancerClient`并把请求交给它处理



接下来，我们看看`LoadBalancerInterceptor`拦截器是如何交由`LoadBalancerClient`处理的： 

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    private LoadBalancerClient loadBalancer;

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this.loadBalancer = loadBalancer;
    }

    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        return this.loadBalancer.execute(serviceName,
                new LoadBalancerRequest<ClientHttpResponse>() {
                    @Override
                    public ClientHttpResponse apply(final ServiceInstance instance)
                            throws Exception {
                        HttpRequest serviceRequest = new ServiceRequestWrapper(request,
                                instance);
                        return execution.execute(serviceRequest, body);
                    }
                });
    }

    private class ServiceRequestWrapper extends HttpRequestWrapper {

        private final ServiceInstance instance;

        public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance) {
            super(request);
            this.instance = instance;
        }

        @Override
        public URI getURI() {
            URI uri = LoadBalancerInterceptor.this.loadBalancer.reconstructURI(
                    this.instance, getRequest().getURI());
            return uri;
        }
    }
}
```



认识`RibbonClientConfiguration`配置类，可以知道在整合时默认采用了`ZoneAwareLoadBalancer`来实现负载均衡器。 

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
        IRule rule, IPing ping) {
    ZoneAwareLoadBalancer<Server> balancer = LoadBalancerBuilder.newBuilder()
            .withClientConfig(config).withRule(rule).withPing(ping)
            .withServerListFilter(serverListFilter).withDynamicServerList(serverList)
            .buildDynamicServerListLoadBalancer();
    return balancer;
}
```



`LoadBalancerClient`的实现类：`RibbonLoadBalancerClient`。 

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
            serviceId), serverIntrospector(serviceId).getMetadata(server));

    RibbonLoadBalancerContext context = this.clientFactory
            .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

    try {
        T returnVal = request.apply(ribbonServer);
        statsRecorder.recordStats(returnVal);
        return returnVal;
    }
    catch (IOException ex) {
        statsRecorder.recordStats(ex);
        throw ex;
    }
    catch (Exception ex) {
        statsRecorder.recordStats(ex);
        ReflectionUtils.rethrowRuntimeException(ex);
    }
    return null;
}
```



可以看到，在`execute`函数的实现中，第一步做的就是通过`getServer`根据传入的服务名`serviceId`去获得具体的服务实例： 

```java
protected Server getServer(ILoadBalancer loadBalancer) {
    if (loadBalancer == null) {
        return null;
    }
    return loadBalancer.chooseServer("default");
}
```



经过源码跟踪，最终交给了ILoadBalancer类去选择服务实例 

```java
public interface ILoadBalancer {
    public void addServers(List<Server> newServers);
    public Server chooseServer(Object key);
    public void markServerDown(Server server);
    public List<Server> getReachableServers();
    public List<Server> getAllServers();
}
```

- addServers()方法是添加一个Server集合；
- chooseServer()方法是根据key去获取Server；
- markServerDown()方法用来标记某个服务下线；
- getReachableServers()获取可用的Server集合；
- getAllServers()获取所有的Server集合。 



`apply(final ServiceInstance instance)`函数中传入的`ServiceInstance`接口是对服务实例的抽象定义。在该接口中暴露了服务治理系统中每个服务实例需要提供的一些基本信息，比如：serviceId、host、port等，具体定义如下： 

```java
public interface ServiceInstance {

    String getServiceId();

    String getHost();

    int getPort();

    boolean isSecure();

    URI getUri();

    Map<String, String> getMetadata();
}
```



而上面提到的具体包装`Server`服务实例的`RibbonServer`对象就是`ServiceInstance`接口的实现，可以看到它除了包含了`Server`对象之外，还存储了服务名、是否使用https标识以及一个Map类型的元数据集合。 

```java
protected static class RibbonServer implements ServiceInstance {

    private final String serviceId;
    private final Server server;
    private final boolean secure;
    private Map<String, String> metadata;

    protected RibbonServer(String serviceId, Server server) {
        this(serviceId, server, false, Collections.<String, String> emptyMap());
    }

    protected RibbonServer(String serviceId, Server server, boolean secure,
            Map<String, String> metadata) {
        this.serviceId = serviceId;
        this.server = server;
        this.secure = secure;
        this.metadata = metadata;
    }

    // 省略实现ServiceInstance的一些获取Server信息的get函数
    ...
}
```

## 4 负载均衡器

![dynacer](/images/SpringCloud/SpringCloud_dynamicserverlistloadBalancer.png)

### 4.1 AbstractLoadBalancer

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {

    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP
    }

    public Server chooseServer() {
        return chooseServer(null);
    }

    public abstract List<Server> getServerList(ServerGroup serverGroup);

    public abstract LoadBalancerStats getLoadBalancerStats();
}
```

- `AbstractLoadBalancer`是`ILoadBalancer`接口的抽象实现。
- 定义了一个关于服务实例的分组枚举类`ServerGroup`，它包含了三种不同类型：ALL-所有服务实例、STATUS_UP-正常服务的实例、STATUS_NOT_UP-停止服务的实例；
- 实现了一个`chooseServer()`函数，该函数通过调用接口中的`chooseServer(Object key)`实现，其中参数`key`为null，表示在选择具体服务实例时忽略`key`的条件判断；
- 定义了两个抽象函数，`getServerList(ServerGroup serverGroup)`定义了根据分组类型来获取不同的服务实例列表，`getLoadBalancerStats()`定义了获取`LoadBalancerStats`对象的方法，`LoadBalancerStats`对象被用来存储负载均衡器中各个服务实例当前的属性和统计信息，这些信息非常有用，我们可以利用这些信息来观察负载均衡器的运行情况，同时这些信息也是用来制定负载均衡策略的重要依据。 



#### 4.1.1 BaseLoadBalancer

- 定义并维护了两个存储服务实例`Server`对象的列表。一个用于存储所有服务实例的清单，一个用于存储正常服务的实例清单。

  ```
  @Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
  protected volatile List<Server> allServerList = Collections
          .synchronizedList(new ArrayList<Server>());
  @Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
  protected volatile List<Server> upServerList = Collections
          .synchronizedList(new ArrayList<Server>());
  ```

- 定义了之前我们提到的用来存储负载均衡器各服务实例属性和统计信息的`LoadBalancerStats`对象。

- 定义了检查服务实例是否正常服务的`IPing`对象，在`BaseLoadBalancer`中默认为null，需要在构造时注入它的具体实现。

- 定义了检查服务实例操作的执行策略对象`IPingStrategy`，在`BaseLoadBalancer`中默认使用了该类中定义的静态内部类`SerialPingStrategy`实现。根据源码，我们可以看到该策略采用线性遍历ping服务实例的方式实现检查。该策略在当我们实现的`IPing`速度不理想，或是`Server`列表过大时，可能变的不是很为理想，这时候我们需要通过实现`IPingStrategy`接口并实现`pingServers(IPing ping, Server[] servers)`函数去扩展ping的执行策略。

```java
private static class SerialPingStrategy implements IPingStrategy {
    @Override
    public boolean[] pingServers(IPing ping, Server[] servers) {
        int numCandidates = servers.length;
        boolean[] results = new boolean[numCandidates];

        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  PingTask executing ["
                         + numCandidates + "] servers configured");
        }

        for (int i = 0; i < numCandidates; i++) {
            results[i] = false;
            try {
                if (ping != null) {
                    results[i] = ping.isAlive(servers[i]);
                }
            } catch (Throwable t) {
                logger.error("Exception while pinging Server:"
                             + servers[i], t);
            }
        }
        return results;
    }
}
```

- 定义了负载均衡的处理规则`IRule`对象，从`BaseLoadBalancer`中`chooseServer(Object key)`的实现源码，我们可以知道负载均衡器实际进行服务实例选择任务是委托给了`IRule`实例中的`choose`函数来实现。而在这里。

```
public Server chooseServer(Object key) {
    if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            return rule.choose(key);
        } catch (Throwable t) {
            return null;
        }
    }
}
```

- 启动ping任务：在`BaseLoadBalancer`的默认构造函数中，会直接启动一个用于定时检查`Server`是否健康的任务。该任务默认的执行间隔为：10秒。
- `markServerDown(Server server)`：标记某个服务实例暂停服务。

```
public void markServerDown(Server server) {
    if (server == null) {
        return;
    }
    if (!server.isAlive()) {
        return;
    }
    logger.error("LoadBalancer:  markServerDown called on ["
            + server.getId() + "]");
    server.setAlive(false);
    notifyServerStatusChangeListener(singleton(server));
}
```

### 4.2 DynamicServerListLoadBalancer

从`DynamicServerListLoadBalancer`的成员定义中，我们马上可以发现新增了一个关于服务列表的操作对象：`ServerList<T> serverListImpl`。其中泛型`T`从类名中对于T的限定`DynamicServerListLoadBalancer<T extends Server>`可以获知它是一个`Server`的子类，即代表了一个具体的服务实例的扩展类。

`ServerList`接口定义如下所示:

```java
public interface ServerList<T extends Server> {

    public List<T> getInitialListOfServers();

    public List<T> getUpdatedListOfServers();
}
```

它定义了两个抽象方法：

- `getInitialListOfServers`用于获取初始化的服务实例清单，-

- `getUpdatedListOfServers`用于获取更新的服务实例清单。

  搜索源码，我们可以整出如下图的结构： 

  ![serverList8](/images/SpringCloud/SpringCloud_serverList8.png)

从图中我们可以看到有很多个`ServerList`的实现类，那么在`DynamicServerListLoadBalancer`中的`ServerList`默认配置到底使用了哪个具体实现呢？既然在该负载均衡器中需要实现服务实例的动态更新，那么势必需要ribbon具备访问eureka来获取服务实例的能力，所以我们从Spring Cloud整合ribbon与eureka的包`org.springframework.cloud.netflix.ribbon.eureka`下探索，可以找到配置类`EurekaRibbonClientConfiguration`，在该类中可以找到看到下面创建`ServerList`实例的内容 

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config) {
    DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
            config);
    DomainExtractingServerList serverList = new DomainExtractingServerList(
            discoveryServerList, config, this.approximateZoneFromHostname);
    return serverList;   //域名提取服务列表
}
```

可以看到，这里创建的是一个`DomainExtractingServerList`实例，从下面它的源码中我们可以看到在它内部还定义了一个`ServerList list`。同时，`DomainExtractingServerList`类中对`getInitialListOfServers`和`getUpdatedListOfServers`的具体实现，其实委托给了内部定义的`ServerList list`对象，而该对象是通过创建`DomainExtractingServerList`时候，由构造函数传入的`DiscoveryEnabledNIWSServerList`实现的。 

```java
public class DomainExtractingServerList implements ServerList<DiscoveryEnabledServer> {

    private ServerList<DiscoveryEnabledServer> list;
    private IClientConfig clientConfig;
    private boolean approximateZoneFromHostname;

    public DomainExtractingServerList(ServerList<DiscoveryEnabledServer> list,
            IClientConfig clientConfig, boolean approximateZoneFromHostname) {
        this.list = list;
        this.clientConfig = clientConfig;
        this.approximateZoneFromHostname = approximateZoneFromHostname;
    }

    @Override
    public List<DiscoveryEnabledServer> getInitialListOfServers() {
        List<DiscoveryEnabledServer> servers = setZones(this.list
                .getInitialListOfServers());
        return servers;
    }

    @Override
    public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
        List<DiscoveryEnabledServer> servers = setZones(this.list
                .getUpdatedListOfServers());
        return servers;
    }
    ...
}
```

那么`DiscoveryEnabledNIWSServerList`是如何实现这两个服务实例的获取的呢？我们从源码中可以看到这两个方法都是通过该类中的一个私有函数`obtainServersViaDiscovery`来通过服务发现机制来实现服务实例的获取。 

```java
@Override
public List<DiscoveryEnabledServer> getInitialListOfServers(){
    return obtainServersViaDiscovery();
}

@Override
public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
    return obtainServersViaDiscovery();
}
```

`obtainServersViaDiscovery`的实现逻辑，主要依靠`EurekaClient`从服务注册中心中获取到具体的服务实例`InstanceInfo`列表（这里传入的`vipAddress`可以理解为逻辑上的服务名，比如“USER-SERVICE”）。接着，对这些服务实例进行遍历，将状态为“UP”（正常服务）的实例转换成`DiscoveryEnabledServer`对象，最后将这些实例组织成列表返回。 

```java
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

    if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList<DiscoveryEnabledServer>();
    }

    EurekaClient eurekaClient = eurekaClientProvider.get();
    if (vipAddresses!=null){
        for (String vipAddress : vipAddresses.split(",")) {
            List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(
                        vipAddress, isSecure, targetRegion);
            for (InstanceInfo ii : listOfInstanceInfo) {
                if (ii.getStatus().equals(InstanceStatus.UP)) {
                    // 省略了一些实例信息的加工逻辑
                    DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                    des.setZone(DiscoveryClient.getZone(ii));
                    serverList.add(des);
                }
            }
            if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                break;
            }
        }
    }
    return serverList;
}
```



#### 4.2.1 IRule 负载均衡的策略

它有三个方法，其中choose()是根据key 来获取server,setLoadBalancer()和getLoadBalancer()是用来设置和获取ILoadBalancer的

```java
public interface IRule{

    public Server choose(Object key);

    public void setLoadBalancer(ILoadBalancer lb);

    public ILoadBalancer getLoadBalancer();    
}
```

IRule有很多默认的实现类，这些实现类根据不同的算法和逻辑来处理负载均衡 

- BestAvailableRule 选择最小请求数
- ClientConfigEnabledRoundRobinRule 轮询
- RandomRule 随机选择一个server
- RoundRobinRule 轮询选择server
- RetryRule 根据轮询的方式重试
- WeightedResponseTimeRule 根据响应时间去分配一个weight ，weight越低，被选择的可能性就越低
- ZoneAvoidanceRule 根据server的zone区域和可用性来轮询选择

![IRULE2](/images/SpringCloud/SpringCloud_IRULE2.png)

基本流程：

1. 通过LoadBalancerClient来实现的，
2. LoadBalancerClient具体交给了BaseLoadBalancer来处理，
3. BaseLoadBalancer通过配置IRule、IPing等信息，并向EurekaClient获取注册列表的信息，并默认10秒一次向EurekaClient发送“ping”,进而检查是否更新服务列表，
4. 得到注册列表后，BaseLoadBalancer根据IRule的策略进行负载均衡。
  
   
  
   
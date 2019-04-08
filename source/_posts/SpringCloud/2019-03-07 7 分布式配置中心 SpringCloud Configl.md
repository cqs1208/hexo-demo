---
layout: post
title: 07 分布式配置中心
tags:
- Springcloud
categories: Springcloud
description: springcloud 
---

Spring Cloud Config是SpringCloud团队创建的一个全新项目

<!-- more --> 

# 配置中心 Config

​	Spring Cloud Config是SpringCloud团队创建的一个全新项目，用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持，它分为服务端与客户端两个部分。其中服务端也称为分布式配置中心，它是个独立的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密／解密信息等访问接口；而客户端则是微服务架构中的各个微服务应用或基础设施，它们通过指定的配置中心来管理应用资源与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息。SpringCloud Config实现了对服务端和客户端中环境变量和属性配置的抽象映射，所以它除了适用于Spring构建的应用程序之外，也可以在任何其他语言运行的应用程序中使用。由于SpringCloud Config实现的配置中心默认采用Git来存储配置信息，所以使用SpringCloud Config构建的配置服务器，天然就支持对微服务应用配置信息的版本管理，并且可以通过Git客户端工具来方便地管理和访问配置内容。当然它也提供了对其他存储方式的支持，比如SYN仓库、本地化文件系统。接下来，我们从个简单的入门示例开始学习SpringCloud Conftg服务端以及客户端的详细构建与使用方法。

## 1 快速入门

​	在本节中，我们将演示如何构建个基于Git存储的分布式配置中心，同时对配置的详细规则进行讲解，并在客户端中演示如何通过配置指定微服务应用的所属配置中心，并让其能够从配置中心获取配置信息并绑定到代码中的整个过程。

## 2 构建配置中心

​	通过Spring Cloud Config 构建一个分布式配置中心非常简单， 只需要以下三步。

1. 创建一个基础的Spring Boot 工程， 命名为config-server ，并在pom.xml 中引入下面的依赖：

   ```java
    <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
   ```

2. 创建Spring Boot 的程序主类， 并添加＠ EnableConfigServer 注解， 开启SpringCloud Config 的服务端功能。

   ```java
   @SpringBootApplication
   @EnableConfigServer
   public class ConfigApplication {
       public static void main(String[] args) {
           SpringApplication.run(ConfigApplication.class, args);
       }
   }
   ```

3. 在applicatio口.properties 中添加配置服务的基本信息以及Git仓库的相关信息， 如下所示：

   ```java
   spring.application.name=config-server
   server.port=7001
   
   spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/didispace/
   spring.cloud.config.server.git.search-paths=config
   spring.cloud.config.server.git.username=849405266@qq.com
   spring.cloud.config.server.git.password=chen1208
   ```

   其中Git的配置信息分别表示如下内容。

   - spring.cloud.config.server.git.uri ：配置Git仓库位置。
   - spring.cloud.config.server.git.searchPaths ：配置仓库路径下的相对搜索位置， 可以配置多个。
   - spring.cloud.config.server.git.usernarne ：访问Git仓库的用户名。
   - spring.cloud.config.server.git.password ：访问Git仓库的用户密码。

​        到这里， 使用 个通过 Spring Cloud Config 实现， 并使用Git管理配置内容的分布式 配置中心就完成了。 我们可以将该应用先启动起来， 确保没有错误产生， 然后进入下面的学习内容。

## 3 配置规则详解

​	为了验证上面完成的分布式配置中心 config-server，根据Git配置信息中指定的仓库位置， 在 https://github.com/chenqingsong1208/springcloudconfig/下创建一个config目录作为配置仓库，并根据不同环境新建下面4个配置文件：

- didispace.properties
- didispace-dev.properties
- didispace-test.properties
- didispace-prod.properties

在这4个配置文件中均设置了一个from属性，并为每个配置文件分别设置了不同的值，如下所示：

- from=git-default-1.0
- from=git-dev-1.0
- from=git-test-1.0
- from=git-prod-1.0

​         为了测试版本控制，在该Git仓库的master分支中，我们为from属性加入1.0的后缀，同时创建一个config-label-test分支，并将各配置文件中的值用2.0作为后缀。

​	完成了这些准备工作之后，我们就可以通过浏览器、POSTMAN或CURL等工具直接来访问我们的配置内容了。访问配置信息的URL与配置文件的映射关系如下所示：

- /{application}/{profile} [/{label}
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application｝－｛profile}.properties
- /{label}/{application}   -  {profile}.properties

​        上面的url会映射｛application}-{profile}.properties对应的配置文件，其中｛label｝对应Git上不同的分支，默认为master。我们可以尝试构造不同的url来访问 不同的配置内容，比如，要访问master分支，didispace应用的prod 环境，就可以访问这个url:http://localhost:7001/didispace/prod/master，并获得如下返回信息：

```java
{
  "name": "didispace",
  "profiles": [
    "prod"
  ],
  "label": "master",
  "version": "33d891a68a958ad1954f239b2c0d364cc81e926a",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/chenqingsong1208/springcloudconfig/config/didispace-prod.properties",
      "source": {
        "from": "git-prod-1.0"
      }
    },
    {
      "name": "https://github.com/chenqingsong1208/springcloudconfig/config/didispace.properties",
      "source": {
        "from": "git-default-1.0"
      }
    }
  ]
}
```

​        我们可以看到该JSON 中返回了应用名 didi space，环境名 prod，分支名 mater, 以及 default 环境和 prod 环境的配置内容。 另外， 之前没有提到过的 version，从下图我们可以观察到， 它对应的是在 Git 上的 commit 号。

![configGitCommit](/images/SpringCloud/SpringCloud_configGitCommit.png)

同时， 我们可以看到 config-server 的控制台中还输出了下面的内容， 配置服务器在从 Git 中获取配置信息后， 会存储 份在 config-server 的文件系统中， 实质上config-server 是通过 git clone 命
令将配置内容复制了一份在本地存储，然后读取这些内容并返回给微服务应用进行加载。

![configLogger](/images/SpringCloud/SpringCloud_configLogger.png)

​       con fig-server通过Git在本地仓库暂存，可以有效防止当Git仓库出现故障而引起无法加载配置信息的情况。我们可以通过断开网络，再次发起http://localhost: 7001/didispace/prod/master请求，在控制台中可输出如下内容。可以看到，config-server提示无法从远程获取该分支内容的报错信息：Couldnot pull remote for config-label-test，但是它依然会为该请求返回配置内容，这些内容源于之前访问时存于config-server本地文件系统中的配置内容。

![configLoggerClose](/images/SpringCloud/SpringCloud_configLoggerClose.png)

## 4 客户端配置映射

​        在完成了上述验证之后，确定配置服务中心己经正常运作，下面我们尝试如何在微服 务应用中获取上述配置信息。

- 创建一个Spring Boot应用，命名为config-client，并在pom.xml中引入下述依赖：

  ```java
  @SpringBootApplication
  public class ConfigClientApplication {
      public static void main(String[] args) {
          SpringApplication.run(ConfigClientApplication.class, args);
      }
  }
  ```

- 创建bootstrap.properties 配置，来指定获取配置文件的 config-server 位置，例如：

  ```java
  spring.application.name=didispace
  spring.cloud.config.profile=dev
  spring.cloud.config.label=master
  spring.cloud.config.uri=http://localhost:7001/
  
  server.port=7002
  ```

- 上述配置参数与Git中存储的配置文件中各个部分的对应关系如下所示。

  - spri口g.applicatio口．口ame：对应配置文件规则中的｛application｝部分。
  - spring.cloud.config.profile ：对应配置文件规则中的｛profile ｝部分。
  - spring.cloud.config.label ：对应配置文件规则中的｛label｝部分。
  - spri口g.cloud.config.uri：配置中心 config-server的地址。

​         这里需要格外注意，上面这些属性必须配置在bootstrap.properties 中，这样config-server中的配置信息才能被正确加载。在第2章中，我们详细说明了Spring Boot 对配置文件的加载顺序，对于本应用jar包之外的配置文件加载会优先于应用jar包内的配置 内容，而通过bootstrap.properties对 config-server的配置，使得该应用会从config-server中获取一些外部配置信息，这些信息的优先级比本地的内容要高，从 而实现了外部化配置。

- 创建一 个RESTful接口来返回配置中心的from属性，通过＠Value （ ” $｛from｝ ” ）绑定 配置服务中 配置的from属性，具体实现如下：

  ```java
  @RefreshScope
  @RestController
  public class TestController {
  
      @Value("${from}")
      private String from;
  
      @RequestMapping("/from")
      public String from(){
          return this.from;
      }
  }
  ```

- 除了通过＠Value注解绑定注入之外，也可以通过Environment对象来获取配置 属性，比如：

  ```java
  @RefreshScope
  @RestController
  public class TestController {
  
      @Autowired
      private Environment environment;
  
      @RequestMapping("/from")
      public String from(){
          return environment.getProperty("from", "undefined");
      }
  }
  ```

## 5 服务端详解

​	在上一节中，我们实现了一个具备基本结构的配置管理服务端和客户端，同时讲解了其中一些配置的基本原理和规则。在本节中，我们将进一步介绍Spring Cloud Config 服务端的一些相关知识和用法。

## 6 基础架构

​	在动手实践了上面关于SpringCloud Conf堪的基础入门内容之后，在这里我们深入理解一下是如何运作起来的。下图所示的是上一节我们所构建案例的基本结构。其中，主要包含下面几个要素。

- 远程Git仓库：用来存储配置文件的地方，上例中我们用来存储针对应用名为 didispace的多环境配置文件：didispace-{profile}.propertieso

- Config Server：这是我们上面构建的分布式配置中心，config-server工程，在 该工程中指定了所要连接的Git仓库位置以及账户、密码等连接信息。

- 本地Git仓库：在ConfigServer的文件系统中，每次客户端请求获取配置信息时， Config Server从Git仓库中获取最新配置到本地，然后在本地Git仓库中读取并返回。当远程仓库无法获取时，直接将本地内容返回。

- Service A、ServiceB：具体的微服务应用，它们指定了ConfigServer的地址，从而实现从外部化获取应用自己要用的配置信息。这些应用在启动的时候，会向Config Server请求获取配置信息来进行加载。

  ![gitConfigServer](/images/SpringCloud/SpringCloud_gitConfigServer.png)

客户端应用从配置管理中获取配置信息遵从下面的执行流程：

1. 应用启动时，根据bootstrap.properties中配置的应用名｛applicatio川、环境名｛profile｝、分支名｛label｝，向ConfigServer请求获取配置信息。
2. Config Server根据自己维护的Git仓库信息和客户端传递过来的配置定位信息去查找配置信息。
3. 通过gitclone命令将找到的配置信息下载到ConfigServer的文件系统中。
4. Config Server创建Spring的ApplicationContext实例，并从Git本地仓库中 加载配置文件，最后将这些配置内容读取出来返回给客户端应用。
5. 客户端应用在获得外部配置文件后加载到客户端的ApplicationContext实例，该配置内容的优先级高于客户端Jar包内部的配置内容，所以在Jar包中重复的内容将不再被加载。

​        Config Server巧妙地通过git clone将配置信息存于本地，起到了缓存的作用，即使当git服务无法访问的时候，依然可以取ConfigServer中的缓存内容进行使用

## 7 git配置仓库

​	在 Spring Cloud Config 的服务端，对于配置仓库的默认实现采用了 Git。 Git 非常适用 于存储配置内容，它可以非常方便地使用各种第二方工具来对其内容进行管理更新和版本化， 同时 Git 仓库的 Hook 功能还可以帮助我们实时地监控配置内容的修改。 其中， Git 自身的版本控制功能正是其他一些配置中心所欠缺的， 通过 Git 进行存储意味着， 一个应用的不同部署实例可以从 Spring Cloud Config 的服务端获取不同的版本配置，从而支持一些特殊的应用场景。

​	由于 Spring Cloud Config 中默认使用 Git，所以对于 Git 的配置也非常简单， 只需在Config Server 的 application.properties 中设置 spri口g.cloud.config.server. git.uri 属性，为其指定 Git 仓库的网络地址和账户信息即可， 比如在快速入门一节中的例子：

```java
spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/didispace/
spring.cloud.config.server.git.search-paths=config
spring.cloud.config.server.git.username=849405266@qq.com
spring.cloud.config.server.git.password=chen1208
```

​	如果我们将该值通过 file ：／／前缀来设置为一个文件地址（在 Windows 系统中，需要使用 file ：／／／来定位文件内容），那么它将以本地仓库的方式运行，这样我们就可以脱离 Git 服务端来快速进行调试与开发， 比如：

```java
spring.cloud.config.server.git.uri=file://${user.home}/config-repo
```

​	其中，$｛ user.home ）代表当前用户的所属目录。 file ：／／配置的本地文件系统方式虽然对于本地开发调试时使用非常方便，但是该方式也仅用于开发与测试， 在生产环境中 请务必搭建自己的 Git 仓库来存储配置资源。

### 7.1 占位符配置URL

​	{application ｝、｛ profile ｝、｛ label ）这些占位符除了用于标识配置文件的规则之外，还可以用于 Conf1g Server 中对 Git 仓库地址的 URI 配置。 比如， 我们可以通过 {application｝占位符来实现一个应用对应一个 Git 仓库目录的配置效果，具体配置实现如下：

```java
spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/{application}/
spring.cloud.config.server.git.search-paths=config
spring.cloud.config.server.git.username=849405266@qq.com
spring.cloud.config.server.git.password=chen1208
```

​	其中，｛ application｝代表了应用名， 所以当客户端应用向 Config Server 发起获取配置的请求时， Config Server 会根据客户端的 spri口g.application. 口ame 信息来填充{application｝占位符以定位配置资源的存储位置， 从而实现根据微服务应用的属性动态获取不同的配置。另外，在这些占位符中，{label}参数较为特别，如果git的分支和标签包含“/“，那么{label}参数在HTTP的URL中应该使用”（_）“来替代，一避免改变了URL含义，指向其他的URL资源。

​	当我们使用Git作为配置中心来存储各个微服务应用配置文件的时候，该功能会变得非常有用，通过在URL中使用占位符可以帮助我们规划和实现通用的仓库配置。例如，我们可以对微服务应用做如下规划。

- **代码库：** 使用服务名作为Git仓库名称，比如会员服务的代码库https://github.com/chenqingsong1208/didispace/config
- **配置库：** 使用服务名加上－config后缀作为Git仓库名称，比如上面会员服务对应的 配置库地址位置https://github.com/chenqingsong1208/didispace/config-config

这时，我们就可以使用spri口g.cloud.config.server.git.uri=https://github.com/chenqingsong1208/didispace/{application}-config配置，来同时匹配多个不 同服务的配置仓库。

## 8 本地仓库

​	在使用了Git或SVN仓库之后， 文件都会在ConfigServer的本地文件系统中存储 份，这些文件默认会被存储于以 config-repo 为前缀的临时目录中， 比如名为/tmp/config-repo－＜随机数＞的目录。 由于其随机性以及临时目录的特性， 可能会有一些不可预知的后果， 为了避免将来可能会出现的问题， 最好的办法就是指定 个固定的位置 来存储这些重要信息。我们只需要通过spring.cloud.config.server.git.basedir或spring.cloud.config.server.svn.basedir来配置一个我们准备好的目录即可

## 9 本地文件系统

​	Spring Cloud Config也提供了 种不使用Git仓库或SVN仓库的存储方式， 而是使用 本地文件系统的存储方式来保存配置信息。 实现方式也非常简单， 我们只需要设置属性spring. profiles. acti ve= nati ve, Config Server会默认从应用的 src/main/ resource 目录下搜索配置文件。 如果需要指定搜索配置文件的路径， 我们可以通过spring.cloud.config.server.native.searchLocations 属性来指定具体的配 置文件位置。

## 10 健康监测

​	使用占位符来配置URI的时候，很有可能在控制台中出现类似这样的警告信息（以 spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/didispace/config-config

​	而引起该警告的原因是由于SpringCloud Config的服务端为spring-boot­actuator模块的／health端点实现了对应的健康检测器：org.springframework. cloud.config.server.config.ConfigServerHealthindicator。在该检测器中， 默认构建了一个application为app的仓库。而根据之前的配置规则：｛applicatio叫一config.git，该检测器会不断地检查https://github.com/chenqingsong1208/didispace/config-config仓库是否可以连通。此时，我们可以访问配置中心的／health端点来查看它的健康状态，具体返回信息如下：

```java
{
  "status": "DOWN"
}
```

从／health端点的返回信息中，我们可以看到，由于无法连通https://github.com/chenqingsong1208/didispace/config-config仓库，使得配置中心的可用状态是DOWN。虽然我们依然可以通过U阳的方式访问该配置中心，但是当将配置中心服务化使用的时候， 该状态将影响它的服务可用性判断。所以，我们需要了解它的健康检测配置，并让它的健康检测器正常工作起来.

​	根据默认配置规则，我们可以直接在Git仓库中创建一个名为app-config的配置库， 让健康检测器能够访问到它。另外，也可以配置一个实际存在的仓库来进行连通检测，比如下面的配置，它实现了通过连接check-repo-config仓库来进行健康监测：

```java
spring.application.name=springcloudconfig
server.port=7001

spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/{application}/config

spring.cloud.config.server.health.repositories.check.name=didispace
spring.cloud.config.server.health.repositories.check.label=master
spring.cloud.config.server.health.repositories.check.profiles=default
```

​	由于健康检测的repositories是个Map对象，所以实际使用时我们可以配置多个。而每个配置中包含了与定位仓库地址时类似的三个元素。

- name: 应用名
- label： 分支名
- Profiles：环境名

如果我们不想使用健康检测器，也可以通过使用 spring.cloud.config. server.health.enabled=false 参数设置 来关闭它 。

## 11 属性覆盖

​	Config Server还有 一 个 “ 属性覆盖 ”的特性，它可以让开发人员为 所有的应用提供 配 置属性，只需要通过spring.cloud.config.server.overrides 属性来设置键值对的参数，这些参数会以Map的方式加载到客户端的配置中。 比如：

```java
spring.cloud.config.server.overrides.name=didi
spring.cloud.config.server.overrides.from=shanghai
```

​	通过该属性配置的参数，不会被Spring Cloud的客户端修改，并且Spring Cloud 客户端从Config Server中获取配置信息时， 都会取得这些配置信息。 利用该特性可以方便地为 Spring Cloud 应用配置 一些共同属性或是默认属性。 当然，这些属性并非强制的，我们可以通过改变客户端中更高优先级的配置 方式（比如，配置 环境变量或是系统属性〉，来选择是否使用Config Server 提供的默认值 。

## 12 安全保护

​	由于配置中心存储的内容比较敏感，做一 定的安全处理是必需的。 为配置中心 实现 安全保护的方式有很多，比如物理网络限制、 0Auth2授权等。不过，由于我们的微服务应用和配置中心都构建于Spring Boot基础上， 所以与Spring Security结合使用会更加方便。

​	我们只需要在配置中心的pom.xml中加入spring-boot-starter-security依赖，不需要做任何其他改动就能实现对配置中心访问的安全保护 。

```java
<dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-security</artifactId>
 </dependency>
```

默认情况下，我们 可以获得一 个名为 user的用户 ，并且在配置中心启动的时候，在日志中打印出该用户的随机密码，

​	大多数情况下，我们并不会使用随机生成密码的机制。 我们可以在配置文件中指定用户和密码， 比如：

```java
security.user.name=user
security.user.password=chen1208
```

​	由于我们己经为config-server设置了安全保护，如果这时候连接到配置中心的客户端中没有设置对应的安全信息，在获取配置信息时会返回401错误。所以，需要通过配置的方式在客户端中加入安全信息来通过校验，比如：

```java
spring.cloud.config.username=user
spring.cloud.config.password=chen1208
```

## 13 加密解密

​	在微服务架构中， 我们通常会采用 Dev。”的组织方式来降低因团队间沟通造成的巨大成本，以加速微服务应用的交付能力。 这就使得原本由运维团队控制的线上信息将交由微服务所属组织的成员自行维护， 其中将会包括大量的敏感信息， 比如数据库的账户与密码等。 显然，如果我们直接将敏感信息以明文的方式存储于微服务应用的配置文件中是非常危险的。针对这个问题，Spring Cloud Config 提供了对属性进行加密解密的功能，以保护配置文件中的信息安全。 

## 14 高可用配置

​	当要将配置中心部署到生产环境中时，与服务注册中心一样，我们也希望它是一个高可用的应用。SpringCloud Config实现服务端的高可用非常简单，主要有以下两种方式。

- **传统模式：** 不需要为这些服务端做任何额外的配置，只需要遵守一个配置规则，将所有的ConfigServer都指向同一个Git仓库，这样所有的配置内容就通过统一的共享文件系统来维护。而客户端在指定ConfigServer位置时，只需要配置ConfigServer上层的负载均衡设备地址即可
- **服务模式：** 除了上面这种传统的实现模式之外， 我们也可以将 Config Server 作为一 个普通的微服务应用， 纳入 Eureka 的服务治理体系中。 这样我们的微服务应用就可以通过配置中心的服务名来获取配置信息， 这种方式比起传统的实现模式来说更加 有利于维护， 因为对于服务端的负载均衡配置和客户端的配置中心指定都通过服务治理机制一并解决了， 既实现了高可用， 也实现了自维护。 由于这部分的实现需要客户端的配合， 具体示例读者可详细阅读 “客户端详解”一节中的 “服务化配置中 ”
  心 小节。

## 15 客户端详情

​	在学习了关于SpringCloud Config服务端的大量配置和使用细节之后，接下来我们将通过本节内容继续学习SpringCloud Config客户端的使用与配置。

## 16 URL指定配置中心

​	Spring Cloud Conf屯的客户端在启动的时候， 默认会从工程的class path中加载配置信息并启动应用。 只有当我们配置 spring.cloud.config.uri的时候， 客户端应用才会尝试连接SpringCloud Config的服务端来获取远程配置信息并初始化Spring环境配置。 同时， 我们必须将该参数配置在bootstrap.properties、 环境变量或是其他优先级高于应用Jar包内的配置信息中，才能正确加载到远程配置。若不指定 spring.cloud.config. uri 参数的话， Spring Cloud Config 的客 户 端会默 认尝试连接 http:// localhost:8888 。

​	在之前的快速入门示例中， 我们就是以这种方式实现的， 就如下面的配置， 其中 spring.applicatio口 .name和spring.cloud.config.profile用于定位配置信息。

```java
spring.application.name=didispace
spring.cloud.config.profile=dev
spring.cloud.config.label=master
spring.cloud.config.uri=http://localhost:7001/
```

## 17 配置中心服务化

​	在第3章中， 我们已经学会了如何构建服务 注册中心 、如何发现与注册服务。 那么Config Server是否也能以服务的方式注册到服务中心， 并被其他应用所发现来实现配置信息的获取呢？答案是肯定的。 在SpringCloud中， 我们也可以把Config Server视为微服务架构中与其他业务服务 样的 个基本单元。
	下面， 我们就来详细介绍如何将ConfigServer注册到服务中心， 并通过服务发现来访问ConfigServer并获取Git仓库中的配置信息。 下面的内容将基于快速入门中实现的 config-server和config一 client工程来进行改造实现。

### 17.1 服务端配置

- 在config-server的 pom.xml中增加spring-cloud-starter-eureka依赖。以实现将分布式配置中心加入Eureka的服务治理体系。

  ```java
   <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-eureka</artifactId>
   </dependency>
  ```

- 在 applicatio口.properties 中配置参数 eureka.client.serviceUrl. defaultZone 以指定服务注册中心的位置， 详细内容如下：

  ```java
  spring.application.name=config-server
  server.port=7001
  
  eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
  
  spring.cloud.config.server.git.uri=https://github.com/chenqingsong1208/{application}/
  spring.cloud.config.server.git.search-paths=config
  spring.cloud.config.server.git.username=849405266@qq.com
  spring.cloud.config.server.git.password=chen1208
  ```

- 在应用主类中，新增＠ EnableDiscoveryClient 注解，用来将 config-server 注册到上面配置的服务注册中心上去。

  ```java
  @EnableDiscoveryClient
  @SpringBootApplication
  @EnableConfigServer
  public class ConfigApplication {
      public static void main(String[] args) {
          SpringApplication.run(ConfigApplication.class, args);
      }
  }
  ```

- 启动该应用

## 17.2 客户端配置

- 在 config-clie川的 pom.xml 中新增 spring-cloud-starter-eureka 依赖， 以实现客户端发现 config-server 服务， 具体配置如下：

  ```java
  <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-eureka</artifactId>
          </dependency>
  ```

- 在bootstrap.properties中，按如下配置:

  ```java
  spring.application.name=didispace
  server.port=7002
  
  eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
  
  spring.cloud.config.discovery.enabled=true
  spring.cloud.config.discovery.service-id=config-server
  spring.cloud.config.profile=dev
  ```

  其中， 通过eureka.clie口t.serviceUrl.defaultZone 参数指定服务注册中心， 用于服务的注册与发现； 再将 spring.cloud.config.discovery.enabled 参数设置为 true， 开启通过服务来访问 Conf1g Server 的功能；最后利用spring.cloud.config.discovery.serviceid 参数来指定Config Server 注 册的服务名。这里的spring.application.name 和 spri口g.cloud.config.profile 如之前通过URI 的方式访问的时候 样， 用来定位 Git 中的资源。

- 在应用主类中，增加＠ EnableDiscoveryClie川注解，用来发现 config-server 服务， 利用其来加载应用配置：

  ```java
  @EnableDiscoveryClient
  @SpringBootApplication
  public class ConfigClientApplication {
      public static void main(String[] args) {
          SpringApplication.run(ConfigClientApplication.class, args);
      }
  }
  ```

- 沿用之前我们创建的Controller来加载Git中的配置信息

  ```java
  @RefreshScope
  @RestController
  public class TestController {
      @Value("${from}")
      private String from;
  
      @RequestMapping("/from")
      public String from(){
          return this.from;
      }
  }
  ```

- 访问客户端应用提供的服务http://localhost:7002/from，此时，我们会返回在Git仓库中didispace-dev.properties文件中配置的from属性内容：
  "git-deb-1.0"

## 18 失败快速响应与重试

​	Spring Cloud Config的客户端会预先加载很多其他信息，然后再开始连接ConfigServer进行属性的注入。当我们构建的应用较为复杂的时候，可能在连接ConfigServer之前花费较长的启动时间，而在一些特殊场景下，我们又希望可以快速知道当前应用是否能顺利地从ConfigServer获取到配置信息，这对在初期构建调试环境时，可以减少很多等待启动的时间。要实现客户端优先判断ConfigServer获取是否正常，并快速响应失败内容，只需在 bootstrap.properties中配置参数spring.cloud.config.failFast=true即可。

​	我们可以实验一下，在未配置该参数前，不启动 Config Server，直接启动客户端应用，可以获得下面的报错信息。 同时，在报错之前，可以看到客户端应用已经加载了很多内容，
比如 Controller 的请求等。

​	加上 spring.cloud.config.failFast=true 参数之后，再启动客户端应用，可 以获得下面的报错信息，并且前置的加载内容少了很多，这样通过该参数有效避免了当Config Server 配置有误时，不需要多等待前置的一些加载时间，实现了快速返回失败信息。
	上面，我们演示了当 Config Server 岩机或是客户端配置不正确导致连接不到而启动失
败的情况，快速响应的配置可以发挥比较好的效果。 但是，若只是因为网络波动等其他间
歇性原因导致的问题，直接启动失败似乎代价有些高。 所以，Config 客户端还提供了自动重试的功能，在开启重试功能前，先确保己经配置了 spring.cloud.config. f ailFas·t=true，再进行下面的操作。

- 在客户端的 pom.xml 中增加 spri口g-retry 和 spring-boot-starter-aop 依赖，具体如下：

  ```java
    <dependency>
              <groupId>org.springframework.retry</groupId>
              <artifactId>spring-retry</artifactId>
          </dependency>
  
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-aop</artifactId>
          </dependency>
  ```

- 不需要再做其他任何配置，启动客户端应用

​        若对默认的最大重试次数和重试问隔等设置不满意，还可以通过下面的参数进行调整。

- spring.cloud.config.retry.multiplier：初始重试问隔时间（单位为毫秒〉，默认为1000 毫秒
- spring.cloud.config.retry.initial-interval：下 一间隔的乘数，默认为1.1，所以当最初间隔是 1000 毫秒时，下 一次失败后的间隔为 1100 毫秒。
- spri口g.cloud.co口fig.retry.max-interval： 最大间隔时间，默认为2000毫秒。
- spring.cloud.config.retry.rnax-atternpts：最大重试次数，默认为6次。

## 19 获取远程配置

​	在入门示例中，我们对｛application｝、｛profile ）、｛label ）这些参数己经有了一定的了解。在 Git 仓库中，一个形如｛application}-{profile} . proper.ties或{application}-{profile}.yml的配置文件，通过 URL请求和客户端配置的访问对应可以总结如下。

- 通过向Conf1g Server发送GET请求以直接的方式获取，可用下面的链接形式。
  - 不带｛label｝分支信息，默认访问master分支，可使用：
    - ／｛application}-{profile }.yml
    - ／｛application｝－｛profile}.properties
  - 带｛label｝分支信息，可使用：
    - ／｛label}/{application}-{profile}.yrnl
    - ／｛application}/{profile} [/{label}] 
    - ／｛label}/{application}-{profile}.properties
- 通过客户端配置方式加载的内容如下所示。
  - spring.application.口arne：对应配置文件中的｛application｝内容。
  - spri口g.cloud.co口fig.profile：对应配置文件中｛profile｝内容。
  - spring.cloud.co口fig.label：对应分支内容，如不配置，默认为rnaster

## 20 动态刷新配置

​	有时候，我们需要对配置内容做些实时更新，那么SpringCloud Config是否可以实现呢？答案显然是可以的。下面，我们以快速入门中的示例作为基础，看看如何进行改造来实现配置内容的实时更新。

首先，回顾一下，当前我们已经实现了哪些内容

- config：定义在git仓库中的一个目录，其中存储了应用名为didispace的多环境配置文件，配置文件中有一个from参数
- config-server：配置了Git仓库的服务端。
- config-client：指定了config-server为配置中心的客户端，应用名为didispace,用来访问配置服务器已获取信息。该应用中提供了一个/from接口，它会获取config/didispace-dev.properties中的from属性返回

​        在改造程序之前，我们先将config-server和config-client都启动起来，并访问客户端提供的REST接口http://localhost:7002/frorn来获取配置信息，获得的返回内容为git-dev-1.0。接着，我们可以尝试使用Git工具修改当前配置的内容，比如，将config-repo/didispace-dev.properties中的from的值从frorn=git­dev-1.0修改为frorn=git-dev-2.0，再访问http://localhost:7002/frorn，可以看到其返回内容还是git-dev-1.0。

​	接下来，我们将在config-client端做一些改造以实现配置信息的动态刷新。

- 在config-client的porn.xrnl中新增spring-boot-starter-actuator监控模块。其中包含了／refresh端点的实现，该端点将用于实现客户端应用配置信息的重新获取与刷新。

  ```java
  <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-actuator</artifactId>
          </dependency>
  ```

- 重新启动config-client,访问一次http://localhost:7002/from,可以看到当前的配置值

- 修改git仓库config/didispace-dev.properties文件中from的值，

- 再访问一次http://localhost:7002/from，可以看到配置值没有改变

- 通过POST请求发送到http://localhost:7002/refresh，我们可以看到返回内容如下，代表from参数的配置内容被更新了：

- 再访问一次http://localhost:7002/from,可以看到配置值已经是更新后的值了。

​       通过上面的介绍，大家不难想到，该功能还可以同Git仓库的WebHook功能进行关联，当有Git提交变化时，就给对应的配置主机发送／refresh请求来实现配置信息的实时更新。但是，当我们的系统发展壮大之后，维护这样的刷新清单也将成为个非常大的负担，而且很容易犯错，那么有什么办法可以解决这个复杂度呢？后续我们将介绍如何通过Spring Cloud Bus来实现以消息总线的方式进行配置变更的通知，并完成集群上的批量配置 更新。

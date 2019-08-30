# 	:triangular_flag_on_post:	编写Zuul微服务网关

<p id="t"></p>

:arrow_down:[编写Zuul微服务网关](#a1)

:arrow_down:[管理端点](#a2)

:arrow_down:[路由配置详解](#a3)

:arrow_down:[-](#a4)

<b id="a1"></b>

### :arrow_up_small: 编写Zuul微服务网关

:arrow_up:[返回目录](#t)

经过前文的讲解，微服务架构已经初具雏形，但还有一些问题——不同的微服务一般会有不同的网络地址，而外部客户端（例如手机APP）可能需要调用多个服务的接口才能完成一个业务需求。例如一个电影购票的手机APP，可能调用多个微服务的接口才能完成一次购票的业务流程。如下

![](http://s15.sinaimg.cn/mw690/001l8XD7zy76r0c7Xsi0e&690)

如果让客户端直接与各个微服务通信，会有以下问题：

* 客户端会多次请求不同的微服务，增加了客户端的复杂性。
* 存在跨域请求，在一定场景下处理相对复杂。
* 认证复杂，每个服务都需要独立认证。
* 难以重构，随着项目的迭代，可能需要重新划分微服务。例如，可能将多个服务合并成一个或者将一个服务拆分成多个。如果客户端直接与微服务通信，那么重构将很难实施。

* 某些微服务可能使用了对防火墙/浏览器不友好的协议，直接访问时会有一定的困难。

![](http://ws1.sinaimg.cn/large/006tNc79ly1fr50yvfb4nj318g12ytdx.jpg)

使用微服务网关的好处：

微服务网关封装了应用程序的内部结构，客户端只用跟网关交互，而无须直接调用特定微服务的接口。这样，开发就可以得到简化。不仅如此，使用微服务网关还有以下优点：

* 易于监控。可在微服务网关收集监控数据并将其推送到外部系统进行分析。
* 易于认证。可在微服务网关上进行认证，然后再将请求转发到后端的微服务，而无须在每个微服务中进行认证。
* 减少了客户端与各个微服务之间的交互次数。

Zuul是Netflix开源的微服务网关，它可以和Eureka，Ribbon，Hystrix等组件配合使用，Zuul的核心是一系列的过滤器。这些过滤器可以完成以下功能：

* 身份认证与安全：识别每个资源的验证要求，并拒绝那些与要求不符的请求。
* 审查与监控：在边缘位置追踪有意义的数据和统计结果，从而带来精确的生产视图。
* 动态路由：动态地将请求路由到不同的后端集群。·压力测试：逐渐增加指向集群的流量，以了解性能。
* 负载分配：为每一种负载类型分配对应容量，并弃用超出限定值的请求。
* 静态响应处理：在边缘位置直接建立部分响应，从而避免其转发到内部集群。
* 多区域弹性：跨越AWS Region进行请求路由，旨在实现ELB（Elastic Load Balancing）使用的多样化，以及让系统的边缘更贴近系统的使用者。

Spring Cloud对Zuul进行了整合与增强。目前，Zuul使用的默认HTTP客户端是Apache HTTP Client，也可以使用RestClient或者okhttp3.0kHttpClient。如果想要使用RestClient，可以设置ribbon.restclient.enabled=true；想要使用okhttp3.0kHttpClient，可以设置ribbon.okhttp.enabled=true。

下面让我们来看看如何实现：

重新创建一个Zuul的项目添加依赖：

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>1.4.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
            <version>1.4.0.RELEASE</version>
        </dependency>
    </dependencies>
```

在启动类上添加启动注解：

```java
@EnableZuulProxy
@SpringBootApplication
public class start {
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```

编写配置文件：

```java
server:
  port: 8762
spring:
  application:
    name: Zuul
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

可以看到我们注册了`http://localhost:8761/eureka`的服务器上，启动项目，和服务相关的组件启用。然后查看Eureka Server，注册成功如下：

![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a7.png)

然后就可以通过这个zuul组件可以访问其他组件的控制器,格式如`http://localhost:8762/组件名称/组件对应的url方法`比如：

`http://localhost:8762/server1/index`  ： 访问server1组件中的index方法。

`http://localhost:8762/buyserver/user/1` : 访问buyserver组件中的user/1路径

由于Zuul自带Ribbon组件效果，所以使用Ribbon可以达到负载均衡的效果。不仅如此，Zuul还具有Hystrix监控信息如下,打开Hystrix访问界面，输入
`http://localhost:8762/hystrix.stream`就可以看到如下各个服务的信息：

![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a8.png)

说明Zuul已经整合了HyStrix

****

<b id="a2"></b>

### :arrow_up_small: 管理端点

:arrow_up:[返回目录](#t)

当@EnableZuulProxy注解和Actuator一起使用时，Zuul会暴露链两个端点：/routes，/filters借助这两个端点我们可以更清楚，更方直观地查看管理Zuul。

**routes端点**

使用GET方法访问可以返回Zuul当前映射的路由表列表。使用POST方法访问端点会强制刷新当前列表路由表。还可以使用`/routes?format=details`查看更多详细信息。

由于zuul已经包含了actuator，所以这里不需要添加依赖，只需要修改配置文件即可：

```java
management:
  security:
    enabled: false
```

然后访问`http://localhost:8762/routes`可以看到如下信息：

```java
{"/server1/**":"server1","/buyserver/**":"buyserver","/zuul/**":"zuul","/userserver/**":"userserver"}
```

说明可以通过这些url路径来访问不同的信息。当然也可以访问`http://localhost:8762/routes?format=details`来查看更多详细信息。

**filters端点**

filters端点提供了当前所有过滤器的详情，并按照类型分类。

***

<b id="a3"></b>

### :arrow_up_small: 路由配置详解

:arrow_up:[返回目录](#t)

前面我们可以通过url访问不同的服务，但是现在我们想修改url路径格式，或者只想让部分服务生效，那么就需要配置路由。

**自定义微服务路径**

修改配置文件，添加如下内容

```
zuul:
  routes:
    UserServer: /us/**
    Server1: /ms/**
    BuyServer: /bs/**
```

上面就是将以服务组件名的url自定义为我们想要url索引，所以接下里就可以使用`http://localhost:8762/ms/index`方式来索引不同的方法。

**忽略指定微服务**

```
zuul:
  routes:
    UserServer: /us/**
    BuyServer: /bs/**
  ignored-services: Server1,UserServer
```

ignored-services可以忽略指定的服务，但是routes又会自定义url，当两个都存在时 routes可以忽略 ignored-services屏蔽的url，所以UserServer可以访问。

所以可以忽略全部只开启一个：

```
zuul:
  routes:
    BuyServer: /bs/**
  ignored-services: '*'     --*代表全部
```

还可以忽略更细微的路径比如某服务下的/index路径可以如下：

```java
zuul:
  ignored-services: /**/index/**
```

忽略所有/index/的路径。



























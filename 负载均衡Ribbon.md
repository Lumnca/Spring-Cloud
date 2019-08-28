# 	:triangular_flag_on_post:负载均衡Ribbon

<p id="t"></p>

:arrow_down:[为消费者整合Ribbon](#a1)

:arrow_down:[Ribbon配置自定义](#a2)

:arrow_down:[脱离Eureka使用Ribbon](#a3)


<b id="a1"></b>

### :arrow_up_small: 为消费者整合Ribbon

:arrow_up:[返回目录](#t)

Ribbon 是Nelix发布的负载均衡器，它有助于控制HTTP和TCP客户端的行为。为Ribbon 配置服务提供者地址列表后，Ribbon就可基于某种负载均衡算法，自动地帮助服务消费者去请求。Ribbon默认为我们提供了很多的负载均衡算法，例如轮询、随机等。当然，我们也可为Ribbon实现自定义的负载均衡算法。
在Spring cloud中，当Ribbon与Burela配合使用时，Ribon可自动从Eurela Servwer获取服务提供者地址列表，并基于负载均衡算法，请求其中一个服务提供者实例。

下面介绍如何整合Ribbon

首先将前面的UserServer再复制一份，并修改端口，让其两个注册在Eureka Server上。为了使其有负载均衡的功能，还需要添加依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
            <version>2.0.3.RELEASE</version>
        </dependency>
```

成功后如下：

![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a4.png)

可以看到UserServer注册了两个。接下来就是，消费者的编写（BuyServer为消费者，Server为Eureka Server）

为RestTemplate添加@LoadBalanced注解

```java
@SpringBootApplication
public class start {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```

@LoadBalanced注解使其可以整合Ribbon，具有负载均衡的能力。对控制器进行修改：

```java
@RestController
public class index {
    @Autowired
    RestTemplate restTemplate;
    @Autowired
    LoadBalancerClient loadBalancerClient;
    @GetMapping("/user/{id}")
    public User user(@PathVariable Integer id){
        return this.restTemplate.getForObject("http://UserServer/"+id,User.class);
    }
    @GetMapping("/getServer")
    public String getServer(){
        ServiceInstance serviceInstance = loadBalancerClient.choose("UserServer");
        return "主机IP地址:"+ serviceInstance.getHost() + "</br>" +
                "微服务ID:"+ serviceInstance.getServiceId() + "</br>" +
                "响应端口:"+serviceInstance.getPort()+ "</br>" +
                 "Url:"+serviceInstance.getUri();
    }
}

```

然后运行这几个项目，访问`http://localhost:100/getServer`多次刷新页面，可以看到端口的变化，可以自动每次访问的服务器都是不同的端口。

<b id="a2"></b>

### :arrow_up_small: Ribbon配置自定义

:arrow_up:[返回目录](#t)

在前面的负责均衡配置中，我们可以知道每次刷新分配的客户端是不同的这是采用了均摊的思想，如果想自定义分配机制，

在Spring Cloud中，Ribbon默认的配置类是RibbonClientConfiguration。也可以使用自定义类，如下：

在服务消费者中添加配置规则类：

```java
@Configuration
public class RibbonConfiguration {
    @Bean
    public IRule ribbonRule(){
        return new RandomRule();  //随机分配
    }
}
```

然后应用配置：

```java
@Configuration
@RibbonClient(name = "UserServer",configuration = RibbonConfiguration.class)
public class Ribbon {

}
```

运行刷新页面，就会发现这次的端口是随机的。当然也可以使用最佳分配法：

```java
@Configuration
public class RibbonConfiguration {
    @Bean
    public IRule ribbonRule(){
        return new BastAvailableRule();  
    }
}
```


使用java代码配置可能有些麻烦，可以使用属性自定义配置信息：

```
UserServer :
  ribbon :
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

* NfLoadBalancerclassiame：配置ILoadBalancer的实现类。

* NfLoadBalancerRuleCLassllame：配置Rule的实现类。

* NFLoadBalancerPingcLassllame：配置IPing的实现类。

* NIWSServerListClassName：配置 ServerList的实现类。

* NIWSServerListFilterClassName：配置ServerListFilter的实现类。

若配置成如下形式，则表示对所有Ribbon Client都使用RandonRule：

```
ribbon：
   NLoadBalancerRuleclasslNane:com.netflix.losatbalancer.Randonfule
```

<b id="a1"></b>

### :arrow_up_small: 脱离Eureka使用Ribbon

:arrow_up:[返回目录](#t)

在前文的示例中，是将Ritbon与Burka配合使用的。但现实中可能不具备这样的条件，荷如二些遗留的微服务，它们可能并没有注册到urcka Server上，甚至根本不是使用Spring Cloud开发的，此时要想使用Ribbon实现负载均衡，就需要做额外的设置：

这里为了演示，我们把服务消费者的客户端弃用

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>1.4.0.RELEASE</version>
        </dependency>
```

改为

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
```

如果启动类含有@EnableDiscoveryClient注解就去掉，然后修改配置文件：

```xml
server:
  port: 100
spring:
  application:
    name: BuyServer
UserServer :
  ribbon:
    listOfServers: http://localhost:8080/users,http://localhost:8081/users
```

UserServer是调用微服务的名称。listOfServers为你想要访问的url路径。像这样就可以完成，如果仅想Ribbon运用，不想使用Eureka的服务发现功能，可以添加如下配置：

```
ribbon.eureka.enable=false
```

**饥饿加载**

Spring cloud会为每个名称的RibbonClient维护一个子应用程序上下文（还记得Spimg Framework中的父子上下文吗？），这个上下文默认是懒加载的。指定名称的Ribbon Client 第一次请求时，对应的上下文才会被加载，因此，首次请求往往会比较慢。从Spring Cloud Dalston开始，我们可配置饥饿加载。例如：

```
ribbon:
  eager-load:
    enabled: true
    clients: client1,client1
```

这样，对于名为client1、clent2的Ribbon Client，将在启动时就加载对应的子应用程序上下文，从而提高首次请求的访问速度。






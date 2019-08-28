# 	:triangular_flag_on_post :负载均衡Ribbon

<p id="t"></p>

:arrow_down:[为消费者整合Ribbon](#a1)


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

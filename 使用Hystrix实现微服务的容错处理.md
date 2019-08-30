# 	:triangular_flag_on_post:	使用Hystrix实现微服务的容错处理

<p id="t"></p>

:arrow_down:[实现容错的手段](#a1)

:arrow_down:[使用Hystrix实现容错](#a2)

:arrow_down:[Feign使用Hystrix](#a3)

:arrow_down:[使用Hystrix Dashboard可视化监控数据](#a4)

<b id="a1"></b>

### :arrow_up_small: 实现容错的手段

:arrow_up:[返回目录](#t)

如果服务提供者响应非常缓慢，那么消费者对提供者的请求就会被强制等待，直到提供者响应或超时。在高负载场景下，如果不做任何处理，此类问题可能会导致服务消费者
的资源耗竭甚至整个系统的崩溃。例如，曾经发生过一个案例——某电子商务网站在一个黑色星期五发生过载。过多的并发请求，导致用户支付的请求延迟很久都没有响应，
在等待很长时间后最终失败。支付失败又导致用户重新刷新页面并再次尝试支付，进一步增加了服务器的负载，最终整个系统都崩溃了。当依赖的服务不可用时，服务自身会
不会被拖垮？这是我们要考虑的问题。

**雪崩效应**

微服务架构的应用系统通常包含多个服务层。微服务之间通过网络进行通信，从而支撑起整个应用系统，因此，微服务之间难免存在依赖关系。我们知道，任何微服务都并非
100%可用，网络往往也很脆弱，因此难免有些请求会失败。我们常把“基础服务故障”导致“级联故障”的现象称为雪崩效应。雪崩效应描述的是提供者不可用导致消费者不可用
，并将不可用逐渐放大的过程。如下图所示，A作为服务提供者（基础服务），B为A的服务消费者，C和D是B的服务消费者。当A不可用引起了B的不可用，并将不可用像滚
雪球一样放大到C和D时，雪崩效应就形成了。

![](https://ss1.bdstatic.com/70cFuXSh_Q1YnxGkpoWK1HF6hhy/it/u=1814771833,683912152&fm=26&gp=0.jpg)

**如何容错**

要想防止雪崩效应，必须有一个强大的容错机制，该容错机制需实现以下两点：


* 为网络请求设置超时：必须为网络请求设置超时。正常情况下，一个远程调用一南在几十毫秒内就能得到响应了。如果依赖的服务不可用或者网络有问题，那么响应时间
就会变得很长（几十秒）。通常情况下，一次远程调用对应着一个线程/进程。如果响应太慢，这个线程/进程激得不到释放。而线程/进程又对应着系统资源，如果得不到
释放的线程/进程越积越多，资源就会逐渐被耗尽，最终导致服务的不可用。因此，必须为每个网络请求设置超时，让资源尽快释放。

* 使用断路器模式：试想一下，如果家里没有断路器，当电流过载时（例如功率过大短路等），电路不断开，电路就会升温，甚至可能烧断电路、引发火灾。使用断路器，
电路一旦过载就会跳闸，从而可以保护电路的安全。在电路超载的问题被解决后，只需关闭断路器，电路就可以恢复正常。同理，如果对某个微服务的请求有大量超时
（常常说明该微服务不可用），再去让新的请求访问该服务已经没有任何意义，只会无谓消耗资源。例如，设置了超时时间为1s，如果短时间内有大量的请求无法在1s内
得到响应，就没有必要再去请求依赖的服务了。断路器可理解为对容易导致错误的操作的代理。这种代理能够统计一段时间内调用失败的次数，并决定是正常请求依赖的
服务还是直接返回。断路器可以实现快速失败，如果它在一段时间内检测到许多类似的错误（例如超时），就会在之后的一段时间内，强迫对该服务的调用快速失败，即
不再请求所依赖的服务。这样，应用程序就无须再浪费CPU时间去等待长时间的超时。断路器也可自动诊断依赖的服务是否已经恢复正常。如果发现依赖的服务已经恢复
正常，那么就会恢复请求该服务。使用这种方式，就可以实现微服务的“自我修复”—一当依赖的服务不正常时，打开断路器时快速失败，从而防止雪崩效应；当发现依赖
的服务恢复正常时，又会恢复请求。断路器状态转换的逻辑如图7-2所示，简单讲解一下。-正常情况下，断路器关闭，可正常请求依赖的服务。-当一段时间内，请求失
败率达到一定阀值（例如错误率达到50%，或100次/分钟等），断路器就会打开。此时，不会再去请求依赖的服务。断路器打开一段时间后，会自动进入“半开”状态。此时
，断路器可允许一个请求访问依赖的服务。如果该请求能够调用成功，则关闭断路器；否则继续保持打开状态。

***

<b id="a2"></b>

### :arrow_up_small: 使用Hystrix实现容错

:arrow_up:[返回目录](#t)

Hystix是由Netlix开源的一个延迟和容错库，用于隔离访问远程系统、服务或者第三方库，防止级联失败，从而提升系统的可用性与容错性。Hysix主要通过以下几点实现
延迟和容错。

为服务消费者项目添加依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
            <version>2.0.3.RELEASE</version>
        </dependency>
```

在启动类上添加@EnableHystrix注解或者@EnableCircuitBreaker从而为项目启用断路器支持。

修改控制器使其具备容错能力。

```java
@RestController
public class index {
    @Autowired
    UserFeignClient userFeignClient;
    //容错指向方法
    @HystrixCommand(fallbackMethod = "defaultUser")
    @GetMapping("/user/{id}")
    public User user(@PathVariable("id") Integer id){
        return  this.userFeignClient.findById(id);
    }
    //容错处理方法
    public User defaultUser(Integer id){
        User user = new User();
        user.setId(-1);
        user.setName("默认用户");
        user.setUsername("默认角色");
        return user;
    }
}
```

这样就可以了，启动所有项目，运行成功返回数据正常！然后关闭服务器提供者就会显示默认数据。上面只是在请求失败的情况下才会显示默认消息，如果想设置超时和队
列可以添加额外配置：

```java
    @HystrixCommand(fallbackMethod = "defaultUser",commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
    },
            threadPoolProperties = {
                    @HystrixProperty(name = "coreSize", value = "30"),
                    @HystrixProperty(name = "maxQueueSize", value = "101"),
                    @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
                    @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),
                    @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "12"),
                    @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "1440")
    })
```

其中coresize是核心线程数，即可以同时接受的请求数，maximumSize最大线程数，即超过核心线程数之后，多余的线程会处于空闲状态，等核心线程使用它。开启
maximumSize需要设置allowMaximumSizeToDivergeFromCoreSize为true。

超过了最大线程数之后的请求会被hystrix降级处理，即调用hystrxi中的fallback属性指定的降级方法。如果没有指定降级方法，则会默认返回null。

maxQueueSize（最大排队长度。默认-1，使用SynchronousQueue。其他值则使用 LinkedBlockingQueue。如果要从-1换成其他值则需重启，即该值不能动态调整。

queueSizeRejectionThreshold（排队线程数量阈值，默认为5，达到时拒绝，如果配置了该选项，队列的大小是该队列）

keepAliveTimeMinutes 此属性设置保持活动时间，以分钟为单位。

具体信息可以参考[Hystrix配置信息](https://github.com/Netflix/Hystrix/wiki/Configuration)

如果你想获取异常原因，只需要在回退的方法上加一个Throwable参数即可，如下：

```java
    public User defaultUser(Integer id,Throwable throwable){
        System.out.println(throwable.getMessage());
        User user = new User();
        user.setId(-1);
        user.setName("默认用户");
        user.setUsername("默认角色");
        return user;
    }
```

如果你并不想触发回退方法，可以添加属性ignoreExceptions，如下：

```java
    @HystrixCommand(fallbackMethod = "defaultUser",ignoreExceptions = Exception.class)
    @GetMapping("/user/{id}")
    public User user(@PathVariable("id") Integer id){
        return  this.userFeignClient.findById(id);
    }

    public User defaultUser(Integer id,Throwable throwable){
        System.out.println(throwable.getMessage());
        User user = new User();
        user.setId(-1);
        user.setName("默认用户");
        user.setUsername("默认角色");
        return user;
    }
 ```
 
 像这样就不会调用回退方法。
 
 **断路器的状态与监控**
 
 之前我们说过应用监控的Actuator，断路器的状态也会暴露在Actuator提供的/health端点中，这样就可以直接地了解断路器的状态，添加Actuator依赖就会在health端口中显示，。
 
 我们发现，尽管执行了回退逻辑，返回了默认用户，但此时日时srx的状态依然是UP，这是因为我们的失败率还没有达到阀值（默认是5s内20次失败。这是很多初学者会遇到的误区，这里再次强调—执行回退逻辑并不代表断路器已经打开。请求失败、超时、被拒绝以及断路器打开时等都会执行回退逻辑。
 
 所以要持续快速地访问网址直到请求快速返回，才会显示断路器打开的结果。除此之外Hystrix还提供了监控，在添加上面的依赖之后，访问`http://localhost:100/hystrix.stream`可以看到容错信息。
 
 ***
 
<b id="a3"></b>

### :arrow_up_small: Feign使用Hystrix

:arrow_up:[返回目录](#t)

前面是通过实现了注解属性来实现回退的，Feign是以借口形式的，它没有方法实体，所以前面的不适合用于Feign，要实现Feign整合Hystrix，只需要开启这个设置就行了，因为Feign已经整合了Hystrix的只需要设置**feign.hystrix.enabled=true**即可。然后在接口处声明回退类：

```java
@FeignClient(name = "UserServer",fallback = UserFeignClientFallBack.class)
public interface UserFeignClient {
    @RequestMapping(value = "/users/{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);
}
```

构建回退类：

```java
@Component
public class UserFeignClientFallBack implements  UserFeignClient{

    @Override
    public User findById(Integer id) {
        return new User();
    }
}

```
 
 启动测试得到的是一样的效果。还可以异常检测，只不过这时使用的是fallbackFactory属性如下：
 
 ```java
 @FeignClient(name = "UserServer",fallbackFactory = UserFeignClientFallBack.class)
public interface UserFeignClient {
    @RequestMapping(value = "/users/{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);
    @RequestMapping(value = "/test",method = RequestMethod.GET)
    String test();
}
 ```
 
 对应的处理类：
 
 ```java
 @Component
public class UserFeignClientFallBack implements FallbackFactory<UserFeignClient> {
    @Override
    public UserFeignClient create(final Throwable throwable) {
        return new UserFeignClient() {
            //应该把异常参数在各个方法中使用
            @Override
            public User findById(Integer id) {
                System.out.println("异常信息:"+throwable.getMessage());
                return new User();
            }

            @Override
            public String test() {
                System.out.println("异常信息:"+throwable.getMessage());
                return "TEST";
            }
        };
    }
}
 ```
 

这样就可以完成信息异常报出了，这里提供了一个正常的test的方法，该方法在没有报错的情况是不显示异常处理信息的，因为该方法并没有回退。

如果并不想Feign使用Hystrix，可以禁用Hystrix。为指定的用户程序禁用需要添加配置类，并引用：

```java
//为指定Feign客户端禁用Hystix：借助Feign的自定义配置，可轻松为指定名称的Feign客户端禁用Hystrix。例如：
@Configuration 
public class Feign0isableystrixconfiguration{
   @Bean
   @Scope（"prototype"）
   public Feign.Builder feignBuilder（）{
        return Feign.builder（）；
   }      
 }   
 ```
想要禁用Hystrix的@FeignClient，引用该配置类即可，例如：

```java
@FeignClient（name="user"，configuration=FeignDisableHystrixconfiguration.class）
        public interface UserFeignClient{
              //...
        }
 ```       
全局禁用Hystrix：也可为Feign全局禁用Hystix。只需在application.yml中配置feignhystrix.enabled=false即可。

<b id="a4"></b>

### :arrow_up_small: 使用Hystrix Dashboard可视化监控数据

:arrow_up:[返回目录](#t)

前面说的应用监控都是端口化配置，对于Hystrix，他也有图形化界面操控，如下

引入依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
```

编写启动类：

```java
@SpringBootApplication
@EnableHystrixDashboard
public class start {
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```
 
 然后直接通过`http://localhost:100/hystrix`就可以访问，只不过需要在url一栏输入你的Hystrix测试端口信息，输入后看到如下信息
 
 ![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a5.png)
 
 在指标信息中Host，Cluster是访问与请求的延迟的时间，访问量越大，时间越高。

这是单一一个实例查询，这样做有点慢，需要一个一个的查，可以使用Turbine来实现集群管理。如下添加依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
        </dependency>
```

在开始类启动注解：

```java
@EnableTurbine
public class start {
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```

然后编写文件：

```java
server:
  port: 100
spring:
  application:
    name: BuyServer
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    prefer-ip-address: true
feign:     #开启服务
  hystrix:
    enabled: true
turbine:
  cluster-name-expression: "'default'"
  app-config: UserServer,BuyServer  #--引用组件上的服务
```
 
 
 然后在`http://localhost:100/hystrix`输入`http://localhost:100/turbine.stream`就可以看到集群信息。
 
 ![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a6.png)
 
 当然要想其他项目具有HyStrix功能，需要加入依赖：
 
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
            <version>2.0.3.RELEASE</version>
        </dependency>
```

并在启动类中开启@EnableCircuitBreaker注解，这样就可以使用/hystrix.stream端口监控。

这里是由于方便我把所有的服务都写在一个项目中，可以根据自己需要重新写一个项目并加入。
 
 
 
 
 
 
 
 
 
 
 
 

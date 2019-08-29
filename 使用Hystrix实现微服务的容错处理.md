# 	:triangular_flag_on_post:	使用Hystrix实现微服务的容错处理

<p id="t"></p>

:arrow_down:[实现容错的手段](#a1)

:arrow_down:[使用Hystrix实现容错](#a2)

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


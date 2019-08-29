# 	:triangular_flag_on_post:	Feign实现声明式REST调用

<p id="t"></p>

:arrow_down:[Feign简介与整合](#a1)

:arrow_down:[自定义Feign配置](#a2)

<b id="a1"></b>

### :arrow_up_small: Feign简介与整合

:arrow_up:[返回目录](#t)

在前面的调用中，我们可以知道RestTemplate实现了REST API调用，如下代码:

```java
    @GetMapping("/user/{id}")
    public User user(@PathVariable Integer id){
        return this.restTemplate.getForObject("http://127.0.0.1/users/"+id,User.class);
    }
```


可以看到我们是使用字符串的方式构造URL的，该URL参只有一个，但是在显示中参数有很多，所以这样开发效率低，并且难以维护。如果想解决这个问题，可以使用Feign。

Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。Feign可帮助我们更加便捷、优雅地调用HTTPAPI。
在Spring Cloud中，使用Feign非常简单——创建一个接口，并在接口上添加一些注解，代码就完成了。Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。
Spring Cloud对Feign进行了增强，使Feign支持了Spring MVC注解，并整合了Ribbon和Eureka，从而让Feign的使用更加方便。

前面使用的是RestTemplate调用REST fulAPI的，这里我们使用Feign，实现声明式的RESTFul API调用。

添加依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

创建Feign接口：

```java
@FeignClient(name = "UserServer",url = "http://localhost:8080/users")
//这里提供了url参数是由于客户端没注册到Eureka Server上，如果注册到了上面，可以不写这个参数，直接提供name 属性即可。
public interface UserFeignClient {
    @RequestMapping(value = "{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);
}
```

创建控制器：

```java
@RestController
public class index {
    @Autowired
    UserFeignClient userFeignClient;
    @GetMapping("/user/{id}")
    public User user(@PathVariable Integer id){
        return this.userFeignClient.findById(id);
    }
}
```

修改启动类,开启@EnableFeignClients注解：

```java
@SpringBootApplication
@EnableFeignClients
public class start {
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```

运行启动后访问成功即可。

***

<b id="a2"></b>

### :arrow_up_small: 自定义Feign配置

:arrow_up:[返回目录](#t)

很多场景下需要自定义Fegin配置，可以使用java代码配置或者属性配置。

首先看下java代码配置：

在Spring Cloud中，Feign的默认配置类是FeignCLientsconfiguration，该类定义了Feign默认使用的编码器、解码器、所使用的契约等。
Spring Cloud 允许通过注解QFeignClient的configuration属性自定义Feign的配置，自定义配置的优先级比FeignClientsConfiguration要高。
在Spring Cloud文档中可看到以下段落，描述了Spring Cloud提供的默认配置。另外，有的配置尽管没有提供默认值，但是Spring也会扫描其中列出的类型（也就是说，这部分配置也能自定义）。


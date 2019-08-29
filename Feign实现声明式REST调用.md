# 	:triangular_flag_on_post:	Feign实现声明式REST调用

<p id="t"></p>

:arrow_down:[Feign简介与整合](#a1)

:arrow_down:[自定义Feign配置](#a2)

:arrow_down:[使用Feign构造多参数的请求](#a3)

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

添加一个配置类：

```java
public class FeignConfiguration {
    @Bean
    public Contract feignContract(){
        return new feign.Contract.Default();
    }
    /*
    自定义的一个拦截器
    */
    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor(){
        return new BasicAuthRequestInterceptor("client","123");
    }
}
```

然后在接口处引用配置类：

```java
@FeignClient(name = "UserServer",configuration = FeignConfiguration.class)
public interface UserFeignClient {
    //这里使用feign自带的路径注解，方法类型+url路径
    @RequestLine("GET users/{id}")
     User findById(@Param("id") Integer id);
}
```

在Spring Cloud Edgware中，Feign的配置类（例如本例中的FeignConfiguration类）无须添加@Configuration注解；如果加了@Configuration注解，那么该类不能存放在主应用程序上下文@ComponentScan所扫描的包中。否则，该类中的配置的feign.Decoder、feign.Encoder、feign.Contract等配置就会被所有的eFeignClient共享。为避免造成问题，**最佳实践**是不在指定名称的Feign配置类上添加@Configuration注解。

当然也可以使用属性来配置,直接在配置文件下添加信息：

```java
feign：
    client:
        config：
            #实例名称
            feignName(UserServer)：
                #相当于Request.Options
                connectTimeout：5000
                #相当于Request.Options 
                readTimeout：5000
                #配置Feign的日志级别，相当于代码配置方式中的Logger 
                loggerLevel:full
                #Feign的错误解码器，相当于代码配置方式中的ErrorDecoder 
                errorDecoder:com.example.SimpleErrorDecoder
                #配置重试，相当于代码配置方式中的Retryer 
                retryer:com.example.SimpleRetryer
                #配置拦截器，相当于代码配置方式中的RequestInterceptor 
                requestInterceptors：
                    -com.example.FooRequestInterceptor
                    -com.example.BarRequestInterceptor 
                decode404：false
```

上面是对于单个配置的，对于配置所以的feign，只需要把名称改为default即可：

```yml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full

```
***

<b id="a3"></b>

### :arrow_up_small: 使用Feign构造多参数的请求

:arrow_up:[返回目录](#t)

前面我们的示例都是只有一个参数，如果参数含有多个，那我们就需要构造多参数请求，如下：

**GET 请求多个参数**

```java
@FeignClient(name = "UserServer")
public interface UserFeignClient {
    @RequestMapping(value = "/users/{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);

    @RequestMapping(value = "/get",method = RequestMethod.GET)
     String get(@RequestParam("id")Integer id,@RequestParam("name")String name);
}
```

控制器：

```java
@RestController
public class index {
    @Autowired
    UserFeignClient userFeignClient;
    @GetMapping("/user/{id}")
    public User user(@PathVariable("id") Integer id){
        return  this.userFeignClient.findById(id);
    }
    @GetMapping("/get")
    public String get(@RequestParam("id")Integer id,@RequestParam("name") String name){
        return userFeignClient.get(id,name);
    }
}
```

这里添加了一个get方法可以接受两个参数，然后就可以在服务提供者哪里调用下的该方法，所以在服务提供者哪里需要提供get接口：

```java
@RestController
public class index {
    @GetMapping("/get")
    public String get(@RequestParam("id")Integer id, @RequestParam("name")String name){
        return "ID:"+id+"<br>"+"Name:"+name;
    }
}
```

也可以使用Map来构建：

```java
@FeignClient(name = "UserServer")
public interface UserFeignClient {
    @RequestMapping(value = "/users/{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);

    @RequestMapping(value = "/get",method = RequestMethod.GET)
     String get(@RequestParam Map<String,Object> map);
}
```

调用时：

```java
    @GetMapping("/get")
    public String get(@RequestParam("id")Integer id,@RequestParam("name") String name){
        Map<String,Object> map = Maps.newHashMap();
        map.put("id",id);
        map.put("name",name);
        return userFeignClient.get(map);
    }
```

**POST请求包含多个参数**

与GET样式一样只不过需要转换方法类型与参数形式，如果是类参数，像如下即可：

```java
@FeignClient(name = "UserServer")
public interface UserFeignClient {
    @RequestMapping(value = "/users/{id}",method = RequestMethod.GET)
     User findById(@PathVariable("id") Integer id);

    @RequestMapping(value = "/post",method = RequestMethod.POST)
    User infoUser(@RequestBody User user);
}
```


然后控制器也是一样：

```java
@RestController
public class index {
    @Autowired
    UserFeignClient userFeignClient;
    @GetMapping("/user/{id}")
    public User user(@PathVariable("id") Integer id){
        return  this.userFeignClient.findById(id);
    }

    @PostMapping("/post")
    public User post(@RequestBody User user){
       return userFeignClient.infoUser(user);
    }
}
```

最后在服务提供者哪里提供该方法引用：

```java
@RestController
public class index {
    @PostMapping("/post")
    public User post(@RequestBody User user){
        return user;
    }
}
```


























# 	:triangular_flag_on_post:	编写Zuul微服务网关

<p id="t"></p>

:arrow_down:[编写Zuul微服务网关](#a1)

:arrow_down:[管理端点](#a2)

:arrow_down:[路由配置详解](#a3)

:arrow_down:[Zuul安全与Header](#a4)

:arrow_down:[Zuul上传文件](#a5)

:arrow_down:[Zuul的过滤器](#a6)

:arrow_down:[编写过滤器](#a7)

:arrow_down:[Zuul的容错与回退](#a8)

:arrow_down:[Zuul的高可用与隔离策略线程池](#a9)


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

<b id="a4"></b>

### :arrow_up_small: Zuul安全与Header

:arrow_up:[返回目录](#t)

一般来说，可以在同一个系统中的服务之间共享Header。不过应尽量防止让一些敏感的Header外泄，因此在许多场景下，需要通过为路径指定一些列敏感Header列表，如：

```java
zuul:
  sensitive-headers: Cookie,Set-Cookie,Authorization
```

如上指定全局的敏感列表，如果是单一项目，在服务名称下再写属性即可：

```yml
zuul:
  routes:
    BuyServer:
     sensitive-headers: Cookie,Set-Cookie,Authorization
```


**忽略Header**

可以使用zuul.ignored-headers来忽略一些属性：

```java
zuul:
  ignored-headers: Header1,Header2
```

这样设置后，Header1，Header2就不会传播到其他服务中。默认情况下，zulignoredheaders是空值，但如果Spring Securiy在项目的cdaspath中，那么zxulignored-headers的默认值就是Pragma，cache-Control，X-frane-ptions，X-Content-Type-0ptions，X-xS-Protection，Expires。所以，当Spring Security在项目的claspath中，同时又需要使用下游微服务的Sing securty 的Hesder时，可以将zuul.igoreseurity-Headers 设置为false。

<b id="a5"></b>

### :arrow_up_small: Zuul上传文件

:arrow_up:[返回目录](#t)

对于小于1M以内文件上传，无须任何处理，即可正常上传，对于大于10M以上的文件上传需要为上传路径添加zuul路径前缀，也可以使用zuul.servlet-path自定义路径。

也就是说，假设`zuul.routes.microservice-file-upload =/microservice-file-upload/**`如果`http://{H0ST]：[PORT}/upload` 是微服务microservice-file-upload的上传路径，则可使用Zuul的`/zuul/microservice-file-upload/upload`路径上传大文件。
如果Zuul使用了Ribbon做负载均衡，那么对于超大的文件（例如500M），需要提升时设置，例如：

#上传大文件时，要将超时时间设长一些，否则会报超时异常。

```java
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds：60000
    ribbon：ConnectTimeout：3000
        ReadTimeout：60000
```

下面编写一个文件上传微服务，在前面的zuul工程中添加web依赖：

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

编写控制器：

```java
@RestController
public class index {
    @RequestMapping(value = "/fileUpload",method = RequestMethod.POST)
    public @ResponseBody
    String handleFileUpload(@RequestParam(value = "file",required = true)MultipartFile file)throws IOException {
        byte[] bytes = file.getBytes();
        File fileToSave = new File( ClassUtils.getDefaultClassLoader().getResource("")
                .getPath()+"static/"+ file.getOriginalFilename());
        FileCopyUtils.copy(bytes,fileToSave);
        return fileToSave.getAbsolutePath();
    }
}
```

 ClassUtils.getDefaultClassLoader().getResource("").getPath()为获取class绝对路径。
 
 然后修改配置文件：
 
 ```java
 server:
  port: 8762
spring:
  application:
    name: Zuul
  http:
    multipart:
      max-file-size: 200Mb
      max-request-size: 2500Mb
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
management:
  security:
    enabled: false
zuul:
  routes:
    BuyServer: /bs/**
 ```

添加一个上传界面：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form action="/fileUpload" method="post" enctype="multipart/form-data">
        <input type="file" name="file" ><br>
        <input type="submit" value="提交">
    </form>
</body>
</html>
```

上传成功表明配置成功！

<b id="a6"></b>

### :arrow_up_small: Zuul的过滤器

:arrow_up:[返回目录](#t)

Zuul大部分功能都是通过过滤器来实现的。zuul中定义了4种标准过滤器类型，这些过滤器类型对应请求的典型生命周期。

* PRE：这种过滤器在请求被路由之前调用。可利用这种过滤器实现身份验证，在集群中选择请求的微服务，记录调试信息等。

* ROUTING： 这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient 等请求微服务。

* POST： 这种过滤器在路由到微服务以后执行，这种过滤器可以用来为响应添加标准的Http Header，收集统计信息和指标，将响应从微服务发送给微服务。

* ERROR： 在其他阶段发送错误时执行该过滤器。

除了默认的过滤器类型，Zuul还允许创建自定义的过滤器类型。例如，可以定制一种STATIC类型的过滤器，直接在Zuul中生成响应，而不将请求转发到后端的微服务。

**内置过滤器详解**

Spring Cloud默认为Zuul编写并启用了一些过滤器，这些过滤器有什么作用呢？我们不妨结合@EnableZuulServer，@EnableZuulProxy两个注解进行讨论。

可将@EnableZuulProxy 简单理解为@EnableZulserver的增强版。事实上，当zm1与Eureka、Ribbon 等组件配合使用时，@EnableZuul.Proxy是我们最常用的注解，本书前文所使用的也是@EnableZuulServer。

我们先来了解一下什么是RequestContext，其用于在过滤器之间传递消息，它的数据保存在每个请求的ThreadLocal中。它用于存储请求路由到哪里，错误，HTTPServletRequest，HTTPServletResponse等信息，RequestContext扩展了。所以任何数据都可以存储在RequestContext中。

**@EnableZuulServer 所启用的过滤器**

>pre类型过滤器

* 1.ServletDetectionFilter：该过滤器用于检查请求是否通过Spring Dispatcher。检查后，通过FilterConstants.IS_DISPATCHER_SERVLET_REQUEST_KEY设置布尔值。
* 2.FormBodyWrapperFilter：解析表单数据，并为请求重新编码。
* 3.DebugFilter：顾名思义，调试用的过滤器。当设置zuul.include-debug-header=true或设置zuul.debug.request=true，并在请求时加上了debug=true的参数，例如`$ZUUL_HOST：ZUUL_PORT/some-path？debug=true`就会开启该过滤器。该过滤器会把 RequestContext.setDebugRouting（）以及RequestContext.setDebugRequest（）设为true。

>route类型过滤器

SendForwardFilter：该过滤器使用Servlet RequestDispatcher转发请求，转发位置存储在Requestcontext的属性`FilterConstants.FORWARD_TO._KEY`中。这对转发到Zuul自身的端点很有用。可将路由设成：

```
zuu1：
  routes：
    abc：
      path:/path-a/**
      ur1：forward:/path-b
```

然后访问`ZUUL_H0ST:ZUUL_PORT/path-a/**`观察该过滤器的执行过程。

>post类型过滤器

SendRlesponseFilter：将代理请求的响应写入当前响应。

>error 类型过滤器

SendErrorFilter：若RequestContext.getThrowable（）不为null，则默认转发到/error，也可以设置error.path属性来修改默认的转发路径。

**@EnableZuulProxy所启用的过滤器**

如果使用注解@nableZulProxy，那么除上述过滤器之外，Spring Cloud 还会安装和过滤器。

>pre类型过滤器

Prabecorationfiter：该过滤器根据提供的outelocator确定路由到的地址，以及怎样去路由。同时，该过滤器还为下游请求设置各种代理相关的header。

>route 类型过滤器

1.RibbonRoutingFilter：该过滤器使用Ribbon、Hystrix和可插拔的HTTP客户端发送请求。serviceld 在RequestContext的属性FilterConstants.SERVICE_ID.KEY中。该过滤器可使用如下这些不同的HTTP客户端。

* Apache HttpClient：默认的HTTP客户端。
* Squareup OkHttpClient v3：若需使用该客户端，需保证com.squareup.okhttp3的依赖在classpath中，并设置ribbon.okhttp.enabled=true。

* Netflix Ribbon HTTP Client：设置ribbon.restclient.enabled=true即可启用该HTTP客户端。该客户端有一定限制，例如不支持PATCH方法，另外，它有内置的重试机制。

2.SimpleHostRoutingFilter：该过滤器通过Apache HttpClient向指定的URL发送请求。在RequestContext.getRouteHost（）中。


建议阅读以上过滤器的源码，将对Zuul有一个更加深入的认识。其中，最重要的过滤器当属RibbonRoutingFilter了，因为整合Ribbon、Hystrix以及发送请求都在该过滤器中完成。

形如如下内容的路由不会经过RibbonRoutingFilter，而是走Simplelost-RoutingFilter。

```
zuu1：
  routes：
    user-route：
      ur1：http://localhost：8000/
      path:/user/**
```
可借助/filters端点查看过滤器详情。目前FormBodyWrapperFilter的代码实现并不高效，若你的应用没有Form表单提交，可禁用该过滤器，从而获取更好的性能表现。

<b id="a7"></b>

### :arrow_up_small: 编写过滤器

:arrow_up:[返回目录](#t)

下面介绍一个Zuul过滤器，编写过滤器比较简单，只需要继承ZuulFilter，然后实现几个抽象方法就可以了：

```java
package com;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;

import javax.servlet.http.HttpServletRequest;


public class PreRequestLogFilter extends ZuulFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(PreRequestLogFilter.class);
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER-1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        //获取Request
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        
        PreRequestLogFilter.LOGGER.info("请求方法："+request.getMethod()+"请求路径："+request.getRequestURI()+"请求IP："+getIpAdrress(request));
        return  null;
    }
    //获取IP地址方法
    private static String getIpAdrress(HttpServletRequest request) {
        String Xip = request.getHeader("X-Real-IP");
        String XFor = request.getHeader("X-Forwarded-For");
        if(StringUtils.isNotEmpty(XFor) && !"unKnown".equalsIgnoreCase(XFor)){
            //多次反向代理后会有多个ip值，第一个ip才是真实ip
            int index = XFor.indexOf(",");
            if(index != -1){
                return XFor.substring(0,index);
            }else{
                return XFor;
            }
        }
        XFor = Xip;
        if(StringUtils.isNotEmpty(XFor) && !"unKnown".equalsIgnoreCase(XFor)){
            return XFor;
        }
        if (StringUtils.isBlank(XFor) || "unknown".equalsIgnoreCase(XFor)) {
            XFor = request.getHeader("Proxy-Client-IP");
        }
        if (StringUtils.isBlank(XFor) || "unknown".equalsIgnoreCase(XFor)) {
            XFor = request.getHeader("WL-Proxy-Client-IP");
        }
        if (StringUtils.isBlank(XFor) || "unknown".equalsIgnoreCase(XFor)) {
            XFor = request.getHeader("HTTP_CLIENT_IP");
        }
        if (StringUtils.isBlank(XFor) || "unknown".equalsIgnoreCase(XFor)) {
            XFor = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (StringUtils.isBlank(XFor) || "unknown".equalsIgnoreCase(XFor)) {
            XFor = request.getRemoteAddr();
        }
        return XFor;
    }
}
```

然后在启动类上添加Bean：

```java
@EnableZuulProxy
@SpringBootApplication
@EnableEurekaClient
public class start {
    @Bean
    public PreRequestLogFilter preRequestLogFilter(){
        return new PreRequestLogFilter();
    }
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```

运行测试`http://localhost:8762/zuul/index`

可以看到：

```
请求方法：GET请求路径：/zuul/index请求IP：0:0:0:0:0:0:0:1
```

由于是本地部署，所以IP地址是那样。下面介绍该类的4和重写方法：

由代码可知，自定义的ZulFiler需实现以下几个方法。

* fiteriye：返回过滤器的类型。有pre、对应上文的几种过滤器。译细可D类、rote、post、eror等几种取值，分别中的注释。可以参考comnetflix.zuu1.ZuutFilter.fiterType（）

*  filterOrder：返回一个int值来指定过法器的执行顺序，不同的过法器允许运行相同的数字。

* shouldFilter：返回一个bolean值来判断该过滤器是否要执行，true表示执行，false表示不执行。

* run：过滤器的具体逻辑。本例中让它打印了请求的HTTP方法以及请求的地址。

Pre类型的过滤器都是在请求之间运行的，我们还可以写一个post类型的过滤器在请求之后运行：

```java
public class PostRequestFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }

    @Override
    public int filterOrder() {
        return 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();

        HttpServletRequest request = ctx.getRequest();
        Enumeration<String> httpHeadrInfo =  request.getHeaderNames();
        System.out.println("Http头部信息");
        while (httpHeadrInfo.hasMoreElements()){
            String info = httpHeadrInfo.nextElement();
            System.out.println(info+":"+request.getHeader(info));
        }
        return null;
    }
}
```

自行添加在启动类上运行的Bean，这时点击运行可以看到：

```
请求方法：GET
请求路径：/zuul/index
请求IP：0:0:0:0:0:0:0:1

...

Http头部信息
host:localhost:8762
connection:keep-alive
cache-control:max-age=0
upgrade-insecure-requests:1
user-agent:Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36
accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
accept-encoding:gzip, deflate, br
accept-language:zh-CN,zh;q=0.9,en;q=0.8
cookie:.AspNet.Consent=yes
```

说明post后执行。pre先执行。可以利用过滤器完成很多的处理比如安全认证，灰度发布，限流等。这里就不一一介绍了。可以参考网上的资料。当然禁用过滤器除了上面的代码方法外，还可以通过zuul.<SimpClassName>.<FilterType>.disable=true禁用SimpClassName所对应的过滤器类。

***

<b id="a8"></b>

### :arrow_up_small: Zuul的容错与回退

:arrow_up:[返回目录](#t)

前面知道了zuul已经整合了hystrix所以我们可以在解监控界面上查看信息，但是如果其中一个服务被关闭了或者不能启用，那么怎么执行回退呢？要想为zuul添加回退功能，需要实现 ZuulFallbackProvider 接口,并提供一个 ClientHttpResponse作为回响。

```java
@Component
public class MyFallbackProvider implements FallbackProvider {
    @Override
    public ClientHttpResponse fallbackResponse(Throwable cause) {
        if(cause instanceof HystrixTimeoutException){
            return response(HttpStatus.INTERNAL_SERVER_ERROR);
        }
        else {
            return this.fallbackResponse();
        }
    }

    @Override
    public String getRoute() {
        //*代表所有路径服务
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return null;
    }
    private ClientHttpResponse response(final HttpStatus status){
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return status.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {

            }

            @Override
            //内容信息反馈
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("服务不可用请稍后再试！".getBytes());
            }

            @Override
            //header设定
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                MediaType mt = new MediaType("application","json", Charset.forName("UTF-8"));
                headers.setContentType(mt);
                return  headers;
            }
        };
    }
}
```

像这样就完成了zuul回退处理，关闭一些服务，再运行这些服务，就会看到回退信息。

Zuul整合了Ribbon实现了负载均衡，而Ribbon默认是懒加载的，可能会导致首次请求较慢的问题。可以使用配置饥饿加载。

```java
zuul:
  ribbon:
    eager-load:
      enabled: true
```


Query String 编码配置：

```java
zuul:
  force-original-query-string-encoding: true
```

<b id="a9"></b>

### :arrow_up_small: Zuul的高可用与隔离策略线程池

:arrow_up:[返回目录](#t)

在之前打开的Hystrix监控界面我们看到的内容如下：

![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a10.png)

这是由于默认情况下Zuul的Hystrix隔离策略是SEMAPHORE,可以使用：

```
zuul:
  ribbon-isolation-strategy: thread
```

配置可以将策略改为 thread，这样就可以看到Pool一栏的数据了。

![](https://github.com/Lumnca/Spring-Cloud/blob/master/img/a9.png)





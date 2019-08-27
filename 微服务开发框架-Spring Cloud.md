# 	:triangular_flag_on_post:	微服务开发框架---Spring Cloud

对于微服务概念这里不做多的解释，直接介绍主要知识内容。

<p id="t"></p>

:arrow_down:[服务提供者与消费者](#a1)

:arrow_down:[为项目整合Spring Boot Actuator](#a2)

:arrow_down:[硬编码问题](#a3)

<b id="a1"></b>

### :arrow_up_small: 服务提供者与消费者

:arrow_up:[返回目录](#t)

使用微服务构建的是分布式系统，微服务之间通过网络进行通信，我们使用服务提供者与服务消费者来描述微服务之间的调用关系。

|名称|定义|
|:--|:---|
|服务提供者|服务的被调用方（为其他服务提供服务的服务）|
|服务消费者|服务的调用方（依赖其他服务的服务）|

如下我们编写一个电影微服务，如下架构模式：

消费者----->电影微服务------>用户微服务

首先编写服务提供者（使用Spring Boot）：

添加依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>simple-user</groupId>
    <artifactId>simple-user</artifactId>
    <version>1.0-SNAPSHOT</version>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-rest</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.39</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.9</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.SR3</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

值得注意的是Spring Cloud要与Spring Boot版本对应才行，对于Spring Boot在2.0.x以上的版本来说需要使用的是`<version>Finchley.SR3</version>`对于1.5.x的是`<version>Edgware.RELEASE</version>`.

基础的使用Spring Boot，这里使用了jpa整合mysql。还添加了cloud的依赖和maven插件。

添加数据配置信息：

```java
server.port=8080

spring.datasource.url=jdbc:mysql://47.106.254.86/ex?characterEncoding=utf8&useSSL=true
spring.datasource.username=lumnca
spring.datasource.password=xxx
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.database=mysql
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL57Dialect
spring.jpa.show-sql=true
```


创建实体类：

```java
@Entity(name = "user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Integer id;
    @Column
    private String username;
    @Column
    private String name;
    @Column
    private Integer age;
    @Column
    private BigDecimal balance;

    public void setBalance(BigDecimal balance) {
        this.balance = balance;
    }

    public BigDecimal getBalance() {
        return balance;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public Integer getAge() {
        return age;
    }

    public Integer getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }
}

```

建表语句等自行设计，添加DAO：

```
@Repository
public interface UserDao extends JpaRepository<User,Integer> {
}
```

然后运行启动后可以根据`http://localhost:8080/users/x`来获取id=x的信息，到这里接完成了一个用户信息微服务，接下来创建服务消费者：

重新创建一个项目，依赖和前面一样（只需要基本的web与cloud依赖就行）同样的创建一个实体类，该类是POJO：

```java
public class User {

    private Integer id;

    private String username;

    private String name;

    private Integer age;

    private BigDecimal balance;

    public String getUsername() {
        return username;
    }

    public Integer getId() {
        return id;
    }

    public Integer getAge() {
        return age;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public BigDecimal getBalance() {
        return balance;
    }

    public void setBalance(BigDecimal balance) {
        this.balance = balance;
    }
}
```

创建启动类：

```java
@SpringBootApplication
public class start {
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
    public static  void main(String[] args){
        SpringApplication.run(start.class,args);
    }
}
```
创建控制器：

```java
@RestController
public class index {
    @Autowired
    RestTemplate restTemplate;
    @GetMapping("/getUser/{id}")
    public User get(@PathVariable Integer id){
        return restTemplate.getForObject("http://localhost:8080/users/"+id,User.class);
    }
    @GetMapping("/buy/{id}")
    public String buy(@PathVariable Integer id){
        User user =  restTemplate.getForObject("http://localhost:8080/users/"+id,User.class);
        if(user.getBalance().doubleValue() >= 90.00){
            double f = user.getBalance().doubleValue()-90.00;
            return "购票成功！余额："+f;
        }
        else {
            return "购票失败！余额不足";
        }
    }
}
```

这样就可以完成了整个微服务的构建。

<b id="a2"></b>

### :arrow_up_small: 为项目整合Spring Boot Actuator

:arrow_up:[返回目录](#t)

前面介绍spring Boot时我们知道提供了项目监控端点，我们可以通过这些端点来查看程序信息，步骤也非常简单，如下：

在前面的基础上添加依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
```

一个是actuator，一个是安全管理security。

添加额外的配置信息：

```java
management.endpoint.shutdown.enabled=true
management.endpoints.web.base-path=/
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=when_authorized
management.endpoint.health.roles=ADMIN

spring.security.user.name=admin
spring.security.user.password=123
spring.security.user.roles=ADMIN
```

一个适用于登录账号，一个是监控信息配置设置。弄好之后访问`http://localhost:8080/health`即可看到如下信息显示：

```javascript
{ 
    "status":"UP",
    "details":
          { "diskSpace":
               {  "status":"UP",
                   "details":
                   {
                       "total":338773929984,
                       "free":105464279040,
                       "threshold":10485760}
                   },
                   "db":
                   {
                      "status":"UP",
                      "details":
                         {
                             "database":"MySQL",
                             "hello":1
                         }
               }
           }
}
```

当然也可以配置其他信息如info端口：

```
info.app.name=@project.artifactId@
info.app.mysql=@mysql.version@
```


可以根据自己的需要配置你所需要的端点。

<b id="a3"></b>

### :arrow_up_small: 硬编码问题

:arrow_up:[返回目录](#t)

上面我们已经完成了一个简单的微服务系统，但是存在着一些问题，如下：

```java
@RestController
public class index {
    @Autowired
    RestTemplate restTemplate;
    @GetMapping("/getUser/{id}")
    public User get(@PathVariable Integer id){
        return restTemplate.getForObject("http://localhost:8080/users/"+id,User.class);
    }
    @GetMapping("/buy/{id}")
    public String buy(@PathVariable Integer id){
        User user =  restTemplate.getForObject("http://localhost:8080/users/"+id,User.class);
        if(user.getBalance().doubleValue() >= 90.00){
            double f = user.getBalance().doubleValue()-90.00;
            return "购票成功！余额："+f;
        }
        else {
            return "购票失败！余额不足";
        }
    }
}
```

可以知道我们的网络提供者IP地址是固定的，当然我们也可以添加到配置文件中：

```java
user:
  userServiceUrl: http://localhost:8080/users/
```

然后再来引用这个：

```java
  @Value("${user.userServiceUrl}")
    private String userServiceUrl;
    @Autowired
    RestTemplate restTemplate;
    @GetMapping("/buy/{id}")
    public String buy(@PathVariable Integer id){
        System.out.println(userServiceUrl+id);
        User user =  restTemplate.getForObject(userServiceUrl+id,User.class);

        if(user.getBalance().doubleValue() >= 90.00){
            double f = user.getBalance().doubleValue()-90.00;
            return "购票成功！余额："+f;
        }
        else {
            return "购票失败！余额不足";
        }
    }
```

传统的做法确实是这样的，但是这样有一个问题那就是当服务提供者的网络地址发生变化时，那么就需要重新修改程序，这样会引来很大的麻烦。所以微服务需要很多组件来完善这些功能，这些在后面将会一一介绍。












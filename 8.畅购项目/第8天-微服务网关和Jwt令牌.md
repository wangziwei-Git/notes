# 第8章 微服务网关和Jwt令牌

课程回顾：

1、学习了thymeleaf以及它的基本语法

- thymeleaf介绍

  - 它是模板引擎技术
  - 原理：  模板    +     数据    =   输出
  - 技术：freemarker、thymeleaf、velocity。
  - 与springboot：自动集成   开箱即用

- 基本语法：在页面引入th标签（类似：jstl   c标签）

  - 文本数据：th:text、th:utext
  - 指定form表单请求：th:action      请求：@{url}      获取服务器端的值：${}
  - 遍历数据：th:each
  - 流程控制：th:if，条件满足，执行代码块   th:unless，条件不满足，执行代码块
  - 引入其他模块：th:include

- 内置函数：#函数名称.方法

  ~~~
  1、#dates：日期操作
  2、#strings：字符串操作
  3、#maps：对map的操作
  ~~~

2、完成了搜索页面相关操作

- 回显检索的关键字
- 回显商品列表数据
- 回显商品分类、品牌、规格列表（Map<String,  Set<String>>）数据
- 通过条件对商品列表数据进行过滤
  - 本质：拼接过滤条件
- 隐藏用户已选择的过滤条件
- 展示用户已选择的条件
- 移除用于已选的条件
- 商品列表分页

3、通过thymeleaf生成商品详情的静态页

- 分析：静态页需要的数据
  - 商品基本信息
  - 商品库存信息
  - 商品分类信息
  - 商品图片列表
  - 商品规格列表
- 模板   +   （业务）数据   =   输出
- 生成静态页好处：提高响应的时间   
- 代码：TemplateEngine.process(模板，数据，输出)
- 访问：404
  - 第一次404：生成的静态页没有编译
  - 第二次404：没有对静态资源放行

4、更新商品上架程序      业务进行解耦：RabbitMQ

- 更新上架状态【goods服务】
- 将商品保存到ES【search服务】
- 生成该商品详情的静态页【item服务】



学习目标：   网关   +    JWT

+ 了解什么是微服务网关以及它的作用
+ 掌握微服务网关的系统搭建以及使用
+ 掌握通过网关实现限流
+ 掌握用户个人中心微服务的搭建
+ 了解JWT鉴权的介绍
+ 掌握JWT的鉴权的使用



# 1 微服务网关

![1608254042125](./总img/8/1608254042125.png)



## 1.1 微服务网关的概述

不同的微服务一般会有不同的网络地址，而外部客户端可能需要调用多个服务的接口才能完成一个业务需求，如果让客户端直接与各个微服务通信，会有以下的问题：

+ 客户端会多次请求不同的微服务，增加了客户端的复杂性
+ 存在跨域请求，在一定场景下处理相对复杂
+ 认证复杂，每个服务都需要独立认证
+ 难以重构，随着项目的迭代，可能需要重新划分微服务。例如，可能将多个服务合并成一个或者将一个服务拆分成多个。如果客户端直接与微服务通信，那么重构将会很难实施
+ 某些微服务可能使用了防火墙 / 浏览器不友好的协议，直接访问会有一定的困难

以上这些问题可以借助网关解决。

网关是介于客户端和服务器端之间的中间层，所有的外部请求都会先经过 网关这一层。也就是说，API 的实现方面更多的考虑业务逻辑，而安全、性能、监控可以交由 网关来做，这样既提高业务灵活性又不缺安全性，典型的架构图如图所示：

![1557824391432](./总img/8/1557824391432.png)

优点如下：

+ 安全 ，只有网关系统对外进行暴露，微服务可以隐藏在内网，通过防火墙保护。
+ 易于监控。可以在网关收集监控数据并将其推送到外部系统进行分析。
+ 易于认证。可以在网关上进行认证，然后再将请求转发到后端的微服务，而无须在每个微服务中进行认证。
+ 减少了客户端与各个微服务之间的交互次数
+ 易于统一授权。

总结：微服务网关就是一个系统，通过暴露该微服务网关系统，方便我们进行相关的鉴权，安全控制，日志统一处理，易于监控的相关功能。



## 1.2 微服务网关技术

实现微服务网关的技术有很多，

+ nginx  Nginx (tengine x) 是一个高性能的[HTTP](https://baike.baidu.com/item/HTTP)和[反向代理](https://baike.baidu.com/item/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86/7793488)web服务器，同时也提供了IMAP/POP3/SMTP服务
+ zuul ,Zuul 是 Netflix 出品的一个基于 JVM 路由和服务端的负载均衡器。
+ spring-cloud-gateway, 是spring 出品的 基于spring 的网关项目，集成断路器，路径重写，性能比Zuul好。

我们使用gateway这个网关技术，无缝衔接到基于spring cloud的微服务开发中来。

gateway官网：

https://spring.io/projects/spring-cloud-gateway



# 2 网关系统使用

# 2.1 需求分析

​	由于我们开发的系统 有包括前台系统和后台系统，后台的系统 给管理员使用。那么也需要调用各种微服务，所以我们针对 系统管理搭建一个网关系统。分析如下：

![1561959723219](./总img/8/\1561959723219.png)



## 2.2 搭建后台网关系统

### 2.2.1 添加依赖

引入依赖

`修改changgou-gateway工程`，打包方式为pom

![1566202174389](./总img/8/1566202174389.png)

pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>changgou-parent</artifactId>
        <groupId>com.changgou</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>changgou-gateway</artifactId>
    <packaging>pom</packaging>
    <modules>
        <module>changgou-gateway-web</module>
    </modules>

    <!--网关依赖-->
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>

</project>
```

### 2.2.2 搭建changgou-gateway-web工程

#### 2.2.2.1 创建工程

由上可知道，由于 需要有多个网关，所以为了管理方便。我们新建一个项目`<changgou-gateway-web>`，打包方式为pom,在里面建立各种网关系统模块即可。如图所示：

![1566202300066](./总img/8/1566202300066.png)

在changgou-gateway工程中，`创建 changgou-gateway-web工程`，该网关主要用于对后台微服务进行一个调用操作，将多个微服务串联到一起。

pom.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>changgou-gateway</artifactId>
        <groupId>com.changgou</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>changgou-gateway-web</artifactId>
    <description>
        普通web请求网关
    </description>
</project>
```



#### 2.2.2.2 编写启动类

在changgou-gateway-web中创建一个引导`启动类`com.changgou.GatewayWebApplication，代码如下：

```java
@SpringBootApplication
@EnableEurekaClient
public class GatewayWebApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayWebApplication.class,args);
    }
}
```



#### 2.2.2.3 编写application.yml文件

在changgou-gateway-web的resources下创建application.yml,代码如下：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]': # 匹配所有请求
              allowedOrigins: "*" #跨域处理 允许所有的域
              allowedMethods: # 支持的方法
                - GET
                - POST
                - PUT
                - DELETE
  application:
    name: gateway-web
server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
#暴露的端点(监控程序bean)
management:
  endpoint:
    gateway:
      enabled: true
    web:
      exposure:
        include: true
```



## 2.3 跨域配置说明

有时候，我们需要对所有微服务跨域请求进行处理，则可以在gateway中进行跨域支持。修改application.yml,添加如下代码：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]': # 匹配所有请求
              allowedOrigins: "*" #跨域处理 允许所有的域
              allowedMethods: # 支持的方法
                - GET
                - POST
                - PUT
                - DELETE
```



## 2.4 网关过滤配置

![1562033084310](./总img/8/\1562033084310.png)

路由过滤器允许以某种方式修改传入的HTTP请求或传出的HTTP响应。 路径过滤器的范围限定为特定路径。 Spring Cloud Gateway包含许多内置的GatewayFilter工厂。如上图，根据请求路径路由到不同微服务去，这块可以使用Gateway的路由过滤功能实现。

过滤器 有 20 多个 实现 类， 包括 头部 过滤器、 路径 类 过滤器、 Hystrix 过滤器 和 变更 请求 URL 的 过滤器， 还有 参数 和 状态 码 等 其他 类型 的 过滤器。

内置的过滤器工厂有22个实现类，包括 头部过滤器、路径过滤器、Hystrix 过滤器 、请求URL 变更过滤器，还有参数和状态码等其他类型的过滤器。根据过滤器工厂的用途来划分，可以分为以下几种：Header、Parameter、Path、Body、Status、Session、Redirect、Retry、RateLimiter和Hystrix。



### 2.4.1 Host 路由-略 

比如用户请求cloud.itheima.com的时候，可以将请求路由给http://localhost:18081服务处理，如下配置：

![1566371757753](./总img/8/1566371757753.png)

`和globalcors对其`上图配置如下：

```properties
      routes:
      		#唯一标识,可以乱写
            - id: changgou_goods_route
              #本地网关(路由代理或转发到具体某个服务)
              uri: http://localhost:18081
              #断言
              predicates:
              #域名
              - Host=cloud.itheima.com**

#**：匹配任意目录，可以是一个“*”或者多个“*”，没有区别
```

#### 注意

注意：此时要想让cloud.itheima.com访问本地计算机，要配置`C:\Windows\System32\drivers\etc\hosts`文件,映射配置如下：

![image-20201220154352567](D:\Java笔记_Typora\我添加的img\image-20201220154352567.png)

![image-20201220154421844](D:\Java笔记_Typora\我添加的img\image-20201220154421844.png)

```
127.0.0.1 cloud.itheima.com
```



* 在本地地址加本地连接回来

* 启动工程

  ![image-20201220154732194](D:\Java笔记_Typora\我添加的img\image-20201220154732194.png)

* 测试请求

  `http://cloud.itheima.com:8001/brand`,效果如下：

![1566372033946](./总img/8/1566372033946.png)



### 2.4.2 路径匹配过滤配置

我们还可以根据请求路径实现对应的路由过滤操作，例如请求中以`/brand/`路径开始的请求，都直接交给`http://localhost:18081`服务处理，如下配置：

![1566372062737](./总img/8/1566372062737.png)



上图配置如下：

```properties
      routes:
            - id: changgou_goods_route
              uri: http://localhost:18081
              #断言
              predicates:
              #路径匹配过滤配置:多级目录
              - Path=/brand/**


/brand:开头必须是"/brand"请求的url
/*:匹配一级目录，例：http://localhost:8001/brand/1115
/**:匹配多级目录，例：http://localhost:8001/brand/search/1/6
```



测试请求`<http://localhost:8001/brand/1115> `,效果如下：

![1566372165536](./总img/8/1566372165536.png)



### 2.4.3 PrefixPath 过滤配置

用户每次请求路径的时候，我们可以给真实请求加一个统一前缀，例如用户请求`http://localhost:8001`的时候我们让它请求真实地址`http://localhost:8001/brand`，如下配置：

![1566372374332](./总img/8/1566372374332.png)

上图配置如下：

```properties
      routes:
            - id: changgou_goods_route
              uri: http://localhost:18081
              predicates:
              #- Host=cloud.itheima.com**
              - Path=/**
              #局部过滤器
              filters:
              #前缀  :xxxGateway...
              - PrefixPath=/brand

/**:匹配多级目录
- PrefixPath:指定请求的前缀路径（也就是在访问的时候可以不同加“/brand”）
```



测试请求`http://localhost:8001/`效果如下：

![1566372461707](./总img/8/1566372461707.png)



### 2.4.4 StripPrefix 过滤配置

很多时候也会有这么一种请求，用户请求路径是`/api/brand`,而真实路径是`/brand，这时候我们需要去掉`/api`才是真实路径，此时可以使用SttripPrefix功能来实现路径的过滤操作，如下配置：

![1566372601482](./总img/8/1566372601482.png)

上图配置如下：

```properties
      routes:
            - id: changgou_goods_route
              uri: http://localhost:18081
              predicates:
              #- Host=cloud.itheima.com**
              - Path=/**
              filters:
              #- PrefixPath=/brand
              - StripPrefix=1

StripPrefix:可以携带的路径个数，
例如：http://localhost:8001/api/brand  或 http://localhost:8001/xxx/brand
```



测试请求`http://localhost:8001/api/brand`,效果如下：

![1566372641706](./总img/8/1566372641706.png)





### 2.4.5 LoadBalancerClient 路由过滤器(客户端负载均衡) 

上面的路由配置每次都会将请求给指定的`URL`处理，但如果在以后生产环境，并发量较大的时候，我们需要根据服务的名称判断来做负载均衡操作，可以使用`LoadBalancerClientFilter`来实现负载均衡调用。`LoadBalancerClientFilter`会作用在url以lb开头的路由，然后利用`loadBalancer`来获取服务实例，构造目标`requestUrl`，设置到`GATEWAY_REQUEST_URL_ATTR`属性中，供`NettyRoutingFilter`使用。

修改application.yml配置文件，代码如下：

![1566372699137](./总img/8/1566372699137.png)

上图配置如下：

```properties
      routes:
            - id: changgou_goods_route
              #uri: http://localhost:18081
              uri: lb://goods
              predicates:
              #- Host=cloud.itheima.com**
              - Path=/**
              filters:
              #- PrefixPath=/brand
              - StripPrefix=1
```



测试请求路径`http://localhost:8001/api/brand`

![1566372750975](./总img/8/1566372750975.png)

yml文件最终情况

```yml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]': # 匹配所有请求
            allowedOrigins: "*" #跨域处理 允许所有的域
            allowedMethods: # 支持的方法
              - GET
              - POST
              - PUT
              - DELETE
      routes:
        - id: changgou_goods_route
          #本地网关
#          uri: http://localhost:18081
          #动态路由
          uri: lb://goods
          #断言
          predicates:
            #域名
#            - Host=cloud.itheima.com**
            #路径匹配过滤配置
          - Path=/**
          filters:
          #前缀  :xxxGateway...
#          - PrefixPath=/brand
          #可以携带的路径个数，
          - StripPrefix=1
  application:
    name: gateway-web
server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
#暴露的端点(监控程序bean)
management:
  endpoint:
    gateway:
      enabled: true
    web:
      exposure:
        include: true
```



## 2.5 网关限流

网关可以做很多的事情，比如，限流，当我们的系统 被频繁的请求的时候，就有可能 将系统压垮，所以 为了解决这个问题，需要在每一个微服务中做限流操作，但是如果有了网关，那么就可以在网关系统做限流，因为所有的请求都需要先通过网关系统才能路由到微服务中。

![1608257077051](./总img/8/1608257077051.png)

Nginx：限流

- 限制单个客户端访问速率以及连接数
- 限制总连接数



网关（产品：Gateway、Nginx）限流：

- rate：**速率**，每秒访问的次数    rate=1/s
- brush：将多余的请求放入队列中   brush=1     队列：集成Redis。











### 2.5.1 思路分析

![](./总img/8/\1557909861570.png)



### 2.5.2 令牌桶算法

令牌桶算法是比较常见的限流算法之一，大概描述如下：
1）所有的请求在处理之前都需要拿到一个可用的令牌才会被处理；
2）根据限流大小，设置按照一定的速率往桶里添加令牌；
3）桶设置最大的放置令牌限制，当桶满时、新添加的令牌就被丢弃或者拒绝；
4）请求达到后首先要获取令牌桶中的令牌，拿着令牌才可以进行其他的业务逻辑，处理完业务逻辑之后，将令牌直接删除；
5）令牌桶有最低限额，当桶中的令牌达到最低限额的时候，请求处理完之后将不会删除令牌，以此保证足够的限流

如下图：

![1557910299016](./总img/8/1557910299016.png)

这个算法的实现，有很多技术，Guaua是其中之一，redis客户端也有其实现。



### 2.5.3 使用令牌桶进行请求次数限流

spring cloud gateway 默认使用redis的RateLimter限流算法来实现。所以我们要使用首先需要引入redis的依赖

#### 2.5.3.1 引入redis依赖

![1566212285804](./总img/8/1566212285804.png)

在changgou-gateway的pom.xml中引入redis的依赖

```xml
<!--redis-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    <version>2.1.3.RELEASE</version>
</dependency>
```



#### 2.5.3.2 定义KeyResolver

在Applicatioin引导类中添加如下代码，KeyResolver用于计算某一个类型的限流的KEY也就是说，可以通过KeyResolver来指定限流的Key。

我们可以根据IP来限流，比如每个IP每秒钟只能请求一次，`在GatewayWebApplication启动类中定义`key的获取，获取客户端IP，将IP作为key，如下代码：

```java
/***
 * IP限流
 * @return
 */
@Bean(name="ipKeyResolver")
public KeyResolver userKeyResolver() {
    return new KeyResolver() {
        @Override
        public Mono<String> resolve(ServerWebExchange exchange) {
            //获取远程客户端IP
            String hostName = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            //打印
            System.out.println("hostName:"+hostName);
            //返回
            return Mono.just(hostName);
        }
    };
}
```



#### 2.5.3.3 修改application.yml

指定限制流量的配置以及REDIS的配置，如图

修改如下图：

![1562049737562](./总img/8/\1562049737562.png)

配置代码如下：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]': # 匹配所有请求
              allowedOrigins: "*" #跨域处理 允许所有的域
              allowedMethods: # 支持的方法
                - GET
                - POST
                - PUT
                - DELETE
      routes:
            - id: changgou_goods_route
              uri: lb://goods
              predicates:
              - Path=/api/brand/**
              filters:
              - StripPrefix=1
              - name: RequestRateLimiter #请求数限流 名字不能随便写
                args:
                  key-resolver: "#{@ipKeyResolver}"  #spring-expression(SpEL表达式)获取bean,对应启动类
                  redis-rate-limiter.replenishRate: 1   #每秒访问1次
                  redis-rate-limiter.burstCapacity: 1   #将请求放入Redis-List队列

  application:
    name: gateway-web
  #Redis配置
  redis:
    host: 192.168.211.132
    port: 6379

server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
management:
  endpoint:
    gateway:
      enabled: true
    web:
      exposure:
        include: true
```

解释：

`redis-rate-limiter.replenishRate`是您希望允许用户每秒执行多少请求，而不会丢弃任何请求。这是令牌桶填充的速率

`redis-rate-limiter.burstCapacity`是指令牌桶的容量，允许在一秒钟内完成的最大请求数,将此值设置为零将阻止所有请求。

 key-resolver: "#{@ipKeyResolver}" 用于通过SPEL表达式来指定使用哪一个KeyResolver.

如上配置：

表示 一秒内，允许 一个请求通过，令牌桶的填充速率也是一秒钟添加一个令牌。

最大突发状况 也只允许 一秒内有一次请求，可以根据业务来调整 。



- `- StripPrefix配置说明`			

- ~~~properties
   routes:
   	- id: changgou_goods_route
   	uri: http://localhost:18081
   	predicates:
   	- Path=/**
   	filters:
   	- StripPrefix=1
   说明：http://localhost:port/xxx/brand   xxx可以任意匹配  路由：http://localhost:port/brand	
   
   
   
   routes:
         - id: changgou_goods_route
         uri: lb://goods
         predicates:
         - Path=/api/brand/**
         filters:
         - StripPrefix=1
   
     说明：http://localhost:8001/api/brand   路由：http://localhost:18081/brand
   
   ~~~

  测试

* 每秒1次正常

* 连续多次请求会发生如下情况(多次请求会发生如下情况)

![1566373479537](./总img/8/1566373479537.png)







# 3 用户登录

用户表结构：changgou_user：tb_user

![1608258704547](./总img/8/1608258704547.png)



## 3.1 表结构介绍

changgou_user表如下：

![1562050583653](./总img/8/\1562050583653.png)



用户信息表tb_user

```sql
CREATE TABLE `tb_user` (
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(100) NOT NULL COMMENT '密码，加密存储',
  `phone` varchar(20) DEFAULT NULL COMMENT '注册手机号',
  `email` varchar(50) DEFAULT NULL COMMENT '注册邮箱',
  `created` datetime NOT NULL COMMENT '创建时间',
  `updated` datetime NOT NULL COMMENT '修改时间',
  `source_type` varchar(1) DEFAULT NULL COMMENT '会员来源：1:PC，2：H5，3：Android，4：IOS',
  `nick_name` varchar(50) DEFAULT NULL COMMENT '昵称',
  `name` varchar(50) DEFAULT NULL COMMENT '真实姓名',
  `status` varchar(1) DEFAULT NULL COMMENT '使用状态（1正常 0非正常）',
  `head_pic` varchar(150) DEFAULT NULL COMMENT '头像地址',
  `qq` varchar(20) DEFAULT NULL COMMENT 'QQ号码',
  `is_mobile_check` varchar(1) DEFAULT '0' COMMENT '手机是否验证 （0否  1是）',
  `is_email_check` varchar(1) DEFAULT '0' COMMENT '邮箱是否检测（0否  1是）',
  `sex` varchar(1) DEFAULT '1' COMMENT '性别，1男，0女',
  `user_level` int(11) DEFAULT NULL COMMENT '会员等级',
  `points` int(11) DEFAULT NULL COMMENT '积分',
  `experience_value` int(11) DEFAULT NULL COMMENT '经验值',
  `birthday` datetime DEFAULT NULL COMMENT '出生年月日',
  `last_login_time` datetime DEFAULT NULL COMMENT '最后登录时间',
  PRIMARY KEY (`username`),
  UNIQUE KEY `username` (`username`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```



## 3.2 用户微服务创建

创建工程之前，先使用代码生成器生成对应的业务代码。

### 3.2.1 创建user-api

在changgou-service-api中`创建changgou-service-user-api工程`，并将逆向工程生成的代码pojo拷贝到工程中`com.changgou.user.feign`，如下图：

![1566227793521](./总img/8/1566227793521.png)



### 3.2.2 创建service-user工程

在changgou-service中`创建changgou-service-user微服务`,并将逆向工程生成的代码复制该工程中，如下图：

![1566228254513](./总img/8/1566228254513.png)



### 3.2.3 添加依赖

在changgou-service-user的pom.xml引入如下依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>changgou-service</artifactId>
        <groupId>com.changgou</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>changgou-service-user</artifactId>

    <!--依赖-->
    <dependencies>
        <dependency>
            <groupId>com.changgou</groupId>
            <artifactId>changgou-service-user-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
</project>
```



### 3.2.4 创建启动类

在changgou-service-user微服务中`创建启动类`com.changgou.UserApplication，代码如下：

```java
@SpringBootApplication//启动类注解
@EnableEurekaClient//eureka注册中心
@MapperScan(basePackages = {"com.changgou.user.dao"})//包扫描
public class UserApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}
```



### 3.2.5 application.yml配置

在changgou-service-user的resources中创建application.yml配置，代码如下：

```properties
server:
  port: 18088
spring:
  application:
    name: user
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://192.168.211.132:3306/changgou_user?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: 123456
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
feign:
  hystrix:
    enabled: true
```

### 测试

查询所有;`<http://localhost:18088/address>`

![image-20201220171743415](D:\Java笔记_Typora\我添加的img\image-20201220171743415.png)

## 3.3 登录

注意:不完整:只是为了测试Gateway网关和JWT鉴权(令牌)

登录的时候，

* 需要进行密码校验，
* 这里`采用了BCryptPasswordEncoder进行加密`，
* 需要将`资料`中的BCrypt工具类导入到common工程中，
* 其中BCrypt.checkpw("明文","密文")方法`用于对比密码是否一致`。

`修改changgou-service-user的com.changgou.user.controller.UserController类`添加登录方法，代码如下：

```java
	/**
     * @author wzw
     * 用户登录
     * @Date 17:21 2020/12/20
     * @param username 账号
     * @param password 密码
     * @return entity.Result
    **/
    @RequestMapping("/login")
    public Result login(String username, String password){
        //实现功能:查询指定用户
        User user = userService.findById(username);
        //判空,算法BCrypts加密密码做比较
        if (user != null && BCrypt.checkpw(password, user.getPassword())){
            //成功
            return new Result(true, StatusCode.OK, "登录成功");
        }

        //失败
        return new Result(false, StatusCode.LOGINERROR, "账号或者密码错误");
    }
```

注意：这里密码进行了加密。

测试：`<http://localhost:18088/user/login?username=szitheima&password=szitheima>`



![1566373919577](./总img/8/1566373919577.png)



## 3.4 网关关联

![1562055045607](./总img/8/\1562055045607.png)

在我们平时工作中，并不会直接将微服务暴露出去，一般都会使用网关对接，`实现对微服务的一个保护作用`，如上图，当用户访问`/api/user/`的时候我们再根据用户请求调用用户微服务的指定方法。当然，除了`/api/user/`还有`/api/address/`、`/api/areas/`、`/api/cities/`、`/api/provinces/`都需要由user微服务处理，

修改网关工程`changgou-gateway-web`的application.yml配置文件，如下代码：

![1566229631093](./总img/8/1566229631093.png)

上图代码如下：

```properties
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]': # 匹配所有请求
              allowedOrigins: "*" #跨域处理 允许所有的域
              allowedMethods: # 支持的方法
                - GET
                - POST
                - PUT
                - DELETE
      routes:
            - id: changgou_goods_route
              uri: lb://goods
              predicates:
              - Path=/api/album/**,/api/brand/**,/api/cache/**,/api/categoryBrand/**,/api/category/**,/api/para/**,/api/pref/**,/api/sku/**,/api/spec/**,/api/spu/**,/api/stockBack/**,/api/template/**
              filters:
              - StripPrefix=1
              - name: RequestRateLimiter #请求数限流 名字不能随便写 ，使用默认的facatory
                args:
                  key-resolver: "#{@ipKeyResolver}"
                  redis-rate-limiter.replenishRate: 1
                  redis-rate-limiter.burstCapacity: 1
            #用户微服务
            - id: changgou_user_route
              uri: lb://user
              predicates:
              - Path=/api/user/**,/api/address/**,/api/areas/**,/api/cities/**,/api/provinces/**
              filters:
              - StripPrefix=1
  application:
    name: gateway-web
  #Redis配置
  redis:
    host: 192.168.211.132
    port: 6379

server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
management:
  endpoint:
    gateway:
      enabled: true
    web:
      exposure:
        include: true
```



使用Postman访问`<http://localhost:8001/api/user/login?username=szitheima&password=szitheima> `，效果如下：

![1566229672590](./总img/8/1566229672590.png)



# 4 JWT介绍

![1608261337537](./总img/8/1608261337537.png)

JWT去解决该问题。

JWT：JSON Web Token（JWT）是一个非常轻巧的规范。这个规范**允许我们使用JWT在用户和服务器之间传递安全可靠的信息（字符串）**。



JWT的组成部分：

1、header：头部，存储一些用户的基本信息（介绍）【可有可无】  有，对头部信息，进行编码（base64编码）

2、playload：载荷，存储用户的重要信息（用户名、角色）【必须滴】

3、signature：签名/签证，对字符串进行**加密【加密算法】**



- 用户认证：单机环境（如果多服务场景，那么不能保证session共享，**而且不安全**。）

![1566275650282](D:\Java笔记_Typora\我添加的img\1566275650282.png)

- 用户认证：多服务场景

![1566276189171](D:\Java笔记_Typora\我添加的img\1566276189171.png)

- 加密-解密（数字签名）

![1566279010136](D:\Java笔记_Typora\我添加的img\1566279010136-1608446222453.png)





## 4.1 需求分析

我们之前已经搭建过了网关，使用网关在网关系统中比较适合进行权限校验。

![1562059385713](./总img/8/\1562059385713.png)

那么我们可以采用JWT的方式来实现鉴权校验。



## 4.2 什么是JWT

JSON Web Token（JWT）是一个非常轻巧的规范。这个规范允许我们使用JWT在用户和服务器之间传递安全可靠的信息。



## 4.3 JWT的构成

一个JWT实际上就是一个字符串，它由三部分组成，`头部、载荷与签名。`

### 4.3.1 头部（Header）

`头部`用于描述关于该JWT的`最基本的信息`，例如其类型以及签名所用的算法等。这也可以被表示成一个JSON对象。

```json

{"typ":"JWT","alg":"HS256"}
在头部指明了签名算法是HS256算法。 
我们进行BASE64编码http://base64.xpcha.com/，编码后的字符串如下：
```

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

> 小知识：Base64是一种基于64个可打印字符来表示二进制数据的表示方法。由于2的6次方等于64，所以每6个比特为一个单元，对应某个可打印字符。三个字节有24个比特，对应于4个Base64单元，即3个字节需要用4个可打印字符来表示。JDK 中提供了非常方便的 **BASE64Encoder** 和 **BASE64Decoder**，用它们可以非常方便的完成基于 BASE64 的编码和解码



```java
@Test
public void testEncoder() throws Exception{
    String msg = "www.itheima.com";
    // 编码
    byte[] encode = Base64.getEncoder().encode(msg.getBytes("UTF-8"));
    String encodeMsg = new String(encode, "UTF-8");
    System.out.println("编码后：" + encodeMsg);
    // 解码
    byte[] decode = Base64.getDecoder().decode(encode);
    String decodeMsg = new String(decode, "UTF-8");
    System.out.println("解码后：" + decodeMsg);
}
```


### 4.3.2 载荷（playload）

载荷就是存放有效信息的地方。这个名字像是特指飞机上承载的货品，这些有效信息包含三个部分

（1）`标准中注册的声明`（建议但不强制使用）



这些都是载荷中的属性信息。expire（设置ticket/token的过期时间）

```
iss: jwt签发者
sub: jwt所面向的用户
aud: 接收jwt的一方
exp: jwt的过期时间，这个过期时间必须要大于签发时间
nbf: 定义在什么时间之前，该jwt都是不可用的.
iat: jwt的签发时间
jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
```

（2）`公共的声明`

公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密.   

（3）`私有的声明`

私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。

这个指的就是自定义的claim。比如下面面结构举例中的admin和name都属于自定的claim。这些claim跟JWT标准规定的claim区别在于：JWT规定的claim，JWT的接收方在拿到JWT之后，都知道怎么对这些标准的claim进行验证(还不知道是否能够验证)；而private claims不会验证，除非明确告诉接收方要对这些claim进行验证以及规则才行。

定义一个payload:

```
{"sub":"1234567890","name":"John Doe","admin":true}
```

然后将其进行base64加密，得到Jwt的第二部分。

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```

### 4.3.3 签证（signature）

jwt的第三部分是一个`签证信息`，这个签证信息由三部分组成：

> header (base64后的)
>
> payload (base64后的)
>
> secret

这个部分需要base64加密后的header和base64加密后的payload使用.连接组成的字符串，然后通过header中声明的加密方式进行加盐secret组合加密，然后就构成了jwt的第三部分。

```
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

将这三部分用.连接成一个完整的字符串,构成了最终的jwt:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

### **注意**：

secret是保存在服务器端的，jwt的签发生成也是在服务器端的，secret就是用来进行jwt的签发和jwt的验证，所以，它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。



# 5 JJWT的介绍和使用

JJWT是一个提供端到端的JWT创建和验证的Java库。永远免费和开源(Apache License，版本2.0)，JJWT很容易使用和理解。它被设计成一个以建筑为中心的流畅界面，隐藏了它的大部分复杂性。

官方文档：

https://github.com/jwtk/jjwt



## 5.2 创建TOKEN

(1)依赖引入

`在changgou-parent项目`中的pom.xml中添加依赖：

![1566280945845](./总img/8/1566280945845.png)

```xml
<!--鉴权-->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.0</version>
</dependency>
```



(2)创建测试

在changgou-common的/test/java下创建`测试类`，并设置测试方法

```java
/**
 * @author wzw
 * 创建token
 * @Date 18:33 2020/12/20
 * @param null
 * @return
**/
@Test
public void createToken(){
    // 1.创建JWT
    JwtBuilder builder = Jwts.builder();
    
    // 2.构建头信息
    Map<String, Object> map = new HashMap<>();
    //信息没要求:可以见名之意
    map.put("alg", "HS256");
    map.put("keyId", "JWT");
    //设置头信息
    builder.setHeader(map);
    
    // 3.构建载荷信息
    //id
    builder.setId("001");
    //签发人
    builder.setIssuer("张三");
    //签发时间
    builder.setIssuedAt(new Date());
    
    // 4.添加签证(签名)(算法HS256,公司提供的密钥)
    builder.signWith(SignatureAlgorithm.HS256, "itheima");

    // 5.生成token
    String token = builder.compact();
    
    //加密格式:头信息.荷载.签证
    System.out.println("token:" + token);
}
```

运行打印结果：

```properties
eyJrZXlJZCI6IkpXVCIsImFsZyI6IkhTMjU2In0.eyJqdGkiOiIwMDEiLCJpc3MiOiLlvKDkuIkiLCJpYXQiOjE1NjYyODE5MDd9.ZsaAc2g5EvSssz11xJPloKoRmWwn63ek3jr0TrvNgiY
```

再次运行，会发现每次运行的`结果是不一样`的，因为我们的载荷中包含了时间。



## 5.2 TOKEN解析

我们刚才已经创建了token ，在web应用中这个操作是由服务端进行然后发给客户端，客户端在下次向服务端发送请求时需要携带这个token（这就好像是拿着一张门票一样），那服务端接到这个token 应该解析出token中的信息（例如用户id）,根据这些信息查询数据库返回相应的结果。

```java
@Test
public void testParseToken(){
    // 被解析的令牌
    String token = "eyJrZXlJZCI6IkpXVCIsImFsZyI6IkhTMjU2In0.eyJqdGkiOiIwMDEiLCJpc3MiOiLlvKDkuIkiLCJpYXQiOjE1NjYyODI3MjEsImV4cCI6MTU2NjI4Mjc1MX0.c8Tw0HWXypEb6c9bhJ7SuAA7I1tnEyUBUHZ-p_acm3M";
    // 创建解析对象
    JwtParser parser = Jwts.parser();
    //设置签证(签名)
    parser.setSigningKey("itheima");
    //parseClaimsJws:获取token中内容,getBody:获取载荷内容
    Claims claims = parser.parseClaimsJws(token).getBody();
    //= parser.parseClaimsJws(token).getBody();
    //打印
    System.out.println(claims);
}
```

运行打印效果：

```
header={keyId=JWT, alg=HS256},body={jti=001, iss=张三, iat=1566281907},signature=ZsaAc2g5EvSssz11xJPloKoRmWwn63ek3jr0TrvNgiY
```

试着将token或签名秘钥篡改一下，会发现运行时就会报错，所以解析token也就是验证token.



## 5.3 设置过期时间

有很多时候，我们并不希望签发的token是永久生效的，所以我们可以为token添加一个过期时间。

### 5.3.1 token过期设置

![1566282784577](./总img/8/1566282784577.png)

解释：

```Java
// 30秒后过期
builder.setExpiration(new Date(System.currentTimeMillis() + 30000));    
```

超过30s后，执行相关如下：







### 5.3.2 解析TOKEN

```java
@Test
public void testParseToken(){
    // 被解析的令牌
    String token = "eyJrZXlJZCI6IkpXVCIsImFsZyI6IkhTMjU2In0.eyJqdGkiOiIwMDEiLCJpc3MiOiLlvKDkuIkiLCJpYXQiOjE1NjYyODI3MjEsImV4cCI6MTU2NjI4Mjc1MX0.c8Tw0HWXypEb6c9bhJ7SuAA7I1tnEyUBUHZ-p_acm3M";
    // 创建解析对象
    JwtParser parser = Jwts.parser();
    //设置签证
    parser.setSigningKey("itheima");
    //parseClaimsJws:获取token中内容:header,body,签名
    Jws<Claims> claimsJws = parser.parseClaimsJws(token);
    //打印
    System.out.println(claimsJws);
}
```

打印效果：

![1566282831696](./总img/8/1566282831696.png)

当前时间超过过期时间，则会报错。



## 5.4 自定义claims

我们刚才的例子只是存储了id和subject两个信息，如果你想存储更多的信息（例如角色）可以定义自定义claims。

创建测试类，并设置测试方法：

创建token:

![1566283158341](./总img/8/1566283158341.png)

```java
//声明自定义载荷信息
HashMap<String, Object> claims = new HashMap<>();
claims.put("school", "五道口学习");
claims.put("address", "北五环");
builder.setClaims(claims);
```



运行打印效果：

```properties
eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI4ODgiLCJzdWIiOiLlsI_nmb0iLCJpYXQiOjE1NjIwNjMyOTIsImFkZHJlc3MiOiLmt7HlnLPpu5Hpqazorq3nu4PokKXnqIvluo_lkZjkuK3lv4MiLCJuYW1lIjoi546L5LqUIiwiYWdlIjoyN30.ZSbHt5qrxz0F1Ma9rVHHAIy4jMCBGIHoNaaPQXxV_dk
```



解析TOKEN:

```JAVA
@Test
public void testParseToken(){
    // 被解析的令牌
    String token = "eyJrZXlJZCI6IkpXVCIsImFsZyI6IkhTMjU2In0.eyJhZGRyZXNzIjoi5YyX5LqU546vIiwic2Nob29sIjoi5LqU6YGT5Y-j6IGM5Lia5oqA5pyv5a2m6ZmiIn0.bJRDjdLsLMbgFuuOYlLR6qPK9MrTHqbeW8Ggbm7JDaU";
    // 创建解析对象
    JwtParser parser = Jwts.parser();
    //设置签证(签名)
    parser.setSigningKey("itheima");
    //parseClaimsJws:获取token中内容,getBody:获取载荷内容
    Claims claims = parser.parseClaimsJws(token).getBody();
    //打印
    System.out.println(claims);
}
```

运行效果：

![1566283281140](./总img/8/1566283281140.png)



# 6 鉴权处理

## 6.1 思路分析

![1562069596308](./总img/8/\1562069596308.png)

```properties
1.用户通过访问微服务网关调用微服务，同时携带头文件信息
2.在微服务网关这里进行拦截，拦截后获取用户要访问的路径
3.识别用户访问的路径是否需要登录，如果需要，识别用户的身份是否能访问该路径[这里可以基于数据库设计一套权限]
4.如果需要权限访问，用户已经登录，则放行
5.如果需要权限访问，且用户未登录，则提示用户需要登录
6.用户通过网关访问用户微服务，进行登录验证
7.验证通过后，用户微服务会颁发一个令牌给网关，网关会将用户信息封装到头文件中，并响应用户
8.用户下次访问，携带头文件中的令牌信息即可识别是否登录
```



## 6.2 添加TOKEN工具类

在changgou-common中创建类entity.JwtUtil（创建令牌），主要辅助生成Jwt令牌信息，代码如下：

![1566284746112](./总img/8/1566284746112.png)

```java
    public class JwtUtil {

    //有效期为
    public static final Long JWT_TTL = 3600000L;// 60 * 60 *1000  一个小时

    //Jwt令牌信息
    public static final String JWT_KEY = "itcast";

    public static String createJWT(String id, String subject, Long ttlMillis) {
        //指定算法
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;

        //当前系统时间
        long nowMillis = System.currentTimeMillis();
        //令牌签发时间
        Date now = new Date(nowMillis);

        //如果令牌有效期为null，则默认设置有效期1小时
        if(ttlMillis==null){
            ttlMillis=JwtUtil.JWT_TTL;
        }

        //令牌过期时间设置
        long expMillis = nowMillis + ttlMillis;
        Date expDate = new Date(expMillis);

        //生成秘钥
        SecretKey secretKey = generalKey();

        //封装Jwt令牌信息
        JwtBuilder builder = Jwts.builder()
                .setId(id)                    //唯一的ID
                .setSubject(subject)          // 主题  可以是JSON数据
                .setIssuer("admin")          // 签发者
                .setIssuedAt(now)             // 签发时间
                .signWith(signatureAlgorithm, secretKey) // 签名算法以及密匙
                .setExpiration(expDate);      // 设置过期时间
        return builder.compact();
    }

    /**
     * 生成加密 secretKey
     * @return
     */
    public static SecretKey generalKey() {
        byte[] encodedKey = Base64.getEncoder().encode(JwtUtil.JWT_KEY.getBytes());
        SecretKey key = new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
        return key;
    }


    /**
     * 解析令牌数据
     * @param jwt
     * @return
     * @throws Exception
     */
    public static Claims parseJWT(String jwt) throws Exception {
        SecretKey secretKey = generalKey();
        return Jwts.parser()
                .setSigningKey(secretKey)
                .parseClaimsJws(jwt)
                .getBody();
    }
}
```



## 6.3 网关过滤器拦截请求处理

### 6.3.1 复制JwtUtil

拷贝JwtUtil到changgou-gateway-web（解析令牌）中（该工程并未依赖common工程，因此需要添加该工具类）

![1566284881954](./总img/8/1566284881954.png)





### 6.3.2 自定义全局过滤器

在changgou-gateway-web中`创建过滤器类`，如图所示：`com.changgou.filter.AuthorizeFilter`

![1566286756924](./总img/8/1566286756924.png)

AuthorizeFilter代码如下：

```java
/**
 * @ClassName AuthorizeFilter
 * @Description 自定义全局过滤器
 * @Author 传智播客
 * @Date 15:08 2019/8/20
 * @Version 2.1
 **/
@Component
public class AuthorizeFilter implements GlobalFilter, Ordered{

    // 定义常量
    private static final String AUTHORIZE_TOKEN = "Authorization";

    /**
     * @author 栗子
     * @Description 业务处理
     * @Date 15:09 2019/8/20
     * @param exchange
     * @param chain
     * @return reactor.core.publisher.Mono<java.lang.Void>
     **/
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        // 获取request、response
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        
        // 1、判断用户是否是登录操作，如果是，则直接放行
        //获取用户请求路径
        String url = request.getURI().getPath(); 
        // 判断是否是登录
        if (url.startsWith("/api/user/login")){ 
            //放行
            return chain.filter(exchange);
        }

        // 2、其他请求，需要判断用户是否登录（判断用户是否携带token信息）,三种方式都判断
        // 2.1 从请求参数获取token
        String token = request.getQueryParams().getFirst(AUTHORIZE_TOKEN);
        // 非空判断
        if (StringUtils.isEmpty(token)){
            // 2.2 从请求头获取token
            token = request.getHeaders().getFirst(AUTHORIZE_TOKEN);
        }
        // 非空判断
        if (StringUtils.isEmpty(token)){
            // 2.3 从cookie中获取token
            HttpCookie cookie = request.getCookies().getFirst(AUTHORIZE_TOKEN);
            if (cookie != null){
                token = cookie.getValue();
            }
        }
        
        // 3、如果没有token，不放行
        if (StringUtils.isEmpty(token)){
            // 设置响应状态码,401无效认证
            response.setStatusCode(HttpStatus.UNAUTHORIZED);   
            return response.setComplete();
        }

        // 4、token存在，需要解析token
        try {
            // 解析成功
            Claims claims = JwtUtil.parseJWT(token);
        } catch (Exception e) {
            e.printStackTrace();
            // 解析失败
            response.setStatusCode(HttpStatus.UNAUTHORIZED);    // 设置响应状态码
            return response.setComplete();
        }
        
        // 5.放行
        return chain.filter(exchange);
    }

    /**
     * @author 栗子
     * @Description 过滤器执行顺序
     * @Date 15:09 2019/8/20
     * @param
     * @return int
     **/
    @Override
    public int getOrder() {
        return 0;
    }
}
```





## 6.5 配置过滤规则

修改网关系统的yml文件（请求这些资源都需要用户携带token）

![1566287088829](./总img/8/1566287088829.png)

上述代码如下：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]': # 匹配所有请求
              allowedOrigins: "*" #跨域处理 允许所有的域
              allowedMethods: # 支持的方法
                - GET
                - POST
                - PUT
                - DELETE
      routes:
            - id: changgou_goods_route
              uri: lb://goods
              predicates:
              - Path=/api/album/**,/api/brand/**,/api/cache/**,/api/categoryBrand/**,/api/category/**,/api/para/**,/api/pref/**,/api/sku/**,/api/spec/**,/api/spu/**,/api/stockBack/**,/api/template/**
              filters:
              - StripPrefix=1
              - name: RequestRateLimiter #请求数限流 名字不能随便写 ，使用默认的facatory
                args:
                  key-resolver: "#{@ipKeyResolver}"
                  redis-rate-limiter.replenishRate: 1
                  redis-rate-limiter.burstCapacity: 1
            #用户微服务
            - id: changgou_user_route
              uri: lb://user
              predicates:
              - Path=/api/user/**,/api/address/**,/api/areas/**,/api/cities/**,/api/provinces/**
              filters:
              - StripPrefix=1

  application:
    name: gateway-web
  #Redis配置
  redis:
    host: 192.168.211.132
    port: 6379

server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka
  instance:
    prefer-ip-address: true
management:
  endpoint:
    gateway:
      enabled: true
    web:
      exposure:
        include: true
```



## 6.6 未登录测试

### 测试访问

#### 失败测试

`<http://localhost:8001/api/brand>`，效果如下：

![1566287454004](./总img/8/1566287454004.png)

#### 登录测试:

`<http://localhost:8001/api/user/login?username=szitheima&password=szitheima>`![image-20201220205428766](D:\Java笔记_Typora\我添加的img\image-20201220205428766.png)

参考官方手册：

https://cloud.spring.io/spring-cloud-gateway/spring-cloud-gateway.html#_stripprefix_gatewayfilter_factory





## 6.7 会话保持

用户每次请求的时候，我们都需要获取令牌数据，方法有多重，可以在每次提交的时候，将数据提交到头文件中，也可以将数据存储到Cookie中，每次从Cookie中校验数据，还可以每次将令牌数据以参数的方式提交到网关，这里面采用Cookie的方式比较容易实现。

修改user微服务，每次登录的时候，添加令牌信息到Cookie中，修改changgou-service-user的`com.changgou.user.controller.UserController`的`login`方法，代码如下：

![1596874988386](./总img/8/1596874988386.png)

~~~java
@RequestMapping("/login")
public Result login(String username, String password, HttpServletResponse response){
    // 实现功能:判断用户名是否正确
    User user = userService.findById(username);
    //非空判断
    if (user != null && BCrypt.checkpw(password, user.getPassword())){
        // 用户登录成功后，生成token
        Map<String, Object> map = new HashMap<>();
        //添加头信息
        map.put("role", "ADMIN");
        //转json格式
        String subject = JSON.toJSONString(map);
        //调用工具类创建token(用户名,头信息,过期时间60秒(有默认))
        String token = JwtUtil.createJWT(username, subject,  Long.parseLong("60000"));
        // 将token存储到cookie中
        Cookie cookie = new Cookie("Authorization", token);
        //设置范围
        cookie.setPath("/");
        //二级域名,cookie共享
        cookie.setDomain("localhost");
        //响应Cookie
        response.addCookie(cookie);
        //返回结果
        return new Result(true, StatusCode.OK, "用户登录成功");
    }else {
        return new Result(false, StatusCode.LOGINERROR, "用户名或者密码错误");
    }
}
~~~

## 6.8 已登录测试

### 注意

* token过期时间60秒

1、用户先登录`<http://localhost:8001/api/user/login?username=szitheima&password=szitheima>`

![1566374583354](./总img/8/1566374583354.png)

2、再次访问：`<http://localhost:8001/api/brand>`

![1566290777315](./总img/8/1566290777315.png)
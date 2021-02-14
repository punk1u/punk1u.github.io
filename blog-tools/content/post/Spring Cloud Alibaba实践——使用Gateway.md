---
title: "Spring Cloud Alibaba实践——使用Gateway" 
date: 2021-02-05T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——使用Gateway

在微服务系统中，可能会有很多的微服务，且每个服务可能会有多个实例。如果在客户端直接使用每个微服务实例的地址去访问微服务，会变得非常复杂且难以维护。而且如果客户端和服务端之间需要做登录权限等权限验证功能，就需要后台每个微服务都要支持权限验证功能，增加代码冗余和复杂度。可能还会有后台提供的接口的通信协议不是需要的协议的情况。这个时候就需要引入网关来解决，可以将以上的`反向代理`、`面向客户端负载均衡`、`权限验证`，`协议转换`的功能统一放在网关层解决。



## Spring Cloud Gateway概述

`Spring Cloud Gateway`是`Sring Cloud`的第二代网关实现，未来会取代第一代网关`Zuul`。`Spring Cloud Gateway`基于`Netty`、`Reactor`以及`WebFlux`构建。



优点：

1. 性能强劲

   得益于`Netty`、`Reactor`的高性能，`Spring Cloud Gateway`的性能非常好，是`Zuul`的`1.6`倍。

2. 功能强大

   内置了很多实用功能，比如转发、监控、限流等。

3. 设计优雅，易扩展



缺点：

1. 依赖`Netty`和`WebFlux`

   不是`Servlet`编程模型，而是`Reactive`模型，有一定的适应成本

2. 不能在`Tomcat`等`Servlet`容器下工作，也不能构建成`WAR`包

3. 不支持`Spring Boot 1.x`



## 编写Spring Cloud Gateway

新建`Gateway`网关项目并添加相应的`Nacos`服务发现依赖、`Gateway`网关依赖，`Gateway`项目的`Maven`依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>tech.punklu</groupId>
    <artifactId>gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gateway</name>
    <description>网关</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <!-- 为Gateway添加Nacos的依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <!-- 整合actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>

            <!--整合spring cloud-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--整合spring cloud alibaba-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.2.3.RELEASE</version>
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





添加相应的配置，配置`Gateway`从`Nacos`发现服务：

```yaml
server:
  port: 8040
spring:
  application:
    # 指定Gateway注册到Nacos上的名称
    name: gateway
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    gateway:
      discovery:
        locator:
          # 让Gateway通过服务发现组件找到其他的微服务
          enabled: true
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always

```



启动`用户中心`、`内容中心`、`Gateway`网关服务，访问`Gateway`的`http://127.0.0.1:8040/content-center/shares/1?origin=browser`以及`http://127.0.0.1:8040/user-center/users/1?origin=browser`，可以看到已经可以正常通过网关的反向代理访问`用户中心`、`内容中心`的接口并查询到相应的数据了。并且访问的时候并不是通过目标微服务的ip地址直接访问的，而是通过各个微服务自定义的服务名进行访问的，也就是网关实现了自动转发功能。



## Gateway核心概念

1. Route(路由)

   `Spring Cloud Gateway`的基础元素，可简单理解为一条转发的规则。包含：`ID`、`目标URL`、`Predicate`集合以及`Filter`集合。

2. Predicate（谓词）

   即`java.util.function.Predicate`，是函数式编程的一个`API`。`Spring Cloud Gateway`使用`Predicate`实现路由的匹配条件。控制请求是否走转发规则的条件。

3. Filter（过滤器）

   修改请求以及响应。



以下面的配置为例：

```yaml
spring:
  cloud:
	gateway:
	  route:
		id: some_route
		uri: http://127.0.0.1:8082
		predicates:
		  Path=/users/{id}
		filters:
		  AddRequestHeader=X-Request-Foo,Bar
```



可以看出一个路由(Route)由`id`、`uri`、`predicates`、`filters`过滤器集合组成。代表如果访问`/users/{id}`这个路径就会进入这个路由规则，然后经过过滤器的规则过滤后被转发到`http://127.0.0.1:8082`这个`uri`。



上面的初始化配置：

```yaml
gateway:
  discovery:
    locator:
      # 让Gateway通过服务发现组件找到其他的微服务
      enabled: true
```

代表没有特殊需求，让`Gateway`按照默认方式处理所有请求。



## Gateway路由谓词工厂

`Gateway`内置了很多谓词工厂进行谓词规则的配置，它们统一实现了`RoutePredicateFactory`接口。按照用途的不同可以分为四大类：

1. 时间相关

   - `AfterRoutePredicateFactory`
   - `BeforeRoutePredicateFactory`
   - `BetweenRoutePredicateFactory`

2. Cookie相关

   - `CookieRoutePredicateFactory`

3. Header相关

   - `HeaderRoutePredicateFactory`

   - `HostRoutePredicateFactory`

4. 请求相关

   - `MethodRoutePredicateFactory`
   - `PathRoutePredicateFactory`
   - `QueryRoutePredicateFactory`
   - `RemoteRoutePredicateFactory`



可以使用这些内置的谓词工厂来定制路由规则。



## 自定义路由谓词工厂

当内置的谓词工厂不能满足需求时就需要自定义路由谓词工厂了。



比如，像`12306`一样限定服务只能在`上午9:00`-`下午5:00`之间才能访问，就需要自定义路由谓词工厂了。



首先添加配置项：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: between_route
          uri: lb://user-center
          predicates:
            - TimeBetween=上午9:00,下午11:00
```



为了解析这个自定义的配置信息需要新建一个配置信息类存放这个配置：

```java
package tech.punklu.gateway;

import lombok.Data;

import java.time.LocalTime;

@Data
public class TimeBetweenConfig {

    /**
     * 起始时间
     */
    private LocalTime start;

    /**
     * 结束时间
     */
    private LocalTime end;
}

```



然后新建`谓词规则`类：

```java
package tech.punklu.gateway;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.time.LocalTime;
import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

/**
 * 自定义谓词规则类
 */
@Component
public class TimeBetweenRoutePredicateFactory extends AbstractRoutePredicateFactory<TimeBetweenConfig> {

    public TimeBetweenRoutePredicateFactory() {
        super(TimeBetweenConfig.class);
    }

    /**
     * 判断路由谓词规则的方法
     * @param config
     * @return
     */
    @Override
    public Predicate<ServerWebExchange> apply(TimeBetweenConfig config) {
        LocalTime start = config.getStart();
        LocalTime end = config.getEnd();
        return exchange->{
          LocalTime now = LocalTime.now();
          return now.isAfter(start) && now.isBefore(end);
        };
    }

    /**
     * 从配置文件获取配置的谓词规则信息并把值赋给配置对象(TimeBetweenConfig)里的属性的方法
     * @return
     */
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("start","end");
    }
}

```

需要注意的是，这个类的类名必须以`RoutePredicateFactory`结尾（Gateway的默认规定）。



然后启动网关服务，访问`http://127.0.0.1:8040/users/1?origin=browser`可以看到网关直接返回`404`，而当修改配置文件内的时间区间，使当前时间处于设置的区间内时，再次访问这个`url`就可以正常访问了。需要特别注意的是，这个`url`中并没有指定这个接口所在的`user-center`标识，但是依然`Gateway`可以正常找到对应的`用户服务`并调用，是因为在配置项里已经通过`uri`的形式指定了`用户服务`:

```yaml
uri: lb://user-center
```



## Gateway过滤器工厂

过滤器可以为请求和响应添加一些业务逻辑。可以灵活的操作请求和响应，比如操作请求的`parameter`和`header`等信息。



比如，现在要在`Gateway`加一个在发送向`用户中心`的请求上添加一个名为`X-Request-Foo`，值为`Bar`的`Header`项。可以添加如下配置：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: between_route
          uri: lb://user-center
          predicates:
            - TimeBetween=上午0:00,下午11:59
          # 添加一个向发往用户中心的请求上添加Header项的拦截器,key为X-Request-Foo,value为Bar
          filters:
            - AddRequestHeader=X-Request-Foo, Bar
```



启动项目后，在`NettyRoutingFilter`的`Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)`方法上打上注解，然后访问`http://127.0.0.1:8040/users/1?origin=browser`，可以在断点处看到请求的`Header`中已经被加上了配置文件中指定的`Header`项的`K-V`值。



## 自定义过滤器工厂

和自定义路由谓词工厂一样，也可以在`Gateway`中自定义过滤器工厂。



过滤器生命周期:

1. pre

   Gateway转发请求之前

2. post

   Gateway转发请求之后



假设现在的需求是进入过滤器的时候，打印一下相应的日志。



首先新建过滤器工厂类：

```java
package tech.punklu.gateway;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractNameValueGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

@Slf4j
@Component
public class PreLogGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {
    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return ((exchange, chain) -> {
            log.info("请求进来了...{},{}",config.getName(),config.getValue());
            ServerHttpRequest modifiedRequest = exchange
                    .getRequest()
                    .mutate()
                    .build();
            ServerWebExchange modifiedExchange = exchange.mutate().request(modifiedRequest).build();
            return chain.filter(modifiedExchange);
        });
    }
}

```

需要注意的是，过滤器类类名必须以`GatewayFilterFactory`结尾。



然后增加过滤器配置项：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: between_route
          uri: lb://user-center
          predicates:
            - TimeBetween=上午0:00,下午11:59
          # 添加一个向发往用户中心的请求上添加Header项的拦截器,key为X-Request-Foo,value为Bar
          filters:
            - PreLog=a,b
```

在过滤器中添加一个配置项，将参数传递进过滤器工厂类里，参数的key`PreLog`需要和过滤器工厂类的前缀保持一致。



重启`Gateway`，访问`http://127.0.0.1:8040/users/1?origin=browser`，可以看到日志里已经打印了请求执行的日志以及传入的参数。



## Gateway监控

只要为Spring Cloud Gateway添加Spring Boot Actuator（ `spring-boot-starter-actuator` ）的依赖，并将 `gateway` 端点暴露，即可获得若干监控端点，监控 & 操作Spring Cloud Gateway的方方面面。



```yaml
management:
  endpoints:
    web:
      exposure:
        # 当然暴露'*' 更好啦..
        include: gateway
```



监控端点一览表：

以下所有端点都挂在`/actuator/gateway/` 下面。
例如：`routes` 的全路径是 `/actuator/gateway/routes` ，以此类推。



| ID            | HTTP Method | Description                                     |
| ------------- | ----------- | ----------------------------------------------- |
| globalfilters | GET         | 展示所有的全局过滤器                            |
| routefilters  | GET         | 展示所有的过滤器工厂（GatewayFilter factories） |
| refresh       | POST        | 清空路由缓存                                    |
| routes        | GET         | 展示路由列表                                    |
| routes/{id}   | GET         | 展示指定id的路由的信息                          |
| routes/{id}   | POST        | 新增一个路由                                    |
| routes/{id}   | DELETE      | 删除一个路由                                    |






---
title: "Spring Cloud Alibaba实践——调用链监控Sleuth" 
date: 2021-02-12T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——调用链监控Sleuth



## 需求和背景

在微服务场景下，应用被拆分成多个微服务，并互相调用，在这种情况下，跨微服务的API调用发生异常或者发生性能瓶颈时，需要快速定位并解决问题。这时可以使用调用链监控工具来监控微服务调用链的实时情况。



## <span id="monitorTheory">调用链监控原理</span>

以之前`内容中心`里的`/shares/{id}`接口调用`用户中心`的`/users/{id}`接口查询投稿人的信息为例：

1. `客户端发送(Client Send)--CS`

   表示`内容中心`向用户中心发送请求。

2. `服务端接收(Server Receive)--SR`

   表示`用户中心`接收到请求。

3. `服务端响应(Server Send)——SS`

   表示`用户中心`接收到请求并处理完成后响应`内容中心`。

4. `客户端接收(Client Receive)--CR`

   表示`内容中心`接收到`用户中心`的响应。



其中这四个阶段包括三个过程。第一个过程是`CS-SR`，这个过程表示从客户端发送请求开始，到服务端接收请求后的网络通讯时间。第二个过程是`SR-SS`，这个过程表示从服务端接收到请求开始，到服务端执行请求结束时的处理时间。第三个过程是`SS-CR`，这个过程表示从服务端发起响应请求开始，到客户端接收到响应结束时的网络通讯时间。



通过记录这三个过程的持续时间，即可实现对调用链的监控。



## 整合Sleuth

`Sleuth`是一个`Spring Cloud`的分布式跟踪解决方案。



### Sleuth术语

1. `Span(跨度)`
   `Sleuth`的基本工作单元，它用一个64位的id唯一标识。除`ID`外，`span`还包含其他数据，例如描述、时间戳事件、键值对的注解（标签）、`span ID`、`span父ID`等。


   `Span`对应上面的[调用链监控原理](#monitorTheory)里的四阶段中的某一阶段。

2. `trace(跟踪)`

   一组`span`组成的树状结构称为`trace`。


   `trace`对应一个调用链的多个执行步骤的详细信息。

3. `Annotation(标注)`

   - `CS(Client Sent 客户端发送)`：客户端发起一个请求，该`annotation`描述了`span`的开始。
   - `SR(Server Received服务器端接收)`：服务器端获得请求并准备处理它。
   - `SS(Server Sent服务器端发送)`：该`annotation`表明完成请求处理（当响应发回客户端时）。
   - `CR(Client Received客户端接收)`：`span`结束的标识。客户端成功接收到服务器端的响应。



### 为用户中心整合Sleuth

首先添加`Sleuth`的依赖：

```xml
<!-- 添加Spring Cloud  sleuth依赖 -->
<!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
    <version>2.2.7.RELEASE</version>
</dependency>
```



可在配置项中配置`Sleuth`的相关日志打印信息：

```yaml
logging:
  level:
    org.springframework.cloud.sleuth: debug
```



启动用户中心后，访问`127.0.0.1:8081/users/1`，不添加`X-Token`可以看到后台打印出了对应的`Sleuth`的日志信息，包含`应用名称`、`traceId`、`spanId`、`是否将信息传递到Zipkin`等信息。



但是，这种日志信息不方便查看，不利于快速定位问题，所以需要引入`Zipkin`客户端来更方便的查看调用链监控信息。



### Zipkin搭建与整合

`Zipkin`是`Twitter`开源的分布式追踪系统，主要用来收集系统的`时序数据`，从而追踪系统的调用问题。



访问如下地址下载最新`Zipkin`控制台程序：

```
https://search.maven.org/remote_content?g=io.zipkin&a=zipkin-server&v=LATEST&c=exec
```



使用`jar -jar `命令运行下载下来的`jar`包即可。



然后在`用户中心`中添加`Zipkin`的依赖：

```xml
<!-- 添加Spring zipkin依赖，其中已包含sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.2.7.RELEASE</version>
</dependency>
```



因为`Zipkin`的依赖项中已经包含了`Sleuth`了，所以之前的`Sleuth`的依赖项可以去除掉。



然后添加`Zipkin`的配置信息：

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411/
  sleuth:
    sampler:
      # 抽样率，默认是0.1(10%)
      probability: 1.0
```



抽样率设置为`1.0`表示将把全部的调用信息都上传到`Zipkin`客户端上，以便测试使用。





启动`用户中心`，访问两次`127.0.0.1:8081/users/1`接口，一次带`X-Token`参数，一次不带`X-Token`参数，然后访问`http://127.0.0.1:9411/zipkin/`，选择`serviceName`为`用户中心`，查询可以看到刚才的两次调用记录，一次成功，一次失败，查看详情可以看到详细HTTP信息以及对应的执行时间信息。同时`Zipkin`控制台还支持丰富的按条件查询的功能，可以很方便地查询监控数据。



### 为所有微服务整合Zipkin

将前面为`内容中心`整合`Zipkin`的相关代码添加到`Gateway`和`用户中心`里，并配置好`Gateway`网关到`用户中心`、`内容中心`的映射，如下所示：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_route
          uri: lb://user-center
          predicates:
            - TimeBetween=上午0:00,下午11:59
            # 配置Gateway要映射到用户中心的uri的地址前缀
            - Path=/users/**
          # 添加一个向发往用户中心的请求上添加Header项的拦截器
          filters:
            - AddRequestHeader=X-Request-Foo, Bar
            - PreLog=a,b
        - id: content_route
          uri: lb://content-center
          predicates:
            - TimeBetween=上午0:00,下午11:59
            # 配置Gateway要映射到内容中心的uri的地址前缀
            - Path=/shares/**,/admin/**
          # 添加一个向发往内容中心的请求上添加Header项的拦截器
          filters:
            - AddRequestHeader=X-Request-Foo, Bar
            - PreLog=a,b
```



然后启动三个微服务，访问`127.0.0.1:8040/shares/1?origin=browser`两次，一次带`X-Token`，一次不带。因为这个接口的调用路径是`Gateway->content-center->user-center`，所以此时访问`http://127.0.0.1:9411/zipkin`，选择`serviceName`为`gateway`的服务，可以查询到刚才的两次调用记录，一次成功，一次失败，点击成功的那条记录查看详情，可以看到调用链路上的每一次记录，及每一个节点的开始时间，可以很方便地分析微服务之间的调用链信息及相关性能瓶颈等问题。



## Zipkin持久化

默认情况下`Zipkin`里的调用链监控记录是保存在内存中的，在生产环境需要对其进行持久化，可以将其保存在`MySQL`、`ElasticSearch`、`Cassandra`等工具中。`MySQL`存储方案，当数据量较大时，会出现性能问题，会需要秒级才能查询到数据的情况。



相关文档：`https://github.com/openzipkin/zipkin#storage-component`






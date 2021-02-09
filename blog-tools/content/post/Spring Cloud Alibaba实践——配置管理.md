---
title: "Spring Cloud Alibaba实践——配置管理" 
date: 2021-02-07T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——配置管理



## 为什么要实现配置管理？

- 不同环境，不同配置
- 配置属性动态刷新



## 使用Nacos实现配置管理



## 远程、本地配置优先级问题

默认情况下，远程Nacos的配置优先级高于本地配置，可以通过以下三个配置项修改优先级顺序（需要把这几个配置放在远程Nacos上才能使这几个配置生效）：

```yaml
spring:
  cloud:
  	config:
  	  # 是否允许本地配置覆盖远程配置
  	  override-none: true
  	  # 一切以本地配置为准
  	  allow-override: false
  	  # 系统环境变量或系统属性才能覆盖远程配置文件的配置
  	  # 本地配置文件中配置优先级低于远程配置
  	  override-system-properties: false
```



## 使用Nacos完成远程配置管理

以内容中心为例，首先添加`Nacos配置管理`的依赖：

```xml
<!-- 添加Nacos配置管理依赖 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```



对于普通的配置管理来说，使用默认配置即可满足需求，例如，只需要在`Nacos`的默认命名空间中管理`内容中心`的配置，则只需要在`Nacos控制台`中配置中心里默认的`public`命名空间里使用默认的`DEFAULT_GROUP`组里新建一个`data-id`名为`content-center.yaml`的配置信息即可。但是需要注意的是，这里的命名需要以如下方式命名：

```xml
${prefix}-${spring.profiles.active}.${file-extension} 
```



其中：

```xml
a). prefix 默认spring.application.name 的值，也可以通过配置项 spring.cloud.nacos.config.prefix来配置

b). file-extension默认properties，比如我这里使用的是yaml，那么更改spring.cloud.nacos.config.file-extension= yaml

c). Group默认DEFAULT_GROUP，也可以通过配置项 spring.cloud.nacos.config.group来配置
```



这个`data-id`中只有一个测试的配置项：

```yaml
nacosKey:
  nacosValue
```



然后在内容中心中新建`bootstrap.yml`文件，并在其中添加如下的配置项引导`Nacos配置中心`:

```yaml
server:
  port: 8082

spring:
  application:
    # 向Nacos注册服务的时候会使用这个名字
    name: content-center
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        # 配置Nacos上的配置项的文件扩展名格式，当Nacos中使用了非properties格式的配置时，必须指明，因为默认值为properties
        file-extension: yaml
        # 配置当Nacos中的配置项的值变更时，自动更新到项目中（默认为true）
        refresh-enabled: true
```

注意这里使用bootstrap.yml而非application.yml，避免applicaton.yml后加载于nacos配置并覆盖

SpringBoot读取配置文件顺序：`bootstrap.yml`>`bootstrap.yaml`>`bootstrap.properties`>`nacos的配置`>`application.yml`>`application.yaml`>`application.properties`。



编写测试类：

```java
package tech.punklu.contentcenter.controller.content.nacos;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 不光需要在配置文件中自动刷新配置：refresh-enabled: true，
 * 还需要在使用到Nacos配置信息的类上添加@RefreshScope注解才能实现自动更新
 */
@RestController
@RequestMapping("/nacos")
@RefreshScope
public class NacosController {

    @Value("${nacosKey}")
    private String nacosKey;

    @GetMapping(value = "/getNacosValue",produces = MediaType.APPLICATION_JSON_VALUE)
    public String getNacosValue(){
        return nacosKey;
    }
}
```



启动项目，访问`http://127.0.0.1:8082/nacos/getNacosValue?origin=browser`，可以看到已经可以正常访问`Nacos`中配置的值了。



## 实现Nacos配置共享

在实际应用中，会有多个服务中有一些通用配置的情况，以及各个项目的配置需要互相分开维护的情况。对于这种情况，可以通过`Nacos`的命名空间`Namespace`功能和`Group`组来实现对应的需求。



假设现在有一个所有服务共用的配置项`common`，还有一个所有`spring-cloud`公用的配置项`spring-cloud-common`，以及一个`内容中心`单独使用的配置项`content-center`。



如下所示，新建一个专门用于这个项目的名为`spring-cloud-alibaba-demo`的`命名空间`，以及这个命名空间底下的不同的`group`下的两个不同层级的共享配置和一个`内容中心`自己的配置：

| 命名空间                  | group               | data-id                  |
| ------------------------- | ------------------- | ------------------------ |
| spring-cloud-alibaba-demo | common              | common.yaml              |
| spring-cloud-alibaba-demo | spring-cloud-common | spring-cloud-common.yaml |
| spring-cloud-alibaba-demo | content-center      | content-center.yaml      |



其中`common.yaml`的配置信息为：

```yaml
commonKey:
  commonValue
```



`spring-cloud-common.yaml`的配置信息为：

```yaml
springCloudCommonKey:
  springCloudCommonValue
```



`  content-center.yaml`表示`内容中心`的配置，配置信息为：

```yaml
contentKey:
  contentValue
```



在`bootstrap.yml`配置文件里添加如下配置实现`共享配置`和`内容中心独有的配置`：

```yaml
server:
  port: 8082

spring:
  application:
    # 向Nacos注册服务的时候会使用这个名字
    name: content-center
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: 7cf9b1f3-3c42-4e4a-91e9-4d1b68a78a98
        file-extension: yaml
        shared-configs[0]:
          data-id: common.yaml
          group: common
          refresh: true
        shared-configs[1]:
          data-id: spring-cloud-common.yaml
          group: spring-cloud-common
          refresh: true
        extension-configs[0]:
          data-id: content-center.yaml
          group: content-center
          refresh: true


```



可以看到，主要是使用了`shared-configs`来实现为`内容中心`引入两个共享配置，用`extension-configs`为`内容中心`引入了`内容中心`独有的配置。



编写相关测试方法：

```java
package tech.punklu.contentcenter.controller.content.nacos;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 不光需要在配置文件中自动刷新配置：refresh-enabled: true，
 * 还需要在使用到Nacos配置信息的类上添加@RefreshScope注解才能实现自动更新
 */
@RestController
@RequestMapping("/nacos")
@RefreshScope
public class NacosController {

//    @Value("${nacosKey}")
//    private String nacosKey;
//
//    @GetMapping(value = "/getNacosValue",produces = MediaType.APPLICATION_JSON_VALUE)
//    public String getNacosValue(){
//        return nacosKey;
//    }

    @Value("${commonKey}")
    private String commonKey;
    @GetMapping(value = "/getCommonValue",produces = MediaType.APPLICATION_JSON_VALUE)
    public String getCommonValue(){
        return commonKey;
    }

    @Value("${springCloudCommonKey}")
    private String springCloudCommonKey;
    @GetMapping(value = "/getSpringCloudCommonValue",produces = MediaType.APPLICATION_JSON_VALUE)
    public String getSpringCloudCommonValue(){
        return springCloudCommonKey;
    }

    @Value("${contentKey}")
    private String contentKey;
    @GetMapping(value = "/getContentValue",produces = MediaType.APPLICATION_JSON_VALUE)
    public String getContentValue(){
        return contentKey;
    }


}

```



启动`内容中心`后，访问定义的这几个方法，可以看到都已经访问到了这三个`Nacos`上不同配置文件里配置，实现了`配置共享`。且在`Nacos`控制台修改其中的值后，重新访问，也可获取到最新的值，实现了不用重新打包在线更新配置项的功能。



## 使用Nacos配置管理注意事项

使用`Nacos`进行配置管理有几个需要注意的地方：

1. 不能做到所有配置都在线更新

   一些只在项目启动时使用的配置项，如`JDBC连接信息`等内容在项目启动后，在`Nacos`控制台里修改是没有效果的。

2. Spring Boot使用Nacos的方式和Spring Cloud是不同的

   Spring Boot和Spring Cloud使用`Nacos`的方式有点区别，后续找机会整理吧。





## Nacos持久化

如果要将Nacos用于生产环境，必须对Nacos进行持久化，使其配置信息保存到数据库中。



Nacos使用的是内嵌数据库 `Derby`(Apache Derby)，目前Nacos仅支持`Mysql`数据库，且版本要求：`5.6.5+`。



新建数据库`nacos`，导入Nacos安装目录下的`conf/nacos-mysql.sql`的sql文件。

可以看到数据库增加了11张表。



修改`Nacos`配置文件，打开`conf/application.properties`，修改数据库配置：

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/mynacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=123456
```



登录Nacos，导出原来的配置文件。

修改数据库配置之后，原配置自然就不在了，所以提前做好备份，后面可以直接导入。每个命名空间都要单独导出。



重新启动Nacos，登录后发现配置全没了。

按照原来的命名空间，重新创建。

然后，在对应的命名空间下导入原配置文件即可。
---
title: "Spring Cloud Alibaba实践——使用Feign" 
date: 2021-01-29T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——使用Feign

## Feign的使用

之前内容中心调用用户中心的接口是直接通过`RestTemplate`的方式调用的，但是当接口数量变多，接口需要的参数过多时，这种方式会变得很难维护，因此需要引入`Feign`组件来完成服务间的接口调用。



### 为内容中心引入Feign



首先引入依赖，因为Feign是属于Spring Cloud的组件而不是Spring Cloud Alibaba的组件，所以需要额外引入`Spring Cloud`的依赖项：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <!--整合spring cloud-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Greenwich.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
<dependencyManagement>
```



然后在Spring Boot的启动类上添加上Feign的注解：`@EnableFeignClients`。



再开发内容中心调用用户中心的客户端代理类：

```java
package tech.punklu.contentcenter.feignclient;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import tech.punklu.contentcenter.domain.dto.user.UserDTO;

/**
 * 调用用户中心的Feign客户端代理类
 */
@FeignClient(name = "user-center")
public interface UserCenterFeignClient {

    @GetMapping("/users/{id}")
    UserDTO findById(@PathVariable Integer id);
}

```



可以看到这里用到的都是`Spring MVC`的注解，所以最后生成的`URL`会是：`http://user-center/users/{id}`，和之前直接写死在`ShareService`中的如下这行代码中的`URL`一致：

```java
UserDTO userDTO = restTemplate.getForObject("http://user-center/users/{userId}",UserDTO.class,userId);
```



将ShareService中的这行代码替换为调用刚写好的调用用户中心的客户端,，方法最终代码如下所示：

```java
@Autowired
private UserCenterFeignClient userCenterFeignClient;

/**
* 根据内容id获取分享内容的详情
* @param id
* @return
*/
public ShareDTO findById(Integer id){
    // 获取分享详情
    Share share = this.shareMapper.selectByPrimaryKey(id);
    // 获取发布人id
    Integer userId = share.getUserId();

    // 从ribbon负载均衡中心获得用户中心的地址

    UserDTO userDTO = this.userCenterFeignClient.findById(userId);
    // 消息装备
    ShareDTO shareDTO = new ShareDTO();
    BeanUtils.copyProperties(share,shareDTO);
    shareDTO.setWxNickname(userDTO.getWxNickname());
    return shareDTO;
}
```



启动项目，访问`http://127.0.0.1:8082/shares/1`，可以发现依然可以访问，且当具有多个用户中心实例时，依然可以支持之前的Ribbon相关的功能。



### Feign的组成

| 接口               | 作用                                   | 默认值                                                       |
| ------------------ | -------------------------------------- | ------------------------------------------------------------ |
| Feign.Builder      | Feign的入口                            | Feign.Builder                                                |
| Client             | Feign底层用什么去请求                  | 和Ribbon配合时：<br />LoadBalancerFeignClient<br />不会和Ribbon配合时feign.Client.Default |
| Contract           | 契约，注解支持                         | SpringMvcContract                                            |
| Encoder            | 编码器，用于将对象转换成HTTP请求消息体 | SpringEncoder                                                |
| Decoder            | 解码器，将响应消息体转换成对象         | ResponseEntityDecoder                                        |
| Logger             | 日志管理器                             | Slf4jLogger                                                  |
| RequestInterceptor | 用于为每个请求添加通用逻辑             | 无                                                           |

需要注意的是`SpringMvcContract`，Feign默认是不使用`Spring MVC`形式的写法的，为了方便使用，通过`SpringMvcContract`来扩展为`Spring MVC`形式的写法。



### 代码方式细粒度配置Feign日志级别

Feign默认是不打印日志的，而且Feign自定义了四种不同的日志级别，需要单独设置才能开启Feign日志的打印。



| 级别       | 打印内容                                      |
| ---------- | --------------------------------------------- |
| NONE(默认) | 不记录任何日志                                |
| BASIC      | 仅记录请求方法、URL、响应状态代码以及执行时间 |
| HEADERS    | 记录BASIC级别的基础上，记录请求和响应的header |
| FULL       | 记录请求和响应的header、body和元数据          |



首先添加配置类：

```java
package tech.punklu.contentcenter.configuration;

import feign.Logger;
import org.springframework.context.annotation.Bean;

/**
 * 配置Feign的日志级别
 */
public class UserCenterFeignConfiguration {

    @Bean
    public Logger.Level level(){
        // 让Feign打印所有请求的细节
        return Logger.Level.FULL;
    }
}

```



**要注意的是，不能加@Configuration注解（原因同Ribbon配置类），否则会被当成Feign的全局日志级别配置，如果加上@Configuration注解又不想被当成全局配置，需要将此类移到Spring Boot启动类所在的目录之外的地方，避免被Spring Boot扫描到导致父子上下文问题。**



然后，在`Feign Cient`类上的`Feign`注解里加上配置属性：

```java
package tech.punklu.contentcenter.feignclient;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import tech.punklu.contentcenter.configuration.UserCenterFeignConfiguration;
import tech.punklu.contentcenter.domain.dto.user.UserDTO;

/**
 * 调用用户中心的Feign客户端代理类
 */
@FeignClient(name = "user-center",configuration = UserCenterFeignConfiguration.class)
public interface UserCenterFeignClient {

    @GetMapping("/users/{id}")
    UserDTO findById(@PathVariable Integer id);
}

```



最后，还需要将Client的日志设置为Debug并附加到`Spring Boot`本身的日志里去：

```yaml
logging:
  level:
    tech.punklu.contentcenter.feignclient.UserCenterFeignClient: debug
```

启动内容中心服务，访问`http://127.0.0.1:8082/shares/1`可以发现内容中心服务的日志里已经有了Feign调用日志的信息了。



### 属性方式细粒度配置Feign日志级别

除了代码方式，也可通过配置文件属性来配置Feign的日志级别：

```yaml
logging:
  level:
    tech.punklu.contentcenter.feignclient.UserCenterFeignClient: debug
feign:
  client:
    config:
      # 想要调用的微服务的名称
      user-center:
        loggerLevel: full

```

**虽然是通过属性配置了，但是将Client的日志设置为Debug并附加到`Spring Boot`本身的日志里去的这段配置依然必须要有。**



将代码方式的配置注解注释掉：

```java
package tech.punklu.contentcenter.feignclient;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import tech.punklu.contentcenter.configuration.UserCenterFeignConfiguration;
import tech.punklu.contentcenter.domain.dto.user.UserDTO;

/**
 * 调用用户中心的Feign客户端代理类
 */
//@FeignClient(name = "user-center",configuration = UserCenterFeignConfiguration.class)
@FeignClient(name = "user-center")
public interface UserCenterFeignClient {

    @GetMapping("/users/{id}")
    UserDTO findById(@PathVariable Integer id);
}

```



重启内容中心服务，访问`http://127.0.0.1:8082/shares/1`可以发现内容中心服务的日志里依然有Feign调用日志的信息了。



### 代码方式全局配置Feign日志级别

将之前内容中心的这段细粒度配置用户中心日志级别的配置注释掉：

```yaml
#feign:
#  client:
#    config:
#      # 想要调用的微服务的名称
#      user-center:
#        loggerLevel: full
```

然后在`Spring Boot`启动类上的Feign注解`@EnableFeignClients`添加属性配置默认全局Feign配置信息：

```java
package tech.punklu.contentcenter;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;
import tech.punklu.contentcenter.configuration.UserCenterFeignConfiguration;
import tk.mybatis.spring.annotation.MapperScan;

@MapperScan("tech.punklu.contentcenter.dao")
@SpringBootApplication
@EnableFeignClients(defaultConfiguration = UserCenterFeignConfiguration.class)
public class ContentCenterApplication {

    public static void main(String[] args) {
        SpringApplication.run(ContentCenterApplication.class, args);
    }

}

```

启动后，访问`http://127.0.0.1:8082/shares/1`，发现Feign的日志级别依然是Full。





### 属性方式全局配置Feign日志级别

将上面的`Spring Boot`启动类上的Feign注解`@EnableFeignClients`的属性注释掉：

```java
package tech.punklu.contentcenter;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;
import tech.punklu.contentcenter.configuration.UserCenterFeignConfiguration;
import tk.mybatis.spring.annotation.MapperScan;

@MapperScan("tech.punklu.contentcenter.dao")
@SpringBootApplication
@EnableFeignClients//(defaultConfiguration = UserCenterFeignConfiguration.class)
public class ContentCenterApplication {

    public static void main(String[] args) {
        SpringApplication.run(ContentCenterApplication.class, args);
    }


    /**
     * 在Spring容器中，创建一个对象，类型是RestTemplate，
     * 名称/id是方法名
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}

```



然后添加配置:

```yaml
feign:
  client:
    config:
      # 想要调用的微服务的名称
      default:
        loggerLevel: full

```

可以发现，和细粒度属性配置的区别在于将服务的名称换成了default就成了默认的全局配置。



启动项目，访问`http://127.0.0.1:8082/shares/1`，可以看到Feign的日志级别依然是`Full`。





### Feign属性配置支持的配置项

```yaml
feign.client.config:
	<feignName>:
		connectTimeout: 5000 #连接超时时间
		readTimeout: 5000 #读取超时时间
		loggerLevel: full #日志级别
		errorDecoder: com.example.SimpleErrorDecoder #错误解码器
		retryer: com.example.SimpleRetryer 			# 重试策略
		requestInterceptors:
			- com.example.FooRequestInterceptor		# 拦截器
		# 是否对404错误码解码
		# 处理逻辑详见feign.SynchronousMethodHandler #executeAndDecode
		decode404: false
		encoder: com.example.SimpleEncoder #解码器
		decoder: com.example.SimpleDecoder #解码器
		contract: com.example.SimpleContract #契约
```



### Feign两种配置方式的区别

| 配置方式 | 优点                                               | 缺点                                                         |
| -------- | -------------------------------------------------- | ------------------------------------------------------------ |
| 代码配置 | 基于代码，更加灵活                                 | 如果Feign的配置类加了Configuration注解，需要注意父子上下文，线上修改得重新打包发布 |
| 属性配置 | 易上手、配置直观、线上修改无需打包发布、优先级更高 | 极端场景下没有代码配置方式灵活                               |



优先级：全局代码 < 全局属性 < 细粒度代码 < 细粒度属性配置



应优先使用属性配置，且应尽量使用一种配置方式，避免不必要的优先级问题。



### Feign多参数构造请求

对于Get请求，Feign可以使用：

1. 在类参数上添加@SpringQueryMap注解
2. 使用@RequestParam注解
3. 使用Map来构建参数



以第一种方式为例，在用户中心中新增方法：

```java
@GetMapping("/testFeignParam")
public User testFeignParam(User user){
	return user;
}
```



在内容中心中新增对该方法的Feign代理方法：

```java
@GetMapping("testFeignParam")
UserDTO testFeignParam(@SpringQueryMap UserDTO userDTO);
```

即可实现多参数请求构造。



对于Post请求，直接使用Spring MVC中自带的`@RequestBody`注解即可。



### Feign脱离Ribbon使用

之前的`Feign`调用都是通过`Ribbon`调用注册在`Nacos`上的实例的接口，有时需要脱离`Ribbon`直接调用没注册在`Nacos`上的服务下的接口。比如，直接调用百度的网址。



新建`Feign`代理类：

```java
package tech.punklu.contentcenter.feignclient;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "baidu",url = "http://www.baidu.com")
public interface TestBaiduFeignClient {

    @GetMapping("")
    String index();
}

```



添加`Controller`接口：

```java
@Autowired
private TestBaiduFeignClient testBaiduFeignClient;

@GetMapping("/baidu")
public String baidu(){
	return this.testBaiduFeignClient.index();
}
```



启动服务后，访问`http://127.0.0.1:8082/baidu`，可以看到已经访问到了百度的首页。



需要注意的是，虽然通过`FeignClient`注解里的`url`属性添加了百度的地址，但是`name`属性依然必须要有，否则会导致项目无法启动。



### Feign性能优化

`Feign`的性能比`RestTemplate`的性能要差很多，主要有两个原因：

1. 默认使用UrlConnection，而不是连接池
2. 日志级别过高导致的性能损失



对于第二个原因，将Feign的日志级别设置为`BASIC`即可，对于第一个原因，可以通过配置连接池的方式解决：

首先添加依赖：

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
</dependency>
```



然后添加配置项：

```yaml
feign:
  httpclient:
    # 让feign使用apache httpclient做请求；而不是默认的urlconnection，提高性能
    enabled: true
    # feign的最大连接数
    max-connections: 200
    # feign单个路径的最大连接数
    max-connections-per-route: 50
```


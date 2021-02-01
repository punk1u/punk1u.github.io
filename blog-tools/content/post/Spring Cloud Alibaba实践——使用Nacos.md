---
title: "Spring Cloud Alibaba实践——使用Nacos" 
date: 2021-01-25T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——使用Nacos

Nacos 致力于发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，快速实现动态服务发现、服务配置、服务元数据及流量管理。



## 使用Nacos实现服务发现



### 搭建Nacos Server

从`https://github.com/alibaba/nacos/releases`下载Nacos Server的启动包，



解压缩后，进入`bin`目录，执行`.\startup.cmd -m standalone`命令即可启动Nacos Server。



访问`localhost:8848/nacos`即可进入`Nacos Server`控制台，默认用户名密码都是`Nacos`。



### 将服务注册到Nacos

为用户中心服务添加Nacos服务发现依赖配置：

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
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
```



添加配置项：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        # 指定nacos server的地址
        server-addr: localhost:8848
```



启动用户中心服务后，可以在Nacos控制台上的服务列表里看到用户中心已经被注册到Nacos中了。



使用同样的操作，将内容中心也注册到Nacos中。



### 验证服务发现

在内容中心里新建如下测试类：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class TestController {

    /**
     * DiscoveryClient对于Nacos、Zookeeper、eureka都适用
     */
    @Autowired
    private DiscoveryClient discoveryClient;

    /**
     * 测试服务发现，证明内容中心可以找到用户中心
     * @return 用户中心所有实例的地址信息
     */
    @GetMapping("/getInstances")
    public List<ServiceInstance> getInstances(){
        // 查询指定服务的所有实例的信息
        return this.discoveryClient.getInstances("user-center");
    }
}

```



启动内容中心后，访问`http://127.0.0.1:8082/getInstances`，可以看到如下的返回信息（用户中心的实例信息）：

```json
[
    {
        "serviceId": "user-center",
        "host": "192.168.9.223",
        "port": 8081,
        "secure": false,
        "metadata": {
            "nacos.instanceId": "192.168.9.223#8081#DEFAULT#DEFAULT_GROUP@@user-center",
            "nacos.weight": "1.0",
            "nacos.cluster": "DEFAULT",
            "nacos.healthy": "true",
            "preserved.register.source": "SPRING_CLOUD"
        },
        "uri": "http://192.168.9.223:8081",
        "scheme": null,
        "instanceId": null
    }
]
```

说明已实现服务注册发现。



在idea中，打开`Edit Configurations`，选择UserCenterApplication，复制一份出来，并使用`-Dserver.port=8083`的VM参数再次启动一个UserCenterApplication的实例，重新访问`http://127.0.0.1:8082/getInstances`，可以发现，此时获取到的用户中心实例已经变成了两个。



### 为内容中心引入Nacos

之前的Spring Boot项目雏形中，内容中心调用用户中心时，是通过写死的用户中心接口的URL调用的。如下:

```java
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
    // 调用用户微服务查询对应的用户信息
    UserDTO userDTO = restTemplate.getForObject("http://localhost:8081/users/{id}",UserDTO.class,userId);
    // 消息装备
    ShareDTO shareDTO = new ShareDTO();
    BeanUtils.copyProperties(share,shareDTO);
    shareDTO.setWxNickname(userDTO.getWxNickname());
    return shareDTO;
}
```



但是这种方式，当用户中心的地址发生了变化后就会失效，为了实现这种硬编码的解绑，可以通过从Nacos中获取已注册的服务信息及接口URL。



如下所示：

```java
@Autowired
private DiscoveryClient discoveryClient;

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
    // 获取用户中心所有实例的信息
    List<ServiceInstance> instances = discoveryClient.getInstances("user-center");
    String targetUrl = instances.stream().map(instance->instance.getUri().toString() + "/users/{id}").findFirst().orElseThrow(()->new IllegalArgumentException("当前没有用户中心实例"));
    log.info("请求的目标地址:{}",targetUrl);
    // 调用用户微服务查询对应的用户信息
    UserDTO userDTO = restTemplate.getForObject(targetUrl,UserDTO.class,userId);
    // 消息装备
    ShareDTO shareDTO = new ShareDTO();
    BeanUtils.copyProperties(share,shareDTO);
    shareDTO.setWxNickname(userDTO.getWxNickname());
    return shareDTO;
}
```


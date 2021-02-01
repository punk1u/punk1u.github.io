---
title: "Spring Cloud Alibaba实践——使用Ribbon" 
date: 2021-01-27T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——使用Ribbon



## 使用Ribbon实现负载均衡



### 为内容中心引入Ribbon

因为Nacos的服务发现组件`spring-cloud-starter-alibaba-nacos-discovery`内已经包含了`Ribbon`，所以不需要再另外添加Ribbon的依赖。



在内容中心的启动类里声明`RestTemplate`bean对象的方法上添加`@LoadBalanced`注解，即可实现`RestTemplate`与`Ribbon`负载均衡的整合，从而实现内容中心调用用户中心时负载均衡的效果。



```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;
import tk.mybatis.spring.annotation.MapperScan;

@MapperScan("tech.punklu.contentcenter.dao")
@SpringBootApplication
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

启用Ribbon不需要额外的配置信息。



将下面这段内容中心中的ShareService中的查找内容的代码

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

<span id = "RibbonUseCode">>替换为:</span

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

    // 从ribbon负载均衡中心获得用户中心的地址

    UserDTO userDTO = restTemplate.getForObject("http://user-center/users/{userId}",UserDTO.class,userId);
    // 消息装备
    ShareDTO shareDTO = new ShareDTO();
    BeanUtils.copyProperties(share,shareDTO);
    shareDTO.setWxNickname(userDTO.getWxNickname());
    return shareDTO;
}
```

这样，当RestTemplate发起请求的时候，Ribbon会自动把`http://user-center/users/{userId}`中的`user-center`替换为用户中心在`Nacos`上的地址并进行负载均衡。



启动两个用户中心服务，并启动内容中心服务，访问内容中心服务的`http://127.0.0.1:8082/shares/1`接口，可以发现，Ribbon已实现了自动寻找用户中心地址并负载均衡地访问的功能。



### <span id = "RibbonTool">Ribbon组成</span>

Ribbon内部接口及其作用：

| 接口                     | 作用                       | 默认值                                                       |
| ------------------------ | -------------------------- | ------------------------------------------------------------ |
| IClientConfig            | 读取配置                   | DefaultClientConfigImpl                                      |
| IRule                    | 负载均衡规则，选择实例     | ZoneAvoidanceRule                                            |
| IPing                    | 筛选掉ping不通的实例       | DummyPing                                                    |
| ServerList<Server>       | 交给Ribbon的实例列表       | Ribbon：ConfigurationBasedServerList<br />Spring Cloud Alibaba：NacosServerList |
| ServerListFilter<Server> | 过滤掉不符合条件的实例     | ZonePreferenceServerListFilter                               |
| ILoadBalancer            | Ribbon的入口               | ZoneAwareLoadBalancer                                        |
| ServerListUpdater        | 更新交给Ribbon的List的策略 | PollingServerListUpdater                                     |



如果对Ribbon的默认处理策略不满意，可以自己实现对应的接口并实现相应的逻辑。




### <span id = "RibbonBalanceRule">Ribbon负载均衡规则</span>

| 规则名称                  | 特点                                                         |
| ------------------------- | ------------------------------------------------------------ |
| AvailabilityFilteringRule | 过滤掉一直连接失败的被标记为circuit tripped的后端Server，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤server的逻辑，其实就是检查status里记录的各个Server的运行状态 |
| BestAvailableRule         | 选择一个最小的并发请求的Server，逐个考察Server，如果Server被tripped了，则跳过 |
| RandomRule                | 随机选择一个Server                                           |
| ResponseTimeWeightedRule  | 已废弃，作用同WeightedResponseTimeRule                       |
| RetryRule                 | 对选定的负载均衡策略机上重试机制，在一个配置时间段内当选择Server不成功，则一直尝试使用subRule的方式选择一个可用的Server |
| RoundRobinRule            | 轮询选择，轮询index，选择index对应位置的Server               |
| WeightedResponseTimeRule  | 根据响应时间加权，响应时间越长，权重越小，被选中的可能性越低 |
| ZoneAvoidanceRule         | 复合判断Server所Zone的性能和Server的可用性选择Server，在没有Zone的环境下，类似于轮询（RoundRobinRule） |



默认负载均衡规则是`ZoneAvoidanceRule`。



### Java代码配置Ribbon负载均衡规则

Ribbon可以实现细粒度的负载均衡规则配置，比如A服务需要调用B服务和C服务的接口，可以为A调用B服务的接口配置一个负载均衡规则，再为A调用C服务的接口配置另一个不同的负载均衡规则。



以内容中心调用用户中心为例，在内容中心中增加如下配置类：

```java
package tech.punklu.contentcenter.configuration;

import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Configuration;
import ribbonconfiguration.RibbonConfiguration;

@Configuration
@RibbonClient(name = "user-center",configuration = RibbonConfiguration.class)
public class UserCenterRibbonConfiguration {
}

```

可以看到，在RibbonClient这个注解中，指定了使用这个规则的被调用服务。以及要使用的规则为`自定义的`RibbonConfiguration配置类里指定的规则。



自定义RibbonConfiguration：

```java
package ribbonconfiguration;

import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.RandomRule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RibbonConfiguration {

    @Bean
    public IRule ribbonRule(){
        return new RandomRule();
    }
}
```



***需要特别注意的是，这里的自定义负载均衡规则类RibbonConfiguration不能在Spring Boot启动类所在的包下面，而要在和Spring Boot启动类所在的包平级的包下面创建。层级关系如下：***

```
tech.punklu.contentcenter.configuration
	ContentCenterApplication (Spring Boot启动类)
ribbonconfiguration
	RibbonConfiguration (自定义Ribbon负载均衡规则类)
```



***原因在于Spring的父子上下文问题，如果将自定义的Ribbon负载均衡规则放到了和Spring Boot启动类同层级的目录下，就会被Spring Boot扫描到自定义类上的`@Configuration`注解，从而导致最后一个被扫描到的自定义Ribbon规则类被当成全局自定义负载均衡规则。因此，为了实现细粒度的负载均衡规则，必须确保自定义的Ribbon负载均衡规则类处于独立的子上下文中而不是Spring默认的父上下文中。***



启动两个用户中心，并在用户中心查询用户的接口中添加被调用时的日志信息，再多次访问`http://127.0.0.1:8082/shares/1`，可以发现，此时已经实现了自定义规则中的随机负载均衡规则，而不是Ribbon默认的`ZoneAvoidanceRule`规则。



### <span id = "RibbonPropConfig">属性配置Ribbon负载均衡规则</span>

除了上面的通过代码方式配置Ribbon负载均衡规则的方法，还可以通过属性配置的方式指定内容中心访问用户中心的负载均衡规则。如下所示：

```yaml
user-center:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

在内容中心配置类`application.yml`中添加上述配置后，去掉代码方式的配置类注解，如下所示：

```java
/*
@Configuration
@RibbonClient(name = "user-center",configuration = RibbonConfiguration.class)*/
public class UserCenterRibbonConfiguration {
}
```

再次重新启动两个用户中心服务和内容中心服务，可以发现依然实现了和代码配置负载均衡规则一样的随机访问用户中心的效果。



### 代码配置和属性配置的区别

| 配置方式 | 优点                                                         | 缺点                                                         |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 代码配置 | 基于代码，更加灵活                                           | 有小坑（父子上下文）<br />线上修改需要重新打包发布           |
| 属性配置 | 易上手<br />配置更加直观<br />线上修改无需重新打包、发布<br />**优先级比代码配置更高** | 极端场景下没有代码配置方式灵活（但基本上属性配置都可满足要求） |



应该尽量使用属性配置，属性配置实现不了的情况再考虑代码配置。



### 设置Ribbon全局负载均衡规则

如果不需要细粒度的负载均衡规则配置，只需要将项目配置里的细粒度配置去除掉，然后将 ：

```java
@Configuration
@RibbonClient(name = "user-center",configuration = RibbonConfiguration.class)*/
public class UserCenterRibbonConfiguration {
}
```

这个ribbon负载规则配置类中的：

1. 将@RibbonClient注解替换成@RibbonClients注解
2. 将注解中的configuration属性设置为自定义的负载均衡规则类RibbonConfiguration即可



如下所示：

```java
@Configuration
@RibbonClients(defaultConfiguration = RibbonConfiguration.class)
public class UserCenterRibbonConfiguration {
}
```





### Ribbon支持的配置项

上面只配置了Ribbon的负载均衡规则，实际上，上方的[Ribbon组成](#RibbonTool)所展示的Ribbon内部接口，都可以按照配置负载均衡规则方式进行配置，比如，可以配置`Ping`的方式为`PingUrl`，与配置负载均衡规则一样，也要避免和Spring Boot的主上下文放在一起，在配置负载均衡规则的代码下面跟着继续配置即可：

```java
package ribbonconfiguration;

import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.PingUrl;
import com.netflix.loadbalancer.RandomRule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RibbonConfiguration {

    /**
     * 自定义内容中心调用用户中心时的Ribbon负载均衡规则为随机
     * @return
     */
    @Bean
    public IRule ribbonRule(){
        return new RandomRule();
    }

    /**
     * 自定义Ping的方式
     */
    @Bean
    public IPing ping(){
        return new PingUrl();
    }
}

```



也可以使用项目配置文件的属性完成所有的Ribbon配置，使用方式如下：

`<clientName>.ribbon.如下属性：`

- `NFLoadBalancerClassName`：`ILoadBalancer`实现类
- `NFLoadBalancerRuleClassName`：`IRule`实现类
- `NFLoadBalancerPingClassName`：`IPing`实现类
- `NIWSServerListClassName`：`ServerList`实现类
- `NIWSServerListFilterClassName`：`ServerListFilter`实现类



如同上面的用项目配置文件配置负载均衡规则：[属性配置Ribbon负载均衡规则](#RibbonPropConfig)



### Ribbon延迟加载问题

在上面的这段[Ribbon使用代码](#RibbonUseCode)中，下面这行代码中的`user-center`会被Ribbon自动替换为一个用户中心的实例地址:

```java
UserDTO userDTO = restTemplate.getForObject("http://user-center/users/{userId}",UserDTO.class,userId);
```

但是Ribbon的一个默认配置是延迟加载，也就是说，在上面这行代码第一次被执行到，即内容中心第一次调用用户中心时会被比较慢。为了解决这个问题，可以在内容中心的配置文件中指定加载策略为`饥饿加载`，即可显著提高第一次跨服务调用时的速度：

```yaml
# 配置Ribbon加载策略为饥饿加载，提高第一次调用用户中心的速度(默认为延迟加载)，
# clients配置项如果有多个，可以使用逗号分割
ribbon:
  eager-load:
    clients: user-center
```

如果要指定多个其他服务被内容中心调用时均为饥饿加载，可以在clients属性中使用逗号分隔。



### 用Nacos为Ribbon配置服务权重

在Nacos的控制台可以为服务的每个实例配置权重，权重越大，被访问到的几率越大。可以为不同配置的机器配置不同的权重实现性能高的机器访问次数越多的功能。



**但是，Ribbon内置的[负载均衡规则](#RibbonBalanceRule)都不支持Nacos的权重，所以需要自己进行二次开发扩展实现自己的支持Nacos权重的Ribbon负载均衡规则。**



首先定义自己的负载均衡规则类：

```java
package tech.punklu.contentcenter.configuration;

import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import com.alibaba.cloud.nacos.ribbon.NacosServer;
import com.alibaba.nacos.api.exception.NacosException;
import com.alibaba.nacos.api.naming.NamingService;
import com.alibaba.nacos.api.naming.pojo.Instance;
import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.BaseLoadBalancer;
import com.netflix.loadbalancer.Server;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;

/**
 * 自定义的支持Nacos权重配置的Ribbon负载均衡规则类
 */
@Slf4j
public class NacosWeightedRule  extends AbstractLoadBalancerRule {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
        // 可用来读取配置文件，并初始化NacosWeightedRule
    }

    @Override
    public Server choose(Object o) {
        BaseLoadBalancer loadBalancer = (BaseLoadBalancer) this.getLoadBalancer();
        // 获取想要请求的微服务的名称
        String name = loadBalancer.getName();
        // 拿到服务发现的相关API
        NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();
        // nacos client自动通过基于权重的负载均衡算法，选择出一个可用的实例
        try {
            Instance instance = namingService.selectOneHealthyInstance(name);
            log.info("选择的实例是: port = {}, instance = {}",instance.getPort(),instance);
            return new NacosServer(instance);
        } catch (NacosException e) {
            e.printStackTrace();
        }
        return null;
    }
}

```



主要逻辑就是调用Nacos提供的接口，由Nacos根据要调用的服务名及服务下的各个实例的不同权重选择出一个可用的实例。



然后再将这个类作为负载均衡规则类添加到Ribbon的配置类中去，调换掉原有的Ribbon定义好的规则类：

```java
package ribbonconfiguration;

import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.PingUrl;
import com.netflix.loadbalancer.RandomRule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import tech.punklu.contentcenter.configuration.NacosWeightedRule;


@Configuration
public class RibbonConfiguration {

    /**
     * 自定义内容中心调用用户中心时的Ribbon负载均衡规则为随机
     * @return
     */
    @Bean
    public IRule ribbonRule(){
        return new NacosWeightedRule();
    }
}
```





启动两个用户中心实例和内容中心实例，在Nacos上，将两个用户中心实例的权重一个置为0，一个置为1，多次访问`http://127.0.0.1:8082/shares/1`，可以发现此时只会访问到权重不为0的那个用户中心实例上，将两个全部设为1后，两个实例都可访问到，即实现了动态配置Nacos实例权重并负载均衡的功能。



Nacos Client内置了基于权重的负载均衡算法，之所以还要整合Ribbon，是因为为了符合`Spring Cloud`的标准，`Spring Cloud`有一个子项目`Spring Cloud Commons`，定义了`Spring Cloud`的一些标准，`Spring Cloud Commons`下面又有一个子项目`Spring Cloud LoadBalancer`，定义了负载均衡的一些标准，但是并没有根据权重负载均衡的概念，所以`Spring Cloud Alibaba`遵循了这个标准，整合了Ribbon，因为Ribbon是符合这个标准的，并且是这个标准唯一的实现。



### 使Ribbon优先调用同一集群

为了容灾，会在多个地方的机房部署项目。但为了提升性能，会希望实现同机房优先调用，如果同机房找不到可用的实例，再跨机房调用。使用Nacos的Cluster可以很方便地实现。



首先自定义优先调用同集群实例的负载均衡规则类：

```java
package tech.punklu.contentcenter.configuration;

import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import com.alibaba.cloud.nacos.ribbon.NacosServer;
import com.alibaba.nacos.api.exception.NacosException;
import com.alibaba.nacos.api.naming.NamingService;
import com.alibaba.nacos.api.naming.pojo.Instance;
import com.alibaba.nacos.client.naming.core.Balancer;
import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.BaseLoadBalancer;
import com.netflix.loadbalancer.Server;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.util.CollectionUtils;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

@Slf4j
public class NacosSameClusterWeightedRule extends AbstractLoadBalancerRule {


    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

    @Override
    public Server choose(Object o) {
        // 从配置文件获取当前服务(内容中心)所在的机房(集群)名称
        String clusterName = nacosDiscoveryProperties.getClusterName();

        BaseLoadBalancer loadBalancer = (BaseLoadBalancer) this.getLoadBalancer();
        // 获取想要请求的微服务的名称
        String name = loadBalancer.getName();
        // 拿到服务发现的相关API
        NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();


        try {
            // 找到指定服务的所有实例 A，true表示只查找健康的可用的实例
            List<Instance> instances = namingService.selectInstances(name, true);
            // 过滤出相同集群下的所有实例 B
            List<Instance> sameClusterInstances =
                    instances.stream().filter(instance -> Objects.equals(instance.getClusterName(), clusterName)).collect(Collectors.toList());
            // 如果B是空，就用A
            List<Instance> instancesToBeChosen = new ArrayList<>();
            if (CollectionUtils.isEmpty(sameClusterInstances)){
                instancesToBeChosen = instances;
                log.warn("发生跨集群的调用,name= {}, clusterName = {}, instances = {}",
                        name,clusterName,instances);
            }else {
                instancesToBeChosen = sameClusterInstances;
            }
            // 基于权重的负载均衡算法，返回一个实例，需要使用Nacos提供的接口二次封装调用，达到从指定的实例中根据权重选择一个可用的实例的效果
            Instance instance = ExtendBalancer.getHostByRandomWeight2(instancesToBeChosen);
            log.info("选择的实例是 port = {},instance = {}",instance.getPort(),instance);
            return new NacosServer(instance);
        } catch (NacosException e) {
            log.error("发生异常了",e);
            return null;
        }
    }


}
class ExtendBalancer extends Balancer{
    public static Instance getHostByRandomWeight2(List<Instance> hosts){
        return getHostByRandomWeight(hosts);
    }
}

```



然后将该负载均衡类设置为Ribbon的默认负载均衡规则类：

```java
package tech.punklu.contentcenter.configuration;

import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.cloud.netflix.ribbon.RibbonClients;
import org.springframework.context.annotation.Configuration;
import ribbonconfiguration.RibbonConfiguration;


@Configuration
@RibbonClient(name = "user-center",configuration = NacosSameClusterWeightedRule.class)
public class UserCenterRibbonConfiguration {
}

```



将内容中心的配置文件中的Nacos cluster-name改为BJ（北京）:

```yaml
spring:
  cloud:
    nacos:
      discovery:
        # 指定nacos server的地址
        server-addr: localhost:8848
        cluster-name: BJ
```



然后启动一个cluster-name为BJ的用户中心实例，一个cluster-name为NJ的用户中心实例，多次访问`http://127.0.0.1:8082/shares/1`，可以发现全都访问到了cluster-name为BJ的用户中心实例上了，说明已经实现了访问同集群下服务的功能。


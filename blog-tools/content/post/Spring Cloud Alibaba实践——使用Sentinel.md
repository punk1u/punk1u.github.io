---
title: "Spring Cloud Alibaba实践——使用Sentinel" 
date: 2021-01-31T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——使用Sentinel

## 使用Sentinel实现服务容错

当微服务调用链底部的一个被众多微服务实例调用的服务不可用时，很容易导致这个底层服务所在的众多链路一起不能正常提供服务。在这种情况下，为了避免雪崩效应，就需要实现服务容错功能。



微服务常见容错方案：

1. 超时

   通过超时来释放资源，这样就不容易被拖死，只要释放够快。

2. 限流

   通过评估来限制流量，防止微服务被拖死。

3. 仓壁模式

   资源有对立线程池，拥有自己拒绝策略。资源之间不相互影响。

4. 断路器模式

   监控错误率或者错误次数达到一定阈值，就跳闸，就认为依赖微服务不可用，监控加开关



其中，断路器方案可以很巧妙地通过三状态来实现服务的自动保护和自动恢复：

1. 断路器关闭

   此时接口可正常使用

2. 断路器打开

   当一段时间内服务调用失败的次数达到了阈值，触发断路器，后续的调用直接失败

3. 断路器半开

   断路器打开后不是一直处在打开状态，会在后续的某个时间点尝试执行一个请求，如果请求成功，说明此时下游服务已可用，断路器关闭，否则继续这个过程直到服务可用。通过这种机制，既保护了服务不会因为下游服务的失败而被拖垮，又实现了服务的自我修复。



### 搭建Sentinel控制台

 在`https://github.com/alibaba/Sentinel/releases`下载`sentinel`的`jar`包，然后使用`java -jar sentinel-dashboard-1.8.0.jar`命令运行即可。



### 用Sentinel为内容中心实现服务容错

首先为内容中心引入依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```



添加Sentinel相关配置信息：

```yaml
spring:
    sentinel:
      transport:
        # 指定sentinel控制台的地址
        dashboard: localhost:8080
```



启动内容中心，此时内容中心相关接口还未出现在`Sentinel`中，因为`Sentinel`默认和`Ribbon`一样都是懒加载的。



访问一下`http://127.0.0.1:8082/shares/1`，可以发现此时该接口路径已经出现在`Sentinel`中了。



### Sentinel流控规则

在`Sentinel`控制台上，可以看到菜单栏中的`簇点链路`功能。其中包含已经访问过的接口路径，比如`/shares/1`，可以点击该接口后面的`流控`按钮为其添加流控规则。



流控规则可以对不同的`来源`的`QPS`或`线程数`设置单机阈值进行限流。其中，高级模式中可以设置`流控模式`和`流控效果`。



流控模式：

1. 直接

   最好理解的流控模式，比如设置该接口的`QPS`为1，多次访问 `/shares/1`，就会出现部分请求直接返回`Blocked by Sentinel (flow limiting)`的情况。

2. 关联

   当关联的资源达到阈值，就限流自己。

   比如，设置`/shares/{id}`的接口的关联接口为`/getInstances`，`QPS`的阈值为1，并编写如下测试类代码循环调用`/getInstances`:

   ```java
   public class Test {
   
       public static void main(String[] args) {
           RestTemplate restTemplate = new RestTemplate();
           for (int i = 0;i < 1000000; i ++){
               String object = restTemplate.getForObject("http://localhost:8082/getInstances", String.class);
               System.out.println(object);
           }
       }
   }
   
   ```

   在此循环调用类执行时，访问`http://127.0.0.1:8082/shares/1`，会发现虽然没有设置这个接口的直接流控规则，但是因为设置了关联流控规则，依然因为`/getInstances`的`QPS`超过阈值而不能访问。

   **关联的流控规则可以用在两个相互影响性能的接口上，比如一个接口查询，另一个接口修改，如果看重修改的性能，可以为查询接口设置流控规则，关联修改接口，这样当修改接口`QPS`过大时，查询接口会被限制，进而保证修改接口的性能。也就是说，关联的流控规则是为了保护被关联的接口。**

3. 链路

   只记录指定链路上的流量。

   新建TestService并创建`common`方法：

   ```java
   package tech.punklu.contentcenter;
   
   import com.alibaba.csp.sentinel.annotation.SentinelResource;
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.stereotype.Service;
   
   @Slf4j
   @Service
   public class TestService {
   
   
       @SentinelResource("common")
       public String common(){
           log.info("common....");
           return "common";
       }
   }
   
   ```

   

   在`TestController`中新建两个接口调用这个接口：

   ```java
   @Autowired
   private TestService testService;
   
   @GetMapping("/test-a")
   public String testA(){
       this.testService.common();
       return "test-a";
   }
   
   @GetMapping("/test-b")
   public String testB(){
       this.testService.common();
       return "test-b";
   }
   ```

   

   增加如下配置：

   ```yaml
   spring:
     cloud:
       sentinel:
         # 配置web接口上下文不合并
         web-context-unify: false
   ```

   

   之所以增加这个接口是因为像上述代码中，出现两个web接口都调用被`@SentinelResource`注解标注的方法时，如果不增加这个配置项，会只在一个接口中出现`common`的上下文，且失去链路流控规则的效果。**这个配置是在Spring Cloud Alibaba 2.2.1.RELEASE（不包括） 之后才出现的。之前的版本需要另外通过代码配置的方式解决。**

   启动内容中心服务，先后访问`http://127.0.0.1:8082/test-a`和`http://127.0.0.1:8082/test-b`，使这两个接口出现在`Sentinel`控制台中。然后在`Sentinel`控制台的簇点链路中可以看到如下的层级关系：

   ```
   /test-a 
   	common
   /test-b
   	common
   ```

   选择`/test-a`下的`common`进行流控配置，`QPS`阈值设置为1，流控模式选择链路，流控模式的入口资源选择`/test-a`，然后不断访问`http://127.0.0.1:8082/test-a`，可以发现也会被`Sentinel`直接返回错误。而不断访问`http://127.0.0.1:8082/test-b`就不会出现这种情况。



流控规则中的`针对来源`指的是微服务级别的调用来源，`入口资源`指的是`API`级别的调用来源。



流控规则中还可配置流控效果，流控效果可选配置：

1. 快速失败

   直接失败、抛出异常。相关源码：`com.alibaba.csp.sentinel.slots.block.flow.controller.DefaultController`

2. Warm Up(预热)

   根据`codeFactor`(冷加载因子，默认为3)的值，从最开始的`阈值/codeFactor`作为最初的阈值，经过预热时长，才到达设置的最高`QPS`阈值。适用于秒杀服务等流量爆发增长的场景下保护微服务。
   

   Warm Up的相关讲解文档：[限流-冷启动](https://github.com/alibaba/sentinel/wiki/%E9%99%90%E6%B5%81---%E5%86%B7%E5%90%AF%E5%8A%A8)
   

   相关源码：`com.alibaba.csp.sentinel.slots.block.flow.controller.WarmUpController`

3. 排队等待

   匀速排队，让请求以均匀的速度通过，阈值类型必须设成`QPS`，否则无效。**排队等待的效果必须使用`QPS`而不能使用线程作为阈值类型。线程无效**


   将`common`的流控`QPS`阈值设置为1，流控效果设置为超时等待，等待时间设置长一些（5000000），然后编写如下测试代码：

   ```java
   public static void main(String[] args) {
       RestTemplate restTemplate = new RestTemplate();
       for (int i = 0;i < 1000000; i ++){
           String object = restTemplate.getForObject("http://localhost:8082/test-a", String.class);
           System.out.println(object);
       }
   }
   ```

   运行上述代码，可以发现虽然`common`的`QPS`阈值只有1，但是这个循环里的后续请求并未直接失败，而是排队执行，通过这种方式保证了不会因为突然特别多的线程而导致微服务挂掉的情况，也实现了不直接丢弃请求的功能。
   

   匀速排队相关讲解文档：[限流-匀速排队](https://github.com/alibaba/Sentinel/wiki/%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6-%E5%8C%80%E9%80%9F%E6%8E%92%E9%98%9F%E6%A8%A1%E5%BC%8F)


   相关源码：`com.alibaba.csp.sentinel.slots.block.flow.controller.RateLimiterController`



### Sentinel降级规则

在`Sentinel`控制台菜单栏里也有专门的`降级规则`入口，降级规则相比流控规则要简单很多。



降级规则里的降级策略有三种：

1. 慢调用比例

   平均响应时间


   在簇点链路里的`/shares{id}`这个`URL`上设置降级规则，选择`RT`策略，最大RT设置为1，熔断时长设置为5，最小请求数也设置为5。比例阈值代表当慢调用的比例超过这个值时触发断路器。代表当平均响应时间（秒级统计）超出阈值并且在设置的熔断时长（5s内）通过的请求数大于等于设置的最小请求数并且比例阈值大于设置的值时，则触发降级（断路器打开），熔断时长（5s后）结束后关闭降级。

2. 异常比例

   和慢调用比例的概念相似，只是把触发降级规则的条件改为了`比例阈值`、`熔断时长`、`最小请求数`。

3. 异常数

   把触发降级规则的条件改为了`异常数`、`熔断时长`、`最小请求数`。**需要注意的是，异常数的统计层级是以分钟计的，而不像其他两个是以秒计的。因为是以异常数为判断条件，所当熔断的时间结束时，异常数依然大于设置的阈值，就会再次触发降级，因此最好将熔断时长设置大一点。**

   

   降级相关源码：`com.alibaba.csp.sentinel.slots.block.degrade.DegradeRule#passCheck`



### Sentinel热点规则

热点规则可以针对某个接口的某个参数进行热点降级，即当`统计窗口时长`内该接口的指定参数有值且数量大于设置好的阈值时，对该接口进行保护，直接抛出异常。



新建如下接口：

```java
@GetMapping("/test-hot")
@SentinelResource("hot")
public String testHot(@RequestParam(required = false)String a,
					@RequestParam(required = false)String b){
    return a + " " + b;
}
```

启动后访问该接口，然后在`Sentinel`控制台上的簇点链路里对该接口进行热点规则设置：

1. 参数索引

   即参数在接口参数列表中的下标，从0开始计算。

2. 单机阈值

   即为该参数有值时的统计窗口时长中的最大请求个数，当超过这个个数时，后续请求会直接抛出异常。

3. 统计窗口时长

   设置在多长时间内指定的参数有值的请求数量，才会使接口抛出异常。



设置参数索引为0，单机阈值和统计窗口时长都为1，然后不停的访问`http://127.0.0.1:8082/test-hot?a=1&b=2`，会发现部分请求直接返回错误。而不停的访问`http://127.0.0.1:8082/test-hot?b=2`时则不会抛出异常。可见规则是对第一个参数a使用的。



除此之外，还可以在高级选项里设置`参数例外项`。例如`参数值`设为5，`限流阈值`设为`1000`，参数类型设置为`1000`，然后不停访问`http://127.0.0.1:8082/test-hot?a=5`，会发现不会被直接返回异常。也就是说，可以在对单个参数设置全局热点规则后，再对这个参数的`特殊值`设置另外的热点规则。



**热点规则需要特别注意的是，参数类型必须是基本类型或`String`，否则不会生效。**



相关源码：`com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowChecker#passCheck`






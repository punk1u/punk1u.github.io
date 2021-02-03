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





### Sentinel系统规则

Sentinel系统规则支持五种阈值类型：

1. LOAD

   当系统一分钟的负载超过阈值，且并发线程数超过**系统容量**时触发，建议设置为CPU核心数*2.5（**仅对Linux/Unix-like机器生效，可以使用`uptime命令查看load负载`**）。

   系统容量 = maxQps * minRt

   1. maxQps

      Sentinel秒级统计出来的最大QPS

   2. minRt

      Sentinel秒级统计出来的最小响应时间


   相关源码：`com.alibaba.csp.sentinel.slots.system.SystemRuleManager#checkBbr`

2. RT

   所有入口流量的平均RT达到阈值触发

3. 线程数

   所有入口流量的并发线程数达到阈值触发

4. 入口QPS

   所有入口流量的QPS达到阈值触发


   相关源码：`com.alibaba.csp.sentinel.slots.system.SystemRuleManager#checkSystem`

5. CPU使用率



### Sentinel授权规则

授权规则可用来对其他微服务进行授权，比如对`/shares/{id}`这个接口增加授权规则，设置`流控应用`为test，授权类型为白名单，即意味着这个接口对名为`test`的微服务开放调用，同理，还可设置黑名单，禁止某个服务调用。



### 代码配置规则

前面的规则设置都是通过`Sentinel`的控制台进行的，`Sentinel`也支持通过代码配置的形式设置规则。

```java
@GetMapping("/test-add-flow-rule")
public String testAddFlowRule(){
    this.initFlowQpsRule();
    return "success";
}
    
private void initFlowQpsRule(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule("/shares/1");
    // 设置QPS阈值
    rule.setCount(20);
    // 设置阈值类型
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule.setLimitApp("default");
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```



### Sentinel与控制台通信原理

微服务会集成`sentinel-transport-simple-http`模块，通过这个模块把微服务集成到`Sentinel控制台`上，并定时给`Sentinel控制台`发送心跳。此时`Sentinel控制台`可以知道各个微服务的地址，在`Sentinel控制台`的机器列表里可以看到对应的微服务机器信息（ip、端口号等）。`Sentinel`也就可以获取到各个微服务的监控信息（通过调用微服务的监控API），也可以通过调用微服务上的接收`Sentinel`规则的接口将设置好的规则推送过去。



- 注册/心跳发送

  `com.alibaba.csp.sentinel.transport.heartbeat.SimpleHttpHeartbeatSender`

- 通信API

  `com.alibaba.csp.sentinel.command.CommandHandler`的实现类



### 控制台相关配置项

应用端连接控制台配置项：

```properties
spring.cloud.sentinel.transport:
	# 指定控制台的地址
	dashboard: localhost:8080
	# 指定和控制台通信的IP
	# 如不配置，会自动选择一个IP注册
	client-ip: 127.0.0.1
	# 指定和控制台通信的端口，默认值8719
	# 如不设置，会自动从8719开始扫描，依次+1，直到找到未被占用的接口
	port: 8719
	# 心跳发送周期，默认值null
	# 但在SimpleHttpHeartbeatSender会用默认值10秒
	heartbeat-interval-ms: 10000
```



控制台启动时的可选配置项：

| 配置项                                            | 默认值         | 描述                                                         |
| ------------------------------------------------- | -------------- | ------------------------------------------------------------ |
| server.port                                       | 8080           | 指定端口                                                     |
| csp.sentinel.dashboard.server                     | localhost:8080 | 指定地址                                                     |
| project.name                                      | -              | 指定程序的名称                                               |
| sentinel.dashboard.auth.username[1.6版本开始支持] | sentinel       | Dashboard登录账号                                            |
| sentinel.dashboard.auth.password[1.6版本开始支持] | sentinel       | Dashboard登录密码                                            |
| server.servlet.session.timeout[1.6版本开始支持]   | 30分钟         | 登录Session过期时间<br />配置为7200表示7200秒<br />配置为60m表示60分钟 |



例如，可以在启动时指定登录的账号密码:

```shell
java -jar -Dsentinel.dashboard.auth.username=punk1u -Dsentinel.dashboard.auth.password=punk1u sentinel-dashboard-1.8.0.jar
```



### Sentinel相关API

在之前的例子中，`Sentinel`都是用来保护`Spring MVC`的接口，但是`Sentinel`不止可以保护`Spring MVC`定义的接口，也可保护其他资源。



首先添加配置项禁用掉`Sentinel`对`Spring MVC`的监控和保护，防止干扰：

```yaml
spring:
  cloud:
    sentinel:
      filter:
        # 关闭掉对Spring MVC端点的保护
        enabled: false
```



在`TestController`中增加如下测试代码：

```java
@GetMapping("/test-sentinel-api")
public String testSentinelAPI(@RequestParam(required = false) String a){
    // 定义一个sentinel保护的资源
    Entry entry = null;
    try {
        entry = SphU.entry("test-sentinel-api");
        // 被保护的业务逻辑
        return a;
    }
    // 如果被保护的资源被限流或者降级了，就会抛BlockException
    catch (BlockException e) {
        log.warn("限流，或者降级了",e);
        return "限流，或者降级了";
    }
    finally {
        if (entry != null){
            // 退出entry
            entry.exit();
        }
    }
}
```



启动内容中心，访问`http://127.0.0.1:8082/test-sentinel-api?a=2`接口，可以看到此时相关的`Spring MVC`接口已经不再出现在`Sentinel`控制台上了。此时可以对这个`非Spring MVC` 资源进行限流，设置QPS阈值为1，不停访问`http://127.0.0.1:8082/test-sentinel-api?a=2`就可以看到流控规则生效，直接报错了。



在上面的代码中，通过使用`Sentinel`的`SphU`类保护被指定的名为`test-sentinel-api`的资源。当`QPS`、`线程数`、`RT`、`错误率`、`错误次数`等指标超过阈值时直接抛出`BlockException`。



需要注意的是，这种用法对于抛出的其他类型的异常是不会进行捕获并返回相应的`Sentinel错误信息`的。要想使自己抛出的异常也被`Sentinel`降级，需要进行特殊处理：

```java
@GetMapping("/test-sentinel-api")
public String testSentinelAPI(@RequestParam(required = false) String a){
    // 定义一个sentinel保护的资源
    Entry entry = null;
    try {
        entry = SphU.entry("test-sentinel-api");
        // 被保护的业务逻辑
        if (StringUtils.isBlank(a)){
        	throw new IllegalArgumentException("a不能为空");
        }
    return a;
    }
    // 如果被保护的资源被限流或者降级了，就会抛BlockException
    catch (BlockException e) {
        log.warn("限流，或者降级了",e);
        return "限流，或者降级了";
    }catch (IllegalArgumentException e){
        // 统计IllegalArgumentException发生的次数、占比...
        Tracer.trace(e);
        return "参数非法!";
    }
    finally {
        if (entry != null){
            // 退出entry
            entry.exit();
        }
    }
}
```



通过`Tracer`捕获其他类型的异常后，再重新启动应用，设置该接口的`QPS`的阈值为1，访问`http://127.0.0.1:8082/test-sentinel-api`，会发现此时`IllegalArgumentException`也可以触发流控规则了。



之前简略提过可以对接口设置针对不同的微服务来源设置流控规则。可以通过`Sentinel`里的`ContextUtil`来完成对应的功能实现。



修改代码：

```java
@GetMapping("/test-sentinel-api")
public String testSentinelAPI(@RequestParam(required = false) String a){

    String resourceName = "test-sentinel-api";
    ContextUtil.enter(resourceName,"test-service");

    // 定义一个sentinel保护的资源
    Entry entry = null;
    try {
        entry = SphU.entry("test-sentinel-api");
        // 被保护的业务逻辑
        if (StringUtils.isBlank(a)){
        	throw new IllegalArgumentException("a不能为空");
        }
        return a;
    }
    // 如果被保护的资源被限流或者降级了，就会抛BlockException
    catch (BlockException e) {
        log.warn("限流，或者降级了",e);
        return "限流，或者降级了";
    }catch (IllegalArgumentException e){
        // 统计IllegalArgumentException发生的次数、占比...
        Tracer.trace(e);
        return "参数非法!";
    }
    finally {
        if (entry != null){
            // 退出entry
            entry.exit();
        }
        ContextUtil.exit();
    }
}
```



定义一个名称为`test-sentinel-api`的资源，保护从`test-service`这个微服务来的请求。启动后，先访问`http://127.0.0.1:8082/test-sentinel-api`，会发现此时未触发流控规则，返回结果为`参数非法`，说明未被流控，对该接口新增流控规则，`针对来源`填写代码里声明的`test-service`，再多次访问`http://127.0.0.1:8082/test-sentinel-api`会发现此时已经被流控，将`针对来源`改为其他值再次访问，则不会触发流控。



### SentinelResource注解

上面的通过`ContextUtil`、`Tracer`、`SphU`的方式实现限流的方式会导致流控降级代码和业务代码耦合，所以应该使用`@Sentinel`注解实现相关功能。



```java
@GetMapping("/test-sentinel-resource")
@SentinelResource(value = "test-sentinel-resource",
        blockHandler = "block",
        fallback = "fallback")
public String testSentinelResource(@RequestParam(required = false) String a){

    // 被保护的业务逻辑
    if (StringUtils.isBlank(a)){
        throw new IllegalArgumentException("a cannot be blank");
    }
    return a;
}

/**
 * 处理限流或降级
 * @param a
 * @param e
 * @return
 */
public String block( String a,BlockException e){
    log.warn("限流，或者降级了 block",e);
    return "限流，或者降级了 block";
}

/**
 * 处理降级
 * @param a
 * @param e
 * @return
 */
public String fallback( String a,BlockException e){
    log.warn("限流，或者降级了 fallback",e);
    return "限流，或者降级了 fallback";
}
```



可以看到通过`@Sentinel`注解可以只在方法中关心业务代码，在注解的属性中指定对应的流控和降级处理方法。`1.6`以后还可以在注解属性中指定对应的`流控处理类`和`降级处理类`。**流控处理类和降级处理类里的方法必须使用static修饰符修饰。**



相关源码：`com.alibaba.csp.sentinel.annotation.aspectj.SentinelResourceAspect`



### Sentinel整合RestTemplate

在启动类中的声明`RestTemplate`实例的方法上添加上`@SentinelRestTemplate`注解即可实现。之后在`Sentinel`控制台中即可看到对应的接口并设置相应规则。



可通过开关关闭，便于本地调试：

```yaml
resttemplate:
	sentinel:
		# 关闭@SentinelRestTemplate注解
		enabled: false
```



相关源码：`org.springframework.cloud.alibaba.sentinel.custom.SentinelBeanPostPrecessor`





### Sentinel整合Feign

首先添加配置项：

```yaml
feign:
	sentinel:
		# 为Feign整合Sentinel
		enabled: true
```



启动项目，`http://127.0.0.1:8082/shares/1`。然后可以在`Sentinel`对控制台看到该接口依赖的用户中心的接口，对用户中心的该接口设置流控规则，设置`QPS`阈值为1。然后不停访问``http://127.0.0.1:8082/shares/1``，会发现已经被流控了。



除此之外，还可为Feign设置当限流降级发生时的自己的处理逻辑：

新建`UserCenterFeignClient`的对应异常处理类:

```java
package tech.punklu.contentcenter.feignclient.fallback;

import org.springframework.stereotype.Component;
import tech.punklu.contentcenter.domain.dto.user.UserDTO;
import tech.punklu.contentcenter.feignclient.UserCenterFeignClient;

@Component
public class UserCenterFeignClientFallback implements UserCenterFeignClient {

    @Override
    public UserDTO findById(Integer id) {
        UserDTO userDTO = new UserDTO();
        userDTO.setWxNickname("一个默认用户");
        return userDTO;
    }
}
```



即新建一个类实现之前的用户中心`Feign`代理接口，并实现对应的流控降级后的默认处理方法，然后在`UserCenterFeignClient`类上的`@FeignClient`注解上增加`fallback`属性，将fallback的默认兜底处理类指向刚才创建好的类：

```java
@FeignClient(name = "user-center",fallback = UserCenterFeignClientFallback.class)
public interface UserCenterFeignClient {

}
```



启动项目，设置流控规则后，重复访问`http://127.0.0.1:8082/shares/1`后，会发现虽然触发了流控规则，但是并没有直接抛出异常且`wxNickName`仍然有值，只是被替换为了兜底策略里的默认值。



此时控制台里并没有打印进入兜底策略的相关日志，如果想要显示这样的日志，可以使用`@FeignClient`注解的`fallbackFactory`属性来指定对应的兜底类，首先新建兜底类：

```java
package tech.punklu.contentcenter.feignclient.fallback;

import feign.hystrix.FallbackFactory;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import tech.punklu.contentcenter.domain.dto.user.UserDTO;
import tech.punklu.contentcenter.feignclient.UserCenterFeignClient;

@Component
@Slf4j
public class UserCenterFeignClientFallbackFactory
        implements FallbackFactory<UserCenterFeignClient> {

    @Override
    public UserCenterFeignClient create(Throwable throwable) {
        return new UserCenterFeignClient() {
            @Override
            public UserDTO findById(Integer id) {
                log.warn("远程调用被限流/降级了",throwable);
                UserDTO userDTO = new UserDTO();
                userDTO.setWxNickname("一个默认用户");
                return userDTO;
            }

            @Override
            public UserDTO testFeignParam(UserDTO userDTO) {
                return null;
            }
        };
    }
}

```



然后在`UserCenterFeignClient`中通过`@FeignClient`的`fallbackFactory`注解指定新创建的兜底类即可：

```java
@FeignClient(name = "user-center",fallbackFactory = UserCenterFeignClientFallbackFactory.class)
public interface UserCenterFeignClient {


}
```



启动项目后重复上述测试、增加流控步骤，可以发现此时`wxNickName`依然有默认值，且控制台也打印出了对应的兜底日志。



相关源码：`org.springframework.cloud.alibaba.sentinel.feign.SentinelFeign`





### Sentinel错误页优化

之前的代码中，不管是限流还是降级或是其他错误，都会统一返回同样的Sentinel默认错误信息，不能区分错误类型。为了实现根据不同的错误类型返回不同的信息，需要编写专门的处理逻辑：

```java
package tech.punklu.contentcenter.feignclient.blockhandler;

import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.BlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityException;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeException;
import com.alibaba.csp.sentinel.slots.block.flow.FlowException;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException;
import com.alibaba.csp.sentinel.slots.system.SystemBlockException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
@Component
public class MyUrlBlockHandler implements BlockExceptionHandler {

    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, BlockException e) throws Exception {
        ErrorMsg errorMsg = null;
        if (e instanceof FlowException){
            // 限流异常
            errorMsg = ErrorMsg.builder().
                    status(100)
                    .msg("限流了")
                    .build();
        }else if (e instanceof DegradeException){
            // 降级异常
            errorMsg = ErrorMsg.builder().
                    status(101)
                    .msg("降级了")
                    .build();
        }else if (e instanceof SystemBlockException ){
            // 系统规则异常
            errorMsg = ErrorMsg.builder().
                    status(102)
                    .msg("系统规则（负载/...不满足要求）了")
                    .build();
        }else if (e instanceof AuthorityException){
            // 授权异常
            errorMsg = ErrorMsg.builder().
                    status(103)
                    .msg("授权规则不通过")
                    .build();
        }else if (e instanceof ParamFlowException){
            // 参数热点异常
            errorMsg = ErrorMsg.builder().
                    status(104)
                    .msg("热点参数限流")
                    .build();
        }
        // 设置HTTP状态码
        httpServletResponse.setStatus(500);
        httpServletResponse.setCharacterEncoding("utf-8");
        httpServletResponse.setHeader("Content-Type","application/json;charset=utf-8");
        httpServletResponse.setContentType("application/json;charset=utf-8");
        new ObjectMapper().writeValue(httpServletResponse.getWriter(),errorMsg);
    }
}

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
class ErrorMsg{
    private Integer status;

    private String msg;
}

```



如上所示，实现`Sentinel`的扩展接口`BlockExceptionHandler`，即可以实现对应的区分错误来源并做相应的限制。



启动后分别设置流控、降级规则，并不停访问``http://127.0.0.1:8082/shares/1``接口，可以看到已经根据不同的错误返回了不同的错误信息了。



### Sentinel区分来源

在`Sentinel`控制台里的`流控规则`里的`针对来源`和`授权规则`里的`流控应用`都需要区分`调用者的来源`才能生效。为了区分调用者的来源，也需要进行二次开发。



首先创建一个实现`Sentinel`的`RequestOriginParser`接口的实现类：

```java
package tech.punklu.contentcenter.feignclient.blockhandler;

import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.RequestOriginParser;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

@Component
public class MyRequestOriginParser implements RequestOriginParser {



    @Override
    public String parseOrigin(HttpServletRequest httpServletRequest) {
        // 从请求参数中获取名为origin的参数并返回
        // 如果获取不到origin参数，直接抛出异常
        // 实际使用时，来源信息最好放在header中
        String origin = httpServletRequest.getParameter("origin");
        if (StringUtils.isBlank(origin)){
            throw new IllegalArgumentException("origin must be specified");
        }
        return origin;
    }
}

```



启动项目，直接访问`http://127.0.0.1:8082/shares/1`接口，因为没有指定`origin`参数，所以直接报错了。添加来源信息后`http://127.0.0.1:8082/shares/1?origin=browser`，可以正常访问。



对该接口添加授权规则，指定`流控应用`为`browse`，授权类型为`browser`，代表该接口不对浏览器开发，再次访问`http://127.0.0.1:8082/shares/1?origin=browser`则显示被授权规则拦截了。



对该接口添加流控规则，设置`针对来源`项为`browser`，QPS为1，多次访问`http://127.0.0.1:8082/shares/1?origin=browser`，发现会被限流。设置`针对来源`项为其他值比如`android`时，则不会被限流，说明流控规则起到了针对来源的效果。



**不管是区分来源还是错误页优化，在底层都是通过`Sentinel`的`CommonFilter`过滤器实现的相关功能。在过滤器中对`来源信息`、抛出的异常等信息进行先行处理。获取到我们自定义的相关处理类并调用相关方法进行的。**
















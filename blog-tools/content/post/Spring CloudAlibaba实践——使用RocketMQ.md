---
title: "Spring Cloud Alibaba实践——使用RocketMQ" 
date: 2021-02-03T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# 使用Spring Cloud Alibaba RocketMQ



## 初步完成管理员审核功能

在内容中心中，编写管理员审核的相关接口：

```java
package tech.punklu.contentcenter.controller.content;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import tech.punklu.contentcenter.domain.dto.content.ShareAuditDTO;
import tech.punklu.contentcenter.domain.entity.content.Share;
import tech.punklu.contentcenter.service.content.ShareService;

@RestController
@RequestMapping("/admin/shares")
public class ShareAdminController {

    @Autowired
    private ShareService shareService;

    @PutMapping("/audit/{id}")
    public Share auditById(@PathVariable Integer id,
                           @RequestBody ShareAuditDTO auditDTO){
        // TODO 认证、授权
        return this.shareService.auditById(id,auditDTO);
    }
}

```



<span id="auditService">相关`Service`方法：</span>

```java
public Share auditById(Integer id, ShareAuditDTO auditDTO) {
    // 查询Share是否存在，不存在或者当前的audit_status != NOT_YET，那么抛异常
    Share share = this.shareMapper.selectByPrimaryKey(id);
    if (share == null){
    throw new IllegalArgumentException("参数非法！该分享不存在！");
    }
    if (!Objects.equals("NOT_YET",share.getAuditStatus())){
    throw new IllegalArgumentException("参数非法！该分享已审核过");
    }
    // 审核资源，将状态设为PASS/REJECT
    share.setAuditStatus(auditDTO.getAuditStatusEnum().toString());
    this.shareMapper.updateByPrimaryKey(share);
    // 如果是PASS，那么为发布人添加积分
    // 异步执行，缩短响应耗时
    // TODO 用MQ完成异步添加积分的操作
    return share;
}
```



相应的`内容审核 `对象：

```java
package tech.punklu.contentcenter.domain.dto.content;

import lombok.Data;
import tech.punklu.contentcenter.domain.enums.AuditStatusEnum;

@Data
public class ShareAuditDTO {

    /**
     * 审核状态
     */
    private AuditStatusEnum auditStatusEnum;

    /**
     * 原因
     */
    private String reason;
}

```



相应的`审核状态`枚举类：

```java
package tech.punklu.contentcenter.domain.enums;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum  AuditStatusEnum {

    /**
     * 待审核
     */
    NOT_YET,
    /**
     * 审核通过
     */
    PASS,
    /**
     * 审核不通过
     */
    REJECT

}

```





## Spring实现异步的方法

1. AsyncRestTemplate

   参考文档：[Spring 的异步HTTP请求AsyncRestTemplate](https://blog.csdn.net/jiangchao858/article/details/86709750)

2. @Async注解

   参考文档：[Creating Asynchronous Methods](https://spring.io/guides/gs/async-method/)

3. WebClient(Spring 5.0引入)

   参考文档：[WebClient](https://docs.spring.io/spring-framework/docs/5.1.8.RELEASE/spring-framework-reference/web-reactive.html#webflux-client)

4. 消息队列

   使用消息队列如`Kafka`、`RocketMQ`等来完成异步功能。

   

   **对于上述的管理员审核功能（[审核相关代码](#auditService)）中的TODO代码后相应的审核成功后给用户添加积分的功能的异步操作，可以在内容中心使用MQ生产消息，用户中心从MQ监听到内容中心生产的添加积分的消息并进行相应的处理，便完成了添加积分步骤的异步操作。**



## MQ适用场景

1. 异步处理

   把一些耗时、但不阻塞主流程的业务用MQ做异步处理，可以提升用户体验

2. 流量削峰填谷

   比如，秒杀场景中，刚开始的瞬间参与的人数过多，可以用MQ控制参加活动的人数，超过限定的人数后，直接返回秒杀失败，起到削峰填谷的效果。

3. 解耦微服务

   比如两个微服务，A会通知B进行相应的处理，如果使用接口的方式传递消息，当B服务不可用的时候会无法完成相应的逻辑，如果使用MQ从A向B传送消息，当B从不可用状态恢复时，依然可以读取到A发送的消息并进行处理。



## 搭建RocketMQ

安装教程：[windows下搭建RocketMQ及相应的控制台](https://www.jianshu.com/p/4a275e779afa)



## RocketMQ的术语/概念

1. TOPIC(主题)

   一类消息的集合，RocketMQ的基本订阅单位

2. 消息模型

   - Producer(生产者，生产消息)
   - Broker(消息代理，存储消息、转发消息)
   - Consumer(消费者，消费消息)

3. 部署结构

   - Name Server(名字服务)：RocketMQ的服务发现组件，记录了TOPIC和Broker之间的地址映射关系
   - Broker Server(代理服务器)：消息中转角色，负责存储消息、转发消息

4. 消费模式

   - PullConsumer（拉取式消费)：应用调用Consumer的拉取信息方法从Broker Server拉取消息
   - Push Consumer (推动式消费)：Broker收到消息后主动推给消费端，该模式实时性较高

5. Group(组)

   - Producer Group（生产者组）：同一类Producer的集合，这类Producer发送同一类消息
   - Consumer Group（消费者组）：同一类Consumer的集合，这类Consumer消费同一类消息

6. 消息传播模式

   - Clustering（集群）：相同Consumer Group的每个Consumer实例平均分摊消息
   - BroadCasting（广播）：相同Consumer Group的每个Consumer实例都接受全量的消息

7. 消息类型

   普通消息、顺序消息、定时/延时消息、事务消息



## Spring消息编程模型

借助Spring抽象好的消息编程模型，可以非常方便的使用各种MQ进行开发，同时降低变更MQ时相应的代码改动量。



## 内容中心引入RocketMQ生产者

因为是从`内容中心`向`用户中心`发送添加积分的消息，所以`内容中心`是生产者。



首先为`内容中心`添加`RocketMQ`的依赖：

```xml
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-spring-boot-starter -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```



然后添加RocketMQ的相关配置项：

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  # 设置rocketMQ消息生产者组
  producer:
    group: test-group
```



最后，完成之前管理器审核代码里的添加积分操作，通过MQ发送添加积分的消息：

```java
@Autowired
private RocketMQTemplate rocketMQTemplate;

@Transactional(rollbackFor = Exception.class)
public Share auditById(Integer id, ShareAuditDTO auditDTO) {
    // 查询Share是否存在，不存在或者当前的audit_status != NOT_YET，那么抛异常
    Share share = this.shareMapper.selectByPrimaryKey(id);
    if (share == null){
    	throw new IllegalArgumentException("参数非法！该分享不存在！");
    }
    if (!Objects.equals("NOT_YET",share.getAuditStatus())){
    	throw new IllegalArgumentException("参数非法！该分享已审核过");
    }
    // 审核资源，将状态设为PASS/REJECT
    share.setAuditStatus(auditDTO.getAuditStatusEnum().toString());
    this.shareMapper.updateByPrimaryKey(share);

    // 如果是PASS，那么发送消息给rocketMQ，让用户中心去消费，并为发布人添加积分
    // 异步执行，缩短响应耗时
    this.rocketMQTemplate.convertAndSend("add-bonus",
                    UserAddBonusMsgDTO.builder()
                    .userId(share.getUserId())
                    .bonus(50).build()
    );
    return share;
}
```



其中，`add-bonus`即为要发送的消息的`TOPIC`(主题)。



UserAddBonusMsgDTO为添加积分的消息对象：

```java
package tech.punklu.contentcenter.domain.dto.message;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserAddBonusMsgDTO {

    /**
     * 为谁加积分
     */
    private Integer userId;

    /**
     * 加多少积分
     */
    private Integer bonus;
}

```



然后启动内容中心服务，先将数据库表`Share`分享表中的`audit_status`审核状态字段修改为`NOT_YET`，表示还没审核。然后用`PostMan`模拟` PUT 127.0.0.1:8082/admin/shares/audit/1?origin=browser`审核请求，可以看到内容中心正常返回了如下的数据:

```json
{
    "id": 1,
    "userId": 1,
    "title": "spring",
    "createTime": "2021-01-27T13:16:30.000+0000",
    "updateTime": "2021-01-27T13:16:30.000+0000",
    "isOriginal": false,
    "author": "punklu",
    "cover": "xxx",
    "summary": "",
    "price": 0,
    "downloadUrl": "",
    "buyCount": 1,
    "showFlag": false,
    "auditStatus": "PASS",
    "reason": ""
}
```



访问之前搭建好的`RocketMQ`控制台，可以在`消息`一栏里看到名称为`add-bonus`的`TOPIC`主题，并可在具体搜索结果里看到刚才的请求向`RocketMQ Server`发送的消息，消息的`Body`为：

```json
{"userId":1,"bonus":50}
```

即为刚才的请求生成的添加积分的消息。



## 用户中心引入RocketMQ消费者

内容中心引入了生产者生产给用户添加积分的消息。还需要在用户中心引入一个消费该消息的功能。



首先，给用户中心添加依赖：

```xml
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-spring-boot-starter -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```



然后，添加相应的`RocketMQ`配置项：

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
```



然后，编写相应的消费添加积分消息的类：

```java
package tech.punklu.usercenter.rocketmq;

import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import tech.punklu.usercenter.dao.bonus.BonusEventLogMapper;
import tech.punklu.usercenter.dao.user.UserMapper;
import tech.punklu.usercenter.domain.dto.message.UserAddBonusMsgDTO;
import tech.punklu.usercenter.domain.entity.bonus.BonusEventLog;
import tech.punklu.usercenter.domain.entity.user.User;

import java.util.Date;

@Service
@RocketMQMessageListener(consumerGroup = "consumer-group",topic = "add-bonus")
public class AddBonusListener implements RocketMQListener<UserAddBonusMsgDTO> {


    @Autowired
    private UserMapper userMapper;

    @Autowired
    private BonusEventLogMapper bonusEventLogMapper;

    @Override
    public void onMessage(UserAddBonusMsgDTO userAddBonusMsgDTO) {
        // 当收到消息时，执行的业务
        // 1、为用户加积分
        Integer userId = userAddBonusMsgDTO.getUserId();
        // 要加的积分
        Integer bonus = userAddBonusMsgDTO.getBonus();
        User user = this.userMapper.selectByPrimaryKey(userId);
        user.setBonus(user.getBonus() + userAddBonusMsgDTO.getBonus());
        this.userMapper.updateByPrimaryKeySelective(user);
        // 2、记录日志到bonus_event_log表里面
        this.bonusEventLogMapper.insert(
                        BonusEventLog.builder()
                        .userId(userId)
                        .value(bonus)
                        .event("CONTRIBUTE")
                        .createTime(new Date())
                        .description("投稿加积分...")
                        .build()
        );
    }
}
```



其中，接收到的消息转换对象`UserAddBonusMsgDTO`与内容中心中的相同。



相应的积分日志相关的`CRUD`代码及积分日志`Bean`对象：

```java
package tech.punklu.usercenter.domain.entity.bonus;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.Column;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;
import java.util.Date;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "bonus_event_log")
public class BonusEventLog {
    /**
     * Id
     */
    @Id
    @GeneratedValue(generator = "JDBC")
    private Integer id;

    /**
     * user.id
     */
    @Column(name = "user_id")
    private Integer userId;

    /**
     * 积分操作值
     */
    private Integer value;

    /**
     * 发生的事件
     */
    private String event;

    /**
     * 创建时间
     */
    @Column(name = "create_time")
    private Date createTime;

    /**
     * 描述
     */
    private String description;
}
```



```java
package tech.punklu.usercenter.dao.bonus;

import tech.punklu.usercenter.domain.entity.bonus.BonusEventLog;
import tk.mybatis.mapper.common.Mapper;

public interface BonusEventLogMapper extends Mapper<BonusEventLog> {
}
```



```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.itmuch.usercenter.dao.bonus.BonusEventLogMapper">
    <resultMap id="BaseResultMap" type="com.itmuch.usercenter.domain.entity.bonus.BonusEventLog">
        <!--
          WARNING - @mbg.generated
        -->
        <id column="id" jdbcType="INTEGER" property="id" />
        <result column="user_id" jdbcType="INTEGER" property="userId" />
        <result column="value" jdbcType="INTEGER" property="value" />
        <result column="event" jdbcType="VARCHAR" property="event" />
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime" />
        <result column="description" jdbcType="VARCHAR" property="description" />
    </resultMap>
</mapper>
```



首先查看下数据库里`id`为`1`的用户的相关信息：

| id   | wx_id | wx_nickname | roles | avatar_url | create_time         | update_time         | bonus |
| ---- | ----- | ----------- | ----- | ---------- | ------------------- | ------------------- | ----- |
| 1    | 111   | punklu      | 111   | xxx        | 2021-01-27 13:05:53 | 2021-01-27 13:05:53 | 100   |


可以看到此时的积分为`100`，且积分日志表里没有记录。



启动用户中心，内容中心在启动后会收取到`RocketMQ`上存储的添加积分的消息并进行处理。此时查看对应的`用户表`和`积分记录表`可以看到`用户表`中对应的用户行数据已经加上了`50`点积分，且积分日志表`bonus_event_log`也已经有了这一次添加积分的记录。



## 基于RocketMQ实现分布式事务

在之前的代码中，大致的逻辑是，审核过程的最后，`内容中心`向`RocketMQ`发送一条添加积分的消息，`用户中心`监听到这条消息后会根据消息中的内容进行相应的给用户添加积分。但是，如果在`内容中心`中向`RocketMQ`发送添加积分消息的代码后面还有一段执行代码，并且这段代码可能会抛出异常，那么当抛出异常时因为这个方法上添加了`Spring`的`@Transactional`注解，此时前面对数据库的操作将会回滚，但是添加积分的消息已经发送到`RockerMQ`。此时如果消息被`用户中心`监听到就会出现不应添加积分但加了的情况，为了避免这种情况，可以使用`RocketMQ`的事务消息来保证`内容中心`和`用户中心`的分布式事务。



### RockerMQ事务消息流程原理

<span a="rocketmqtransactionjpg">RocketMQ事务消息的流程及大概原理如下图所示：</span>

![RocketMQ 分布式事务实现原理.jpg](https://i.loli.net/2021/02/04/WGCiLxF4nU7qQXD.jpg)



RT，在事务消息中，生产者最开始发送给`RocketMQ`的是一个`半消息`，即还不可生效不可被消费者消费的消息，此时`本地事务还没有执行`。`RocketMQ`接收到`半消息`后向生产者反馈已接受到半消息并通知`生产者`再执行提交本地的事务。如果`生产者`的本地事务执行成功，则生产者向`RocketMQ`发送确认消息通知`RocketMQ`把之前的`半消息`提交为可被消费者消费的消息并投递给消费者。`RocketMQ`接收到`本地事务执行成功，转换半消息状态`的返回结果后，会再次回查生产者确认本地事务执行成功。根据回查的结果来进行`提交/回滚`。如果生产者的本地事务执行失败，则生产者向`RocketMQ`发送回滚要求，让`RocketMQ`的`Broker`回滚掉之前的半消息。同时，可能会有生产者出现问题导致长时间不向`RocketMQ `发送`回滚/提交` `半消息`的二次确认请求。当`RocketMQ`在这种情形下长时间没有接收到生产者的`半消息`确认请求时，会主动回查生产者的本地事务状态，如果一直检查不到最新的`生产者`本地事务状态，则也会回滚掉`RocketMQ`中之前存储的那条`半消息`，如此便保证了`生产者`和`消费者`的事务一致性。



### 编码实现分布式事务

首先添加`RocketMQ`的事务日志表：

```sql
USE `content_center`;

-- -----------------------------------------------------
-- Table `rocketmq_transaction_log`
-- -----------------------------------------------------
create table rocketmq_transaction_log
(
  id             int auto_increment comment 'id' primary key,
  transaction_Id varchar(45) not null comment '事务id',
  log            varchar(45) not null comment '日志'
)
comment 'RocketMQ事务日志表';
```



以及`内容中心`对应的对象和`DAO`层：

```java
package tech.punklu.contentcenter.domain.dto.message;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.Column;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

@AllArgsConstructor
@NoArgsConstructor
@Builder
@Data
@Table(name = "rocketmq_transaction_log")
public class RocketmqTransactionLog {
    /**
     * id
     */
    @Id
    @GeneratedValue(generator = "JDBC")
    private Integer id;

    /**
     * 事务id
     */
    @Column(name = "transaction_Id")
    private String transactionId;

    /**
     * 日志
     */
    private String log;
}
```



```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="tech.punklu.contentcenter.dao.message.RocketmqTransactionLogMapper">
    <resultMap id="BaseResultMap" type="tech.punklu.contentcenter.domain.dto.message.RocketmqTransactionLog">
        <!--
          WARNING - @mbg.generated
        -->
        <id column="id" jdbcType="INTEGER" property="id" />
        <result column="transaction_Id" jdbcType="VARCHAR" property="transactionId" />
        <result column="log" jdbcType="VARCHAR" property="log" />
    </resultMap>
</mapper>
```



首先改造原有的审核通过后发送`RocketMQ`消息的代码：

```java
public Share auditById(Integer id, ShareAuditDTO auditDTO) {
    // 查询Share是否存在，不存在或者当前的audit_status != NOT_YET，那么抛异常
    Share share = this.shareMapper.selectByPrimaryKey(id);
    if (share == null){
    	throw new IllegalArgumentException("参数非法！该分享不存在！");
    }
    if (!Objects.equals("NOT_YET",share.getAuditStatus())){
    	throw new IllegalArgumentException("参数非法！该分享已审核过");
    }
    // 如果是PASS，那么发送消息给rocketMQ，让用户中心去消费，并为发布人添加积分
    if(AuditStatusEnum.PASS.equals(auditDTO.getAuditStatusEnum())){
        // 发送半消息（即在事务中发送消息）
        // UUID作为分布式事务id
        String transactionId = UUID.randomUUID().toString();
        this.rocketMQTemplate.sendMessageInTransaction(
                "add-bonus",
                MessageBuilder.
                withPayload(UserAddBonusMsgDTO.builder()
                    .userId(share.getUserId())
                    .bonus(50)
                    .build()
                )
                .setHeader(RocketMQHeaders.TRANSACTION_ID, transactionId)
                .setHeader("share_id", id)
                .build(), auditDTO
        );
    }else {
        // 说明是要拒绝掉这个投稿，只需要将数据库中这个投稿的审核状态改为拒绝即可，
        // 不需要发送给用户增加积分的MQ消息
        this.auditById(id,auditDTO);
    }

    return share;
}
```

**这一步的改动是去掉了原有的`@Transactional`注解，并将原有的直接发送`RocketMQ`消息改为了如果审核状态是`PASS`即需要给用户增加积分的情况，向`RocketMQ`发送`半消息`，除了`半消息`之外，还在`RocketMQ`消息的`Header`里设置了这个`RocketMQ`事务消息的的`事务ID`，便于后续确定事务，以及在`Header`中添加了`投稿id`、以及添加了`ShareAuditDTO`作为参数传递到`RocketMQ`，便于之后的执行本地事务获取审核数据。这一步是完成了上面RocketMQ事务消息原理图中的第一步：发送半消息。**)



然后添加`RocketMQ`分布式事务处理类：

```java
package tech.punklu.contentcenter.rocketmq;

import org.apache.rocketmq.spring.annotation.RocketMQTransactionListener;
import org.apache.rocketmq.spring.core.RocketMQLocalTransactionListener;
import org.apache.rocketmq.spring.core.RocketMQLocalTransactionState;
import org.apache.rocketmq.spring.support.RocketMQHeaders;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import tech.punklu.contentcenter.dao.message.RocketmqTransactionLogMapper;
import tech.punklu.contentcenter.domain.dto.content.ShareAuditDTO;
import tech.punklu.contentcenter.domain.dto.message.RocketmqTransactionLog;
import tech.punklu.contentcenter.service.content.ShareService;

/**
 * 用于对添加积分功能实现RocketMQ调用内容中心：
 * 1、执行本地事务
 * 2、检查本地事务执行状态
 */
@RocketMQTransactionListener
public class AddBonusTransactionListener implements RocketMQLocalTransactionListener {

    @Autowired
    private ShareService shareService;

    @Autowired
    private RocketmqTransactionLogMapper rocketmqTransactionLogMapper;

    /**
     * 执行本地事务
     * @param message 生产者（内容中心发送的半消息）
     * @param o 生产发送半消息时和半消息一起发送的参数对象
     * @return
     */
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message message, Object o) {
        MessageHeaders headers = message.getHeaders();
        // 获取之前生产者（内容中心）发送半消息时指定的分布式事务id,确定本次的事务
        String transactionId = (String) headers.get(RocketMQHeaders.TRANSACTION_ID);
        // 获取之前生产者（内容中心）发送半消息时放在header中的投稿id
        Integer shareId = Integer.valueOf((String)headers.get("share_id"));

        // 尝试执行生产者（内容中心）的本地事务
        try{
            this.shareService.auditByIdWithRocketMqLog(shareId, (ShareAuditDTO) o,transactionId);
            // 有可能失败的地方，可能在执行完本地事务后，还没来得及告诉RocketMQ本地事务已完成，
            // 生产者就挂掉了，或者网络暂时不通了，所以需要
            // 有下面的RocketMQ回查生产者本地事务状态的接口(checkLocalTransaction)
            return RocketMQLocalTransactionState.COMMIT;
        }catch (Exception e){
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }

    /**
     * rocketMQ回查本地事务是否执行成功的接口
     * @param message
     * @return
     */
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message message) {
        MessageHeaders headers = message.getHeaders();
        String transactionId = (String)headers.get(RocketMQHeaders.TRANSACTION_ID);
        // 根据生产者发送半消息时指定的事务id从数据库查询这个事务的执行记录
        RocketmqTransactionLog transactionLog = this.rocketmqTransactionLogMapper.selectOne(
            RocketmqTransactionLog.builder()
                    .transactionId(transactionId)
                    .build()
        );
        // 如果查询到了这条记录，说明本地事务已执行成功，否则，说明执行失败,通过RocketMQ回滚这条消息
        if (transactionLog != null) {
            return RocketMQLocalTransactionState.COMMIT;
        }
        return RocketMQLocalTransactionState.ROLLBACK;
    }
}

```

可以看到这个类实现了`RocketMQLocalTransactionListener`及其中的两个方法，两个方法的作用分别是：

1. `RocketMQ`接收到半消息后调用`生产者`执行本地事务

   


   这一步是完成了上面RocketMQ事务消息原理图中的第三步：`RocketMQ接收到半消息后`调用执行`生产者`的本地事务。可以看到，在执行本地事务的过程中，还向`rocketmq_transaction_log`分布式事务执行记录表插入了一条事务成功执行的记录，用于后续`RocketMQ`回查事务执行状态时查询。同时，这一步会根据本地事务的执行结果返回通知`RocketMQ`本地事务执行的状态是成功还是失败。

2. 被`RocketMQ`回查本地事务执行状态，避免因为网络波动等原因导致实际上本地事务已成功执行，但没有通知`RocketMQ`将半消息转换为`正式的可被消费的消息`的情况。

   


   这一步是完成了上面RocketMQ事务消息原理图中的第五、六步：`RocketMQ长时间未接收到提交/回滚半消息的请求时或接收到本地事务成功执行的返回结果时再次回查生产者的本地事务执行状态`。经过这一步之后，`RocketMQ`就会将`半消息`提交为可被消费者消费的消息。



本地事务执行的代码中涉及到的执行`更新投稿状态`和`写入事务执行成功结果`的日志的代码如下：

```java
/**
* 更新投稿消息审核状态的变化
* @param id
* @param auditDTO
*/
@Transactional(rollbackFor = Exception.class)
public void auditByIdInDB(Integer id,ShareAuditDTO auditDTO) {
    Share share = Share.builder()
        .id(id)
        .auditStatus(auditDTO.getAuditStatusEnum().toString())
        .reason(auditDTO.getReason())
        .build();
    this.shareMapper.updateByPrimaryKeySelective(share);
}

/**
* 先执行本地事务，再记录事务成功执行的信息在数据库表里，提供给RocketMQ回查
* @param id
* @param auditDTO
* @param transactionId
*/
@Transactional(rollbackFor = Exception.class)
public void auditByIdWithRocketMqLog(Integer id,
    								ShareAuditDTO auditDTO,
    								String transactionId){
    // 更新数据库的投稿状态
    this.auditByIdInDB(id,auditDTO);
    // 写入本地事务执行成功的日志，用于后续RocketMQ回查确认
    this.rocketmqTransactionLogMapper.insertSelective(
        RocketmqTransactionLog.builder()
        .transactionId(transactionId)
        .log("审核分享...")
        .build()
    );
}
```



其中非常关键的一点就是将`写入本地事务执行成功的日志`和`更新数据库投稿状态`的代码放在同一个事务中。




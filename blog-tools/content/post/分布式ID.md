---
title: "分布式ID" 
date: 2020-12-31T12:29:39+08:00
draft: false
tags: ["分布式","笔记","JAVA"]
categories:
  - "分布式"
  - "笔记"
  - "JAVA"
---



# 分布式ID



## 实现方式



### 号段模式

每次批量获取ID，将ID列表缓存在本地，提升效率。不需要每次都去请求ID。直到当前号段用完了，才去获取下一号段。



### 雪花算法

例子：

| 1 bit符号位 | 41 bit时间戳位                            | 10 bit工作进程位 | 12 bit序列号位 |
| ----------- | ----------------------------------------- | ---------------- | -------------- |
| 0           | 00000000000000000000000000000000000000000 | 0000000000       | 000000000000   |

因为ID都是正数，所以符号位舍弃，时间戳位是时间戳的二进制表示形式。工作进程位表示机器序号。比如前五位表示服务器区域ID，后五位是服务器的标识。序列号位可以使用自增id。这样就可以保证分布式ID的唯一性及高可用。



## 使用Leaf

Leaf是美团开源的分布式ID生成组件，同时支持号段模式和雪花算法模式。



### 编译Leaf

目前Leaf尚未提供官方MAVEN仓库地址，所以需要自己下载源码编译。

```
git clone git@github.com:Meituan-Dianping/Leaf.git
cd Leaf
git checkout feature/spring-boot-starter
mvn clean install -Dmaven.test.skip=true 
```



编译完成后可在本地MAVEN仓库找到对应的JAR包，即可引入依赖并使用。



### 引入依赖

```xml
<dependency>
	<artifactId>leaf-boot-starter</artifactId>
    <groupId>com.sankuai.inf.leaf</groupId>
    <version>1.0.1-RELEASE</version>
</dependency>
```



在classpath下创建leaf.properties配置文件：

```
leaf.name=com.sankuai.leaf.opensource.test
leaf.segment.enable=false
#leaf.segment.url=
#leaf.segment.username=
#leaf.segment.password=

leaf.snowflake.enable=false
#leaf.snowflake.address=
#leaf.snowflake.port=
```



### 启动类Leaf注解

不管是使用号段模式还是使用雪花算法模式，都需要在启动类上标注上LeafServer注解：

```java
import com.sankuai.inf.leaf.plugin.annotation.EnableLeafServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableLeafServer
public class DistributedIdApplication {

    public static void main(String[] args) {
        SpringApplication.run(DistributedIdApplication.class,args);
    }
}
```



### 使用号段模式

如果使用号段模式，需要建立DB表，并配置leaf.jdbc.url, leaf.jdbc.username, leaf.jdbc.password

如果不想使用该模式配置leaf.segment.enable=false即可。



号段模式所需的数据表：

```sql
CREATE DATABASE leaf;
CREATE TABLE `leaf_alloc` (
  `biz_tag` varchar(128)  NOT NULL DEFAULT '',
  `max_id` bigint(20) NOT NULL DEFAULT '1',
  `step` int(11) NOT NULL,
  `description` varchar(256)  DEFAULT NULL,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`biz_tag`)
) ENGINE=InnoDB;

insert into leaf_alloc(biz_tag, max_id, step, description) values('leaf-segment-test', 1, 2000, 'Test leaf Segment Mode Get Id');
```

其中`biz_tag`是获取`id`需要使用的`key`，`step`是每次从数据库获取缓存的id个数。

如果需要使用多个号段ID，只需要多配置几条记录并使用对应的`biz_tag`作为key即可。



在leaf.properties中配置leaf.jdbc.url, leaf.jdbc.username, leaf.jdbc.password参数



然后如下使用即可：

```java
import com.sankuai.inf.leaf.common.Result;
import com.sankuai.inf.leaf.service.SegmentService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IdController {

    @Autowired
    private SegmentService segmentService;

    @GetMapping("/segment")
    public Result segment(){
        // key值为数据库中的biz_tag的值
        return segmentService.getId("leaf-segment-test");
    }

}
```



### 使用雪花算法

Leaf的雪花算法需要依赖`Zookeeper`，因为需要通过把IP地址存在ZK中的方式来生成对应的机器位数值。



引入ZK依赖：

```xml
<!--zk-->
<dependency>
	<groupId>org.apache.curator</groupId>
	<artifactId>curator-recipes</artifactId>
	<version>2.6.0</version>
	<exclusions>
		<exclusion>
			<artifactId>log4j</artifactId>
			<groupId>log4j</groupId>
			</exclusion>
		</exclusions>
</dependency>
```



在leaf.properties中配置雪花算法及ZK：

```
# 启用雪花算法
leaf.snowflake.enable=true
# ZK地址
leaf.snowflake.address=zk.springboot.cn/
# ZK端口
leaf.snowflake.port=2181
```



使用雪花算法生成ID：

```java
import com.sankuai.inf.leaf.common.Result;
import com.sankuai.inf.leaf.service.SnowflakeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IdController {

    @Autowired
    private SnowflakeService snowflakeService;


    @GetMapping("snowflake")
    public Result snowflake(){
        return snowflakeService.getId("key");
    }

}
```



因为雪花算法依赖于将IP保存在ZK中，所以一台机器不可以用来生成多个分布式ID，否则会有产生相同ID的可能。




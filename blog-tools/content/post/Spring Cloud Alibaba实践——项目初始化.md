---
title: "Spring Cloud Alibaba实践——项目初始化" 
date: 2021-01-24T12:29:39+08:00
draft: false
tags: ["Spring Cloud","笔记","JAVA"]
categories:
  - "Spring Cloud"
  - "笔记"
  - "JAVA"
---



# Spring Cloud Alibaba实践——项目初始化

## 项目目标

搭建一个由Spring Cloud Alibaba开发的包含投稿、分享、积分等功能的微服务项目。



## 创建数据库表结构

分为内容中心和用户中心两个数据库：

1. 用户中心建表SQL

   ```sql
   CREATE DATABASE user_center;
   USE `user_center`;
   
   -- -----------------------------------------------------
   -- Table `user`
   -- -----------------------------------------------------
   CREATE TABLE IF NOT EXISTS `user` (
     `id` INT NOT NULL AUTO_INCREMENT COMMENT 'Id',
     `wx_id` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '微信id',
     `wx_nickname` VARCHAR(64) NOT NULL DEFAULT '' COMMENT '微信昵称',
     `roles` VARCHAR(100) NOT NULL DEFAULT '' COMMENT '角色',
     `avatar_url` VARCHAR(255) NOT NULL DEFAULT '' COMMENT '头像地址',
     `create_time` DATETIME NOT NULL COMMENT '创建时间',
     `update_time` DATETIME NOT NULL COMMENT '修改时间',
     `bonus` INT NOT NULL DEFAULT 300 COMMENT '积分',
     PRIMARY KEY (`id`))
   COMMENT = '分享';
   
   
   -- -----------------------------------------------------
   -- Table `bonus_event_log`
   -- -----------------------------------------------------
   CREATE TABLE IF NOT EXISTS `bonus_event_log` (
     `id` INT NOT NULL AUTO_INCREMENT COMMENT 'Id',
     `user_id` INT NULL COMMENT 'user.id',
     `value` INT NULL COMMENT '积分操作值',
     `event` VARCHAR(20) NULL COMMENT '发生的事件',
     `create_time` DATETIME NULL COMMENT '创建时间',
     `description` VARCHAR(100) NULL COMMENT '描述',
     PRIMARY KEY (`id`),
     INDEX `fk_bonus_event_log_user1_idx` (`user_id` ASC) )
   ENGINE = InnoDB
   COMMENT = '积分变更记录表';
   ```

2. 内容中心建表SQL

   ```sql
   CREATE DATABASE content_center;
   USE `content_center`;
   
   -- -----------------------------------------------------
   -- Table `share`
   -- -----------------------------------------------------
   CREATE TABLE IF NOT EXISTS `share` (
     `id` INT NOT NULL AUTO_INCREMENT COMMENT 'id',
     `user_id` INT NOT NULL DEFAULT 0 COMMENT '发布人id',
     `title` VARCHAR(80) NOT NULL DEFAULT '' COMMENT '标题',
     `create_time` DATETIME NOT NULL COMMENT '创建时间',
     `update_time` DATETIME NOT NULL COMMENT '修改时间',
     `is_original` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否原创 0:否 1:是',
     `author` VARCHAR(45) NOT NULL DEFAULT '' COMMENT '作者',
     `cover` VARCHAR(256) NOT NULL DEFAULT '' COMMENT '封面',
     `summary` VARCHAR(256) NOT NULL DEFAULT '' COMMENT '概要信息',
     `price` INT NOT NULL DEFAULT 0 COMMENT '价格（需要的积分）',
     `download_url` VARCHAR(256) NOT NULL DEFAULT '' COMMENT '下载地址',
     `buy_count` INT NOT NULL DEFAULT 0 COMMENT '下载数 ',
     `show_flag` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否显示 0:否 1:是',
     `audit_status` VARCHAR(10) NOT NULL DEFAULT 0 COMMENT '审核状态 NOT_YET: 待审核 PASSED:审核通过 REJECTED:审核不通过',
     `reason` VARCHAR(200) NOT NULL DEFAULT '' COMMENT '审核不通过原因',
     PRIMARY KEY (`id`))
   ENGINE = InnoDB
   COMMENT = '分享表';
   
   
   -- -----------------------------------------------------
   -- Table `mid_user_share`
   -- -----------------------------------------------------
   CREATE TABLE IF NOT EXISTS `mid_user_share` (
     `id` INT NOT NULL AUTO_INCREMENT,
     `share_id` INT NOT NULL COMMENT 'share.id',
     `user_id` INT NOT NULL COMMENT 'user.id',
     PRIMARY KEY (`id`),
     INDEX `fk_mid_user_share_share1_idx` (`share_id` ASC) ,
     INDEX `fk_mid_user_share_user1_idx` (`user_id` ASC) )
   ENGINE = InnoDB
   COMMENT = '用户-分享中间表【描述用户购买的分享】';
   
   
   -- -----------------------------------------------------
   -- Table `notice`
   -- -----------------------------------------------------
   CREATE TABLE IF NOT EXISTS `notice` (
     `id` INT NOT NULL AUTO_INCREMENT COMMENT 'id',
     `content` VARCHAR(255) NOT NULL DEFAULT '' COMMENT '内容',
     `show_flag` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否显示 0:否 1:是',
     `create_time` DATETIME NOT NULL COMMENT '创建时间',
     PRIMARY KEY (`id`));
   ```





## Spring Boot版本的项目雏形

一个很简单的单纯使用`Spring Boot`搭建的项目雏形，代码地址：[Spring Boot Project](https://github.com/punk1u/Spring-Cloud-Alibaba-Demo/tree/main/spring-boot-project)。

后续的有关`Spring Cloud Alibaba`的开发都基于这个雏形项目。
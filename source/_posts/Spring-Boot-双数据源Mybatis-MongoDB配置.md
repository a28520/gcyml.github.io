---
title: Spring Boot 双数据源Mybatis+MongoDB配置
date: 2018-05-18 13:35:09
categories: "Srping Boot" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Spring Boot
     - Mybatis
---

## 前言

最近有个项目需要用到结构化的数据和非结构化的数据，于是选择了mysql和mongoDb。整个项目是基于Spring Boot创建的，相比于Spring MVC，Spring Boot集成了常用的第三方依赖库，具有搭建迅速，配置更少的优点。
<!-- more -->

## 技术栈

* Spring Boot
* Mybatis
* MongoDB
* Mysql

## 正文

添加相关第三方依赖

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>

	<!-- mysql依赖 -->
	<dependency>
		<groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
	</dependency>

	<dependency>
		<groupId>org.mybatis.spring.boot</groupId>
		<artifactId>mybatis-spring-boot-starter</artifactId>
		<version>${mybatis.version}</version>
	</dependency>

	<!-- mongodb -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-mongodb</artifactId>
	</dependency>
</dependencies>
```

.yml 配置

> 此处和普通Spring Boot + Mybatis项目最大不同在于在data节点加入了MongoDB的相关参数，后面会指定MongoDB数据扫描指定的DAO层位置。

``` yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/dbName?characterEncoding=UTF-8&useSSL=true
    username: mysqlUsername
    password: mysqlPassword
  data:
    mongodb:
     
      host: 127.0.0.1
      port: 12345
      username: mongoUsername
      password: mongoPassword
      database: database
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.multi.datasource.dao.mysql
```

主启动类

> 这里关键添加了@EnableMongoRepositories("com.multi.datasource.dao.mongo")这行配置，设置MnogoDB的DAO层扫描路径。

``` java
@SpringBootApplication
@EnableMongoRepositories("com.multi.datasource.dao.mongo")
public class Application {
	public static void main(String[] args) throws Exception {
         SpringApplication.run(Application.class, args);
     }
}
```

## 结语

总体思路是通过不同数据源扫描不同路径的DAO层实现，Mybatis
和MongDB双数据源的配置还是比较简单的。在此只是简单做个记录。
---
title: Spring-boot+Mybatis+Maven+MySql搭建实例
date: 2018-05-18 15:26:27
categories: "Srping Boot" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Spring Boot
     - Maven
     - Mybatis
---
## 前言
最近读了spring-boot开发手册，spring-boot相比于spring-mvc封装了很多常用的依赖，并且内置了tomcat容器，启动生成的jar包即可启动项目，也是目前流行的微服务常用框架。本文主要用到了spring-boot，以及mybatis，数据库用到了mysql。
[源码下载](http://download.csdn.net/download/baidu_18987603/10246161)  

## 准备工作

1.首先创建一个表：

``` sql
CREATE TABLE `t_user` (  
  `USER_ID` int(11) NOT NULL AUTO_INCREMENT,  
  `USER_NAME` char(30) NOT NULL,  
  `USER_PASSWORD` char(10) NOT NULL,  
  `USER_EMAIL` char(30) NOT NULL,  
  PRIMARY KEY (`USER_ID`),  
  KEY `IDX_NAME` (`USER_NAME`)  
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8  
```

插入一些数据：

``` sql
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (1, '林炳文', '1234567@', 'ling20081005@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (2, 'evan', '123', 'fff@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (3, 'kaka', 'cadg', 'fwsfg@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (4, 'simle', 'cscs', 'fsaf@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (5, 'arthur', 'csas', 'fsaff@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (6, '小德', 'yuh78', 'fdfas@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (7, '小小', 'cvff', 'fsaf@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (8, '林林之家', 'gvv', 'lin@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (9, '林炳文Evankaka', 'dfsc', 'ling2008@126.com');  
INSERT INTO t_user (USER_ID, USER_NAME, USER_PASSWORD, USER_EMAIL) VALUES (10, 'apple', 'uih6', 'ff@qq.com'); 
```

## 创建工程

我习惯于先创建好maven项目，构建目录再导入到编译器中，这样的好处就是搭建好一个脚手架模板，后面改改参数就可以用到各个工程里面。

### 构建pom.xml

``` xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.boot</groupId>
	<artifactId>testSpringBoot</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>testSpringBoot</name>
	<packaging>jar</packaging>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/>
	</parent>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		
		<!-- mybatis依赖 -->
		<dependency>
		    <groupId>org.mybatis.spring.boot</groupId>
		    <artifactId>mybatis-spring-boot-starter</artifactId>
		    <version>1.3.1</version>
		</dependency>
		
		<!-- 引入thymelaf 则不需要引入web依赖，若不需要thymelaf则需要添加spring-boot-starter-web -->
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		
		<!-- mysql依赖 -->
	    <dependency>
	        <groupId>mysql</groupId>
	        <artifactId>mysql-connector-java</artifactId>
	    </dependency>
	    
	     <dependency>
	      <groupId>org.springframework.boot</groupId>
	      <artifactId>spring-boot-starter-test</artifactId>
	    </dependency>
	</dependencies>
	<build>
	<plugins>
		<!-- 要使生成的jar可运行，需要加入此插件 -->
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
</project>
```

### 工程目录

创建的文件目录如图：
![工程目录](http://upload-images.jianshu.io/upload_images/2837760-c420f2d2633efd5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 启动器以及controller

**（1） 启动器**
在com.boot(即最外层目录文件)下写一个如下main方法:

``` java
package com.boot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication 
public class Application {
	public static void main(String[] args) throws Exception {
         SpringApplication.run(Application.class, args);
     }
}
```

**（2）controller**
在com.boot.web下创建一个类如下:

``` java
package com.boot.web;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class MainController {
    @RequestMapping(value = "/")
    String home() {
    return "Hello World!";
    }
}
```

**（3）启动项目**
找到com.boot下的Application以java Application方式启动，然后打开浏览器输入localhost:8080就会出现Hello World!
![启动页面](http://upload-images.jianshu.io/upload_images/2837760-e0f7bb853152ffc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 添加java代码

**（1）User.java**
对应数据库中表的字段，放在src/main/java下的包com.boot.domain

``` java
package com.boot.domain;

public class User {  
    private Integer userId;  
    private String userName;  
    private String userPassword;  
    private String userEmail;  
  
    public Integer getUserId() {  
        return userId;  
    }  
  
    public void setUserId(Integer userId) {  
        this.userId = userId;  
    }  
  
    public String getUserName() {  
        return userName;  
    }  
  
    public void setUserName(String userName) {  
        this.userName = userName;  
    }  
  
    public String getUserPassword() {  
        return userPassword;  
    }  
  
    public void setUserPassword(String userPassword) {  
        this.userPassword = userPassword;  
    }  
  
    public String getUserEmail() {  
        return userEmail;  
    }  
  
    public void setUserEmail(String userEmail) {  
        this.userEmail = userEmail;  
    }  
  
    @Override  
    public String toString() {  
        return "User [userId=" + userId + ", userName=" + userName  
                + ", userPassword=" + userPassword + ", userEmail=" + userEmail  
                + "]";  
    }  
      
}  

```

**（2）UserDao.java**

Dao接口类，用来对应mapper文件。放在src/main/java下的包com.lin.dao,内容如下：

``` java
package com.boot.dao;

import org.apache.ibatis.annotations.Mapper;

import com.boot.domain.User;
@Mapper
public interface UserDao {  
    public User selectUserById(Integer userId);  
  
}  

```

**（3）UserService.java和UserServiceImpl.java**

service接口类和实现类，放在src/main/java下的包com.lin.service,内容如下：

UserService.java

``` java
package com.boot.service;

import com.boot.domain.User;

public interface UserService {
	User selectUserById(Integer userId); 
}
```

UserServiceImpl.java

``` java
package com.boot.service.impl;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.boot.dao.UserDao;
import com.boot.domain.User;
import com.boot.service.UserService;
@Service
public class UserServiceImpl implements UserService{
	@Autowired  
	private UserDao userDao;  
	
	@Override
	public User selectUserById(Integer userId) {
		return userDao.selectUserById(userId);  
	}

}
```

**（4）mapper文件**

用来和dao文件对应，放在src/main/resources中的com/boot/mapper目录下
![UserMapper.xml路径](http://upload-images.jianshu.io/upload_images/2837760-c7768a67a18d5586.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

UserMapper.xml

``` xml
<?xml version="1.0" encoding="UTF-8"?>    
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"    
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.boot.dao.UserDao">  
<!--设置domain类和数据库中表的字段一一对应，注意数据库字段和domain类中的字段名称不致，此处一定要！-->  
    <resultMap id="BaseResultMap" type="com.boot.domain.User">  
        <id column="USER_ID" property="userId" jdbcType="INTEGER" />  
        <result column="USER_NAME" property="userName" jdbcType="CHAR" />  
        <result column="USER_PASSWORD" property="userPassword" jdbcType="CHAR" />  
        <result column="USER_EMAIL" property="userEmail" jdbcType="CHAR" />  
    </resultMap>  
    <!-- 查询单条记录 -->  
    <select id="selectUserById" parameterType="int" resultMap="BaseResultMap">  
        SELECT * FROM t_user WHERE USER_ID = #{userId}  
    </select>  
</mapper> 
```

## 资源配置

下列所有文件均在src/main/resources目录下
**（1）spring-boot配置**
不少人都Properties资源文件来配置，不过这种文件在eclipse编码的默认设置是ISO-8859-1，需要修改eclipse的设置才能显示中文。因此我比较喜欢用yml文件来配置，一个是结构明显，另外一个不用考虑编码的问题。

application.yml

``` yml
spring:
  thymeleaf:
    mode: HTML5
    encoding: UTF-8
    content-type: text/html
    # 开发禁用缓存
    cache: false
  datasource:
    url: jdbc:mysql://localhost:3306/ssmDb?characterEncoding=UTF-8&useSSL=true
    username: root
    password: password
    # 可省略驱动配置, sprin-boot 会由url检测出驱动类型
    # driver-class-name: com.mysql.jdbc.Driver
    hikari:
      connection-timeout: 60000 
mybatis:
  mapperLocations: classpath:com/boot/mapper/*.xml
  typeAliasesPackage: cn.boot.domain
# spring-boot默认打印输出info级别以上的，可在此处修改输出级别
logging:
  level:
    root: info
```

**（2）日志打印logback-spring.xml**

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

	<!-- 控制台输出 -->
	<appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%date [%thread] %-5level %logger{80} - %msg%n</pattern>
		</encoder>
	</appender>

	<!-- 时间滚动输出 level为 INFO 日志 -->
	<appender name="file-info" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>INFO</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY </onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>./logs/info.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>30</MaxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>%date [%thread] %-5level %logger{80} - %msg%n</pattern>
		</encoder>
	</appender>

	<!-- 时间滚动输出 level为 DEBUG 日志 -->
	<appender name="file-debug" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>DEBUG</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY </onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>./logs/debug.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>30</MaxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>%date [%thread] %-5level %logger{80} - %msg%n</pattern>
		</encoder>
	</appender>

	<!-- 时间滚动输出 level为 ERROR 日志 -->
	<appender name="file—error" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>ERROR</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY </onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>./logs/error.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>30</MaxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>%date [%thread] %-5level %logger{80} - %msg%n</pattern>
		</encoder>
	</appender>

	<logger name="org.apache.ibatis" level="INFO"/>
    <logger name="java.sql.Connection" level="debug" />
    <logger name="java.sql.Statement" level="debug" />
    <logger name="java.sql.PreparedStatement" level="debug" />

	<root level="info">
		<appender-ref ref="stdout" />
		<appender-ref ref="file-info" />
	</root>
</configuration>
```

## 单元测试

spring-boot单元测试只需引入spring-boot-starter-test即可，里面集成了Junit以及Spring Test & Spring Boot Test等依赖。
![测试类目录](http://upload-images.jianshu.io/upload_images/2837760-e9cb88abf30402eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**（1）测试基类**

``` java
package com.boot.baseTest;

import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public abstract class SpringTestCase {  
	Logger logger = LoggerFactory.getLogger(this.getClass());
} 
```

**（2）测试类**

``` java
package com.boot.serviceTest;

import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;

import com.boot.baseTest.SpringTestCase;
import com.boot.domain.User;
import com.boot.service.UserService;

public class UserServiceTest extends SpringTestCase{
	Logger logger = LoggerFactory.getLogger(this.getClass());
	@Autowired  
    private UserService userService; 
	@Test  
    public void selectUserByIdTest(){  
        User user = userService.selectUserById(10);  
        logger.info("查找结果" + user);  
    }  

}
```

选中测试类右键run as选中Junit Test即可，可以看到打印输出了id为10的User对象
>2018-02-04 21:20:07,442 [main] INFO  com.boot.serviceTest.UserServiceTest - 查找结果User [userId=10, userName=apple, userPassword=uih6, userEmail=ff@qq.com]

![数据库数据](http://upload-images.jianshu.io/upload_images/2837760-886eb4ea3fb52016.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
若看到正常输出，此时可以开始配置web页面了。

## web页面配置

**（1）静态资源**
静态资源放在src/main/resources/static目录下
![图片.png](http://upload-images.jianshu.io/upload_images/2837760-9309700417afcc8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**（2）index.html**
spring-boot支持thymeleaf模板引擎，模板文件默认放在src/main/resources/templates目录下

``` html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta content="text/html;charset=UTF-8"/>
    <meta name="viewport" content="width=device-width,initial-scale=1"/>
    <link th:href="@{/js/jquery-1.11.3.min.js}" rel="stylesheet"/>
</head>
<body>  
	<h2>Hello World!</h2>  
    <p th:text="${user.userId}"></p>
    <p th:text="${user.userName}"></p>
    <p th:text="${user.userPassword}"></p>
    <p th:text="${user.userEmail}"></p>
</body>  
</html>  
```

xmlns:th="http://www.thymeleaf.org"命名空间，将镜头转化为动态的视图，需要进行动态处理的元素使用“th:”前缀；link引入jquery框架，通过@{}引入web静态资源（括号里面是资源路径）访问model中的数据通过${}访问。

**（4）自定义错误页面**

spring-boot错误页面默认在src/main/resources/static/error或者src/main/resources/public/error下，在此简单写了个404页面，命名为404.html放到目录下即可

404.html

``` html
<!DOCTYPE html>
<html >
<body>  
	<h2>404 not found</h2>  
</body>  
</html>  
```

**（5）controller**

``` java
package com.boot.web;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

import com.boot.domain.User;
import com.boot.service.UserService;


@Controller
@RequestMapping("/user")
public class UserController {
	@Autowired 
	private UserService userService;  
	@RequestMapping(value = "")
	public String getIndex(Model model){      
        User user = userService.selectUserById(1);  
        model.addAttribute("user", user);   
        return "index";    
    } 
}
```

**（6）启动工程**

启动工程以后打开http://localhost:8080/user即可看到如下图：
![index页面](http://upload-images.jianshu.io/upload_images/2837760-bc00ceb58f411891.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

若打开任意不存在的页面，会返回404.html中内容
![404](http://upload-images.jianshu.io/upload_images/2837760-882d2e6d83ddef5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 打包工程
使用package命令给工程打包成jar包

``` 
mvn package
```

此时会在target下生成一个jar包，启动即可：

```
java -jar target\testSpringBoot-0.0.1-SNAPSHOT.jar
```

输出如下

```
λ java -jar target\testSpringBoot-0.0.1-SNAPSHOT.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.9.RELEASE)

```
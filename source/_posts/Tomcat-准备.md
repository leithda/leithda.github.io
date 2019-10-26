---
title: Tomcat-准备
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 1805298928
date: 2019-10-26 00:00:00
---


> 教程来自 [《How Tomcat Works》]()
>
> 书中讨论tamcat版本较低，只是用于学习其编码设计。

<!-- More -->
#  学习前准备

1. 创建 maven 项目[`how-tomcat-works`]
2. pom文件如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.leithda</groupId>
    <artifactId>how-tomcat-works</artifactId>
    <version>1.0</version>


    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>servlet-api</artifactId>
            <version>2.5</version>
        </dependency>
        <!-- 引入一下 Tomcat相关Jar 的目的是为了学习:How Tomcat Works这本书 -->
        <dependency>
            <groupId>tomcat</groupId>
            <artifactId>catalina</artifactId>
            <version>4.1.9</version>
        </dependency>
        <dependency>
            <groupId>tomcat</groupId>
            <artifactId>bootstrap</artifactId>
            <version>4.0.6</version>
        </dependency>
        <dependency>
            <groupId>tomcat</groupId>
            <artifactId>naming-resources</artifactId>
            <version>5.5.12</version>
        </dependency>
        <dependency>
            <groupId>tomcat</groupId>
            <artifactId>naming-common</artifactId>
            <version>5.0.28</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate.validator</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>6.0.14.Final</version>
            <scope>compile</scope>
        </dependency>
        <!-- Junit 测试类 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

    </dependencies>

</project>
```

# How Tomcat Works 目录简介

{% asset_img flow.png Tomcat处理请求过程 %}

## 1.[一个简答的Web服务器](1975698977.html)

问题：请求HttpServer没有看到响应的内容

原因：socket输出之前要有HTTP响应头

```java
output.write("HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n".getBytes());
```

## 2.[一个简单的Servlet容器](991341650.html)

面向接口编程

反射

门面模式

## 3. [连接器](2076341340.html)

使用tomcat-util RequestUtil解析请求参数和cookie

实现的Connector是Tomcat 4默认连接器的一个简化版本


## 4.[Tomcat的默认连接器](479209129.html)

HTTP 1.1 的新特性

Tomcat 4默认连接器的原理

## 5.[容器](3459939477.html)

Context, Wrapper Container

容器接到请求后，由Pipeline处理

## 6.[生命周期](1950977268.html)

实现LifeCycle管理组件的生命周期

优雅的启动，关闭关联的组件

## 7.[日志系统](3784073105.html)

FileLogger的实现

## 8.[加载器](4141534283.html)

Tomcat为何要实现自己的加载器

Loader, Reloader接口

WepappLoader, WepappClassLoader

## 9.[Session管理](1571172711.html)
Tomcat 如何管理 Session
Session 的生命周期

## 10.[安全](3630618985.html)
Tomcat 如何控制web程序访问内容
如何进行鉴权

## 11.[StandardWrapper](1604995939.html)

StandardWrapper工作原理,loadServlet过程

## 12.[StandardContext](1923304143.html)

start做了哪些工作

Tomcat 4中组件使用各自的线程来处理一些定时任务,Tomcat 5中为了节省资源所有后台任务共享一个线程


## 13.Host和Engine

使用Host,Engine容器

## 14.Server和Service组件

Server优雅的方式启动,关闭整个catalina

## 15.Digester库


## 16.ShutdownHook

没说和Tomcat有啥关系




---
title: Spring Cloud 之 Eureka-服务注册与发现
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
date: 2019-9-12
abbrlink: 518531817
---

> Eureka 是一个用于服务发现与注册的组件

<!-- More -->
## Eureka Server
1. springboot 项目中引入eureka依赖
```xml
    <!-- Eureka 注册中心 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
```

2. 修改配置文件，进行服务器配置
```yml
# 服务名称
spring:
  application:
    name: eureka
# 服务端口号
server:
  port: 8671
#Eureka 相关配置
eureka:
  client:
    service-url:
      defaultZone: http://localhost:${server.port}/eureka/
    # 是否从其他的服务中心同步服务列表
    fetch-registry: false
    # 是否把自己作为服务注册到其他服务注册中心
    register-with-eureka: false
  server:
    # 自我保护机制，当15分钟内正常心跳占比低于85%会触发，锁定服务列表，使其不失效。
    enable-self-preservation: false  
```

3. boot启动类添加注解，开启`Eureka Server`
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```

4. 启动后访问网页 `http://localhost:8671` 已经可以看见eureka的管理界面

## Eureka Client
1. 添加依赖
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
```

2. 编写配置代码
```yml
# 服务名称
spring:
  application:
    name: eureka-provider
# 服务提供者端口号
server:
  port: 8672
# 配置Eureka Server 信息
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/
  # 自定义实例编号
  instance:
    instance-id: ${spring.application.name}:${server.port}:@project.version@
    # 配置使用主机名注册服务
#    hostname: testnode
    # 优先使用IP地址方式进行注册服务
    prefer-ip-address: true
    # 配置使用指定IP
    ip-address: 127.0.0.1
```

3. 启动类增加注解，开启`Eureka Client`功能
```java
@SpringBootApplication
@EnableDiscoveryClient  // 明确是Eureka可以使用 @EnableEurekaClient
public class EurekaProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class, args);
    }
}
```

4. 启动服务端程序后启动客户端，可以在服务端管理界面中看到注册信息

## Eureka 高可用集群

1. 配置host文件
```text
127.0.0.1       peer1
127.0.0.1       peer2
```

2. 使用多profile或者多项目模式(这里采用多profile模式)
```yml
####  application-peer1.yml ####
# Eureka 服务端配置
eureka:
  client:
    service-url:
      defaultZone: http://peer2:8672/eureka/
  instance:
    # 配置通过主机名方式注册
    hostname: peer1
    # 配置实例编号
    instance-id: ${eureka.instance.hostname}:${server.port}:@project.version@
  # 集群节点之间读取超时时间。单位：毫秒
  server:
    peer-node-read-timeout-ms: 1000
# 服务端口号
server:
  port: 8671
####  application-peer2.yml ####
# Eureka 服务端配置
eureka:
  client:
    service-url:
      defaultZone: http://peer1:8671/eureka/
  instance:
    # 配置通过主机名方式注册
    hostname: peer2
    # 配置实例编号
    instance-id: ${eureka.instance.hostname}:${server.port}:@project.version@
  # 集群节点之间读取超时时间。单位：毫秒
  server:
    peer-node-read-timeout-ms: 1000
# 服务端口号
server:
  port: 8672
```

3. 配置客户端，发布到peer1中
```yml
# 服务名称
spring:
  application:
    name: eureka-provider
# 服务提供者端口号
server:
  port: 8673
# 配置Eureka Server 信息
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/
  # 自定义实例编号
  instance:
    instance-id: ${spring.application.name}:${server.port}:@project.version@
    # 配置使用主机名注册服务
#    hostname: testnode
    # 优先使用IP地址方式进行注册服务
    prefer-ip-address: true
    # 配置使用指定IP
    ip-address: 127.0.0.1
```

4. 测试

`clean && package`当前项目，打开终端进入target项目
执行`java -jar eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=peer1` 开启第一个 Eureka Server，同理开启第二个Servet

运行客户端程序，打开`http://peer1:8671`发现注册节点，打开`http://peer2:8672`发现注册节点，至此集群建设完成.
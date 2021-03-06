---
title: Spring Cloud 之 Config-配置中心
categories:
  - 框架
  - SpringCloud
  - 基础
tags:
  - Spring Cloud
author: 长歌
abbrlink: 1798410625
date: 2019-09-27 20:30:00
---

分布式配置中心Spring Cloud Config

<!-- More -->

## Config Server 从本地读取配置文件
1. 新建Moudle工程[eureka-config-center]
2. 引入依赖
```xml
<!-- cloud 配置中心 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<!-- eureka 客户端 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
3. 配置文件
```yml
spring:
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/shared # 本地配置文件路径
  profiles:
    active: native  # 配置文件加载方式
  application:
    name: config-server # 应用名称
server:
  port: 8680
# 配置Eureka Server 信息
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/  # 默认服务端节点
  # 自定义实例编号
  instance:
    instance-id: ${spring.application.name}:${server.port}
    # 配置使用主机名注册服务
    #    hostname: testnode
    # 优先使用IP地址方式进行注册服务
    prefer-ip-address: true
    # 配置使用指定IP
    ip-address: 127.0.0.1
```

4. 在`resources`目录下新建`shared`目录用于存放配置文件，并新建配置文件`eureka-client-dev.yml`
```yml
# 配置Eureka Server 信息
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/  # 默认服务端节点
  # 自定义实例编号
  instance:
    instance-id: ${spring.application.name}:${server.port}
    # 配置使用主机名注册服务
    #    hostname: testnode
    # 优先使用IP地址方式进行注册服务
    prefer-ip-address: true
    # 配置使用指定IP
    ip-address: 127.0.0.1
foo: foo version 1 # 配置中心测试用变量
```

5. `eureka-client`工程中，增加依赖[`spring-cloud-config-server`],删除`application.yml`,新建`bootstrap.yml`如下
```yml
# bootstrap.yml配置文件相对于application更优先
spring:
  application:
    name: eureka-client
  cloud:
    config:
      uri: http://localhost:8680  # 配置中心服务端地址
      fail-fast: true
  profiles:
    active: dev
# 最终读取的配置文件为 {spring.application.name}-{spring.profiles.active}.yml
# 即 eureka-client-dev.yml
# 服务提供者端口号
server:
  port: 8673
```

6. 在`HiController`中增加测试方法
```java
@RestController
public class HiController {

    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String home(@RequestParam String name){
        return " hi "+name+", l am from port : " +port;
    }

    @Value("${foo}")
    String foo;

    @RequestMapping(value="/foo")
    public String testConfig(){
        return foo;
    }
}
```

7. 测试
- 启动Config-server
- 启动eureka-server,eureka-client
- 访问`http://localhost:8671`，查看`eureka-client`是否启动
- 访问`http://localhost:8673/foo`,查看配置文件是否读取成功


## Config Server 从git仓库读取配置文件
> 主要是config-server的配置文件的修改，配置文件如下

```yml
server:
  port: 9680
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/username/repoName # username 为用户名, repoName为仓库名
          search-paths: respo
          username: username
          password:
        default-label: master # 分支
  application:
    name: config-server
```

## 构建高可用的 Config Server
通过Eureka 注册多个Config Server,其他服务获取配置文件时，会轮流从服务端获取


## 使用 Spring Cloud Bus 刷新配置
> 使用方法待续 ...


配置中心相关文章

- [Java后端技术-为什么不用原生的Spring Cloud Config!](https://mp.weixin.qq.com/s?__biz=MzI1NDQ3MjQxNA==&mid=2247488749&idx=1&sn=342f6c87963ce12afea99edb975bf09d&chksm=e9c5ed5cdeb2644ae14e63e0d57539fde63db6fd641322967149613fa32639c7408f869553cc&mpshare=1&scene=1&srcid=&key=f6869c76f8fd06b84a971a4bb4e95e0ee1418209652c81c5d52aa1b62925340c93db14b5181d64f10c3530a7203524202cf3fb366405ce7b8aeb331458846a85423a818fc4526f09076a1762b3b61061&ascene=1&uin=MjIwOTU0OTc2MQ%3D%3D&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=4SHwNuXGEl9e6IyR6hVpph%2FZaUUaM2jZdZVML95LvoXudyGDdG1GWml9sB%2BNOrCe)
- [芋道源码-深度对比三种主流微服务配置中心](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247486679&idx=1&sn=70bc4042528703b1d1d53746f14a3063&chksm=fa497366cd3efa70a57fdc17d970f32e28582961c65f7d2cedb521403edf41abdbbeef3e17ee&mpshare=1&scene=1&srcid=&key=5064705dbe24d98836f97c6cc78abb16a296cb61458a39711a0974fe488f9adc4e888c50f4eea88951ca01bf7e42502c4bddb40a45fdc2937dbf2214427e50c1de74ba0d7aa27334dd33ad7717d2ac6d&ascene=1&uin=MjIwOTU0OTc2MQ%3D%3D&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=4SHwNuXGEl9e6IyR6hVpph%2FZaUUaM2jZdZVML95LvoXudyGDdG1GWml9sB%2BNOrCe)

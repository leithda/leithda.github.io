---
title: Spring Cloud 之 Zipkin-链路追踪
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
abbrlink: 3833359923
date: 2019-09-29 07:45:00
---

在分布式系统中提供链路追踪
<!-- More -->

## Zipkin
### 作用
- 确认一个请求有哪些服务参与
- 服务耗时分析
- 服务异常定位

### 搭建Zipkin服务
- 官网提供3种方式进行服务搭建
1. 使用docker启动
```sh
docker run -d -p 9411:9411 openzipkin/zipkin
```

2. 下载jar包，通过`java`启动
```sh
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

3. 编译源码启动
```sh
# get the latest source
git clone https://github.com/openzipkin/zipkin
cd zipkin
# Build the server and also make its dependencies
./mvnw -DskipTests --also-make -pl zipkin-server clean install
# Run the server
java -jar ./zipkin-server/target/zipkin-server-*exec.jar
```

4. 启动后，通过访问`localhost:9411`确认服务是否启动成功
- 其中`docker`镜像启动时,参数`-p 9411:9411`不可忽略，否则可能导致9411端口访问异常



### 使用 `zipkin` 服务
1. 在需要进行追踪的服务中引入依赖
```xml
        <!-- zipkin 链路追踪 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```

2. 配置 `zipkin` 服务端地址
```yml
spring:
  application:
    name: service-zuul
  # zipkin 配置
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      percentage: 1.0 # 收集数据百分比,默认0.1, 1.0表示 100%
```

2. 测试

启动`eureka-server`、`eureka-config`、`eureka-client`、`eureka-feign-cousumer`、`eureka-ribbon-consumer`、`eureka-zuul-client`，访问`http://localhost:5000/ribbonapi/hi?name=leithda&token=123` ，打开`http://127.0.0.1:9411/zipkin/` 点击查找，查询相关链路追踪信息

- 链路追踪界面如图
{% asset_img zipkin.png 链路追踪界面 %}
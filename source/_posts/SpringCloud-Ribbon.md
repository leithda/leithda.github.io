---
title: Spring Cloud 之 Ribbon-负载均衡
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
date: 2019-9-19
abbrlink: 2968946741
---

*负载均衡*是指将负载分摊到多个执行单元上，常见的负载均衡有两种方式。 一种是独立进 程单元，通过负载均衡策略，将请求转发到不同的执行单元上，例如 Ngnix。另一种是将负载 均衡逻辑以代码的形式封装到服务消费者的客户端上，服务消费者客户端维护了一份服务提供 者的信息列表，有了信息列表，通过负载均衡策略将请求分摊给多个服务提供者，从而达到负 载均衡的目的。

Ribbon属于上述的第二种
<!-- More -->
## RestTemplate
RestTemplate是一个访问第三方阻ST削 API 接口的网络请求框架。
它封装了HTTP的常用请求。

## 使用RestTemplate和Ribbon来消费服务
### 开启Eureke Server和Provider
1. 首先开启Eureka Server端口为 8671
2. 然后开启两个相同应用名Eureka Client，端口分别为8673、8674
### 增加Eureka Consumer消费服务
1. 新增Moudle[eureka-ribbon-consumer],增加`web`、`eureka-client`、`ribbon`依赖
2. 配置文件
```yml
spring:
  application:
    name: eureka-consumer
server:
  port: 8675
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8671/eureka/
```
3. 入口类增加`@EnableEurekaClient`注解开启`Eureka Client`功能
4. 增加配置文件，使`Ribbon`和`RestTemplate`结合实现负载均衡
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-19
 * Description: Ribbon 配置文件
 */
@Configuration
public class RibbonConf {
    @Bean
    @LoadBalanced
        // 注入这个Bean后，当前RestTemplate就结合Ribbon开启了负载均衡功能
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

5. 新增服务
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-19
 * Description: Ribbon服务
 */
@Service
public class RibbonServ {
    @Autowired
    RestTemplate restTemplate;

    public String hi(String name) {
        // http://eureka-client 注册到Eureka Server的应用名称
        return restTemplate.getForObject("http://eureka-client/hi?name=" + name, String.class);
    }
}
```

6. 新增控制器类
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-19
 * Description: Ribbon 控制器
 */
@RestController
public class RibbonController {

    @Autowired
    RibbonServ ribbonServ;

    @GetMapping("/hi")
    public String hi(@RequestParam(required = false,defaultValue = "leithda") String name){
        return ribbonServ.hi(name);
    }
}
```

7. 启动新增项目，访问`http://localhost:8675/hi?name=test`多次，会看到输出如下:
```html
hi test , i am from port : 8763
hi test , i am from port : 8764
```

至此：负载均衡功能实现


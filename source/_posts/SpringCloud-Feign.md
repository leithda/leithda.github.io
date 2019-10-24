---
title: Spring Cloud 之 Feign-声明式调用
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
date: 2019-9-20
abbrlink: 874139413
---
Feign是Netflix开发的声明式、模板化的HTTP客户端， Feign可以帮助我们更快捷、优雅地调用HTTP API。

Spring Cloud Feign帮助我们定义和实现依赖服务接口的定义。在Spring Cloud feign的实现下，只需要创建一个接口并用注解方式配置它，即可完成服务提供方的接口绑定，简化了在使用Spring Cloud Ribbon时自行封装服务调用客户端的开发量。

Spring Cloud Feign具备可插拔的注解支持，支持Feign注解、JAX-RS注解和Spring MVC的注解
<!--  More-->

## 使用
1. 新建Moudle[eureka-feign-consumer],引入依赖`eureka-client`、`feign`、`web`
2. 编写配置文件
```yml
spring:
  application:
    name: eureka-feign-consumer
server:
  port: 8676
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/
```

3. 启动类增加注解开启Feign功能
```java
@SpringBootApplication
@EnableEurekaClient /* 开启 Eureke Client 功能 */
@EnableFeignClients /* 开启 Feign Client 功能 */
public class EurekaFeignConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaFeignConsumerApplication.class, args);
    }
}
```

4. 编写Feign配置类
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Feign 配置类
 */
@Configuration
public class FeignConf {
    /**
     * 注入Retryer实现调用失败重试，如果不手动注入会自动注入默认配置类
     * @return
     */
    @Bean
    public Retryer feignRetryer(){
        // 重试间隔 100ms, 重试时间 1s, 最大重试次数 5次
        return new Retryer.Default(100,SECONDS.toMillis(1),5);
    }
}
```

5. 编写Feign接口调用远程服务
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Feign 接口
 */
/*
 * 接口上增加@FeignClient 注解声明这是一个Feign Client
 * 其中 value 为远程调用其他服务的服务名，configuration指明配置类,配置类注入Bean实现失败重试
 * 内部有个 hiFromClientEureka 方法，方法通过Feign远程调用eureka-client的 "/hi" API接口
 * */
@FeignClient(value = "eureka-client",configuration = FeignConf.class)
public interface EurekaClientFeign {

    @GetMapping(value = "/hi")
    String hiFromClientEureka(@RequestParam(value = "name") String name);
}
```

6. 编写Service及Controller
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Feign 服务
 */
@Service
public class FeignServ {

    @Autowired
    EurekaClientFeign eurekaClientFeign;

    public String hi(String name){
        return eurekaClientFeign.hiFromClientEureka(name);
    }
}

// ========================================================================================

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Feign 控制器
 */
@RestController
public class FeignController {

    @Autowired
    FeignServ feignServ;

    @GetMapping("/hi")
    public String hi(@RequestParam(defaultValue = "leithda",required = false) String name){
        return feignServ.hi(name);
    }
}
```

8. 启动新增项目，访问`http://localhost:8676/hi?name=test`多次，会看到输出如下:
```html
hi test , i am from port : 8763
hi test , i am from port : 8764
```

至此：负载均衡功能实现

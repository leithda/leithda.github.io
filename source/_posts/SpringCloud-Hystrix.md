---
title: Spring Cloud 之 Hystrix-熔断器
categories:
  - 框架
  - SpringCloud
  - 基础
tags:
  - Spring Cloud
author: 长歌
date: 2019-9-24
abbrlink: 1776285993
---

在一个分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，如何能够保证在一个依赖出问题的情况下，不会导致整体服务失败，这个就是Hystrix需要做的事情。Hystrix提供了熔断、隔离、Fallback、cache、监控等功能，能够在一个、或多个依赖同时出现问题时保证系统依然可用。
<!-- More -->

## Hystrix的设计原则
 - 防止单个服务的故障耗尽整个服务器的资源
 - 快速失败，当某个服务出现故障，调用该服务的请求快速失败，而不是线程等待
 - 提供回退(fallback)方案，在请求发生故障时，提供设定好的回退方案
 - 提供熔断机制，防止故障扩散到其他服务
 - 提供监控组件Hystrix Dashboard ,可以实时监控熔断器的状态


## 在 `RestTemplate` 和 `Ribbon` 上使用熔断器
1. 首先引入`hystrix`依赖
```xml
        <!-- hystrix 熔断器 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```

2. 启动类增加注解开启熔断器功能
```java
@SpringBootApplication
@EnableEurekaClient // 开启Eureka 注册中心功能
@EnableHystrix  // 开启Hystrix 熔断器功能
public class EurekaRibbonConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaRibbonConsumerApplication.class, args);
    }

}
```

3. 修改Serv 方法，开启熔断功能
```java
@Service
public class RibbonServ {
    @Autowired
    RestTemplate restTemplate;
    @HystrixCommand(fallbackMethod ="hiError")
    public String hi(String name) {
        // http://eureka-provider 注册到Eureka Server的应用名称
        return restTemplate.getForObject("http://eureka-client/hi?name=" + name, String.class);
    }

    /**
     * Hystrix 快速失败方法
     * @param name
     * @return
     */
    public String hiError(String name){
        return "hi "+name+" sorry , An Error";
    }
}
```

4. 依次开启Eureka Server,Client,Feign-Consumer，访问`http://localhost:8675/hi`,关闭后Client后重新访问，发现返回字符串为`hi"+name+" sorry , An Error`。 后续访问都会进入到`fallbackMethod`指定的方法中，熔断器生效后一段时间内会处于半打开状态，将一定数量的请求执行正常逻辑，如果成功，关闭熔断。


## 在`Feign`上使用熔断器
1. `Feign`内置了`Hystrix`熔断器，使用时只需要修改配置文件
```yml
spring:
  application:
    name: eureka-feign-consumer
server:
  port: 8676
###  Eureka 配置  ###
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8671/eureka/
###  Feign 配置  ###
feign:
  hystrix:
    enabled: true # 开启熔断器功能
```

2. `Feign`接口增加熔断
```java
/*
* 接口上增加 @FeignClient 注解声明这是一个Feign Client
* 其中 value 为远程调用其他服务的服务名，configuration指明配置类
* 内部有个 hiFromClientEureka 方法，方法通过Feign远程调用eureka-client的 "/hi" API接口
* fallback hystrix逻辑处理类，需要实现本接口，进行快速失败方法实现
* */
@FeignClient(value = "ureuka-client",configuration = FeignConf.class,fallback = HiHystrix.class)
public interface EurekaClientFeign {

    @GetMapping(value = "/hi")
    String hiFromClientEureka(@RequestParam(value = "name") String name);
}
```

3. 编写`Hystrix`逻辑处理类
```java
@Component  // 需要以组件形式注入
public class HiHystrix implements EurekaClientFeign {
    // 实现熔断接口，实现快速失败方法
    @Override
    public String hiFromClientEureka(String name) {
        return "hi "+name+" , this is an Error";
    }
}
```

## 使用`Hystrix Dashboard`监控熔断器状态
### 在`RestTemplate`中使用
1. 加入依赖（*hystrix已经在项目中加入*）
```xml
<!-- hystrix 熔断器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<!-- 解决找不到 HystrixCommand注解 -->
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-javanica</artifactId>
    <version>RELEASE</version>
</dependency>
<!-- 监控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>2.1.4.RELEASE</version>
</dependency>
<!-- hystrix 熔断器监控 -->
<!--<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>-->
<!-- 上面的依赖找不到@EnableHystrixDashboard注解-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

2. 启动类增加注解开启功能,增加servlet配置(spring boot 2.0需要)设置监控界面url。
```java
@SpringBootApplication
@EnableEurekaClient // 开启Eureka 注册中心功能
@EnableHystrix  // 开启Hystrix 熔断器功能
/*以下两个注解为新增注解*/
@EnableHystrixDashboard // 开启 Hystrix 熔断器监控
@EnableCircuitBreaker   // 开启断路器
public class EurekaRibbonConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaRibbonConsumerApplication.class, args);
    }

    /**
     * 配置 hystrix dashboard 的servlet，解决hystrix.stream无法使用问题
     * @return registrationBean
     */
    @Bean
    public ServletRegistrationBean getServlet(){
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
}
```
3. 依次启动 Eureka Server,Eureka Client, Eureka Ribbon Consumer。
- 首先访问`http://localhost:8675/hi`进行请求
- 然后打开`http://localhost:8675/hystrix.stream` 此时已经可以看见熔断器的数据指标。
- 图形化界面可以访问`http://localhost:8675/hystrix`

{% asset_img hystrix_1.png Hystrix管理界面 %}
- 按照如图所示填写后点击`Monitor Stream`进入图形化展示界面

{% asset_img hystrix_2.png Hystrix监控界面 %}

关于界面的更多信息请查阅[官方文档](https://github.com/NetFlix/Hystrix/wiki/Dashboard)

### 在`Feign`中使用

1. 加入依赖
```xml
<!-- 监控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!--Hystrix 依赖 主要是用  @HystrixCommand-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<!-- 解决找不到@EnableHystrixDashboard注解 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>d>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

2. 开启功能
```java
@SpringBootApplication
@EnableEurekaClient /* 开启 Eureke Client 功能 */
@EnableFeignClients /* 开启 Feign Client 功能 */
@EnableCircuitBreaker   // 开启熔断器 也可以使用 @EnableHystrix 开启熔断器
@EnableHystrixDashboard // 开启 Hystrix 熔断器监控
public class EurekaFeignConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaFeignConsumerApplication.class, args);
    }

    /**
     * 配置 hystrix dashboard 的servlet，解决hystrix.stream无法使用问题
     * @return registrationBean
     */
    @Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
}
```

3. 剩下的步骤与RestTemplate中相同，不再演示。


## 使用`Turbine` 聚合监控
使用`Hystrix Dashboard`时，每个服务都有一个监控页面，很不方便，Netflix 开源了 Hystrix 的另一个组件 Turbine用于聚合多个监控，集中展示

1. 新建Moudle工程[eureka-monitor-client]
2. 引入依赖
```xml
 <!-- springboot 监控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- turbine 聚合监控 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

3. 配置文件
```yml
spring:
  application:
    name: service-turbine
server:
  port: 8679

turbine:
  instanceUrlSuffix: hystrix.stream # 由于监控的服务将监控url改为hystrix.stream 这里需要设置，默认的是 acuator/hystrix.stream
  aggregator:
    clusterConfig: default   # 指定聚合哪些集群，多个使用","分割，默认为default。可使用http://.../turbine.stream?cluster={clusterConfig之一}访问
  appConfig: eureka-ribbon-consumer,eureka-feign-consumer  # 配置Eureka中的serviceId列表，表明监控哪些服务
  clusterNameExpression: new String("default")
  # 1. clusterNameExpression指定集群名称，默认表达式appName；此时：turbine.aggregator.clusterConfig需要配置想要监控的应用名称
  # 2. 当clusterNameExpression: default时，turbine.aggregator.clusterConfig可以不写，因为默认就是default
  # 3. 当clusterNameExpression: metadata['cluster']时，假设想要监控的应用配置了eureka.instance.metadata-map.cluster: ABC，则需要配置，同时turbine.aggregator.clusterConfig: ABC
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8671/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
    # 配置使用主机名注册服务
    #    hostname: testnode
    # 优先使用IP地址方式进行注册服务
    prefer-ip-address: true
    # 配置使用指定IP
    ip-address: 127.0.0.1
```

4. 开启功能
```java
@SpringBootApplication
@EnableTurbine  /* 开启 Turbine 功能 */
public class EurekaMonitorClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaMonitorClientApplication.class, args);
    }
}
```

5. 测试
- 开启 Eureka-server,client以及两个consumer
- 访问`http://localhost:8675/hi`及`http://localhost:8676/hi`
- 访问`http://localhost:8675/hystrix`按照如图所示填写

{% asset_img turbine_1.png turbine聚合监控 %}
- 监控界面如图

{% asset_img turbine_2.png turbine聚合监控 %}
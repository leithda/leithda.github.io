---
title: Spring Cloud 之 Admin-微服务监控
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
abbrlink: 1287611694
date: 2019-09-29 20:30:00
---

Spring Boot Adm in 用于管理和监控一个或者多个 Spring Boot 程序。 Spring Boot Admin 分 为 Server 端和 Client 端， Client 端可以通过 Http 向 Server 端注册，也可以结合 Spring Cloud 的服务注册组件 Eureka 进行注册。 Spring Boot Admin 提供了用 AngularJs 编写的 Ul 界面，用 于管理和监控。其中监控内容包括 Spring Boot 的监控组件 Actuator 的各个 Http 节点，也支持 更高级的功能，包括 Turbine 、 Jmx 、 Loglevel 等。
<!-- More -->
## 使用cloud admin进行监控
### 服务端
1. 新建Moudle工程[admin-center]
2. 引入依赖
```xml
<!-- boot 监控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- admin 管理服务端 -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
<!-- eureka 客户端 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

3. 编写配置文件
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
spring:
  application:
    name: admin-server
#  boot:
#    admin:
#      probed-endpoints: health, env, metrics, httptrace:trace, httptrace, threaddump:dump, threaddump, jolokia, info, logfile, refresh, flyway, liquibase, heapdump, loggers, auditevents, mappings, scheduledtasks, configprops, caches, beans
server:
  port: 5001
management:
  endpoints:
    web:
      exposure:
        include: "*" # 关闭安全验证
logging:
  file: "log/boot-admin-simple.log"
```

4. 开启功能
```java
@SpringBootApplication
@EnableEurekaClient
@EnableAdminServer
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }
}
```

### 客户端
1. 加入依赖
```xml
<!-- admin 监控 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2. 配置文件`eureka-client-dev.xml`
```yml
logging:
  file: "log/eureka-client.log" # 日志文件
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```
- 其他客户端操作相同

### 验证
1. 启动`eureka-server:8671`,`config-server:9680`,`eureka-client:8763`,`admin-server:5001`
2. 在浏览器上访问监控主页`http://localhost:5001`
效果如图:
{% asset_img admin_1.png Admin 监控界面%}
{% asset_img admin_2.png Admin 监控界面%}

## 集成 `spring security`

1. 在`admin-server`中加入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

2. 修改配置文件
```yml
spring:
  security:
    user:
      name: "admin"
      password: "admin"
      
eureka:
  instance:
    metadata-map:
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}
```

3. 编写配置类
```java
@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {

    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter( "redirectTo" );

        http.authorizeRequests()
                .antMatchers( adminContextPath + "/assets/**" ).permitAll()
                .antMatchers( adminContextPath + "/login" ).permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage( adminContextPath + "/login" ).successHandler( successHandler ).and()
                .logout().logoutUrl( adminContextPath + "/logout" ).and()
                .httpBasic().and()
                .csrf().disable();
        // @formatter:on
    }
}
```

4. 重启工程，打开`http://localhost:5001` 会重定向进行登录.
{% asset_img admin_3.png Admin登录界面 %}

## 集成邮箱报警功能

1. `admin-server` 加入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

2. 配置文件中加入邮箱配置
```yml
spring.mail.host: smtp.163.com
spring.mail.username: leithda
spring.mail.password:
spring.boot.admin.notify.mail.to: 123456@qq.com
```

3. 服务不健康、上线或者下线会给指定邮箱发送邮件
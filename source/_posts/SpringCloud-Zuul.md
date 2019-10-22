title: Spring Cloud 之 Zuul-路由网关
categories:
  - Java
  - Spring Cloud
tags:
  - Spring Boot
  - 微服务
  - Spring Cloud
author: 长歌
date: 2019-9-27 07:30:00
---

智能路由网关组件Zuul，用于构建边界服务（Edge Service），致力于动态路由、过滤、监控、弹性伸缩和安全
<!-- More -->
## 作用
- Zuul、Ribbon以及Eureka组合，可以实现智能路由和负载均衡的功能
- 网关将API接口聚合，统一对外暴露，保护服务内部暴露
- 网关可以进行认证鉴权，防止非法请求操作API接口
- 网关可以实现监控，实时日志输出，记录请求
- 实现流量监控，高流量情况下，对服务进行降级
- API接口从内部分离出来，方便测试

## 原理
Zuul是通过Servlet实现的，Zuul通过自定义的ZuulServlet对请求进行控制，Zuul的核心是过滤器

1. PRE过滤器：在路由请求到服务前执行，这种过滤器可以用作安全验证。
2. ROUTING过滤器：用于将请求路由到具体的微服务实例。默认情况，他通过Http Client进行网络请求
3. POST过滤器： 请求已被路由到微服务实例后执行，一般用作收集统计信息、指标，以及将响应传输到客户端
4. ERROR过滤器，在其他过滤器发生错误时执行

## Zuul
### 搭建Zuul服务
1. 新建Moudle工程[eureka-zuul-client]
2. 添加依赖web,eureka,zuul
```xml
<!-- spring boot web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- eureka 注册客户端 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!-- zuul 网关 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```
3. 配置文件
```yml
spring:
  application:
    name: service-zuul
server:
  port: 5000
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8671/eureka/
  instance:
    instance-id: ${spring.application.name}:${server.port}
##   Zuul Settings ##
zuul:
  routes: # 路由设置
    hiapi:
      path: /hiapi/**
      serviceId: eureka-client  # 可以使用 url: http://localhost:8672 指定url后取消负载均衡
    ribbonapi:
      path: /ribbonapi/**
      serviceId: eureka-ribbon-consumer
    feignapt:
      path: /feignapi/**
      serviceId: eureka-feign-consumer
```

4. 开启功能
```java
@SpringBootApplication
@EnableZuulProxy    // 开启 Zuul 网关功能
@EnableEurekaClient // 开启 Eureka Clinet 功能
public class EurekaZuulClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaZuulClientApplication.class, args);
    }
}
```

4. 测试
 - 依次启动eureka-server,client,client2,ribbon-client,feign-client,zuul-client
 - 多次访问`http://localhost:5000/hiapi/hi?name=leithda`

### 在Zuul上配置API接口的版本号
1. 通过`zuul-prefix`设置版本号
```yml
##   Zuul Settings ##
zuul:
  routes: # 路由设置
    hiapi:
      path: /hiapi/**
      serviceId: eureka-client  # 可以使用 url: http://localhost:8672 指定url后取消负载均衡
    ribbonapi:
      path: /ribbonapi/**
      serviceId: eureka-ribbon-consumer
    feignapt:
      path: /feignapi/**
      serviceId: eureka-feign-consumer
  prefix: /v1    # 设置前缀作为版本号
```
2. 重启`eureka-zuul-client`,访问`http://localhost:5000/v1/hiapi/hi?name=leithda`

### 在Zuul上配置熔断器
在Zuul中实现熔断器需要实现`FallbackProvider`接口
```java

/**
 * Created by Intelli IDEA
 *
 * User: 长歌
 * Date: 2019/4/11
 * Time: 12:30
 * Desc: Zuul 熔断器设置
 */
@Component
public class MyFallbackProvider implements FallbackProvider {
    /**
     * 指定熔断那些路由的服务
     * @return 返回服务名，* 表示所有服务
     */
    public String getRoute() {
        return "*"; // "eureka-client"
    }

    /**
     * 进入熔断后执行的逻辑
     * @param route 熔断路由
     * @param cause 失败原因
     * @return
     */
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        System.out.println("route:"+route);
        System.out.println("cause:"+cause.getMessage());
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "ok";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("ok ! this is an error,i'm the fallback.".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```

### 在Zuul中使用过滤器
1. 编写过滤器代码，需要继承`ZuulFilter`类，编写如下四个方法

```java
/**
 * Created by Intelli IDEA
 * User: 长歌
 * Date: 2019/4/11
 * Time: 12:52
 * Desc: 自定义过滤器
 */
@Component
public class MyFilter extends ZuulFilter {
    /**
     * 设置过滤器类型
     * @return 过滤器类型
     */
    public String filterType() {
        return PRE_TYPE;
    }

    /**
     * 设置过滤器权重
     * @return 权值，越小越先执行
     */
    public int filterOrder() {
        return 0;
    }

    /**
     * 是否过滤，如果为true，执行run方法
     * @return 是否过滤
     */
    public boolean shouldFilter() {
        // 根据逻辑判断是否需要进行过滤
        RequestContext ctx = RequestContext . getCurrentContext() ;
        HttpServletRequest request= ctx . getRequest();
        if(request.getRequestURI().contains("hi")){
            return true;
        }
        return false;
    }

    /**
     * 过滤方法
     * @return 返回值，作用未知
     * @throws ZuulException 异常
     */
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext . getCurrentContext() ;
        HttpServletRequest request= ctx . getRequest();
        Object accessToken = request.getParameter("token");
        if(accessToken==null){
            System.out.println("token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try{
                ctx.getResponse().getWriter().write("token is empty");
            }catch (Exception e){
                return null;
            }
        }
        return null;
    }
}
```
2. 测试

重启zuul-client,分别访问`http://localhost:5000/v1/ribbonapi/hi?name=leithda`、`http://localhost:5000/v1/hiapi/hi?name=leithda`以及`http://localhost:5000/v1/hiapi/hi?name=leithda&token=test`

### Zuul的常见使用方式
1. Zuul基于`Servlet`实现，采用异步阻塞模型，性能不及Ngnix，可以横向扩展解决性能问题
2. 不同渠道使用不同的Zuul进行路由
3. 使用Ngnix和Zuul结合来做负载均衡，暴露在最外面的是 Ngnix 主从双热备进行 Keepalive, Ngnix 经过某种路由策略，将请求路由转发到 Zuul 集群上， Zuul 最终将请求分发到具体的服务上。
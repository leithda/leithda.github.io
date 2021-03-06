---
title: Tomcat-主机(host)和引擎(engine)
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 700695093
date: 2019-10-26 00:00:00
---

这篇文章主要介绍的是Host容器和Engine容器。如果你想在同个Tomcat上部署运行多个Context容器的话，你就需要使用Host容器，从理论上来讲，如果你的Tomcat只想要部署一个Context容器的话，可以不使用Host容器
在`org.apache.catalina.Context`接口的描述有以下一段话
- Context容器的父容器通常是Host容器，也有可能是其他实现，如果不是必要的话，就可以不使用父容器。
- 但是在tomcat的实际部署中，总会使用一个Host容器，原因如下：    Engine容器表示Catalina的整个Servlet引擎，如果使用了Engine容器，那么它总是处于容器层级的最顶层，添加到Engine容中的子容器通常是`org.apache.cataline.Host`或者`Context`的实现。默认情况下Tomcat会使用一个Engine容器并使用一个Host容器作为其子容器

<!-- More -->

# Host
## Host 接口

```java
public interface Host extends Container {
    String ADD_ALIAS_EVENT = "addAlias";
    String REMOVE_ALIAS_EVENT = "removeAlias";

    String getAppBase();

    void setAppBase(String var1);

    boolean getAutoDeploy();

    void setAutoDeploy(boolean var1);

    void addDefaultContext(DefaultContext var1);

    String getName();

    void setName(String var1);

    void importDefaultContext(Context var1);

    void addAlias(String var1);

    String[] findAliases();

    Context map(String var1);

    void removeAlias(String var1);
}
```

- 最重要的是`#map()`方法，找到合适的`Context`来处理请求

## StandardHost
### 构造函数

```java
    // StandardHost.java

    public StandardHost() {
        this.pipeline.setBasic(new StandardHostValve());
    }
```

- 在构造方法中加入基本阀门`StandardHostValve`

### start

```java
    // StandardHost.java

    private String errorReportValveClass = "org.apache.catalina.valves.ErrorReportValve";

    public synchronized void start() throws LifecycleException {
        if (this.errorReportValveClass != null && !this.errorReportValveClass.equals("")) {
            try {
                Valve valve = (Valve)Class.forName(this.errorReportValveClass).newInstance();
                this.addValve(valve);
            } catch (Throwable var2) {
                this.log(ContainerBase.sm.getString("standardHost.invalidErrorReportValveClass", this.errorReportValveClass));
            }
        }

        this.addValve(new ErrorDispatcherValve());
        super.start();
    }
```

- `#start`方法中加入两个一般阀门`errorReportValveClass`和`ErrorDispatcherValve`

### invoke
- 由于`StandardHost`没有实现`#invoke()`方法,所以调用它的父类`ContainerBase`的`#invoke()`方法，转而调用基本阀门的`#invoke`方法，调用`#StandardHost.map()`方法获得一个合适的上下文处理器处理请求。
- `StandardHost`的`#map()`方法如下:
  
```java
    // StandardHost.java

    public Context map(String uri) {
        if (uri == null) {
            return null;
        } else {
            Context context = null;
            String mapuri = uri;

            while(true) {
                context = (Context)this.findChild(mapuri);
                if (context != null) {
                    break;
                }
                int slash = mapuri.lastIndexOf(47);
                if (slash < 0) {
                    break;
                }
                mapuri = mapuri.substring(0, slash);
            }

            if (context == null) {
                context = (Context)this.findChild("");
            }

            if (context == null) {
                this.log(ContainerBase.sm.getString("standardHost.mappingError", uri));
                return null;
            } else {
                return context;
            }
        }
    }
```

## StandardHostMapper
- `StandardHost`的父类`ContainerBase`使用`#addDefaultMapper()`方法创建一个默认映射器。

```java
    // ContainerBase.java
    protected void addDefaultMapper(String mapperClass) {
        if (mapperClass != null) {
            if (this.mappers.size() < 1) {
                try {
                    Class clazz = Class.forName(mapperClass);
                    Mapper mapper = (Mapper)clazz.newInstance();
                    mapper.setProtocol("http");
                    this.addMapper(mapper);
                } catch (Exception var4) {
                    this.log(sm.getString("containerBase.addDefaultMapper", mapperClass), var4);
                }

            }
        }
    }
```

- `mapperClass`由`StandardHost`指定`private String mapperClass = "org.apache.catalina.core.StandardHostMapper";`
- `StandardHost`的`#start()`方法的最后调用`super.start()`方法，
- `StandardHostMapper`中最重要的方法是`#map()`方法.

```java
    // StandardHostMapper.java

    public Container map(Request request, boolean update) {
        if (update && request.getContext() != null) {
            return request.getContext();
        } else {
            String uri = ((HttpRequest)request).getDecodedRequestURI();
            Context context = this.host.map(uri);
            if (update) {
                request.setContext(context);
                if (context != null) {
                    ((HttpRequest)request).setContextPath(context.getPath());
                } else {
                    ((HttpRequest)request).setContextPath((String)null);
                }
            }

            return context;
        }
    }
```

## StandardHostValve

```java
    public void invoke(Request request, Response response, ValveContext valveContext) throws IOException, ServletException {
        if (request.getRequest() instanceof HttpServletRequest && response.getResponse() instanceof HttpServletResponse) {
            StandardHost host = (StandardHost)this.getContainer();
            Context context = (Context)host.map(request, true); // <1> 
            if (context == null) {
                ((HttpServletResponse)response.getResponse()).sendError(500, sm.getString("standardHost.noContext"));
            } else {
                Thread.currentThread().setContextClassLoader(context.getLoader().getClassLoader());
                HttpServletRequest hreq = (HttpServletRequest)request.getRequest();
                String sessionId = hreq.getRequestedSessionId();
                if (sessionId != null) {
                    Manager manager = context.getManager();
                    if (manager != null) {
                        Session session = manager.findSession(sessionId);
                        if (session != null && session.isValid()) {
                            session.access();   // <2>
                        }
                    }
                }

                context.invoke(request, response);  // <3>
            }
        }
    }
```

- `<1>`处，调用`StandardHost`的`#map()`方法获取一个合适的上下文
- `<2>`处，调用`Session`的`#access()`方法更新`Session`的最后进入时间
- `<3>`处，调用`Host`的`#invoke()`方法，处理请求

# Engine
## Engine 接口

```java
public interface Engine extends Container {
    String getDefaultHost();

    void setDefaultHost(String var1);

    String getJvmRoute();

    void setJvmRoute(String var1);

    Service getService();

    void setService(Service var1);

    void addDefaultContext(DefaultContext var1);

    void importDefaultContext(Context var1);
}
```

## StandardEngine
### 构造函数

```java
    // StandardEngine.java
    public StandardEngine() {
        this.pipeline.setBasic(new StandardEngineValve());
    }
```

- 在构造函数中设置基本阀门

### 添加子容器

```java
    // StandardEngine.java

    public void addChild(Container child) {
        if (!(child instanceof Host)) {
            throw new IllegalArgumentException(ContainerBase.sm.getString("standardEngine.notHost"));
        } else {
            super.addChild(child);
        }
    }
```

- 添加非`Host`子容器的时候，会报错

### StandardEngineValve

```java
    // StandardEngineValve.java

    public void invoke(Request request, Response response, ValveContext valveContext) throws IOException, ServletException {
        if (request.getRequest() instanceof HttpServletRequest && response.getResponse() instanceof HttpServletResponse) {
            HttpServletRequest hrequest = (HttpServletRequest)request;
            if ("HTTP/1.1".equals(hrequest.getProtocol()) && hrequest.getServerName() == null) {
                ((HttpServletResponse)response.getResponse()).sendError(400, sm.getString("standardEngine.noHostHeader", request.getRequest().getServerName()));
            } else {
                StandardEngine engine = (StandardEngine)this.getContainer();
                Host host = (Host)engine.map(request, true);
                if (host == null) {
                    ((HttpServletResponse)response.getResponse()).sendError(400, sm.getString("standardEngine.noHost", request.getRequest().getServerName()));
                } else {
                    host.invoke(request, response);
                }
            }
        }
    }
```

# 应用程序
## 测试 Host
### 上下文监听器

```java
public class SimpleContextConfig implements LifecycleListener {

    public void lifecycleEvent(LifecycleEvent event) {
        if (Lifecycle.START_EVENT.equals(event.getType())) {
            Context context = (Context) event.getLifecycle();
            context.setConfigured(true);
        }
    }
}
```

### 启动类

```java
public class Bootstrap1 {
    public static void main(String[] args) {
        //invoke: http://localhost:8080/app1/Primitive or http://localhost:8080/app1/Modern
        System.setProperty("catalina.base", System.getProperty("user.dir"));
        Connector connector = new HttpConnector();

        Wrapper wrapper1 = new StandardWrapper();
        wrapper1.setName("Primitive");
        wrapper1.setServletClass("PrimitiveServlet");
        Wrapper wrapper2 = new StandardWrapper();
        wrapper2.setName("Modern");
        wrapper2.setServletClass("ModernServlet");

        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/app1");
        context.setDocBase("app1");

        context.addChild(wrapper1);
        context.addChild(wrapper2);

        LifecycleListener listener = new SimpleContextConfig();
        ((Lifecycle) context).addLifecycleListener(listener);

        Host host = new StandardHost();
        host.addChild(context);
        host.setName("localhost");
        host.setAppBase("webapps");

        Loader loader = new WebappLoader();
        context.setLoader(loader);
        // context.addServletMapping(pattern, name);
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");

        connector.setContainer(host);
        try {
            connector.initialize();
            ((Lifecycle) connector).start();
            ((Lifecycle) host).start();

            // make the application wait until we press a key.
            System.in.read();
            ((Lifecycle) host).stop();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### web项目路径
- 在项目根路径下创建`webapps`文件夹,对应文件如下：
```bash
webapps
└── app1
    └── WEB-INF
        ├── classes
        │   ├── ModernServlet.class
        │   ├── PrimitiveServlet.class
        │   └── SessionServlet.class
        └── web.xml
```

- 其中`web.xml`内容如下：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>

<!DOCTYPE web-app
    PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
    "http://java.sun.com/dtd/web-app_2_3.dtd">

<web-app>
  <servlet>
    <servlet-name>Modern</servlet-name>
    <servlet-class>ModernServlet</servlet-class>
  </servlet>
  <servlet>
    <servlet-name>Primitive</servlet-name>
    <servlet-class>PrimitiveServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>Modern</servlet-name>
    <url-pattern>/Modern</url-pattern>
  </servlet-mapping>
  <servlet-mapping>
    <servlet-name>Primitive</servlet-name>
    <url-pattern>/Primitive</url-pattern>
  </servlet-mapping>
</web-app>
```

## 测试 Engine
### 启动类

```java
public class Bootstrap2 {
    public static void main(String[] args) {
        //invoke: http://localhost:8080/app1/Primitive or http://localhost:8080/app1/Modern
        System.setProperty("catalina.base", System.getProperty("user.dir"));
        Connector connector = new HttpConnector();

        Wrapper wrapper1 = new StandardWrapper();
        wrapper1.setName("Primitive");
        wrapper1.setServletClass("PrimitiveServlet");
        Wrapper wrapper2 = new StandardWrapper();
        wrapper2.setName("Modern");
        wrapper2.setServletClass("ModernServlet");

        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/app1");
        context.setDocBase("app1");

        context.addChild(wrapper1);
        context.addChild(wrapper2);

        LifecycleListener listener = new SimpleContextConfig();
        ((Lifecycle) context).addLifecycleListener(listener);

        Host host = new StandardHost();
        host.addChild(context);
        host.setName("localhost");
        host.setAppBase("webapps");

        Loader loader = new WebappLoader();
        context.setLoader(loader);
        // context.addServletMapping(pattern, name);
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");

        Engine engine = new StandardEngine();
        engine.addChild(host);
        engine.setDefaultHost("localhost");

        connector.setContainer(engine);
        try {
            connector.initialize();
            ((Lifecycle) connector).start();
            ((Lifecycle) engine).start();

            // make the application wait until we press a key.
            System.in.read();
            ((Lifecycle) engine).stop();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 启动流程
1. `Bootstrap2`中`Engine.start()`,然后`#StandardEngine.start()`方法中调用父类`ContainerBase.start()`方法，加载默认映射器`StandardEngineMapper`，然后启动组件(加载器,日志，定时器等)，然后启动子容器`StandardHost`.
2. `#StandardHost.start()`方法中，加入阀门`errorReportValveClass`，然后调用`#super.start()`调用父类的`start()`方法，加载默认映射器`StandardHostMapper`，启动组件，启动子容器`StandardContext`
3. `#StandardContext.start()`方法中，设置资源路径，设置加载器，设置管理器，加载映射器`org.apache.catalina.util.CharsetMapper`，设置工作目录`localhost/app1`
4. 如果`3`中没有发生错误，继续启动，加载默认映射器`org.apache.catalina.core.StandardContextMapper`,启动组件，启动第一个子容器`StandardWrapper(Modern)`
5. 包装器的`#start()`的方法中同样调用父类`ContainerBase`的`#start()`方法完成启动。启动组件，启动子容器，启动包装器流水线`PipeLine`，如果阀门实现了生命周期接口，启动阀门，处理完成返回
6. 承接第4步，当第一个子容器`StandardWrapper(Modern)`启动完成后启动第二个子容器`StandardWrapper(Primitive)`,过程忽略
7. 之后启动`StandardContext`的流水线作业，完成后启动上下文容器的管理器`StandardManager`，然后上下文设置欢迎文件,启动监听器，启动过滤器，设置可用标志
8.启动 `Host`的流水线，启动`Engine`的流水线

- 时序图大体如下：
{% mermaid sequenceDiagram %}
participant a as bootstrap
participant b as Engine
participant c as Host
participant d as Context
participant e as Wrapper
a->>b:Engine.start()
Note right of b: 启动组件<br/>启动子容器
b->>c:Host.start()
Note right of c: 加入阀门errorReportValveClass<br>启动组件<br>启动子容器
c->>d:Context.start()
Note right of d: 设置资源路径<br>设置加载器<br>设置管理器<br>设置工作目录<br>
d->>e:Wrapper.start()
Note right of e:pipeline.start()<br>启动阀门
e-->>d:start() end
Note over d: pipeline.start()<br>manager.start()<br>设置欢迎文件<br>listener.start()<br>filter.start()<br>设置可用
d-->>c: start() end
Note over c: pipeline.start()
c-->>b: start() end
Note over b: pipeline.start()
b-->>a: startup end
{% endmermaid %}

## 处理请求流程
- 时序图如下:
{% mermaid sequenceDiagram %}
participant a as HttpProcessor
participant b as StandardEngine
participant c as StandardHost
participant d as StandardContext
participant e as StandardWrapper
Note left of a: receive request
a->>b: getContainer().invoke()
Note right of b: Pipeline.invoke()
b->>c: engine.map().invoke()
Note right of c: Pipeline.invoke()
c->>d: host.map().invoke()
Note right of d: Pipeline.invoke()
d->>e: context.map().invoke()
Note right of e: Pipeline.invoke()<br>FilterChain.doFilter()<br>servlet.service
{% endmermaid %}

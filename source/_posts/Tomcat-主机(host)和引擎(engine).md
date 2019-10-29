---
title: Tomcat-主机(host)和引擎(engine)
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 700695093
date: 2019-10-26 00:00:00
---

这篇文章主要介绍的是Host容器和Engine容器。如果你想在同个Tomcat上部署运行多个Context容器的话，你就需要使用Host容器，从理论上来讲，如果你的Tomcat只想要部署一个Context容器的话，可以不使用Host容器
在`org.apache.catalina.Context`接口的描述有以下一段话
- Context容器的父容器通常是Host容器，也有可能是其他实现，如果不是必要的话，就可以不使用父容器。
- 但是在tomcat的实际部署中，总会使用一个Host容器，原因如下：    Engine容器表示Catalina的整个Servlet引擎，如果使用了Engine容器，那么它总是处于容器层级的最顶层，添加到Engine容中的子容器通常是`org.apache.cataline.Host`或者`Context`的实现。默认情况下Tomcat会使用一个Engine容器并使用一个Host容器作为其子容器

<!-- More -->

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
- 由于`StandardHost`没有实现`#invoke()`方法,所以调用它的父类`ContainerBase`的`#invoke()`方法，转而调用基本阀门的`#invoke`方法，调用`#Standard.map()`方法获得一个合适的上下文处理器处理请求。
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

## StandardDefaultMapper

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
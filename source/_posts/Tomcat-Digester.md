---
title: Tomcat-服务和服务器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 2030302893
date: 2019-11-10 08:00:00
---

未避免Tomcat启动代码硬编码，Tomcat使用了`server.xml`来灵活配置Tomcat组件，例如上下文可以进行如下配置：
```xml
# To set the path and docBase properties you use attributes in the XML
# element:
<context docBase="myApp" path="/myApp"/>
```
Tomcat 使用开源工具 [Digester](http://commons.apache.org/proper/commons-digester/) 来讲 XML 元素转换为 Java 对象
<!-- More -->

## Digester
Digester 是 Apache Jakarta 项目下面的开源项目。可以在[Digester](http://jakarta.apache.org/commons/digester/)下载。Digester
由三个包组成，被包装到 commons-digeser.jar 文件中。
- org.apache.commons.digester.提供了基于规则（rules-based）的任意 XML 文档处理。
- org.apache.commons.digester.rss.演示了如何使用Digester 解析 XML 文档，跟很多地方使用的 rich site summary 格式比较。
- org.apache.commons.digester.xmlrules.该包提供了给Digester 提供基于 XML 文档的规则

## Tomcat中的Digester

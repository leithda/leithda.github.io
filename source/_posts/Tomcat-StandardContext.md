title: Tomcat-StandardContext
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-10-22
---

标准上下文容器`StandardContext`对象的初始化和配置,然后讨论跟其相关的类`StandardContextMapper`(Tomcat 4)和 `ContextConfig` 类

<!-- More -->

## StandardContext配置
- 一个`StandardContext`创建之后，调用`#start`方法，这样它就可以接收 HTTP 请求了，如果启动失败，设置`available`为false，`available`表示容器可用性
- `#start`方法启动成功，`StandardContext`会配置它的属性。它会读取和分析`%CATALINA_HOME%/conf`下的`web.xml。还需要增加验证器处理器和证书处理器
- `StandardContext`加载配置通过事件监听器处理，它的属性`configured`用来表示容器是否被配置。之前的`SimpleContextConfig`直接更改标志。
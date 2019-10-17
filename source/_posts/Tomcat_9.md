title: Tomcat-Session管理
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-10-17
---

Catalina 通过一个叫管理器的组件来完成 session 管理工作,该组件由org.apache.catalina.Manager interface 接口表示。一个管理器通常跟一个上下文容器相关联,它负责创建、更行以及销毁 session 对象并能给任何请求组件返回一个合法的 session。
<!-- More -->

## Sessions
### `Session`接口
```java
public interface Session {
    String SESSION_CREATED_EVENT = "createSession";
    String SESSION_DESTROYED_EVENT = "destroySession";

    String getAuthType();

    void setAuthType(String var1);

    long getCreationTime();

    void setCreationTime(long var1);

    String getId();

    void setId(String var1);

    String getInfo();

    long getLastAccessedTime();

    Manager getManager();

    void setManager(Manager var1);

    int getMaxInactiveInterval();

    void setMaxInactiveInterval(int var1);

    void setNew(boolean var1);

    Principal getPrincipal();

    void setPrincipal(Principal var1);

    HttpSession getSession();

    void setValid(boolean var1);

    boolean isValid();

    void access();

    void addSessionListener(SessionListener var1);

    void expire();

    Object getNote(String var1);

    Iterator getNoteNames();

    void recycle();

    void removeNote(String var1);

    void removeSessionListener(SessionListener var1);

    void setNote(String var1, Object var2);
}
```
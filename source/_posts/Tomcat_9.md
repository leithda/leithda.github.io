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
- `#setManager`和`getManager`用于设置持有`session`对象的管理器
- `#getLastAccessedTime`用于管理器调用，确定`session`是否合法
- `#access`方法，用于进入`session`时，更新它的最后访问时间

### `StandardSession`
#### 成员变量
```java
    private HashMap attributes = new HashMap();
    private transient String authType = null;
    private transient Method containerEventMethod = null;
    private static final Class[] containerEventTypes;
    private long creationTime = 0L;
    private transient int debug = 0;
    private transient boolean expiring = false;
    private transient StandardSessionFacade facade = null;
    private String id = null;
    private static final String info = "StandardSession/1.0";
    private long lastAccessedTime;
    private transient ArrayList listeners;
    private Manager manager;
    private int maxInactiveInterval;
    private boolean isNew;
    private boolean isValid;
    private transient HashMap notes;
    private transient Principal principal;
    private static StringManager sm;
    private static HttpSessionContext sessionContext;
    private transient PropertyChangeSupport support;
    private long thisAccessedTime;
```

#### `#getSession`
```java
    public HttpSession getSession() {
        if (this.facade == null) {
            this.facade = new StandardSessionFacade(this);
        }

        return this.facade;
    }
```
- `#getSession`方法返回一个`StandardSession`的外观类。

#### `#expire`
```java
    /**
     * 设置session过期
     *
     * @param      notify  是否提醒
     */
    public void expire(boolean notify) {
        // 更新有效期标志
        if (!this.expiring) {
            this.expiring = true;
            this.setValid(false);
            // 从管理器中删除本session
            if (this.manager != null) {
                this.manager.remove(this);
            }

            String[] keys = this.keys();

            // 解除与这个session绑定的对象
            for(int i = 0; i < keys.length; ++i) {
                this.removeAttribute(keys[i], notify);
            }

            // 释放 Session 的销毁事件
            if (notify) {
                this.fireSessionEvent("destroySession", (Object)null);  // <1>
            }

            // 释放 所有相应的事件监听器观察的事件
            Context context = (Context)this.manager.getContainer();
            Object[] listeners = context.getApplicationListeners();
            if (notify && listeners != null) {
                HttpSessionEvent event = new HttpSessionEvent(this.getSession());

                for(int i = 0; i < listeners.length; ++i) {
                    int j = listeners.length - 1 - i;
                    if (listeners[j] instanceof HttpSessionListener) {
                        HttpSessionListener listener = (HttpSessionListener)listeners[j];

                        try {
                            this.fireContainerEvent(context, "beforeSessionDestroyed", listener);
                            listener.sessionDestroyed(event);
                            this.fireContainerEvent(context, "afterSessionDestroyed", listener);
                        } catch (Throwable var13) {
                            try {
                                this.fireContainerEvent(context, "afterSessionDestroyed", listener);
                            } catch (Exception var12) {
                            }

                            this.log(sm.getString("standardSession.sessionEvent"), var13);
                        }
                    }
                }
            }

            // 设置session过期
            this.expiring = false;
            if (this.manager != null && this.manager instanceof ManagerBase) {
                this.recycle();
            }

        }
    }
```
- `<1>`处，观察者模式实现，触发事件，由事件处理器做出响应
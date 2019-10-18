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

#### 构造方法
```java
    public StandardSession(Manager manager) {
        this.lastAccessedTime = this.creationTime;
        this.listeners = new ArrayList();
        this.manager = null;
        this.maxInactiveInterval = -1;
        this.isNew = false;
        this.isValid = false;
        this.notes = new HashMap();
        this.principal = null;
        this.support = new PropertyChangeSupport(this);
        this.thisAccessedTime = this.creationTime;
        this.manager = manager;
        if (manager instanceof ManagerBase) {
            this.debug = ((ManagerBase)manager).getDebug();
        }

    }
```
- 通过构造方法设置管理器，保证每个`Session`都有管理器

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
    /**
     * Session 事件的监听器
     */
    private transient ArrayList listeners;

    /**
     * 管理器
     */
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

#### `getSession`
```java
    public HttpSession getSession() {
        if (this.facade == null) {
            this.facade = new StandardSessionFacade(this);
        }

        return this.facade;
    }
```
- `#getSession`方法返回一个`StandardSession`的外观类。

#### `expire`
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
- 一个`Session`如果在`maxInactiveInterval`时间内没有被进入则会使用此方法置为无效
- `<1>`处，观察者模式实现，触发事件，由事件处理器做出响应

### 管理器

#### `Manager`接口

```java
public interface Manager {
    Container getContainer();

    void setContainer(Container var1);

    boolean getDistributable();

    void setDistributable(boolean var1);

    String getInfo();

    int getMaxInactiveInterval();

    void setMaxInactiveInterval(int var1);

    void add(Session var1);

    void addPropertyChangeListener(PropertyChangeListener var1);

    Session createSession();

    Session findSession(String var1) throws IOException;

    Session[] findSessions();

    void load() throws ClassNotFoundException, IOException;

    void remove(Session var1);

    void removePropertyChangeListener(PropertyChangeListener var1);

    void unload() throws IOException;
}
```

- 定义了用来管理session的基本接口。包括:`#createSession`,`#findSession`,`#add`,`#remove`等对session操作的方法；还有`#getMaxActive`,`#setMaxActive`,`getActiveSessions`活跃会话的管理；还有Session有效期的接口；以及与`Container`相关联的接口

#### `ManagerBase`

- 实现了`Manager`接口，提供了基本的功能,使用`HashMap`存放session,提供了对session的`create\find\add\remove`功能，并且在`#createSession`中使用类`#generateSessionId`方法来生成会话id,作为`session`的唯一标识

#### `StandardManager`类

- 继承了`ManagerBase`类，Tomcat默认的Session管理器。对session提供了持久化功能，tomcat关闭的时候会将`session`保存到`javax.servlet.context.tempdir`路径下的`SESSIONS.ser`，启动的时候从此文件中加载`session`

## Session的生命周期

### 解析获取requestedSessionId

- 当我们调用`#Request.getSession()`时,最终调用`#HttpRequestBase.doGetSession`方法，代码如下：

```java
// HttpRequestBase.java
    private HttpSession doGetSession(boolean create) {
        if (this.context == null) {
            return null;
        } else {
            if (this.session != null && !this.session.isValid()) {
                this.session = null;
            }

            if (this.session != null) {		// <1>
                return this.session.getSession();
            } else {
                Manager manager = null;
                if (this.context != null) {
                    manager = this.context.getManager();
                }

                if (manager == null) {
                    return null;
                } else {
                    if (this.requestedSessionId != null) {	// <2>
                        try {
                            this.session = manager.findSession(this.requestedSessionId);
                        } catch (IOException var4) {
                            this.session = null;
                        }

                        if (this.session != null && !this.session.isValid()) {
                            this.session = null;
                        }

                        if (this.session != null) {
                            return this.session.getSession();
                        }
                    }

                    if (!create) {	// <3>
                        return null;
                    } else if (this.context != null && this.response != null && this.context.getCookies() && this.response.getResponse().isCommitted()) {
                        throw new IllegalStateException(RequestBase.sm.getString("httpRequestBase.createCommitted"));
                    } else {
                        this.session = manager.createSession();
                        return this.session != null ? this.session.getSession() : null;
                    }
                }
            }
        }
    }
```

- `<1>`处，如果`session`存在，直接返回
- `<2>`处，如果session不存在但`requestedSessionId`存在，通过`Manager`管理器类获取`Session`
- `<3>`处，上述都没找到对应`Session`,判断是否新建`Session`，如果条件满足则新建。
- 关于`requestSessionId`，tomcat支持从`cookie`和`url`中获取

```java
// CoyoteAdapter.postParseRequest方法
    protected boolean postParseRequest(Request req, org.apache.catalina.connector.Request request, Response res, org.apache.catalina.connector.Response response) throws IOException, ServletException {
  			// ...
            String sessionID;
            if (request.getServletContext().getEffectiveSessionTrackingModes().contains(SessionTrackingMode.URL)) {
                sessionID = request.getPathParameter(SessionConfig.getSessionUriParamName(request.getContext()));
                if (sessionID != null) {
                    request.setRequestedSessionId(sessionID);
                    request.setRequestedSessionURL(true);
                }
            }

            try {
                this.parseSessionCookiesId(request);	// <1>
            } catch (IllegalArgumentException var20) {
                if (!response.isError()) {
                    response.setError();
                    response.sendError(400);
                }

                return true;
            }
        	//...
    }
```

- 首先去`url`中解析`sessionId`，如果找不到则从`cookie`中获取（代码`<1>`）。
- 从`cookie`中获取sessionId代码如下:

```java
// CoyoteAdapter.java
    protected void parseSessionCookiesId(org.apache.catalina.connector.Request request) {
        Context context = request.getMappingData().context;
        if (context == null || context.getServletContext().getEffectiveSessionTrackingModes().contains(SessionTrackingMode.COOKIE)) {
            ServerCookies serverCookies = request.getServerCookies();
            int count = serverCookies.getCookieCount();
            if (count > 0) {
                String sessionCookieName = SessionConfig.getSessionCookieName(context);

                for(int i = 0; i < count; ++i) {
                    ServerCookie scookie = serverCookies.getCookie(i);
                    if (scookie.getName().equals(sessionCookieName)) {
                        if (!request.isRequestedSessionIdFromCookie()) {
                            this.convertMB(scookie.getValue());
                            request.setRequestedSessionId(scookie.getValue().toString());
                            request.setRequestedSessionCookie(true);
                            request.setRequestedSessionURL(false);
                            if (log.isDebugEnabled()) {
                                log.debug(" Requested cookie session id is " + request.getRequestedSessionId());
                            }
                        } else if (!request.isRequestedSessionIdValid()) {
                            this.convertMB(scookie.getValue());
                            request.setRequestedSessionId(scookie.getValue().toString());
                        }
                    }
                }
            }
        }
    }
```

- 遍历`cookie`，找到`name`=`jsessionid`的值赋值给`request`的`requestedSessionId`属性

### `findSession`查找`Session`

```java
//  ManagerBase.java
    protected HashMap sessions = new HashMap();

    public Session findSession(String id) throws IOException {
        if (id == null) {
            return null;
        } else {
            HashMap var2 = this.sessions;
            synchronized(var2) {
                Session session = (Session)this.sessions.get(id);
                return session;
            }
        }
    }
```

- 以`StandardManager`为例，其中`session`变量继承自`ManagerBase`类

### `createSession`创建`Session`

```java
//  ManagerBase.java
    public Session createSession() {
        Session session = null;
        ArrayList var2 = this.recycled;
        synchronized(var2) {
            int size = this.recycled.size();
            if (size > 0) {
                session = (Session)this.recycled.get(size - 1);
                this.recycled.remove(size - 1);
            }
        }

        if (session != null) {
            ((Session)session).setManager(this);
        } else {
            session = new StandardSession(this);
        }

        ((Session)session).setNew(true);
        ((Session)session).setValid(true);
        ((Session)session).setCreationTime(System.currentTimeMillis());
        ((Session)session).setMaxInactiveInterval(this.maxInactiveInterval);
        String sessionId = this.generateSessionId();	// <1> 
        String jvmRoute = this.getJvmRoute();
        if (jvmRoute != null) {
            sessionId = sessionId + '.' + jvmRoute;
            ((Session)session).setId(sessionId);
        }

        ((Session)session).setId(sessionId);	// <2>
        return (Session)session;
    }
```

- `<1>`处，调用`#generateSessionId`方法生成`session`的`Id`

```java
//  ManagerBase.java
    protected synchronized String generateSessionId() {
        Random random = this.getRandom();
        byte[] bytes = new byte[16];
        this.getRandom().nextBytes(bytes);
        bytes = this.getDigest().digest(bytes);
        StringBuffer result = new StringBuffer();

        for(int i = 0; i < bytes.length; ++i) {
            byte b1 = (byte)((bytes[i] & 240) >> 4);
            byte b2 = (byte)(bytes[i] & 15);
            if (b1 < 10) {
                result.append((char)(48 + b1));
            } else {
                result.append((char)(65 + (b1 - 10)));
            }

            if (b2 < 10) {
                result.append((char)(48 + b2));
            } else {
                result.append((char)(65 + (b2 - 10)));
            }
        }

        return result.toString();
    }
```



- <2>处，setId方法中，将session保存`session`的`HashMap`中，代码如下:

```java
// StandardSession    
public void setId(String id) {
        if (this.id != null && this.manager != null) {
            this.manager.remove(this);
        }

        this.id = id;
        if (this.manager != null) {
            this.manager.add(this);
        }
        // ... 省略部分代码
    }
```



### 销毁Session

- Tomcat会定期检测处不活跃的`session`,处于内存及安全性的考虑会将其删除，启动Tomcat的同时会启动一个后台线程检测。

```java
// PersistentManagerBase
    public void run() {
        while(!this.threadDone) {
            this.threadSleep();	// 睡眠 checkInterval 秒
            this.processExpires();
            this.processPersistenceChecks();
        }
    }
```

- 在`PersistentManagerBase`类中，在`#start`方法中会调用`#ThreadStart`方法，然后启动线程。最终调用`#processExpires`方法检测过期`session`

```java
    protected void processExpires() {
        if (this.started) {
            long timeNow = System.currentTimeMillis();
            Session[] sessions = this.findSessions();

            for(int i = 0; i < sessions.length; ++i) {
                StandardSession session = (StandardSession)sessions[i];
                if (session.isValid() && this.isSessionStale(session, timeNow)) {
                    session.expire();
                }
            }
        }
    }
```
- 如果session失效或者过期，调用session的`#expire`方法

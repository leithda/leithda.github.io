---
title: Tomcat-生命周期
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 1950977268
date: 2019-10-11 00:00:00
---

Catalina 由多个组件组成,当 Catalina 启动的时候,这些组件也会启动。当Catalina 停止的时候,这些组件也必须有机会被清除。例如,当一个容器停止 工作的时候,它必须唤醒所有加载的 servlet 的 destroy 方法,而 session 管理 器要保存 session 到二级存储器中。保持组件启动和停止一致的的机制通过实现 org.apache.catalina.Lifecycle 接口来实现
<!-- More -->

## 解析
### `Lifecycle`
```java
package org.apache.catalina;

public interface Lifecycle {
    String START_EVENT = "start";
    String BEFORE_START_EVENT = "before_start";
    String AFTER_START_EVENT = "after_start";
    String STOP_EVENT = "stop";
    String BEFORE_STOP_EVENT = "before_stop";
    String AFTER_STOP_EVENT = "after_stop";

    void addLifecycleListener(LifecycleListener var1);

    LifecycleListener[] findLifecycleListeners();

    void removeLifecycleListener(LifecycleListener var1);

    void start() throws LifecycleException;

    void stop() throws LifecycleException;
}
```
- 最主要的方法是`#start`和`#stop`，一些容器提供了这些方法的实现，所以它的父组件可以通过这些方法启动和停止他们
- 另外几个方法和监听器有关

### `LifecycleEvent`
```java
package org.apache.catalina;

public final class LifecycleEvent extends EventObject {
    private Object data;
    private Lifecycle lifecycle;
    private String type;

    public LifecycleEvent(Lifecycle lifecycle, String type) {
        this(lifecycle, type, (Object)null);
    }

    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {
        super(lifecycle);
        this.data = null;
        this.lifecycle = null;
        this.type = null;
        this.lifecycle = lifecycle;
        this.type = type;
        this.data = data;
    }

    // getter方法 
}
```
- LifecycleEvnet表示一个生命周期事件

### `LifecycleListener`
```java
package org.apache.catalina;

public interface LifecycleListener {
    void lifecycleEvent(LifecycleEvent var1);
}
```

### `LifecycleSupport`
```java
package org.apache.catalina.util;

import org.apache.catalina.Lifecycle;
import org.apache.catalina.LifecycleEvent;
import org.apache.catalina.LifecycleListener;

public final class LifecycleSupport {
    private Lifecycle lifecycle = null;
    /**
     * 监听器数组
     */
    private LifecycleListener[] listeners = new LifecycleListener[0];

    public LifecycleSupport(Lifecycle lifecycle) {
        this.lifecycle = lifecycle;
    }

    /**
     * [添加监听器]
     * @param listener [监听器]
     */
    public void addLifecycleListener(LifecycleListener listener) {
        LifecycleListener[] var2 = this.listeners;
        synchronized(var2) {
            LifecycleListener[] results = new LifecycleListener[this.listeners.length + 1];

            for(int i = 0; i < this.listeners.length; ++i) {
                results[i] = this.listeners[i];
            }

            results[this.listeners.length] = listener;
            this.listeners = results;
        }
    }

    public LifecycleListener[] findLifecycleListeners() {
        return this.listeners;
    }

    /**
     * [生命周期事件处理]
     * @param type [事件类型]
     * @param data [对象]
     */
    public void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this.lifecycle, type, data);
        LifecycleListener[] interested = null;
        LifecycleListener[] var5 = this.listeners;
        synchronized(var5) {
            interested = (LifecycleListener[])this.listeners.clone();
        }

        for(int i = 0; i < interested.length; ++i) {
            interested[i].lifecycleEvent(event);
        }

    }

    public void removeLifecycleListener(LifecycleListener listener) {
        LifecycleListener[] var2 = this.listeners;
        synchronized(var2) {
            int n = -1;

            for(int i = 0; i < this.listeners.length; ++i) {
                if (this.listeners[i] == listener) {
                    n = i;
                    break;
                }
            }

            if (n >= 0) {
                LifecycleListener[] results = new LifecycleListener[this.listeners.length - 1];
                int j = 0;

                for(int i = 0; i < this.listeners.length; ++i) {
                    if (i != n) {
                        results[j++] = this.listeners[i];
                    }
                }

                this.listeners = results;
            }
        }
    }
}
```

## 代码
### `SimpleContext`
```java
public class SimpleContext implements Context, Pipeline, Lifecycle {

    public SimpleContext() {
        pipeline.setBasic(new SimpleContextValve());
    }

    //子容器
    protected HashMap children = new HashMap();
    //类加载器
    private Loader loader = null;
    //容器生命周期事件支持类
    protected LifecycleSupport lifecycle = new LifecycleSupport(this);
    //管道线
    private SimplePipeline pipeline = new SimplePipeline(this);
    //servlet路基映射
    private HashMap servletMappings = new HashMap();
    //默认协议映射器
    protected Mapper mapper = null;
    //协议映射器集合
    protected HashMap mappers = new HashMap();
    //父容器
    private Container parent = null;
    //启动状态标记
    protected boolean started = false;

    public Object[] getApplicationListeners() {
        return lifecycle.findLifecycleListeners();
    }

    public void setApplicationListeners(Object listeners[]) {
        for (Object listener : listeners) {
            if(listener instanceof LifecycleListener){
                lifecycle.addLifecycleListener((LifecycleListener) listener);
            }
        }
    }

    // ... 省略部分代码

    public void addServletMapping(String pattern, String name) {
        synchronized (servletMappings) {
            servletMappings.put(pattern, name);
        }
    }

    // ... 省略部分代码

    public String findServletMapping(String pattern) {
        synchronized (servletMappings) {
            return ((String) servletMappings.get(pattern));
        }
    }

    // ... 省略部分代码


    // =========================Container接口的实现=====================================

    // ... 省略部分代码
    public void addMapper(Mapper mapper) {
    // this method is adopted from addMapper in ContainerBase
    // the first mapper added becomes the default mapper
    // 该方法通过ContainerBase父类调用
    // 第一个映射器将成为默认映射器
    mapper.setContainer((Container) this);      // May throw IAE
    this.mapper = mapper;
    synchronized(mappers) {
      // 如果协议对应的映射器不唯一，抛异常
      if (mappers.get(mapper.getProtocol()) != null)
        throw new IllegalArgumentException("addMapper:  Protocol '" +
          mapper.getProtocol() + "' is not unique");
      
      // 设置映射器关联当前的容器
      mapper.setContainer((Container) this);      // May throw IAE
      
      // 添加到映射器数组
      mappers.put(mapper.getProtocol(), mapper);
      if (mappers.size() == 1)
        this.mapper = mapper;
      else
        this.mapper = null;
    }
    }

    public void addPropertyChangeListener(PropertyChangeListener listener) {
    }

    public Container findChild(String name) {
        if (name == null)
            return (null);
        synchronized (children) {       // Required by post-start changes
            return ((Container) children.get(name));
        }
    }

    public Container[] findChildren() {
        synchronized (children) {
            Container results[] = new Container[children.size()];
            return ((Container[]) children.values().toArray(results));
        }
    }

    public ContainerListener[] findContainerListeners() {
        return null;
    }

    public Mapper findMapper(String protocol) {
        // the default mapper will always be returned, if any,
        // regardless the value of protocol
        if (mapper != null)
            return (mapper);
        else
            synchronized (mappers) {
                return ((Mapper) mappers.get(protocol));
            }
    }

    public Mapper[] findMappers() {
        return null;
    }

    public void invoke(Request request, Response response)
            throws IOException, ServletException {
        pipeline.invoke(request, response);
    }

    public Container map(Request request, boolean update) {
        //this method is taken from the map method in org.apache.cataline.core.ContainerBase
        //the findMapper method always returns the default mapper, if any, regardless the
        //request's protocol
        Mapper mapper = findMapper(request.getRequest().getProtocol());
        if (mapper == null)
            return (null);

        // Use this Mapper to perform this mapping
        return (mapper.map(request, update));
    }

    // ... 省略部分代码

    // =========================Lifecycle接口的实现=====================================
    // implementation of the Lifecycle interface's methods
    // 实现生命周期接口的注册监听器方法
    public void addLifecycleListener(LifecycleListener listener) {
        // 使用生命周期支持类注册监听器（关联当前容器的事件和监听器关联）
        lifecycle.addLifecycleListener(listener);
    }

    public LifecycleListener[] findLifecycleListeners() {
        return null;
    }

    public void removeLifecycleListener(LifecycleListener listener) {
        // 使用生命周期支持类注销监听器
        lifecycle.removeLifecycleListener(listener);
    }
    
    /**
    * 实现生命周期接口的start方法
    * 其中try代码块分别启动了与之相关联的生命周期组件，包括在载入器、管道和映射器等
    * 这就是所谓的单一启动/关闭机制。使用这种机制，只需启动最高层级的组件即可
    */
    public synchronized void start() throws LifecycleException {
        if (started)
            throw new LifecycleException("SimpleContext has already started");

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_START_EVENT, null);
        started = true;
        try {
            // Start our subordinate components, if any
            if ((loader != null) && (loader instanceof Lifecycle))
                ((Lifecycle) loader).start();

            // Start our child containers, if any
            Container children[] = findChildren();
            for (int i = 0; i < children.length; i++) {
                if (children[i] instanceof Lifecycle)
                    ((Lifecycle) children[i]).start();
            }

            // Start the Valves in our pipeline (including the basic),
            // if any
            if (pipeline instanceof Lifecycle)
                ((Lifecycle) pipeline).start();
            // Notify our interested LifecycleListeners
            lifecycle.fireLifecycleEvent(START_EVENT, null);
        } catch (Exception e) {
            e.printStackTrace();
        }

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
    }

    /**
     * 停止方法
     */
    public void stop() throws LifecycleException {
        if (!started)
            throw new LifecycleException("SimpleContext has not been started");
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_STOP_EVENT, null);
        lifecycle.fireLifecycleEvent(STOP_EVENT, null);
        started = false;
        try {
            // Stop the Valves in our pipeline (including the basic), if any
            if (pipeline instanceof Lifecycle) {
                ((Lifecycle) pipeline).stop();
            }

            // Stop our child containers, if any
            Container children[] = findChildren();
            for (int i = 0; i < children.length; i++) {
                if (children[i] instanceof Lifecycle)
                    ((Lifecycle) children[i]).stop();
            }
            if ((loader != null) && (loader instanceof Lifecycle)) {
                ((Lifecycle) loader).stop();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_STOP_EVENT, null);
    }
    // =========================Lifecycle接口的实现=====================================
}
```

- `#start`方法会启动所有子容器及相关组件
    ```java
        /**
     * 启动方法
     * @throws LifecycleException 异常
     */
    public synchronized void start() throws LifecycleException {
        if (started)
            throw new LifecycleException("SimpleContext has already started");

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_START_EVENT, null); // <1>
        started = true;
        try {
            // Start our subordinate components, if any
            if ((loader != null) && (loader instanceof Lifecycle))
                ((Lifecycle) loader).start();

            // Start our child containers, if any
            Container children[] = findChildren();
            for (int i = 0; i < children.length; i++) {
                if (children[i] instanceof Lifecycle)
                    ((Lifecycle) children[i]).start();
            }

            // Start the Valves in our pipeline (including the basic),
            // if any
            if (pipeline instanceof Lifecycle)
                ((Lifecycle) pipeline).start();
            // Notify our interested LifecycleListeners
            lifecycle.fireLifecycleEvent(START_EVENT, null);    //<2>
        } catch (Exception e) {
            e.printStackTrace();
        }

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null); // <3>
    }
    ```
    - `<1>`处，在重复启动检查后，唤醒启动前[BEFORE_START_EVENT]事件
    - `<2>`处，首先启动所有子容器，然后启动流水线组件，最后唤醒启动[START_EVENT]事件
    - `<3>`处，启动事件完成且无异常，唤醒启动后[AFTER_START_EVENT]事件
- `#stop`方法会停止所有子容器和所有组件,流程同`#start`一致
- 这样设计后，只要启动最高层的组件就可以实现整个容器的启动

### `SimpleContextLifecycleListener`
```java
public class SimpleContextLifecycleListener implements LifecycleListener {
    public void lifecycleEvent(LifecycleEvent event) {
        Lifecycle lifecycle = event.getLifecycle();
        System.out.println("SimpleContextLifecycleListener's event " + event.getType().toString());
        if (Lifecycle.START_EVENT.equals(event.getType())) {
            System.out.println("启动容器.");
        } else if (Lifecycle.STOP_EVENT.equals(event.getType())) {
            System.out.println("终止容器.");
        }
    }
}
```

### `SimpleLoader`
- 代码同第五章一致，区别就是实现了`Lifecycle`接口，`#start`启动方法打印了一些字符串到控制台上。

### `SimplePipeline`
- 同`SimpleLoader`处理方法一致

### `SimpleWrapper`
```java
// SimpleWrapper.java

    public synchronized void start() throws LifecycleException {
        System.out.println("Starting Wrapper " + name);
        if (started)
            throw new LifecycleException("Wrapper already started");

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_START_EVENT, null);
        started = true;

        // Start our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(START_EVENT, null);
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
    }

    public void stop() throws LifecycleException {
        System.out.println("Stopping wrapper " + name);
        // Shut down our servlet instance (if it has been initialized)
        try {
            instance.destroy();
        } catch (Throwable t) {
        }
        instance = null;
        if (!started)
            throw new LifecycleException("Wrapper " + name + " not started");
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_STOP_EVENT, null);

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(STOP_EVENT, null);
        started = false;

        // Stop the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).stop();
        }

        // Stop our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle)) {
            ((Lifecycle) loader).stop();
        }

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_STOP_EVENT, null);
    }
```
- `SimpleWrapper`中的`#start`方法和`SimpleContext`中类似，启动组件并触发`BEFORE_START_EVENT`、`START_EVENT`、`AFTER_START_EVENT`事件。

### Bootstrap
```java
public class Bootstrap {
    public static void main(String[] args) {
        // 创建连接器
        Connector connector = new HttpConnector();

        // 创建servlet容器Primitive（最小的servlet容器，代表一个servlet）
        Wrapper wrapper1 = new SimpleWrapper();
        wrapper1.setName("Primitive");
        wrapper1.setServletClass("PrimitiveServlet");

        // 创建servlet容器Modern（最小的servlet容器，代表一个servlet）
        Wrapper wrapper2 = new SimpleWrapper();
        wrapper2.setName("Modern");
        wrapper2.setServletClass("ModernServlet");

        // 创建context servlet容器，包含两个servlet（wrapper1，wrapper2）
        Context context = new SimpleContext();
        context.addChild(wrapper1);
        context.addChild(wrapper2);

        // 添加映射器，负责查找context示例中的子容器来处理http请求
        Mapper mapper = new SimpleContextMapper();
        mapper.setProtocol("http");
        context.addMapper(mapper);

        // 添加context生命周期监听器
        LifecycleListener listener = new SimpleContextLifecycleListener();
        ((Lifecycle) context).addLifecycleListener(listener);

        // 添加类加载器
        Loader loader = new SimpleLoader();
        context.setLoader(loader);

        // context.addServletMapping(pattern, name);
        // 添加servlet映射
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");

        //将连接器和context容器关联
        connector.setContainer(context);
        try {
            // 连接器初始化
            connector.initialize();

            // 启动连接器
            ((Lifecycle) connector).start();

            //启动容器
            ((Lifecycle) context).start();

            // make the application wait until we press a key.
            System.in.read();

            //关闭容器
            ((Lifecycle) context).stop();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 测试
- 启动`Bootstrap.main`方法。依次访问`http://localhost:8080/Modern`和`http://localhost:8080/Primitive`
控制台输出如下
```
HttpConnector Opening server socket on all host IP addresses
HttpConnector[8080] Starting background thread
SimpleContextLifecycleListener's event before_start
启动加载器
Starting Wrapper Primitive
Starting Wrapper Modern
SimpleContextLifecycleListener's event start
启动容器.
SimpleContextLifecycleListener's event after_start
ModernServlet -- init
PrimitiveServlet init
PrimitiveServlet service
```
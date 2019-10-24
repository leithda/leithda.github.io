---
title: Tomcat-Tomcat的默认连接器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-9-29
abbrlink: 479209129
---

Tomcat 连接器是一个可以插入 servlet 容器的独立模块,已经存在相当多的连接器了,
包括 Coyote, mod_jk, mod_jk2 和 mod_webapp。一个 Tomcat 连接器必须符合以下条件:

1. 必须实现接口 org.apache.catalina.Connector。
2. 必须创建请求对象,该请求对象的类必须实现接口 org.apache.catalina.Request。
3. 必须创建响应对象,该响应对象的类必须实现接口 org.apache.catalina.Response。
<!-- More -->

## 解析
tomcat4 的默认连接器类似于第 3 章的简单连接器。

1. 等待前来的 HTTP 请求,创建 request 和 response 对象,然后把 request 和 response 对象传递给容器。
2. 连接器是通过调用接口`org.apache.catalina.Container` 的 `invoke` 方法来传递 `request` 和 `response` 对象。
3. 在 invoke 方法里边,容器加载 servlet,调用它的 service 方法,管理会话,记录出错日志等等

- 理解`Connector`与`Processor`[`HttpProcessor`需要查看源码] 之间的处理流程是关键

## 代码
### 默认容器类
```java
package cn.leithda.htw.chapter4.core;

import org.apache.catalina.*;

import javax.naming.directory.DirContext;
import javax.servlet.Servlet;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.beans.PropertyChangeListener;
import java.io.File;
import java.io.IOException;
import java.net.URL;
import java.net.URLClassLoader;
import java.net.URLStreamHandler;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-24
 * Description: 默认容器
 */
public class SimpleContainer implements Container {

    /**
     *  WEB_ROOT 静态资源根路径
     */
    String WEB_ROOT = System.getProperty("user.dir") + File.separator
            + "src"+File.separator+"main"+File.separator+"resources";

    public String getInfo() {
        return null;
    }

    public Loader getLoader() {
        return null;
    }

    public void setLoader(Loader loader) {

    }

    public Logger getLogger() {
        return null;
    }

    public void setLogger(Logger logger) {

    }

    public Manager getManager() {
        return null;
    }

    public void setManager(Manager manager) {

    }

    public Cluster getCluster() {
        return null;
    }

    public void setCluster(Cluster cluster) {

    }

    public String getName() {
        return null;
    }

    public void setName(String s) {

    }

    public Container getParent() {
        return null;
    }

    public void setParent(Container container) {

    }

    public ClassLoader getParentClassLoader() {
        return null;
    }

    public void setParentClassLoader(ClassLoader classLoader) {

    }

    public Realm getRealm() {
        return null;
    }

    public void setRealm(Realm realm) {

    }

    public DirContext getResources() {
        return null;
    }

    public void setResources(DirContext dirContext) {

    }

    public void addChild(Container container) {

    }

    public void addContainerListener(ContainerListener containerListener) {

    }

    public void addMapper(Mapper mapper) {

    }

    public void addPropertyChangeListener(PropertyChangeListener propertyChangeListener) {

    }

    public Container findChild(String s) {
        return null;
    }

    public Container[] findChildren() {
        return new Container[0];
    }

    public ContainerListener[] findContainerListeners() {
        return new ContainerListener[0];
    }

    public Mapper findMapper(String s) {
        return null;
    }

    public Mapper[] findMappers() {
        return new Mapper[0];
    }

    public void invoke(Request request, Response response) throws IOException, ServletException {
        String servletName = ((HttpServletRequest)request).getRequestURI();
        servletName = servletName.substring(servletName.lastIndexOf("/") + 1);
        URLClassLoader loader = null;
        try {
            URL[] urls = new URL[1];
            URLStreamHandler streamHandler = null;
            File classPath = new File(WEB_ROOT);
            String repository = (new URL("file", null, classPath.getCanonicalPath() + File.separator)).toString();
            urls[0] = new URL(null, repository, streamHandler);
            loader = new URLClassLoader(urls);
        } catch (IOException e) {
            System.out.println(e.toString());
        }
        Class myClass = null;
        try {
            myClass = loader.loadClass(servletName);
        } catch (ClassNotFoundException e) {
            System.out.println(e.toString());
        }

        Servlet servlet = null;

        try {
            servlet = (Servlet) myClass.newInstance();
            servlet.service((HttpServletRequest) request, (HttpServletResponse) response);
        } catch (Exception e) {
            System.out.println(e.toString());
        } catch (Throwable e) {
            System.out.println(e.toString());
        }
    }

    public Container map(Request request, boolean b) {
        return null;
    }

    public void removeChild(Container container) {
    }

    public void removeContainerListener(ContainerListener containerListener) {
    }

    public void removeMapper(Mapper mapper) {
    }

    public void removePropertyChangeListener(PropertyChangeListener propertyChangeListener) {
    }
}
```

### Bootstrap 启动类
```java
package cn.leithda.htw.chapter4.startup;

import cn.leithda.htw.chapter4.core.SimpleContainer;
import org.apache.catalina.connector.http.HttpConnector;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-24
 * Description: 启动类
 */
public class Bootstrap {
    public static void main(String[] args) {
        HttpConnector connector = new HttpConnector();  // <1>
        SimpleContainer container = new SimpleContainer();
        connector.setContainer(container);
        try {
            connector.initialize(); // <2>
            connector.start();  // <3>

            // make the application wait until we press any key.
            System.in.read();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

- `<1>`处,`HttpConnector`类实现了`Lifecycle`接口进行生命周期管理，创建`HttpConnector`后应该调用它的`#initiialize`和`start`方法
- `<2>`处，`#HttpConnector.initialize`方法，调用`#open`方法，通过工厂类创建一个`java.net.ServerSocket`实例
- `<3>`处，`#start`方法，开启线程处理方法`#threadStart()`，根据参数创建好最小线程，代码如下:
```java
    public void start() throws LifecycleException {
        if (this.started) {
            throw new LifecycleException(this.sm.getString("httpConnector.alreadyStarted"));
        } else {
            this.threadName = "HttpConnector[" + this.port + "]";
            this.lifecycle.fireLifecycleEvent("start", (Object)null);
            this.started = true;
            this.threadStart(); // <3.1>

            while(this.curProcessors < this.minProcessors && (this.maxProcessors <= 0 || this.curProcessors < this.maxProcessors)) {
                HttpProcessor processor = this.newProcessor();
                this.recycle(processor);
            }

        }
    }
```
 - `<3.1>`处，开启线程处理，接收`socket`输入，调用`#createProcessor`创建处理器，然后调用`#HttpProcessor.assign`方法，将`socket`注册到`HttpPprocessor`中
    
    - `HttpProcessor#createProcessor`中，如果调用`#HttpProcessor.newProcessor`方法创建新的处理器时,会调用`#start`方法开启线程处理
     ```java
        private HttpProcessor newProcessor() {
            HttpProcessor processor = new HttpProcessor(this, this.curProcessors++);
            if (processor instanceof Lifecycle) {
                try {
                    processor.start();  // 开启线程
                } catch (LifecycleException var3) {
                    this.log("newProcessor", var3);
                    return null;
                }
        }
    
            this.created.addElement(processor);
            return processor;
        }
     ```
    - `HttpProcessor#run`方法中,会调用`#process`方法进行处理。其中,`#await`方法会判断变量`available`，这个变量也是实现异步化的关键。最后调用`#HttpConnector.recycle`方法，将此进程放回容器线程栈中
    ```java
    public void run() {
        while(!this.stopped) {
            Socket socket = this.await();
            if (socket != null) {
                try {
                    this.process(socket);
                } catch (Throwable var5) {
                    this.log("process.invoke", var5);
            }
    
                this.connector.recycle(this);
            }
    }
    
        Object var6 = this.threadSync;
        synchronized(var6) {
            this.threadSync.notifyAll();
        }
    }
    ```
    - `#HttpProcessor.process`方法中，会调用`#HttpConnector.invoke`方法进行处理
    
        ```java
           private void process(Socket socket) {
                boolean ok = true;
                boolean finishResponse = true;
                SocketInputStream input = null;
                OutputStream output = null;
        
                try {
                    input = new SocketInputStream(socket.getInputStream(), this.connector.getBufferSize());
                } catch (Exception var27) {
                    this.log("process.create", var27);
                    ok = false;
                }
        
                this.keepAlive = true;
        
                while(!this.stopped && ok && this.keepAlive) {
                    finishResponse = true;
        
                    try {
                        this.request.setStream(input);
                        this.request.setResponse(this.response);
                        output = socket.getOutputStream();
                        this.response.setStream(output);
                        this.response.setRequest(this.request);
                        ((HttpServletResponse)this.response.getResponse()).setHeader("Server", SERVER_INFO);
                    } catch (Exception var26) {
                        this.log("process.create", var26);
                        ok = false;
                    }
        
                    // ... 处理流程有报错会将 ok 置为 false
                    
                    try {
                        this.response.setHeader("Date", FastHttpDateFormat.getCurrentDate());
                        if (ok) {
                            /** 调用 invoke方法进行处理*/
                            this.connector.getContainer().invoke(this.request, this.response);
                        }
                    } catch (ServletException var20) {
                        // ...
                    }
        
                    if (finishResponse) {
                        try {
                            this.response.finishResponse();
                        } catch (IOException var16) {
                            ok = false;
                        } catch (Throwable var17) {
                            this.log("process.invoke", var17);
                            ok = false;
                        }
        
                        try {
                            this.request.finishRequest();
                        } catch (IOException var14) {
                            ok = false;
                        } catch (Throwable var15) {
                            this.log("process.invoke", var15);
                            ok = false;
                        }
        
                        try {
                            if (output != null) {
                                output.flush();
                            }
                        } catch (IOException var13) {
                            ok = false;
                        }
                    }
        
                    // 长连接处理
                    if ("close".equals(this.response.getHeader("Connection"))) {
                        this.keepAlive = false;
                    }
        
                    this.status = 0;
                    this.request.recycle();
                    this.response.recycle();
                }
                // 判断是否关机
                try {
                    this.shutdownInput(input);
                    socket.close();
                } catch (IOException var11) {
                } catch (Throwable var12) {
                    this.log("process.invoke", var12);
                }
        
                socket = null;
            }
        ```
    
        
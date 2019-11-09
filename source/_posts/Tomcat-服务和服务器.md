---
title: Tomcat-服务和服务器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 561747658
date: 2019-11-09 00:00:00
---


服务器提供了Servlet的启动和停止机制，解决了一个服务只能解决一种处理的问题(前面只有一个连接器，通过端口8080处理http请求，不能添加另一个连接器来处理https的请求).
<!-- More -->


## Server 服务器
- `org.apache.catalina.Server`接口表示一个服务器，它提供了一个优雅的方式启动和停止系统。服务器使用了组件---服务(`Service`)，它用来持有组件，例如容器或者1(or 多个)连接器

```java
public interface Server {
    String getInfo();

    NamingResources getGlobalNamingResources();

    void setGlobalNamingResources(NamingResources var1);

    int getPort();

    void setPort(int var1);

    String getShutdown();

    void setShutdown(String var1);

    void addService(Service var1);

    void await();

    Service findService(String var1);

    Service[] findServices();

    void removeService(Service var1);

    void initialize() throws LifecycleException;
}
```

## StandardServer
### initialize() 方法

```java
    // StandardServer.java

    public void initialize() throws LifecycleException {
        if (this.initialized) { // <1>
            throw new LifecycleException(sm.getString("standardServer.initialize.initialized"));
        } else {
            this.initialized = true;

            for(int i = 0; i < this.services.length; ++i) {
                this.services[i].initialize();
            }

        }
    }
```
- `<1>`处，使用`initialized` 变量避免多次初始化。`#stop()`方法不会改变`initialized`变量的值，再次启动时不会进行初始化

### start() 方法
```java
    public void start() throws LifecycleException {
        if (this.started) {
            throw new LifecycleException(sm.getString("standardServer.start.started"));
        } else {
            this.lifecycle.fireLifecycleEvent("before_start", (Object)null);
            this.lifecycle.fireLifecycleEvent("start", (Object)null);
            this.started = true;
            Service[] var1 = this.services;
            synchronized(var1) {
                int i = 0;

                while(true) {
                    if (i >= this.services.length) {
                        break;
                    }

                    if (this.services[i] instanceof Lifecycle) {
                        ((Lifecycle)this.services[i]).start();
                    }

                    ++i;
                }
            }

            this.lifecycle.fireLifecycleEvent("after_start", (Object)null);
        }
    }
```
- `start()`会启动所有服务及其相关组件，同样使用`started`变量避免重复启动


###　stop() 方法

```java
    public void stop() throws LifecycleException {
        if (!this.started) {
            throw new LifecycleException(sm.getString("standardServer.stop.notStarted"));
        } else {
            this.lifecycle.fireLifecycleEvent("before_stop", (Object)null);
            this.lifecycle.fireLifecycleEvent("stop", (Object)null);
            this.started = false;

            for(int i = 0; i < this.services.length; ++i) {
                if (this.services[i] instanceof Lifecycle) {
                    ((Lifecycle)this.services[i]).stop();
                }
            }

            this.lifecycle.fireLifecycleEvent("after_stop", (Object)null);
        }
    }
```
- 重置`started`的值并停止所有服务

### await() 方法
- `#await()`方法负责整个部署的停止机制，代码如下：
```java
    public void await() {
        ServerSocket serverSocket = null;

        try {
            serverSocket = new ServerSocket(this.port, 1, InetAddress.getByName("127.0.0.1"));
        } catch (IOException var11) {
            System.err.println("StandardServer.await: create[" + this.port + "]: " + var11);
            var11.printStackTrace();
            System.exit(1);
        }

        while(true) {
            Socket socket;
            InputStream stream;
            while(true) {
                socket = null;
                stream = null;

                try {
                    socket = serverSocket.accept();
                    socket.setSoTimeout(10000);
                    stream = socket.getInputStream();
                    break;
                } catch (AccessControlException var12) {
                    System.err.println("StandardServer.accept security exception: " + var12.getMessage());
                } catch (IOException var13) {
                    System.err.println("StandardServer.await: accept: " + var13);
                    var13.printStackTrace();
                    System.exit(1);
                    break;
                }
            }

            StringBuffer command = new StringBuffer();

            int expected;
            for(expected = 1024; expected < this.shutdown.length(); expected += this.random.nextInt() % 1024) {
                if (this.random == null) {
                    this.random = new Random(System.currentTimeMillis());
                }
            }

            boolean match;
            while(expected > 0) {
                match = true;

                int ch;
                try {
                    ch = stream.read();
                } catch (IOException var10) {
                    System.err.println("StandardServer.await: read: " + var10);
                    var10.printStackTrace();
                    ch = -1;
                }

                if (ch < 32) {
                    break;
                }

                command.append((char)ch);
                --expected;
            }

            try {
                socket.close();
            } catch (IOException var9) {
            }

            match = command.toString().equals(this.shutdown);
            if (match) {
                try {
                    serverSocket.close();
                } catch (IOException var8) {
                }

                return;
            }

            System.err.println("StandardServer.await: Invalid command '" + command.toString() + "' received");
        }
    }
```

## 服务
- 一个`Service`表示一个服务，一个服务可以有一个容器或多个连接器。可以添加多个连接器并与容器关联.

```java
public interface Service {
    Container getContainer();

    void setContainer(Container var1);

    String getInfo();

    String getName();

    void setName(String var1);

    Server getServer();

    void setServer(Server var1);

    void addConnector(Connector var1);

    Connector[] findConnectors();

    void removeConnector(Connector var1);

    void initialize() throws LifecycleException;
}

```

## StandardService
- `StandardService`是`Service`接口的标准实现，初始化方法`#initialize()`初始化所有添加到该服务的连接器。该类实现了`Lifecycle`接口，调用它的`#start()`方法可以启动所有的连接器和容器.

### 容器和连接器
- 一个`StandardService`实例包含两种组件，一个容器和多个连接器，分别使用`container`与`connectors`保存。多个连接器使得Tomcat服务于多个协议.
```java
    private Connector[] connectors = new Connector[0];
    private Container container = null;
```

### 关联容器与连接器
- 要将一个容器与连接器关联，调用`#serContainer()`方法
```java
    // StanndardService.java

    public void setContainer(Container container) {
        Container oldContainer = this.container;
        if (oldContainer != null && oldContainer instanceof Engine) {
            ((Engine)oldContainer).setService((Service)null);
        }

        this.container = container;
        if (this.container != null && this.container instanceof Engine) {
            ((Engine)this.container).setService(this);
        }

        if (this.started && this.container != null && this.container instanceof Lifecycle) {
            try {
                ((Lifecycle)this.container).start();
            } catch (LifecycleException var7) {
            }
        }

        Connector[] var3 = this.connectors;
        synchronized(var3) {
            for(int i = 0; i < this.connectors.length; ++i) {
                this.connectors[i].setContainer(this.container);
            }
        }

        if (this.started && oldContainer != null && oldContainer instanceof Lifecycle) {
            try {
                ((Lifecycle)oldContainer).stop();
            } catch (LifecycleException var6) {
            }
        }

        this.support.firePropertyChange("container", oldContainer, this.container);
    }
```

### 添加/删除连接器

```java
    // StanndardService.java

    public void addConnector(Connector connector) {
        Connector[] var2 = this.connectors;
        synchronized(var2) {
            connector.setContainer(this.container);
            connector.setService(this);
            Connector[] results = new Connector[this.connectors.length + 1];
            System.arraycopy(this.connectors, 0, results, 0, this.connectors.length);
            results[this.connectors.length] = connector;
            this.connectors = results;
            if (this.initialized) {
                try {
                    connector.initialize();
                } catch (LifecycleException var7) {
                    var7.printStackTrace(System.err);
                }
            }

            if (this.started && connector instanceof Lifecycle) {
                try {
                    ((Lifecycle)connector).start();
                } catch (LifecycleException var6) {
                }
            }

            this.support.firePropertyChange("connector", (Object)null, connector);
        }
    }

    public void removeConnector(Connector connector) {
        Connector[] var2 = this.connectors;
        synchronized(var2) {
            int j = -1;

            for(int i = 0; i < this.connectors.length; ++i) {
                if (connector == this.connectors[i]) {
                    j = i;
                    break;
                }
            }

            if (j >= 0) {
                if (this.started && this.connectors[j] instanceof Lifecycle) {
                    try {
                        ((Lifecycle)this.connectors[j]).stop();
                    } catch (LifecycleException var9) {
                    }
                }

                this.connectors[j].setContainer((Container)null);
                connector.setService((Service)null);
                int k = 0;
                Connector[] results = new Connector[this.connectors.length - 1];

                for(int i = 0; i < this.connectors.length; ++i) {
                    if (i != j) {
                        results[k++] = this.connectors[i];
                    }
                }

                this.connectors = results;
                this.support.firePropertyChange("connector", connector, (Object)null);
            }
        }
    }
```

### 生命周期方法

#### initialize()
```java
    public void initialize() throws LifecycleException {
        if (this.initialized) {
            throw new LifecycleException(sm.getString("standardService.initialize.initialized"));
        } else {
            this.initialized = true;
            Connector[] var1 = this.connectors;
            synchronized(var1) {
                for(int i = 0; i < this.connectors.length; ++i) {
                    this.connectors[i].initialize();
                }

            }
        }
    }
```

#### start()
```java
    public void start() throws LifecycleException {
        if (this.started) {
            throw new LifecycleException(sm.getString("standardService.start.started"));
        } else {
            this.lifecycle.fireLifecycleEvent("before_start", (Object)null);
            System.out.println(sm.getString("standardService.start.name", this.name));
            this.lifecycle.fireLifecycleEvent("start", (Object)null);
            this.started = true;
            if (this.container != null) {
                Container var1 = this.container;
                synchronized(var1) {
                    if (this.container instanceof Lifecycle) {
                        ((Lifecycle)this.container).start();
                    }
                }
            }

            Connector[] var6 = this.connectors;
            synchronized(var6) {
                for(int i = 0; i < this.connectors.length; ++i) {
                    if (this.connectors[i] instanceof Lifecycle) {
                        ((Lifecycle)this.connectors[i]).start();
                    }
                }
            }

            this.lifecycle.fireLifecycleEvent("after_start", (Object)null);
        }
    }
```

#### stop()
```java
    public void stop() throws LifecycleException {
        if (!this.started) {
            throw new LifecycleException(sm.getString("standardService.stop.notStarted"));
        } else {
            this.lifecycle.fireLifecycleEvent("before_stop", (Object)null);
            this.lifecycle.fireLifecycleEvent("stop", (Object)null);
            System.out.println(sm.getString("standardService.stop.name", this.name));
            this.started = false;
            Connector[] var1 = this.connectors;
            synchronized(var1) {
                int i = 0;

                while(true) {
                    if (i >= this.connectors.length) {
                        break;
                    }

                    if (this.connectors[i] instanceof Lifecycle) {
                        ((Lifecycle)this.connectors[i]).stop();
                    }

                    ++i;
                }
            }

            if (this.container != null) {
                Container var7 = this.container;
                synchronized(var7) {
                    if (this.container instanceof Lifecycle) {
                        ((Lifecycle)this.container).stop();
                    }
                }
            }

            this.lifecycle.fireLifecycleEvent("after_stop", (Object)null);
        }
    }
```

## 测试
### 上下文监听器
- 同前几章节一样，创建上下文监听器类，跳过读取配置文件过程，设置配置完成标志为`true`

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
public final class Bootstrap {
    public static void main(String[] args) {

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

        Service service = new StandardService();
        service.setName("Stand-alone Service");
        Server server = new StandardServer();
        server.addService(service);
        service.addConnector(connector);

        //StandardService class's setContainer will call all its connector's setContainer method
        service.setContainer(engine);

        // Start the new server
        if (server instanceof Lifecycle) {
            try {
                server.initialize();
                ((Lifecycle) server).start();
                server.await(); // <1> 
                // the program waits until the await method returns,
                // i.e. until a shutdown command is received.
            } catch (LifecycleException e) {
                e.printStackTrace(System.out);
            }
        }

        // Shut down the server
        if (server instanceof Lifecycle) {
            try {
                ((Lifecycle) server).stop();
            } catch (LifecycleException e) {
                e.printStackTrace(System.out);
            }
        }
    }
}
```
- `<1>`处，调用`#server.await()`方法，等待关闭指令，收到关闭指令后，调用服务器的`#stop()`方法，关闭它的其他组件

### Stopper
```java
public class Stopper {
    public static void main(String[] args) {
        // the following code is taken from the Stop method of
        // the org.apache.catalina.startup.Catalina class
        int port = 8005;
        try {
            Socket socket = new Socket("127.0.0.1", port);
            OutputStream stream = socket.getOutputStream();
            String shutdown = "SHUTDOWN";
            for (int i = 0; i < shutdown.length(); i++)
                stream.write(shutdown.charAt(i));
            stream.flush();
            stream.close();
            socket.close();
            System.out.println("The server was successfully shut down.");
        }
        catch (IOException e) {
            System.out.println("Error. The server has not been started.");
        }
    }
}
```
- 发送关闭指令到8085端口，关闭tomcat服务器。

### 测试输出
```shell
HttpConnector Opening server socket on all host IP addresses
Starting service Stand-alone Service
Apache Tomcat/4.1.9
WebappLoader[/app1]: Deploying class repositories to work directory D:\work\Github\code_warehouse\java\opensource\how-tomcat-works\work\_\localhost\app1
WebappLoader[/app1]: Deploy class files /WEB-INF/classes to D:\work\Github\code_warehouse\java\opensource\how-tomcat-works\webapps\app1\WEB-INF\classes
StandardManager[/app1]: Seeding random number generator class java.security.SecureRandom
StandardManager[/app1]: Seeding of random number generator has been completed
HttpConnector[8080] Starting background thread
# 执行Stopper.main 后
Stopping service Stand-alone Service
HttpConnector[8080] Stopping background thread

############################################
The server was successfully shut down.
```

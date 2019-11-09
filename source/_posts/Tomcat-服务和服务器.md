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

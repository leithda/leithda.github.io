---
title: Tomcat-StandardContext
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 1923304143
date: 2019-10-22 00:00:00
---

标准上下文容器`StandardContext`对象的初始化和配置,然后讨论跟其相关的类`StandardContextMapper`(Tomcat 4)和 `ContextConfig` 类

<!-- More -->

## StandardContext 配置
- 一个`StandardContext`创建之后，调用`#start`方法，这样它就可以接收 HTTP 请求了，如果启动失败，设置`available`为false，`available`表示容器可用性
- `#start`方法启动成功，`StandardContext`会配置它的属性。它会读取和分析`%CATALINA_HOME%/conf`下的`web.xml。还需要增加验证器阀门和证书处阀门
- `StandardContext`加载配置通过事件监听器处理，它的属性`configured`用来表示容器是否被配置。之前的`SimpleContextConfig`直接更改标志，表示容器配置完成

## StandardContext 的构造函数
```java
    public StandardContext() {
        this.pipeline.setBasic(new StandardContextValve());
        this.namingResources.setContainer(this);
    }
```
- 构造函数中增加了一个基础阀门`StandardContextValve`

## 启动 StandardContext
- `#start`方法初始化`StandardContext`对象并让监听器监听这个实例,如果配置成功,监听器将`configured`属性设置为`true`
- 最后`#start`方法将`available`属性设置为`true`或者`false`.如果为`true`表示该`Standard`已经配置完成且所有子容器和组件已经启动成功

```java
    public synchronized void start() throws LifecycleException {
        if (this.started) {
            throw new LifecycleException(ContainerBase.sm.getString("containerBase.alreadyStarted", this.logName()));
        } else {
            // <1> 触发 before_start 事件
            this.lifecycle.fireLifecycleEvent("before_start", (Object)null);
            if (this.debug >= 1) {
                this.log("Processing start(), current available=" + this.getAvailable());
            }

            // <2> 设置 available configured 属性为false
            this.setAvailable(false);
            this.setConfigured(false);
            boolean ok = true;
            // <3> 设置源(resources)
            if (this.getResources() == null) {
                  try {
                    if (this.docBase != null && this.docBase.endsWith(".war")) {
                        this.setResources(new WARDirContext());
                    } else {
                        this.setResources(new FileDirContext());
                    }
                } catch (IllegalArgumentException var15) {}

                if (ok) {
                    DirContext dirContext = ((ProxyDirContext)this.resources).getDirContext();
                    if (dirContext != null && dirContext instanceof BaseDirContext) {
                        ((BaseDirContext)dirContext).allocate();
                    }
                }
            }

            // <4> 设置加载器
            if (this.getLoader() == null) {
                if (this.getPrivileged()) {
                    this.setLoader(new WebappLoader(this.getClass().getClassLoader()));
                } else {
                    this.setLoader(new WebappLoader(this.getParentClassLoader()));
                }
            }

            // <5> 设置管理器
            if (this.getManager() == null) {
                this.setManager(new StandardManager());
            }

            // <6> 初始化属性 map
            this.getCharsetMapper();
            this.postWorkDirectory();
            String useNamingProperty = System.getProperty("catalina.useNaming");
            if (useNamingProperty != null && useNamingProperty.equals("false")) {
                this.useNaming = false;
            }

            if (ok && this.isUseNaming() && this.namingContextListener == null) {
                this.namingContextListener = new NamingContextListener();
                this.namingContextListener.setDebug(this.getDebug());
                this.namingContextListener.setName(this.getNamingContextName());
                this.addLifecycleListener(this.namingContextListener);
            }

            ClassLoader oldCCL = this.bindThread();

            if (ok) {
                try {
                    // <7> 启动上下文相关组件
                    this.addDefaultMapper(this.mapperClass);
                    this.started = true;
                    if (this.loader != null && this.loader instanceof Lifecycle) {
                        ((Lifecycle)this.loader).start();
                    }

                    if (this.logger != null && this.logger instanceof Lifecycle) {
                        ((Lifecycle)this.logger).start();
                    }

                    this.unbindThread(oldCCL);
                    oldCCL = this.bindThread();
                    if (this.cluster != null && this.cluster instanceof Lifecycle) {
                        ((Lifecycle)this.cluster).start();
                    }

                    if (this.realm != null && this.realm instanceof Lifecycle) {
                        ((Lifecycle)this.realm).start();
                    }

                    if (this.resources != null && this.resources instanceof Lifecycle) {
                        ((Lifecycle)this.resources).start();
                    }

                    // <8> 启动子容器(包装器)
                    Mapper[] mappers = this.findMappers();

                    for(int i = 0; i < mappers.length; ++i) {
                        if (mappers[i] instanceof Lifecycle) {
                            ((Lifecycle)mappers[i]).start();
                        }
                    }

                    Container[] children = this.findChildren();

                    for(int i = 0; i < children.length; ++i) {
                        if (children[i] instanceof Lifecycle) {
                            ((Lifecycle)children[i]).start();
                        }
                    }

                    // <9> 启动流水线
                    if (this.pipeline instanceof Lifecycle) {
                        ((Lifecycle)this.pipeline).start();
                    }

                    // <10> 触发 start 事件
                    this.lifecycle.fireLifecycleEvent("start", (Object)null);
                    // <11> 启动管理器
                    if (this.manager != null && this.manager instanceof Lifecycle) {
                        ((Lifecycle)this.manager).start();
                    }
                } finally {
                    this.unbindThread(oldCCL);
                }
            }

            // 检查 conifgured 的值
            if (!this.getConfigured()) {
                ok = false;
            }

            if (ok) {
                this.getServletContext().setAttribute("org.apache.catalina.resources", this.getResources());
            }

            oldCCL = this.bindThread();
            if (ok) {
                // 设置欢迎文件
                this.postWelcomeFiles();    
            }

            if (ok && !this.listenerStart()) {
                ok = false;
            }

            if (ok && !this.filterStart()) {
                ok = false;
            }

            if (ok) {
                // 加载子容器(Wrapper)
                this.loadOnStartup(this.findChildren());    
            }

            this.unbindThread(oldCCL);
            if (ok) {
                // 设置 available 为 true
                this.setAvailable(true);    
            } else {
                this.log(ContainerBase.sm.getString("standardContext.startFailed"));

                try {
                    this.stop();
                } catch (Throwable var13) {}

                this.setAvailable(false);
            }

            // 触发 after_start 事件
            this.lifecycle.fireLifecycleEvent("after_start", (Object)null);
        }
    }
```

## invoke 方法
- `#invoke`方法由相关联的连接器调用，如果该上下文是一个主机(Host)的子容器，由该主机的`#invoke`方法调用。

```java
    public void invoke(Request request, Response response) throws IOException, ServletException {
        while(this.getPaused()) {   // <1>
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException var4) {
            }
        }

        super.invoke(request, response);    // <2>
    }
```
- `<1>`处，检查是否正在重加载应用程序，如果是，等待加载完成(1s)
- `<2>`处，调用父类`ContextBase`的`#invoke`方法
```java
    // ContextBase.java
    public void invoke(Request request, Response response) throws IOException, ServletException {
        this.pipeline.invoke(request, response);
    }
```

## StandardContextMapper 
- 对于每一个请求，`#invoke`方法都会调用流水线基本阀门(StaFdardContextValve)的`#invoke`方法
- `StandardContextValve`需要得到一个处理请求的包装器。`StandardContextValve`使用上下文容器中的`map`来查找合适的包装器。然后调用包装器的`#invoke`方法

### 添加映射
- `StandardContext`的父类`ContainerBase`定义了`#addDefaultMapper`方法添加默认映射
```java
//  ContainerBase.java
    protected void addDefaultMapper(String mapperClass) {
        if (mapperClass != null) {
            if (this.mappers.size() < 1) {
                try {
                    Class clazz = Class.forName(mapperClass);
                    Mapper mapper = (Mapper)clazz.newInstance();
                    mapper.setProtocol("http");
                    this.addMapper(mapper);
                } catch (Exception var4) {
                    this.log(sm.getString("containerBase.addDefaultMapper", mapperClass), var4);
                }

            }
        }
    }
```

- `StandardContext`在`#start`方法中调用它的`#addDefaultMapper`方法，传递一个`mapperClass`变量
```java
//  StandardContext.java
            if (ok) {
                try {
                    this.addDefaultMapper(this.mapperClass);
                }
            }
```
- `mapperClass`的定义如下:
```java
     private String mapperClass = "org.apache.catalina.core.StandardContextMapper"; 
```

### 关联容器
- `org.apache.catalina.Mapper` 接口的标准实现是`org.apache.catalina.core.StandardContextMapper`。只能和上下文容器关联，方法如下:
```java
//  StandardContextMapper.java
    public void setContainer(Container container) {
        if (!(container instanceof StandardContext)) {
            throw new IllegalArgumentException(sm.getString("httpContextMapper.container"));
        } else {
            this.context = (StandardContext)container;
        }
    }
```
- 映射器中最重要的是`#map`方法，它获得一个HTTP请求返回的子容器,方法签名如下:`public Container map(Request request, boolean update) `

## StandardContextValve
- `StandardContextValve`调用上下文容器的`#map`方法来处理 HTTP 请求，传递的参数是`org.apache.catalina.Request`对象，`#map`方法(`ContainerBase `类中)返回一个对应协议的映射器，然后调用映射器的`#map`方法
```java
    // ContainerBase.java
    public Container map(Request request, boolean update) {
        Mapper mapper = this.findMapper(request.getRequest().getProtocol());
        return mapper == null ? null : mapper.map(request, update);
    }
```

- `StandardMapper`的`#map`方法

  ```java
  	// StandardMapper.java
  
  	public Container map(Request request, boolean update) {
          int debug = this.context.getDebug();
          if (update && request.getWrapper() != null) {
              return request.getWrapper();
          } else {
              // 识别上下文的相关URL映射
              String contextPath = ((HttpServletRequest)request.getRequest()).getContextPath();
              String requestURI = ((HttpRequest)request).getDecodedRequestURI();
              String relativeURI = requestURI.substring(contextPath.length());
              if (debug >= 1) {
                  this.context.log("Mapping contextPath='" + contextPath + "' with requestURI='" + requestURI + "' and relativeURI='" + relativeURI + "'");
              }
  			// 通过匹配的方法获得一个包装器
              Wrapper wrapper = null;
              String servletPath = relativeURI;
              String pathInfo = null;
              String name = null;
              // <1> 规则1： 精确匹配
              if (wrapper == null) {
                  if (debug >= 2) {
                      this.context.log("  Trying exact match");
                  }
  
                  if (!relativeURI.equals("/")) {
                      name = this.context.findServletMapping(relativeURI);
                  }
  
                  if (name != null) {
                      wrapper = (Wrapper)this.context.findChild(name);
                  }
  
                  if (wrapper != null) {
                      servletPath = relativeURI;
                      pathInfo = null;
                  }
              }
  
              // <2> 规则2：前缀匹配
              int slash;
              if (wrapper == null) {
                  if (debug >= 2) {
                      this.context.log("  Trying prefix match");
                  }
  
                  servletPath = relativeURI;
  
                  while(true) {
                      name = this.context.findServletMapping(servletPath + "/*");
                      if (name != null) {
                          wrapper = (Wrapper)this.context.findChild(name);
                      }
  
                      if (wrapper != null) {
                          pathInfo = relativeURI.substring(servletPath.length());
                          if (pathInfo.length() == 0) {
                              pathInfo = null;
                          }
                          break;
                      }
  
                      slash = servletPath.lastIndexOf(47);
                      if (slash < 0) {
                          break;
                      }
  
                      servletPath = servletPath.substring(0, slash);
                  }
              }
  
              // <3> 规则3: 扩展匹配
              if (wrapper == null) {
                  if (debug >= 2) {
                      this.context.log("  Trying extension match");
                  }
  
                  slash = relativeURI.lastIndexOf(47);
                  if (slash >= 0) {
                      String last = relativeURI.substring(slash);
                      int period = last.lastIndexOf(46);
                      if (period >= 0) {
                          String pattern = "*" + last.substring(period);
                          name = this.context.findServletMapping(pattern);
                          if (name != null) {
                              wrapper = (Wrapper)this.context.findChild(name);
                          }
  
                          if (wrapper != null) {
                              servletPath = relativeURI;
                              pathInfo = null;
                          }
                      }
                  }
              }
  
              // <4> 规则4: 默认匹配
              if (wrapper == null) {
                  if (debug >= 2) {
                      this.context.log("  Trying default match");
                  }
  
                  name = this.context.findServletMapping("/");
                  if (name != null) {
                      wrapper = (Wrapper)this.context.findChild(name);
                  }
  
                  if (wrapper != null) {
                      servletPath = relativeURI;
                      pathInfo = null;
                  }
              }
  
              if (debug >= 1 && wrapper != null) {
                  this.context.log(" Mapped to servlet '" + wrapper.getName() + "' with servlet path '" + servletPath + "' and path info '" + pathInfo + "' and update=" + update);
              }
  
              if (update) {
                  request.setWrapper(wrapper);
                  ((HttpRequest)request).setServletPath(servletPath);
                  ((HttpRequest)request).setPathInfo(pathInfo);
              }
  
              return wrapper;
          }
      }
  ```

## 添加映射与包装器

```java
	// Bootstrap.java
		Wrapper wrapper1 = new StandardWrapper();
        wrapper1.setName("Primitive");
        wrapper1.setServletClass("PrimitiveServlet");
        Wrapper wrapper2 = new StandardWrapper();
        wrapper2.setName("Modern");
        wrapper2.setServletClass("ModernServlet");

        Context context = new StandardContext();
		// 添加包装器
        context.addChild(wrapper1);
        context.addChild(wrapper2);
       
		// 添加映射
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");
```

## 重加载支持

- `StandardContext` 定义了 `reloadable `属性来标识是否支持应用程序的重加载
- 允许重加载时，如果`web.xml`或者`WEB-INF/classes`目录下的文件改变的时候会重加载

### WebappLoader 加载器

- `Loader`接口的标准实现，通过`#setContainer`方法关联`StandardContext`代码如下:

```java
	// WebappLoader.java    

	public void setContainer(Container container) {
        if (this.container != null && this.container instanceof Context) {
            ((Context)this.container).removePropertyChangeListener(this);
        }

        Container oldContainer = this.container;
        this.container = container;
        this.support.firePropertyChange("container", oldContainer, this.container);
        if (this.container != null && this.container instanceof Context) {	// <1>
            this.setReloadable(((Context)this.container).getReloadable());
            ((Context)this.container).addPropertyChangeListener(this);
        }

    }
```

- `<1>`处，如果容器是上下文容器，设置重加载属性为容器重加载属性。

- `<2>`处，调用`#setReloadable`方法，代码如下:

  ```java
      // WebappLoader.java
  
  	public void setReloadable(boolean reloadable) {
          boolean oldReloadable = this.reloadable;
          this.reloadable = reloadable;
          this.support.firePropertyChange("reloadable", new Boolean(oldReloadable), new Boolean(this.reloadable));
          if (this.started) {
              if (!oldReloadable && this.reloadable) {	// <1>
                  this.threadStart();
              } else if (oldReloadable && !this.reloadable) {	// <2>
                  this.threadStop();
              }
  
          }
      }
  ```

  - `<1>`处，重加载属性由`false`变为`true`，调用`#threadStart()`方法
  - `<2>`处，重加载属性由`true`变为`false`，调用`#threadStop()`方法

### backgroundProcess 方法

- 一个上下文容器需要加载器和管理器的支持。这些组件需要一个单独的线程来处理后台过程(background process)。例如: 加载器通过一个线程检查类文件和 JAR 文件的时间戳来支持重加载、管理器需要线程检查 Session 过期时间进行过期处理
- 为了节省资源，Tomcat使用了一种不同的方式处理。所有的后台过程都分享同一个线程，如果一个组件或者容器需要定期的来执行操作，它 需要做的就是将代码写入到`#backgroundProcess`方法

- 共享线程由`ContainerBase`对象创建，在它的`#start`方法中调用`#threadStart`方法

```java
protected void threadStart() { 
  if (thread != null) 
    return; 
  if (backgroundProcessorDelay <= 0) 
    return; 
  threadDone = false; 
  String threadName = "ContainerBackgroundProcessor[" + toString() + 
    "]"; 
  thread = new Thread(new ContainerBackgroundProcessor(), threadName); 
  thread.setDaemon(true); 
  thread.start(); 
} 
```

- 方法`#threadStart`传递一个`ContainerBackgroundProcessor`对象创建一个新线程，它实现了`Runnable`接口，代码如下:

  ```java
  protected class ContainerBackgroundProcessor implements Runnable { 
    public void run() { 
      while (!threadDone) { 
        try { 
          Thread.sleep(backgroundProcessorDelay * 1000L); 
        } 
        catch (InterruptedException e) { 
          ; 
        } 
        if (!threadDone) { 
          Container parent = (Container) getMappingObject(); 
          ClassLoader cl = 
            Thread.currentThread().getContextClassLoader(); 
          if (parent.getLoader() != null) { 
            cl = parent.getLoader().getClassLoader(); 
          } 
          processChildren(parent, cl); 
        } 
      } 
    } 
    
    protected void processChildren(Container container, ClassLoader cl) { 
      try { 
        if (container.getLoader() != null) { 
          Thread.currentThread().setContextClassLoader 
            (container.getLoader().getClassLoader()); 
        } 
        container.backgroundProcess(); 
      } 
      catch (Throwable t) { 
        log.error("Exception invoking periodic operation: ", t); 
      } 
      finally { 
        Thread.currentThread().setContextClassLoader(cl); 
      } 
      Container[] children = container.findChildren(); 
      for (int i = 0; i < children.length; i++) { 
        if (children[i].getBackgroundProcessorDelay() <= 0) { 
          processChildren(children[i], cl); 
        } 
      } 
    } 
  } 
  
  ```

  - 是`ContainerBase`的内部类，`#run`方法中，有一个`while`循环定期调用它的`#processChildren`方法，`#processChildren`方法内调用`#backgroundProcess`方法来处理它的每个子容器的`#processChildren`方法。所以`ContainerBase`的子类可以有一个线程周期性的执行任务

  - 举个栗子: 检查时间戳或者Session对象的过期时间

    ```java
    // StandardContext.java in Tomcat5
    public void backgroundProcess() { 
      if (!started) 
        return; 
      count = (count + 1) % managerChecksFrequency; 
      if ((getManager() != null) && (count == 0)) { 
        try { 
          getManager().backgroundProcess(); 
        } 
        catch ( Exception x ) { 
          log.warn("Unable to perform background process on manager", x); 
        } 
      } 
      if (getloader() != null) { 
        if (reloadable && (getLoader().modified())) { 
          try { 
            Thread.currentThread().setContextClassloader 
              (StandardContext.class.getClassLoader()); 
            reload(); 
          } 
          finally { 
            if (getLoader() != null) { 
              Thread.currentThread().setContextClassLoader 
                (getLoader().getClassLoader()); 
            } 
          } 
        } 
        if (getLoader() instanceof WebappLoader) { 
          ((WebappLoader) getLoader()).closeJARs(false); 
        } 
      } 
    } 
    
    ```

    


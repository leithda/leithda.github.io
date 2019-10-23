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

## StandardContextMapper 
- 对于每一个请求，`#invoke`方法都会调用流水线基本阀门(StaFdardContextValve)的`#invoke`方法
- `StandardContextValve`需要得到一个处理请求的包装器
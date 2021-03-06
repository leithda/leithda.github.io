---
title: Tomcat-部署器
abbrlink: 2507917456
date: 2019-12-24 16:03:28
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
---

要使得一个web应用可以访问,一个上下文必须先部署在主机上。在 Tomcat 中,一个上下文可以以 WAR 文件的形式部署,也可以直接将整个应用程序部署在 Tomcat 安装目录的 wabapp 目录下面
<!-- More -->

## 部署一个Web上下文
在[服务与服务器]()章节中，我们通过如下方式初始化一个`StandardHost`并将上下文对象作为子容器添加到上面
```java
        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/app1");
        context.setDocBase("app1");

        Host host = new StandardHost();
        host.addChild(context);
```
- 在实际部署中，并没有这些代码，Tomcat是通过监听机制实现部署和安装Web应用程序的。具体细节如下：
- 前面介绍过，当 Catalina启动时，会使用`Digester`会解析`server.xml`文档中的XML元素转换为Java对象。Catalina累定义了createStartDigester方法用于添加Digester规则。下面是方法中的一行   
`digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));`

```java
// HostRuleSet.java
// addRuleInstances(Digester digester) 方法片段

digester.addObjectCreate(this.prefix + "Host", "org.apache.catalina.core.StandardHost", "className");
digester.addSetProperties(this.prefix + "Host");
digester.addRule(this.prefix + "Host", new CopyParentClassLoaderRule(digester));
digester.addRule(this.prefix + "Host", new LifecycleListenerRule(digester, "org.apache.catalina.startup.HostConfig", "hostConfigClass")); // <1>
```
- HostRuleSet 继承了 RuleSetBase,实现了addRuleInstances来定义解析规则。
- `<1>`处，创建一个HostConfig对象，并将其作为生命周期监听器添加到host上。
- HostConfig的lifecycleEvent方法如下
    ```java
    public void lifecycleEvent(LifecycleEvent event) {
        try {
            this.host = (Host)event.getLifecycle();
            if (this.host instanceof StandardHost) {    // <1>
                int hostDebug = ((StandardHost)this.host).getDebug();
                if (hostDebug > this.debug) {
                    this.debug = hostDebug;
                }

                this.setDeployXML(((StandardHost)this.host).isDeployXML());
                this.setLiveDeploy(((StandardHost)this.host).getLiveDeploy());
                this.setUnpackWARs(((StandardHost)this.host).isUnpackWARs());
            }
        } catch (ClassCastException var3) {
            this.log(sm.getString("hostConfig.cce", event.getLifecycle()), var3);
            return;
        }

        if (event.getType().equals("start")) {
            this.start();   // <2>
        } else if (event.getType().equals("stop")) {
            this.stop();
        }

    }
    ```
    - `<1>`处，如果主机是`StandardHost`的实例，调用三个方法，`isDeployXML`标志该主机是否部署一个上下文部署文件；`getLiveDeploy`默认是`true`,标志是否需要周期性检查新部署;`isUnpackWARs`属性定义了是否需要解压部署`WAR`文件
    - `<2>`处，当收到主机的`start`事件时，调用`start`来部署应用程序。代码如下：
        ```java
        protected void start() {
            if (this.debug >= 1) {
                this.log(sm.getString("hostConfig.start"));
            }

            if (this.host.getAutoDeploy()) {
                this.deployApps();  // <1> 
            }

            if (this.isLiveDeploy()) {
                this.threadStart(); // <2>
            }

        }
        ```
        - `<1>`处，如果`autoDeploy`属性为true,调用`#deployApps`方法发布应用。
        - `<2>`处，如果`liveDeploy`属性为true,调用`#threadStart()`方法启动一个新线程
        ```java
        protected void deployApps() {
            if (this.host instanceof Deployer) {
                if (this.debug >= 1) {
                    this.log(sm.getString("hostConfig.deploying"));
                }

                File appBase = this.appBase();
                if (appBase.exists() && appBase.isDirectory()) {
                    String[] files = appBase.list();
                    this.deployDescriptors(appBase, files);
                    this.deployWARs(appBase, files);
                    this.deployDirectories(appBase, files);
                }
            }
        }
        ```

### Deploying a Descriptor
可以通过编写一个Xml文件描述一个上下文对象。
用来部署`%CATALINA_HOME%/webapps`(Tomcat4)或`%CATALINA_HOME%/server/webapps/`(Tomcat5)下的XML文件
```java
    protected void deployDescriptors(File appBase, String[] files) {
        if (this.deployXML) {
            for(int i = 0; i < files.length; ++i) {
                if (!files[i].equalsIgnoreCase("META-INF") && !files[i].equalsIgnoreCase("WEB-INF") && !this.deployed.contains(files[i])) {
                    File dir = new File(appBase, files[i]);
                    if (files[i].toLowerCase().endsWith(".xml")) {
                        this.deployed.add(files[i]);
                        String file = files[i].substring(0, files[i].length() - 4);
                        String contextPath = "/" + file;
                        if (file.equals("ROOT")) {
                            contextPath = "";
                        }

                        if (this.host.findChild(contextPath) == null) {
                            this.log(sm.getString("hostConfig.deployDescriptor", files[i]));

                            try {
                                URL config = new URL("file", (String)null, dir.getCanonicalPath());
                                ((Deployer)this.host).install(config, (URL)null);
                            } catch (Throwable var8) {
                                this.log(sm.getString("hostConfig.deployDescriptor.error", files[i]), var8);
                            }
                        }
                    }
                }
            }
        }
    }
```

### Deploying a WAR File
可以部署War文件形式的web应用
```java
    protected void deployWARs(File appBase, String[] files) {
        for(int i = 0; i < files.length; ++i) {
            if (!files[i].equalsIgnoreCase("META-INF") && !files[i].equalsIgnoreCase("WEB-INF") && !this.deployed.contains(files[i])) {
                File dir = new File(appBase, files[i]);
                if (files[i].toLowerCase().endsWith(".war")) {
                    this.deployed.add(files[i]);
                    String contextPath = "/" + files[i];
                    int period = contextPath.lastIndexOf(".");
                    if (period >= 0) {
                        contextPath = contextPath.substring(0, period);
                    }

                    if (contextPath.equals("/ROOT")) {
                        contextPath = "";
                    }

                    if (this.host.findChild(contextPath) == null) {
                        URL url;
                        if (this.isUnpackWARs()) {
                            this.log(sm.getString("hostConfig.expand", files[i]));

                            try {
                                url = new URL("jar:file:" + dir.getCanonicalPath() + "!/");
                                String path = this.expand(url);
                                url = new URL("file:" + path);
                                ((Deployer)this.host).install(contextPath, url);
                            } catch (Throwable var9) {
                                this.log(sm.getString("hostConfig.expand.error", files[i]), var9);
                            }
                        } else {
                            this.log(sm.getString("hostConfig.deployJar", files[i]));

                            try {
                                url = new URL("file", (String)null, dir.getCanonicalPath());
                                url = new URL("jar:" + url.toString() + "!/");
                                ((Deployer)this.host).install(contextPath, url);
                            } catch (Throwable var10) {
                                this.log(sm.getString("hostConfig.deployJar.error", files[i]), var10);
                            }
                        }
                    }
                }
            }
        }
    }
```

### Deploying a Directory
可以讲整个目录拷贝到`%CATALINA_HOME%/webapps`目录下来部署一个应用。
```java
    protected void deployDirectories(File appBase, String[] files) {
        for(int i = 0; i < files.length; ++i) {
            if (!files[i].equalsIgnoreCase("META-INF") && !files[i].equalsIgnoreCase("WEB-INF") && !this.deployed.contains(files[i])) {
                File dir = new File(appBase, files[i]);
                if (dir.isDirectory()) {
                    this.deployed.add(files[i]);
                    File webInf = new File(dir, "/WEB-INF");
                    if (webInf.exists() && webInf.isDirectory() && webInf.canRead()) {
                        String contextPath = "/" + files[i];
                        if (files[i].equals("ROOT")) {
                            contextPath = "";
                        }

                        if (this.host.findChild(contextPath) == null) {
                            this.log(sm.getString("hostConfig.deployDir", files[i]));

                            try {
                                URL url = new URL("file", (String)null, dir.getCanonicalPath());
                                ((Deployer)this.host).install(contextPath, url);
                            } catch (Throwable var8) {
                                this.log(sm.getString("hostConfig.deployDir.error", files[i]), var8);
                            }
                        }
                    }
                }
            }
        }
    }
```

### Live Deploy
如果`liveDeploy`属性为`true`时，调用`#threadStart()`方法，代码如下:
```java
    protected void threadStart() {
        if (this.thread == null) {
            if (this.debug >= 1) {
                this.log(" Starting background thread");
            }

            this.threadDone = false;
            this.threadName = "HostConfig[" + this.host.getName() + "]";
            this.thread = new Thread(this, this.threadName);
            this.thread.setDaemon(true);
            this.thread.start();
        }
    }

        public void run() {
        if (this.debug >= 1) {
            this.log("BACKGROUND THREAD Starting");
        }

        while(!this.threadDone) {
            this.threadSleep();
            this.deployApps();
            this.checkWebXmlLastModified(); // <1>
        }

        if (this.debug >= 1) {
            this.log("BACKGROUND THREAD Stopping");
        }

    }
```
- `<1>`处，会遍历所有部署的上下文，检查web.xml文件以及WEB-INF目录下面内容的时间戳。如果改变，就重启上下文。

## 部署器
### Deployer 接口
一个部署器由`org.apache.catalina.Deployer`接口表示。上述三种部署方法最终调用的`#install()`方法就来自于部署器
```java
public interface Deployer {
    String PRE_INSTALL_EVENT = "pre-install";
    String INSTALL_EVENT = "install";
    String REMOVE_EVENT = "remove";

    String getName();

    void install(String var1, URL var2) throws IOException;

    void install(URL var1, URL var2) throws IOException;

    Context findDeployedApp(String var1);

    String[] findDeployedApps();

    void remove(String var1) throws IOException;

    void start(String var1) throws IOException;

    void stop(String var1) throws IOException;
}
```

- StandardHost使用`org.apache.catalina.core.StandardHostDeployer`来完成部署和安装web应用程序
```java
    // StandardHost.java

    private Deployer deployer = new StandardHostDeployer(this);

    public void install(String contextPath, URL war) throws IOException {
        this.deployer.install(contextPath, war);
    }

    public synchronized void install(URL config, URL war) throws IOException {
        this.deployer.install(config, war);
    }

    public Context findDeployedApp(String contextPath) {
        return this.deployer.findDeployedApp(contextPath);
    }

    public String[] findDeployedApps() {
        return this.deployer.findDeployedApps();
    }

    public void remove(String contextPath) throws IOException {
        this.deployer.remove(contextPath);
    }

    public void start(String contextPath) throws IOException {
        this.deployer.start(contextPath);
    }

    public void stop(String contextPath) throws IOException {
        this.deployer.stop(contextPath);
    }
```
- `#install()`方法用于安装web应用程序，安装后，一个上下文将会被添加到StandardHost

### StandardHostDeployer
#### Install a Descriptor
StandardHostDeployer类有两个install方法。  
其中本节方法用于安装 descriptor(即通过xml文件部署)
```java
    public synchronized void install(URL config, URL war) throws IOException {
        if (config == null) {
            throw new IllegalArgumentException(sm.getString("standardHost.configRequired"));
        } else if (!this.host.isDeployXML()) {
            throw new IllegalArgumentException(sm.getString("standardHost.configNotAllowed"));
        } else {
            String docBase = null;
            String url;
            if (war != null) {
                url = war.toString();
                this.host.log(sm.getString("standardHost.installingWAR", url));
                if (url.startsWith("jar:")) {
                    url = url.substring(4, url.length() - 2);
                }

                if (url.startsWith("file://")) {
                    docBase = url.substring(7);
                } else {
                    if (!url.startsWith("file:")) {
                        throw new IllegalArgumentException(sm.getString("standardHost.warURL", url));
                    }

                    docBase = url.substring(5);
                }
            }

            this.context = null;
            this.overrideDocBase = docBase;
            url = null;

            try {
                InputStream stream = config.openStream();
                Digester digester = this.createDigester();
                digester.setDebug(this.host.getDebug());
                digester.clear();
                digester.push(this);
                digester.parse(stream);
                stream.close();
                url = null;
            } catch (Exception var14) {
                this.host.log(sm.getString("standardHost.installError", docBase), var14);
                throw new IOException(var14.toString());
            } finally {
                if (url != null) {
                    try {
                        url.close();
                    } catch (Throwable var13) {
                    }
                }
            }
        }
    }
```

#### Installing a WAR File and a Directory
安装一个上下文路径的字符串形式表示或者一个URL表示的WAR文件
```java
    public synchronized void install(String contextPath, URL war) throws IOException {
        if (contextPath == null) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathRequired"));
        } else if (!contextPath.equals("") && !contextPath.startsWith("/")) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathFormat", contextPath));
        } else if (this.findDeployedApp(contextPath) != null) {
            throw new IllegalStateException(sm.getString("standardHost.pathUsed", contextPath));
        } else if (war == null) {
            throw new IllegalArgumentException(sm.getString("standardHost.warRequired"));
        } else {
            this.host.log(sm.getString("standardHost.installing", contextPath, war.toString()));
            String url = war.toString();
            String docBase = null;
            if (url.startsWith("jar:")) {
                url = url.substring(4, url.length() - 2);
            }

            if (url.startsWith("file://")) {
                docBase = url.substring(7);
            } else {
                if (!url.startsWith("file:")) {
                    throw new IllegalArgumentException(sm.getString("standardHost.warURL", url));
                }

                docBase = url.substring(5);
            }

            try {
                Class clazz = Class.forName(this.host.getContextClass());
                Context context = (Context)clazz.newInstance();
                context.setPath(contextPath);
                context.setDocBase(docBase);
                if (context instanceof Lifecycle) {
                    clazz = Class.forName(this.host.getConfigClass());
                    LifecycleListener listener = (LifecycleListener)clazz.newInstance();
                    ((Lifecycle)context).addLifecycleListener(listener);
                }

                this.host.fireContainerEvent("pre-install", context);
                this.host.addChild(context);
                this.host.fireContainerEvent("install", context);
            } catch (Exception var8) {
                this.host.log(sm.getString("standardHost.installError", contextPath), var8);
                throw new IOException(var8.toString());
            }
        }
    }
```

#### Starting A Context
```java
    public void start(String contextPath) throws IOException {
        if (contextPath == null) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathRequired"));
        } else if (!contextPath.equals("") && !contextPath.startsWith("/")) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathFormat", contextPath));
        } else {
            Context context = this.findDeployedApp(contextPath);
            if (context == null) {
                throw new IllegalArgumentException(sm.getString("standardHost.pathMissing", contextPath));
            } else {
                this.host.log("standardHost.start " + contextPath);

                try {
                    ((Lifecycle)context).start();
                } catch (LifecycleException var4) {
                    this.host.log("standardHost.start " + contextPath + ": ", var4);
                    throw new IllegalStateException("standardHost.start " + contextPath + ": " + var4);
                }
            }
        }
    }
```

#### Stoping A Context
```java
    public void stop(String contextPath) throws IOException {
        if (contextPath == null) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathRequired"));
        } else if (!contextPath.equals("") && !contextPath.startsWith("/")) {
            throw new IllegalArgumentException(sm.getString("standardHost.pathFormat", contextPath));
        } else {
            Context context = this.findDeployedApp(contextPath);
            if (context == null) {
                throw new IllegalArgumentException(sm.getString("standardHost.pathMissing", contextPath));
            } else {
                this.host.log("standardHost.stop " + contextPath);

                try {
                    ((Lifecycle)context).stop();
                } catch (LifecycleException var4) {
                    this.host.log("standardHost.stop " + contextPath + ": ", var4);
                    throw new IllegalStateException("standardHost.stop " + contextPath + ": " + var4);
                }
            }
        }
    }
```

## 总结
部署器用于部署和安装web应用，由`org.apache.catalina.Deployer`表示。`StandardHost`实现了`Deployer`，它借助`StandardHostDeployer`这个帮助类完成部署和安装。`StandardHostDeployer`提供了部署和安装应用，以及启动和停止上下文容器的代码。
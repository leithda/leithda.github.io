---
title: Tomcat-启动
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 3300275107
date: 2019-12-07 10:04:00
---

{% cq %} 重点关注Tomcat如何使用`Catalina`和`Bootstrap`类进行启动的 {% endcq %}
<!-- More -->

## Catalina类
- `Catalina`是`Tomcat`的启动类.它包含一个解析配置文件的`Digester`类，它还封装了一个Server对象来提供服务。正常情况下，可以调用它的`#process()`方法来启动`Tomcat`(可以传递适合的参数,如`-help`、`-config`等)。它的`#process()`方法签名如下:
```java
    public void process(String[] args) {
        this.setCatalinaHome(); // <1>
        this.setCatalinaBase(); // <2>

        try {
            if (this.arguments(args)) { // <3>
                this.execute(); // <4>
            }
        } catch (Exception var3) {
            var3.printStackTrace(System.out);
        }

    }
```
    - `<1>`及`<2>`处，设置两个系统属性`catalina.home`和`catalina.base`默认为`user.dir`。
    - `<3>`处，调用`#arguments()`方法处理参数列表,代码如下:
    ```java
        protected boolean arguments(String[] args) {
        boolean isConfig = false;
        if (args.length < 1) {
            this.usage();
            return false;
        } else {
            for(int i = 0; i < args.length; ++i) {
                if (isConfig) {
                    this.configFile = args[i];
                    isConfig = false;
                } else if (args[i].equals("-config")) {
                    isConfig = true;
                } else if (args[i].equals("-debug")) {
                    this.debug = true;
                } else if (args[i].equals("-nonaming")) {
                    this.useNaming = false;
                } else {
                    if (args[i].equals("-help")) {
                        this.usage();
                        return false;
                    }

                    if (args[i].equals("start")) {
                        this.starting = true;
                    } else {
                        if (!args[i].equals("stop")) {
                            this.usage();
                            return false;
                        }

                        this.stopping = true;
                    }
                }
            }

            return true;
        }
    }
    ```
    - `<4>`处，当参数处理返回`true`时，调用`#execute()`方法，启动或者停止Tomcat。

### start 方法
- `#start()`方法主要分为以下几个步骤:

1. 创建`Digester`类并解析配置文件.`#createStartDigester`方法下面再分析，对`Digester`存有疑问的请移步[Tomcat-Digester](./2030302893.html)进行查看
2. 设置系统属性`catalina.useNaming`等
3. 如果使用安全管理器，初始化其访问权限
4. 替换系统的输出为`Tomcat`的日志组件`SystemLogHandler`
5. 启动服务器，设置关闭钩子防止不正常终止服务，调用`#await`方法直到收到`SHUTDOWN`命令
6. 停止服务器，移除关闭钩子防止应用执行两次清理程序。调用`#stop`方法停止应用程序。

```java
    protected void start() {
        /** step1 : 创建StartDigester并解析文件 */
        Digester digester = this.createStartDigester();
        File file = this.configFile();

        try {
            InputSource is = new InputSource("file://" + file.getAbsolutePath());
            FileInputStream fis = new FileInputStream(file);
            is.setByteStream(fis);
            digester.push(this);
            digester.parse(is);
            fis.close();
        } catch (Exception var8) {
            System.out.println("Catalina.start: " + var8);
            var8.printStackTrace(System.out);
            System.exit(1);
        }

        /** step2 : 设置系统属性 */
        String access;
        String definition;
        if (!this.useNaming) {
            System.setProperty("catalina.useNaming", "false");
        } else {
            System.setProperty("catalina.useNaming", "true");
            access = "org.apache.naming";
            definition = System.getProperty("java.naming.factory.url.pkgs");
            if (definition != null) {
                access = access + ":" + definition;
            }

            System.setProperty("java.naming.factory.url.pkgs", access);
            access = System.getProperty("java.naming.factory.initial");
            if (access == null) {
                System.setProperty("java.naming.factory.initial", "org.apache.naming.java.javaURLContextFactory");
            }
        }

        /** step3 : 安全管理器处理。如果使用，设置访问权限 */
        if (System.getSecurityManager() != null) {
            access = Security.getProperty("package.access");
            if (access != null && access.length() > 0) {
                access = access + ",";
            } else {
                access = "sun.,";
            }

            Security.setProperty("package.access", access + "org.apache.catalina.,org.apache.jasper.");
            definition = Security.getProperty("package.definition");
            if (definition != null && definition.length() > 0) {
                definition = definition + ",";
            } else {
                definition = "sun.,";
            }

            Security.setProperty("package.definition", definition + "java.,org.apache.catalina.,org.apache.jasper.");
        }

        /** step4 : 替换系统System.out及System.err */
        SystemLogHandler log = new SystemLogHandler(System.out);
        System.setOut(log);
        System.setErr(log);
        Thread shutdownHook = new Catalina.CatalinaShutdownHook();

        /** step5 : 启动服务器 */
        if (this.server instanceof Lifecycle) {
            try {
                this.server.initialize();
                ((Lifecycle)this.server).start();

                try {
                    // 设置关闭钩子
                    Runtime.getRuntime().addShutdownHook(shutdownHook);
                } catch (Throwable var7) {
                }
                // 服务器等待，直到收到SHUTDOWN命令
                this.server.await();
            } catch (LifecycleException var10) {
                System.out.println("Catalina.start: " + var10);
                var10.printStackTrace(System.out);
                if (var10.getThrowable() != null) {
                    System.out.println("----- Root Cause -----");
                    var10.getThrowable().printStackTrace(System.out);
                }
            }
        }

        /** step6 : 关闭服务器 */
        if (this.server instanceof Lifecycle) {
            try {
                try {
                    // 移除关闭钩子，避免执行两次清理程序
                    Runtime.getRuntime().removeShutdownHook(shutdownHook);
                } catch (Throwable var6) {
                }

                ((Lifecycle)this.server).stop();
            } catch (LifecycleException var9) {
                System.out.println("Catalina.stop: " + var9);
                var9.printStackTrace(System.out);
                if (var9.getThrowable() != null) {
                    System.out.println("----- Root Cause -----");
                    var9.getThrowable().printStackTrace(System.out);
                }
            }
        }

    }
```

### stop 方法
- 相较于`#start()`方法，`#stop()`方法处理略微简单。**PS**: `stop`方法创建Digester方法通过的是`#createStopDigester()`方法，切勿与`start`方法中的混淆。

```java
    protected void stop() {
        Digester digester = this.createStopDigester();
        File file = this.configFile();

        try {
            InputSource is = new InputSource("file://" + file.getAbsolutePath());
            FileInputStream fis = new FileInputStream(file);
            is.setByteStream(fis);
            digester.push(this);
            digester.parse(is);
            fis.close();
        } catch (Exception var7) {
            System.out.println("Catalina.stop: " + var7);
            var7.printStackTrace(System.out);
            System.exit(1);
        }

        try {
            Socket socket = new Socket("127.0.0.1", this.server.getPort());
            OutputStream stream = socket.getOutputStream();
            String shutdown = this.server.getShutdown();

            for(int i = 0; i < shutdown.length(); ++i) {
                stream.write(shutdown.charAt(i));
            }

            stream.flush();
            stream.close();
            socket.close();
        } catch (IOException var8) {
            System.out.println("Catalina.stop: " + var8);
            var8.printStackTrace(System.out);
            System.exit(1);
        }

    }
```

### StartDigester
```java
    protected Digester createStartDigester() {
        Digester digester = new Digester();
        if (this.debug) {
            digester.setDebug(999);
        }

        digester.setValidating(false);
        digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer", "className");
        digester.addSetProperties("Server");
        digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");
        digester.addObjectCreate("Server/GlobalNamingResources", "org.apache.catalina.deploy.NamingResources");
        digester.addSetProperties("Server/GlobalNamingResources");
        digester.addSetNext("Server/GlobalNamingResources", "setGlobalNamingResources", "org.apache.catalina.deploy.NamingResources");
        digester.addObjectCreate("Server/Listener", (String)null, "className");
        digester.addSetProperties("Server/Listener");
        digester.addSetNext("Server/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");
        digester.addObjectCreate("Server/Service", "org.apache.catalina.core.StandardService", "className");
        digester.addSetProperties("Server/Service");
        digester.addSetNext("Server/Service", "addService", "org.apache.catalina.Service");
        digester.addObjectCreate("Server/Service/Listener", (String)null, "className");
        digester.addSetProperties("Server/Service/Listener");
        digester.addSetNext("Server/Service/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");
        digester.addObjectCreate("Server/Service/Connector", "org.apache.catalina.connector.http.HttpConnector", "className");
        digester.addSetProperties("Server/Service/Connector");
        digester.addSetNext("Server/Service/Connector", "addConnector", "org.apache.catalina.Connector");
        digester.addObjectCreate("Server/Service/Connector/Factory", "org.apache.catalina.net.DefaultServerSocketFactory", "className");
        digester.addSetProperties("Server/Service/Connector/Factory");
        digester.addSetNext("Server/Service/Connector/Factory", "setFactory", "org.apache.catalina.net.ServerSocketFactory");
        digester.addObjectCreate("Server/Service/Connector/Listener", (String)null, "className");
        digester.addSetProperties("Server/Service/Connector/Listener");
        digester.addSetNext("Server/Service/Connector/Listener", "addLifecycleListener", "org.apache.catalina.LifecycleListener");
        digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
        digester.addRuleSet(new EngineRuleSet("Server/Service/"));
        digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
        digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Default"));
        digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/DefaultContext/"));
        digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/Default"));
        digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/DefaultContext/"));
        digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
        digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));
        digester.addRule("Server/Service/Engine", new SetParentClassLoaderRule(digester, this.parentClassLoader));
        return digester;
    }
```
- 这里简单分析下前三条规则，即:
```java
    // 遇到server元素，创建 StandardServer 对象，如果设置了 className 属性，则创建 className指定的类
    digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer", "className");
    // 将适用于Server的属性的值填充到生成的Server对象中
    digester.addSetProperties("Server");

    /*
    将Server对象压入栈中并与栈中对象(Catalina)关联.通过Catalina.setServer方法。其中Catalina对象在start()方法中首先加入栈中.
    
    protected void start() {
        Digester digester = this.createStartDigester();
        File file = this.configFile();

        try {
            // ... 忽略部分代码
            digester.push(this);
            digester.parse(is);
            fis.close();
        } catch (Exception var8) {
            // ... 忽略部分代码
        }
        // ... 忽略部分代码
    }
     */
    digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");
```
- 这里如果存在困难，请移步[Tomcat-Digester](./2030302893.html)进行查看Digester的相关分析.

### StopDigester
```java
    protected Digester createStopDigester() {
        Digester digester = new Digester();
        if (this.debug) {
            digester.setDebug(999);
        }

        digester.addObjectCreate("Server", "org.apache.catalina.core.StandardServer", "className");
        digester.addSetProperties("Server");
        digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");
        return digester;
    }
```

## Bootstrap
- 当执行`startup.bat`或是`startup.sh`时，调用的就是`Bootstrap`的`main`方法

```java


package org.apache.catalina.startup;

import java.io.File;
import java.lang.reflect.Method;

public final class Bootstrap {
    /**
     * 调试标志
     */
    private static int debug = 0;

    public Bootstrap() {
    }

    public static void main(String[] args) {
        /** step1 : 调试标志调整 */
        for(int i = 0; i < args.length; ++i) {
            if ("-debug".equals(args[i])) {
                debug = 1;
            }
        }

        /** step2 : 设置系统属性 */
        if (System.getProperty("catalina.base") == null) {
            System.setProperty("catalina.base", getCatalinaHome());
        }

        /** step3 : 创建加载器 */
        ClassLoader commonLoader = null;
        ClassLoader catalinaLoader = null;
        ClassLoader sharedLoader = null;

        try {
            File[] unpacked = new File[1];
            File[] packed = new File[1];
            ClassLoaderFactory.setDebug(debug);
            unpacked[0] = new File(getCatalinaHome(), "common" + File.separator + "classes");
            packed[0] = new File(getCatalinaHome(), "common" + File.separator + "lib");
            commonLoader = ClassLoaderFactory.createClassLoader(unpacked, packed, (ClassLoader)null);
            unpacked[0] = new File(getCatalinaHome(), "server" + File.separator + "classes");
            packed[0] = new File(getCatalinaHome(), "server" + File.separator + "lib");
            catalinaLoader = ClassLoaderFactory.createClassLoader(unpacked, packed, commonLoader);
            unpacked[0] = new File(getCatalinaHome(), "classes");
            packed[0] = new File(getCatalinaHome(), "lib");
            sharedLoader = ClassLoaderFactory.createClassLoader(unpacked, packed, commonLoader);
        } catch (Throwable var12) {
            log("Class loader creation threw exception", var12);
            System.exit(1);
        }

        Thread.currentThread().setContextClassLoader(catalinaLoader);

        try {
            /** step4 : 加载启动相关类 */
            if (System.getSecurityManager() != null) {
                String basePackage = "org.apache.catalina.";
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedGetRequestDispatcher");
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedGetResource");
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedGetResourcePaths");
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedLogMessage");
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedLogException");
                catalinaLoader.loadClass(basePackage + "core.ApplicationContext$PrivilegedLogThrowable");
                catalinaLoader.loadClass(basePackage + "core.ApplicationDispatcher$PrivilegedForward");
                catalinaLoader.loadClass(basePackage + "core.ApplicationDispatcher$PrivilegedInclude");
                catalinaLoader.loadClass(basePackage + "connector.HttpRequestBase$PrivilegedGetSession");
                catalinaLoader.loadClass(basePackage + "loader.WebappClassLoader$PrivilegedFindResource");
                catalinaLoader.loadClass(basePackage + "session.StandardSession");
                catalinaLoader.loadClass(basePackage + "util.CookieTools");
                catalinaLoader.loadClass(basePackage + "util.URL");
                catalinaLoader.loadClass(basePackage + "util.Enumerator");
                catalinaLoader.loadClass("javax.servlet.http.Cookie");
            }

            if (debug >= 1) {
                log("Loading startup class");
            }

            /** step 5 : 加载 Catalina 类并调用其 process() 方法 */
            Class startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
            Object startupInstance = startupClass.newInstance();
            if (debug >= 1) {
                log("Setting startup class properties");
            }

            String methodName = "setParentClassLoader";
            Class[] paramTypes = new Class[]{Class.forName("java.lang.ClassLoader")};
            Object[] paramValues = new Object[]{sharedLoader};
            Method method = startupInstance.getClass().getMethod(methodName, paramTypes);
            method.invoke(startupInstance, paramValues);
            if (debug >= 1) {
                log("Calling startup class process() method");
            }

            methodName = "process";
            paramTypes = new Class[]{args.getClass()};
            paramValues = new Object[]{args};
            method = startupInstance.getClass().getMethod(methodName, paramTypes);
            method.invoke(startupInstance, paramValues);
        } catch (Exception var11) {
            System.out.println("Exception during startup processing");
            var11.printStackTrace(System.out);
            System.exit(2);
        }

    }

    private static String getCatalinaHome() {
        return System.getProperty("catalina.home", System.getProperty("user.dir"));
    }

    private static void log(String message) {
        System.out.print("Bootstrap: ");
        System.out.println(message);
    }

    private static void log(String message, Throwable exception) {
        log(message);
        exception.printStackTrace(System.out);
    }
}
```

## 批处理文件(启动脚本)
> 这部分内容暂时留空，后续单独学习相应的语法后再来补充。

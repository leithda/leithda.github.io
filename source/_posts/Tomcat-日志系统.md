---
title: Tomcat-日志系统
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 3784073105
date: 2019-10-14 00:00:00
---

日志系统是一个记录信息的组件。在 Catalina 中，日志系统是一个相对简单的跟容器相关联的组件。Tomcat 在 `org.apache.catalina.logger` 包中提供了多个不同的日志系统

<!-- More -->
## 解析
### 日志接口
```java
public interface Logger {
    int FATAL = -2147483648;
    int ERROR = 1;
    int WARNING = 2;
    int INFORMATION = 3;
    int DEBUG = 4;

    Container getContainer();

    void setContainer(Container var1);

    String getInfo();

    int getVerbosity();

    void setVerbosity(int var1);

    void addPropertyChangeListener(PropertyChangeListener var1);

    void log(String var1);

    void log(Exception var1, String var2);

    void log(String var1, Throwable var2);

    void log(String var1, int var2);

    void log(String var1, Throwable var2, int var3);

    void removePropertyChangeListener(PropertyChangeListener var1);
}

```

### Tomcat日志系统
- Tomcat 提供了三种日志系统`FileLogger`, `SystemErrLogger`, 和 `SystemOutLogger`
他们的关系如下图:
![](./Logger.png)

### `LoggerBase `类
```java
public abstract class LoggerBase implements Logger {
    protected Container container = null;
    protected int debug = 0;
    protected static final String info = "org.apache.catalina.logger.LoggerBase/1.0";
    protected PropertyChangeSupport support = new PropertyChangeSupport(this);
    /**
     * 默认等级 ERROR
     */
    protected int verbosity = 1;

    public LoggerBase() {
    }

    public Container getContainer() {
        return this.container;
    }

    public void setContainer(Container container) {
        Container oldContainer = this.container;
        this.container = container;
        this.support.firePropertyChange("container", oldContainer, this.container);
    }

    public int getDebug() {
        return this.debug;
    }

    public void setDebug(int debug) {
        this.debug = debug;
    }

    public String getInfo() {
        return "org.apache.catalina.logger.LoggerBase/1.0";
    }

    public int getVerbosity() {
        return this.verbosity;
    }

    public void setVerbosity(int verbosity) {
        this.verbosity = verbosity;
    }

    public void setVerbosityLevel(String verbosity) {
        if ("FATAL".equalsIgnoreCase(verbosity)) {
            this.verbosity = -2147483648;
        } else if ("ERROR".equalsIgnoreCase(verbosity)) {
            this.verbosity = 1;
        } else if ("WARNING".equalsIgnoreCase(verbosity)) {
            this.verbosity = 2;
        } else if ("INFORMATION".equalsIgnoreCase(verbosity)) {
            this.verbosity = 3;
        } else if ("DEBUG".equalsIgnoreCase(verbosity)) {
            this.verbosity = 4;
        }

    }

    public void addPropertyChangeListener(PropertyChangeListener listener) {
        this.support.addPropertyChangeListener(listener);
    }

    public abstract void log(String var1);

    public void log(Exception exception, String msg) {
        this.log((String)msg, (Throwable)exception);
    }

    public void log(String msg, Throwable throwable) {
        CharArrayWriter buf = new CharArrayWriter();
        PrintWriter writer = new PrintWriter(buf);
        writer.println(msg);
        throwable.printStackTrace(writer);
        Throwable rootCause = null;
        if (throwable instanceof LifecycleException) {
            rootCause = ((LifecycleException)throwable).getThrowable();
        } else if (throwable instanceof ServletException) {
            rootCause = ((ServletException)throwable).getRootCause();
        }

        if (rootCause != null) {
            writer.println("----- Root Cause -----");
            rootCause.printStackTrace(writer);
        }

        this.log(buf.toString());
    }

    public void log(String message, int verbosity) {
        if (this.verbosity >= verbosity) {
            this.log(message);
        }

    }

    public void log(String message, Throwable throwable, int verbosity) {
        if (this.verbosity >= verbosity) {
            this.log(message, throwable);
        }

    }

    public void removePropertyChangeListener(PropertyChangeListener listener) {
        this.support.removePropertyChangeListener(listener);
    }
}
```
- `LoggerBase`抽象类，实现了`#log`以外的所有方法

### `SystermOutLogger` 类

```java
public class SystemOutLogger extends LoggerBase {
    protected static final String info = "org.apache.catalina.logger.SystemOutLogger/1.0";

    public SystemOutLogger() {
    }

    public void log(String msg) {
        System.out.println(msg);
    }
}
```

- 继承了`LoggerBase`类，实现了 `#log`方法，输出日志到控制台

### `SystemErrLogger`类

```java
public class SystemErrLogger extends LoggerBase {
    protected static final String info = "org.apache.catalina.logger.SystemErrLogger/1.0";

    public SystemErrLogger() {
    }

    public void log(String msg) {
        System.err.println(msg);
    }
}
```



### `FileLogger`类

```java
public class FileLogger extends LoggerBase implements Lifecycle {
    private String date = "";
    // 文件夹
    private String directory = "logs";
    // 说明信息
    protected static final String info = "org.apache.catalina.logger.FileLogger/1.0";
    protected LifecycleSupport lifecycle = new LifecycleSupport(this);
    // 前缀
    private String prefix = "catalina.";
    // 多语言
    private StringManager sm = StringManager.getManager("org.apache.catalina.logger");
    private boolean started = false;
    // 后缀
    private String suffix = ".log";
    // 输出时间戳标志
    private boolean timestamp = false;
    // 文件输出对象
    private PrintWriter writer = null;

    public FileLogger() {
    }

    // prefix,suffix,directory，timestamp 的 getter/setter 方法

    /**
     * 记录日志
     * @param msg   日志消息（<1>日期部分有修改）
     */
    public void log(String msg) {
        LocalDateTime localDateTIme = LocalDateTime.now();
        LocalDate localDate = localDateTIme.toLocalDate();
        LocalTime localTime = localDateTIme.toLocalTime();
        String sDate = localDate.format(DateTimeFormatter.BASIC_ISO_DATE);
        String sTime = localTime.format(DateTimeFormatter.ISO_LOCAL_TIME);
        if (!this.date.equals(sDate)) {
            synchronized(this) {
                if (!this.date.equals(sDate)) {
                    this.close();
                    this.date = sDate;
                    this.open();
                }
            }
        }

        if (this.writer != null) {
            if (this.timestamp) {
                this.writer.println(sTime + " " + msg);
            } else {
                this.writer.println(msg);
            }
        }
    }

    /**
     * 关闭文件
     */
    private void close() {
        if (this.writer != null) {
            this.writer.flush();
            this.writer.close();
            this.writer = null;
            this.date = "";
        }
    }

    /**
     * 打开文件
     */
    private void open() {
        File dir = new File(this.directory);
        if (!dir.isAbsolute()) {
            dir = new File(System.getProperty("catalina.base"), this.directory);
        }

        dir.mkdirs();

        try {
            String pathname = dir.getAbsolutePath() + File.separator + this.prefix + this.date + this.suffix;
            this.writer = new PrintWriter(new FileWriter(pathname, true), true);
        } catch (IOException var3) {
            this.writer = null;
        }

    }

    // =========================Lifecycle接口的实现=====================================
    public void addLifecycleListener(LifecycleListener listener) {
        this.lifecycle.addLifecycleListener(listener);
    }

    public LifecycleListener[] findLifecycleListeners() {
        return this.lifecycle.findLifecycleListeners();
    }

    public void removeLifecycleListener(LifecycleListener listener) {
        this.lifecycle.removeLifecycleListener(listener);
    }

    public void start() throws LifecycleException {
        if (this.started) {
            throw new LifecycleException(this.sm.getString("fileLogger.alreadyStarted"));
        } else {
            this.lifecycle.fireLifecycleEvent("start", (Object)null);
            this.started = true;
        }
    }

    public void stop() throws LifecycleException {
        if (!this.started) {
            throw new LifecycleException(this.sm.getString("fileLogger.notStarted"));
        } else {
            this.lifecycle.fireLifecycleEvent("stop", (Object)null);
            this.started = false;
            this.close();
        }
    }
}
```

- 其中`#log`方法，判断日期部分，由于`java8`更改为通过`java.time.*`  

## 代码

### `SimpleContext`类

```java
// 只列出关于logger的属性和方法，其他忽略
public class SimpleContext implements Context, Pipeline, Lifecycle {

    public SimpleContext() {
        pipeline.setBasic(new SimpleContextValve());
    }

    //子容器
    protected HashMap children = new HashMap();
    //类加载器
    private Loader loader = null;
    //日志
    private Logger logger = null;
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

    // ... 部分代码省略
    public Logger getLogger() {
        return logger;
    }

    public void setLogger(Logger logger) {
        this.logger = logger;
    }
    // ... 部分代码省略

    //methods of the Container interface
    // ... 部分代码省略

    // method implementations of Pipeline
    // ... 部分代码省略

    // implementation of the Lifecycle interface's methods
    // ... 部分代码省略

    public synchronized void start() throws LifecycleException {
        log("starting Context");    // <1> 日志
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
        }
        catch (Exception e) {
            e.printStackTrace();
        }

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
        log("Context started"); // <2> 日志
    }

    public void stop() throws LifecycleException {
        log("stopping Context");    // <3> 日志
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
        }
        catch (Exception e) {
            e.printStackTrace();
        }
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_STOP_EVENT, null);
        log("Context stopped"); // <4> 日志
    }

    /**
     * 记录日志
     * @param msg   日志消息
     */
    private void log(String message) {
        Logger logger = this.getLogger();
        if (logger!=null)
            logger.log(message);
    }
}
```

- 其中，`<1>``<2>``<3>``<4>`处使用`#log`方法记录日志。 `#log`方法在下面，代码简单，不在赘述

### `Bootstrap`类

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

        // <1> ------ add logger --------
        // 添加日志记录
        // 设置系统变量
        System.setProperty("catalina.base", System.getProperty("user.dir"));
        FileLogger logger = new FileLogger();
        logger.setPrefix("FileLog_");
        logger.setSuffix(".txt");
        logger.setTimestamp(true);
        logger.setDirectory("webroot");
        context.setLogger(logger);

        //---------------------------
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

- `<1>`处，实例化日志对象并设置到容器中。

## 测试

- 访问地址` http://localhost:8080/Modern `

### 网页内容

```html
Headers
host : localhost:8080
connection : keep-alive
cache-control : max-age=0
upgrade-insecure-requests : 1
user-agent : Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36
sec-fetch-mode : navigate
sec-fetch-user : ?1
accept : text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
sec-fetch-site : none
accept-encoding : gzip, deflate, br
accept-language : zh-CN,zh;q=0.9
Method
GET
Parameters
Query String
null
Request URI
/Modern
```

### 控制台输出如下

```shell
SimpleContextLifecycleListener's event before_start
Starting SimpleLoader
Starting Wrapper Primitive
Starting Wrapper Modern
SimpleContextLifecycleListener's event start
Starting context.
SimpleContextLifecycleListener's event after_start
ModernServlet -- init
```

### 日志文件（`FileLog_20191014.txt`）如下

```verilog
21:09:33.318 HttpConnector Opening server socket on all host IP addresses
21:09:33.333 HttpConnector[8080] Starting background thread
21:09:33.358 HttpProcessor[8080][0] Starting background thread
21:09:33.359 HttpProcessor[8080][1] Starting background thread
21:09:33.359 HttpProcessor[8080][2] Starting background thread
21:09:33.359 HttpProcessor[8080][3] Starting background thread
21:09:33.36 HttpProcessor[8080][4] Starting background thread
21:09:33.36 starting Context
21:09:33.36 Context started
21:10:13.483 HttpProcessor[8080][4] process.invoke
java.lang.NoClassDefFoundError: org/apache/tomcat/util/log/SystemLogHandler
    at org.apache.catalina.connector.RequestBase.recycle(RequestBase.java:562)
    at org.apache.catalina.connector.HttpRequestBase.recycle(HttpRequestBase.java:417)
    at org.apache.catalina.connector.http.HttpRequestImpl.recycle(HttpRequestImpl.java:195)
    at org.apache.catalina.connector.http.HttpProcessor.process(HttpProcessor.java:1101)
    at org.apache.catalina.connector.http.HttpProcessor.run(HttpProcessor.java:1151)
    at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.ClassNotFoundException: org.apache.tomcat.util.log.SystemLogHandler
    at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    ... 6 more

21:10:18.444 HttpProcessor[8080][3] process.invoke
java.lang.NoClassDefFoundError: org/apache/tomcat/util/log/SystemLogHandler
    at org.apache.catalina.connector.RequestBase.recycle(RequestBase.java:562)
    at org.apache.catalina.connector.HttpRequestBase.recycle(HttpRequestBase.java:417)
    at org.apache.catalina.connector.http.HttpRequestImpl.recycle(HttpRequestImpl.java:195)
    at org.apache.catalina.connector.http.HttpProcessor.process(HttpProcessor.java:1101)
    at org.apache.catalina.connector.http.HttpProcessor.run(HttpProcessor.java:1151)
    at java.lang.Thread.run(Thread.java:748)
```
- 上述报错由于缺少依赖，导致SystemLogHandler类找不到，解决方法添加依赖如下:
    ```xml
            <!--  解决写入文件日志报错-->
            <dependency>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-core</artifactId>
                <version>9.0.24</version>
            </dependency>
    ```
- 报错代码`#RequestBase.recycle()`方法,<1>位置处，代码如下
    ```java
    //   RequestBase.java
    import org.apache.tomcat.util.log.SystemLogHandler;

    public abstract class RequestBase implements ServletRequest, Request {

    // ... 部分代码忽略
        public void recycle() {
            String log = SystemLogHandler.stopCapture();    // <1> 
            if (log != null && log.length() > 0) {
                this.context.getServletContext().log(log);
            }

            this.attributes.clear();
            this.authorization = null;
            this.characterEncoding = null;
            this.contentLength = -1;
            this.contentType = null;
            this.context = null;
            this.input = null;
            this.locales.clear();
            this.notes.clear();
            this.protocol = null;
            this.reader = null;
            this.remoteAddr = null;
            this.remoteHost = null;
            this.response = null;
            this.scheme = null;
            this.secure = false;
            this.serverName = null;
            this.serverPort = -1;
            this.socket = null;
            this.stream = null;
            this.wrapper = null;
        }

        // ... 部分代码忽略
    }
    ```
### 更新依赖后的日志文件
```shell
08:34:51.302 HttpConnector Opening server socket on all host IP addresses
08:34:51.329 HttpConnector[8080] Starting background thread
08:34:51.367 HttpProcessor[8080][0] Starting background thread
08:34:51.368 HttpProcessor[8080][1] Starting background thread
08:34:51.376 HttpProcessor[8080][2] Starting background thread
08:34:51.377 HttpProcessor[8080][3] Starting background thread
08:34:51.378 HttpProcessor[8080][4] Starting background thread
08:34:51.378 starting Context
08:34:51.378 Context started
```

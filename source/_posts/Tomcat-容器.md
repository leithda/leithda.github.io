---
title: Tomcat-容器
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
date: 2019-10-9
abbrlink: 3459939477
---

容器是一个处理用户 servlet 请求并返回对象给 web 用户的模块。`org.apache.catalina.Container` 接口定义了容器的形式,有四种容器:`Engine`(引擎), `Host`(主机), `Context`(上下文), 和 `Wrapper`(包装器)

<!-- More -->

## 解析
### 容器
- 对于`Container`首先要知道它一共是四种容器:
1. `Engine`：表示整个`Container`的`servlet`容器
2. `Host`:表示一个拥有数个上下文的虚拟主机
3. `Context`：表示一个`web`应用，一个`Context`包含一个或多个`Wrapper`
4. `Wrapper`：表示一个独立的`servlet`
    - 他们的标准实现分别是`StandardEngine`、`StandardHost`、`StandardContext`和`StandardWrapper`
    - 结构图如下:
    {% asset_img container.png Container结构图 %}

### 流水线任务`PipeLining Task`解析
   主要讨论以下四个接口`Pipeline,、Valve,、ValveContext、Contained`
- `invoke`方法通过责任链模式进行处理，可以通过`server.xml`来动态添加和删除处理者
- `Continer`的`invoke`方法在`org.apache.catalina.core.ContainerBase`中实现，代码如下:
```java
// ContainerBase.java
public void invoke(Request request, Response response) throws IOException, ServletException {
    this.pipeline.invoke(request, response);
}
```
- 当容器中的`invoke`方法被调用时，容器会进行责任链模式的处理，前者处理完成，后者进行处理，基础的处理放在最后面。伪代码如下：
```java
// invoke each valve added to the pipeline
for (int n=0; n<valves.length; n++) {
valve[n].invoke( ... );
}
// then, invoke the basic valve
basicValve.invoke( ... );
```

#### 接口解析
1. `Pineline`接口
```java
public interface Pipeline {
    Valve getBasic();

    void setBasic(Valve var1);

    void addValve(Valve var1);

    Valve[] getValves();

    void invoke(Request var1, Response var2) throws IOException, ServletException;

    void removeValve(Valve var1);
}
```
- `#setBasic`和`#getBasic`方法用于设置基本处理者，它会在责任链的最后被调用，负责处理`request`和回复`response`
2. `Value`接口
```java
public interface Valve {
    String getInfo();

    void invoke(Request var1, Response var2, ValveContext var3) throws IOException, ServletException;
}
```
- `#invoke`方法用于处理请求，`#getInfo`返回处理者信息

3. `ValveContext`接口
```java
public interface ValveContext {
    String getInfo();

    // 用于唤醒下一处理者
    void invokeNext(Request var1, Response var2) throws IOException, ServletException;
}
```
- 其中最重要的方法是`#invokeNext`方法，它用于当前处理者完成任务前唤醒下一处理者，责任链模式实现

4. `Contained`接口
```java
public interface Contained {
    Container getContainer();

    void setContainer(Container var1);
}
```
- 一个处理者可以选择性的实现此接口，该接口定义了其实现类和容器关联类

5. `Wrapper`接口
```java
public interface Wrapper extends Container {
    // ...

    // 定位该包装器表示的servlet实例
    Servlet allocate() throws ServletException;

    // ...

    // 负责加载和初始化 servlet
    void load() throws ServletException;

    // ...
}
```

- 主要方法如代码所示，其余代码后续分析。

#### 流程分析

- 首先入口是`Container`的`invoke`方法，实现类在`ContainerBase`中，实现如下:
```java
// ContainerBase.java
    public void invoke(Request request, Response response) throws IOException, ServletException {
        this.pipeline.invoke(request, response);
    }
```
- 调用`Pineline`的`invoke`方法，实现如下:
```java
// StandardPipeline.java

    public void invoke(Request request, Response response) throws IOException, ServletException {
        (new StandardPipeline.StandardPipelineValveContext()).invokeNext(request, response);
    }

```

- 最终调用`#invokeNext方法`，实现如下:
```java
// StandardPipeline.java

    protected class StandardPipelineValveContext implements ValveContext {
        protected int stage = 0;

        protected StandardPipelineValveContext() {
        }

        public String getInfo() {
            return StandardPipeline.this.info;
        }

        // 责任链模式实现
        public void invokeNext(Request request, Response response) throws IOException, ServletException {
            int subscript = this.stage++;
            if (subscript < StandardPipeline.this.valves.length) {
                StandardPipeline.this.valves[subscript].invoke(request, response, this);
            } else {
                if (subscript != StandardPipeline.this.valves.length || StandardPipeline.this.basic == null) {
                    throw new ServletException(StandardPipeline.sm.getString("standardPipeline.noValve"));
                }

                StandardPipeline.this.basic.invoke(request, response, this);
            }
        }
    }
```

## 代码

### `SimpleLoader`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 加载器
 */
public class SimpleLoader implements Loader {

    public static final String WEB_ROOT = System.getProperty("user.dir") + File.separator + "webroot";

    ClassLoader classLoader = null;

    Container container = null;


    public SimpleLoader() {
        try {
            URL[] urls = new URL[1];
            URLStreamHandler streamHandler = null;
            File classPath = new File(WEB_ROOT);
            String repository = (new URL("file", null, classPath.getCanonicalPath() + File.separator)).toString();
            urls[0] = new URL(null, repository, streamHandler);
            classLoader = new URLClassLoader(urls);
        } catch (IOException e) {
            System.out.println(e.toString());
        }
    }

//... 省略部分代码
    
}
```

- `Loader`实现类，用于定位`servlet`，主要成员`WEB_ROOT`、`classLoader`、`container`

### `SimplePipeline`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 责任链类
 */
public class SimplePipeline implements Pipeline {

    public SimplePipeline(Container container) {
        this.container = container;
    }

    // The basic Valve (if any) associated with this Pipeline.
    protected Valve basic = null;
    // The Container with which this Pipeline is associated.
    protected Container container = null;
    // the array of Valves

    protected Valve valves[] = new Valve[0];

    public Valve getBasic() {
        return basic;
    }

    public void setBasic(Valve valve) {
        this.basic = valve;
        ((Contained) valve).setContainer(container);
    }

    public void addValve(Valve valve) {
        if (valve instanceof Contained)
            ((Contained) valve).setContainer(this.container);

        synchronized (valves) {
            Valve results[] = new Valve[valves.length + 1];
            System.arraycopy(valves, 0, results, 0, valves.length);
            results[valves.length] = valve;
            valves = results;
        }
    }

    public Valve[] getValves() {
        return valves;
    }

    /**
     * 启动流水作业
     * @param request 请求
     * @param response 响应
     */
    public void invoke(Request request, Response response) throws IOException, ServletException {
        // Invoke the first Valve in this pipeline for this request
        (new SimplePipelineValveContext()).invokeNext(request, response);
    }

    public void removeValve(Valve valve) {

    }

    // this class is copied from org.apache.catalina.core.StandardPipeline class's
    // StandardPipelineValveContext inner class.
    protected class SimplePipelineValveContext implements ValveContext {

        protected int stage = 0;

        public String getInfo() {
            return null;
        }

        public void invokeNext(Request request, Response response)
                throws IOException, ServletException {
            int subscript = stage;
            stage = stage + 1;
            // Invoke the requested Valve for the current request thread
            if (subscript < valves.length) {
                valves[subscript].invoke(request, response, this);
            } else if ((subscript == valves.length) && (basic != null)) {
                basic.invoke(request, response, this);
            } else {
                throw new ServletException("No valve");
            }
        }
    } // end of inner class
}
```

### `SimpleWrapper`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 包装类
 */
public class SimpleWrapper implements Wrapper, Pipeline {
    // the servlet instance
    private Servlet instance = null;
    /**
     * servlet 名称
     */
    private String servletClass;
    private Loader loader;
    private String name;
    private SimplePipeline pipeline = new SimplePipeline(this);

    /**
     * 该容器可以作为其他容器(比如Context)的子容器
     */
    protected Container parent = null;

    public SimpleWrapper() {
        pipeline.setBasic(new SimpleWrapperValve());
    }

    public synchronized void addValve(Valve valve) {
        pipeline.addValve(valve);
    }

    public Servlet allocate() throws ServletException {
        // Load and initialize our instance if necessary
        if (instance == null) {
            try {
                instance = loadServlet();
            } catch (ServletException e) {
                throw e;
            } catch (Throwable e) {
                throw new ServletException("Cannot allocate a servlet instance", e);
            }
        }
        return instance;
    }

    /**
     * 加载对应的 serlvet
     * @return
     */
    private Servlet loadServlet() throws ServletException {
        if (instance != null)
            return instance;

        Servlet servlet = null;
        String actualClass = servletClass;
        if (actualClass == null) {
            throw new ServletException("servlet class has not been specified");
        }

        Loader loader = getLoader();
        // Acquire an instance of the class loader to be used
        if (loader == null) {
            throw new ServletException("No loader.");
        }
        ClassLoader classLoader = loader.getClassLoader();

        // Load the specified servlet class from the appropriate class loader
        Class classClass = null;
        try {
            if (classLoader != null) {
                classClass = classLoader.loadClass(actualClass);
            }
        } catch (ClassNotFoundException e) {
            throw new ServletException("Servlet class not found");
        }
        // Instantiate and initialize an instance of the servlet class itself
        try {
            servlet = (Servlet) classClass.newInstance();
        } catch (Throwable e) {
            throw new ServletException("Failed to instantiate servlet");
        }

        // Call the initialization method of this servlet
        try {
            servlet.init(null);
        } catch (Throwable f) {
            throw new ServletException("Failed initialize servlet.");
        }
        return servlet;
    }

    public String getInfo() {
        return null;
    }

    /**
     * 获取加载器
     * 如果当前容器没有，返回其父容器的加载器，否则返回null
     * @return 加载器
     */
    public Loader getLoader() {
        if (loader != null)
            return (loader);
        if (parent != null)
            return (parent.getLoader());
        return (null);
    }

    // ... 省略部分代码

    public void invoke(Request request, Response response)
            throws IOException, ServletException {
        pipeline.invoke(request, response);
    }

    public boolean isUnavailable() {
        return false;
    }

    public void load() throws ServletException {
        instance = loadServlet();
    }

    // ... 省略部分代码

    // method implementations of Pipeline
    public Valve getBasic() {
        return pipeline.getBasic();
    }

    public void setBasic(Valve valve) {
        pipeline.setBasic(valve);
    }

    public Valve[] getValves() {
        return pipeline.getValves();
    }

    public void removeValve(Valve valve) {
        pipeline.removeValve(valve);
    }
}

```

- 通过`#getLoader`方法获取加载器，`#load`方法用于加载`servlet`类

### `SimpleWrapperValve`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 基本阀门
 */
public class SimpleWrapperValve implements Valve, Contained {

    protected Container container;

    /**
     * 处理方法
     * @param request 请求对象
     * @param response 响应对象
     * @param valveContext 阀门上下文对象
     */
    public void invoke(Request request, Response response, ValveContext valveContext)
            throws IOException, ServletException {

        SimpleWrapper wrapper = (SimpleWrapper) getContainer();
        ServletRequest sreq = request.getRequest();
        ServletResponse sres = response.getResponse();
        Servlet servlet = null;
        HttpServletRequest hreq = null;
        if (sreq instanceof HttpServletRequest)
            hreq = (HttpServletRequest) sreq;
        HttpServletResponse hres = null;
        if (sres instanceof HttpServletResponse)
            hres = (HttpServletResponse) sres;

        // 定位 servlet 处理请求
        try {
            servlet = wrapper.allocate();
            if (hres != null && hreq != null) {
                servlet.service(hreq, hres);
            } else {
                servlet.service(sreq, sres);
            }
        } catch (ServletException e) {
        }
    }

    public String getInfo() {
        return null;
    }

    public Container getContainer() {
        return container;
    }

    public void setContainer(Container container) {
        this.container = container;
    }
}

```

- 基础阀门，流水线作业的最后一个步骤会调用此阀门的`#invoke`方法

### `ClientIPLoggerValve`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: IP阀门
 */
public class ClientIPLoggerValve implements Valve, Contained {

    protected Container container;

    public void invoke(Request request, Response response, ValveContext valveContext)
            throws IOException, ServletException {

        // Pass this request on to the next valve in our pipeline
        valveContext.invokeNext(request, response);
        System.out.println("Client IP Logger Valve");
        ServletRequest sreq = request.getRequest();
        System.out.println(sreq.getRemoteAddr());
        System.out.println("------------------------------------");
    }

    public String getInfo() {
        return "IP阀门";
    }

    public Container getContainer() {
        return container;
    }

    public void setContainer(Container container) {
        this.container = container;
    }
}

```

- 示例阀门之一，用于打印请求IP信息

### `HeaderLoggerValve`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 请求头 阀门
 */
public class HeaderLoggerValve implements Valve, Contained {

    protected Container container;

    public void invoke(Request request, Response response, ValveContext valveContext)
            throws IOException, ServletException {

        // Pass this request on to the next valve in our pipeline
        valveContext.invokeNext(request, response);

        System.out.println("Header Logger Valve");
        ServletRequest sreq = request.getRequest();
        if (sreq instanceof HttpServletRequest) {
            HttpServletRequest hreq = (HttpServletRequest) sreq;
            Enumeration headerNames = hreq.getHeaderNames();
            while (headerNames.hasMoreElements()) {
                String headerName = headerNames.nextElement().toString();
                String headerValue = hreq.getHeader(headerName);
                System.out.println(headerName + ":" + headerValue);
            }

        } else
            System.out.println("Not an HTTP Request");

        System.out.println("------------------------------------");
    }

    public String getInfo() {
        return "请求头阀门";
    }

    public Container getContainer() {
        return container;
    }

    public void setContainer(Container container) {
        this.container = container;
    }
}
```

- 示例阀门之二，用于打印Http请求头部信息

### `Bootstrap`

```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-9
 * Description: 启动类
 */
public class Bootstrap1 {


    /**
     * 启动一个只包含一个Wrapper实例的Web应用
     * http://localhost:8080/ModernServlet
     */
    public static void main(String[] args) {

        HttpConnector connector = new HttpConnector();

        // 实例化容器
        Wrapper wrapper = new SimpleWrapper();
        wrapper.setServletClass("ModernServlet");
        wrapper.setLoader(new SimpleLoader());

        // 实例化阀门并添加
        Valve valve1 = new HeaderLoggerValve();
        Valve valve2 = new ClientIPLoggerValve();
        ((Pipeline) wrapper).addValve(valve1);
        ((Pipeline) wrapper).addValve(valve2);

        // 设置容器(单个Wrapper容器)
        connector.setContainer(wrapper);

        try {
            connector.initialize();
            connector.start();

            // make the application wait until we press a key.
            System.in.read();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

- 启动类，实例化一个容器并添加到连接器，启动

## 测试

- 访问地址`http://localhost:8080/ModernServlet`

控制台打印信息如下:

```tex
HttpConnector Opening server socket on all host IP addresses
HttpConnector[8080] Starting background thread
ModernServlet -- init
Client IP Logger Valve
0:0:0:0:0:0:0:1
------------------------------------
Header Logger Valve
host:localhost:8080
connection:keep-alive
upgrade-insecure-requests:1
user-agent:Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36
sec-fetch-mode:navigate
accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
sec-fetch-site:none
accept-encoding:gzip, deflate, br
accept-language:zh-CN,zh;q=0.9
------------------------------------
```


---
title: Tomcat-StandardWrapper
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 1604995939
date: 2019-10-20 00:00:00
---

主要介绍了标准包装器`StandardWrapper`如何实例化servlet
<!-- More -->

## 请求处理过程
如图:
{% asset_img Http请求处理过程.png Http请求处理过程 %}

- 连接器接收到请求，创建请求和响应对象
- 连接器调用`StandardContext`的`#invoke`方法
- `StandardContext`的`#invoke`方法调用流水线的`#invoke`方法，所以`StandardContext`的流水线会调用`StandardContextValve`的`#invoke`方法
- `StandardContextValve`的`#invoke`方法会找到合适的`Wrapper`对请求服务并调用包装器的`#invoke`方法
- `StandardWrapper`是包装器的标准实现,`StandardWrapper`对象的`#invoke`方法调用流水线的`#invoke`方法
- `StandardWrapper`流水线的基本阀门是`StandardWrapperValve`。因此`StandardWrapperValve`的`#invoke`方法会被调用
- `StandardWrapperValve`的`#invoke`方法会调用包装器的`#allocate`方法获得一个`servlet`实例
- 当一个`servlet`需要被加载的时候，方法`#allocate`调用方法`#load`来加载一个`servlet`
- 方法`#load`会调用`servlet`的`#init`方法

## SingleThreadModel
```java
package javax.servlet;

/** @deprecated */
public interface SingleThreadModel {
}
```
- 实现该接口的`servlet`可以保证不会有两个线程同时使用`servlet`的`#service`方法，**PS:**1. 只保证`#service`方法的线程安全，不能保证整个`servlet`是线程安全的 2. 已经废弃，Servlet2.3 和 Servlet 2.4 的容器还支持此接口

## StandardWrapper
- 一个`StandardWrapper`对象的主要职责是：加载它表示的`servlet`并分配它一个实例。调用`#service`方法交给`StandardWrapperValve`去完成
- 加载`servlet`时，如果没有实现`SingleThreadModel`接口，`StandardWrapper`加载给`servlet`一次，对于以后的请求返回相同的实例即可。`StandardWrapper`假设`servlet`的`#service`方法是线程安全的，所以没有创建`servlet`的多个实例，线程安全需要开发者自己解决。
- 对于 `STM servlet`,`StandardWrapper`必须保证不能同时有两个线程提交`STM servlet`的`#service`方法

### Allocate the Servlet
```java
    public Servlet allocate() throws ServletException {
        if (this.debug >= 1) {
            this.log("Allocating an instance");
        }

        if (this.unloading) {
            throw new ServletException(ContainerBase.sm.getString("standardWrapper.unloading", this.getName()));
        } else {
            if (!this.singleThreadModel) {  // <1> 非 STM servlet的加载
                if (this.instance == null) {
                    synchronized(this) {
                        if (this.instance == null) {
                            try {
                                this.instance = this.loadServlet(); // <1.1> 加载Serlvet
                            } catch (ServletException var10) {
                                throw var10;
                            } catch (Throwable var11) {
                                throw new ServletException(ContainerBase.sm.getString("standardWrapper.allocate"), var11);
                            }
                        }
                    }
                }

                if (!this.singleThreadModel) {
                    if (this.debug >= 2) {
                        this.log("  Returning non-STM instance");
                    }

                    ++this.countAllocated;
                    return this.instance;
                }
            }

            Stack var1 = this.instancePool;
            synchronized(var1) {    // <2> STM servlet的加载
                while(this.countAllocated >= this.nInstances) {
                    if (this.nInstances < this.maxInstances) {
                        try {
                            this.instancePool.push(this.loadServlet());
                            ++this.nInstances;
                        } catch (ServletException var6) {
                            throw var6;
                        } catch (Throwable var7) {
                            throw new ServletException(ContainerBase.sm.getString("standardWrapper.allocate"), var7);
                        }
                    } else {
                        try {
                            this.instancePool.wait();
                        } catch (InterruptedException var8) {
                        }
                    }
                }

                if (this.debug >= 2) {
                    this.log("  Returning allocated STM instance");
                }

                ++this.countAllocated;
                Servlet var2 = (Servlet)this.instancePool.pop();
                return var2;
            }
        }
    }
```
- `<1>`处，负责完成非`STM servlet`的加载。调用`#loadServlet()`加载`servlet`
- `<2>`处，完成`STM servlet`的加载，循环检查，直到当前实例的数量小于`countAllocated`，如果实例树小于`maxInstances`，调用`#loadServlet`方法并将实例加入实例池中，如果当前实例数`nInstances`大于等于`maxInstances`他会调用池堆栈的`#wait()`方法，直到一个实例被返回。

### Loading the Servlet
```java
    public synchronized Servlet loadServlet() throws ServletException {
        // 如果不是STM servlet并且该实例已经加载过，直接返回
        if (!this.singleThreadModel && this.instance != null) { // <1> 加载 非STM servlet
            return this.instance;
        } else {
            PrintStream out = System.out;
            SystemLogHandler.startCapture();
            Servlet servlet = null;

            try {
                String actualClass = this.servletClass;
                if (actualClass == null && this.jspFile != null) {  // <2> 请求 jsp 页面
                    Wrapper jspWrapper = (Wrapper)((Context)this.getParent()).findChild("jsp");
                    if (jspWrapper != null) {
                        actualClass = jspWrapper.getServletClass();
                    }
                }

                // 如果未指定servlet类
                if (actualClass == null) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.notClass", this.getName()));
                }

                // 判断加载器
                Loader loader = this.getLoader();
                if (loader == null) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.missingLoader", this.getName()));
                }

                ClassLoader classLoader = loader.getClassLoader();
                if (this.isContainerProvidedServlet(actualClass)) { // <3> 如果该servlet是特殊的servlet
                    classLoader = this.getClass().getClassLoader();
                    this.log(ContainerBase.sm.getString("standardWrapper.containerServlet", this.getName()));
                }

                Class classClass = null;

                // <4> 加载 servlet
                try {
                    if (classLoader != null) {
                        classClass = classLoader.loadClass(actualClass);
                    } else {
                        classClass = Class.forName(actualClass);
                    }
                } catch (ClassNotFoundException var22) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.missingClass", actualClass), var22);
                }

                if (classClass == null) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.missingClass", actualClass));
                }

                // <5> 实例化 servlet
                try {
                    servlet = (Servlet)classClass.newInstance();
                } catch (ClassCastException var20) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.notServlet", actualClass), var20);
                } catch (Throwable var21) {
                    this.unavailable((UnavailableException)null);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.instantiate", actualClass), var21);
                }

                if (!this.isServletAllowed(servlet)) {  // <6> 检查 servlet是否可以访问
                    throw new SecurityException(ContainerBase.sm.getString("standardWrapper.privilegedServlet", actualClass));
                }

                // <7> 检查该 servlet是否属于 ContainerServlet
                if (servlet instanceof ContainerServlet && this.isContainerProvidedServlet(actualClass)) {
                    ((ContainerServlet)servlet).setWrapper(this);
                }

                // <8>
                try {
                    this.instanceSupport.fireInstanceEvent("beforeInit", servlet);
                    servlet.init(this.facade);
                    if (this.loadOnStartup > 0 && this.jspFile != null) {   // <9> 如果Servlet是JSP页面且loadOnStarup 大于 0
                        HttpRequestBase req = new HttpRequestBase();
                        HttpResponseBase res = new HttpResponseBase();
                        req.setServletPath(this.jspFile);
                        req.setQueryString("jsp_precompile=true");
                        servlet.service(req, res);
                    }

                    this.instanceSupport.fireInstanceEvent("afterInit", servlet);
                } catch (UnavailableException var23) {
                    this.instanceSupport.fireInstanceEvent("afterInit", servlet, var23);
                    this.unavailable(var23);
                    throw var23;
                } catch (ServletException var24) {
                    this.instanceSupport.fireInstanceEvent("afterInit", servlet, var24);
                    throw var24;
                } catch (Throwable var25) {
                    this.instanceSupport.fireInstanceEvent("afterInit", servlet, var25);
                    throw new ServletException(ContainerBase.sm.getString("standardWrapper.initException", this.getName()), var25);
                }

                this.singleThreadModel = servlet instanceof SingleThreadModel;
                if (this.singleThreadModel && this.instancePool == null) {  // <10> 
                    this.instancePool = new Stack();
                }

                this.fireContainerEvent("load", this);
            } finally {
                // <11>
                String log = SystemLogHandler.stopCapture();
                if (log != null && log.length() > 0) {
                    if (this.getServletContext() != null) {
                        this.getServletContext().log(log);
                    } else {
                        out.println(log);
                    }
                }

            }

            return servlet;
        }
    }
```
- `<1>`处，加载非`STM servlet`处理
- `<2>`处，如果请求的JSP页面，进行相应处理
- `<3>`处，如果请求的`servlet`是Tomcat的特殊`Servlet`,获取另一个`ClassLoader`实例，这样就可以访问`Catalina`内部了
- `<4>`处，开始加载`servlet`类
- `<5>`处，开始实例化`servlet`类
- `<6>`处，检查该`servlet`是否可以被访问
- `<7>`处，`ContainerServlet`实现了`org.apache.catalina.CotainerServlet`,它可以访问`Catalina`的内部函数，如果该`Servlet`是`ContainerServlet`，调用`#ContainerServlet.setWrapper()`，传递该`StandardWrapper`实例
- `<8>`处，触发`BEFORE_INIT_EVENT`事件，并调用发送者的`#init()`方法，参数`facade`是一个`javax.servlet.ServletConfig`对象
- `<9>`处，如果`Servlet`是JSP页面且`loadOnStarup` 大于 0,调用该`Servlet`的`#service`方法
- `<10>`处，如果该`Servlet`是一个`STM Servlet`并且实例池为空，创建实例池
- `<11>`处，停止捕获`System.out`和`System.err`，并将加载过程中的信息使用该容器的`#log()`方法记录到日志系统

### ServletConfig 对象
- `StandardWrapper`实现了`javax.servlet.ServletConfig`接口，该接口有以下四个方法`#getServletContext()`,`#getServletName()`,`#getInitParameter()`和`#getInitParameterNames()`

#### getServletContext 
```java
    public ServletContext getServletContext() {
        if (this.parent == null) {
            return null;
        } else {
            return !(this.parent instanceof Context) ? null : ((Context)this.parent).getServletContext();
        }
    }
```
- **PS:**目前不能单独部署一个包装器来表示一个`Servlet`,包装器必须从属于一个上下文容器，这样才能使用`ServletConfig`

#### getServletName
```java
    public String getServletName() {
        return this.getName();
    }

    public String getName() {
        return this.name;
    }
```

#### getInitParameter
- `StandardWrapper`通过`#addInitParameter`添加参数,保存在`HashMap`类`parameters`中

### Parent and Children
- 一个包装器表示一个独立的`Servlet`的容器，包装器不能有子容器，父容器只能是一个上下文容器。相关代码如下:
```java
    public void addChild(Container child) {
        throw new IllegalStateException(ContainerBase.sm.getString("standardWrapper.notChild"));
    }

    public void setParent(Container container) {
        if (container != null && !(container instanceof Context)) {
            throw new IllegalArgumentException(ContainerBase.sm.getString("standardWrapper.notContext"));
        } else {
            super.setParent(container);
        }
    }
```

### StandardWrapperFacade
```java
public final class StandardWrapperFacade implements ServletConfig {
    private ServletConfig config = null;

    public StandardWrapperFacade(StandardWrapper config) {
        this.config = config;
    }

    public String getServletName() {
        return this.config.getServletName();
    }

    public ServletContext getServletContext() {
        ServletContext theContext = this.config.getServletContext();
        if (theContext != null && theContext instanceof ApplicationContext) {
            theContext = ((ApplicationContext)theContext).getFacade();
        }

        return theContext;
    }

    public String getInitParameter(String name) {
        return this.config.getInitParameter(name);
    }

    public Enumeration getInitParameterNames() {
        return this.config.getInitParameterNames();
    }
}
```
- 外观类，保证安全，方法内部调用`StandardWrapper`相关方法

### StandardWrapperValve
```java
final class StandardWrapperValve extends ValveBase {
    private int debug = 0;
    private FilterDef filterDef = null;
    private static final String info = "org.apache.catalina.core.StandardWrapperValve/1.0";
    private static final StringManager sm = StringManager.getManager("org.apache.catalina.core");

    StandardWrapperValve() {
    }

    public String getInfo() {
        return "org.apache.catalina.core.StandardWrapperValve/1.0";
    }

    /**
     * 阀门处理方法
     *
     * @param      request           Http请求对象
     * @param      response          Http响应对象
     * @param      valveContext      阀门上下文对象
     *
     */
    public void invoke(Request request, Response response, ValveContext valveContext) throws IOException, ServletException {
        boolean unavailable = false;
        Throwable throwable = null;
        StandardWrapper wrapper = (StandardWrapper)this.getContainer();
        ServletRequest sreq = request.getRequest();
        ServletResponse sres = response.getResponse();
        Servlet servlet = null;
        HttpServletRequest hreq = null;
        if (sreq instanceof HttpServletRequest) {
            hreq = (HttpServletRequest)sreq;
        }

        HttpServletResponse hres = null;
        if (sres instanceof HttpServletResponse) {
            hres = (HttpServletResponse)sres;
        }

        if (!((Context)wrapper.getParent()).getAvailable()) {
            hres.sendError(503, sm.getString("standardContext.isUnavailable"));
            unavailable = true;
        }

        if (!unavailable && wrapper.isUnavailable()) {
            this.log(sm.getString("standardWrapper.isUnavailable", wrapper.getName()));
            if (hres != null) {
                long available = wrapper.getAvailable();
                if (available > 0L && available < 9223372036854775807L) {
                    hres.setDateHeader("Retry-After", available);
                }

                hres.sendError(503, sm.getString("standardWrapper.isUnavailable", wrapper.getName()));
            }

            unavailable = true;
        }

        try {
            if (!unavailable) {
                servlet = wrapper.allocate();   // <1> 获取 servlet 实例
            }
        } catch (ServletException var19) {
            this.log(sm.getString("standardWrapper.allocateException", wrapper.getName()), var19);
            throwable = var19;
            this.exception(request, response, var19);
            servlet = null;
        } catch (Throwable var20) {
            this.log(sm.getString("standardWrapper.allocateException", wrapper.getName()), var20);
            throwable = var20;
            this.exception(request, response, var20);
            servlet = null;
        }

        try {
            response.sendAcknowledgement();
        } catch (IOException var17) {
            sreq.removeAttribute("org.apache.catalina.jsp_file");
            this.log(sm.getString("standardWrapper.acknowledgeException", wrapper.getName()), var17);
            throwable = var17;
            this.exception(request, response, var17);
        } catch (Throwable var18) {
            this.log(sm.getString("standardWrapper.acknowledgeException", wrapper.getName()), var18);
            throwable = var18;
            this.exception(request, response, var18);
            servlet = null;
        }

        ApplicationFilterChain filterChain = this.createFilterChain(request, servlet);  // <2> 获取过滤链

        try {
            String jspFile = wrapper.getJspFile();
            if (jspFile != null) {
                sreq.setAttribute("org.apache.catalina.jsp_file", jspFile);
            } else {
                sreq.removeAttribute("org.apache.catalina.jsp_file");
            }

            if (servlet != null && filterChain != null) {
                filterChain.doFilter(sreq, sres);   // <3> 执行过滤方法
            }

            sreq.removeAttribute("org.apache.catalina.jsp_file");
        } catch (IOException var24) {
            sreq.removeAttribute("org.apache.catalina.jsp_file");
            this.log(sm.getString("standardWrapper.serviceException", wrapper.getName()), var24);
            throwable = var24;
            this.exception(request, response, var24);
        } catch (UnavailableException var25) {
            sreq.removeAttribute("org.apache.catalina.jsp_file");
            this.log(sm.getString("standardWrapper.serviceException", wrapper.getName()), var25);
            wrapper.unavailable(var25);
            long available = wrapper.getAvailable();
            if (available > 0L && available < 9223372036854775807L) {
                hres.setDateHeader("Retry-After", available);
            }

            hres.sendError(503, sm.getString("standardWrapper.isUnavailable", wrapper.getName()));
        } catch (ServletException var26) {
            sreq.removeAttribute("org.apache.catalina.jsp_file");
            this.log(sm.getString("standardWrapper.serviceException", wrapper.getName()), var26);
            throwable = var26;
            this.exception(request, response, var26);
        } catch (Throwable var27) {
            sreq.removeAttribute("org.apache.catalina.jsp_file");
            this.log(sm.getString("standardWrapper.serviceException", wrapper.getName()), var27);
            throwable = var27;
            this.exception(request, response, var27);
        }

        try {
            if (filterChain != null) {
                filterChain.release();  // <4> 释放过滤链
            }
        } catch (Throwable var23) {
            this.log(sm.getString("standardWrapper.releaseFilters", wrapper.getName()), var23);
            if (throwable == null) {
                throwable = var23;
                this.exception(request, response, var23);
            }
        }

        try {
            if (servlet != null) {
                wrapper.deallocate(servlet);    // <5> 释放servlet 
            }
        } catch (Throwable var22) {
            this.log(sm.getString("standardWrapper.deallocateException", wrapper.getName()), var22);
            if (throwable == null) {
                throwable = var22;
                this.exception(request, response, var22);
            }
        }

        try {
            if (servlet != null && wrapper.getAvailable() == 9223372036854775807L) {    // <6> 如果servlet 无法使用
                wrapper.unload();
            }
        } catch (Throwable var21) {
            this.log(sm.getString("standardWrapper.unloadException", wrapper.getName()), var21);
            if (throwable == null) {
                this.exception(request, response, var21);
            }
        }

    }

    private ApplicationFilterChain createFilterChain(Request request, Servlet servlet) {
        if (servlet == null) {
            return null;
        } else {
            ApplicationFilterChain filterChain = new ApplicationFilterChain();
            filterChain.setServlet(servlet);
            StandardWrapper wrapper = (StandardWrapper)this.getContainer();
            filterChain.setSupport(wrapper.getInstanceSupport());
            StandardContext context = (StandardContext)wrapper.getParent();
            FilterMap[] filterMaps = context.findFilterMaps();
            if (filterMaps != null && filterMaps.length != 0) {
                String requestPath = null;
                if (request instanceof HttpRequest) {
                    HttpServletRequest hreq = (HttpServletRequest)request.getRequest();
                    String contextPath = hreq.getContextPath();
                    if (contextPath == null) {
                        contextPath = "";
                    }

                    String requestURI = ((HttpRequest)request).getDecodedRequestURI();
                    if (requestURI.length() >= contextPath.length()) {
                        requestPath = requestURI.substring(contextPath.length());
                    }
                }

                String servletName = wrapper.getName();
                int n = 0;

                for(int i = 0; i < filterMaps.length; ++i) {
                    if (this.matchFiltersURL(filterMaps[i], requestPath)) {
                        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)context.findFilterConfig(filterMaps[i].getFilterName());
                        if (filterConfig != null) {
                            filterChain.addFilter(filterConfig);
                            ++n;
                        }
                    }
                }

                for(int i = 0; i < filterMaps.length; ++i) {
                    if (this.matchFiltersServlet(filterMaps[i], servletName)) {
                        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)context.findFilterConfig(filterMaps[i].getFilterName());
                        if (filterConfig != null) {
                            filterChain.addFilter(filterConfig);
                            ++n;
                        }
                    }
                }

                return filterChain;
            } else {
                return filterChain;
            }
        }
    }

    private void exception(Request request, Response response, Throwable exception) {
        ServletRequest sreq = request.getRequest();
        sreq.setAttribute("javax.servlet.error.exception", exception);
        ServletResponse sresponse = response.getResponse();
        if (sresponse instanceof HttpServletResponse) {
            ((HttpServletResponse)sresponse).setStatus(500);
        }

    }

    private void log(String message) {
        Logger logger = null;
        if (this.container != null) {
            logger = this.container.getLogger();
        }

        if (logger != null) {
            logger.log("StandardWrapperValve[" + this.container.getName() + "]: " + message);
        } else {
            String containerName = null;
            if (this.container != null) {
                containerName = this.container.getName();
            }

            System.out.println("StandardWrapperValve[" + containerName + "]: " + message);
        }

    }

    private void log(String message, Throwable throwable) {
        Logger logger = null;
        if (this.container != null) {
            logger = this.container.getLogger();
        }

        if (logger != null) {
            logger.log("StandardWrapperValve[" + this.container.getName() + "]: " + message, throwable);
        } else {
            String containerName = null;
            if (this.container != null) {
                containerName = this.container.getName();
            }

            System.out.println("StandardWrapperValve[" + containerName + "]: " + message);
            System.out.println("" + throwable);
            throwable.printStackTrace(System.out);
        }

    }

    private boolean matchFiltersServlet(FilterMap filterMap, String servletName) {
        return servletName == null ? false : servletName.equals(filterMap.getServletName());
    }

    private boolean matchFiltersURL(FilterMap filterMap, String requestPath) {
        if (requestPath == null) {
            return false;
        } else {
            String testPath = filterMap.getURLPattern();
            if (testPath == null) {
                return false;
            } else if (testPath.equals(requestPath)) {
                return true;
            } else if (testPath.equals("/*")) {
                return true;
            } else {
                int slash;
                if (!testPath.endsWith("/*")) {
                    if (testPath.startsWith("*.")) {
                        int slash = requestPath.lastIndexOf(47);
                        slash = requestPath.lastIndexOf(46);
                        if (slash >= 0 && slash > slash) {
                            return testPath.equals("*." + requestPath.substring(slash + 1));
                        }
                    }

                    return false;
                } else {
                    for(String comparePath = requestPath; !testPath.equals(comparePath + "/*"); comparePath = comparePath.substring(0, slash)) {
                        slash = comparePath.lastIndexOf(47);
                        if (slash < 0) {
                            return false;
                        }
                    }

                    return true;
                }
            }
        }
    }
}
```
- `<1>`处，调用`#allocate()`方法获取`servlet`实例
- `<2>`处，调用`#createFilterChain`方法获取过滤链
- `<3>`处，调用过滤链的`#doFilter()`方法，包括`#servlet.service()`方法
- `<4>`处，释放过滤链
- `<5>`处，释放`servlet`，`countAllocated--`,如果是`STM servlet`，将`servlet`加入实例池
- `<6>`处，如果`servlet`无法使用了，调用包装器的`#unload()`方法

#### FilterDef
```java
public final class FilterDef {
    private String description = null;
    /**
     * 展示名称
     */
    private String displayName = null;
    /**
     * 实现接口的过滤器全限定名
     */
    private String filterClass = null;
    /**
     * 过滤器名称，保证唯一
     */
    private String filterName = null;
    /**
     * 与此过滤器关联的大图标
     */
    private String largeIcon = null;
    /**
     * 初始化参数
     */
    private Map parameters = new HashMap();
    /**
     * 与此过滤器关联的小图标
     */
    private String smallIcon = null;

    public FilterDef() {
    }

    // Getter or Setter

    /**
     * 添加初始化参数
     *
     * @param      name   参数名称
     * @param      Valve  参数值
     */
    public void addInitParameter(String name, String Valve) {
        this.parameters.put(name, Valve);
    }

    public String toString() {
        StringBuffer sb = new StringBuffer("FilterDef[");
        sb.append("filterName=");
        sb.append(this.filterName);
        sb.append(", filterClass=");
        sb.append(this.filterClass);
        sb.append("]");
        return sb.toString();
    }
}
```
- 一个`FilterDef`表示一个过滤器定义，每一个属性都表示一个可以在过滤器中出现的子元素

#### ApplicationFilterConfig
```java
final class ApplicationFilterConfig implements FilterConfig {
    private Context context = null;
    private Filter filter = null;
    private FilterDef filterDef = null;

    public ApplicationFilterConfig(Context context, FilterDef filterDef) throws ClassCastException, ClassNotFoundException, IllegalAccessException, InstantiationException, ServletException {
        this.context = context;
        this.setFilterDef(filterDef);
    }

    public String getFilterName() {
        return this.filterDef.getFilterName();
    }

    public String getInitParameter(String name) {
        Map map = this.filterDef.getParameterMap();
        return map == null ? null : (String)map.get(name);
    }

    public Enumeration getInitParameterNames() {
        Map map = this.filterDef.getParameterMap();
        return map == null ? new Enumerator(new ArrayList()) : new Enumerator(map.keySet());
    }

    public ServletContext getServletContext() {
        return this.context.getServletContext();
    }

    public String toString() {
        StringBuffer sb = new StringBuffer("ApplicationFilterConfig[");
        sb.append("name=");
        sb.append(this.filterDef.getFilterName());
        sb.append(", filterClass=");
        sb.append(this.filterDef.getFilterClass());
        sb.append("]");
        return sb.toString();
    }

    Filter getFilter() throws ClassCastException, ClassNotFoundException, IllegalAccessException, InstantiationException, ServletException {
        if (this.filter != null) {
            return this.filter;
        } else {
            String filterClass = this.filterDef.getFilterClass();
            ClassLoader classLoader = null;
            if (filterClass.startsWith("org.apache.catalina.")) {
                classLoader = this.getClass().getClassLoader();
            } else {
                classLoader = this.context.getLoader().getClassLoader();
            }

            ClassLoader oldCtxClassLoader = Thread.currentThread().getContextClassLoader();
            Class clazz = classLoader.loadClass(filterClass);
            this.filter = (Filter)clazz.newInstance();
            this.filter.init(this);
            return this.filter;
        }
    }

    FilterDef getFilterDef() {
        return this.filterDef;
    }

    void release() {
        if (this.filter != null) {
            this.filter.destroy();
        }

        this.filter = null;
    }

    void setFilterDef(FilterDef filterDef) throws ClassCastException, ClassNotFoundException, IllegalAccessException, InstantiationException, ServletException {
        this.filterDef = filterDef;
        if (filterDef == null) {
            if (this.filter != null) {
                this.filter.destroy();
            }

            this.filter = null;
        } else {
            Filter var2 = this.getFilter();
        }
    }
}
```
- `ApplicationFilterConfig`负责管理web应用程序启动时创建的过滤器实例
- `#getFilter()`方法负责加载过滤器并初始化过滤器

#### ApplicationFilterChain
```java
final class ApplicationFilterChain implements FilterChain {
    private ArrayList filters = new ArrayList();
    private Iterator iterator = null;
    private Servlet servlet = null;
    private static final StringManager sm = StringManager.getManager("org.apache.catalina.core");
    private InstanceSupport support = null;

    public ApplicationFilterChain() {
    }

    public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
        if (System.getSecurityManager() != null) {
            final ServletRequest req = request;
            final ServletResponse res = response;

            try {
                AccessController.doPrivileged(new PrivilegedExceptionAction() {
                    public Object run() throws ServletException, IOException {
                        ApplicationFilterChain.this.internalDoFilter(req, res);
                        return null;
                    }
                });
            } catch (PrivilegedActionException var7) {
                Exception e = var7.getException();
                if (e instanceof ServletException) {
                    throw (ServletException)e;
                }

                if (e instanceof IOException) {
                    throw (IOException)e;
                }

                if (e instanceof RuntimeException) {
                    throw (RuntimeException)e;
                }

                throw new ServletException(e.getMessage(), e);
            }
        } else {
            this.internalDoFilter(request, response);
        }

    }

    private void internalDoFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
        if (this.iterator == null) {
            this.iterator = this.filters.iterator();
        }

        if (this.iterator.hasNext()) {
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)this.iterator.next();
            Filter filter = null;

            try {
                filter = filterConfig.getFilter();
                this.support.fireInstanceEvent("beforeFilter", filter, request, response);
                filter.doFilter(request, response, this);
                this.support.fireInstanceEvent("afterFilter", filter, request, response);
            } catch (IOException var9) {
                if (filter != null) {
                    this.support.fireInstanceEvent("afterFilter", filter, request, response, var9);
                }

                throw var9;
            } catch (ServletException var10) {
                if (filter != null) {
                    this.support.fireInstanceEvent("afterFilter", filter, request, response, var10);
                }

                throw var10;
            } catch (RuntimeException var11) {
                if (filter != null) {
                    this.support.fireInstanceEvent("afterFilter", filter, request, response, var11);
                }

                throw var11;
            } catch (Throwable var12) {
                if (filter != null) {
                    this.support.fireInstanceEvent("afterFilter", filter, request, response, var12);
                }

                throw new ServletException(sm.getString("filterChain.filter"), var12);
            }
        } else {
            try {
                this.support.fireInstanceEvent("beforeService", this.servlet, request, response);
                if (request instanceof HttpServletRequest && response instanceof HttpServletResponse) {
                    this.servlet.service((HttpServletRequest)request, (HttpServletResponse)response);
                } else {
                    this.servlet.service(request, response);
                }

                this.support.fireInstanceEvent("afterService", this.servlet, request, response);
            } catch (IOException var13) {
                this.support.fireInstanceEvent("afterService", this.servlet, request, response, var13);
                throw var13;
            } catch (ServletException var14) {
                this.support.fireInstanceEvent("afterService", this.servlet, request, response, var14);
                throw var14;
            } catch (RuntimeException var15) {
                this.support.fireInstanceEvent("afterService", this.servlet, request, response, var15);
                throw var15;
            } catch (Throwable var16) {
                this.support.fireInstanceEvent("afterService", this.servlet, request, response, var16);
                throw new ServletException(sm.getString("filterChain.servlet"), var16);
            }
        }
    }

    void addFilter(ApplicationFilterConfig filterConfig) {
        this.filters.add(filterConfig);
    }

    void release() {
        this.filters.clear();
        this.iterator = this.iterator;
        this.servlet = null;
    }

    void setServlet(Servlet servlet) {
        this.servlet = servlet;
    }

    void setSupport(InstanceSupport support) {
        this.support = support;
    }
}
```
- 重点关注它的`#doFilter()`方法，最终调用`#internalDoFilter()`方法，方法中，如果当前过滤器不是链中最后一个过滤器,调用`#this.doFilter()`执行过滤，否则调用`#servlet.service()`的方法
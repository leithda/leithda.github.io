title: Tomcat-加载器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-10-16
---

- 前面介绍的加载器实例，加载的`servlet`实例可以使用`classpath`下的任何类和类库，这样是不安全的，`servlet`应该只允许 访问`WEB-INF`目录及子目录下的类及`WEB-INF/lib`下的类库
- 加载器应该支持`WEB-INF`下目录变更或者`WEB-INF/lib`目录改变时重新加载。Tomcat 的加载器实现中使用一个单独的线程来检查 servlet 和支持类文件的时间戳。要支持类的自动加载 功能,一个加载器类必须实现 org.apache.catalina.loader.Reloader 接口
<!-- More -->

## Java类加载器
- J2SE1.2开始，JVM使用三种类加载器：`bootstrap类加载器`,`extension类加载器`,`system类加载器` 
- `bootstrap类加载器`用于引导JVM,它使用本地代码实现，加载JVM需要的类到函数中。它还负责加载所有的Java核心类，例如`java.lang`、`java.io`包，另外它还会查找核心类库`rt.jar`，`i18n.jar`等，这些类库JVM会根据操作系统来查找
- `extension类加载器`负责加载标准扩展目录下面的类，SUN公司的JVM的标准扩展目录是`/jdk/jre/lib/ext`,将jar包放到目录下即可
- `system类加载器`是默认的加载器,它在环境变量classpath目录下查找相应的类

### 委派模型(delegation model)
- 每次类一加载，`system`首先调用，然后它会将任务委托给它的父类`extension`,`extension`委托给`bootstrap`。因此，`bootstrap`类首先加载类，如加载不到，`extension`类加载，最后`system`进行加载，如果都找不到，会抛出一个`java.lang.ClassNotFoundException`异常。
- 安全性，JVM信任`Object`类，它可以访问任何在硬盘上的内容，恶意的人自己写一个`Object`类，加载时，委托给`bootstrap`后，它首先找到核心库，找到标准的`java.lang.Object`类并实例化它，这样，自定义的`Objet`类永远不会被加载

### 扩展加载器
- 扩展加载器可以通过扩展`java.lang.ClassLoader`来扩展自己的类加载器
- tomcat类加载器
    - 要制定类加载器的某些特定规则
    - 缓存以前加载的类
    - 实现预加载类以预备使用

## 解析 
### `Loader` 接口
```java
public interface Loader {
    ClassLoader getClassLoader();

    Container getContainer();

    void setContainer(Container var1);

    boolean getDelegate();

    void setDelegate(boolean var1);

    String getInfo();

    boolean getReloadable();

    void setReloadable(boolean var1);

    void addPropertyChangeListener(PropertyChangeListener String var = null;ar1);

    void addRepository(String var1);

    String[] findRepositories();

    boolean modified();

    void removePropertyChangeListener(PropertyChangeListener var1);
}
```
- `setDelegate`用于设置是否委派给父类加载器
- `#setReloadable`用于设置是否使用重加载

### 结构图
- `WebappLoader`包含一个`org.apache.catalina.loader.WebappClassLoader`实例，该类扩展了`java.net.URLClassLoader`类。
- 无论何时，容器相关的加载器加载`servlet`时，容器先调用`#getClassLoader`类获取加载器，然后容器调用`#loadClass`方法来加载`servlet`类
- `Loader`和它的实现类结构如图
{% asset_img WebappLoader.png Loader 结构图 %}

### `Reloader`接口
```java
public interface Reloader {
    void addRepository(String var1);

    String[] findRepositories();

    boolean modified();
}
```
- 每个加载器实现重加载需要实现此接口
- `#modified`方法用于判断类是否被改变.在 web 应用程序中的 `servlet` 任何实现类被修改的时候该方法返回 true

### `WebappLoader`类

- `WebappLoader`类是`Loader`接口的实现，它表示一个web应用程序的加载器，负责给web应用程序加载类。
- 它实现了`Lifecycle`接口，可以由关联容器启动和停止
- 它实现了`Runnable`接口，可以通过一个线程重复调用`#modified`方法，如果返回`true`，类通过上下文重新加载自己，具体实现方式后续介绍
- `WebappLoader`类的`#start`方法被调用的时候，它将会完成以下几项重要任务
  - 创建一个类加载器
  - 设置类仓库
  - 设置类路径
  - 设置访问权限
  - 创建一个线程以实现重加载机制

#### 创建类加载器

```java
   /**
    * 创建加载器
    */
  private WebappClassLoader createClassLoader() throws Exception {
        Class clazz = Class.forName(this.loaderClass);
        WebappClassLoader classLoader = null;
        if (this.parentClassLoader == null) {
            classLoader = (WebappClassLoader)clazz.newInstance();
        } else {
            Class[] argTypes = new Class[]{class$java$lang$ClassLoader == null ? (class$java$lang$ClassLoader = class$("java.lang.ClassLoader")) : class$java$lang$ClassLoader};
            Object[] args = new Object[]{this.parentClassLoader};
            Constructor constr = clazz.getConstructor(argTypes);
            classLoader = (WebappClassLoader)constr.newInstance(args);
        }

        return classLoader;
    }
```

- 使用`loaderClass`保存字符串型加载器类名，可以通过`#setLoaderClass`方法加载自定义的加载器，注意自定义加载器要继承`WebappClassLoader`,因为方法返回`WebappClassLoader`类

#### 设置库

```java
    private void setRepositories() {
        if (this.container instanceof Context) {
            ServletContext servletContext = ((Context)this.container).getServletContext();
            if (servletContext != null) {
                File workDir = (File)servletContext.getAttribute("javax.servlet.context.tempdir");
                if (workDir != null) {
                    this.log(sm.getString("webappLoader.deploy", workDir.getAbsolutePath()));
                    DirContext resources = this.container.getResources();   // <1> 
                    String classesPath = "/WEB-INF/classes";
                    DirContext classes = null;

                    Object classRepository;
                    try {
                        classRepository = resources.lookup(classesPath);
                        if (classRepository instanceof DirContext) {
                            classes = (DirContext)classRepository;
                        }
                    } catch (NamingException var18) {
                    }

                    if (classes != null) {  
                        classRepository = null;
                        String absoluteClassesPath = servletContext.getRealPath(classesPath);
                        File classRepository;
                        if (absoluteClassesPath != null) {
                            classRepository = new File(absoluteClassesPath);
                        } else {
                            classRepository = new File(workDir, classesPath);
                            classRepository.mkdirs();
                            this.copyDir(classes, classRepository);
                        }

                        this.log(sm.getString("webappLoader.classDeploy", classesPath, classRepository.getAbsolutePath()));
                        this.classLoader.addRepository(classesPath + "/", classRepository); // <2> 
                    }

                    String libPath = "/WEB-INF/lib";
                    this.classLoader.setJarPath(libPath);   // <3> 
                    DirContext libDir = null;

                    try {
                        Object object = resources.lookup(libPath);
                        if (object instanceof DirContext) {
                            libDir = (DirContext)object;
                        }
                    } catch (NamingException var17) {
                    }

                    if (libDir != null) {
                        boolean copyJars = false;
                        String absoluteLibPath = servletContext.getRealPath(libPath);
                        File destDir = null;
                        if (absoluteLibPath != null) {
                            destDir = new File(absoluteLibPath);
                        } else {
                            copyJars = true;
                            destDir = new File(workDir, libPath);
                            destDir.mkdirs();
                        }

                        try {
                            NamingEnumeration enum = resources.listBindings(libPath);

                            while(true) {
                                String filename;
                                File destFile;
                                Resource jarResource;
                                do {
                                    Binding binding;
                                    do {
                                        if (!enum.hasMoreElements()) {
                                            return;
                                        }

                                        binding = (Binding)enum.nextElement();
                                        filename = libPath + "/" + binding.getName();
                                    } while(!filename.endsWith(".jar"));

                                    destFile = new File(destDir, binding.getName());
                                    this.log(sm.getString("webappLoader.jarDeploy", filename, destFile.getAbsolutePath()));
                                    jarResource = (Resource)binding.getObject();
                                } while(copyJars && !this.copy(jarResource.streamContent(), new FileOutputStream(destFile)));

                                JarFile jarFile = new JarFile(destFile);
                                this.classLoader.addJar(filename, jarFile, destFile);
                            }
                        } catch (NamingException var19) {
                        } catch (IOException var20) {
                            var20.printStackTrace();
                        }
                    }
                }
            }
        }
    }
```

- `DirContext`目录抽象类，包含一些用于检查和更新与对象关联的属性以及搜索目录的方法。
- `<2>`处，将`WEB-INF/classes`路径添加到仓库中
- `<3>`处，将`WEB-INF/lib`路径设置到jarPath

#### 设置类路径

- 该任务由 start 方法调用 `setClassPath` 方法完成,setClassPath 方法会给servlet 上下文分配一个 String 类型属性保存 Jasper JSP 编译的类路径,该内容先不予讨论

#### 设置访问权限

```java
    private void setPermissions() {
        if (System.getSecurityManager() != null) {
            if (this.container instanceof Context) {
                ServletContext servletContext = ((Context)this.container).getServletContext();
                File workDir = (File)servletContext.getAttribute("javax.servlet.context.tempdir");
                if (workDir != null) {
                    try {
                        String workDirPath = workDir.getCanonicalPath();
                        this.classLoader.addPermission(new FilePermission(workDirPath, "read,write"));
                        this.classLoader.addPermission(new FilePermission(workDirPath + File.separator + "-", "read,write,delete"));
                    } catch (IOException var16) {
                    }
                }

                try {
                    URL rootURL = servletContext.getResource("/");
                    this.classLoader.addPermission(rootURL);
                    String contextRoot = servletContext.getRealPath("/");
                    if (contextRoot != null) {
                        try {
                            contextRoot = (new File(contextRoot)).getCanonicalPath() + File.separator;
                            this.classLoader.addPermission(contextRoot);
                        } catch (IOException var14) {
                        }
                    }

                    URL classesURL = servletContext.getResource("/WEB-INF/classes/");
                    if (classesURL != null) {
                        this.classLoader.addPermission(classesURL); // <1>
                    }

                    URL libURL = servletContext.getResource("/WEB-INF/lib/");
                    if (libURL != null) {
                        this.classLoader.addPermission(libURL); // <2>
                    }

                    File classesDir;
                    if (contextRoot != null) {
                        if (libURL != null) {
                            classesDir = new File(contextRoot);
                            File libDir = new File(classesDir, "WEB-INF/lib/");
                            String path = null;

                            try {
                                path = libDir.getCanonicalPath() + File.separator;
                            } catch (IOException var13) {
                            }

                            if (path != null) {
                                this.classLoader.addPermission(path);
                            }
                        }
                    } else if (workDir != null) {
                        String path;
                        if (libURL != null) {
                            classesDir = new File(workDir, "WEB-INF/lib/");
                            path = null;

                            try {
                                path = classesDir.getCanonicalPath() + File.separator;
                            } catch (IOException var12) {
                            }

                            this.classLoader.addPermission(path);
                        }

                        if (classesURL != null) {
                            classesDir = new File(workDir, "WEB-INF/classes/");
                            path = null;

                            try {
                                path = classesDir.getCanonicalPath() + File.separator;
                            } catch (IOException var11) {
                            }

                            this.classLoader.addPermission(path);
                        }
                    }
                } catch (MalformedURLException var15) {
                }

            }
        }
    }
```

- `<1>`和`<2>`处增加权限

#### 开启自动重载线程

```java
    private void threadStart() {
        if (this.thread == null) {
            if (!this.reloadable) {
                throw new IllegalStateException(sm.getString("webappLoader.notReloadable"));
            } else if (!(this.container instanceof Context)) {
                throw new IllegalStateException(sm.getString("webappLoader.notContext"));
            } else {
                if (this.debug >= 1) {
                    this.log(" Starting background thread");
                }

                this.threadDone = false;
                this.threadName = "WebappLoader[" + this.container.getName() + "]";
                this.thread = new Thread(this, this.threadName);
                this.thread.setDaemon(true);
                this.thread.start();
            }
        }
    }
```

- 线程处理方法`#run`方法代码如下

```java
    public void run() {
        if (this.debug >= 1) {
            this.log("BACKGROUND THREAD Starting");
        }

        while(!this.threadDone) {
            this.threadSleep(); // <1>
            if (!this.started) {    // <2>
                break;
            }

            try {
                if (!this.classLoader.modified()) { // <3>
                    continue;
                }
            } catch (Exception var2) {
                this.log(sm.getString("webappLoader.failModifiedCheck"), var2);
                continue;
            }

            this.notifyContext();   // <4> 
            break;
        }

        if (this.debug >= 1) {
            this.log("BACKGROUND THREAD Stopping");
        }

    }
```

- `<1>`处，线程休眠，时间由`checkInterval`定义，单位是秒
- `<2>`处，判断线程启动状态变量`started`,如果线程停止，直接跳出循环
- `<3>`处，调用`#classLoader.modified`方法，如果类被修改了，调用`<4>`处的`#notifyContext`方法，代码如下：

```java
    private void notifyContext() {
        WebappLoader.WebappContextNotifier notifier = new WebappLoader.WebappContextNotifier();
        (new Thread(notifier)).start();
    }
```

- 方法不直接调用`Context`的`#reload`方法，它先初始化一个`WebappLoader.WebappContextNotifier`内部类，然后启动一个线程，这样重载就由另一个线程完成，代码如下：

```java
    protected class WebappContextNotifier implements Runnable {
        protected WebappContextNotifier() {
        }

        public void run() {
            ((Context)WebappLoader.this.container).reload();
        }
    }
```

### `WebappClassLoader`类
- `WebappClassLoader`类继承了`java.net.URLClassLoader`类
- 它缓存了以前加载的类，WebappClassLoader在源列表中以及特定的jar文件中查找类
- 处于安全考虑，它不允许一些特定的类被加载
```java
    private static final String[] triggers = new String[]{"javax.servlet.Servlet"};
```
- 委派给系统加载器时，它不允许加载一些包里的类或者它的子包中的类
```java
private static final String[] packageTriggers = new String[]{"javax", "org.xml.sax", "org.w3c.dom", "org.apache.xerces", "org.apache.xalan"};
```

#### 缓存
- 每个被加载的类都被当做一个源，源被`ResourceEntry`类表示.一个`ResourceEntry`保存一个`byte`类型的数组表示该类，最后修改的数据或者副本等等。
```java
public class ResourceEntry {

    public long lastModified = -1L;
    /**
     * 资源的二进制内容
     */
    public byte[] binaryContent = null;
    public Class loadedClass = null;
    /**
     * 资源被加载的URL
     */
    public URL source = null;
    public URL codeBase = null;
    public Manifest manifest = null;
    public Certificate[] certificates = null;

    public ResourceEntry() {
    }
}
```
#### 加载类
```java
    /**
     * [加载类]
     * @param  name                   类名
     * @param  resolve                是否链接该类，链接后证明该类可用
     * @return                        类对象
     */
    public Class loadClass(String name, boolean resolve) throws ClassNotFoundException {
        if (this.debug >= 2) {
            this.log("loadClass(" + name + ", " + resolve + ")");
        }

        Class clazz = null;
        if (!this.started) {
            this.log("Lifecycle error : CL stopped");
            throw new ClassNotFoundException(name);
        } else {
            clazz = this.findLoadedClass0(name);    // <1>
            if (clazz != null) {
                if (this.debug >= 3) {
                    this.log("  Returning class from cache");
                }

                if (resolve) {
                    this.resolveClass(clazz);
                }

                return clazz;
            } else {
                clazz = this.findLoadedClass(name); // <2> 
                if (clazz != null) {
                    if (this.debug >= 3) {
                        this.log("  Returning class from cache");
                    }

                    if (resolve) {
                        this.resolveClass(clazz);
                    }

                    return clazz;
                } else {
                    try {
                        clazz = this.system.loadClass(name);    // <3>
                        if (clazz != null) {
                            if (resolve) {
                                this.resolveClass(clazz);
                            }

                            return clazz;
                        }
                    } catch (ClassNotFoundException var11) {
                    }


                    if (this.securityManager != null) { // <4> 
                        int i = name.lastIndexOf(46);
                        if (i >= 0) {
                            try {
                                this.securityManager.checkPackageAccess(name.substring(0, i));
                            } catch (SecurityException var10) {
                                String error = "Security Violation, attempt to use Restricted Class: " + name;
                                System.out.println(error);
                                var10.printStackTrace();
                                this.log(error);
                                throw new ClassNotFoundException(error);
                            }
                        }
                    }

                    boolean delegateLoad = this.delegate || this.filter(name);
                    ClassLoader loader;
                    if (delegateLoad) { // <5>
                        if (this.debug >= 3) {
                            this.log("  Delegating to parent classloader");
                        }

                        loader = this.parent;
                        if (loader == null) {
                            loader = this.system;
                        }

                        try {
                            clazz = loader.loadClass(name);
                            if (clazz != null) {
                                if (this.debug >= 3) {
                                    this.log("  Loading class from parent");
                                }

                                if (resolve) {
                                    this.resolveClass(clazz);
                                }

                                return clazz;
                            }
                        } catch (ClassNotFoundException var9) {
                        }
                    }

                    if (this.debug >= 3) {
                        this.log("  Searching local repositories");
                    }

                    try {
                        clazz = this.findClass(name);
                        if (clazz != null) {
                            if (this.debug >= 3) {
                                this.log("  Loading class from local repository");
                            }

                            if (resolve) {
                                this.resolveClass(clazz);
                            }

                            return clazz;
                        }
                    } catch (ClassNotFoundException var8) {
                    }

                    if (!delegateLoad) {    // <6>
                        if (this.debug >= 3) {
                            this.log("  Delegating to parent classloader");
                        }

                        loader = this.parent;
                        if (loader == null) {
                            loader = this.system;
                        }

                        try {
                            clazz = loader.loadClass(name);
                            if (clazz != null) {
                                if (this.debug >= 3) {
                                    this.log("  Loading class from parent");
                                }

                                if (resolve) {
                                    this.resolveClass(clazz);
                                }

                                return clazz;
                            }
                        } catch (ClassNotFoundException var7) {
                        }
                    }

                    throw new ClassNotFoundException(name); // <7>
                }
            }
        }
    }
```
- `<1>`处，判断目标类是否加载过，代码如下:
```java
    protected Class findLoadedClass0(String name) {
        ResourceEntry entry = (ResourceEntry)this.resourceEntries.get(name);
        return entry != null ? entry.loadedClass : null;
    }
```
- `<2>`处，每个类加载器都维护有自己的一份已加载类名字空间，该方法就是在该名字空间中寻找指定的类是否已存在，如果存在就返回给类的引用，否则就返回 null
- `<3>`处，调用系统加载器加载目标类
- `<4>`处，如果使用安全管理器，判断目标类是否允许加载，如果不允许，抛出`ClassNotFound`异常
- `<5>`处，如果使用委派模式，使用父加载器(没有则使用系统加载器)加载目标类
- `<6>`处，如果在当前的源中找不到目标类并且没有使用委派标志,使用父类加载器。如果父类加载器为 null,使用系统加载器
- `<7>`处，如果最终目标类仍然找不到,抛出 ClassNotFoundException 异常

## 应用程序

### 监听器类

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

### 使用前面章节的类
- SimplePipiline
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/10/16
 * Description: 流水线 Copyed from package chapter7.SimplePipeline
 */
public class SimplePipeline implements Pipeline, Lifecycle {

    public SimplePipeline(Container container) {
        setContainer(container);
    }

    // The basic Valve (if any) associated with this Pipeline.
    protected Valve basic = null;
    // The Container with which this Pipeline is associated.
    protected Container container = null;
    // the array of Valves
    protected Valve valves[] = new Valve[0];

    public void setContainer(Container container) {
        this.container = container;
    }

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

    public void invoke(Request request, Response response)
            throws IOException, ServletException {
        // Invoke the first Valve in this pipeline for this request
        (new StandardPipelineValveContext()).invokeNext(request, response);
    }

    public void removeValve(Valve valve) {
    }

    // implementation of the Lifecycle interface's methods
    public void addLifecycleListener(LifecycleListener listener) {
    }

    public LifecycleListener[] findLifecycleListeners() {
        return null;
    }

    public void removeLifecycleListener(LifecycleListener listener) {
    }

    public synchronized void start() throws LifecycleException {
    }

    public void stop() throws LifecycleException {
    }

    // this class is copied from org.apache.catalina.core.StandardPipeline class's
    // StandardPipelineValveContext inner class.
    protected class StandardPipelineValveContext implements ValveContext {
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

- SimpleWrapper
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/10/16
 * Description: 包装类 Copyed from package chapter7.SimpleWrapper
 */
public class SimpleWrapper implements Wrapper, Pipeline, Lifecycle {

    public SimpleWrapper() {
        pipeline.setBasic(new SimpleWrapperValve());
    }

    // the servlet instance
    private Servlet instance = null;
    private String servletClass;
    private Loader loader;
    private String name;
    protected LifecycleSupport lifecycle = new LifecycleSupport(this);
    private SimplePipeline pipeline = new SimplePipeline(this);
    protected Container parent = null;
    protected boolean started = false;

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

    public Servlet loadServlet() throws ServletException {
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

    // ... 省略 getter/setter

    public void invoke(Request request, Response response)
            throws IOException, ServletException {
        pipeline.invoke(request, response);
    }

    public boolean isUnavailable() {
        return false;
    }

    public void load() throws ServletException {
    }

    public Container map(Request request, boolean update) {
        return null;
    }

    public void removeChild(Container child) {
    }

    public void removeContainerListener(ContainerListener listener) {
    }

    public void removeMapper(Mapper mapper) {
    }

    public void removeInitParameter(String name) {
    }

    public void removeInstanceListener(InstanceListener listener) {
    }

    public void removePropertyChangeListener(PropertyChangeListener listener) {
    }

    public void removeSecurityReference(String name) {
    }

    public void unavailable(UnavailableException unavailable) {
    }

    public void unload() throws ServletException {
    }

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

    // implementation of the Lifecycle interface's methods
    public void addLifecycleListener(LifecycleListener listener) {
    }

    public LifecycleListener[] findLifecycleListeners() {
        return null;
    }

    public void removeLifecycleListener(LifecycleListener listener) {
    }

    public synchronized void start() throws LifecycleException {
        System.out.println("Starting Wrapper " + name);
        if (started)
            throw new LifecycleException("Wrapper already started");

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_START_EVENT, null);
        started = true;

        // Start our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(START_EVENT, null);
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
    }

    public void stop() throws LifecycleException {
        System.out.println("Stopping wrapper " + name);
        // Shut down our servlet instance (if it has been initialized)
        try {
            instance.destroy();
        } catch (Throwable t) {
        }
        instance = null;
        if (!started)
            throw new LifecycleException("Wrapper " + name + " not started");
        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(BEFORE_STOP_EVENT, null);

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(STOP_EVENT, null);
        started = false;

        // Stop the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).stop();
        }

        // Stop our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle)) {
            ((Lifecycle) loader).stop();
        }

        // Notify our interested LifecycleListeners
        lifecycle.fireLifecycleEvent(AFTER_STOP_EVENT, null);
    }
}

```

- SimpleWrapperValue
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/10/16
 * Description: 包装器处理器 Copyed from package chapter7.SimpleWrapperValve
 */
public class SimpleWrapperValve implements Valve, Contained {

    protected Container container;

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

        // Allocate a servlet instance to process this request
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

### 启动类
```java
package cn.leithda.htw.chapter8.startup;


import cn.leithda.htw.chapter8.core.SimpleContextConfig;
import cn.leithda.htw.chapter8.core.SimpleWrapper;
import org.apache.catalina.*;
import org.apache.catalina.connector.http.HttpConnector;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.loader.WebappClassLoader;
import org.apache.catalina.loader.WebappLoader;
import org.apache.naming.resources.ProxyDirContext;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-16
 * Description: 启动类
 */
public final class Bootstrap {

    public static void main(String[] args) {
        //invoke: http://localhost:8080/Modern or  http://localhost:8080/Primitive
        // 设置环境变量
        System.setProperty("catalina.base", System.getProperty("user.dir"));

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

        // 创建一个web应用程序
        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/myApp");
        context.setDocBase("myApp");

        context.addChild(wrapper1);
        context.addChild(wrapper2);

        // context.addServletMapping(pattern, name);
        // 添加映射器，负责查找context示例中的子容器来处理http请求
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");

        // add ContextConfig. This listener is important because it configures
        // StandardContext (sets configured to true), otherwise StandardContext
        // won't start
        // 添加context生命周期监听器
        LifecycleListener listener = new SimpleContextConfig();
        ((Lifecycle) context).addLifecycleListener(listener);

        // here is our loader
        // 添加类加载器
        Loader loader = new WebappLoader();
        // associate the loader with the Context
        context.setLoader(loader);

        //将连接器和context容器关联
        connector.setContainer(context);

        try {
            // 连接器初始化
            connector.initialize();

            // 启动连接器
            ((Lifecycle) connector).start();

            //启动容器（web应用程序启动）
            ((Lifecycle) context).start();
            // now we want to know some details about WebappLoader

            WebappClassLoader classLoader = (WebappClassLoader) loader.getClassLoader();
            System.out.println("Resources' docBase: " + ((ProxyDirContext) classLoader.getResources()).getDocBase());
            String[] repositories = classLoader.findRepositories();
            for (int i = 0; i < repositories.length; i++) {
                System.out.println("  repository: " + repositories[i]);
            }

            // make the application wait until we press a key.
            System.in.read();
            System.out.println("Shutdowning...");
            ((Lifecycle) context).stop();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
- 创建`ModernServlet`和`PrimitiveServlet`并将class文件输出到`myApp/WEB-INF/classes文件夹下`
- 启动后，控制台打印信息如下. 其中包含首先访问`http://localhost:8080/Modern`,然后控制台输入`stop`停止服务器
```verilog
HttpConnector Opening server socket on all host IP addresses
HttpConnector[8080] Starting background thread
WebappLoader[/myApp]: Deploying class repositories to work directory /home/leithda/source/github/code_warehouse/java/started/how-tomcat-works/work/_/_/myApp
WebappLoader[/myApp]: Deploy class files /WEB-INF/classes to /home/leithda/source/github/code_warehouse/java/started/how-tomcat-works/myApp/WEB-INF/classes
Starting Wrapper Primitive
Starting Wrapper Modern
StandardManager[/myApp]: Seeding random number generator class java.security.SecureRandom
StandardManager[/myApp]: Seeding of random number generator has been completed
Resources' docBase: /home/leithda/source/github/code_warehouse/java/started/how-tomcat-works/myApp
  repository: /WEB-INF/classes/
ModernServlet -- init
stop
Stopping wrapper Primitive
Stopping wrapper Modern
```

### 启动流程简介
#### 启动流程
- 连接器初始化
1. `#connector.initialize`方法，开启`ServerSocket`并赋值给`connector.serverSocket`
2. `#connector.start`方法，启动`HttpConnector`线程，并且创建最小数量`HttpProcessor`
3. `#context.start`方法，启动容器，启动子容器，进行生命周期事件声明

#### 处理请求(建议debug模式走一遍)
1. 启动`HttpConnector`线程时，监听`socket`请求，如果收到请求，实例化一个`HttpProcessor`实例，并将`socket`注册到`HttpProcessor`中。
2. `HttpProcessor`创建时，会开启`HttpProcessor`线程，`#run`方法中，如果`socket`不为空，会调用`#process`方法进行处理，
    解析请求，生成`request`和`response`对象并调用`#Container.invoke`方法
3. `#Container.invoke`方法会调用`SimplePipeline`对象的`#invoke`方法
4. `#SimplePipeline.invoke`会调用内部类`StandardPipelineValveContext`的`#invokeNext`方法，它会依次调用一般处理器，最后调用基本处理器`StandardContextValve`
5. `StandardContextValve`基本处理器的`#invoke`方法内部会根据请求信息找到对应的`Wrapper`容器，并调用`Wrapper`的`#invoke`方法
6. `SimpleWrapper`的`#invoke`方法也调用`SimplePipeline`的`#invoke`方法，`SimpleWrapper`构造方法中设置`SimplePepeline`的基本处理器为`SimpleWrapperValve`
所以执行基本处理器的`SimpleWrapperValve`的`invoke`方法
7. `SimpleWrapperValue`中的`#invoke`方法会调用`#SimpleWrapper.allocate`,进而调用`#loadServlet`方法
8. `#SimpleWrapper.loadServlet`方法中,首先获取`WebappLoader`，然后通过loader获取`WebappClassLoader`，然后调用`#WebappClassLoader`的`#loadClass`方法加载`Serlvet`
9. 最后调用`#Servlet.service`方法进行`Servlet`处理

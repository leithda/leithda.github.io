---
title: Tomcat-Digester
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 2030302893
date: 2019-11-10 08:00:00
---

未避免Tomcat启动代码硬编码，Tomcat使用了`server.xml`来灵活配置Tomcat组件，例如上下文可以进行如下配置：
```xml
# To set the path and docBase properties you use attributes in the XML
# element:
<context docBase="myApp" path="/myApp"/>
```
Tomcat 使用开源工具 [Digester](http://commons.apache.org/proper/commons-digester/) 来讲 XML 元素转换为 Java 对象
<!-- More -->

## Digester
Digester 是 Apache Jakarta 项目下面的开源项目。可以在[Digester](http://jakarta.apache.org/commons/digester/)下载。Digester
由三个包组成，被包装到 commons-digeser.jar 文件中。
- org.apache.commons.digester.提供了基于规则（rules-based）的任意 XML 文档处理。
- org.apache.commons.digester.rss.演示了如何使用Digester 解析 XML 文档，跟很多地方使用的 rich site summary 格式比较。
- org.apache.commons.digester.xmlrules.该包提供了给Digester 提供基于 XML 文档的规则

## Tomcat中的Digester
- `org.apache.commons.digester.Digester`类是`Digester`库里的主类。

### 获取模式
- 定义xml文件如下:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<employee firstName="Brian" lastName="May">
    <office>
        <address streeName="Wellington Street" streetNumber="110"/>
    </office>
</employee>
```
- 该文档的根元素是`employee`，该元素的模式为`employee`.`office`是子元素，对应的模式为`employee/office`.同理`address`的模式是`employee/office/address`

### 获取规则
- 规则定义了Digester遇到特别的模式必须做的事情。一个规则用`org.apache.commons.digester.Rule`表示，`Rule`中有两个方法:`#begin()`和`#end()`

### 解析文档
- 解析获取模式章节的示例文档流程如下:
{% mermaid sequenceDiagram %}
participant a as Start | End
participant b as employee
participant c as office
participant d as address
a->>b: 遇到employee元素
Note right of b: employee.Rule?<br>存在:Rule.start()
b->>c: 遇到office元素
Note right of c: office.Rule?<br>ture:Rule.start()
c->>d: 遇到address元素
Note right of d: address.Rule?<br>true:Rule.start()
d->>d: address元素结束
Note right of d: address.Rule.end()
d->>c: office结束
Note right of c: office.Rule.end()
c->>b: employee结束
Note right of b: employee.Rule.end()
b->>a: 结束
{% endmermaid %}

### 创建对象
- 创建对象调用`#addObjectCreate()`方法，该方法有四个实现，其中两个最常用的方法签名如下：
```java
public void addObjectCreate(java.lang.String pattern, java.lang.Class clazz)
public void addObjectCreate(java.lang.String pattern, java.lang.String className)
```
- 举个栗子
```java
    digester.addObjectCreate("employee",chapter15.digestertest.Employee.class);
    digester.addObjectCreate("employee", "chapter15.digestertest.Employee");
```

- 另外两个实现如下，它使得类名可以在运行时指定
```java
public void addObjectCreate(java.lang.String pattern, java.lang.String className, java.lang.String attributeName)
public void addObjectCreate(java.lang.String pattern, java.lang.String attributeName, java.lang.Class clazz)
```

### 设置属性
- Digester 对象可以通过`#addSetProperties()`设置对象属性

```java
digester.addObjectCreate("employee","chapter15.digestertest.Employee"); // 创建对象
digester.addSetProperties("employee");  // 设置属性
/*
设置属性：
<employee firstName="Brian" lastName="May">
解析xml中属性到类Employee中
如果Employee类中没有 firstName 和 lastName 属性，会报错
*/
```

### 方法调用
- Digester 允许添加规则，遇到相应的模式调用方法栈中最高层对象的方法，方法为`#addCallMethod`，它的一个实现的签名如下：
```java
public void addCallMethod (java.lang.String pattern, java.lang.String methodName)
```

### 建立对象间的联系
- 通过`#addSetNext()`方法建立两个对象间的联系，当调用`#addObjectCreate()`方法创建两个对象时，它将第二个对象作为参数传递给第一个对象。
```java
/**
 * [addSetNext 建立对象间联系]
 *
 * @param pattern    [识别模式]
 * @param methodName [调用方法]
 */
public void addSetNext(java.lang.String pattern, java.lang.String methodName)
```
- 举个栗子
    ```java
    digester.addObjectCreate("employee", "chapter15.digestertest.Employee");
    digester.addObjectCreate("employee/office", "chapter15.digestertest.Office");

    digester.addSetNext("employee/office", "addOffice");    //其中 addOffice 是 Employee 类的方法,该方法必须接受一个 Office 对象作为参数
    ```

### 验证xml文档
- `validating`属性决定`Digester`是否校验xml文档合法性。通过它的setter进行设置


## 测试
### 测试1
- xml文档
```xml
<!-- employee1.xml -->
<?xml version="1.0" encoding="ISO-8859-1"?>
<employee firstName="Brian" lastName="May">
</employee>
```
- 相关类
```java
public class Employee {
    private String firstName;
    private String lastName;
    private ArrayList offices = new ArrayList();

    public Employee() {
        System.out.println("Creating Employee");
    }
    public String getFirstName() {
        return firstName;
    }
    public void setFirstName(String firstName) {
        System.out.println("Setting firstName : " + firstName);
        this.firstName = firstName;
    }
    public String getLastName() {
        return lastName;
    }
    public void setLastName(String lastName) {
        System.out.println("Setting lastName : " + lastName);
        this.lastName = lastName;
    }
    public void addOffice(Office office) {
        System.out.println("Adding Office to this employee");
        offices.add(office);
    }
    public ArrayList getOffices() {
        return offices;
    }
    public void printName() {
        System.out.println("My name is " + firstName + " " + lastName);
    }
}
```

- 测试方法
```java
    @Test
    public void test1(){
        String path = System.getProperty("user.dir") + File.separator  + "etc";
        File file = new File(path, "employee1.xml");
        Digester digester = new Digester();
        // add rules
        digester.addObjectCreate("employee", "cn.leithda.htw.chapter15.digestertest.Employee");
        digester.addSetProperties("employee");
        digester.addCallMethod("employee", "printName");

        try {
            Employee employee = (Employee) digester.parse(file);
            System.out.println("First name : " + employee.getFirstName());
            System.out.println("Last name : " + employee.getLastName());
        }
        catch(Exception e) {
            e.printStackTrace();
        }
    }
```

### 测试2
- xml文档
```xml
<!-- employee2.xml -->

<?xml version="1.0" encoding="ISO-8859-1"?>
<employee firstName="Freddie" lastName="Mercury">
    <office description="Headquarters">
        <address streetName="Wellington Avenue" streetNumber="223"/>
    </office>
    <office description="Client site">
        <address streetName="Downing Street" streetNumber="10"/>
    </office>
</employee>
```
- 相关类
```java
public class Office {
    private Address address;
    private String description;
    public Office() {
        System.out.println("..Creating Office");
    }
    public String getDescription() {
        return description;
    }
    public void setDescription(String description) {
        System.out.println("..Setting office description : " + description);
        this.description = description;
    }
    public Address getAddress() {
        return address;
    }
    public void setAddress(Address address) {
        System.out.println("..Setting office address : " + address);
        this.address = address;
    }
}

public class Address {
    private String streetName;
    private String streetNumber;
    public Address() {
        System.out.println("....Creating Address");
    }
    public String getStreetName() {
        return streetName;
    }
    public void setStreetName(String streetName) {
        System.out.println("....Setting streetName : " + streetName);
        this.streetName = streetName;
    }
    public String getStreetNumber() {
        return streetNumber;
    }
    public void setStreetNumber(String streetNumber) {
        System.out.println("....Setting streetNumber : " + streetNumber);
        this.streetNumber = streetNumber;
    }
    public String toString() {
        return "...." + streetNumber + " " + streetName;
    }
}
```

- 测试方法
```java
    @Test
    public void test2(){
        String path = System.getProperty("user.dir") + File.separator  + "etc";
        File file = new File(path, "employee2.xml");
        Digester digester = new Digester();
        // add rules
        digester.addObjectCreate("employee", "cn.leithda.htw.chapter15.digestertest.Employee");
        digester.addSetProperties("employee");
        digester.addObjectCreate("employee/office", "cn.leithda.htw.chapter15.digestertest.Office");
        digester.addSetProperties("employee/office");
        digester.addSetNext("employee/office", "addOffice");
        digester.addObjectCreate("employee/office/address", "cn.leithda.htw.chapter15.digestertest.Address");
        digester.addSetProperties("employee/office/address");
        digester.addSetNext("employee/office/address", "setAddress");
        try {
            Employee employee = (Employee) digester.parse(file);
            ArrayList offices = employee.getOffices();
            Iterator iterator = offices.iterator();
            System.out.println("-------------------------------------------------");
            while (iterator.hasNext()) {
                Office office = (Office) iterator.next();
                Address address = office.getAddress();
                System.out.println(office.getDescription());
                System.out.println("Address : " +
                        address.getStreetNumber() + " " + address.getStreetName());
                System.out.println("--------------------------------");
            }

        }
        catch(Exception e) {
            e.printStackTrace();
        }
    }
```



## Rule
- `Rule`类有多个方法，最重要的两个方法是`#begin()`和`#end()`,方法签名如下:
    ```java
    public void begin(org.xml.sax.Attributes attributes) throws java.lang.Exception
    public void end() throws java.lang.Exception
    ```
- `Digester`调用`addObjectCreate`、`addCallMethod`、`addSetNext`时，都会调用`#addRule()`方法
    ```java
    public void addObjectCreate(String pattern, String className) {
        this.addRule(pattern, new ObjectCreateRule(className));
    }

    public void addObjectCreate(String pattern, String className, String attributeName) {
        this.addRule(pattern, new ObjectCreateRule(className, attributeName));
    }


    public void addRule(String pattern, Rule rule) {
        rule.setDigester(this);
        this.getRules().add(pattern, rule);
    }
    ```
    - `ObjectCreateRule`是`Rule`的子类，关于`#begin()`方法和`#end()`方法的实现
    ```java
    public void begin(Attributes attributes) throws Exception {
        String realClassName = this.className;
        if (this.attributeName != null) {
            String value = attributes.getValue(this.attributeName);
            if (value != null) {
                realClassName = value;
            }
        }

        if (this.digester.log.isDebugEnabled()) {
            this.digester.log.debug("[ObjectCreateRule]{" + this.digester.match + "}New " + realClassName);
        }

        Class clazz = this.digester.getClassLoader().loadClass(realClassName);
        Object instance = clazz.newInstance();
        this.digester.push(instance);
    }

    public void end() throws Exception {
        Object top = this.digester.pop();
        if (this.digester.log.isDebugEnabled()) {
            this.digester.log.debug("[ObjectCreateRule]{" + this.digester.match + "} Pop " + top.getClass().getName());
        }

    }
    ```
    - 在 begin 方法中的最后三行创建了对象并将其压到 Digester 类的内部堆栈中, end 方法使用 pop 方法从堆栈中获得对象。
    - Rule 类的其它子类工作方式类似

### 测试3
- xml文档
```xml
<!-- employee1.xml -->
<?xml version="1.0" encoding="ISO-8859-1"?>
<employee firstName="Brian" lastName="May">
</employee>
```
- 相关类
```java
public class EmployeeRuleSet extends RuleSetBase {
    public void addRuleInstances(Digester digester) {
        // add rules
        digester.addObjectCreate("employee", "cn.leithda.htw.chapter15.digestertest.Employee");
        digester.addSetProperties("employee");
        digester.addObjectCreate("employee/office", "cn.leithda.htw.chapter15.digestertest.Office");
        digester.addSetProperties("employee/office");
        digester.addSetNext("employee/office", "addOffice");
        digester.addObjectCreate("employee/office/address",
                "cn.leithda.htw.chapter15.digestertest.Address");
        digester.addSetProperties("employee/office/address");
        digester.addSetNext("employee/office/address", "setAddress");
    }
}
```

- 测试方法
```java
    @Test
    public void test3(){
        String path = System.getProperty("user.dir") + File.separator  + "etc";
        File file = new File(path, "employee2.xml");
        Digester digester = new Digester();
        digester.addRuleSet(new EmployeeRuleSet());
        try {
            Employee employee = (Employee) digester.parse(file);
            ArrayList offices = employee.getOffices();
            Iterator iterator = offices.iterator();
            System.out.println("-------------------------------------------------");
            while (iterator.hasNext()) {
                Office office = (Office) iterator.next();
                Address address = office.getAddress();
                System.out.println(office.getDescription());
                System.out.println("Address : " +
                        address.getStreetNumber() + " " + address.getStreetName());
                System.out.println("--------------------------------");
            }

        }
        catch(Exception e) {
            e.printStackTrace();
        }
    }
```

## ContextConfig
- ContextConfig是一个监听器,在一个实际的`Tomcat`部署中，`StandardContext`的标准监听器是`org.apache.catalina.startup.ContextConfig`类的实例。它会给上下文安装一个证书阀门，更重要的是它会读取并解析`web.xml`文件。将其中的xml元素转化为Java对象。默认的`web.xml`定义了默认的`Servlet`映射，并映射了`MIME`类型的文件扩展，定义了`Session`的实效时间，欢迎文件列表等。
- 在启动类中，必须初始化`ContextConfig`并添加到上下文对象中
```java
LifecycleListener listener = new ContextConfig();
((Lifecycle) context).addLifecycleListener(listener);
```
- `StandardContext`每次触发事件时都会调用`#ContextConfig.lifecycleEvent()`,代码如下:
```java
    // org.apache.catalina.startup.ContextConfig

    public void lifecycleEvent(LifecycleEvent event) {
        try {
            this.context = (Context)event.getLifecycle();
            if (this.context instanceof StandardContext) {
                int contextDebug = ((StandardContext)this.context).getDebug();
                if (contextDebug > this.debug) {
                    this.debug = contextDebug;
                }
            }
        } catch (ClassCastException var3) {
            this.log(sm.getString("contextConfig.cce", event.getLifecycle()), var3);
            return;
        }

        if (event.getType().equals("start")) {
            this.start();   // <1>
        } else if (event.getType().equals("stop")) {
            this.stop();    // <2>
        }

    }
```
- `<1>`处，当事件为`START_EVENT`(即start)时，调用自身的`#start()`方法。代码如下：
    ```java
        private synchronized void start() {
        if (this.debug > 0) {
            this.log(sm.getString("contextConfig.start"));
        }

        this.context.setConfigured(false);
        this.ok = true;
        Container container = this.context.getParent();
        if (!this.context.getOverride()) {
            if (container instanceof Host) {
                ((Host)container).importDefaultContext(this.context);
                container = container.getParent();
            }

            if (container instanceof Engine) {
                ((Engine)container).importDefaultContext(this.context);
            }
        }

        this.defaultConfig();   // <1>
        this.applicationConfig();   // <2> 
        if (this.ok) {
            this.validateSecurityRoles();
        }

        if (this.ok) {
            try {
                this.tldScan();
            } catch (Exception var5) {
                this.log(var5.getMessage(), var5);
                this.ok = false;
            }
        }

        if (this.ok) {
            this.certificatesConfig();
        }

        if (this.ok) {
            this.authenticatorConfig();
        }

        if (this.debug >= 1 && this.context instanceof ContainerBase) {
            this.log("Pipline Configuration:");
            Pipeline pipeline = ((ContainerBase)this.context).getPipeline();
            Valve[] valves = null;
            if (pipeline != null) {
                valves = pipeline.getValves();
            }

            if (valves != null) {
                for(int i = 0; i < valves.length; ++i) {
                    this.log("  " + valves[i].getInfo());
                }
            }

            this.log("======================");
        }

        if (this.ok) {
            this.context.setConfigured(true);
        } else {
            this.log(sm.getString("contextConfig.unavailable"));
            this.context.setConfigured(false);
        }

    }
    ```
    - `<1>`处，读取并解析`%CATALINA_HOME%/conf`下的`web.xml`
    - `<2>`处，读取并解析应用目录下`/WEB-INF/web.xml`文件

### defaultConfig
```java
private void defaultConfig() {
    File file = new File("conf/web.xml");
    if (!file.isAbsolute()) {
        file = new File(System.getProperty("catalina.base"), "conf/web.xml");
    }

    FileInputStream stream = null;

    try {
        stream = new FileInputStream(file.getCanonicalPath());
        stream.close();
        stream = null;
    } catch (FileNotFoundException var25) {
        this.log(sm.getString("contextConfig.defaultMissing"));
        return;
    } catch (IOException var26) {
        this.log(sm.getString("contextConfig.defaultMissing"), var26);
        return;
    }

    Digester var3 = webDigester;
    synchronized(var3) {    // <1>
        try {
            InputSource is = new InputSource("file://" + file.getAbsolutePath());
            stream = new FileInputStream(file);
            is.setByteStream(stream);
            webDigester.setDebug(this.getDebug());
            if (this.context instanceof StandardContext) {
                ((StandardContext)this.context).setReplaceWelcomeFiles(true);
            }

            webDigester.clear();
            webDigester.push(this.context);
            webDigester.parse(is);
        } catch (SAXParseException var21) {
            this.log(sm.getString("contextConfig.defaultParse"), var21);
            this.log(sm.getString("contextConfig.defaultPosition", "" + var21.getLineNumber(), "" + var21.getColumnNumber()));
            this.ok = false;
        } catch (Exception var22) {
            this.log(sm.getString("contextConfig.defaultParse"), var22);
            this.ok = false;
        } finally {
            try {
                if (stream != null) {
                    stream.close();
                }
            } catch (IOException var20) {
                this.log(sm.getString("contextConfig.defaultClose"), var20);
            }

        }

    }
}
```
- `<1>`处，加锁解析`web.xml`文件

### applicationConfig
```java
    private void applicationConfig() {
        InputStream stream = null;
        ServletContext servletContext = this.context.getServletContext();
        if (servletContext != null) {
            stream = servletContext.getResourceAsStream("/WEB-INF/web.xml");
        }

        if (stream == null) {
            this.log(sm.getString("contextConfig.applicationMissing"));
        } else {
            Digester var3 = webDigester;
            synchronized(var3) {
                try {
                    URL url = servletContext.getResource("/WEB-INF/web.xml");
                    InputSource is = new InputSource(url.toExternalForm());
                    is.setByteStream(stream);
                    webDigester.setDebug(this.getDebug());
                    if (this.context instanceof StandardContext) {
                        ((StandardContext)this.context).setReplaceWelcomeFiles(true);
                    }

                    webDigester.clear();
                    webDigester.push(this.context);
                    webDigester.parse(is);
                } catch (SAXParseException var19) {
                    this.log(sm.getString("contextConfig.applicationParse"), var19);
                    this.log(sm.getString("contextConfig.applicationPosition", "" + var19.getLineNumber(), "" + var19.getColumnNumber()));
                    this.ok = false;
                } catch (Exception var20) {
                    this.log(sm.getString("contextConfig.applicationParse"), var20);
                    this.ok = false;
                } finally {
                    try {
                        if (stream != null) {
                            stream.close();
                        }
                    } catch (IOException var18) {
                        this.log(sm.getString("contextConfig.applicationClose"), var18);
                    }

                }

            }
        }
    }
```

### 创建`WebDigester`
- 在`ContextConfig`中，存在一个`WebDigester`类.`private static Digester webDigester = createWebDigester();`，代码如下:
```java
    private static Digester createWebDigester() {
        URL url = null;
        Digester webDigester = new Digester();
        webDigester.setValidating(true);
        url = (class$org$apache$catalina$startup$ContextConfig == null ? (class$org$apache$catalina$startup$ContextConfig = class$("org.apache.catalina.startup.ContextConfig")) : class$org$apache$catalina$startup$ContextConfig).getResource("/javax/servlet/resources/web-app_2_2.dtd");
        webDigester.register("-//Sun Microsystems, Inc.//DTD Web Application 2.2//EN", url.toString());
        url = (class$org$apache$catalina$startup$ContextConfig == null ? (class$org$apache$catalina$startup$ContextConfig = class$("org.apache.catalina.startup.ContextConfig")) : class$org$apache$catalina$startup$ContextConfig).getResource("/javax/servlet/resources/web-app_2_3.dtd");
        webDigester.register("-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN", url.toString());
        webDigester.addRuleSet(new WebRuleSet());   // <1>
        return webDigester;
    }
```
- 熟读前面介绍应该知道`WebRuleSet`是`Rule`的子类，并且知道它的工作方式。代码如下:
```java
public class WebRuleSet extends RuleSetBase {
    protected String prefix;

    public WebRuleSet() {
        this("");
    }

    public WebRuleSet(String prefix) {
        this.prefix = null;
        this.namespaceURI = null;
        this.prefix = prefix;
    }

    public void addRuleInstances(Digester digester) {
        digester.addRule(this.prefix + "web-app", new SetPublicIdRule(digester, "setPublicId"));
        digester.addCallMethod(this.prefix + "web-app/context-param", "addParameter", 2);
        digester.addCallParam(this.prefix + "web-app/context-param/param-name", 0);
        digester.addCallParam(this.prefix + "web-app/context-param/param-value", 1);
        digester.addCallMethod(this.prefix + "web-app/display-name", "setDisplayName", 0);
        digester.addRule(this.prefix + "web-app/distributable", new SetDistributableRule(digester));
        digester.addObjectCreate(this.prefix + "web-app/ejb-local-ref", "org.apache.catalina.deploy.ContextLocalEjb");
        digester.addSetNext(this.prefix + "web-app/ejb-local-ref", "addLocalEjb", "org.apache.catalina.deploy.ContextLocalEjb");
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/description", "setDescription", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/ejb-link", "setLink", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/ejb-ref-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/ejb-ref-type", "setType", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/local", "setLocal", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-local-ref/local-home", "setHome", 0);
        digester.addObjectCreate(this.prefix + "web-app/ejb-ref", "org.apache.catalina.deploy.ContextEjb");
        digester.addSetNext(this.prefix + "web-app/ejb-ref", "addEjb", "org.apache.catalina.deploy.ContextEjb");
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/description", "setDescription", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/ejb-link", "setLink", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/ejb-ref-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/ejb-ref-type", "setType", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/home", "setHome", 0);
        digester.addCallMethod(this.prefix + "web-app/ejb-ref/remote", "setRemote", 0);
        digester.addObjectCreate(this.prefix + "web-app/env-entry", "org.apache.catalina.deploy.ContextEnvironment");
        digester.addSetNext(this.prefix + "web-app/env-entry", "addEnvironment", "org.apache.catalina.deploy.ContextEnvironment");
        digester.addCallMethod(this.prefix + "web-app/env-entry/description", "setDescription", 0);
        digester.addCallMethod(this.prefix + "web-app/env-entry/env-entry-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/env-entry/env-entry-type", "setType", 0);
        digester.addCallMethod(this.prefix + "web-app/env-entry/env-entry-value", "setValue", 0);
        digester.addObjectCreate(this.prefix + "web-app/error-page", "org.apache.catalina.deploy.ErrorPage");
        digester.addSetNext(this.prefix + "web-app/error-page", "addErrorPage", "org.apache.catalina.deploy.ErrorPage");
        digester.addCallMethod(this.prefix + "web-app/error-page/error-code", "setErrorCode", 0);
        digester.addCallMethod(this.prefix + "web-app/error-page/exception-type", "setExceptionType", 0);
        digester.addCallMethod(this.prefix + "web-app/error-page/location", "setLocation", 0);
        digester.addObjectCreate(this.prefix + "web-app/filter", "org.apache.catalina.deploy.FilterDef");
        digester.addSetNext(this.prefix + "web-app/filter", "addFilterDef", "org.apache.catalina.deploy.FilterDef");
        digester.addCallMethod(this.prefix + "web-app/filter/description", "setDescription", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/display-name", "setDisplayName", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/filter-class", "setFilterClass", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/filter-name", "setFilterName", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/large-icon", "setLargeIcon", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/small-icon", "setSmallIcon", 0);
        digester.addCallMethod(this.prefix + "web-app/filter/init-param", "addInitParameter", 2);
        digester.addCallParam(this.prefix + "web-app/filter/init-param/param-name", 0);
        digester.addCallParam(this.prefix + "web-app/filter/init-param/param-value", 1);
        digester.addObjectCreate(this.prefix + "web-app/filter-mapping", "org.apache.catalina.deploy.FilterMap");
        digester.addSetNext(this.prefix + "web-app/filter-mapping", "addFilterMap", "org.apache.catalina.deploy.FilterMap");
        digester.addCallMethod(this.prefix + "web-app/filter-mapping/filter-name", "setFilterName", 0);
        digester.addCallMethod(this.prefix + "web-app/filter-mapping/servlet-name", "setServletName", 0);
        digester.addCallMethod(this.prefix + "web-app/filter-mapping/url-pattern", "setURLPattern", 0);
        digester.addCallMethod(this.prefix + "web-app/listener/listener-class", "addApplicationListener", 0);
        digester.addObjectCreate(this.prefix + "web-app/login-config", "org.apache.catalina.deploy.LoginConfig");
        digester.addSetNext(this.prefix + "web-app/login-config", "setLoginConfig", "org.apache.catalina.deploy.LoginConfig");
        digester.addCallMethod(this.prefix + "web-app/login-config/auth-method", "setAuthMethod", 0);
        digester.addCallMethod(this.prefix + "web-app/login-config/realm-name", "setRealmName", 0);
        digester.addCallMethod(this.prefix + "web-app/login-config/form-login-config/form-error-page", "setErrorPage", 0);
        digester.addCallMethod(this.prefix + "web-app/login-config/form-login-config/form-login-page", "setLoginPage", 0);
        digester.addCallMethod(this.prefix + "web-app/mime-mapping", "addMimeMapping", 2);
        digester.addCallParam(this.prefix + "web-app/mime-mapping/extension", 0);
        digester.addCallParam(this.prefix + "web-app/mime-mapping/mime-type", 1);
        digester.addCallMethod(this.prefix + "web-app/resource-env-ref", "addResourceEnvRef", 2);
        digester.addCallParam(this.prefix + "web-app/resource-env-ref/resource-env-ref-name", 0);
        digester.addCallParam(this.prefix + "web-app/resource-env-ref/resource-env-ref-type", 1);
        digester.addObjectCreate(this.prefix + "web-app/resource-ref", "org.apache.catalina.deploy.ContextResource");
        digester.addSetNext(this.prefix + "web-app/resource-ref", "addResource", "org.apache.catalina.deploy.ContextResource");
        digester.addCallMethod(this.prefix + "web-app/resource-ref/description", "setDescription", 0);
        digester.addCallMethod(this.prefix + "web-app/resource-ref/res-auth", "setAuth", 0);
        digester.addCallMethod(this.prefix + "web-app/resource-ref/res-ref-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/resource-ref/res-sharing-scope", "setScope", 0);
        digester.addCallMethod(this.prefix + "web-app/resource-ref/res-type", "setType", 0);
        digester.addObjectCreate(this.prefix + "web-app/security-constraint", "org.apache.catalina.deploy.SecurityConstraint");
        digester.addSetNext(this.prefix + "web-app/security-constraint", "addConstraint", "org.apache.catalina.deploy.SecurityConstraint");
        digester.addRule(this.prefix + "web-app/security-constraint/auth-constraint", new SetAuthConstraintRule(digester));
        digester.addCallMethod(this.prefix + "web-app/security-constraint/auth-constraint/role-name", "addAuthRole", 0);
        digester.addCallMethod(this.prefix + "web-app/security-constraint/display-name", "setDisplayName", 0);
        digester.addCallMethod(this.prefix + "web-app/security-constraint/user-data-constraint/transport-guarantee", "setUserConstraint", 0);
        digester.addObjectCreate(this.prefix + "web-app/security-constraint/web-resource-collection", "org.apache.catalina.deploy.SecurityCollection");
        digester.addSetNext(this.prefix + "web-app/security-constraint/web-resource-collection", "addCollection", "org.apache.catalina.deploy.SecurityCollection");
        digester.addCallMethod(this.prefix + "web-app/security-constraint/web-resource-collection/http-method", "addMethod", 0);
        digester.addCallMethod(this.prefix + "web-app/security-constraint/web-resource-collection/url-pattern", "addPattern", 0);
        digester.addCallMethod(this.prefix + "web-app/security-constraint/web-resource-collection/web-resource-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/security-role/role-name", "addSecurityRole", 0);
        digester.addRule(this.prefix + "web-app/servlet", new WrapperCreateRule(digester));
        digester.addSetNext(this.prefix + "web-app/servlet", "addChild", "org.apache.catalina.Container");
        digester.addCallMethod(this.prefix + "web-app/servlet/init-param", "addInitParameter", 2);
        digester.addCallParam(this.prefix + "web-app/servlet/init-param/param-name", 0);
        digester.addCallParam(this.prefix + "web-app/servlet/init-param/param-value", 1);
        digester.addCallMethod(this.prefix + "web-app/servlet/jsp-file", "setJspFile", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet/load-on-startup", "setLoadOnStartupString", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet/run-as/role-name", "setRunAs", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet/security-role-ref", "addSecurityReference", 2);
        digester.addCallParam(this.prefix + "web-app/servlet/security-role-ref/role-link", 1);
        digester.addCallParam(this.prefix + "web-app/servlet/security-role-ref/role-name", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet/servlet-class", "setServletClass", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet/servlet-name", "setName", 0);
        digester.addCallMethod(this.prefix + "web-app/servlet-mapping", "addServletMapping", 2);
        digester.addCallParam(this.prefix + "web-app/servlet-mapping/servlet-name", 1);
        digester.addCallParam(this.prefix + "web-app/servlet-mapping/url-pattern", 0);
        digester.addCallMethod(this.prefix + "web-app/session-config/session-timeout", "setSessionTimeout", 1, new Class[]{Integer.TYPE});
        digester.addCallParam(this.prefix + "web-app/session-config/session-timeout", 0);
        digester.addCallMethod(this.prefix + "web-app/taglib", "addTaglib", 2);
        digester.addCallParam(this.prefix + "web-app/taglib/taglib-location", 1);
        digester.addCallParam(this.prefix + "web-app/taglib/taglib-uri", 0);
        digester.addCallMethod(this.prefix + "web-app/welcome-file-list/welcome-file", "addWelcomeFile", 0);
    }
}
```

## 测试
```java
public final class Bootstrap {

    // invoke: http://localhost:8080/app1/Modern or
    // http://localhost:8080/app2/Primitive
    // note that we don't instantiate a Wrapper here,
    // ContextConfig reads the WEB-INF/classes dir and loads all servlets.
    public static void main(String[] args) {
        System.setProperty("catalina.base", System.getProperty("user.dir"));
        Connector connector = new HttpConnector();

        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/app1");
        context.setDocBase("app1");
        LifecycleListener listener = new ContextConfig();
        ((Lifecycle) context).addLifecycleListener(listener);

        Host host = new StandardHost();
        host.addChild(context);
        host.setName("localhost");
        host.setAppBase("webapps");

        Loader loader = new WebappLoader();
        context.setLoader(loader);
        connector.setContainer(host);
        try {
            connector.initialize();
            ((Lifecycle) connector).start();
            ((Lifecycle) host).start();
            Container[] c = context.findChildren();
            int length = c.length;
            for (int i=0; i<length; i++) {
                Container child = c[i];
                System.out.println(child.getName());
            }

            // make the application wait until we press a key.
            System.in.read();
            ((Lifecycle) host).stop();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
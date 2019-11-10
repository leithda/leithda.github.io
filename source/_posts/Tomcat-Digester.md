---
title: Tomcat-Digester
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
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
---
title: Tomcat-一个简单的Servlet容器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 991341650
date: 2019-09-23 19:35:00
---

本章主要介绍了通过静态资源处理器及Servlet处理器分别处理静态资源及简单Servlet的请求信息。
<!-- More -->

## 解析
### Setvlet
Java Servlet 是运行在 Web 服务器或应用服务器上的程序，它是作为来自 Web 浏览器或其他 HTTP 客户端的请求和 HTTP 服务器上的数据库或应用程序之间的中间层
#### 生命周期
1. `init`
生命周期方法:当Servlet第一次被创建对象时执行该方法,该方法在整个生命周期中只执行一次
2. `service`
生命周期方法:对客户端响应的方法,该方法会被执行多次，每次请求该servlet都会执行该方法
3. `destroy`
命周期方法:当Servlet被销毁时执行该方法

#### `Servlet` 创建
1. 实现`Servlet`接口
> Servlet 类提供了五个方法，其中三个生命周期方法和两个普通方法
2. 继承 `GenericServlet` 类
> GenericServlet 是一个抽象类，实现了 Servlet 接口，并且对其中的 init() 和 destroy() 和 service() 提供了默认实现
3. 继承 `HttpServlet` 方法
> HttpServlet 也是一个抽象类，它进一步继承并封装了 GenericServlet，使得使用更加简单方便，由于是扩展了 Http 的内容，所以还需要使用 HttpServletRequest 和 HttpServletResponse，这两个类分别是 ServletRequest 和 ServletResponse 的子类

## 解析
1. HttpServer 启动进行端口监听
2. 创建Request对象进行URI解析，判断是Servlet请求还是静态资源请求
3. 如果是Servlet请求，创建ServletProcessor类通过反射调用自定义Servlet的service方法；否则，创建StaticResourceProcessor进行静态资源的处理

> PS： ServletProcessor，反射生成Servlet进行处理时，传入Req和Res使用包装类，避免在自定义Servlet类中通过转型获取到Servlet和Response对象，造成安全问题。

## 代码
### 修改`Request`请求类

>实现`ServletRequest`，对应方法暂时使用默认实现

```java
package cn.leithda.htw.chapter2;

import javax.servlet.RequestDispatcher;
import javax.servlet.ServletInputStream;
import javax.servlet.ServletRequest;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.util.Enumeration;
import java.util.Locale;
import java.util.Map;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Http 请求类
 */
public class Request implements ServletRequest {
    private InputStream in;
    private String uri;

    public Request(InputStream in) {
        this.in = in;
    }


    String getUri() {
        return uri;
    }

    /**
     * 解析URI方法,取出请求报文中URI部分
     *
     * @param requestString 请求报文字符串e
     * @return URI
     */
    private String parseUri(String requestString) {
        int index1, index2;
        index1 = requestString.indexOf(' ');
        if (index1 != -1) {
            index2 = requestString.indexOf(' ', index1 + 1);
            if (index2 > index1) {
                return requestString.substring(index1+1, index2);
            }
        }
        return null;
    }

    /**
     * 解析方法
     * 读取请求报文,解析出对应的uri赋值到属性中
     */
    void parse() {
        // Read a set of characters from the socket
        StringBuilder request = new StringBuilder(2048);
        int i;
        byte[] buffer = new byte[2048];
        try {
            i = in.read(buffer);
        } catch (Exception e) {
            e.printStackTrace();
            i = -1;
        }

        for (int j = 0; j < i; j++) {
            request.append((char) buffer[j]);
        }
        System.out.println(request.toString());
        uri = parseUri(request.toString());
    }


    // 继承自ServletRequest的方法，使用默认实现，此处省略
    // ...
}
```
- `Response`作同样处理

### 增加`RequestFacade`，请求门面类
```java
package cn.leithda.htw.chapter2;


import javax.servlet.RequestDispatcher;
import javax.servlet.ServletInputStream;
import javax.servlet.ServletRequest;
import java.io.BufferedReader;
import java.util.Enumeration;
import java.util.Locale;
import java.util.Map;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: 请求封装类
 */
public class RequestFacade implements ServletRequest {

    private Request request;

    RequestFacade(Request request) {
        this.request = request;
    }

    /*implements method for ServletRequest*/
    public Object getAttribute(String attribute) {
        return request.getAttribute(attribute);
    }

    // ...
}
```
- `Response`做同样处理

### 创建`ServletProcessor` 处理类
> 用于处理Servlet请求，实例化请求对应的`servlet`类,调用`#servlet#service()`方法进行`servlet`处理。

```java
package cn.leithda.htw.chapter2;

import javax.servlet.Servlet;
import java.io.File;
import java.io.IOException;
import java.net.URL;
import java.net.URLClassLoader;
import java.net.URLStreamHandler;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Servlet 处理类
 */
public class ServletProcessor {

    void process(Request request, Response response) {

        String uri = request.getUri();
        String servletName = uri.substring(uri.lastIndexOf("/") + 1);
        URLClassLoader loader = null;

        try {
            URL[] urls = new URL[1];
            URLStreamHandler streamHandler = null;
            File classPath = new File(Constants.WEB_ROOT);
            String repository = (new URL("file", null, classPath.getCanonicalPath() + File.separator)).toString();
            urls[0] = new URL(null, repository, streamHandler);
            loader = new URLClassLoader(urls);
        } catch (IOException e) {
            System.out.println(e.toString());
        }
        Class myClass = null;
        try {
            assert loader != null;
            // 拼接 包名+类名
            servletName = "cn.leithda.htw.chapter2.servlet."+servletName;
            myClass = loader.loadClass(servletName);
        } catch (ClassNotFoundException e) {
            System.out.println(e.toString());
        }
        /*
         *  使用RequestFacade 封装Request,可以避免在Servlet的service方法中，对输入参数进行向下转型(Request)，从而获取Request
         *  的方法和属性。
         */
        Servlet servlet;
        RequestFacade requestFacade = new RequestFacade(request);
        ResponseFacade responseFacade = new ResponseFacade(response);
        try {
            assert myClass != null;
            servlet = (Servlet) myClass.newInstance();
            servlet.service(requestFacade, responseFacade);
        } catch (Throwable e) {
            System.out.println(e.toString());
        }

    }
}
```

###  创建`StaticResourceProcessor` 处理类
> 静态资源处理类,调用`#response.sendStaticResource()`方法，将静态资源写入响应

```java
package cn.leithda.htw.chapter2;

import java.io.IOException;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: 静态资源处理类
 */
public class StaticResourceProcessor {
    /**
     * 调用 Response 方法实现静态资源处理
     *
     * @param request  请求类
     * @param response 响应类
     */
    void process(Request request, Response response) {
        try {
            response.sendStaticResource();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```



### 简化开发，将常量封装为`Constants`类

```java
package cn.leithda.htw.chapter2;

import java.io.File;
import java.util.Date;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: 常量
 */
public interface Constants {
    /**
     *  WEB_ROOT 静态资源根路径
     */
    String WEB_ROOT = System.getProperty("user.dir") + File.separator + "how-tomcat-works" + File.separator + "src"+File.separator+"main"+File.separator+"resources";
    /**
     *  CLASS_ROOT 二进制文件路径
     */
    String CLASS_ROOT = System.getProperty("user.dir") + File.separator + "how-tomcat-works" + File.separator + "target"+File.separator+"classes";

    /**
     *  应用停止命令
     */
    String SHUTDOWN_COMMAND = "/SHUTDOWN";

    /**
     * Http 正常返回报文头
     */
    String HEADER = "HTTP/1.1 200 OK\r\n "
            + "Content-Type: text/html\r\n"
            + "Date"+new Date()+"\r\n"
            + "\r\n";
}
```

### 测试
- 在浏览器输入
`http://localhost:9091/index.html` 访问静态资源(resource目录下)
`http://localhost:9091/servlet/PrimitiveServlet` 访问`Servlet`方法
`http://localhost:9091/SHUTDOWN` 关闭服务器

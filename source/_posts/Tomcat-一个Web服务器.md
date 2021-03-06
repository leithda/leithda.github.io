---
title: Tomcat-一个Web服务器
categories:
  - 源码
  - Tomcat
tags:
  - 源码
author: 长歌
abbrlink: 1975698977
date: 2019-09-23 18:45:00
---

本章说明 java web 服务器是如何工作的。Web 服务器也成为超文本传输协议(HTTP)服务器，因为它使用 HTTP 来跟客户端进行通信的，这通常是个web 浏览器。一个基于 java 的 web 服务器 使用两个重要的类：java.net.Socket 和 java.net.ServerSocket，并通过 HTTP 消息进行通信。 因此这章就自然是从 HTTP 和这两个类的讨论开始的
<!-- More -->

## 解析
1. 创建HttpServer类进行端口监听
2. 创建Request类，读取请求内容进行请求地址解析
3. 创建Response类，根据请求地址，找到对应资源文件，返回到客户端

## 代码
1. 创建HttpServer类通过socket进行端口监听

```java
package cn.leithda.htw.chapter1;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Date;

public class HttpServer {
    /**
     * web根目录,针对当前项目设置
     */
    public static final String WEB_ROOT = System.getProperty("user.dir") + File.separator 
            + "src"+File.separator+"main"+File.separator+"resources";

    /**
     * 停机命令
     */
    public static final String SHUTDOWN_COMMAND = "/SHUTDOWN";

    /**
     * 响应报文头
     */
    public static final String HEADER = "HTTP/1.1 200 OK\r\n "
            + "Content-Type: text/html\r\n"
            + "Date"+new Date()+"\r\n"
            + "\r\n";

    /**
     * 停机标志
     */
    private boolean shutdown = false;

    public static void main(String[] args) {
        HttpServer httpServer = new HttpServer();
        httpServer.await();
    }

    public void await(){
        ServerSocket serverSocket = null;
        int port = 9090;
        try {
            // 监听 127.0.0.1:9090
            serverSocket = new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
        while(!shutdown){
            Socket socket = null;
            InputStream input = null;
            OutputStream output = null;
            try {
                socket = serverSocket.accept();
                input = socket.getInputStream();
                output = socket.getOutputStream();
                // 创建Http请求类并解析uri
                Request request = new Request(input);
                request.parse();
                // 创建响应类并给出响应
                Response response = new Response(output);
                response.setRequest(request);
                response.sendStaticResource();
                // 关闭socket流
                socket.close();
                // 检查是否包含停机命令并更新停机标志
                shutdown = request.getUri().equals(SHUTDOWN_COMMAND);
            } catch (Exception e) {
                e.printStackTrace();
//                continue; // 后面没语句,此处可以注释
            }
        }
    }
}
```

2. 创建Request类，读取请求内容进行请求地址解析

```java
package cn.leithda.htw.chapter1;

import java.io.IOException;
import java.io.InputStream;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Http 请求类
 */
public class Request {
    // 请求报文模板
//    GET /favicon.ico HTTP/1.1
//    Host: localhost:9090
//    Connection: keep-alive
//    Sec-Fetch-Mode: no-cors
//    User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36
//    Accept: image/webp,image/apng,image/*,*/*;q=0.8
//    Sec-Fetch-Site: same-origin
//    Referer: http://localhost:9090/index.html
//    Accept-Encoding: gzip, deflate, br
//    Accept-Language: zh-CN,zh;q=0.9
    private InputStream input;
    private String uri;

    public Request(InputStream input) {
        this.input = input;
    }

    /**
     * 解析请求
     */
    public void parse() {
        // Read a set of characters from the socket
        StringBuffer request = new StringBuffer(2048);
        int i;
        byte[] buffer = new byte[2048];
        try {
            i = input.read(buffer);
        } catch (IOException e) {
            e.printStackTrace();
            i = -1;
        }
        for (int j = 0; j < i; j++) {
            request.append((char) buffer[j]);
        }
        System.out.println("\n =========== 请求报文 begin==============\n");
        System.out.print(request.toString());
        System.out.println("\n =========== 请求报文 end  ==============\n");
        uri = parseUri(request.toString());
    }

    /**
     * 解析 URI
     *
     * @param requestString 请求字符串
     * @return uri 字符串
     */
    private String parseUri(String requestString) {
        int index1, index2;
        index1 = requestString.indexOf(' ');
        if (index1 != -1) {
            index2 = requestString.indexOf(' ', index1 + 1);
            if (index2 > index1)
                return requestString.substring(index1 + 1, index2);
        }
        return null;
    }

    public String getUri() {
        return uri;
    }
}
```

3. 创建Response类，根据请求地址，找到对应资源文件，返回到客户端

```java
package cn.leithda.htw.chapter1;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.OutputStream;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-9-20
 * Description: Http 响应类
 */
public class Response {
    /*
  HTTP Response = Status-Line
  *(( general-header | response-header | entity-header ) CRLF)
  CRLF
  [ message-body ]
  Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
  */
    private static final int BUFFER_SIZE = 1024;
    private Request request;
    private OutputStream output;
    public Response(OutputStream output) {
        this.output = output;
    }
    public void setRequest(Request request) {
        this.request = request;
    }

    public void sendStaticResource() throws IOException {
        byte[] bytes = new byte[BUFFER_SIZE];
        FileInputStream fis = null;
        try{
//            System.out.println("资源地址"+HttpServer.WEB_ROOT+request.getUri());
            File file = new File(HttpServer.WEB_ROOT,request.getUri());
            if(file.exists()){
                // 写入文件响应头
                output.write(HttpServer.HEADER.getBytes());
                fis = new FileInputStream(file);
                int ch = fis.read(bytes,0,BUFFER_SIZE);
                while(ch!=-1){
                    output.write(bytes);
                    ch = fis.read(bytes,0,BUFFER_SIZE);
                }
            }else{  // 文件没找到
                String errorMessage = "HTTP/1.1 404 File Not Found\r\n" +
                        "Content-Type: text/html\r\n" +
                        "Content-Length: 23\r\n" +
                        "\r\n" +
                        "<h1>File Not Found</h1>";
                output.write(errorMessage.getBytes());
            }
        }catch (Exception e){
            // thrown if cannot instantiate a File object
            System.out.println(e.toString() );
        }finally {
            if(fis!=null){
                fis.close();
            }
        }
    }
}

```


## 在浏览器输入
`http://localhost:9090/index.html` 访问静态资源(resource目录下)
`http://localhost:9090/SHUTDOWN` 关闭服务器

Request字符串：
```http request
GET /index.html HTTP/1.1
Host: localhost:9090
Connection: keep-alive
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6
GET /images/logo.gif HTTP/1.1
Host: localhost:9090
Connection: keep-alive
Accept: image/webp,image/*,*/*;q=0.8
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36
Referer: http://localhost:9090/index.html
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6
```
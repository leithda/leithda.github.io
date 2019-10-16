title: Tomcat-连接器
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-9-24
---

Catalina 中有两个主要的模块：连接器和容器。本章中你将会写一个可以创建更好的处理请求和响应对象的连接器。
<!-- More -->
## 解析
1. 等待 HTTP 请求
2. 为每个请求创建个 HttpProcessor 实例
3. 调用 HttpProcessor 的 process 方法

## 代码
### `HttpConnector`连接器类

```java
public class HttpConnector implements Runnable {

    boolean stoped;
    private String scheme = "http";


    public String getScheme() {
        return scheme;
    }

    public void run() {
        ServerSocket serverSocket = null;
        int port = 9091;
        try {
            serverSocket = new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
        while (!stoped) {
            Socket socket = null;
            try {
                socket = serverSocket.accept();
            } catch (Exception ex) {
                continue;
            }
            HttpProcessor processor = new HttpProcessor(this);
            processor.process(socket);
        }
    }

    public void start() {
        Thread thread = new Thread(this);
        thread.start();
    }
}
```



- 开启线程监听socket，当收到请求时，实例化`HttpProcessor`类，对请求进行处理

### `HttpProcessor` 处理器类
```java
    /**
     * 处理请求
     * @param socket socket
     */
    @SuppressWarnings("all")    // 忽略所有警告
    public void process(Socket socket) {
        SocketInputStream input = null;
        OutputStream output = null;
        try {
            input = new SocketInputStream(socket.getInputStream(), 2048);
            output = socket.getOutputStream();

            request = new HttpRequest(input);   // <1>
            response = new HttpResponse(output); // <2>
            response.setRequest(request);

            response.setHeader("Server", "Pyrmont Servlet Container");

            parseRequest(input, output);    // <3>
            parseHeaders(input);    // <4>
            if (request.getRequestURI().startsWith("/servlet/")) {  // <5>
                ServletProcessor processor = new ServletProcessor();
                processor.process(request, response);
            } else {    // <6>
                StaticResourceProcessor processor = new StaticResourceProcessor();
                processor.process(request, response);
            }
            socket.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
- `<1>`处，解析请求字段，主要包含`uri`、`请求参数`、`session`，代码如下：
    ```java
      /**
     * 解析 Request请求 , 解析【"URI","查询参数 query String","Session"】，以及纠正异常URI
     *
     * @param input  Socket输入流
     * @param output 输出流
     * @throws IOException      IO异常
     * @throws ServletException Servlet异常
     */
    private void parseRequest(SocketInputStream input, OutputStream output)
            throws IOException, ServletException {
        // 解析请求行
        input.readRequestLine(requestLine);
        String method = new String(requestLine.method, 0, requestLine.methodEnd);
        String uri = null;
        String protocol = new String(requestLine.protocol, 0, requestLine.protocolEnd);

        if (method.length() <= 1) {
            throw new ServletException("缺少Http请求方法");
        } else if (requestLine.uriEnd < 1) {
            throw new ServletException("没有请求地址URI");
        }
        // 从请求中解析出查询参数
        int question = requestLine.indexOf("?");
        if (question >= 0) {
            request.setQueryString(new String(requestLine.uri, question + 1, requestLine.uriEnd - question - 1));
            uri = new String(requestLine.uri, 0, question);
        } else {
            request.setQueryString(null);
            uri = new String(requestLine.uri, 0, requestLine.uriEnd);
        }

        // 检查URI路径
        if (!uri.startsWith("/")) {
            int pos = uri.indexOf("://");
            // Parsing out protocol and host name
            if (pos != -1) {
                pos = uri.indexOf('/', pos + 3);
                if (pos == -1) {
                    uri = "";
                } else {
                    uri = uri.substring(pos);
                }
            }
        }

        // 从请求中解析Session ID
        // 可能会存在如下的uri
        // xxxx.jsp;jsessionid=xxxxx;
        // URL重写功能。为了防止一些用户把Cookie禁止而无法使用session而设置的功能。
        // jsessionid后面的一长串就是你服务器上的session的ID号,这样无需cookie也可以使用session.
        String match = ";jsessionid=";
        int semicolon = uri.indexOf(match);
        if (semicolon >= 0) {
            String rest = uri.substring(semicolon + match.length());
            int semicolon2 = rest.indexOf(';');
            if (semicolon2 >= 0) {
                request.setRequestedSessionId(rest.substring(0, semicolon2));
                rest = rest.substring(semicolon2);
            } else {
                request.setRequestedSessionId(rest);
                rest = "";
            }
            request.setRequestedSessionURL(true);
            uri = uri.substring(0, semicolon) + rest;
        } else {
            request.setRequestedSessionId(null);
            request.setRequestedSessionURL(false);
        }
        // 纠正异常url
        String normalizedUri = normalize(uri);
        request.setMethod(method);
        request.setProtocol(protocol);

        if (normalizedUri != null) {
            request.setRequestURI(normalizedUri);
        } else {
            request.setRequestURI(uri);
        }

        if (normalizedUri == null) {
            throw new ServletException("Invalid URI: " + uri + "'");
        }
    }
    ```

- `<2>`处，解析请求头信息，代码如下:
    ```java
        /**
     * 解析头部信息，解析【"cookie", "content-length", "content-type"】这些字段
     *
     * @param input Socket输入流
     * @throws IOException      IO异常
     * @throws ServletException Servlet异常
     */
    private void parseHeaders(SocketInputStream input)
            throws IOException, ServletException {
        while (true) {
            HttpHeader header = new HttpHeader();
            // 这里会读取input的数据到header，每次读取一对 name/value。
            // 如果全部读取完成，nameEnd 和 valueEnd 会被设置为 0。
            input.readHeader(header);
            if (header.nameEnd == 0) {
                if (header.valueEnd == 0) {
                    // 读取完所有数据，返回
                    return;
                } else {
                    throw new ServletException(sm.getString("httpProcessor.parseHeaders.colon"));
                }
            }
            String name = new String(header.name, 0, header.nameEnd);
            String value = new String(header.value, 0, header.valueEnd);
            request.addHeader(name, value);

            // cookie的格式：
            // Cookie: name=value; name2=value2
            if (name.equals("cookie")) {
                Cookie cookie[] = RequestUtil.parseCookieHeader(value); // 调用 RequestUtil中的 parseCookieHeader解析Cookie
                for (int i = 0; i < cookie.length; i++) {
                    if (cookie[i].getName().equals("jsessionid")) {
                        if (!request.isRequestedSessionIdFromCookie()) {
                            // 只有缓存中没有时才会执行
                            request.setRequestedSessionId(cookie[i].getValue());
                            request.setRequestedSessionCookie(true);
                            request.setRequestedSessionURL(false);
                        }
                    }
                    request.addCookie(cookie[i]);
                }
            } else if (name.equals("content-length")) { // content-length
                int n = -1;
                try {
                    n = Integer.parseInt(value);
                } catch (NumberFormatException e) {
                    e.printStackTrace();
                    throw new ServletException(sm.getString("httpProcessor.parseHeaders.contentLength"));
                }
                request.setContentLength(n);

            } else if (name.equals("content-type")) {   // content-type
                request.setContentType(value);
            }
        }
    }
    ```
- `<3>`处，当请求字符串中以`/servlet/`开始时，实例化`ServletProcessor`进行处理
- `<4>`处，实例化`StaticResourceProcessor`进行处理

### `ServletProcessor` Serlvet处理器类

```java
public class ServletProcessor {

    /**
     * 处理 Servlet 请求
     *
     * @param req  http 请求
     * @param resp http 响应
     */
    public void process(HttpRequest req, HttpResponse resp) {
        String uri = req.getRequestURI();
        String servletName = uri.substring(uri.indexOf("/") + 1);
        URLClassLoader loader = null;
        try {
            // 创建一个URL类加载器
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
            // 加载 Servlet 类
            servletName = Constants.BasePackage + "." + servletName;
            myClass = loader.loadClass(servletName);
        } catch (ClassNotFoundException e) {
            System.out.println(e.toString());
        }

        Servlet servlet = null;

        try {
            servlet = (Servlet) myClass.newInstance();
            HttpRequestFacade requestFacade = new HttpRequestFacade(req);
            HttpResponseFacade responseFacade = new HttpResponseFacade(resp);
            // 调用 Servlet 的service方法进行处理
            servlet.service(requestFacade, responseFacade);
            ((HttpResponse) resp).finishResponse();
        } catch (Exception e) {
            System.out.println(e.toString());
        } catch (Throwable e) {
            System.out.println(e.toString());
        }
    }
}
```

- 核心方法`process`，根据请求参数实例化对应`servlet`类，调用`#servlet.service(req,resp)`方法进行处理

### `StaticResourceProcessor` 静态资源处理器类

```java
public class StaticResourceProcessor {
    public void process(HttpRequest request, HttpResponse response) {
        try {
            response.sendStaticResource();
        }
        catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
- 直接调用 `#response.sendStaticResource()`方法，写入响应返回

### `ModernServlet` 自定义`servlet`方法
```java
public class ModernServlet extends HttpServlet {

    public void init(ServletConfig config) {
        System.out.println("ModernServlet -- init");
    }

    public void doGet(HttpServletRequest request,
                      HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/html");
        PrintWriter out = response.getWriter();
        out.println(Constants.RES_HEADER);
        out.println("<html>");
        out.println("<head>");
        out.println("<title>Modern Servlet</title>");
        out.println("</head>");
        out.println("<body>");

        out.println("<h2>Headers</h2");
        Enumeration headers = request.getHeaderNames();
        while (headers.hasMoreElements()) {
            String header = (String) headers.nextElement();
            out.println("<br>" + header + " : " + request.getHeader(header));
        }

        out.println("<br><h2>Method</h2");
        out.println("<br>" + request.getMethod());

        out.println("<br><h2>Parameters</h2");
        Enumeration parameters = request.getParameterNames();
        while (parameters.hasMoreElements()) {
            String parameter = (String) parameters.nextElement();
            out.println("<br>" + parameter + " : " + request.getParameter(parameter));
        }

        out.println("<br><h2>Query String</h2");
        out.println("<br>" + request.getQueryString());

        out.println("<br><h2>Request URI</h2");
        out.println("<br>" + request.getRequestURI());

        out.println("</body>");
        out.println("</html>");

    }
}
```


## 测试
- 在浏览器输入
`http://localhost:9092/index.html` 访问静态资源(resource目录下)
`http://localhost:9092/servlet/ModernServlet` 访问`Servlet`方法
`http://localhost:9092/SHUTDOWN` 关闭服务器

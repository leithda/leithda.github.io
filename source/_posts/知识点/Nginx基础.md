---
title: Nginx基础
category: 知识点
tag: Nginx
abbrlink: 2814237589
date: 2021-07-07 21:30:00
---

{% cq %}

Nginx ("engine x") 是一个高性能的 **HTTP** 和 **反向代理** 服务器，也是一个 IMAP/POP3/SMTP 代理服务器。

{% endcq %}
<!-- more -->


# Nginx基础

## Nginx是什么

Nginx ("engine x") 是一个高性能的 **HTTP** 和 **反向代理** 服务器，也是一个 IMAP/POP3/SMTP 代理服务器。

Nginx的功能：

- Nginx提供基本的HTTP服务，可以作为HTTP代理服务器和反向代理服务器，支持通过缓存加速访问，可以完成简单的负载均衡和容错，支持包过滤功能，支持SSL等
- Nginx提供高级HTTP服务，可以进行自定义配置，支持虚拟主机，支持URL重定向，支持网络监控，支持流媒体传输等
- Nginx可以作为邮件代理服务器，它支持IMAP/POP3代理服务功能，支持内部SMTP代理服务功能。
- 更多可参考官方网站：https://nginx.org/en/


## 安装

> 各系统安装参考[官方文档](https://nginx.org/en/linux_packages.html)进行安装。本文以CentOS为例



- 安装所需工具

```bash
sudo yum install yum-utils
```



- 设置yum软件源信息

```bash
# 创建源文件
sudo touch /etc/yum.repos.d/nginx.repo
# 编辑文件
sudo vi /etc/yum.repos.d/nginx.repo

# 粘贴以下内容并保存
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```



- 如果想要安装主线版本的Nginx，使用如下命令

```bash
sudo yum-config-manager --enable nginx-mainline
```



- 查看Nginx版本

```bash
sudo yum list nginx --showduplicates | sort -r 
```



- 安装Nginx

```bash
# sudo yum install nginx 默认安装最新版本
sudo yum install nginx-1.18.0-2.el7.ngx # 安装指定版本Nginx
```



## 基础命令

> 官方命令介绍页面：[官网链接](http://nginx.org/en/docs/switches.html)

- `-? | -h`，打印命令行帮助文档
- `-c file`，使用指定配置文件启动路径，默认值为`/etc/nginx/nginx.conf`
- `-e file`，使用替代错误日志`*file*`来存储日志而不是默认文件 (1.19.5)。特殊值`stderr`选择标准错误文件
- `-g directives`，配置全局指令，如

```bash
nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw.ncpu`;"
```

- `-p prefix`， 设置 nginx 路径前缀，即保存服务器文件的目录（默认值为`/etc/nginx/`）

- `-q`，测试配置文件期间禁止打印非Error信息
- `-s single`，发送`single`信号到到Nginx主进程，`single`选项如下
  - `stop`，快速关闭程序
  - `quit`，平滑关闭程序
  - `reload`，重载配置文件，新启动一个主进程，平滑的关闭旧的工作进程
  - `reopen`，重新打开日志文件
- `-t`，测试配置文件，Nginx会检查配置文件的语法是否正确，然后尝试打开配置中引入的文件
- `-T`，与 相同`-t`，但另外将配置文件转储到标准输出 (1.9.2)
- `-v`，打印 nginx 版本
- `-V`，打印 nginx 版本、编译器版本和配置参数



## 基础配置

> 首先查看Nginx默认配置文件，如下

```nginx
# 用户
user  nginx; 

# 工作进程数
worker_processes  1; 

# 错误日志路径及日志级别
error_log  /var/log/nginx/error.log warn; 
# pid文件存储路径
pid        /var/run/nginx.pid; 


events {
    # 最大连接数
    worker_connections  1024; 
}


http {
    # 引入Mime类型定义
    include       /etc/nginx/mime.types; 
    # 默认类型
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
	# 访问日志 日志级别
    access_log  /var/log/nginx/access.log  main; 

    # 是否启用sendfile
    sendfile        on;

    # 客户端保持链接最大时间
    keepalive_timeout  65;
	
    # 是否开启压缩功能
    #gzip  on;

    server {
        listen       80;
        server_name  localhost;


        # 设置静态资源
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        # 将服务器错误页面重定向到静态文件50x.html

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
}

```



Nginx配置模块

1. **全局块**：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
2. **events块**：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
3. **http块**：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
4. **server块**：配置虚拟主机的相关参数，一个http中可以有多个server。
5. **location块**：配置请求的路由，以及各种页面的处理情况。





## 基础功能

> 对常见功能做一个总结

### 动静分离

将Nginx作为一个静态资源服务器来处理。演示部分只截取`http`块主要配置，其他配置按照场景自行配置。

```nginx
http {

    # 配置上游服务器
    upstream mall {
        server 192.168.33.10:8000;
        server 192.168.33.11:8000;
    }

    server {
        # 配置监听
        listen       80;
        listen  [::]:80;
        server_name  mall.com;

        # 设置静态资源
        location /static/ {
            root   /usr/share/nginx/html;
        }

        # 动态资源转发
        location / {
            proxy_set_header Host $host; # 转发时携带Host请求头
            proxy_pass http://mall;
        }
    }
}
```

- 访问`mall.com/static`路径下的资源时，会请求`/usr/share/nginx/html`下的对应路径的静态资源
- 访问其他路径时，会转发到上游服务`192.168.33.10\192.168.33.11`的`8000`端口进行动态资源的请求
- 使用上述配置也完成了一个基于名称(mall.com)的虚拟主机的创建，也可以指定IP进行创建基于IP的虚拟主机



### 反向代理

#### 反向代理的作用

- 保障应用服务器的安全，加了一层代理
- 实现负载均衡
- 跨域



#### 配置

```nginx
http {

    # 配置上游服务器
    upstream mall {
        server 192.168.33.10:8000;
        server 192.168.33.11:8000;
    }

    server {
        # 配置监听
        listen       80;
        listen  [::]:80;
        server_name  mall.com;


        # 反向代理
        location / {
            proxy_set_header Host $host; # 转发时携带Host请求头
            proxy_pass http://mall;
        }
    }
}
```





### 负载均衡

#### 轮训

```nginx
upstream mysvr {
    server 192.168.33.10:8091;
    server 192.168.33.11:8091;
}
```



#### 热备

```nginx
upstream mysvr {
    server 192.168.33.10:8091;  # 主节点发生异常时，请求切换到备份节点
    server 192.168.33.11:8091 bak;
}
```



#### 权重

```nginx
upstream mysvr {
    server 192.168.33.10:8091 weight=2;
    server 192.168.33.11:8091 weight=3; # 默认权重为1，轮询即权重都为1情况
}
```



#### 最少连接(可加权)

```nginx
upstream mysvr {
    least_conn;
    server 192.168.33.10:8091;
    server 192.168.33.11:8091;
}
```



#### IP哈希

```nginx
upstream mysvr {
    server 192.168.33.10:8091;
    server 192.168.33.11:8091;
    ip_hash;
}
```



#### 普通hash

```nginx
upstream mysvr {
    hash $request_uri;
    
    server 192.168.33.10:8091;
    server 192.168.33.11:8091;

}
```





#### 负载均衡的其他参数

- down，表示当前的server暂时不参与负载均衡。

- backup，预留的备份机器。当其他所有的非backup机器出现故障或者忙的时候，才会请求backup机器，因此这台机器的压力最轻。

- max_fails，允许请求失败的次数，默认为1。当超过最大次数时，返回proxy_next_upstream 模块定义的错误。

- fail_timeout，在经历了max_fails次失败后，暂停服务的时间。max_fails可以和fail_timeout一起使用。

```nginx
upstream mysvr { 
    server 192.168.0.107:8091 weight=2 max_fails=2 fail_timeout=2;
    server 192.168.0.102:8091 weight=1 max_fails=2 fail_timeout=1;    
}
```



> 一篇内容很难将Nginx里的内容全都覆盖到，最主要的是我现在也不会，后续按需分析Nginx其他部分如缓存、压缩、Rewrite等功能 ~~~

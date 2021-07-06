



# Nginx从入门到放弃

## Nginx是什么

Nginx ("engine x") 是一个高性能的 **HTTP** 和 **反向代理** 服务器，也是一个 IMAP/POP3/SMTP 代理服务器。

解决的问题：

- 高并发

- 负载均衡

- 高可用

- 虚拟主机

- 伪静态

- 动静分离

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
user  nginx; # 系统用户
worker_processes  1; # 工作进程数

error_log  /var/log/nginx/error.log warn; # 错误日志路径及日志级别
pid        /var/run/nginx.pid; # pid文件存储路径


events {
    worker_connections  1024; # 最大连接数
}


http {
    include       /etc/nginx/mime.types; # 
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main; 

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
}

```







## 基础功能




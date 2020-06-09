---
title: Nginx之gzip内容压缩
abbrlink: 2600560154
date: 2020-06-09 23:01:04
categories:
  - Nginx
tags:
  - Nginx
  - 中间件
author: 长歌
---

{% cq %}
Nginx支持通过设置进行指定文件类型的压缩。(此处略水，后续补充)
{% endcq %}


# 什么是GZIP
GZIP是若干文件压缩程序的简称，通常指GNU计划的实现，此处的GZIP代表的就是GUN ZIP，这也是HTTP1.1协议定义的两种压缩方法中最常用的一种压缩方法，客户端浏览器大都支持这种压缩格式。

## nginx 的 gzip设置
> 可以配置在http块，server 块或者location块中设置。 gzip 的设置分为三部分
> - ngx_http_gunzip_module
> - ngx_http_gzip_module
> - ngx_http_gzip_static_module

### [ngx_http_gunzip_module](http://nginx.org/en/docs/http/ngx_http_gunzip_module.html)
> 设置Nginx服务器对不支持gzip的客户端返回解压后的数据，如果客户的浏览器支持压缩还仍然返回压缩的后的数据  
> 
> 默认不编译此模块，通过 `--with-http_gunzip_module`参数进行编译

```conf
location /storage/ {
# on为打开off为关闭
    gunzip on;
# gunzip_buffers number size；
# number为nginx服务器向系统向系统申请缓存空间的个数，size为每个空间的大小，单位
# 为k，默认情况下number * size的大小为128k，其中size 的值取系统内存页一页的大小为4KB或者8KB即可
    gunzip_buffers 32 4k;
}
```

### [ngx_http_gzip_module](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)
> 此模块允许使用`gzip`方法压缩响应，一般情况下可以将数据大小减少一半或更多。


```conf
# 开启压缩
gzip on ;

# 设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。 例如 4 4k 代表以4k为单位，按照原始数据大小以4k为单位的4倍申请内存。 4 8k 代表以8k为单位，按照原始数据大小以8k为单位的4倍申请内存。
# 如果没有设置，默认值是申请跟原始数据相同大小的内存空间去存储gzip压缩结果。
gzip_buffers

#压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间
gzip_comp_level

# IE6及以下禁止压缩
gzip_disable "MSIE [1-6]\."; 

# 值为1.0和1.1 代表是否压缩http协议1.0，选择1.0则1.0和1.1都可以压缩
gzip_http_version 1.0 


# 设置允许压缩的页面最小字节数，页面字节数从header头得content-length中进行获取。
# 默认值是0，不管页面多大都压缩。建议设置成大于2k的字节数，小于2k可能会越压越大。
gzip_min_length 2k;

# 默认值：off
# Nginx作为反向代理的时候启用，开启或者关闭后端服务器返回的结果，匹配的前提是
# 后端服务器必须要返回包含"Via"的 header头。
# 
# off - 关闭所有的代理结果数据的压缩
# expired - 启用压缩，如果header头中包含 "Expires" 头信息
# no-cache - 启用压缩，如果header头中包含 "Cache-Control:no-cache" 头信息
# no-store - 启用压缩，如果header头中包含 "Cache-Control:no-store" 头信息
# private - 启用压缩，如果header头中包含 "Cache-Control:private" 头信息
# no_last_modified - 启用压缩,如果header头中不包含 "Last-Modified" 头信息
# no_etag - 启用压缩 ,如果header头中不包含 "ETag" 头信息
# auth - 启用压缩 , 如果header头中包含 "Authorization" 头信息
# any - 无条件启用压缩
gzip_proxied expired no-cache no-store private auth;

# 默认值: gzip_types text/html (默认不对js/css文件进行压缩)
# 压缩类型，匹配MIME类型进行压缩
# 不能用通配符 text/*
# (无论是否指定)text/html默认已经压缩 
# 设置哪压缩种文本文件可参考 conf/mime.types
gzip_types text/plain application/x-javascript text/css application/xml;  

# 给CDN和代理服务器使用， 用于设置使用gzip时是否在响应的头部增加`vary:Accept-Encoding`告诉客
# 户端内容已在服务端进行了压缩，默认设置为off,还可以使用Nginx配置的add_header指令
# 强制在Nginx服务器的响应头部添加“Vary:Accept-Encoding”也可以实现相同的效果。
gzip_vary on;
```

### [ngx_http_gzip_static_module](http://nginx.org/en/docs/http/ngx_http_gzip_static_module.html)
> 该模块允许发送`.gz`文件给客户端(当支持压缩的客户端请求的数据已经压缩并已`.gz`后缀保存在服务器上)；该模块使用的是静态编码，在http响应头部包含content-length头域来指明报文的长度，用于服务器可以确定响应数据的长度的情况，而ngx_http_gzip_module使用chunked编码动态压缩，主要用于服务器无法确定响应数据长度的情况，比如较大文件的下载等情形，此时就要实时生成数据的长度，用法与ngx_http_gzip_module一样 
> 
> 默认不编译此模块，通过 ` --with-http_gzip_static_module` 参数进行编译

```conf
# on为开启并检查客户端浏览器是否中吃gzip压缩功能，off为关闭，always一直发送gzip压
# 缩文件，而不检查浏览器是否支持gzip压缩
gzip_static off | on | always; 
```

### 注意事项
- 图片，mp3等静态资源，不需要进行压缩。(压缩率较小，浪费CPU资源)
- 比较小的文件不需要压缩

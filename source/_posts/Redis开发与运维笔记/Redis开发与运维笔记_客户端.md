---
title: Redis开发与运维笔记_客户端
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: leithda
abbrlink: 3206834815
date: 2021-05-26 22:00:00

---

{% cq %}
Redis是用单线程来处理多个客户端的访问，因此作为Redis的开发和运维人员需要了解Redis服务端和客户端的通信协议，以及主流编程语言的Redis客户端使用方法，同时还需要了解客户端管理的相应API以及开发运维中可能遇到的问题
{% endcq %}

<!-- more -->

参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)

<hr>


# 客户端

## 客户端通信协议

Redis的通信协议是在TCP协议之上构建的。Redis定制了RESP(Redis Serialization Protocol,Redis序列化协议)实现客户端与服务端的正常交互。

### 发送命令格式

```bash
*<参数数量> CRLF
$<参数1的字节数量> CRLF
<参数1> CRLF
...
$<参数N的字节数量> CRLF
<参数N> CRLF
```

- 以`set hello world`为例，命令为`*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n`

### 返回结果格式

- 状态回复：在RESP中第一个字节是"+"
- 错误回复："-"
- 整数回复：":"
- 字符串回复："$"
- 多条字符串回复："*"

> Redis-cli只能看到最终结果，想要看到Redis的真正返回结果可以使用`telnet`、`nc`、甚至使用Socket程序进行模拟。
> 当结果中包含不存在的值(nil)，返回$-1。


## Java客户端Jedis

### 获取Jedis

使用maven,gradle等将Redis坐标加入依赖中。

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.5.1</version>
</dependency>
```

### Jedis的基本使用方法

```java
Jedis jedis = new Jedis("127.0.0.1", 6379);
jedis.set("hello", "world");
String value = jedis.get("hello");
```

Jedis初始化需要两个参数，除此之外，包含四个参数的构造函数也比较常用。

```java
/**
 * host: 主机地址
 * port: 主机IP
 * connectionTimeout: 客户端连接超时
 * soTimeout: 客户端读写超时
 */
Jedis(final String host, final int port, final int connectionTimeout, final int soTimeout)
```

除了使用传统的String类型，Jedis还提供了基于字节数组的操作。可以使用序列化工具将Java对象进行序列化保存到Redis中，读取后再反序列化为对应对象。

### Jedis连接池的使用方法

连接池的优缺点不再赘述，主要是提高复用性，节省每次进行连接及创建对象的开销。  
Jedis提供了JedisPool这个类作为Jedis的连接池对象，使用Apache的通用池对象工具common-pool作为资源的管理工具。

```java
// common-pool连接池配置，这里使用默认配置，后面小节会介绍具体配置说明 
GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig(); 
// 初始化Jedis连接池
JedisPool jedisPool = new JedisPool(poolConfig, "127.0.0.1", 6379);


Jedis jedis = null;
try{
  jedis = jedisPool.getResource();
  jedis.get("hello");
}catch(Exception e){
  logger.error(e.getMessage(),e);
}finally{
  if(jedis!=null){
    jedis.close();
  }
}
```

### Jedis中Pipeline的使用方法

- 利用jedis对象生成一个pipeline对象，`jedis.pipelined()`
- 使用pipeline对象进行jedis操作，此时的命令不会真正执行
- 使用pipeline.sync()完成此次pipeline对象的调用

```java
public void mdel(List<String> keys){
  Jedis jedis = new Jedis("127.0.0.1", 6379);

  Pipeline pipeline = jedis.pipelined();
  for(String key : keys){
    pipeline.del(key);
  }

  pipeline.sync();
}
```

- 除了`pipeline.sync()`，还可以使用`List<Object> Pipeline#syncAndReturnAll()`获取pipeline的执行结果。

### Jedis的Lua脚本

Jedis中关于Lua脚本的内容与Redis-cli十分类似，主要有以下几个函数：

```java
Object eval(String script, int keyCount, String... params) 
Object evalsha(String sha1, int keyCount, String... params) 
String scriptLoad(String script)
```

## Python客户端redis-py

略过此篇幅；


## 客户端管理

### 客户端API

1. `client list`: 列出与Redis实例连接的客户端信息

```bash
127.0.0.1:6379> client list
id=3 addr=127.0.0.1:54162 fd=8 name= age=4 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 obl=0 oll=0 omem=0 events=r cmd=client
```

- 标识：id,addr,fd,name
  - id：客户端连接唯一标识，自增，重启后重置为0
  - addr：客户端连接的ip和端口
  - fd: socket的文件描述符。`=-1`标识当前客户端不是外部客户端，而是Redis内部的伪装客户端
  - name: 客户端的名字。对应`client setName`和`client getName`
- 输入缓冲区：qbuf、qbuf-free
  - Redis为每个客户端分配了输入缓冲区，临时保存客户端的命令。
  - qbuf和qbuf-free分别代表缓冲区的总容量和剩余容量。Redis没有配置设置缓冲区大小，而是会根据输入内容的大小不同动态调整，缓冲区不能超过1G，超过后客户端将被关闭。
  - 输入缓冲区使用不当可能造成两个问题：
    1. 客户端的输入缓冲区超过1G，客户端将会被关闭
    2. 多个客户端使用过多内存，会使Redis的内存占用超出`maxmemory`，造成数据丢失、键值淘汰、OOM等情况
    3. 解决以上问题的方法：1)可以通过定期执行`client list`命令，收集qbuf和qbuf-free.2)通过info命令的info clients模块，当最大的缓冲区超过一定阈值进行报警。
- 输出缓冲区：obl、oll、omen
  - Redis为每个客户端分配了输出缓冲区，保存命令执行的结果返回给客户端。它可以通过参数`client-output-buffer-limit`来进行设置。

- 客户端的存活状态 age、idle
  - age：客户端已经连接的时间
  - idle：最近一次的空闲时间

- 客户端的限制 maxclients和timeout
  - maxclients:最大客户端连接数，默认是10000，可通过`info clients`查看当前Redis的连接数
  - timeout: 客户端最大空闲时间，为防止客户端没有关闭连接造成连接数过多。开发过程中应注意是否设置此参数，避免造成客户端错误

- 客户端类型：flag
  - flag：当前客户端的类型，S：当前客户端是slave客户端，N：普通客户端，O：当前客户端正在执行monitor命令。

2. `clinet setName`和`client getName`
3. `client kill ip:port`  手动杀掉客户端连接
4. `client pause timeout(毫秒)` 暂停客户端连接

- `client puase`只对普通和发布订阅客户端有效
- `client pause`可以用一种可控的方式将客户端连接从一个Redis节点切换到另一个Redis节点

5. `monitor` 监控Redis正在执行的命令。需要注意的是，高并发场景下，多个客户端执行的命令通过monitor进行监视，可能造成输出缓冲区内存暴涨。


### 客户端相关配置

- timeout: 客户端空闲连接的超时时间
- maxclients: 客户端最大连接数
- tcp-keepalive: 检测TCP连接活性的周期，默认为0，不检测。
- tcp-backlog: TCP三次握手后，会将接受的连接放入队列，参数为此队列大小，默认511。受操作系统影响，一般不需要调整。Linux中如果`/proc/sys/net/core/somaxconn`小于tcp-backlog.那么Redis启动时会看到如下日志

```bash
# WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/ sys/net/core/somaxconn is set to the lower value of 128.
```

### 客户端统计片段 `info clients`

```bash
127.0.0.1:6379> info clients
# Clients
connected_clients:1 # 当前Redis的客户端连接数
client_recent_max_input_buffer:2  # 当前所有输入缓冲区中占用的最大容量
client_recent_max_output_buffer:0 # 当前输出缓冲区队列对象个数的最大值
blocked_clients:0 # 正在执行阻塞命令的客户端个数
```

除此之外`info stats`中也包含两个客户端相关的统计指标.

```bash
127.0.0.1:6379> INFO stats
# Stats
total_connections_received:1  # 客户端连接总数
...
rejected_connections:0 # 拒绝客户端连接数
...
```

## 本章重点回顾

1. RESO保证客户端与服务端的正常通信，是各种编程语言开发的基础
2. 区分Redis直连和连接池的区别，在生产环境中，应该使用连接池
3. Jedis客户端没有内置序列化，需要自己选用
4. 客户端输入缓冲区不能配置，强制限制在1G之内，但是不会受到`maxmemory`限制
5. 客户端输出缓冲区支持普通客户端、发布订阅客户端、复制客户端配置，会受到`maxmemory`限制。
6. Redis的`timeout`配置可以自动关闭闲置客户端，`tcp-keepalive`参数可以周期性检查关闭无效TCP连接
7. `info clients`可以帮助开发运维人员找到客户端可能存在的问题
8. 理解Redis通信原理和建立完善的监控系统对快速定位解决客户端常见问题非常有帮助
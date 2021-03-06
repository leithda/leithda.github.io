---
abbrlink: 1097430507
title: Redis开发与运维笔记-初识Redis
date: 2021-03-01 10:00:00
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: 长歌
---

{% cq %}
本章将带领读者进入Redis的世界，了解它的前世今生、众多特性、典型应用场景、安装配置、如何好用等，最后会对Redis发展过程中的重要版本进行说明
{% endcq %}
<!-- more -->

参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)
<hr>

# 初始Redis

## 盛赞Redis

​  Redis[1]是一种基于键值对(key-value)的NoSQL数据库，与很多键值对 数据库不同的是，Redis中的值可以是由string(字符串)、hash(哈希)、 list(列表)、set(集合)、zset(有序集合)、Bitmaps(位图)、 HyperLogLog、GEO(地理信息定位)等多种数据结构和算法组成，因此 Redis可以满足很多的应用场景，而且因为Redis会将所有数据都存放在内存 中，所以它的读写性能非常惊人。不仅如此，Redis还可以将内存的数据利 用快照和日志的形式保存到硬盘上，这样在发生类似断电或者机器故障的时 候，内存中的数据不会“丢失”。除了上述功能以外，Redis还提供了键过 期、发布订阅、事务、流水线、Lua脚本等附加功能。总之，如果在合适的 场景使用好Redis，它就会像一把瑞士军刀一样所向披靡。

> Redis官网：  [redis.io](http://redis.io)

## Redis特性

1. 速度快

   - 数据基于内存
   - 使用C语言实现，更接近操作系统
   - Redis使用了单线程架构，避免多线程线程间切换开销
   - Redis代码优化

2. 5种数据结构

   String(字符串)、Hash(哈希)、List(列表)、Set(集合)、ZSet(有序集合)

3. 丰富的功能

   - 过期功能：方便缓存的实现
   - 发布/订阅功能：可以实现消息系统
   - 支持Lua脚本，可以利用Lua创造出新的Redis命令
   - 提供了简单的事务
   - 提供了Pipeline(流水线)功能，可以减少网络开销

4. 稳定简单

   源码剪短而不简单

5. 客户端语言多

   提供了多种语言的客户端

6. 持久化

   RDB和AOF

7. 主从复制

   Redis提供了复制功能，实现了多个数据相同的Redis副本

8. 高可用和分布式

   Redis 2.8版本实现了高可用实现`Redis Sentinel`，它能够保证Redis节点的故障发现和故障自动转移。Redis 3.0版本实现了分布式实现`Redis Cluster`，它是Redis真正的分布式实现，提供了高可用、读写和容量的扩展

## Redis的使用场景

### Redis可以做什么

1. 缓存
2. 排行榜
3. 计数器应用
4. 社交网络
5. 消息队列系统

### Redis不可以做什么

1. 数据量过大不适合使用Redis，Redis数据存放在内存中，数据量过大会导致内存资源紧张
2. 常用热数据适合缓存，反之，冷数据如果缓存会造成内存资源的浪费

## 用好Redis的建议

1. 使用者应了解Redis的**单线程模型**，**持久化策略(RDB&AOF)**。
2. 阅读Redis源码，吃透原理并可以根据业务需求对Redis进行定制化开发

## 正确安装并启动Redis

### 安装Redis

```bash
wget http://download.redis.io/releases/redis-3.0.7.tar.gz
tar xzf redis-3.0.7.tar.gz
ln -s redis-3.0.7 redis
cd redis
make
make install
```

- 下载Redis指定版本的源码压缩包到当前目录。 
- 解压缩Redis源码压缩包。 
- 建立一个redis目录的软连接，指向redis-3.0.7。 
- 进入redis目录。 
- 编译(编译之前确保操作系统已经安装gcc)。 
- 安装

> 注： 安装后会将Redis相关运行文件放到`/usr/local/bin`目录下

### 配置、启动、操作、关闭Redis

​  Redis安装后，`/usr/local/bin`目录下会新增几个redis开头的可执行文件，说明如下：

| 可执行文件       | 作用                               |
| ---------------- | ---------------------------------- |
| redis-server     | 启动Redis                          |
| redis-cli        | Redis命令行工具                    |
| redis-benchmark  | Redis基准测试工具                  |
| redis-check-aof  | Redis AOF 持久化文件检测和修复工具 |
| redis-check-dump | Redis RDB 持久化文件检测和修复工具 |
| redis-sentinel   | 启动 Redis Sentinel                |

1. 启动Redis

   - 默认配置，执行`redis-server`

   - 运行启动，`redis-server --configKey1 configValue1 --configKey2 configValue2`

     例如：`redis-server --port 6378`

   - 配置文件启动，`redis-server /opt/redis/redis.conf`

2. Redis命令行客户端

   `redis-cli -h {host} -p {port}`，例如`redis-cli -h 127.0.0.1 -p 6379`

3. 停止Redis服务

   `redis-cli shutdown`

## Redis的重大版本

> Redis借鉴了Linux对于版本号的命名规则，版本号第二位为奇数则表示版本为非稳定版本，且为下一稳定版本的开发版本

1. Redis 2.6
   - 服务端支持Lua脚本。
   - 去掉虚拟内存相关功能。
   - 放开对客户端连接数的硬编码限制。
   - 键的过期时间支持毫秒。
   - 从节点提供只读功能。
   - 两个新的位图命令:bitcount和bitop。
   - 增强了redis-benchmark的功能:支持定制化的压测，CSV输出等功能。
   - 基于浮点数自增命令:incrbyfloat和hincrbyfloat。
   - redis-cli可以使用--eval参数实现Lua脚本执行。
   - shutdown命令增强。
   - info可以按照section输出，并且添加了一些统计项。
   - 重构了大量的核心代码，所有集群相关的代码都去掉了，cluster功 能将会是3.0版本最大的亮点。
   - sort命令优化。
2. Redis 2.8
   - 添加部分主从复制的功能，在一定程度上降低了由于网络问题，造成频繁全量复制生成RDB对系统造成的压力。
   - 尝试性地支持IPv6。
   - 可以通过config set命令设置maxclients。
   - 可以用bind命令绑定多个IP地址。
   - Redis设置了明显的进程名，方便使用ps命令查看系统进程。
   - config rewrite命令可以将config set持久化到Redis配置文件中。
   - 发布订阅添加了pubsub命令。
   - Redis Sentinel第二版，相比于Redis2.6的Redis Sentinel，此版本已经 变成生产可用。

> 可以通过github仓库查看各版本重大改动： [Redis版本](https://github.com/redis/redis/tags)

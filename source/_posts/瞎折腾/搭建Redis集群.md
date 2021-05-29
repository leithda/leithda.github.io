---
title: Redis集群搭建
abbrlink: 778231993
category: 瞎折腾
tag: Redis
date: 2021-05-29 12:50:00
---

{% cq %}

这里只是采用手工方式创建一个Redis集群，后续会补充集群的高级部分内容。

{% endcq %}

<!-- more -->



# 搭建Redis集群



## 安装Redis

> 此处以Linux(CentOS)为例。其他方式大同小异

### 下载

进入[官网](https://redis.io/)下载，或者在服务器上执行一下命令下载Redis，注意版本。

```bash
$ wget https://download.redis.io/releases/redis-6.2.3.tar.gz
$ tar xzf redis-6.2.3.tar.gz
```



### 安装

```bash
$ cd redis-6.2.3
$ make
```

- 其中，如果报错`/bin/sh: cc: command not found`时，执行`sudo yum -y install gcc gcc-c++ libstdc++-devel` 安装gcc。执行`make distclean  && make`继续安装。

- make完成会在src目录下生成`redis-server`等redis相关可执行文件。

- 执行`make install`命令将redis相关命令加入到`/usr/local/bin`目录下



## 准备节点

新建reids配置文件夹

```bash
mkdir -p /home/dev/data/redis
cd /home/dev/data/redis
mkdir conf data
touch redis.conf
```

- redis.conf文件为官网提供配置文件，用于参考。

在`conf`目录下新建配置文件，名称采用`redis-{port}.conf`格式。配置如下：

```properties
#节点端口 
port 6379 
# 开启集群模式 
cluster-enabled yes 
# 节点超时时间，单位毫秒 
cluster-node-timeout 15000 
# 集群内部配置文件 
cluster-config-file "nodes-6379.conf"
# 指定本地数据库文件名，默认值为dump.rdb
dbfilename dump-6379.rdb
# dump 文件保存目录
dir /home/dev/data/redis/data
```

- 配置后的文件目录如下：

  ```bash
  ├── redis
      ├── conf
      │   ├── redis-6379.conf
      │   ├── redis-6380.conf
      │   ├── redis-6381.conf
      │   ├── redis-6382.conf
      │   ├── redis-6383.conf
      │   └── redis-6384.conf
      ├── data
      └── redis.conf
  ```

  

## 启动

进入到`conf`目录依次执行

```bash
redis-server redis-6379.conf &
redis-server redis-6380.conf &
redis-server redis-6381.conf &
redis-server redis-6382.conf &
redis-server redis-6383.conf &
redis-server redis-6384.conf &
```

- 此处可能遇到报错` Server can't set maximum open files to 10032 because of OS error: Operation not permitted.`。使用`ulimit -n`查看系统限制文件句柄数。处理方法如下：

  ```bash
  # vi /etc/security/limits.conf 修改后重启
  * soft nofile 65535
  * hard nofile 65535
  ```



- 连接到集群内节点查看集群状态

  ```bash
  [dev@localhost data]$ redis-cli
  127.0.0.1:6379> cluster info
  cluster_state:fail
  cluster_slots_assigned:0
  cluster_slots_ok:0
  cluster_slots_pfail:0
  cluster_slots_fail:0
  cluster_known_nodes:1
  cluster_size:0
  cluster_current_epoch:0
  cluster_my_epoch:0
  cluster_stats_messages_sent:0
  cluster_stats_messages_received:0
  
  ```



## 节点握手

使用`cluster meet {host} {port}`加入6380、6381后的集群状态。

>  建议新开一个会话用于操作，由于之前启动使用&方式，使用当前会话会因为输出日志影响阅读效果。

```bash
127.0.0.1:6379> cluster meet 127.0.0.1 6381
OK
127.0.0.1:6379> 2077:M 29 May 2021 07:19:54.654 # IP address for this node updated to 127.0.0.1
127.0.0.1:6379> 
127.0.0.1:6379> cluster info
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:0
cluster_current_epoch:2
cluster_my_epoch:2
cluster_stats_messages_ping_sent:30
cluster_stats_messages_pong_sent:32
cluster_stats_messages_meet_sent:2
cluster_stats_messages_sent:64
cluster_stats_messages_ping_received:32
cluster_stats_messages_pong_received:32
cluster_stats_messages_received:64
127.0.0.1:6379> cluster nodes
118cc85c73c3f9bee9653d2cbca7881dbbd7957f 127.0.0.1:6380@16380 master - 0 1622272870594 1 connected
7366db1f582ad74dadda9691368959285f9e4255 127.0.0.1:6379@16379 myself,master - 0 1622272868000 2 connected
bce5759afdbbee47ad417b683ea43a5f5bbfade7 127.0.0.1:6381@16381 master - 0 1622272871601 0 connected
```





## 分配槽

```bash
redis-cli -h 127.0.0.1 -p 6379 cluster addslots {0..5461}
redis-cli -h 127.0.0.1 -p 6380 cluster addslots {5462..10922}
redis-cli -h 127.0.0.1 -p 6381 cluster addslots {10923..16383}
```

- 分配槽后进入到使用客户端查看集群状态

  ```bash
  [dev@localhost data]$ redis-cli
  127.0.0.1:6379> cluster info
  cluster_state:ok
  cluster_slots_assigned:16384
  cluster_slots_ok:16384
  cluster_slots_pfail:0
  cluster_slots_fail:0
  cluster_known_nodes:6
  cluster_size:3
  cluster_current_epoch:5
  cluster_my_epoch:2
  cluster_stats_messages_ping_sent:1457
  cluster_stats_messages_pong_sent:1458
  cluster_stats_messages_meet_sent:5
  cluster_stats_messages_sent:2920
  cluster_stats_messages_ping_received:1458
  cluster_stats_messages_pong_received:1462
  cluster_stats_messages_received:2920
  127.0.0.1:6379> cluster nodes
  2b52cbf5867de9a88b97510d7cc584f7aa37bc88 127.0.0.1:6384@16384 master - 0 1622274276210 5 connected
  bce5759afdbbee47ad417b683ea43a5f5bbfade7 127.0.0.1:6381@16381 master - 0 1622274275204 3 connected 10923-16383
  fd341da7a76df6dcab2b1e75e58eb8a74799703c 127.0.0.1:6382@16382 master - 0 1622274273192 0 connected
  118cc85c73c3f9bee9653d2cbca7881dbbd7957f 127.0.0.1:6380@16380 master - 0 1622274274198 1 connected 5462-10922
  7366db1f582ad74dadda9691368959285f9e4255 127.0.0.1:6379@16379 myself,master - 0 1622274275000 2 connected 0-5461
  a7d978b2609a41e108cbc12f115ee66bb1dfa14e 127.0.0.1:6383@16383 master - 0 1622274277216 4 connected
  
  ```



设置从节点。`cluster replicate {nodeId}`

```bash
[dev@localhost data]$ redis-cli -h 127.0.0.1 -p 6382
127.0.0.1:6382> cluster replicate 7366db1f582ad74dadda9691368959285f9e4255
```

- 6383和6384分别设置为6380和6381的从节点

- 此时集群节点状态

  ```bash
  127.0.0.1:6384> cluster nodes
  bce5759afdbbee47ad417b683ea43a5f5bbfade7 127.0.0.1:6381@16381 master - 0 1622274665381 3 connected 10923-16383
  fd341da7a76df6dcab2b1e75e58eb8a74799703c 127.0.0.1:6382@16382 slave 7366db1f582ad74dadda9691368959285f9e4255 0 1622274664000 2 connected
  7366db1f582ad74dadda9691368959285f9e4255 127.0.0.1:6379@16379 master - 0 1622274666390 2 connected 0-5461
  a7d978b2609a41e108cbc12f115ee66bb1dfa14e 127.0.0.1:6383@16383 slave 118cc85c73c3f9bee9653d2cbca7881dbbd7957f 0 1622274667395 1 connected
  2b52cbf5867de9a88b97510d7cc584f7aa37bc88 127.0.0.1:6384@16384 myself,slave bce5759afdbbee47ad417b683ea43a5f5bbfade7 0 1622274665000 3 connected
  118cc85c73c3f9bee9653d2cbca7881dbbd7957f 127.0.0.1:6380@16380 master - 0 1622274665000 1 connected 5462-10922
  ```



> 至此，一个简单的集群搭建完成，关于集群动态伸缩，请求路由，故障转移部分的内容，后会无期~

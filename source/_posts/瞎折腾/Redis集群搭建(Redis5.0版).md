---
title: Redis集群搭建(Redis5.0版)
category: 瞎折腾
tag: Redis
abbrlink: 1044596185
date: 2021-05-30 12:30:00
---



{% cq %}

在Redis5中redis-trib.rb的功能被集成到了redis-cli中，大大简化了redis的集群部署，加快了进群部署的速度，也方便后期维护与扩容。此篇文章参考[官方文档](https://redis.io/topics/cluster-tutorial)搭建Redis集群。

**注：Redis版本应在5.0及以上。**

{% endcq %}

<!-- more -->





# Redis集群搭建(Redis5.0版)

## 编写配置文件

创建配置文件目录`cluster-test`及数据文件目录

```bash
mkdir -p /home/dev/data/redis/data/cluster-test mkdir -p /home/dev/data/redis/conf/cluster-test
```



编写配置文件

```properties
## /home/dev/data/redis/conf/cluster-test/redis-6379.conf 同理编写6380-6386的配置

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
dir /home/dev/data/redis/data/cluster-test
```



启动redis

```bash
redis-server redis-6379.conf &
redis-server redis-6380.conf &
redis-server redis-6381.conf &
redis-server redis-6382.conf &
redis-server redis-6383.conf &
redis-server redis-6384.conf &
```

- 这里可以使用redis官方提供的脚本进行基础集群的搭建。具体查看目录`redis-x.x.x/utils/create-cluster`。其中包含README说明及create-cluster脚本

## 启动集群

```bash
redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 --cluster-replicas 1
```

- **--cluster-replicas：1**：代表集群中的主节点有几个从节点

执行过程如下:

```bash
[dev@localhost create-cluster]$ redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:6383 to 127.0.0.1:6379
Adding replica 127.0.0.1:6384 to 127.0.0.1:6380
Adding replica 127.0.0.1:6382 to 127.0.0.1:6381
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379
   slots:[0-5460] (5461 slots) master
M: e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380
   slots:[5461-10922] (5462 slots) master
M: fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381
   slots:[10923-16383] (5461 slots) master
S: 3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382
   replicates e8e05250b518f5e774ce12f9cd8ff0c285c96bd5
S: 208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383
   replicates e34482fb5455d94959f5b46b41fc17cf72646a4c
S: 6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384
   replicates fcceb74f27471d80e991b33e23817200b47866ff
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 127.0.0.1:6379)
M: e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383
   slots: (0 slots) slave
   replicates e34482fb5455d94959f5b46b41fc17cf72646a4c
S: 6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384
   slots: (0 slots) slave
   replicates fcceb74f27471d80e991b33e23817200b47866ff
M: e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382
   slots: (0 slots) slave
   replicates e8e05250b518f5e774ce12f9cd8ff0c285c96bd5
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## 查看集群状态

- 查看集群状态

```bash
[dev@localhost create-cluster]$ redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:281
cluster_stats_messages_pong_sent:281
cluster_stats_messages_sent:562
cluster_stats_messages_ping_received:276
cluster_stats_messages_pong_received:281
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:562
```



- 查看集群节点信息

```bash
[dev@localhost create-cluster]$ redis-cli cluster nodes
fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381@16381 master - 0 1622448169000 3 connected 10923-16383
208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383@16383 slave e34482fb5455d94959f5b46b41fc17cf72646a4c 0 1622448169000 2 connected
6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384@16384 slave fcceb74f27471d80e991b33e23817200b47866ff 0 1622448170329 3 connected
e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380@16380 master - 0 1622448170000 2 connected 5461-10922
e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379@16379 myself,master - 0 1622448169000 1 connected 0-5460
3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382@16382 slave e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 0 1622448171335 1 connected
```



## 集群扩容

### 启动备用节点：

```bash
redis-server redis-6385.conf &
redis-server redis-6386.conf &
```



### 查看redis

```bash
[dev@localhost cluster-test]$ ps -ef | grep redis
dev      3934  3552  0 07:56 pts/1    00:00:01 redis-server *:6379 [cluster]
dev      3935  3552  0 07:56 pts/1    00:00:01 redis-server *:6380 [cluster]
dev      3936  3552  0 07:56 pts/1    00:00:01 redis-server *:6381 [cluster]
dev      3937  3552  0 07:56 pts/1    00:00:01 redis-server *:6382 [cluster]
dev      3938  3552  0 07:56 pts/1    00:00:01 redis-server *:6383 [cluster]
dev      3959  3552  0 07:56 pts/1    00:00:01 redis-server *:6384 [cluster]
dev      3989  3552  0 08:21 pts/1    00:00:00 redis-server *:6385 [cluster]
dev      3994  3552  0 08:21 pts/1    00:00:00 redis-server *:6386 [cluster]
dev      4001  3552  0 08:21 pts/1    00:00:00 grep --color=auto redis
```



### 添加master节点

> `redis-cli --cluster add-node <host_new>:<port_new> <host>:<port>`
>
> - **\<host_new\>:\<port_new\>**：新加入集群节点
> - **\<host\>:\<port\>**：集群内任意节点



```bash
[blog@localhost cluster-test]$ redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379
>>> Adding node 127.0.0.1:6385 to cluster 127.0.0.1:6379
>>> Performing Cluster Check (using node 127.0.0.1:6379)
M: e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383
   slots: (0 slots) slave
   replicates e34482fb5455d94959f5b46b41fc17cf72646a4c
S: 6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384
   slots: (0 slots) slave
   replicates fcceb74f27471d80e991b33e23817200b47866ff
M: e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382
   slots: (0 slots) slave
   replicates e8e05250b518f5e774ce12f9cd8ff0c285c96bd5
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:6385 to make it join the cluster.
[OK] New node added correctly.
```



### 分配槽

> ```bash
> redis-cli --cluster reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes
> ```
>
> - **\<host\>:\<port\>**：集群内任意节点
> - `--cluster-from`：从集群中哪些节点转移槽，多个几点用`,`分割
> - `--cluster-to`：转移槽到集群中的那个节点
> - `--cluster-slots`：分配的槽的数量
> - `--cluster-yes`：自动确认，不加此参数，需要手动输入yes确认槽迁移计划
>
> - **\<node-id\>**: 节点对应的集群ID，以集群模式启动时会写入到`nodes-{port}.conf`配置中。也可以通过`redis-cli cluster nodes`查看
>
>   ```bash
>   [blog@localhost create-cluster]$ redis-cli cluster nodes
>   b48e9109610b9952cde3f032f77563e441a3d75d 127.0.0.1:6385@16385 master - 0 1622449778389 0 connected
>   fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381@16381 master - 0 1622449776378 3 connected 10923-16383
>   208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383@16383 slave e34482fb5455d94959f5b46b41fc17cf72646a4c 0 1622449774000 2 connected
>   6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384@16384 slave fcceb74f27471d80e991b33e23817200b47866ff 0 1622449776000 3 connected
>   e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380@16380 master - 0 1622449777384 2 connected 5461-10922
>   e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379@16379 myself,master - 0 1622449775000 1 connected 0-5460
>   3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382@16382 slave e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 0 1622449777000 1 connected
>   ```
>
>   



将6379,6380,6381的词槽分别分给6385节点1024个词槽

```bash
redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from e8e05250b518f5e774ce12f9cd8ff0c285c96bd5,e34482fb5455d94959f5b46b41fc17cf72646a4c,fcceb74f27471d80e991b33e23817200b47866ff --cluster-to b48e9109610b9952cde3f032f77563e441a3d75d --cluster-slots 1024 --cluster-yes
```



迁移后的集群节点信息：

```bash
[blog@localhost create-cluster]$ redis-cli cluster nodes
b48e9109610b9952cde3f032f77563e441a3d75d 127.0.0.1:6385@16385 master - 0 1622450708439 7 connected 0-340 5461-5802 10923-11263
fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381@16381 master - 0 1622450708000 3 connected 11264-16383
208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383@16383 slave e34482fb5455d94959f5b46b41fc17cf72646a4c 0 1622450706000 2 connected
6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384@16384 slave fcceb74f27471d80e991b33e23817200b47866ff 0 1622450707433 3 connected
e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380@16380 master - 0 1622450706427 2 connected 5803-10922
e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379@16379 myself,master - 0 1622450705000 1 connected 341-5460
3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382@16382 slave e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 0 1622450709445 1 connected
```



### 添加从节点

> ```bash
> redis-cli --cluster add-node <host_new>:<port_new> <host>:<port> --cluster-slave \
> --cluster-master-id <node-id>
> ```
>
> - `--cluster-slave`：添加从节点
> - `--cluster-master-id <node-id>`：从节点的主节点id



```bash
[blog@localhost create-cluster]$ redis-cli --cluster add-node 127.0.0.1:6386 127.0.0.1:6385 --cluster-slave --cluster-master-id b48e9109610b9952cde3f032f77563e441a3d75d
>>> Adding node 127.0.0.1:6386 to cluster 127.0.0.1:6385
>>> Performing Cluster Check (using node 127.0.0.1:6385)
M: b48e9109610b9952cde3f032f77563e441a3d75d 127.0.0.1:6385
   slots:[0-340],[5461-5802],[10923-11263] (1024 slots) master
S: 208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383
   slots: (0 slots) slave
   replicates e34482fb5455d94959f5b46b41fc17cf72646a4c
M: e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380
   slots:[5803-10922] (5120 slots) master
   1 additional replica(s)
M: e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379
   slots:[341-5460] (5120 slots) master
   1 additional replica(s)
M: fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381
   slots:[11264-16383] (5120 slots) master
   1 additional replica(s)
S: 6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384
   slots: (0 slots) slave
   replicates fcceb74f27471d80e991b33e23817200b47866ff
S: 3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382
   slots: (0 slots) slave
   replicates e8e05250b518f5e774ce12f9cd8ff0c285c96bd5
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 127.0.0.1:6386 to make it join the cluster.
Waiting for the cluster to join

>>> Configure node as replica of 127.0.0.1:6385.
[OK] New node added correctly.
```



- 查看集群状态

  ```bash
  [blog@localhost create-cluster]$ redis-cli cluster nodes
  703596043233487cc79ef364d56a13907ccf7cb6 127.0.0.1:6386@16386 slave b48e9109610b9952cde3f032f77563e441a3d75d 0 1622451294000 7 connected
  b48e9109610b9952cde3f032f77563e441a3d75d 127.0.0.1:6385@16385 master - 0 1622451294000 7 connected 0-340 5461-5802 10923-11263
  fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381@16381 master - 0 1622451293000 3 connected 11264-16383
  208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383@16383 slave e34482fb5455d94959f5b46b41fc17cf72646a4c 0 1622451291000 2 connected
  6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384@16384 slave fcceb74f27471d80e991b33e23817200b47866ff 0 1622451295061 3 connected
  e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380@16380 master - 0 1622451296066 2 connected 5803-10922
  e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379@16379 myself,master - 0 1622451293000 1 connected 341-5460
  3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382@16382 slave e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 0 1622451294055 1 connected
  ```

  



## 集群收缩

> 下线节点 127.0.0.1:6385、127.0.0.1:6386



### 删除master对应的从节点

- 查看从节点

  ```bash
  [blog@localhost create-cluster]$ redis-cli cluster nodes | grep :6386
  703596043233487cc79ef364d56a13907ccf7cb6 127.0.0.1:6386@16386 slave b48e9109610b9952cde3f032f77563e441a3d75d 0 1622451429000 7 connected
  ```

- 从集群中删除从节点

  ```bash
  redis-cli --cluster del-node 127.0.0.1:6379 703596043233487cc79ef364d56a13907ccf7cb6
  ```

  - 后续使用`redis-cli -h 127.0.0.1 -p 6379 cluster nodes`查看集群节点状态，6386节点应该已经被移除。

### 清空槽

```bash
redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from b48e9109610b9952cde3f032f77563e441a3d75d --cluster-to e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 --cluster-slots 341 --cluster-yes

redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from b48e9109610b9952cde3f032f77563e441a3d75d --cluster-to e34482fb5455d94959f5b46b41fc17cf72646a4c --cluster-slots 341 --cluster-yes

redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from b48e9109610b9952cde3f032f77563e441a3d75d --cluster-to fcceb74f27471d80e991b33e23817200b47866ff --cluster-slots 342 --cluster-yes
```

- 由于目标节点只能写一个，因此需要执行三次。

- 查看当前集群槽分配状态

  ```bash
  [blog@localhost create-cluster]$ redis-cli cluster slots
  1) 1) (integer) 0 # 槽起始位置
     2) (integer) 5460 # 槽结束位置
     3) 1) "127.0.0.1" # master:host
        2) (integer) 6379 # master:port
        3) "e8e05250b518f5e774ce12f9cd8ff0c285c96bd5" # master-node-id
     4) 1) "127.0.0.1" # slave:host
        2) (integer) 6382 # slave:port
        3) "3eee4e2dc41e924d313b6fef575fab40055b69d9" # slave-node-id
  2) 1) (integer) 5461
     2) (integer) 5801
     3) 1) "127.0.0.1"
        2) (integer) 6380
        3) "e34482fb5455d94959f5b46b41fc17cf72646a4c"
     4) 1) "127.0.0.1"
        2) (integer) 6383
        3) "208df20aedbbf1687ac66386fc064771793504e6"
  3) 1) (integer) 5802
     2) (integer) 5802
     3) 1) "127.0.0.1"
        2) (integer) 6381
        3) "fcceb74f27471d80e991b33e23817200b47866ff"
     4) 1) "127.0.0.1"
        2) (integer) 6384
        3) "6964ab0161d106093024f643bbe03bf9daf57760"
  4) 1) (integer) 5803
     2) (integer) 10922
     3) 1) "127.0.0.1"
        2) (integer) 6380
        3) "e34482fb5455d94959f5b46b41fc17cf72646a4c"
     4) 1) "127.0.0.1"
        2) (integer) 6383
        3) "208df20aedbbf1687ac66386fc064771793504e6"
  5) 1) (integer) 10923
     2) (integer) 16383
     3) 1) "127.0.0.1"
        2) (integer) 6381
        3) "fcceb74f27471d80e991b33e23817200b47866ff"
     4) 1) "127.0.0.1"
        2) (integer) 6384
        3) "6964ab0161d106093024f643bbe03bf9daf57760"
  ```



### 下线master节点

```bash
[blog@localhost create-cluster]$ redis-cli --cluster del-node 127.0.0.1:6379 b48e9109610b9952cde3f032f77563e441a3d75d
>>> Removing node b48e9109610b9952cde3f032f77563e441a3d75d from cluster 127.0.0.1:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> Sending CLUSTER RESET SOFT to the deleted node.

[blog@localhost create-cluster]$ redis-cli cluster nodes # 查看当前集群节点状态
fcceb74f27471d80e991b33e23817200b47866ff 127.0.0.1:6381@16381 master - 0 1622452136635 10 connected 5802 10923-16383
208df20aedbbf1687ac66386fc064771793504e6 127.0.0.1:6383@16383 slave e34482fb5455d94959f5b46b41fc17cf72646a4c 0 1622452136000 9 connected
6964ab0161d106093024f643bbe03bf9daf57760 127.0.0.1:6384@16384 slave fcceb74f27471d80e991b33e23817200b47866ff 0 1622452138653 10 connected
e34482fb5455d94959f5b46b41fc17cf72646a4c 127.0.0.1:6380@16380 master - 0 1622452137642 9 connected 5461-5801 5803-10922
e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 127.0.0.1:6379@16379 myself,master - 0 1622452136000 8 connected 0-5460
3eee4e2dc41e924d313b6fef575fab40055b69d9 127.0.0.1:6382@16382 slave e8e05250b518f5e774ce12f9cd8ff0c285c96bd5 0 1622452137000 8 connected
```


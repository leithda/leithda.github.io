---
title: Redis开发与运维笔记-Redis的噩梦_阻塞
categories:
  - Redis
tags:
  - Redis
author: 长歌
abbrlink: 3586754738
date: 2020-06-09 22:54:39
---
> 参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)

<!-- more -->




> Redis是典型的单线程架构，所有的读写操作都是在一条主线程中完成 的。当Redis用于高并发场景时，这条线程就变成了它的生命线。如果出现 阻塞，哪怕是很短时间，对于我们的应用来说都是噩梦。导致阻塞问题的场 景大致分为内在原因和外在原因:
>
> - 内在原因包括:不合理地使用API或数据结构、CPU饱和、持久化阻塞 等。
> - 外在原因包括:CPU竞争、内存交换、网络问题等。



# 1. 发现阻塞

总结起来就一句话，通过分析日志及警报机制或者监控Redis的各项指标发现Redis阻塞。



# 2. 内在原因



## 2.1 API或数据结构使用不合理

1. 如何发现慢查询

   通过Redis原生提供的慢查询统计功能即可发现慢查询语句。解决方法如下：

   - 修改为低算法度的命令，如hgetall改为hmget等，禁用keys、sort等命 令。
   - 调整大对象，缩减大对象数据或把大对象拆分为多个小对象

2. 如何发现大对象

   Redis本身提供发现大对象的工具，对应命令`redis-cli-h{ip} -p{port} bigkeys`



## 2.2 CPU饱和

​	使用统计命令`redis-cli -h{ip} -p{port} --stat`获取当前 Redis使用情况。

​	当Redis实例OPS过高导致CPU接近饱和时，无法进行优化，这时需要做集群化水平扩展来分摊OPS压力。如果Redis的OPS不高却使CPU使用率饱和是不正常的。这种情况可能是使用了高算法复杂度的命令。还有一种情况就是过度优化。这种情况有些隐蔽，需要我们根据info commandstats统计信息分析出命令不合理开销时间。如下：

```bash
cmdstat_hset:calls=198757512,usec=27021957243,usec_per_call=135.95
```

​	hset命令的复杂度为O(1)，但平均耗时却达到135微妙。这是由于过度放宽了ziplist的要求。针对ziplist的操作算法复杂度在O(n)和O(n^2^)之间，当hash存储对象过多时，会造成操作更慢且更消耗CPU。



## 2.3 持久化阻塞

持久化引起主线程阻塞的操作主要有：fork阻塞、AOF刷盘阻塞、 HugePage写操作阻塞。



### 2.3.1 fork阻塞

​	fork操作发生在RDB和AOF重写时，Redis主线程调用fork操作产生共享 内存的子进程，由子进程完成持久化文件重写工作。如果fork操作本身耗时 过长，必然会导致主线程的阻塞。



### 2.3.2 AOF刷盘阻塞

​	当我们开启AOF持久化功能时，文件刷盘的方式一般采用每秒一次，后 台线程每秒对AOF文件做fsync操作。当硬盘压力过大时，fsync操作需要等 待，直到写入完成。如果主线程发现距离上一次的fsync成功超过2秒，为了 数据安全性它会阻塞直到后台线程执行fsync操作完成，这种阻塞行为主要 是硬盘压力引起，可以查看Redis日志识别出这种情况，当发生这种阻塞行 为时，会打印如下日志:

```bash
Asynchronous AOF fsync is taking too long (disk is busy). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.
```

​	也可以查看info persistence统计中的aof_delayed_fsync指标，每次发生 fdatasync阻塞主线程时会累加。

> 硬盘压力可能是Redis进程引起的，也可能是其他进程引起的，可以使 用iotop查看具体是哪个进程消耗过多的硬盘资源



### 2.3.3 HugePage写操作阻塞

​	子进程在执行重写期间利用Linux写时复制技术降低内存开销，因此只 有写操作时Redis才复制要修改的内存页。对于开启Transparent HugePages的 操作系统，每次写命令引起的复制内存页单位由4K变为2MB，放大了512 倍，会拖慢写操作的执行时间，导致大量写操作慢查询。例如简单的incr命令也会出现在慢查询中。

> Redis官方文档中针对绝大多数的阻塞问题进行了分类说明，这里不再 详细介绍，细节请见:http://www.redis.io/topics/latency。





# 3. 外在原因

## 3.1 CPU竞争

产生CPU竞争问题主要如下：

- 进程竞争：Redis是典型的CPU密集型应用。可以通过`top`、`sar`等命令定位到CPU消耗的时间点和具体进程
- 绑定CPU：部署Redis时为了充分利用多核CPU，通常一个机器部署多个Redis实例，常见的优化是一个Redis进程绑定到CPU上，降低频繁切换CPU上下文的开销。但是当Redis开启持久化时，绑定CPU会导致子进程也使用父进程的CPU，子进程重写时对单核CPU使用率达到90%以上，父进程与子进程产生激烈CPU竞争。因此开启的持久化或参与复制的主节点不建议绑定CPU。



## 3.2 内存交换

​	内存交换(swap)对于Redis来说是非常致命的，Redis保证高性能的一 个重要前提是所有的数据在内存中。如果操作系统把Redis使用的部分内存 换出到硬盘，由于内存与硬盘读写速度差几个数量级，会导致发生交换后的 Redis性能急剧下降。识别Redis内存交换的检查方法如下:

1. 查询Redis进程号

   ```bash
   # redis-cli -p 6383 info server | grep process_id
   process_id:4476
   ```



2. 根据进程号查询内存交换信息:

   ```bash
   cat /proc/4476/smaps | grep Swap 0 kB
   Swap: 0 kB
   Swap: 4 kB
   Swap: 0 kB
   Swap:	0 kB
   Swap: 0 kB
   .....
   ```

   如果交换量都是0KB或者个别的是4KB，则是正常现象，说明Redis进程 内存没有被交换。预防内存交换的方法有:

   - 证机器充足的可用内存。
   - 确保所有Redis实例设置最大可用内存(maxmemory)，防止极端情况下Redis内存不可控的增长。
   - 降低系统使用swap优先级，如echo10>/proc/sys/vm/swappiness



## 3.3 网络问题

### 3.3.1 连接拒绝

​	当出现网络闪断或者连接数溢出时，客户端会出现无法连接Redis的情况。我们需要区分这三种情况:网络闪断、Redis连接拒绝、连接溢出

1. 网络闪断

   ​	一般发生在网络割接或者带宽耗尽的情况，对于网络闪断的识别比较困难，常见的做法可以通过sar-n DEV查看本机历史 流量是否正常，或者借助外部系统监控工具(如Ganglia)进行识别。具体 问题定位需要更上层的运维支持，对于重要的Redis服务需要充分考虑部署 架构的优化，尽量避免客户端与Redis之间异地跨机房调用。

   

2. Redis连接拒绝

   ​	Redis通过maxclients参数控制客户端最大 连接数，默认10000。当Redis连接数大于maxclients时会拒绝新的连接进入， info stats的rejected_connections统计指标记录所有被拒绝连接的数量:

   ```bash
   redis-cli -p 6384 info Stats | grep rejected_connections 
   rejected_connections:0
   ```
   
   > 当Redis用于大量分布式节点访问且生命周期比较短的场景时，如比较 典型的在Map/Reduce中使用Redis。因为客户端服务存在频繁启动和销毁的 情况且默认Redis不会主动关闭长时间闲置连接或检查关闭无效的TCP连 接，因此会导致Redis连接数快速消耗且无法释放的问题。这种场景下建议 设置tcp-keepalive和timeout参数让Redis主动检查和关闭无效连接。



3. 连接溢出

   连接溢出的原因较多，这里主要介绍两种：

   1. 进程限制

      使用`ulimit -n`查看Linux最大句柄数限制，

   2. backlog队列溢出

      ​	系统对于特定端口的TCP连接使用backlog队列保存。Redis默认的长度 为511，通过tcp-backlog参数设置。如果Redis用于高并发场景为了防止缓慢 连接占用，可适当增大这个设置，但必须大于操作系统允许值才能生效。当 Redis启动时如果tcp-backlog设置大于系统允许值将以系统值为准，Redis打 印如下警告日志:

      ```bash
      netstat -s | grep overflowed
      663 times the listen queue of a socket overflowed
      ```

      > 如果怀疑是backlog队列溢出，线上可以使用cron定时执行netstat-s|grep overflowed统计，查看是否有持续增长的连接拒绝情况



### 3.3.2 网络延迟

​	可以使用`edis-cli -h{host} -p{port} --latency`等系列命令进行延迟测试。减缓网络延迟可以更改拓扑结构来解决。



### 3.3.3 网卡软中断

​	网卡软中断是指由于单个网卡队列只能使用一个CPU，高并发下网卡数据交互都集中在同一个CPU，导致无法充分利用多核CPU的情况。网卡软中 断瓶颈一般出现在网络高流量吞吐的场景，如下使用“top+数字1”命令可以 很明显看到CPU1的软中断指标(si)过高:

```bash
# top
Cpu0 : 15.3%us, 0.3%sy, 0.0%ni, 84.4%id, 0.0%wa, 0.0%hi, 0.0%si, 0.0%st 
Cpu1 : 16.6%us, 2.0%sy, 0.0%ni, 47.1%id, 3.3%wa, 0.0%hi, 31.0%si, 0.0%st 
Cpu2 : 13.3%us, 0.7%sy, 0.0%ni, 86.0%id, 0.0%wa, 0.0%hi, 0.0%si, 0.0%st 
Cpu3 : 14.3%us, 1.7%sy, 0.0%ni, 82.4%id, 1.0%wa, 0.0%hi, 0.7%si, 0.0%st 
.....
Cpu15 : 10.3%us, 8.0%sy, 0.0%ni, 78.7%id, 1.7%wa, 0.3%hi, 1.0%si, 0.0%st
```

> Linux在内核2.6.35以后支持Receive Packet Steering(RPS)，实现了在 软件层面模拟硬件的多队列网卡功能。如何配置多CPU分摊软中断已超出本 书的范畴，具体配置见Torvalds的GitHub文 档:https://github.com/torvalds/linux/blob/master/Documentation/networking/scaling.txt
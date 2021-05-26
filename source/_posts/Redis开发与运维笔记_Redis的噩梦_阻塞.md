---
title: Redis开发与运维笔记-Redis的噩梦-阻塞
categories:
  - DB
  - redis
tags:
  - redis
author: leithda
abbrlink: 3586754738
date: 2021-05-26 22:50:00

---

{% cq %}
Redis是典型的单线程架构，所有的读写操作都是在一条主线程中完成的。当Redis用于高并发场景时，这条线程就变成了它的生命线。如果出现阻塞，哪怕是很短时间，对于我们的应用来说都是噩梦
{% endcq %}

<!-- more -->

> 参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)

<hr>


# Redis的噩梦-阻塞

## 发现阻塞

通过收集Redis日志或者通过监控Redis的关键指标等发现Redis阻塞问题。

## 内在原因

### API或数据结构使用不合理

对大数据量的hash结构执行`hgetall`操作，由于其算法复杂度为`O(n)`，这条命令执行速度必然很慢。这个问题就是典型的不合理使用API和数据结构。

1. 发现慢查询
   通过`slowlog get {n}`获取慢查询日志。更多详情请查看[Redis开发与运维笔记-小功能大用处](./2425201134.html#慢查询分析)

2. 发现大对象
   通过`redis-cli -h {ip} -p {port} bigkeys`发现大对象。其内部采用分段进行scan操作，把黎士扫描过的最大对象统计出来便于分析优化。


### CPU饱和的问题

通过`reids-cli --stat`获取当前Redis使用情况。
通过`top`命令获取进程对CPU的利用率等信息
通过`info commandstats`统计信息分析出命令不合理开销时间，查看是否是因为高算法复杂度或者过度的内存优化问题。

### 持久化相关的阻塞

1. fork阻塞
   持久化时，Redis调用fork操作产生共享内存的子进程，由子进程完成持久化文件重写工作。fork时间过长，阻塞主线程。
   通过`info stats`命令获取到`latest_fork_usec`指标，表示Redis最近一次fork操作耗时

2. AOF刷盘阻塞
   开启AOF持久化后，文件刷盘一般1s一次，如果主线程发现距离上次fsync成功超过2秒，为了数据安全性它会阻塞直到后台线程执行fsync操作完成。通过Redis日志可以识别这种情况。也可以通过`info persistence`统计中的`aof_delayed_fsync`指标，每次发生fdatasync阻塞主线程时会累加。

3. HugePage写操作阻塞
   子进程在执行重写期间利用Linux写时复制技术降低内存开销，因此只有写操作时Redis才复制要修改的内存页。对于开启**Transparent HugePages[^1]**的操作系统，，每次写复制的内存页单位由4K变为2MB，会拖慢写操作的执行时间，导致大量写操作慢查询。

## 外在原因

### CPU竞争

- 进程竞争： Redis是典型的CPU密集型应用，不建议和其他多核CPU密集型服务部署在一起.通过`top`、`sar`命令定位CPU消耗的时间点和具体进程
- 绑定CPU：部署Redis时为了充分利用多核CPU，通常在一个机器上部署多个实例。常见的一种优化是将Redis进程绑定在CPU上，用于降低CPU频繁上下文切换的开销。但是当Redis进行RDB/AOF重写时，如果绑定CPU，子进程重写时会大量消耗CPU资源，对Redis主进程造成影响。因此对于开启持久化或复制的主节点不建议绑定CPU。

### 内存交换

内存交换是非常致命的。Redis保证高性能主要原因是所有数据在内存中，如果操作系统把Redis使用的部分内存换出到硬盘，会造成性能的机制下降。识别Redis内存交换的检查方法如下：

1. 查询Redis进程号

```bash
reids-cli -p 6383 info server | grep process_id
process_id: 4476
```

2. 根据进程号查询内存交换信息

```bash
cat /proc/4476/smaps | grep Swap
Swap: 0kB
Swap: 0kB
Swap: 4kB
Swap: 0kB
Swap: 0kB
.....
```

如果交换量都是0KB或者个别的是4KB，则正常。预防内存交换的方法由：

- 保证机器充足的可用内存
- 确保所有Redis实例设置最大可用内存(maxmemory)，防止极端情况Redis内存不可控的增长
- 降低系统使用swap优先级，如`echo 10 > /proc/sys/vm/swappiness`

### 网络问题

#### 连接拒绝

1. 网络闪断
   一般发生在网络切割或者带宽耗尽的情况，对于网络闪断的识别比较困难，常见做法是通过`sar -n DEV`查看本机历史流量是否正常，或者借助外部系统监控工具如`Ganglia`进行识别。

2. Redis连接拒绝
   Redis客户端数量超过`maxclients`限制时，会拒绝新的连接接入。`info stats`中的`rejected_connections`统计指标记录被拒绝连接的数量。

> 建议根据情况设置`tcp-keepalive`和`timeout`参数让Redis主动检查和关闭无效连接。

3. 连接溢出
   1. 操作系统文件句柄数限制 `ulimit -n`
   2. backlog队列溢出
      系统对于特定的TCP连接使用backlog队列保存。Redis通过`tcp-backlog`设置，默认511.系统的backlog默认128，使用`echo 511 > /proc/sys/net/core/somaxconn`命令修改。
       通过`netstat -s | grep overflowed`查看因backlog队列溢出造成的连接拒绝统计指标。

#### 网络延迟

网络延迟经常出现在跨机房的部署结构上，对于机房之间延迟比较严重的场景需要调整拓扑结构。
测试网络延迟可以使用`redis-cli --latency`相关命令进行测试。
带宽瓶颈通常出现在以下几个方面：

- 机器网卡带宽
- 机器交换机带宽
- 机房之间专线带宽

#### 网卡软中断

网卡软中断是指由于单个网卡队列只能使用一个CPU，高并发下数据交互都集中在一个CPU上，导致无法充分利用多核CPU的情况。网卡软中断瓶颈一般出现在网络高流量吞吐的场景，使用`top`后按数字`1`查看CPU使用情况，其中`si`为软中断参数

## 本章重点回顾

1. 客户端最先感知阻塞等Redis超时行为，加入日志监控报警工具可快速定位阻塞问题，同时需要对Redis进程和机器做全面监控。
2. 阻塞的内在原因：确认主线程是否存在阻塞，检查慢查询等信息，发现不合理使用API或数据结构的情况，如`keys`、`sort`、`hgetall`等。关注CPU使用率防止单核跑满。当硬盘IO资源紧张时，AOF追加也会阻塞主进程。
3. 阻塞的外在原因：CPU竞争、内存交换、网络问题等。

<hr>


[^1]: 关于Huge Pages与 Transparent Huge Pages的关系与区别请参阅博客[潇湘隐者 - Linux传统Huge Pages与Transparent Huge Pages再次学习总结](https://www.cnblogs.com/kerrycode/p/7760026.html)
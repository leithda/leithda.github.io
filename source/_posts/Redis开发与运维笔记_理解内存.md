---
title: Redis开发与运维笔记-理解内存
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: 长歌
abbrlink: 1495994843
date: 2021-03-11 23:00:00
---

{% cq %}
Redis所有的数据都存在内存中，高效利用Redis内存首先需要理解Redis内存消耗在哪里，如何管理内存，最后才能考虑如何优化内存。
{% endcq %}
<!-- more -->

> 参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)

<hr>

# 理解内存

## 内存消耗

### 内存使用统计



`info memory`

| 属性名                  | 属性说明                                                     |
| ----------------------- | ------------------------------------------------------------ |
| used_memory             | Redis分配器分配的内存总量，也就是内部存储的所有数据的内存占用量 |
| used_memory_human       | 以可读的格式返回used_memory                                  |
| used_memory_rss         | 从操作系统的角度显示Redis进程占用的物理内存总量              |
| used_memory_peak        | 内存使用的最大值，表示used_memory的峰值                      |
| used_memory_peak_human  | 以可读的格式返回used_memory_peak                             |
| used_memory_lua         | Lua引擎所消耗的内存大小                                      |
| mem_fragmentation_retio | used_memory_rss/used_memory比值，表示内存碎片率              |
| mem_allocator           | Redis所使用的内存分配器。默认为jemalloc                      |

重点关注的指标：`used_memory_rss`、`used_memory`以及`mem_fragmentation_ratio`

mem_fragmentation>1，说明存在内存碎片。<1时说明操作系统Reids内存交换(Swap)到硬盘。



### 内存消耗划分

Redis进程内消耗主要包括：自身内存+对象内存+缓存内存+内存碎片。

{% asset_img Redis内存消耗划分.png Redis内存消耗划分 %}

- 对象内存
  是Redis内存占用最大的一块，存储着用户所有的数据。

- 缓存内存
  - 客户端缓冲：接入到服务器TCP了解的输入输出缓冲。输入缓冲无法控制，最大为1G。输出缓冲通过`client-output-buffer-limit`控制。普通客户端要注意控制数量，从客户端不要挂载过多，订阅客户端要注意生产及消费消息的速度
  - 复制积压缓冲区：用于实现部分复制功能，Redis提供的一个缓冲区，建议在合理范围内调大，可以有效避免全量复制
  - AOF缓冲区：这部分空间用于在Redis重写期间保存最近的写入命令。 

- 内存碎片
  容易出现高内存碎片的场景：
  1. 频繁做更新操作，例如频繁对已存在的键执行`append、setrange`等更新操作
  2. 大量过期键删除，键对象过期删除后，释放的空间无法得到充分利用，导致碎片率上升。
  3. 解决以上问题的方法：1）尽量做到数据对齐，2）安全重启

### 子进程内存消耗
主要发生在AOF/RDB重写时Redis创建的子进程内存消耗。总结如下：
1. Redis产生的子进程因为写时复制技术，并不需要消耗1倍的父进程内存。预留足够内存防止溢出即可
2. 需要设置`sysctl vm.overcommit_memory=1`允许内核可以分配所有物理内存，防止Redis进程执行fork时因系统剩余内存不足而失败
3. 排查当前系统是否支持并开启THP，如果开启建议关闭，防止copy-on-write期间内存过度消耗

## 内存管理

1. 通过`maxmemory`设置Redis最大内存，需要注意的是由于碎片率的存在，实际消耗的内存可能会比`maxmemory`设置的更大
2. 通过`config set maxmemory 2GB`可以动态调整内存上限
3. 内存回收策略
   - 过期键删除策略：惰性删除和定时任务删除
   - 内存溢出控制策略，通过`maxmemory-policy`参数控制
     - `noeviction`:默认策略，不删除数据。拒绝所有写入操作并返回客户端错误信息`(error)OOM command not allowed when used memory`，此时Redis只响应读操作
     - `vloatile-lru`：根据LRU算法删除设置了超时的键，直到腾出足够空间。如果没有可删除的键，回退到`noeviction`
     - `allkeys-lru`：根据LRU算法删除键。直到腾出足够空间
     - `allkeys-random`：随机删除所有键，直到腾出足够空间
     - `volatile-ttl`：根据键值对象的ttl，删除最近要过期的数据，如果没有，回退到noeviction策略
     - 设置了`maxmemory`时，当`used_memory>maxmemory`的状态下，会触发回收内存操作。应设置足够大的`maxmemory`避免长期处于这种状态下。

## 内存优化


## 本章重点回顾
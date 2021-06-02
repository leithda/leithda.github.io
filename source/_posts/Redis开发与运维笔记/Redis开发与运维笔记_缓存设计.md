---
title: Redis开发与运维笔记-缓存设计
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: leithda
abbrlink: 3358854259
date: 2021-06-02 21:15:00
---



{% cq %}

缓存能够有效地加速应用的读写速度，同时也可以降低后端负载，对日常应用的开发至关重要。但是将缓存加入应用架构后也会带来一些问题，本 章将针对这些问题介绍缓存使用技巧和设计方案

{% endcq %}



<!-- more -->



# 缓存设计

## 缓存的收益与成本

缓存收益：

- 加速读写
- 降低后端负载

缓存成本：

- 数据不一致
- 代码维护成本
- 维护成本



## 缓存更新策略

### LRU/LFU/FIFO算法剔除

清理数据由算法决定，一致性无法得到保障。算法无需开发人员实现，根据业务场景选择合适算法即可。

### 超时剔除

通过给键设置过期时间，过期期间内数据可能存在数据不一致问题。

### 主动更新

对数据一致性要求高，需要在真实数据更新后通过消息或其他方式通知立即更新缓存。一致性最高，但如果主动更新发生了问题，那么这条数据很可能很长时间不会更新，所以建议结合超时剔除一起使用效果会更好。维护成本会比较高，开发者需要自己来完成更新，并保证更新操作的正确性。



| 策略                 | 一致性 | 维护成本 |
| -------------------- | ------ | -------- |
| LRU/LFU/FIFO算法剔除 | 最差   | 低       |
| 超时剔除             | 较差   | 较低     |
| 主动更新             | 强     | 高       |

### 最佳实践

- 低一致性业务建议配置最大内存和淘汰策略的方式使用
- 高一致性业务可以结合使用超时剔除和主动更新，这样即使主动更新出了问题，也能保证数据过期时间后删除脏数据



## 缓存粒度控制

缓存粒度需要在空间以及时间上进行取舍，决定时需要综合数据通用性、空间占用比、代码维护性三点进行取舍



## 穿透优化

缓存穿透是指查询一个根本不存在的数据，缓存层和存储层都不会命中，通常出于容错的考虑，如果从存储层查不到数据则不写入缓存层。



### 缓存空对象

存储层不命中时，仍然将空对象保存到缓存层。这样下次查询会命中缓存层。这样解决会导致以下两个问题：

- 空值进行缓存，浪费了内存

  通常要设置一个较短的过期时间，过期后剔除缓存。

- 数据不一致，过期时间内，当前缓存与储存层存在数据不一致现象

  存储层修改时，主动更新缓存层数据。



### 布隆过滤器

有关布隆过滤器的相关知识，可以参考：https://en.wikipedia.org/wiki/Bloom_filter可以利用Redis的Bitmaps实现布 隆过滤器，GitHub上已经开源了类似的方案，读者可以进行参 考：https://github.com/erikdubbelboer/redis-lua-scaling-bloom-filter。



> 这种方法适用于数据命中不高、数据相对固定、实时性低（通常是数据集较大）的应用场景，代码维护较为复杂，但是缓存空间占用少



## 无底洞优化

分布式服务节点过多时，数据分布到的节点数更多，一次批量操作的数据可能分布在n个节点上，操作的时间为`n*网络+n*命令`，常见的优化思路如下：

### 串行命令

执行n次get命令，Jedis客户端示例如下

```java
List<String> serialMGet(List<String> keys) { 
    // 结果集 
    List<String> values = new ArrayList<String>(); 
    // n次串行get 
    for (String key : keys) 
    { 
        String value = jedisCluster.get(key); 
        values.add(value);
    } 
    return values;
}
```



### 串行IO

使用Smart客户端，将映射到相同槽（节点）的key合并为一个集合，之后对每个节点执行mget或pipeline命令。它的操作时间=node次网络时间+n次命令时间，如果节点数过多，还是存在性能问题。Jedis客户端示例如下：

```java
Map<String, String> serialIOMget(List<String> keys) { 
    // 结果集 
    Map<String, String> keyValueMap = new HashMap<String, String>(); 
    // 属于各个节点的key列表,JedisPool要提供基于ip和port的hashcode方法 
    Map<JedisPool, List<String>> nodeKeyListMap = new HashMap<JedisPool, List<String>>(); 
    // 遍历所有的key
    for (String key : keys) { 
        // 使用CRC16本地计算每个key的slot 
        int slot = JedisClusterCRC16.getSlot(key); 
        // 通过jedisCluster本地slot->node映射获取slot对应的node 
        JedisPool jedisPool = jedisCluster.getConnectionHandler().getJedisPoolFromSlot(slot);
        // 归档 
        if (nodeKeyListMap.containsKey(jedisPool)) { 
            nodeKeyListMap.get(jedisPool).add(key);
        } else { 
            List<String> list = new ArrayList<String>(); 
            list.add(key); 
            nodeKeyListMap.put(jedisPool, list);
        }
    } 
    // 从每个节点上批量获取，这里使用mget也可以使用pipeline 
    for (Entry<JedisPool, List<String>> entry : nodeKeyListMap.entrySet()) {
        JedisPool jedisPool = entry.getKey(); 
        List<String> nodeKeyList = entry.getValue(); 
        // 列表变为数组 
        String[] nodeKeyArray = nodeKeyList.toArray(new String[nodeKeyList.size()]); 
        // 批量获取，可以使用mget或者Pipeline 
        List<String> nodeValueList = jedisPool.getResource().mget(nodeKeyArray); 
        // 归档 
        for (int i = 0; i < nodeKeyList.size(); i++) { 
            keyValueMap.put(nodeKeyList.get(i), nodeValueList.get(i));
        }
    } 
    return keyValueMap;
}
```

### 并行IO

将方案2中的改为多线程执行。由于改为多线程，网络时间变为O(1)，这种方案会增加编程的复杂度。



### hash_tag

hash_tag可以将多个key强制分配到同一个槽(节点)上，它的操作时间=1次网络时间+n次命令时间



### 四种方案对比

| 方案     | 优点                                              | 缺点                                        | 网络IO             |
| -------- | ------------------------------------------------- | ------------------------------------------- | ------------------ |
| 串行命令 | 1）编程简单<br/>2）如果少量keys，性能可以满足要求 | 大量keys请求延迟严重                        | O(keys)            |
| 串行IO   | 1）编程简单<br/>2）少量节点，性能满足要求         | 大量node延迟严重                            | O(nodes)           |
| 并行IO   | 利用并行特性，延迟取决于最慢的节点                | 1）编程复杂<br/>2）多线程，定位问题比较难   | O(max_slow(nodes)) |
| hash_tag | 性能最高                                          | 1）业务维护成本较高<br/>2）容易出现数据倾斜 | O(1)               |



## 雪崩优化

缓存层由于某些原因不能提供服务或大量键同时过期，导致大量的请求发送到存储层，造成级联宕机的情况。

### 保证缓存层高可用

使用`Redis Sentinel` 或 `Redis Cluster`架构增强可用性

### 使用隔离组件为后端限流并降级

对重要资源进行隔离，让每种资源都运行在各自的线程池中，避免因个别服务出现问题导致的全体服务不可用。这部分可以查看[Hystrix](https://github.com/netflix/hystrix)，注意的是Hystrix只适用于Java。

### 提前演练

上线前对各种情况进行演练，包括但不限于，数据库宕机，缓存不可用，大量并发涌入系统等。





## 热点key重建优化

当一个热点key的并发量非常大，并且更新缓存不能在短时间内完成时，缓存失效到重建期间，大量请求的话，会导致大量线程重建缓存，增加后端负载。要解决这个问题可以减少重建缓存的次数。

### 互斥锁

只允许一个线程重建缓存，其他线程等待重建缓存的线程执行完，从缓存中获取数据即可。使用redis的setnx实现如下：

```java
String get(String key) { 
    // 从Redis中获取数据 
    String value = redis.get(key); 
    // 如果value为空，则开始重构缓存 
    if (value == null) { 
        // 只允许一个线程重构缓存，使用nx，并设置过期时间ex 
        String mutexKey = "mutext:key:" + key; 
        if (redis.set(mutexKey, "1", "ex 180", "nx")) { 
            // 从数据源获取数据 
            value = db.get(key); 
            // 回写Redis，并设置过期时间 
            redis.setex(key, timeout, value); 
            // 删除key_mutex 
            redis.delete(mutexKey);
        } 
        // 其他线程休息50毫秒后重试 
        else { 
            Thread.sleep(50); 
            get(key);
        }
    } 
    return value;
}
```



### 永远不过期

缓存中不设置过期时间，过期时间由业务系统来维护。业务系统发现缓存过期后，通过单独的线程去重建缓存。此种方法在重建缓存期间会存在数据不一致情况。



## 本章重点回顾

1. 缓存的使用带来的收益是能够加速读写，降低后端存储负载。 
2. 缓存的使用带来的成本是缓存和存储数据不一致性，代码维护成本增大，架构复杂度增大。 
3. 比较推荐的缓存更新策略是结合剔除、超时、主动更新三种方案共同完成。 
4. 穿透问题：使用缓存空对象和布隆过滤器来解决，注意它们各自的使用场景和局限性。 
5. 无底洞问题：分布式缓存中，有更多的机器不保证有更高的性能。有四种批量操作方式：串行命令、串行IO、并行IO、hash_tag。 
6. 雪崩问题：缓存层高可用、客户端降级、提前演练是解决雪崩问题的重要方法。 
7. 热点key问题：互斥锁、“永远不过期”能够在一定程度上解决热点key问题，开发人员在使用时要了解它们各自的使用成本。


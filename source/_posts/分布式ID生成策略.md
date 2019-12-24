---
title: 分布式ID生成策略
abbrlink: 3146196455
date: 2019-12-24 09:38:49
categories:
  - Distributed
  - Tools
tags:
  - 分布式
  - Snowflake
author: 长歌
---

{% cq %}
常见的分布式ID生成策略解析
{% endcq %}
<!-- More -->

## 使用数据库的auto_increment来生成

### 优点

1. 操作简单，使用数据库原生方法
2. 能够保证唯一且自增
3. id之间的步长固定并且可以自定义

### 缺点

1. 可用性低，数据库常用 **一主多从,读写分离** 架构，生成ID是写请求，主库挂了会影响业务
2. 性能主要取决于数据库性能，性能有上限

### 改进方案

{% asset_img optimise01.png 方案1改进方案 %}

- 增加数据库数量，横向扩展，为每个写库设置不同的初始值并使用相同的渐进步长。
- 仍存在以下缺点：
  - ID非绝对自增，访问DB01 生成ID[0,3],再访问DB02生成ID[1,4]。这种方案只能保证趋势自增
  - 仍存在数据库写压力问题，每次生成ID都需要访问数据库

## 单点批量ID生成

一次按需生成多个ID，每次生成都访问数据库，将数据库修改为最大的ID值，并在内存中记录最大值。

{% asset_img optimise02.png 方案2结构图 %}

### 优点

1. 保证了ID的递增有序
2. 降低了数据库的压力，不必每次生成ID都访问数据库

### 缺点

1. 服务仍是单点
2. 服务挂了重启之后会造成ID不连续

### 改进方案

单点服务常用的优化方案是“备用服务”，也叫“影子服务”，所以可以优化如下：

{% asset_img optimise03.png 方案2优化方案 %}

如上图，主服务挂的时候影子服务器顶上，切换过程对调用方透明，可以自动完成，常用的技术是 **vip+keepalived**。

## uuid/guid

uuid是一种本地生成ID的方法。

`UUID uuid = UUID.randomUUID();`



### 优点

- 本地生成ID，不需要远程调用，时延低
- 扩展性好，没有性能上限



### 缺点

- 无法保证趋势递增
- uuid过长，作为主键建立索引查询效率低



## 取当前毫秒数



### 优点

- 本地生成ID，不需要远程调用，时延低
- 生成的ID趋势递增
- 生成的ID是整数，建立索引后查询效率高



### 缺点

- 并发量超过1000后，会生成重复的ID(可以使用微妙进行优化，但仍有限)



## Redis生成ID

Redis是单线程的，可以通过Redis的原子操作`INCR`和`INCRBY`来生成ID。



### 优点

- 性能高于传统数据库
- ID顺序递增，分页及结果排序方便



### 缺点

- 引入Redis会增加系统复杂度
- 编码复杂



## Twitter开源的Snowflake算法

snowflake 是 twitter 开源的分布式ID生成算法，其核心思想为，一个long型的ID：

- 41 bit 作为毫秒数 - **41位的长度可以使用69年**
- 10 bit 作为机器编号 （5个bit是数据中心，5个bit的机器ID） - **10位的长度最多支持部署1024个节点**
- 12 bit 作为毫秒内序列号 - **12位的计数顺序号支持每个节点每毫秒产生4096个ID序号**

{% asset_img snowflake.png 雪花算法结构图 %}

算法单机每秒内理论上最多可生成1000*(2^12)，也就是400W的ID,完全能满足需求



该算法java版本实现代码如下：



```java
public class SnowflakeIdGenerator {
    //================================================Algorithm's Parameter=============================================
    // 系统开始时间截 (UTC 2017-06-28 00:00:00)
    private final long startTime = 1498608000000L;
    // 机器id所占的位数
    private final long workerIdBits = 5L;
    // 数据标识id所占的位数
    private final long dataCenterIdBits = 5L;
    // 支持的最大机器id(十进制)，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
    // -1L 左移 5位 (worker id 所占位数) 即 5位二进制所能获得的最大十进制数 - 31
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    // 支持的最大数据标识id - 31
    private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);
    // 序列在id中占的位数
    private final long sequenceBits = 12L;
    // 机器ID 左移位数 - 12 (即末 sequence 所占用的位数)
    private final long workerIdMoveBits = sequenceBits;
    // 数据标识id 左移位数 - 17(12+5)
    private final long dataCenterIdMoveBits = sequenceBits + workerIdBits;
    // 时间截向 左移位数 - 22(5+5+12)
    private final long timestampMoveBits = sequenceBits + workerIdBits + dataCenterIdBits;
    // 生成序列的掩码(12位所对应的最大整数值)，这里为4095 (0b111111111111=0xfff=4095)
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);
    //=================================================Works's Parameter================================================
    /**
     * 工作机器ID(0~31)
     */
    private long workerId;
    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;
    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;
    /**
     * 上次生成ID的时间截
     */
    private long lastTimestamp = -1L;
    //===============================================Constructors=======================================================

    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    public SnowflakeIdGenerator(long workerId, long dataCenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("Worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("DataCenter Id can't be greater than %d or less than 0", maxDataCenterId));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
    }

    // ==================================================Methods========================================================
    // 线程安全的获得下一个 ID 的方法
    public synchronized long nextId() {
        long timestamp = currentTime();
        //如果当前时间小于上一次ID生成的时间戳: 说明系统时钟回退过 - 这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出 即 序列 > 4095
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = blockTillNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }
        //上次生成ID的时间截
        lastTimestamp = timestamp;
        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - startTime) << timestampMoveBits) //
                | (dataCenterId << dataCenterIdMoveBits) //
                | (workerId << workerIdMoveBits) //
                | sequence;
    }

    // 阻塞到下一个毫秒 即 直到获得新的时间戳
    protected long blockTillNextMillis(long lastTimestamp) {
        long timestamp = currentTime();
        while (timestamp <= lastTimestamp) {
            timestamp = currentTime();
        }
        return timestamp;
    }

    // 获得以毫秒为单位的当前时间
    protected long currentTime() {
        return System.currentTimeMillis();
    }

    //====================================================Test Case=====================================================
    public static void main(String[] args) {
        SnowflakeIdGenerator idWorker = new SnowflakeIdGenerator(0, 0);
        for (int i = 0; i < 1000; i++) {
            long id = idWorker.nextId();
            System.out.println(Long.toBinaryString(id));
            System.out.println(id);
        }
    }
}
```



## 百度UidGenerator

[UidGenerator](https://github.com/baidu/uid-generator) 是百度开源的分布式ID生成器，基于于snowflake算法的实现



## 美团Leaf

[Leaf](https://github.com/Meituan-Dianping/Leaf) 是美团开源的分布式ID生成器，能保证全局唯一性、趋势递增、单调递增、信息安全，里面也提到了几种分布式方案的对比，但也需要依赖关系数据库、Zookeeper等中间件。



## Reflence

- [GAVIN's Blog]([https://gavinlee1.github.io/2017/06/28/%E5%B8%B8%E8%A7%81%E5%88%86%E5%B8%83%E5%BC%8F%E5%85%A8%E5%B1%80%E5%94%AF%E4%B8%80ID%E7%94%9F%E6%88%90%E7%AD%96%E7%95%A5%E5%8F%8A%E7%AE%97%E6%B3%95%E7%9A%84%E5%AF%B9%E6%AF%94/](https://gavinlee1.github.io/2017/06/28/常见分布式全局唯一ID生成策略及算法的对比/))
- [分布式唯一ID的几种生成方案](https://juejin.im/post/5b3a23746fb9a024e15cad79)


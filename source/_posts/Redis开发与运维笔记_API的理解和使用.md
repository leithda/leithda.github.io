---
title: Redis开发与运维笔记-API的理解和使用
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: 长歌
abbrlink: 4241361649
date: 2021-03-01 18:25:00
---

{% cq %}
Redis提供了5种数据结构，理解每种数据结构的特点对于Redis开发运维非常重要，同时掌握Redis的单线程命令处理机制，会使数据结构和命令的选择事半功倍
{% endcq %}
<!-- more -->

参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)
<hr>

# API的理解和使用

## 预备

### 全局命令
1. 查看所有键

```bash
keys *
```

2. 键总数

```bash
dbsize
```
> `dbsize`命令直接获取Redis内置的键总数，复杂度是**O(1)**。`keys`会获取所有键，复杂度是**O(n)**,当Redis中保存大量键时，线上环境禁止使用。

3. 检查键是否存在

```bash
exists key
```

4. 删除键

```bash
del key [key ...]
```

5. 键过期

```bash
expire key seconds
```

> `ttl`命令会返回键的过期时间。返回结果有以下几种：
> - **大于等于0的整数**：键剩余的过期时间
> - **-1**：键没有设置过期时间
> - **-2**：键不存在


6. 键的数据结构类型

```bash
type key
```

### 数据结构和内部编码
> type命令返回的是当前键的数据结构，它们分别是：`string、hash、list、set、zset`
> 实际上，每种数据结构都有自己底层的内部编码实现
> 可以使用`object encoding key`命令获取当前键使用的内部编码

{% asset_img redis的五种数据结构.png redis的五种数据结构 %}
{% asset_img Redis数据结构和内部编码.png Redis数据结构和内部编码 %}

  Redis这样设计有两点好处：1）改进内部编码时，对外的数据结构和命令没有影响。2）不同的内部编码具有不同的优势。如:ziplist比较节省内存。

### 单线程架构

> Redis使用了单线程架构和**I/O多路复用模型[^1]**来实现高性能的内存数据库服务

#### 单线程模型

  Redis使用单线程处理命令，所以不会存在两条命令同时执行的情况，避免了并发问题。

#### 为什么单线程这么快
  通常来讲，单线程处理能力要比多线程差，为什么Redis这么快？
  1. 纯内存访问
  2. 非阻塞I/O，Redis使用**epoll[^2]**作为I/O多路复用技术的实现
  3. 单线程避免了线程切换和竞争产生的消耗

> 单线程简化了数据结构和算法的实现，避免了线程切换和线程竞争的消耗。但是存在如下问题：单线程模型对每条命令的执行时间是有要求的。如果某个命令执行时间过长，会造成其他命令的阻塞。

## 字符串

> 字符串是Redis最基本的数据结构，字符串类型的值实际可以是字符串(简单的字符串、复杂的字符串[如Json、xml等]、数字[整数、浮点数]、甚至是二进制[图片、音频、视频])，但是值最大不能超过512M

### 命令
#### 常用命令
1. 设置值

  ```bash
  SET key value [expiration EX seconds|PX milliseconds] [NX|XX]
  ```
  - **EX seconds**：秒级过期时间，功能等同`SETEX key seconds value`命令
  - **PX milliseconds**：毫秒级过期时间
  - **NX**：键不存在则设置成功，功能等同`SETNX key value`命令
  - **XX**：键存在则设置成功

  > `setnx`在多个客户端执行时，只有一个客户端才能成功，可以作为分布式锁的一种实现方式。[Redis官方方案](https://redis.io/topics/distlock)。

2. 获取值

  ```bash
  GET key
  ```
  > 如果键不存在，返回`nil`

3. 批量设置值

```bash
MSET key value [key value ...]
```

4. 批量获取值

```bash
MGET key [key ...]
```

> 批量操作可以节省网络开销时间。

5. 计数

```bash
INCR key
INCRBY key increment
DECR key
DECRBY key decrement
INCRBYFLOAT key increment
```
> 编程语言中常用CAS机制实现计数功能，存在一定CPU开销。Redis使用单线程机制则不存在这个问题

#### 不常用命令

1. 追加值 `APPEND key value`
2. 字符串长度 `STRLEN key`
3. 设置并返回值 `GETSET key value`
4. 设置指定位置的字符 `SETRANGE key offset value`
5. 获取部分字符串 `GETRANGE key start end`

### 内部编码

> 字符串的内部编码有三种：
> 1. **int**：8个字节的长整型
> 2. **embstr**：小于等于39个字节的字符串
> 3. **raw**：大于39个字节的字符串

### 典型使用场景

1. 缓存功能

```java
UserInfo getUserInfo(long id){
  String userInfoKey = "user:info" + id;
  String value = redis.get(userInfoKey);
  UserInfo userInfo;
  if(value != null){
    userInfo = deserialize(value);
  }else{
    userInfo = mysql.get(id);
    if(userInfo != null){
      redis.setex(userInfoKey, 3600, serialize(userInfo));
    }
  }
  return userInfo;
}
```

2. 计数

```java
long incrVideoCount(long id){
  key = "video:playCount:" + id;
  return redis.incr(key);
}
```

3. 共享Session

  将Session保存在Redis实现多端共享。

4. 限速

  ```java
  // 60s内短信发送不得超过5次
  boolean phoneVerify(String phoneNumber){
    key = "shortMsg:limit:" + phoneNumber;
    isExists = redis.set(key,1,"EX 60","NX");
    if(isExists != null || redis.incr(key) <= 5){
      // 通过 
    }else{
      // 限速 
    }
  }
  ```

## 哈希

### 命令
1. 设置值
2. 获取值 `HGET key field`
3. 删除field  `HDEL key field [field ...]`
4. 计算field个数 `HLEN key`
5. 批量设置或获取field-value

  ```bash
  HMGET key field [field ...]
  HMSET key field value [field value ...]
  ```

6. 判断field是否存在 `HEXISTS key field`
7. 获取所有的field `HKEYS key`
8. 获取所有的value `HVALS key`
9.  获取所有的field-value  `HGETALL key`
10. hincrby hincrbyfloat

  ```bash
  HINCRBY key field increment
  HINCRBYFLOAT key field increment
  ```

11. 计算value的字符串长度 `HSTRLEN key field`

## 列表

### 命令

| 操作类型 | 操作                                            |
| -------- | ----------------------------------------------- |
| 添加     | `rpush`、`rpushx`、`lpush`、`lpushx`、`linsert` |
| 查       | `lrange`、`lindex`、`llen`                      |
| 删除     | `lpop`、`rpop`、`lrem`、`ltrim`                 |
| 修改     | `lset`                                          |
| 阻塞操作 | `blpop`、`brpop`                                |

#### 添加

- 从列表右侧添加：` RPUSH key value [value ...]`
- 当列表存在时，从右侧添加：`RPUSHX key value`
- 在首个指定元素pivot(前/后)添加元素:`LINSERT key BEFORE|AFTER pivot value`

#### 查

- 获取指定范围内的列表元素:`LRANGE key start stop`
- 根据索引获取元素:`LINDEX key index`
- 获取列表长度:`LLEN key`

#### 删除

- 从列表左侧弹出元素:`LPOP key`
- 删除指定元素:`LREM key count value`
  - **count = 0**:删除所有等于指定值的元素
  - **count < 0**:从右到左删除count个元素
  - **count > 0**:从左到右删除count个元素
- 删除指定范围之外的元素:`LTRIM key start stop`

#### 修改

- 修改指定索引的元素:`LSET key index value`

#### 阻塞操作
- `BLPOP key [key ...] timeout`
- `BRPOP key [key ...] timeout`
  - `lpop/rpop`的阻塞版本，当列表内无元素时，客户端会阻塞等待，直到有元素可以弹出

> - 如果是多个键，brpop会从左到右遍历键，一旦有一个键可以弹出元素，客户端立即返回。
> - 多个客户端阻塞获取同一个key，最先执行获取的最先获取到

### 内部编码

列表类型的内部编码有两种：
- `ziplist(压缩列表)`：元素个数小于`list-max-ziplist-entries(512)`,同时列表中每个元素的值都小于`list-max-ziplist-value(64字节)`,Redis会使用`ziplist`作为`list`的内部编码来减少内存的使用。
- `linkedlist(链表)`: 不满足ziplist的条件时，使用`linkedlist`作为`list`的内部编码实现

> Redis3.2版本提供了quicklist内部编码，简单地说它是以一个ziplist为节点的linkedlist，它结合了ziplist和linkedlist两者的优势，为列表类型提供了一 种更为优秀的内部编码实现，它的设计原理可以参考Redis的另一个作者 **Matt Stancliff的博客[^3]**

### 使用场景

1. 消息队列
2. 文章列表
   1. 文章有三个属性：title、timestamp、content

    ```bash
    hmset acticle:1 title xx timestamp 1476536196 content xxxx
    ...
    hmset acticle:k title yy timestamp 1476512536 content yyyy
    ...
    ```
   2. 添加文章，key是`user:{id}:articles`

    ```bash
    lpush user:1:acticles article:1 article:3
    ...
    lpush user:k:acticles article:5
    ...
    ```
   3. 分页获取文章,获取id为1的前10篇文章

    ```bash
    articles = lrange user:1:articles 0 9
    for article in {articles}
      hgetall {article}
    ```
  
  > - 分页获取文章个数过多时，多次hgetall可能存在性能问题
  > - `lrange` 在列表两端时性能较好，列表数据过大时，可以考虑做二次拆分减小列表长度

> 列表使用场景可参考以下口诀：
> - lpush+lpop=Stack(栈)
> - lpush+rpop=Queue(队列)
> - lpsh+ltrim=Capped Collection(有限集合)
> - lpush+brpop=Message Queue(消息队列)

## 集合

### 命令

#### 集合内操作
1. 添加元素 `SADD key member [member ...]`
2. 删除元素 `SREM key member [member ...]`
3. 计算元素个数 `SCARD key`
4. 判断元素是否在集合中 `SISMEMBER key member`
5. 随机从集合中返回指定个数元素 `SRANDMEMBER key [count]`
6. 从集合随机弹出元素 `SPOP key [count]`
7. 获取所有元素 `SMEMBERS key`
   - 元素过多存在阻塞可能，可以使用`SSCAN key cursor [MATCH pattern] [COUNT count]`来完成

#### 集合间操作
1. 求多个集合的交集: `SINTER key [key ...]`
2. 求多个集合的并集: `SUINON key [key ...]`
3. 求多个集合的差集: `SDIFF key [key ...]`
4. 将交集、并集、差集的结果保存: `SINTERSTORE/SUINONSTORE/SDIFFSTORE destination key [key ...]`

### 内部编码
- `intset(整数集合)`: 集合中元素为整数且个数小于`set-max-intset-entries(512)`时
- `hashtable(哈希表)`

### 使用场景
  集合类型比较经典的使用场景是标签(tag)。

1. 给用户添加标签

```bash
sadd user:1:tags tag1 tag2 tag5
sadd user:2:tags tag2 tag3 tag5
...
sadd user:k:tags tag1 tag2 tag4
...
```

2. 给标签添加用户

```bash
sadd tag1:users user:1 user:3
sadd tag2:users user:1 user:2 user:3
...
sadd tagk:users user:1 user:2
...
```

> 用户和标签的关系应该在一个事务内执行。

3. 删除用户下的标签

```bash
srem user:1:tags tag1 tag5
...
```

4. 删除标签下的用户

```bash
srem tag1:users user:1
srem tag5:users user:1
...
```

5. 计算用户共同感兴趣的标签

```bash
sinter user:1:tags user:2:tags
```

> 集合使用场景可以按照如下方法判断：
> - sadd=Tagging(标签)
> - spop/srandmember=Random item(生成随机数，比如抽奖)
> - sadd+sinter=Social Graph(社交需求)

## 有序集合

> 和集合不同是，Sorted Set为每个元素设置一个`score`作为排序的依据。

> 列表、集合和有序集合的区别：

> |数据结构|是否允许重复|是否有序|有序实现方式|应用场景|
> |---|---|---|---|---|
> |列表|是|是|索引下标|时间轴、消息队列等|
> |集合|否|否|无|标签、社交等|
> |有序集合|否|是|分值|排行榜系统、社交等|

### 命令

#### 集合内
1. 添加成员 `zadd key score member [score member ...]`
2. 计算成员个数 `zcard key`
3. 计算某元素分数 `zscore key member`
4. 计算成员的排名

```bash
zrank key member
zrevrank key member
```

5. 删除成员 `zrem key member [member ...]`
6. 增加成员的分数 `zincrby key increment member`
7. 返回指定排名范围的成员

```bash
zrange key start end [withscores] # 从低到高
zrevrange key start end [withscores]  # 从高到低
```

8. 返回指定分数范围的成员

```bash
zrangebyscore key min max [withscores] [limit offset count] # 从低到高
zrevrangebyscore key max min [withscores] [limit offset count]  # 从高到低
```

9. 返回指定分数范围成员个数 `zcount key min max`
10. 删除指定排名内的升序元素 `zremrangebyrank key start end`
11. 删除指定分数范围的成员 `zremrangebyscore key min max`

#### 集合间

1. 交集 `zinterstore destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum|min|max]`
   - **destination**:交集计算结果保存到这个键
   - **numkeys**:需要做交集计算键的个数
   - **key\[key...\]**:需要做交集计算的键
   - **weights weight\[weight...\]**:每个键的权重，在做交集计算时，每个键中的每个member会将自己分数乘以这个权重，每个键的权重默认是1
   - **aggregate sum|min|max**:计算成员交集后，分值可以按照sum(和)、 min(最小值)、max(最大值)做汇总，默认值是sum


2. 并集 `zunionstore destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum|min|max]`
  - 参数与交集参数一致。

### 内部编码
- `ziplist(压缩列表)`
- `skiplist(跳跃表)`

### 使用场景

  排序集合典型的应用场景就是排行榜系统。例如视频网站需要对用户上传的视频做排行榜，榜单的维度可能是多个方面的:按照时间、按照播放数量、按照获得的赞数。

1. 添加用户赞数

```bash
zadd user:ranking:2016_03_15 mike 3 # mike 上传视频 获得了3个赞
zincrby user:ranking:2016_03_15 mike 1 # mike的视频后续又获得了一个赞
```

2. 取消用户赞数

```bash
zrem user:ranking:2016_03_15 mike # 用户mike视频注销了
```

3. 展示获取赞数最多的十个用户

```bash
zrevrangebyrank user:ranking:2016_03_15 0 9
```

4. 展示用户信息以及用户分数
```bash
hgetall user:info:tom
zscore user:ranking:2016_03_15 tom 
zrank user:ranking:2016_03_15 tom
```

## 键管理

### 单个键管理

1. 键重命名 `rename key newkey`
- 键的新名称存在时，值会被覆盖，可以使用`renamenx key newkey`命令避免
- 重命名期间会执行`del`命令删除旧的键，当键对应的值比较大时存在阻塞风险。

2. 随机返回一个键 `randomkey`
3. 键过期 
除了expire、ttl命令以外，Redis还提供了 expireat、pexpire、pexpireat、pttl、persist等一系列命令
- **expire key seconds**:键在seconds秒后过期
- **expireat key timestamp**:键在秒级时间戳timestamp后过期
- **pttl**: 查询键的剩余时间(毫秒级)
- **pexpire key milliseconds**:键在milliseconds毫秒后过期
- **pexpireat key milliseconds-timestamp**键在毫秒级时间戳timestamp后过期
- 关于Redis相关过期命令，需要注意以下几点：
  - 如果expire key的键不存在，返回结果为0
  - 如果过期时间为负值，键会立即被删除，犹如使用del命令一样
  - persist命令可以将键的过期时间清除
  - 对于字符串类型键，执行set命令会去掉过期时间，这个问题很容易在开发中被忽视
  - Redis不支持二级数据结构(例如哈希、列表)内部元素的过期功能
  - setex命令作为set+expire的组合，不但是原子执行，同时减少了一次 网络通讯的时间

4. 迁移键

- move `move key db`
- dump+restore
  - 在源Redis上，执行`dump key`将对应键值序列化，格式采用RDB格式
  - 在目标Redis上，restore命令将上面序列化的值进行复原，其中ttl参数代表过期时间，如果ttl=0代表没有过期时间
  - 整个过程非原子操作。且开启了两个客户端分别进行操作
- migrate `migrate host port key|"" destination-db timeout [copy] [replace] [keys key [key ...]]`
  - **host**:目标Redis的IP地址
  - **port**:目标Redis的端口
  - **key|""**:Redis3.0.6后，支持迁移多个键，此处填`""`
  - **destination-db**:目标Redis的数据库索引
  - **timeout**:迁移的超时时间(单位为毫秒)
  - **\[copy\]**:如果添加此选项，迁移后并不删除源键
  - **\[replace\]**:如果添加此选项，migrate不管目标Redis是否存在该键都会 正常迁移进行数据覆盖
  - **\[keys key\[key...\]\]**:迁移多个键，例如要迁移key1、key2、key3，此处填 写`keys key1 key2 key3`

> move、dump+restore、migrate三个命令比较

|命令|作用域|原子性|支持多个键|
|---|---|---|---|
|move|Redis实例内部|是|否|
|dump + restore|Redis实例之间|否|否|
|migrate|Redis实例之间|是|是|

### 遍历键

#### 全量遍历键
> `keys pattern`
> pattern支持glob风格的通配符
> - **\***：匹配任意字符
> - **\.**：匹配一个字符
> - **\[\]**:匹配部分字符,`[1,3]`代表1或3，`[1-10]`代表匹配1-10中任意一个数字
> - **\\x**:用来做转义，例如要匹配`*`、`.`等

#### 渐进式遍历
> `scan cursor [match pattern] [count number]`
> - **cursor**:游标，从0开始，遍历至游标为0结束。每次遍历会返回当前游标的值
> **match pattern**:可选参数，它的作用的是做模式的匹配，这点和keys的 模式匹配很像
> **count number**:可选参数，它的作用是表明每次要遍历的键个数，默认值是10，此参数可以适当增大

> 除了`scan`外，Redis还提供了面向哈希类型、集合类型、有序集合的扫 描遍历命令，`hscan、sscan、zscan`
> ```java
> String key = "myset";
> // 定义 pattern
> String pattern = "old:user*";
> // 游标每次从0开始
> String cursor = "0"
> while(true){
>   ScanResult scanResult = redis.sscan(key,cursor,pattern);
>   List elements = scanResult.getResult();
>   if(elements != null && element.size() >0 ){
>     // 批量删除
>     redis.srem(key,elements)
>   }
>   // 获取新的游标
>   cursor = scanResult.getStringCursor();
>   // 如果游标为0表示遍历结束
>   if("0".equals(cursor)){
>   break;
>   }
> }
> ```
> - 当scan过程中，如果发生键值的修改、删除、增加等，可能会发生问题。

### 数据库管理

1. 切换数据库 `select dbIndex`
2. 清除数据库 `flushdb/flushall`

## 本节内容回顾
1. Redis提供了5种数据结构，每种数据结构都有多种内部编码实现。
2. 纯内存存储、IO多路复用技术、单线程架构是造就Redis高性能的三个因素。
3. 由于Redis的单线程架构，所以需要每个命令能被快速执行完，否则会存在阻塞Redis的可能，理解Redis单线程命令处理机制是开发和运维Redis的核心之一
4. 批量操作(例如mget、mset、hmset等)能够有效提高命令执行的效率，但要注意每次批量操作的个数和字节数
5. 了解每个命令的时间复杂度在开发中至关重要，例如在使用keys、hgetall、smembers、zrange等时间复杂度较高的命令时，需要考虑数据规模对于Redis的影响
6. persist命令可以删除任意类型键的过期时间，但是set命令也会删除字符串类型键的过期时间，这在开发时容易被忽视
7. move、dump+restore、migrate是Redis发展过程中三种迁移键的方式，其中move命令基本废弃，migrate命令用原子性的方式实现了dump+restore，并且支持批量操作，是Redis Cluster实现水平扩容的重要工具。
8. scan命令可以解决keys命令可能带来的阻塞问题，同时Redis还提供了hscan、sscan、zscan渐进式地遍历hash、set、zset。


[^1]:I/O多路复用模型详见 [Redis 和 IO 多路复用](https://www.cnblogs.com/john8169/p/9780484.html)  
[^2]:epoll模型详见 [epoll](https://baike.baidu.com/item/epoll/10738144?fr=aladdin)  
[^3]: [https://matt.sh/redis-quicklist](https://matt.sh/redis-quicklist)
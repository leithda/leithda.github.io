---
title: Redis开发与运维笔记_小功能大用处
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: leithda
abbrlink: 2425201134
date: 2021-05-25 22:40:00
---

{% cq %}
Redis提供的5种数据结构已经足够强大，但除此之外，Redis还提供了诸如慢查询分析、功能强大的Redis Shell、Pipeline、事务与Lua脚本、 Bitmaps、HyperLogLog、发布订阅、GEO等附加功能，这些功能可以在某些场景发挥重要的作用
{% endcq %}
<!-- more -->

参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)
<hr>

# 小功能大用处

## 慢查询分析

  客户端执行一条命令分为如下四个阶段：1、发送命令 2、命令排队 3、命令执行 4、返回结果  
  慢查询值统计步骤3的时间，所有没有慢查询并不代表客户端没有超时问题。

### 慢查询的两个参数

- **slowlog-log-slower-than**：慢查询阈值，单位是微妙，默认值10000
  - `=0`时会记录所有命令
  - `<0`时不记录任何命令
- **slowlog-max-len**：慢查询日志列表的最大长度,默认值128

> 通过 `config set`命令动态调整Redis配置
```bash
config set slowlog-log-slower-than 20000 
config set slowlog-max-len 1000
config rewrite  # 持久化到本地配置文件
```

- redis的慢查询日志存放在redis的列表中。慢查询日志项由标识ID、发生时间戳、命令耗时、执行命令四部分组成。可以使用如下api进行查询：
  - 获取慢查询日志 `slowlog get [n]`
  - 获取慢查询日志列表当前的长度 `slowlog len`
  - 慢查询日志重置 `slowlog reset`

### 最佳实践

1. `slowlog-max-len`:线上建议调大，避免溢出丢失慢查询记录，慢查询日志会对长命令进行截取，不会占用过多内存
2. `slowlog-log-slower-than`:默认值10毫秒为慢查询，对于高并发场景，如果命令执行时间超过1毫秒，那么Redis最多可支撑OPS不到1000。建议高OPS场景的Redis设置为1毫秒。
3. 慢查询只记录命令执行时间，客户端请求超时时，可根据慢查询记录分析是否是慢查询导致的超时。
4. 由于慢查询日志是一个先进先出的队列，慢查询较多时可能导致丢失部分慢查询命令，可以定时通过`slowlog get`命令将慢查询日志持久化到其他存储中(如MySQL)，然后制作可视化界面进行展示。

## Redis Shell
  Redis提供了redis-cli、redis-server、redis-benchmark等Shell工具

### redis-cli
1. `-r` 将命令执行多次
2. `-i` 每隔几秒执行一次，需配合`-r`一起使用
3. `-x` 从标准输入`stdin`读取数据作为redis-cli的最后一个参数
4. `-c` 连接Redis Cluster节点时需要使用的，-c选项可以防止moved和ask异常
5. `-a` 如果设置了密码可以使用-a选项，不用再手动输入`auth`命令
6. `--scan和--pattern` 相当于`scan`命令
7. `--slave` 将当前客户端模拟成当前Redis实例的从节点
8. `--rdb` 请求Redis实例生成并发送RDB持久化文件，保存在本地，可以使用它做持久化文件的定期备份
9. `--pipe` 将命令封装成Redis通信协议定义的数据格式，批量发送 给Redis执行.`echo -en '*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n*2\r\n$4\r\nincr\r\ n$7\r\ncounter\r\n' | redis-cli --pipe`
10. `--bigkeys` 使用`scan`命令对Redis的键进行采样，从中找到内存占用比较大的键值，这些键可能是系统的瓶颈
11. `--eval` 用于执行Lua脚本
12. `--latency` 
  - `--latency`：测试客户端到目标Redis实例的延迟
  - `--latency-history`: 每15s输出一次延迟信息
  - `--latency-dist`:以图表形式展示延迟信息

13. `--stat`: 实时获取Redis的重要统计信息
14. `--raw`和`--no-raw`: `--raw`返回格式花数据，`--no-raw`返回原始格式数据。

### redis-server

redis-server 除了启动Redis实例外，还有一个`--test-memory`，用于检测当前机器能否稳定的分配指定容量的内存给Redis.  
`redis-server --test-memory 1024`,检测当前操作系统能否提供1G的内存给Redis

### redis-benchmark
  redis-benchmark可以为Redis做基准性能测试，它提供了很多选项帮助开发和运维人员测试Redis的相关性能，下面分别介绍这些选项。
1. `-c`: 客户端的并发数量，默认50
2. `-n`: 客户端请求总数，默认100000
3. `-q`: 仅显示基准测试的`requests per second`
4. `-r`: 生成随机个数个键，`-r 10000`，代表仅处理生成键的后四位，如`"key:000000004580"、"key:000000004519"`
5. `-p`: 代表每个请求pipeline的数量
6. `-k`: 客户端是否使用keepalive，1：使用，0：不使用，默认为1
7. `-t`: 可以对指定命令进行基准测试，如:`-t get,set`
8. `--csv`: 输出csv格式结果


## Pipeline
  将多条命令封装成一条pipeline命令，节省发送多次命令造成的网络耗时，提供性能可以考虑

原生批量命令和Pipeline对比
- 原生批量命令是原子的，Pipeline是非原子的
- 原生批量命令是支持多个key，Pipeline是支持使用多个命令
- 原生批量命令是Redis服务端支持的，而Pipeline需要服务端和客户端共同实现

## 事务与Lua

### 事务
Redis支持的事务非常简单，`multi`代表事务开始，`exec`代表事务结束。
Redis的错误处理机制：
- 命令错误，事务不进行
- 运行时错误，命令格式正确，但是语义错误，这种事务执行到语义错误的位置时才会停下，而之前的命令已经执行
Redis提供`watch key`命令来保证事务期间，key没有被更改.

### Lua用法简介
Lua是一种脚本语言，它是用C语言实现的。

#### 数据类型及其处理逻辑
Lua提供了`booleans(布尔)、numbers(数值)、strings(字符串)、tables(表格)`几种数据类型
```lua
--### 基本数据类型 ###---
local strings val = "world" -- local代表val是局部变量，没有则是全局变量
print(val)  -- world

local tables myArray = {"redis", "jedis", true, 88.0}
print(myArray[3]) -- true

--### for 的用法 ###---
for i = 1, #myArray
do
  print(myArray[i])
end

for index,value in ipairs(myArray)
do
  print(index)
  print(value)
end

--### while 的用法 ###---
local int sum = 0
local int i = 0
while(i < #myArray)
do
  sum = sum + i;
  i = i + 1
end

--### if 的用法 ###---
if myArray[1] == "jedis"
then
  print("true")
  break
else
  -- do nothing
end

--### 实现hash ###---
local tables user_1 = {age = 28, name = "tom"}
print("user_1 age is " .. user_1["age"])  -- user_1 age is 28 其中..是字符串连接

for key,value in pairs(user_1) 
do 
  print(key .. value)
end
```

#### 函数定义

```lua
function contact(str1, str2)
  return str1..str2
end

print(contact("hello"," world"))  --"hello world"
}
```

### Redis与Lua

#### 在Redis中使用Lua
1. eval `eval 脚本内容 key个数 key列表 参数列表`
```bash
127.0.0.1:6379> eval 'return "hello " .. KEYS[1] .. ARGV[1]' 1 redis world
"hello redisworld"
```
- 脚本内容较长时，可以使用`redis-cli --eval`直接执行文件

1. evalsha

- 加载脚本
```bash
# redis-cli script load "$(cat lua_get.lua)"
"7413dc2440db1fea7c0a0bde841fa68eefaf149c"
```

- 执行脚本  `evalsha 脚本SHA1值 key个数 key列表 参数列表`
```bash
127.0.0.1:6379> evalsha 7413dc2440db1fea7c0a0bde841fa68eefaf149c 1 redis world "hello redisworld"
```

#### Lua的Redis API

```lua
redis.call("set", "hello", "world")
redis.call("get", "hello")
```

- 除此之外，还可以使用redis.pcall函数实现对Redis的调用。`redis.call`如果报错脚本直接返回错误。`redis.pcall`会忽略错误继续执行。

### 案例
Lua的优点：Lua脚本是原子执行的。使用Lua脚本可以定制命令。Lua脚本将多条命令打包，有效减少了网络开销。  

- 当前列表记录着热门用户的ID

```bash
127.0.0.1:6379> lrange hot:user:list 0 -1 
1) "user:1:ratio"
2) "user:8:ratio"
3) "user:3:ratio"
4) "user:99:ratio" 
5) "user:72:ratio"
```

- user:{id}:ratio代表着用户的热度值，它本身是一个字符串类型的键

```bash
127.0.0.1:6379> mget user:1:ratio user:8:ratio user:3:ratio user:99:ratio user:72:ratio
1) "986" 
2) "762" 
3) "556" 
4) "400" 
5) "101"
```

现将列表内用户的热度值+1，并且保证是原子执行，可以使用Lua脚本  

```lua
local mylist = redis.call("lrange", KEYS[1], 0, -1)
local count = 0
for index,key in ipairs(mylist)
do
  redis.call("incr",key)
  count = count + 1
end
return count
```

将上述脚本写入`lrange_and_mincr.lua`文件中，并执行如下操作：  

```bash
redis-cli --eval lrange_and_mincr.lua hot:user:list 
(integer) 5
```

### Redis如何管理Lua

1. `script load script`
2. `script exists sha1 [sha1 ...]`
3. `script flush` 清除已经加载的脚本
4. `script kill` 停止正在执行的脚本，当脚本进行写操作时，无法通过此命令终止，需要使用`shutdown save`关闭Redis实例。


## Bitmaps

- Bitmaps不是一种新的数据结构，它就是在字符串的基础上提供了对位的操作。
- Bitmaps提供了一套命令，可以将字符串想象成以位为单位的数组，数组的每个元素只能存储0和1

### 命令

1. 设置值 `setbit key offset value`
2. 获取值 `gitbit key offset`
3. 获取Bitmaps指定范围值为1的个数 `bitcount [start][end]`
4. Bitmaps间的操作  `bitop op destkey key[key....]`
    bitop是一个复合操作，它可以做多个Bitmaps的and(交集)、or(并 集)、not(非)、xor(异或)操作并将结果保存在destkey中

5. 计算Bitmaps中第一个值为targetBit的偏移量 `bitpos key targetBit [start] [end]`

### Bitmaps分析
  Bitmaps在存储较多有效数据时才能发挥节省内存的效果。

## HyperLogLog
HyperLogLog[^1]不是一种新的数据结构，而是一种基数算法。通过HyperLogLog可以利用极小的空间完成独立总数的统计。HyperLogLog提供了3个命令`pfadd`、`pfcount`、`pfmerge`。

### 添加
```bash
pfadd key element [element ...]
```

### 计算独立用户数
```bash
pfcount key [key ...]
```

### 合并
```bash
pfmerge destkey sourcekey [sourcekey ...]
```

> 结构选型需要确认以下两点：
> - 只为了计算独立总数，不需要获取单条数据
> - 可以容忍一定误差率

## 发布订阅
Redis提供了基于“发布/订阅”模式的消息机制，此种模式下，消息发布者和订阅者不进行直接通信，发布者客户端向指定的频道(channel)发布消息，订阅该频道的每个客户端都可以收到该消息

### 命令
1. 发布消息 `publish channel message`
2. 订阅消息 `subscribe channel [channel ...]`
3. 取消订阅 `unsubscribe [channel [channel ...]]`
4. 按照模式订阅或取消订阅

```bash
psubscribe pattern [pattern...]
punsubscribe [pattern [pattern ...]]
```

5. 查询订阅
   1. 查看活跃的频道 `pubsub channels [pattern]`
   2. 查看频道订阅数 `pubsub numsub [channel ...]`
   3. 查看模式订阅数 `pubsub numpat`

### 使用场景
聊天室、公告牌及服务之间利用消息解耦都可以使用发布订阅模式

## Geo
Redis3.2版本提供了GEO(地理信息定位)功能，支持存储地理位置信 息用来实现诸如附近位置、摇一摇这类依赖于地理位置信息的功能，对于需 要实现这些功能的开发者来说是一大福音。GEO功能是Redis的另一位作者 Matt Stancliff[^2]借鉴NoSQL数据库Ardb[^3]实现的，Ardb的作者来自中国，它提供了优秀的GEO功能。

### 增加地理位置信息
```bash
geoadd key longitude latitude member [longitude latitude member ...]
```
- longitude、latitude、member分别是该地理位置的经度、纬度、成员

### 获取地理信息
```bash
geopos key member [member ...]
```
- 会返回对应位置的经纬度

### 获取两个地理位置之间的距离
```bash
geodist key member1 member2 [unit]
```
- unit:返回结果的单位:m(meters)代表米，km(kilometers)代表公里，mi(miles)代表英里，ft(feet)代表尺

### 获取指定位置范围内的地理信息位置集合
```bash
georadius key longitude latitude radiusm|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
georadiusbymember key member radiusm|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
```
- georadius以经纬度为中心，georadiusbymember以成员为中心
- **withcoord**:返回结果中包含经纬度
- **withdist**:返回结果中包含距中心店的距离
- **withhash**:返回结果中包含gethash
- **COUNT count**:指定返回结果数量
- **asc|desc**:按照距中心的距离做升序或降序
- **store key**:将返回结果的地理信息保存到指定键。使用Sorted Set保存(城市)
- **storedist key**:将返回结果距中心距离保存到指定键

```bash
127.0.0.1:6379> georadiusbymember cities:locations beijing 150 km # 距离北京150km内的城市
1) "beijing"
2) "tianjin"
3) "tangshan"
4) "baoding"
```

### 获取 geohash
```bash
geohash key member [member ...]
```

- **geohash**[^4] 的数据类型为zset，Redis将所有地理位置信息存放在zset中
- 字符串越长，精确度越准确
- 两个字符串越相似，它们之间的距离越近
- geohash编码和经纬度可以相互转换

### 删除地理位置
```bash
zrem key member
```
- geo没有提供删除位置的命令，但是Geo的底层实现是zset，所以可以借用zrem命令实现对地理位置信息的删除。


## 本章重点回顾

- 慢查询中的两个重要参数`slowlog-log-slower-than`和`slowlog-max-len`
- 慢查询不包含命令网络传输和排队时间
- 有必要将慢查询定期存放
- redis-cli的一些重要的选项，例如`--latency`、`–-bigkeys`、`-i`和`-r`组合
- redis-benchmark 的使用方法和重要参数
- Pipeline可以有效减少RTT次数，但每次Pipeline的命令数量不能无节制
- Redis可以使用Lua脚本创造出原子、高效、自定义命令组合
- Redis执行Lua脚本有两种方法:eval和evalsha
- Bitmaps可以用来做独立用户统计，有效节省内存
- Bitmaps中setbit一个大的偏移量，由于申请大量内存会导致阻塞
- HyperLogLog虽然在统计独立总量时存在一定的误差，但是节省的内存量十分惊人
- Redis的发布订阅机制相比许多专业的消息队列系统功能较弱，不具备堆积和回溯消息的能力，但胜在足够简单
- Redis3.2提供了GEO功能，用来实现基于地理位置信息的应用，但底层实现是zset。


[^1]:HyperLogLog的算法是由[Philippe Flajolet](https://en.wikipedia.org/wiki/Philippe_Flajolet)在 The analysis of a near-optimal cardinality estimation algorithm 这篇论文中提出，读者如果有兴趣 可以自行阅读。

[^2]:[https://matt.sh/](https://matt.sh/)

[^3]:[https://github.com/yinqiwen/ardb](https://github.com/yinqiwen/ardb)

[^4]:[https://en.wikipedia.org/wiki/Geohash](https://en.wikipedia.org/wiki/Geohash)


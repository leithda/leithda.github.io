---
title: Redi的string类型和bitmap
abbrlink: 1289357189
date: 2020-06-14 22:56:28
categories:
  - Redis
tags:
  - Redis
  - NoSQL
author: 长歌
---


{% cq %}
string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。

string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。

string 类型是 Redis 最基本的数据类型，string 类型的值最大能存储 512MB。
{% endcq %}
<!-- More -->





# Redis的string和bitmap

## 概述

> 字符串类型是Redis中最为基础的数据存储类型，它在Redis中是二进制安全的，这便意味着该类型可以接受任何格式的数据，如JPEG图像数据或Json对象描述信息等。在Redis中字符串类型的Value最多可以容纳的数据长度是512M。  
> bitmap 操作是redis中一种强大的操作类型  
> 举例：  
> 1. 我们可以使用24bit来代表24小时，用户在对应时间内登录则将对应bit位设置为1。相对于关系型数据库，redis占用内存小，效率更快。
> ```bash
> SETBIT leithda 1 1 # leithda 1点活跃
> SETBIT leithda 8 1 # leithda 8点活跃
> SETBIT leithda 18 1 # leithda 18点活跃
> BITCOUNT leithda 1 3 # 统计 1-3字节(即 1-24 bit位)上活跃的小时数
> ```
>
> 2. 统计 19年11月10日-19年11月12日在线人数(去重)，使用n bit位代表n个候选人，key名称代表时间
> ```bash
> SETBIT 20191110 1 1 # 20191110 1号候选人在线
> SETBIT 20191110 2 1 # 20191110 2号候选人在线
> SETBIT 20191111 1 1 # 20191111 1号候选人在线
> SETBIT 20191112 3 1 # 20191112 3号候选人在线
> BITOP OR destkey 20191110 20191111 20191112 # ”与“操作，去重
> BITCOUNT destkey 0 -1 # 统计10号-12号在线人数
> ```

## 相关命令

> 语法
>
> ```bash
> Last login: Sun Jun 14 16:30:24 on ttys003
> ➜  ~ redis-cli 
> 127.0.0.1:6379> help @string
> 
>   APPEND key value
>   summary: Append a value to a key
>   since: 2.0.0
> 
>   BITCOUNT key [start end]
>   summary: Count set bits in a string
>   since: 2.6.0
> 
>   BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]
>   summary: Perform arbitrary bitfield integer operations on strings
>   since: 3.2.0
> 
>   BITOP operation destkey key [key ...]
>   summary: Perform bitwise operations between strings
>   since: 2.6.0
> 
>   BITPOS key bit [start] [end]
>   summary: Find first bit set or clear in a string
>   since: 2.8.7
> 
>   DECR key
>   summary: Decrement the integer value of a key by one
>   since: 1.0.0
> 
>   DECRBY key decrement
>   summary: Decrement the integer value of a key by the given number
>   since: 1.0.0
> 
>   GET key
>   summary: Get the value of a key
>   since: 1.0.0
> 
>   GETBIT key offset
>   summary: Returns the bit value at offset in the string value stored at key
>   since: 2.2.0
> 
>   GETRANGE key start end
>   summary: Get a substring of the string stored at a key
>   since: 2.4.0
> 
>   GETSET key value
>   summary: Set the string value of a key and return its old value
>   since: 1.0.0
> 
>   INCR key
>   summary: Increment the integer value of a key by one
>   since: 1.0.0
> 
>   INCRBY key increment
>   summary: Increment the integer value of a key by the given amount
>   since: 1.0.0
> 
>   INCRBYFLOAT key increment
>   summary: Increment the float value of a key by the given amount
>   since: 2.6.0
> 
>   MGET key [key ...]
>   summary: Get the values of all the given keys
>   since: 1.0.0
> 
>   MSET key value [key value ...]
>   summary: Set multiple keys to multiple values
>   since: 1.0.1
> 
>   MSETNX key value [key value ...]
>   summary: Set multiple keys to multiple values, only if none of the keys exist
>   since: 1.0.1
> 
>   PSETEX key milliseconds value
>   summary: Set the value and expiration in milliseconds of a key
>   since: 2.6.0
> 
>   SET key value [expiration EX seconds|PX milliseconds] [NX|XX]
>   summary: Set the string value of a key
>   since: 1.0.0
> 
>   SETBIT key offset value
>   summary: Sets or clears the bit at offset in the string value stored at key
>   since: 2.2.0
> 
>   SETEX key seconds value
>   summary: Set the value and expiration of a key
>   since: 2.0.0
> 
>   SETNX key value
>   summary: Set the value of a key, only if the key does not exist
>   since: 1.0.0
> 
>   SETRANGE key offset value
>   summary: Overwrite part of a string at key starting at the specified offset
>   since: 2.2.0
> 
>   STRLEN key
>   summary: Get the length of the value stored in a key
>   since: 2.2.0
> 
> ```

### string型命令

#### SET

> `SET key value [expiration EX seconds | PX milliseconds] [NX|XX]`
>
> 设置指定key的值，EX和PX用于设定有效期单位，NX 当值不存在时设置值，XX当值存在时设置key的值

```bash
127.0.0.1:6379> SET k1 hello EX 5 NX    
OK
127.0.0.1:6379> TTL k1
(integer) 3
127.0.0.1:6379> TTL k1
(integer) -2
127.0.0.1:6379> GET K1
(nil)
```

#### SETNX

> `SETNX key value`
>
> 当key不存在时设置key的值；设置成功返回1，失败返回0

```bash
127.0.0.1:6379> SETNX k1 hello
(integer) 1
127.0.0.1:6379> SETNX k1 hi
(integer) 0
```

#### SETEX

> `SETEX key seconds value`
>
> 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)；key 如果存在会被覆盖

```bash
127.0.0.1:6379> SETEX k1 5 hello
OK
127.0.0.1:6379> TTL k1
(integer) 4
```

#### PSETEX

> `PSETEX key milliseconds value`
>
> 与SETEX类似，过期时间以毫秒为单位

```bash
127.0.0.1:6379> PSETEX k2 60000 world
OK
127.0.0.1:6379> TTL k2
(integer) 56
```

#### MSET

> `MSET key value [key value ...]`
>
> 同时设置一个或多个 key-value 对

```bash
127.0.0.1:6379> MSET k1 hello k2 world
OK
127.0.0.1:6379> GET k1
"hello"
127.0.0.1:6379> GET k2
"world"

```

#### MSETNX

> `MSETNX key value [key value ...]`
>
> 同时设置一个或多个key-value对，当且仅当所有的key不存在时

```bash
127.0.0.1:6379> MSET k1 hello k2 world
OK
127.0.0.1:6379> MSETNX k2 world k3 !
(integer) 0
127.0.0.1:6379> MSETNX k3 hi k4 redis
(integer) 1
127.0.0.1:6379> MGET k3 k4
1) "hi"
2) "redis"
```

#### APPEND

> `APPEND key value`
>
> 如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾.

```bash
127.0.0.1:6379> MGET k1 k2
1) (nil)
2) (nil)
127.0.0.1:6379> set k1 hello
OK
127.0.0.1:6379> append k1 " world"
(integer) 11
127.0.0.1:6379> append k2 "hi redis"
(integer) 8
127.0.0.1:6379> MGET k1 k2
1) "hello world"
2) "hi redis"
```

#### GET

> `GET key`
>
> 返回key的值，如果key不存在返回 `nil`。对非string类型使用会报错

```bash
127.0.0.1:6379> get k1
(nil)
127.0.0.1:6379> set k1 "hello world"
OK
127.0.0.1:6379> get k1
"hello world"
```

#### GETSET

> `GETSET key value`
>
> 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。

```bash
127.0.0.1:6379> GETSET k1 hello
(nil)
127.0.0.1:6379> GETSET k1 world
"hello"
127.0.0.1:6379> get k1
"world"
```

#### MGET

> 省略

#### STRLEN

> `STRLEN key`
>
> 返回 key 所储存的字符串值的长度。

```bash
127.0.0.1:6379> SET k1 hello
OK
127.0.0.1:6379> STRLEN k1
(integer) 5
```

### 数值型命令

#### INCR

> `INCR key`
>
> 将 key 中储存的数字值增一

```bash
127.0.0.1:6379> SET k1 1
OK
127.0.0.1:6379> SET k2 "hello world"
OK
127.0.0.1:6379> INCR k1
(integer) 2
127.0.0.1:6379> INCR k2
(error) ERR value is not an integer or out of range
127.0.0.1:6379> INCR k3
(integer) 1
127.0.0.1:6379> MGET k1 k2 k3
1) "2"
2) "hello world"
3) "1"
```



#### INCRBY

> `INCRBY key increment`
>
> 将 key 所储存的值加上给定的增量值（increment） 



#### INCRBYFLOAT

> `INCRBYFLOAT key increment`
>
> 将 key 所储存的值加上给定的浮点增量值（increment）



#### DECR

> `DECR key`
>
> 将 key 中储存的数字值减一



#### DECRBY

> `DECRBY key decrement`
>
> key 所储存的值减去给定的减量值（decrement）



### bitmap型命令

#### SETBIT

> `SETBIT key offset value`
>
> 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit).

```bash

```

#### GETBIT

> `GETBIT key offset`
>
> 对 key 所储存的字符串值，获取指定偏移量上的位(bit)

```bash

```

#### BITCOUNT

> `BITCOUNT key [start end]`
>
> 统计字符串被设置为1的个数。start和end分别代表起始/终止 字节索引

```basg

```

#### BITPOS

> `BITPOS key bit [start] [end] `
>
> 返回位图中第一个值为bit的二进制位的位置

```bash

```

#### BITOP

> `BITOP operation destkey key [key...]`
>
> 对一个或多个保存二进制位的字符串key进行位元操作,并将结果保存到destksy上
>
> operation可以是AND、OR、NOT、XOR这四种操作中的任意一种：
>
> ```bash
> 
> BITOP ADN destkey key [key...]对一个或者多个key求逻辑并，并将结果保存到destkey
> BITOP OR destkey key [key...]对一个或者多个key求逻辑或，并将结果保存到destkey
> BITOP XOR destkey key [key...]对一个或者多个key求逻辑异或，并将结果保存到destkey
> BITOP NOT destkey key [key...]对给定key求逻辑菲，并将结果保存到destkey
> ```

```bash

```

#### BITFIELD

> `TODO`


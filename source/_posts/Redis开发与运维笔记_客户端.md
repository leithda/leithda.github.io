---
title: Redis开发与运维笔记-客户端
categories:
  - Redis
tags:
  - Redis
author: 长歌
abbrlink: 3206834815
date: 2020-06-09 22:54:39
---
> 参考书籍：[Redis开发与运维](https://book.douban.com/subject/26971561/)

<!-- more -->




# 1. 客户端协议

建议参考书籍正文第4章-4.1的内容进行查看，或者查看文章[Redis-通信协议（protocol）](http://redisdoc.com/topic/protocol.html)



# 2. Java客户端 Jedis



## 2.1 获取Jedis

Maven项目中引入如下依赖

```xml
<dependency> 
  <groupId>redis.clients</groupId> 
  <artifactId>jedis</artifactId> 
  <version>${jedis.version}</version>
</dependency>
```



## 2.2 使用Jedis

```java
Jedis jedis = null; 
try {
  jedis = new Jedis("127.0.0.1", 6379);
	String hello = jedis.get("hello"); 
  System.out.println(hello);
} catch (Exception e) {
	logger.error(e.getMessage(),e); 
} finally {
	if (jedis != null) { 
    jedis.close();
	} 
}
```

- 注：Jedis本身没有提供序列化工具，如需序列化需要开发者引入第三方工具完成。以`protostuff`为例

  - 引入

    ```xml
    <protostuff.version>1.0.11</protostuff.version> 
    <dependency>
    	<groupId>com.dyuproject.protostuff</groupId> 
      <artifactId>protostuff-runtime</artifactId> 
      <version>${protostuff.version}</version>
    </dependency> 
    <dependency>
    	<groupId>com.dyuproject.protostuff</groupId> 
      <artifactId>protostuff-core</artifactId> 
      <version>${protostuff.version}</version>
    </dependency>
    ```

  - 定义实体类

    ```java
    // 俱乐部
    public class Club implements Serializable {
    	private int id;
    	private String name; 
      private String info; 
      private Date createDate; 
      private int rank;
    	// 相应的getter setter省略不占用篇幅
    }
    ```

  - 测试

    ```java
    ProtostuffSerializer protostuffSerializer = new ProtostuffSerializer();
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    
    // === 序列化 ===
    String key = "club:1";
    // 定义实体对象
    Club club = new Club(1, "AC", "米兰", new Date(), 1);
    // 序列化
    byte[] clubBtyes = protostuffSerializer.serialize(club); jedis.set(key.getBytes(), clubBtyes);
    
    // === 反序列化 ===
    byte[] resultBtyes = jedis.get(key.getBytes());
    // 反序列化[id=1, clubName=AC, clubInfo=米兰, createDate=Tue Sep 15 09:53:18 CST // 2015, rank=1]
    Club resultClub = protostuffSerializer.deserialize(resultBtyes);
    ```

## 2.3 Jedis连接池

​	Jedis提供了JedisPool这个类作为对Jedis的连接池，同时使用了Apache的 通用对象池工具common-pool作为资源的管理工具，下面是使用JedisPool操 作Redis的代码示例：

```java

// common-pool连接池配置，这里使用默认配置，后面小节会介绍具体配置说明 
GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig(); // 初始化Jedis连接池
JedisPool jedisPool = new JedisPool(poolConfig, "127.0.0.1", 6379);



Jedis jedis = null; try {
  // 1. 从连接池获取jedis对象
  jedis = jedisPool.getResource(); // 2. 执行操作
  jedis.get("hello");
} catch (Exception e) { 
  logger.error(e.getMessage(),e);
} finally {
  if (jedis != null) {
    // 如果使用JedisPool，close操作不是关闭连接，代表归还连接池
    jedis.close(); 
  }
}

```

- 这里关于连接池的配置不再赘述，通过下面代码简单介绍：

  ```java
  GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
  // 设置最大连接数为默认值的5倍 poolConfig.setMaxTotal(GenericObjectPoolConfig.DEFAULT_MAX_TOTAL * 5); 
  // 设置最大空闲连接数为默认值的3倍
  poolConfig.setMaxIdle(GenericObjectPoolConfig.DEFAULT_MAX_IDLE * 3); 
  // 设置最小空闲连接数为默认值的2倍
  poolConfig.setMinIdle(GenericObjectPoolConfig.DEFAULT_MIN_IDLE * 2); 
  // 设置开启jmx功能
  poolConfig.setJmxEnabled(true);
  // 设置连接池没有连接后客户端的最大等待时间(单位为毫秒) 
  poolConfig.setMaxWaitMillis(3000);
  ```



## 2.4 Jedis使用Pipeline

```java
public void mdel(List<String> keys) { 
  Jedis jedis = new Jedis("127.0.0.1"); // 1)生成pipeline对象
	Pipeline pipeline = jedis.pipelined(); // 2)pipeline执行命令，注意此时命令并未真正执行
	for (String key : keys) {
		pipeline.del(key); 
  }
	// 3)执行命令
	pipeline.sync(); 
  // List<Object> resultList = pipeline.syncAndReturnAll();
}
```

- 使用`pipeline.sync()`调用pipeline.也可以通过`pipeline.syncAndReturnAll()`调用并获取pipeline执行结果



## 2.5 Jedis中使用Lua

```java
// === eval ===
String key = "hello";
String script = "return redis.call('get',KEYS[1])"; 
Object result = jedis.eval(script, 1, key);
// 打印结果为world
System.out.println(result);

// === evalsha ===

String scriptSha = jedis.scriptLoad(script);
Stirng key = "hello";
Object result = jedis.evalsha(scriptSha, 1, key); // 打印结果为world
System.out.println(result);

```





# 3. Python 客户端 redis-py



# 4. 客户端管理

> 运维重点，暂略；



# 5. 客户端常见异常



# 6. 客户端案例分析
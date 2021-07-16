---
title: Seata完成2PC分布式事务
category: 知识点
tag: 
	- 分布式事务
	- 2PC
abbrlink: 3721150893
date: 2021-07-15 20:00:00
---

{% cq %}
Seata 2PC 分布式事务
{% endcq %}

<!-- more -->

# Seata完成2PC分布式事务

## 环境搭建

### 创建MySQL

分别使用虚拟机 192.168.33.51、192.168.33.52、192.168.33.53（多虚拟机环境参考：[搭建基础服务器](./2556627931)，MySQL安装参考：[Docker安装MySQL](./728095789)）创建MySQL示例。对应Docker命令如下：

```bash
docker run -p 3308:3306 --name mysql_dtx \
-v /home/dev/data/mysql/dtx/conf:/etc/mysql/conf.d \
-v /home/dev/data/mysql/dtx/logs:/logs \
-v /home/dev/data/mysql/dtx/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```



### 创建测试数据库



## 创建应用






## 验证


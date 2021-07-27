---
title: Linux安装ZK
category: 软件安装
tag: Zookeeper
date: 2020-01-01 12:00:00
---

{% cq %}
TODO 
{% endcq %}

<!-- more -->

# Linux安装ZK

## 下载安装包
进入到 [zk安装包](https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz)，下载安装包

## 解压安装包
```bash
sudo tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz -C /opt/soft

# 重命名
sudo mv /opt/soft/apache-zookeeper-3.6.3-bin /opt/soft/zookeeper-3.6.3

# 创建数据及日志文件夹
cd /opt/soft/zookeeper-3.6.3
sudo mkdir data logs
```

## 配置文件
```bash
cd /opt/soft/apache-zookeeper-3.6.3-bin/conf

# 拷贝示例配置文件
sudo cp zoo_sample.cfg zoo.cfg
```

## 修改配置文件
```bash
sudo vi zoo.cfg

# 配置zk集群信息
server.1=192.168.33.61:2888:3888
server.2=192.168.33.62:2888:3888
server.3=192.168.33.63:2888:3888
```
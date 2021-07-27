---
title: Linux安装Zookeeper
category: 软件安装
tag: Zookeeper
abbrlink: 549757466
date: 2021-07-27 20:50:00
---

{% cq %}
Linux 安装 Zookeeper
{% endcq %}

<!-- more -->

# Linux安装Zookeeper

## 下载安装包
进入到 [Apache ZooKeeper](http://zookeeper.apache.org/releases.html)，下载安装包(注意下载可执行包，不是源码包)

## 解压安装包
```bash
sudo tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz -C /opt/soft

# 重命名
sudo mv /opt/soft/apache-zookeeper-3.6.3-bin /opt/soft/zookeeper-3.6.3

# 创建数据及日志文件夹
cd /opt/soft/zookeeper-3.6.3
sudo mkdir data
```

## 配置文件
```bash
cd /opt/soft/zookeeper-3.6.3/conf

# 拷贝示例配置文件
sudo cp zoo_sample.cfg zoo.cfg
```


## 修改配置文件

```bash
sudo vi zoo.cfg

# 指定数据文件路径为刚才创建好的目录
dataDir=/opt/soft/zookeeper-3.6.3/data
```



## 启动

```bash
# 进入软件目录
cd /opt/soft/zookeeper-3.6.3

# 启动
./bin/zkServer.sh start zoo.cfg

# 关闭
./bin/zkServer.sh stop zoo.cfg
```



## 查看

使用jps命令查看zookeeper进程

```bash
$ jps
1972 QuorumPeerMain
2011 Jps
```


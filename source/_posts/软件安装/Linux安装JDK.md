---
title: Linux安装JDK
category: 软件安装
tag: jdk
date: 2021-07-27 19:00:00
---

{% cq %}
Linux 安装 JDK
{% endcq %}

<!-- more -->

# Linux安装JDK

## 下载jdk
进入 [jdk下载页面](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html) 下载 `jdk-8u291-linux-x64.tar.gz`安装包并上传至服务器
- 版本任意选择

## 解压
```bash
sudo mkdir -p /opt/soft

sudo tar -zxvf jdk-8u291-linux-x64.tar.gz -C /opt/soft

```

## 配置环境变量
> 编辑`/etc/profile` 文件

```bash
sudo vi /etc/profile

# 在文件最后添加以下内容
export JAVA_HOME=/opt/soft/jdk1.8.0_291  # jdk解压目录
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
export PATH=$PATH:${JAVA_PATH}
```

## 刷新配置文件
```bash
source /etc/profile
```

## 验证
```bash
# dev @ localhost in ~ [10:17:39] 
$ java -version
java version "1.8.0_291"
Java(TM) SE Runtime Environment (build 1.8.0_291-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.291-b10, mixed mode)
```



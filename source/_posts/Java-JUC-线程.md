---
title: Java-JUC-线程
abbrlink: 2466310928
date: 2020-01-15 10:52:08
categories:
  - Java
  - Basic
tags:
  - JUC
  - 并发
  - 多线程
author: 长歌
---

{% cq %}
内容照搬自: [并发番@Thread一文通(1.7版) by 黄志鹏kira](https://www.zybuluo.com/kiraSally/note/823674)  
自己手写一遍加深印象~
{% endcq %}
<!-- More -->

# 什么是线程
## 线程的概述
### 线程组成
- 一个运行程序就是一个进程,而线程是进程中独立运行的子任务
- 线程是操作系统执行流中的最小单位,一个进程可以有多个线程,这些线程与进程共享同一份内存空间
- 线程是系统独立调度和分派CPU的基本单位，通常有`就绪`、`运行`、`阻塞`三种基本状态

### 线程 vs 进程


---
title: Redis设计与实现笔记_简单动态字符串
categories:
  - 数据库
  - Redis
tags:
  - Redis
author: leithda
abbrlink: 1542299328
date: 2021-06-21 22:40:00
---
{% cq %}
Reids没有使用C语言传统的字符串表示，而是定义了一种结构用于表示字符串，即简单动态字符串（Simple Dynamic String，SDS），并将SDS作为Redis默认字符串表示
{% endcq %}
<!-- more -->



# 简单动态字符串




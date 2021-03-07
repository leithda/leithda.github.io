---
title: JDK-集合-概述
categories:
  - 源码
  - JDK
  - 集合
tags:
  - 源码
author: 长歌
abbrlink: 1126326809
date: 2020-06-09 22:34:05
---


{% cq %}
Java 集合框架主要包括两种类型的容器，一种是集合（Collection），存储一个元素集合，另一种是图（Map），存储键/值对映射。Collection 接口又有 3 种子类型，List、Set 和 Queue，再下面是一些抽象类，最后是具体实现类，常用的有 ArrayList、LinkedList、HashSet、LinkedHashSet、HashMap、LinkedHashMap 等等。
集合框架是一个用来代表和操纵集合的统一架构
{% endcq %}
<!-- More -->

Java的集合框架都包含如下内容：
- 接口：是代表集合的抽象数据类型。例如 Collection、List、Set、Map 等。之所以定义多个接口，是为了以不同的方式操作集合对象
- 实现（类）：是集合接口的具体实现。从本质上讲，它们是可重复使用的数据结构，例如：ArrayList、LinkedList、HashSet、HashMap。
- 算法：是实现集合接口的对象里的方法执行的一些有用的计算，例如：搜索和排序。这些算法被称为多态，那是因为相同的方法可以在相似的接口上有着不同的实现。
除了集合，该框架也定义了几个 Map 接口和类。Map 里存储的是键/值对。尽管 Map 不是集合，但是它们完全整合在集合中。


# Collection 集合系列
{% asset_img 集合源码之Collection系列.png 集合源码之Collection系列 %}

# Map 集合系列
{% asset_img 集合源码之Map系列.png 集合源码之Map系列.png %}


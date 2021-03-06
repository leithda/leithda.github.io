---
title: JDK集合源码-ArrayList
abbrlink: 2526212613
date: 2020-03-06 9:00:00
categories:
  - 源码
  - JDK
  - 集合
tags:
  - 源码
author: 长歌
---

{% cq %}
Jdk源码LinkedList解析
{% endcq %}

<!-- more -->

# LinkedList

## 整体架构
LinkedList底层使用双向链表实现，因为是双向链表，只要机器内存充足，理论上是没有大小限制的。

## LinkedList类

```java
public class LinkedList<E>
        extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable{

        }
```
- 继承了`AbstractList`抽象类，默认提供了一般List的操作的基本实现。
- 实现了`Deque`接口，支持队列操作

## 源码解析

### 属性



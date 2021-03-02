---
title: JDK集合源码-ArrayList
abbrlink: 2501046991
date: 2020-03-01 22:30:00
categories:
  - Java
  - Core
tags:
  - 源码
author: 长歌
---

{% cq %}
ArrayList
{% endcq %}
<!-- more -->
# ArrayList

## 整体架构
    ArrayList底层使用数组结构，索引从0开始。

## 类注释
    - 继承了List接口
    - 支持自动扩容
    - size, isEmpty, get, set, iterator, and listIterator复杂度为常数级 O(1)
    - 线程非安全
    - 它的iterator/listIterator支持`fast-fail`机制

## 源码解析
### 属性
```java
private static final int DEFAULT_CAPACITY = 10; // 
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
transient Object[] elementData; 
private int size;   // 元素个数
```
### 方法

#### 构造方法

```java
    // 使用指定容量初始化ArrayList
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }

    // 无参构造函数，数组为空
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    // 指定初始数据初始化
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            // 如果集合类型不是Object类型，转换为Object类型
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 指定初始数据为空，初始化为空数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
- ArrayList使用无参构造函数时，底层数组是空数组，而不是容量为10
- 关于see 6260652,这个编号属于JDK bug库中的编号.关于bug的解释请查看[雨夜听声博客](https://www.cnblogs.com/lsf90/p/5366325.html)

#### 新增和扩容实现


---
title: Java-JUC-关键字
abbrlink: 4169854470
date: 2020-01-06 16:17:13
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
java提供的并发关键字主要如下:`volatile`,`synchronized`和`final`
{% endcq %}
<!-- More -->

## volatile
> volatile变量，用来确保将变量的可见性。当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。
> 
> 在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。
> 
> {% asset_img volatile_1.png 多线程数据操作示意图 %}
> 当对非`volatile`变量进行读写的时候，每个线程先从内存拷贝变量到CPU缓存中。如果计算机有多个CPU，每个线程可能在不同的CPU上被处理，这意味着每个线程可以拷贝到不同的 CPU cache 中。
> 
> 而声明变量是 volatile 的，JVM 保证了每次读变量都从内存中读，跳过 CPU cache 这一步。



### 特性
1. 保证可见性
2. 对于单个volatile变量的读或者写具有原子性.
    如`i++`,其变量的操作为read;inc.write.变量的写操作依赖与当前值,所以不能保证线程安全
3. 禁止指令重排序

## synchronized

## final

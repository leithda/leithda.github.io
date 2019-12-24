---
title: 剑指Offer题解-替换空格
abbrlink: 3123224617
date: 2019-12-23 21:34:04
categories:
  - 算法
tags:
  - 剑指Offer
  - 算法
author: 长歌
---

{% cq %}

题目来自[《剑指Offer》]( https://book.douban.com/subject/6966465/ )一书。

**知识点**: 字符串

{% endcq %}

<!-- More -->


## 题目链接

[牛客网](https://www.nowcoder.com/practice/4060ac7e3e404ad1a894ef3e17650423?tpId=13&tqId=11155&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking)



## 题目描述

实现一个函数，把字符串中的每个空格替换成`%20`。例如输入`"We are happy."`,则输出`"We%20are%20happy"`



## 题解

### 使用正则表达式

```java
public String replaceSpaceWithRegEx(String str) {
    return str.replaceAll("\\s", "%20");
}
```



### 双指针遍历法

```java
public String replaceSpace(String str) {
    StringBuffer sb = new StringBuffer(str);
    int p1 = sb.length() - 1;
    for (int i = 0; i <= p1; i++) {
        if (sb.charAt(i) == ' ') {
            sb.append("  ");
        }
    }
    int p2 = sb.length() - 1;
    while (p1 >= 0 && p2 > p1) {
        char c = sb.charAt(p1--);
        if (c == ' ') {
            sb.setCharAt(p2--, '0');
            sb.setCharAt(p2--, '2');
            sb.setCharAt(p2--, '%');
        } else {
            sb.setCharAt(p2--, c);
        }
    }
    return sb.toString();
}
```


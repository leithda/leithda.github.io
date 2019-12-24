---
title: 剑指Offer题解-从尾到头打印链表
abbrlink: 1532733633
date: 2019-12-23 22:08:17
categories:
  - 算法
tags:
  - 剑指Offer
  - 算法
author: 长歌
---

{% cq %}

题目来自[《剑指Offer》]( https://book.douban.com/subject/6966465/ )一书。

**知识点**: 链表

{% endcq %}

<!-- More -->

## 题目链接

[牛客网](https://www.nowcoder.com/practice/d0267f7f55b3412ba93bd35cfa8e8035?tpId=13&tqId=11156&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking)



## 题目描述

从尾到头反过来打印出链表每个结点的值

```
Input : 1->2->3
Output: 3,2,1
```



## 题解

### 基础代码

- 链表类定义

```java
public class ListNode {
    int val;
    ListNode next = null;

    ListNode(int val) {
        this.val = val;
    }
}
```

### 头插法实现

```java
/**
 * 头插法实现反向输出链表
 * @param listNode 链表节点
 * @return 链表从尾到头顺序的 ArrayList
 */
public ArrayList<Integer> printListByInsertFirst(ListNode listNode) {
    // 头插法构建逆序链表
    ListNode head = new ListNode(-1);
    while (listNode != null) {
        ListNode memo = listNode.next;
        listNode.next = head.next;
        head.next = listNode;
        listNode = memo;
    }
    // 构建 ArrayList
    ArrayList<Integer> ret = new ArrayList<Integer>();
    head = head.next;
    while (head != null) {
        ret.add(head.val);
        head = head.next;
    }
    return ret;
}
```

### 递归实现

```java
/**
 * 递归实现反向输出链表
 * @param listNode 链表节点
 * @return 链表从尾到头顺序的 ArrayList
 */
public ArrayList<Integer> printListByRecursion(ListNode listNode) {
    ArrayList<Integer> ret = new ArrayList<Integer>();
    if (listNode != null) {
        ret.addAll(printListByRecursion(listNode.next));
        ret.add(listNode.val);
    }
    return ret;
}
```

### 通过栈实现

```java
/**
 * 栈实现反向输出链表
 * @param listNode 链表节点
 * @return 链表从尾到头顺序的 ArrayList
 */
public ArrayList<Integer> printListByStack(ListNode listNode) {
    Stack<Integer> stack = new Stack<Integer>();
    while (listNode != null) {
        stack.add(listNode.val);
        listNode = listNode.next;
    }
    ArrayList<Integer> ret = new ArrayList<Integer>();
    while (!stack.isEmpty())
        ret.add(stack.pop());
    return ret;
}
```
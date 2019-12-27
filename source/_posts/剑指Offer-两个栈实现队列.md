---
title: 剑指Offer-两个栈实现队列
abbrlink: 1475820628
date: 2019-12-24 21:42:09
categories:
  - 算法
tags:
  - 剑指Offer
  - 算法
author: 长歌
---

{% cq %}

题目来自[《剑指Offer》]( https://book.douban.com/subject/6966465/ )一书。

**知识点**: 树

{% endcq %}
<!-- More -->

## 题目链接
[牛客网](https://www.nowcoder.com/practice/54275ddae22f475981afa2244dd448c6?tpId=13&tqId=11158&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking)

## 题目描述
用两个栈来实现一个队列，完成队列的Push和Pop操作。 队列中的元素为int类型。


## 题解
入栈时，通过栈1进行入栈，出栈时，将栈1中的元素出栈，入到栈2中，使之顺序再次翻转，完成队列操作

```java
public class Ex07 {
    Stack<Integer> stack1 = new Stack<Integer>();
    Stack<Integer> stack2 = new Stack<Integer>();

    public void push(int node) {
        stack1.push(node);
    }

    public int pop() throws Exception {
        if (stack2.isEmpty())
            while (!stack1.isEmpty())
                stack2.push(stack1.pop());

        if (stack2.isEmpty())
            throw new Exception("queue is empty");

        return stack2.pop();
    }
}
```
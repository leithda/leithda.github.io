---
title: 剑指Offer题解-二维数组中的查找
abbrlink: 1384360267
date: 2019-12-23 20:52:32
categories:
  - 算法
tags:
  - 剑指Offer
  - 算法
author: 长歌
---

{% cq %}

题目来自[《剑指Offer》]( https://book.douban.com/subject/6966465/ )一书。

**知识点**: 数组、查找

{% endcq %}

<!-- More -->


## 题目链接

[牛客网]( https://www.nowcoder.com/practice/abc3fe2ce8e146608e868a70efebf62e?tpId=13&tqId=11154&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking )



## 题目描述

在一个二维数组中，每一行都按照从左到右递增的顺序排序，没一列都按照从上到下递增的顺序排序。给定一个数，判断数组中是否含有该数.

```
array{
	1	2	8	9
	2	4	9	12
	4	7	10	13
	6	8	11	15
}
target: 7 return: true
target: 5 return: false
```



## 题解

该二维数组中的数，小于它的数一点在其左边，大于它的数一定在它下边。所以从右上角开始查找， 每次将二维数组矩阵的中最右上角的数字与要目标数字比较，基于二维数组从左到右从上到下递增，那么当最右上角的数字大于目标数字就可以去掉该列，当最右边的数字小于目标数字的时候就去掉该行，遍历查找。

```java
public boolean find(int target, int[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) {
        return false;
    }
    int rows = matrix.length;   // 行数
    int cols = matrix[0].length;    // 列数
    int r = 0, c = cols - 1; // 右上角
    while (r <= rows - 1 && c >= 0) {
        if (target == matrix[r][c]) {
            return true;
        } else if (target > matrix[r][c]) {
            r++;
        } else {
            c--;
        }
    }
    return false;
}
```




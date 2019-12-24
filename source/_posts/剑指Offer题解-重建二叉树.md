---
title: 剑指Offer题解-重建二叉树
abbrlink: 555281782
date: 2019-12-24 20:06:42
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

## 题目链接
[牛客网](https://www.nowcoder.com/practice/8a19cbe657394eeaac2f6ea9b0f6fcf6?tpId=13&tqId=11157&tPage=1&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking)

## 题目描述
输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。


## 题解
二叉树的前序遍历中，第一个数字是根结点的值，中序遍历中，根结点在中间，左侧是左子树的结点，右子树在根结点的右边

然后单独看左子树，中序中，根结点左侧的为左子树的中序遍历。前序中，根结点后的先是左子树的前序遍历(元素个数与中序保持一致)

```java
public class Ex06 {

    private Map<Integer, Integer> indexForInOrders = new HashMap<>();

    public TreeNode reConstructBinaryTree(int[] pre, int[] in) {
        for (int i = 0; i < in.length; i++)
            indexForInOrders.put(in[i], i);
        return reConstructBinaryTree(pre, 0, pre.length - 1, 0);
    }

    /**
     * 重建二叉树
     * @param pre 前序序列
     * @param preL 前序序列开始索引
     * @param preR 前序序列结束索引
     * @param inL 中序序列开始索引，用于计算左子树大小
     * @return 重建后的二叉树
     */
    public TreeNode reConstructBinaryTree(int[] pre,int preL,int preR,int inL){
        if(preL > preR)
            return null;
        TreeNode  root = new TreeNode(pre[preL]);
        int inIndex = indexForInOrders.get(root.val);   // 根结点在中序中的索引
        int leftTreeSize = inIndex - inL;   // 左子树大小
        root.left = reConstructBinaryTree(pre, preL + 1, preL + leftTreeSize, inL);
        root.right = reConstructBinaryTree(pre, preL + leftTreeSize + 1, preR, inL + leftTreeSize + 1);
        return root;
    }
}
```
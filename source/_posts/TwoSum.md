title: 排序算法
categories:
  - 算法
  - 题目
tags:
  - 算法
  - 力扣
author: 长歌
date: 2019-10-11
---

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。
你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。
<!-- More -->
## 示例:
- 给定 nums = [2, 7, 11, 15], target = 9
- 因为 nums[0] + nums[1] = 2 + 7 = 9
- 所以返回 [0, 1]

## C语言题解
```c
/**
 * Note: The returned array must be malloced, assume caller calls free().
 */
int* twoSum(int* nums, int numsSize, int target, int* returnSize){
    static int a[2]={0};
    

    for (int i = 0; i < numsSize - 1; i++)
    {
        for (int j = i+1; j < numsSize; j++)
        {
            if (nums[i] + nums[j] == target)
            {
                a[0] = i;
                a[1] = j;
                *returnSize = 2;
                return a;
            }
        }
    }
    return 0;
}

```
---
来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/two-sum
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

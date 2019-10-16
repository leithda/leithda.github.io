title: 排序算法
categories:
  - 算法
  - 排序
tags:
  - 排序算法
author: 长歌
date: 2019-10-18
---

必会的排序算法
<!-- More -->

# 排序算法
## 冒泡排序

### 基本思想
数组中两个数两两比较大小，大的放后面

### 过程
- 比较相邻的两个数，如果前者大于后者，交换位置
- 从后到前比较，直到最前面的两个数比较完，最小的数就排列在最前面了

### 平均时间复杂度
O(n^2)

### C语言实现
```c
/**
 * 冒泡排序
 * @param arr
 * @param size
 */
void buddleSort(int *arr,int size){
    int temp;
    for (int i = 0; i < size; ++i) {
        for(int j=size-1;j>i;j--){
            if(arr[j] < arr[j-1]){
                temp = arr[j];
                arr[j] = arr[j-1];
                arr[j-1] = temp;
            }
        }
    }
}
```

## 选择排序

### 基本思想
遍历数组，每次找到最小元素交换到前面

### 过程
- 第一次遍历找到最小的元素，放到数组第一位置
- 第n次遍历，找到第n小元素，放到数组第n位置

### 平均时间复杂度
O(n^2)

### C语言实现
```c
/**
 * 选择排序
 * @param arr 待排序数组
 * @param lenth 数组长度
 */
void selectSort(int *arr,int lenth){
    for(int i=0;i<lenth;i++){
        int minIndex = i;
        for(int j=i+1;j<lenth;j++){
            if(arr[j]<arr[minIndex]){
                minIndex = j;
            }
        }
        if(minIndex != i){
            int temp = arr[i];
            arr[i] = arr[minIndex];
            arr[minIndex] = temp;
        }
    }
}
```

## 插入排序

### 基本思想
假设前n-1个数已经排好序，将第n个数组插入到前n-1个数中

### 过程
- 记录第n个数字
- 从n-1往前遍历，找到第n个数字的位置,查找过程中将n-1到pos之间的数字后移，再将第n个数字插入到pos位置上

### 平均时间复杂度
O(n^2)

### C语言实现
```c
/**
 * 插入排序
 * @param arr 数组
 * @param lenth 数组长度
 */
void insertSort(int *arr, int lenth) {
    int temp;
    int insPos; // 插入位置
    for (int i = 1; i < lenth - 1; ++i) {
        temp = arr[i];
        for (insPos = i - 1; temp < arr[insPos]; insPos--) {
            arr[insPos + 1] = arr[insPos];
        }
        arr[insPos+1] = temp;
    }
}
```


## 希尔排序

### 基本思想
根据增量将数组分为多组，每组子数列进行插入排序，
将增量逐渐减小到1，此时数组基本有序，此时进行插入排序，使数组有序

### 过程
- 增量初始值设置为数组长度，每次除2
- 根据增量将数组分组，执行插入排序

### 平均时间复杂度
复杂，无法计量

### C语言实现
```c
/**
 * 希尔排序
 * @param arr 数组
 * @param lenth 数组长度
 */
void shellSort(int *arr, int lenth) {
    int temp = 0;
    int insPos;
    int increment = lenth;
    while (true) {
        increment = increment / 2;
        for (int j = 0; j < increment; j++) {
            for (int i = j + increment; i < lenth; i = i + increment) {
                temp = arr[i];
                for(insPos = i - increment; temp < arr[insPos]; insPos= insPos - increment){
                    arr[insPos + increment] = arr[insPos];
                }
                arr[insPos + increment] = temp;
            }
        }
        if(increment == 1){
            break;
        }
    }
}
```


## 快速排序

### 基本思想
- 先从数列中取出一个数作为key(key值的选取可以自定义)
- 将比key大的数放在右侧，比key小的数放在左侧
- 对两个子数列再次使用快排


### 平均时间复杂度
O(N*logN)

### C语言实现
```c
/**
 * 快速排序
 * @param arr  数组
 * @param l 左
 * @param r 右
 */
void quickSort(int *arr,int l,int r){
    if(l >=r){
        return;
    }

    int i=l;
    int j=r;
    int key = arr[l];

    while(i<j){
        // 循环找，将大的都移到右边，小的都移到左边
        while(i<j && arr[j]>=key){  // 从右向左找第一个小于key的值
            j--;
        }
        if(i<j){
            arr[i] = arr[j];
            i++;
        }

        while(i<j && arr[i] < key){ // 从左向右找第一个大于key的值
            i++;
        }
        if(i<j){
            arr[j] = arr[i];
            j--;
        }
    }
    arr[i] = key;
    quickSort(arr,l,i-1);
    quickSort(arr,i+1,r);
}
```

## 归并排序

### 基本思想
- 将数列分为两组，对每组进行归并
- 将分组后的数列合并


### 平均时间复杂度
O(N*logN)

### C语言实现
```c
/**
 * 归并排序
 * @param arr 数组
 * @param first 首位
 * @param last  末位
 */
void mergeSort(int *arr,int first,int last,int *temp){
    if(first < last){
        int middle = (first+last)/2;
        mergeSort(arr,first,middle,temp);
        mergeSort(arr,middle+1,last,temp);
        // 合并 arr[first-middle] arr[middle+1-last]
        int i=first;
        int j=middle+1;
        int k = 0;
        while(i<=middle && j<=last){
            if(arr[i]<=arr[j]){
                temp[k] = arr[i];
                i++;
            }else{
                temp[k] = arr[j];
                j++;
            }
            k++;
        }
        while(i<=middle){
            temp[k++] = arr[i++];
        }
        while(j<=last){
            temp[k++] = arr[j++];
        }
        for (int ii = 0; ii < k; ++ii) {
            arr[first+ii] = temp[ii];
        }
    }
}
```

## 堆排序

### 介绍
建议读这篇博客   [《堆排序算法（图解详细流程）》](https://blog.csdn.net/u010452388/article/details/81283998)

### 基本思想
1. 首先将待排序的数组构造成一个最大堆，此时，整个数组的最大值就是堆结构的顶端

2. 将顶端的数与末尾的数交换，此时，末尾的数为最大值，剩余待排序数组个数为n-1

3. 将剩余的n-1个数再构造成大根堆，再将顶端数与n-1位置的数交换，如此反复执行，便能得到有序数组

### 平均时间复杂度
O(N*logN)

### C语言实现
```c
/**
 * 堆排序
 * @param arr 数组
 * @param lenth 数组长度
 */
void heapSort(int *arr, int lenth) {
    // 创建最大堆
    createHeap(arr, lenth);

    while (lenth > 1) {
        // 设置最大值，将最大值移到最后
        arr[lenth - 1] = arr[0] + arr[lenth - 1];
        arr[0] = arr[lenth - 1] - arr[0];
        arr[lenth - 1] = arr[lenth - 1] - arr[0];
        lenth--;
        // 调整余下堆元素
        heapify(arr,0, lenth);
    }
}


/**
 * 创建最大堆
 * @param arr 数组
 * @param lenth 数组长度
 */
void createHeap(int *arr, int lenth) {
    for (int i = 0; i < lenth; i++) {
        // 当前索引
        int currentIndex = i;
        // 父节点索引
        int fatherIndex = (currentIndex - 1) / 2;

        /**
         * 如果当前插入的节点的值大于父节点的值，交换值，且将索引指向父节点
         * 父节点继续和它的父节点比较，直到不大于父节点，退出循环
         */
        while (arr[currentIndex] > arr[fatherIndex]) {
            // 交换当前节点与父节点的值
            arr[currentIndex] = arr[currentIndex] + arr[fatherIndex];
            arr[fatherIndex] = arr[currentIndex] - arr[fatherIndex];
            arr[currentIndex] = arr[currentIndex] - arr[fatherIndex];

            currentIndex = fatherIndex;
            fatherIndex = (currentIndex - 1) / 2;
        }
    }
}

/**
 * 重新调整堆为最大堆
 * @param arr 数组
 * @param index 下沉索引
 * @param lenth 数组长度
 */
void heapify(int *arr, int index, int lenth) {
    int left = 2 * index + 1;
    int right = 2 * index + 2;
    while (left < lenth) {
        // 找到子节点中较大的值的索引
        int bigIndex;
        if (arr[left] < arr[right] && right < lenth) {
            bigIndex = right;
        } else {
            bigIndex = left;
        }

        // 比较父节点与较大值子节点的值
        if (arr[index] > arr[bigIndex]) {
            bigIndex = index;
        }

        // 如果父节点的值最大，已经是大根堆，退出循环
        if (index == bigIndex) {
            break;
        }

        // 父节点不是最大值，与子节点中较大值更换
        arr[index] = arr[index] + arr[bigIndex];
        arr[bigIndex] = arr[index] - arr[bigIndex];
        arr[index] = arr[index] - arr[bigIndex];

        // 重新计算交换之后子节点的索引
        int left = 2 * index + 1;
        int right = 2 * index + 2;
    }
}
```

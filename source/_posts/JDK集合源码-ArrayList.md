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
    // 默认初始化容量
    private static final int DEFAULT_CAPACITY = 10;

    // 空数组对象
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 无参构造函数使用空数组对象
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // ArrayList底层数据结构，transient避免被自动序列化以实现自定义序列化逻辑
    transient Object[] elementData; // non-private to simplify nested class access

    // ArrayList元素个数
    private int size;
```
### 方法

#### 构造方法

```java
    // 指定容量构造函数
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;   // 指定容量为0时，使用EMPTY_ELEMENTDATA进行初始化
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }

    // 无参构造函数
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    // 指定数据构造函数
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
- ArrayList使用无参构造函数时，底层数组是空数组，而不是容量为10。10是无参构造第一次执行add元素扩容后容量
- 关于see 6260652,这个编号属于JDK bug库中的编号.官方查看文档地址:[bugs.java.com](https://bugs.java.com/bugdatabase/viewbug.do?bugid=6260652)，问题在 Java 9 中被解决。

#### 新增和扩容实现

新增元素：
```java
    // 新增元素
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 确保容量足够，会改变modCount值
        elementData[size++] = e;
        return true;
    }

    // 确保容量足够
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    // 计算容量
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { // 无参构造的ArrayList，取10和期望容量的最大值
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    // 确保足够容量，不足进行扩容
    private void ensureExplicitCapacity(int minCapacity) {
        // 记录数组被修改
        modCount++;

        // 如果期望最小容量大于当前数组的长度，进行扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    // 扩容
    private void grow(int minCapacity) {
        // 记录原数组长度
        int oldCapacity = elementData.length;

        // 默认1.5倍扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);

        // 避免溢出
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;

        // 最大限制 int.MAX_VALUE - 8
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);    // 扩容容量过大处理

        // 数组拷贝
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    // 扩容容量过大处理
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) 
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    }
```
- 扩容处理后，数组元素的赋值直接使用`elementData[size++] = e`.没有锁控制，所以这里的操作线程不安全
- 扩容底层实现
  ```java
  // java.lang.System
  /*
   * @param      src      源数组
   * @param      srcPos   原数组拷贝起始位置
   * @param      dest     目标数组
   * @param      destPos  目标数组指定起始位置
   * @param      length   拷贝长度
   */
  public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
  ```

#### 删除元素

```java
  // 根据值删除元素
  public boolean remove(Object o) {
      // 删除元素是null，找到第一个值是null的删除
      if (o == null) {
          for (int index = 0; index < size; index++)
              if (elementData[index] == null) {
                  fastRemove(index);
                  return true;
              }
      } else {
          for (int index = 0; index < size; index++)
              // 非null元素，使用equals判等，想等后根据索引位置进行删除
              if (o.equals(elementData[index])) {
                  fastRemove(index);
                  return true;
              }
      }
      return false;
  }
```
- 删除时使用`#equals`进行判断。数组元素非基本类型时，需要关注equals的具体实现

```java
    // 私有删除方法，不检查是否越界，不返回删除元素值
    private void fastRemove(int index) {
        // 数组结构改变
        modCount++;
        
        // 计算删除后迁移数组长度
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        
        // 数组最后一个位置赋值为null,帮助GC
        elementData[--size] = null; 
    }
```

### 迭代器

迭代器通过实现`java.util.Iterator`类来完成。迭代器主要有以下几个属性:
```java
    // AbstractList迭代器的优化版
    private class Itr implements Iterator<E> {
        int cursor;       // 下一个返回元素的索引
        int lastRet = -1; // 上次迭代过程中索引的位置。元素被删除时为-1，删除时检查避免二次删除
        int expectedModCount = modCount;    // CAS 期望值

        //...
    }
```

迭代方法：
```java
public boolean hasNext() {
    return cursor != size;
}

@SuppressWarnings("unchecked")
public E next() {
    // 版本号检查，避免迭代过程中数组结构发生变化
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    // 更新下次索引位置
    cursor = i + 1;

    // lastRet 为最后一次迭代索引
    return (E) elementData[lastRet = i];
}

public void remove() {
    // 元素删除后，lastRet值为-1，避免多次删除
    if (lastRet < 0)
        throw new IllegalStateException();
    
    // 检查版本号
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        // 删除元素后版本号发生变化，重新赋值
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

## 总结
1. ArrayList底层使用Object数组，支持自动扩容
2. ArrayList线程不安全，多线程使用`Collections#synchronizedList`来保证线程安全。
3. ArrayList扩容和删除元素通过底层`System.arraycopy`数组拷贝实现。
4. ArrayList支持**fail-fast**[^1]快速失败机制，即迭代器迭代期间，如果发生结构变化(modCount)，迭代过程会抛出异常`ConcurrentModificationException`.


[^1]: [fail-fast_百度百科](https://baike.baidu.com/item/fail-fast/16329854?fr=aladdin)
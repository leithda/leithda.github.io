---
title: JDK-集合-抽象类介绍
abbrlink: 4107645288
date: 2020-06-13 00:43:17
categories:
  - 源码
  - JDK
  - 集合
author: 长歌
---

{% cq %}
Java集合相关的抽象类完成了集合的底层建设，为某些集合类型通用的方法提供了默认的实现。
{% endcq %}

<!-- More -->

# 概述
> 集合框架中的抽象类主要为各种集合提供默认的方法实现
>
> 类的定义：
>
> - 继承抽象类可以使用抽象类的默认实现
> - 实现接口表名当前类符合接口定义的规范
>
> 查看类需要关注的主要有三点
>
> 1. 变量，定义的变量的含义
> 2. 需要子类特性化实现的方法被定义为抽象方法，子类必须提供实现。
> 3. 提供方法默认实现(**默认实现可以是抛出异常,子类不支持此类操作无需复写**)供子类使用，子类不复写默认使用父类实现

# 抽象类们

## AbstractCollection

### 抽象方法

```java
/*
 * iterator方法，返回此集合的迭代器
 */
public abstract Iterator<E> iterator();

/*
 * 返回集合大小
 */
public abstract int size();
```

### 默认实现

```java
/*
 * 返回集合是否包含指定元素(方法较简单，此处不再具体介绍)
 */
public boolean contains(Object o) {
    Iterator<E> it = iterator();
    if (o==null) {
        while (it.hasNext())
            if (it.next()==null)
                return true;
    } else {
        while (it.hasNext())
            if (o.equals(it.next()))
                return true;
    }
    return false;
}
/*
 * 返回包含集合所有元素的数组
 * 支持迭代器在遍历中发生操作的情况
 */
public Object[] toArray() {
    // Estimate size of array; be prepared to see more or fewer elements
    Object[] r = new Object[size()];
    Iterator<E> it = iterator();
    for (int i = 0; i < r.length; i++) {
        if (! it.hasNext()) // fewer elements than expected
            return Arrays.copyOf(r, i);
        r[i] = it.next();
    }
    return it.hasNext() ? finishToArray(r, it) : r; // 迭代器遍历过程中发生改变，返回finishToArray()
}

private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
    int i = r.length;
    while (it.hasNext()) {
        int cap = r.length;
        if (i == cap) {
            int newCap = cap + (cap >> 1) + 1;  // 扩容
            // overflow-conscious code
            if (newCap - MAX_ARRAY_SIZE > 0)    // 判断数组长度是否溢出
                newCap = hugeCapacity(cap + 1);
            r = Arrays.copyOf(r, newCap);
        }
        r[i++] = (T)it.next();  // 复制迭代器中新增元素到返回数组中
    }
    // trim if overallocated
    return (i == r.length) ? r : Arrays.copyOf(r, i);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError
            ("Required array size too large");
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

/*
 * 添加元素，默认抛出错误，支持添加元素的集合复写此方法
 */
public boolean add(E e) {
    throw new UnsupportedOperationException();
}

// 其他方法忽略...

```

## AbstractList

> 首先查看定义,继承了 AbstractCollection，实现了 List 接口。
>
> ```java
> public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>{
>   
> }
> ```

### 变量

```java
// 记录修改次数
protected transient int modCount = 0;

/* 子集合类，在集合上进行的操作会作用到父集合
 * 本质上，子集合保留了自己在父集合上的索引位置，进行操作时，检查索引范围是否超出子集合范围，没超出则通
 * 过索引计算直接在父集合上进行相应操作;子集合通过 size 保留本身的大小，通过offset保存子集合相对于父集合的位置
 */
class SubList<E> extends AbstractList<E>{
  // ...
}

/*
 *  RandomAccess 扩展接口
 */
class RandomAccessSubList<E> extends SubList<E> implements RandomAccess{
  // ...
}

/*
 * 迭代器，相对简单，操作时会通过CAS避免数据不一致问题
 * 主要包含 hasNext() next() remove()三种方法
 * 重点：迭代器操作的元素为lastRet索引的元素，cursor指向操作元素的下一位置。
 */
private class Itr implements Iterator<E> {
  // 迭代器指向元素的索引
  int cursor = 0;
  
  // 上次迭代的索引，初始化时及remove()后为-1
  int lastRet = -1;
  
  // 期望修改次数值，发生变化说明列表存在并行改动
  int expectedModCount = modCount;
  
  // 判断是否存在下一个元素
  public boolean hasNext() {
      return cursor != size();
  }

  // 获取到迭代器元素
  public E next() {
      checkForComodification();
      try {
          int i = cursor;
          E next = get(i);
          lastRet = i;
          cursor = i + 1;
          return next;
      } catch (IndexOutOfBoundsException e) {
          checkForComodification();
          throw new NoSuchElementException();
      }
  }

  /*
   * 移除迭代器指向的元素
   */
  public void remove() {
      if (lastRet < 0)  // <1> 迭代器初始化或发生remove操作(迭代器指针后退，参考next指针前进)后不能remove。
          throw new IllegalStateException();
      checkForComodification();

      try {
          AbstractList.this.remove(lastRet);    // <1>处判断的主要原因
          if (lastRet < cursor) // 避免迭代器向前迭代(子类LstItr支持)，向前迭代时不需要移动cursor
              cursor--;
          lastRet = -1;
          expectedModCount = modCount;
      } catch (IndexOutOfBoundsException e) {
          throw new ConcurrentModificationException();
      }
  }

  // 判断数组修改次数在迭代器使用期间是否被改变
  final void checkForComodification() {
      if (modCount != expectedModCount)
          throw new ConcurrentModificationException();
  }
}

/*
 * Itr的升级版本，增加了一些操作
 */
private class ListItr extends Itr implements ListIterator<E>{
  // ...
}
```

### 抽象方法

```java

abstract public E get(int index);
  
```

### 默认实现

```java
// 返回元素在数组中的位置
public int indexOf(Object o) {
    ListIterator<E> it = listIterator();
    if (o==null) {
        while (it.hasNext())
            if (it.next()==null)
                return it.previousIndex();
    } else {
        while (it.hasNext())
            if (o.equals(it.next()))
                return it.previousIndex();
    }
    return -1;
}

// 清空集合元素
public void clear() {
    removeRange(0, size());
}

// 移除集合中下标从 fromIndex 到 toIndex 的元素
protected void removeRange(int fromIndex, int toIndex) {
    ListIterator<E> it = listIterator(fromIndex);
    for (int i=0, n=toIndex-fromIndex; i<n; i++) {
        it.next();
        it.remove();
    }
}

// 判断是否相等
public boolean equals(Object o) {
    if (o == this)
        return true;
    if (!(o instanceof List))
        return false;

    ListIterator<E> e1 = listIterator();
    ListIterator<?> e2 = ((List<?>) o).listIterator();
    while (e1.hasNext() && e2.hasNext()) {
        E o1 = e1.next();
        Object o2 = e2.next();
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    return !(e1.hasNext() || e2.hasNext());
}

```

## AbstractSequentialList

> 先看定义：
>
> ```java
> public abstract class AbstractSequentialList<E> extends AbstractList<E>
> ```
>
> 继承了AbstractList.在原有基础上，对集合的操作改为通过LstItr来进行操作



### 抽象方法

```java
public abstract ListIterator<E> listIterator(int index);
```

### 默认实现

```java
public Iterator<E> iterator() {
    return listIterator();
}

/*
 * 简单列举一个例子。该集合主要是将集合操作改为通过迭代类listIterator(index)来实现，并将错误封装为
 * IndexOutOfBoundsException 方便开发人员定位。其中listIterator(index)为抽象方法需要子类来实现
 */

public E get(int index) {
    try {
        return listIterator(index).next();
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}
```

## AbstractQueue

> 先看定义，该类继承了AbstractCollection，实现了Queue。
>
> ```java
> public abstract class AbstractQueue<E>
>     extends AbstractCollection<E>
>     implements Queue<E> 
> ```
>
> 该类主要提供了几个方法的默认实现。通过`offer、poll、peek`完成队列所需要的`add、addAll、remove、element`方法的实现。简化开发，子类只要实现`offer`、`poll`以及`peek`方法即可。

### 默认实现

```java
public boolean add(E e) {
    if (offer(e))   // Queue.offer
        return true;
    else
        throw new IllegalStateException("Queue full");
}

public E remove() {
    E x = poll();   // Queue.poll
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}

public E element() {
    E x = peek();   // Queue.peek
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}

public void clear() {
    while (poll() != null)
        ;
}

public boolean addAll(Collection<? extends E> c) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    boolean modified = false;
    for (E e : c)
        if (add(e)) // AbstractQueue.add
            modified = true;
    return modified;
}
```

## AbstractSet

> 先看定义，该类继承自AbstractCollection，实现了Set接口
>
> ```java
> public abstract class AbstractSet<E> extends AbstractCollection<E> implements Set<E>
> ```
>
> 看他的三个默认实现，逻辑并不复杂

### 默认实现

```java
    public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Set))
            return false;
        Collection<?> c = (Collection<?>) o;
        if (c.size() != size())
            return false;
        try {
            return containsAll(c);
        } catch (ClassCastException unused)   {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }
    }

    public int hashCode() {
        int h = 0;
        Iterator<E> i = iterator();
        while (i.hasNext()) {
            E obj = i.next();
            if (obj != null)
                h += obj.hashCode();
        }
        return h;
    }

    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);  // 1.7 新增的方法，感觉类似断言Assert~
        boolean modified = false;

        if (size() > c.size()) {
            for (Iterator<?> i = c.iterator(); i.hasNext(); )
                modified |= remove(i.next());
        } else {
            for (Iterator<?> i = iterator(); i.hasNext(); ) {
                if (c.contains(i.next())) {
                    i.remove();
                    modified = true;
                }
            }
        }
        return modified;
    }
```

## AbstractMap

---
title: JDK-集合-LinkedList
abbrlink: 2526212613
date: 2021-03-06 9:00:00
categories:
  - 源码
  - JDK
  - 集合
tags:
  - 源码
author: 长歌
---

{% cq %}
Jdk源码LinkedList解析
{% endcq %}

<!-- more -->

# LinkedList

## 整体架构
LinkedList底层使用双向链表实现，因为是双向链表，只要机器内存充足，理论上是没有大小限制的。

## LinkedList类

```java
public class LinkedList<E>
        extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable{

        }
```
- 继承了`AbstractList`抽象类，默认提供了一般List的操作的基本实现。
- 实现了`Deque`接口，支持队列操作

## 源码解析

### 属性

```java
public class LinkedList<E>
        extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
    /**
     * 元素个数
     */
    transient int size = 0;

    /**
     * 头节点
     */
    transient Node<E> first;

    /**
     * 尾节点
     */
    transient Node<E> last;

    private static class Node<E> {
        /**
         * 节点元素
         */
        E item;
        /**
         * 后继节点
         */
        Node<E> next;
        /**
         * 前驱节点
         */
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

### 操作

#### 新增
新增元素时，可以选择头插法或尾插法，`#add`方法默认从尾部开始追加,`#addFirst`方法默认从头部开始追加。

```java
  /**
   * 头插法
   */
  private void linkFirst(E e) {
      final Node<E> f = first;    // 指向原头节点
      final Node<E> newNode = new Node<>(null, e, f); // 生成新节点，且其后继节点设置为原头节点
      first = newNode;    // 头节点设置为新节点
      // 如果链表为看，设置为节点也为新节点；否则设置原头节点的前驱节点为新节点。
      if (f == null)
          last = newNode;
      else
          f.prev = newNode;
      // 大小和版本变化
      size++;
      modCount++;
  }

  // 尾插法类似,对应方法为:void linkLast(E e)。
```

#### 删除
与添加相对应，删除也分为从头部删除与从尾部删除。从头部删除代码如下：

```java
  /**
   * 从链表头删除
   */
  private E unlinkFirst(Node<E> f) {
      // 保存删除前的元素，用于返回
      final E element = f.item;
      // 记录删除节点的下一个节点
      final Node<E> next = f.next;
      // 帮助GC回收头节点
      f.item = null;
      f.next = null; // help GC

      // 头节点的下一个节点成为头节点
      first = next;

      // next为空，表示删除元素后链表为空，尾节点置空。否则将新头节点的前驱节点置空
      if (next == null)
          last = null;
      else
          next.prev = null;
      // 修改链表大小和版本
      size--;
      modCount++;
      return element;
  }
```

#### 查询

```java
  /**
   * 根据索引查询节点
   */
  Node<E> node(int index) {
      // assert isElementIndex(index);

      if (index < (size >> 1)) {  // 如果节点位置位于链表前半部分
          Node<E> x = first;
          for (int i = 0; i < index; i++)
              x = x.next;
          return x;
      } else {    // 如果节点位置位于链表后半部分
          Node<E> x = last;
          for (int i = size - 1; i > index; i--)
              x = x.prev;
          return x;
      }
  }
```
- LinkedList通过判断节点位置，采取了简单二分法，通过这种方式使循环的次数至少降低了一半，提高了查找的性能。

## 迭代器
LinkedList底层使用双向链表，支持双向的迭代访问。由于Iterator仅支持从头到尾的访问，所以LinkedList实现了ListIterator接口，支持双向迭代访问。

```java
    /**
     * List的子类可以实现，支持双向遍历，向后遍历取当前迭代位置，向前取当前迭代前一位置。
     */
    private class ListItr implements ListIterator<E> {

        /**
         * 记录上次迭代返回元素
         */
        private Node<E> lastReturned;

        /**
         * 迭代元素
         */
        private Node<E> next;

        /**
         * 迭代位置
         */
        private int nextIndex;

        /**
         * 期望值
         */
        private int expectedModCount = modCount;


        /**
         * 带参构造，可以指定索引进行迭代
         */
        ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        /**
         * 判断是否有后继节点
         */
        public boolean hasNext() {
            return nextIndex < size;
        }

        /**
         * 获取后继节点
         */
        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();
            // 记录上次迭代元素
            lastReturned = next;

            // 更新下一迭代节点
            next = next.next;

            // 向后迭代索引+1
            nextIndex++;
            return lastReturned.item;
        }

        /**
         * 判断是否有前驱节点
         */
        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        /**
         * 获取前驱节点
         */
        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            // next为空的情况是使用size初始化ListIterator并进行previous操作。注意这里每次获取元素后，lastReturned和next值相等。
            lastReturned = next = (next == null) ? last : next.prev;

            // 更新迭代索引
            nextIndex--;

            // 返回元素值
            return lastReturned.item;
        }

        /**
         * 获取向后迭代位置
         */
        public int nextIndex() {
            return nextIndex;
        }

        /**
         * 获取向前迭代位置
         */
        public int previousIndex() {
            return nextIndex - 1;
        }

        /**
         * 删除当前元素
         */
        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);

            // 这里应考虑边界情况，注意删除头或尾节点时的值变化
            if (next == lastReturned)   // 向前遍历时，next = lastReturned，删除后，应将next指向原元素的下一元素
                next = lastNext;
            else    // 向后遍历时， next = lastReturned.next，删除后，next指向不变，索引-1
                nextIndex--;
            // 帮助GC
            lastReturned = null;
            expectedModCount++; // 保持期望值一致
        }

        /**
         * 设置当前元素的值
         */
        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        /**
         * 当前迭代位置添加元素
         */
        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            // 添加到next前，nextIndex+1
            nextIndex++;
            // 保持期望一致
            expectedModCount++;
        }

        /**
         * forEach
         */
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }

        /**
         * 检查版本号
         */
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

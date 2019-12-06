---
title: jdk源码-LinkedList
abbrlink: 3563866560
date: 2019-12-06 20:59:02
categories:
  - Java
  - Core
tags:
  - 源码
author: 长歌
---
{% cq %} LinkedList底层由双向链表组成。实现了`List`和`Deque`接口，同时拥有栈和队列的功能。它同步不安全，对应迭代器使用了`Fail-fast`机制。相比于`ArrayList`，它的查询效率低，插入及删除效率高。 {% endcq %}

<!-- More -->
> 我的jdk版本

```cmd
C:\Users\99924>java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)
```

## LinkedList 的结构
```java
    // 内部节点类
    private static class Node<E> {
        // 节点值
        E item;
        // 后继节点
        Node<E> next;   
        // 前驱节点
        Node<E> prev;
        /**
         * 【Node的构造函数】
         * @param prev 前驱节点
         * @param element 节点值
         * @param next 后继节点
         */
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
- `LinkedList`内部由双向链表实现，链表元素为`Node`类。其`item`为元素值,`prev`为元素前驱节点,`next`为元素后继节点。

## LinkedList 的属性
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    // 列表大小
    transient int size = 0;

    // 头结点
    transient Node<E> first;

    // 尾结点
    transient Node<E> last;

    // ... 省略部分代码
}
```

## 节点操作
### 添加节点
1. 头插法，向链表头部添加新节点
```java
   private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

2. 尾插法，向链表尾部添加新节点
```java
    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

3. 向指定节点前加入新节点
```java
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

### 删除节点
1. 删除头节点
```java
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
2. 删除尾节点
```java
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```
3. 删除指定节点
```java
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

## 数据操作
### 添加数据
1. `#add(E element)`方法
```java
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
```

2. `#add(int index,E element)`方法
```java
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```

3. `#addAll()`方法
```java
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        // 添加集合为空直接返回
        if (numNew == 0)
            return false;

        Node<E> pred, succ; // 记录加入集合的前驱pred以及后继succ
        if (index == size) {    // 从链表尾部添加集合 pred=last -> 集合 -> succ=null
            succ = null;
            pred = last;
        } else {    // 从链表中间(下标元素前)添加集合 pred=succ.prev -> 集合 -> succ = node(index)
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) { // 如果集合的后继指针为null,表示尾节点是集合最后一个元素.即pred
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```

### 删除数据

1. `#remove(Object o)`方法
```java
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```
- 由代码可以看出，当传入元素为`null`时，从链表头部遍历;非`null`时，从链表尾部开始遍历，找到对应元素时，调用`#unlink()`方法移除元素。

2. `#remove(int index)`方法
```java
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
```

### 修改数据

1. `#set(int index,E element)`
```java
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```

### 获取数据

```java
    public E get(int index) {
        checkElementIndex(index); // <1>
        return node(index).item; // <2>
    }
```
- `<1>`处，首先调用`#checkElementIndex()`检查下标是否越界.具体方法如下:
    ```java
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }
    ```

- `<2>`处，调用`#node()`方法获取指定节点，将其元素值返回.
    ```java
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
    ```
    - 根据代码我可以知道，当元素下标小于列表长度一半时，从链表头部进行遍历。繁殖从链表尾部进行遍历。

### 获取数据下标

1. `#indexOf(E element)`

```java
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

2. `#lastIndexOf(Object o)`

```java
    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
```

### 遍历

- 通过内部类实现，代码逻辑不复杂，增删改注意分析链表指针变动，不再具体讲~

```java
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;
        private Node<E> next;
        private int nextIndex;
        private int expectedModCount = modCount;

        ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }

        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }

        /**
         * lamda表达式支持
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

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```



## `Deque`接口实现

- 实现都较简单，这里简单罗列下方法及对应功能

| 方法                                        | 说明                                      |
| :------------------------------------------ | :---------------------------------------- |
| `getFirst() : E`                            | 获取集合首个元素                          |
| `getLast() : E`                             | 获取集合最后一个元素                      |
| `removeFirst() : E`                         | 移除集合首个元素                          |
| `removeLast() : E`                          | 移除集合最后一个元素                      |
| `addFirst(E element) : void`                | 向集合头部添加新元素                      |
| `addLast(E element) : void`                 | 向集合尾部添加新元素                      |
| `peek() : E`                                | 获取栈首元素，且不移除                    |
| `element() : E`                             | 同`peek()`                                |
| `poll() : E`                                | 出栈，获取元素同时移除元素                |
| `remove() : E`                              | 移除队首元素，内部通过`removeFirst()`实现 |
| `offerFirst(E e) : boolean`                 | 队首添加元素，内部通过`addFirst()`实现    |
| `offerLast(E e) : boolean`                  | 队尾添加元素，内部通过`addLast()`实现     |
| `peekFirst() : E`                           | 获取栈首元素，不移除                      |
| `peekLast() : E`                            | 获取栈底元素，不移除                      |
| `poolFirst() : E`                           | 栈首元素出栈，获取并移除元素              |
| `poolLast() : E`                            | 栈底元素出栈，获取并移除                  |
| `push(E e) : void`                          | 入栈，内部通过`addFirst()`实现            |
| `pop() : E`                                 | 出栈，内部通过`removeFirst()`实现         |
| `removeFirstOccurrence(Object o) : boolean` | 移除首次出现元素，内部通过`remove()`实现  |
| `removeLastOccurrence(Object o) : boolean`  | 移除最后一次出现元素，自定义实现          |
| `descendingIterator() : Iterator<E>`        | 获取迭代器                                |


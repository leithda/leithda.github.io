---
title: Java之引用类型
abbrlink: 72877877
date: 2020-07-10 16:59:52
categories:
  - 理论
  - Java
author: 长歌
---

# 引用与对象

所有编程语言都有自己操作内存的方式，`C&C++`通过指针操作内存，而`Java`则是通过引用(Reference)

在Java中一切都被视为对象，但是我们使用的，只是指向对象的引用。

<!-- More -->

```
// 创建一个引用，引用可以单独存在
String s;

s = new String("hello");    // 将标识符(引用)指向具体的对象，接下来就可以通过标识符操作实际对象了
System.out.println(s+" world"); 
```

Java中的垃圾回收机制在判断是否回收某个对象的时候，都需要依据“引用”这个概念。

在不同的垃圾回收算法中，对引用的判断也不相同：

- 引用计数法：为所有对象增加一个计数器，每当有引用指向它时，计数器加1，当引用失效时，计数器减1。当计数器为0时，则认为该对象可以被回收。（**已弃用**）
- 可达性分析法：从GC Roots对象开始向下搜索，如果一个对象到GC Roots没有任何引用链相连时，则认为该对象可以被回收



# 四种引用类型

JDK1.2之前，一个对象只有“被引用”和“未被引用”两种状态，这无法描述某有特殊要求的对象。比如：内存充足保留，内存不足才被回收的特殊对象。

JDK1.2之后，Java将引用分为了四种：

1. 强引用(Strong Reference)
2. 软引用(Soft Reference)
3. 弱引用(Weak Reference)
4. 虚引用(Phantom Reference)

## 强引用

Java中默认声明的就是强引用，比如：

```
Object o = new Object();    // 只要 o 指向 Object对象，Object对象就不会被回收
o = null;   // 设置o指向null,上述Object对象所占用的空间会在合适的时间被回收
```

- **强引用存在，GC不会回收被引用的对象，当内存不足时会抛出OOM**。如果想要取消强引用与对象的关联，可以将引用赋值为`null` ，这样GC就可以对该对象进行回收了



## 软引用

**在内存足够时，软引用对象不会被回收；内存不足时，GC会回收软引用对象，回收后内存仍不足会抛出OOM错误。**软引用常常被用来实现缓存。

在JDK1.2之后，用java.lang.ref.SoftReference类来表示软引用



## 弱引用

**无论内存是否足够，只要JVM开始进行垃圾回收，弱引用关联的对象都会被回收。**

在JDK1.2之后，用java.lang.ref.WeakReference类来表示弱引用。



## 虚引用

虚引用是最弱的一种引用关系，如果一个对象持有虚引用，那么它就和没有任何引用一样，随时会被回收。

JDK1.2后，用`java.lang.ref.PhantomReference`来表示虚引用。

```
public class PhantomReference<T> extends Reference<T> {

    /**
     * Returns this reference object's referent.  Because the referent of a
     * phantom reference is always inaccessible, this method always returns
     * <code>null</code>.
     *
     * @return  <code>null</code>
     */
    public T get() {
        return null;
    }

    /**
     * Creates a new phantom reference that refers to the given object and
     * is registered with the given queue.
     *
     * <p> It is possible to create a phantom reference with a <tt>null</tt>
     * queue, but such a reference is completely useless: Its <tt>get</tt>
     * method will always return null and, since it does not have a queue, it
     * will never be enqueued.
     *
     * @param referent the object the new phantom reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     */
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

- 它的get方法返回`null` ,也就是说无法通过虚引用来获取对象，虚对象必须要和ReferenceQueue引用队列一起使用。



# 引用队列

- 引用队列可以和软引用、弱引用及虚引用一起配合使用，当垃圾回收期准备回收一个对象时，如果发现他还有引用(非强引用，有强引用不会被回收)，那么就会在回收对象之前，把这个引用加入到与之关联的引用队列中去。
- 程序中可以通过判断引用队列是否已经加入了引用，判断被引用的对象是否将要被回收，这样就可以在对象被回收之前采取一些措施。
- 与软引用、弱引用不同，虚引用必须和引用队列一起使用
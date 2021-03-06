---
title: Java之ThreadLocal
abbrlink: 355185077
date: 2020-07-10 17:27:09
categories:
  - 源码
  - JDK
  - 多线程
tags: ThreadLocal
author: 长歌
---

> 在JDK 1.2的版本中就提供java.lang.ThreadLocal，ThreadLocal为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序。
> ThreadLocal并不是一个Thread，而是Thread的局部变量，也许把它命名为ThreadLocalVariable更容易让人理解一些。  
> 在JDK5.0中，ThreadLocal已经支持泛型，该类的类名已经变为ThreadLocal<T>。API方法也相应进行了调整，新版本的API方法分别是void set(T value)、T get()以及T initialValue()。

<!-- More -->

# 使用

## 概述

```
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 *
 * <p>For example, the class below generates unique identifiers local to each
 * thread.
 * A thread's id is assigned the first time it invokes {@code ThreadId.get()}
 * and remains unchanged on subsequent calls.
 * <pre>
 * import java.util.concurrent.atomic.AtomicInteger;
 *
 * public class ThreadId {
 *     // Atomic integer containing the next thread ID to be assigned
 *     private static final AtomicInteger nextId = new AtomicInteger(0);
 *
 *     // Thread local variable containing each thread's ID
 *     private static final ThreadLocal&lt;Integer&gt; threadId =
 *         new ThreadLocal&lt;Integer&gt;() {
 *             &#64;Override protected Integer initialValue() {
 *                 return nextId.getAndIncrement();
 *         }
 *     };
 *
 *     // Returns the current thread's unique ID, assigning it if necessary
 *     public static int get() {
 *         return threadId.get();
 *     }
 * }
 * </pre>
 * <p>Each thread holds an implicit reference to its copy of a thread-local
 * variable as long as the thread is alive and the {@code ThreadLocal}
 * instance is accessible; after a thread goes away, all of its copies of
 * thread-local instances are subject to garbage collection (unless other
 * references to these copies exist).
 *
 * @author  Josh Bloch and Doug Lea
 * @since   1.2
 */
```

- 简单理解一下： 这个类提供线程局部变量，这些变量在每个线程都有自己的一份副本，并且通过在线程中使用它的`#get`与`#set`方法操作这些变量，`ThreadLocal`在类中通常是静态且私有的(`即private static`)，且通常关联与线程有关的状态字段(如：用户ID，事务等).
- 总结起来就一句话: **保证同一线程内操作的是同一对象**

## 使用场景

- 存放用户等状态信息(Web应用中保存登录的用户信息)
- 存放`Context`
- 存放Session
- Tomcat中的作业线程数据隔离

```
public class ThreadLocalDemo{
    private static ThreadLocal<Session> sessionHolder = new ThreadLocal<>();
    private static ThreadLocal<User> userHolder = new ThreadLocal<>();
    
    public void login(HttpRequest request){
        Session session = request.getSession();
        if(check(session){
            sessionHolder.set(session); // 使用ThreadLocal的set方法，保证当前线程获取到的session一致
        }
        User user = new User(request.getPatameter("username");  // 伪代码，有问题不用在意
        userHolder.set(user);
    }
                             
    public void logout(HttpRequest request){
        Session session = sessionHolder.get();
        session.close();    // 判空等异常处理取消
    }
}
```

- 目前没有实际应用场景，所以这里使用伪代码完成。
- 不使用ThreadLocal时，有如下两种情况

- - 使用类变量，**多线程访问时，获取到的session是同一个。其中一个线程更改session(比如关闭)会导致其他线程出现不可预知的问题**

```
private static Session session;

public void login(HttpRequest request){
    session = xxx;
}

public void logout(HttpRequest request){
    session.close();
}
```

- - 每个方法中的局部变量。**每次执行方法都会创建一个新的Session对象，造成资源浪费**

```
private Session getSession(Request request){
    return new Session(/*构造参数*/);
}

public void login(HttpRequest request){
    Session session = getSession(request);
    // ...
}

public void logout(HttpRequest request){
    Session session = getSession(request)
    session.close();
}
```

## 作用

1. 减少线程方法间的参数传递，使用 `#ThreadLocal.set` 设置值后，线程内任意一处调用 `#ThreadLocal.get()` 即可获取到值

# 原理
{% asset_img ThreadLocal原理 ThreadLocal原理.png %}

> **个人理解：**
>
> ThreadLocal 实际上相当于Thread的一个工具类，它可以通过自身去查找存储在Thread类里的对应变量并进行对数据的操作
>
> 需要注意的是，一个ThreadLocal 在一个Thread中只会保存一份局部变量，如果需要保存多个状态等信息，需要通过多个ThreadLocal来完成(比如上述例子中的session和user信息)

# 源码

> ThreadLocal 主要有以下几个方法
>
> - set()：写值
> - get() ： 读值
> - remove() ： 清除数据

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

- ThreadLocal提供的`set、get、remove`方法，都是通过`ThreadLocalMap`来完成的。

## set

```
    public void set(T value) {
        Thread t = Thread.currentThread();  // 获取当前线程
        ThreadLocalMap map = getMap(t); // 获取当前线程局部变量
        if (map != null)
            map.set(this, value);   // <1> 赋值
        else
            createMap(t, value);    // <2> 创建局部变量。代码较简单，代码贴在下面
    }

    // ↓↓↓↓↓↓↓↓↓↓ <2> 创建局部变量 ↓↓↓↓↓↓↓↓↓↓↓↓

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);  // <2.1> 关于ThreadLocal的hash值，保证了完美散列值，即每个TheadLocal的 hash 去 & (2^n-1)会完美分布在不同的下标上
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }

    // <2.1> hash值的生成
    private static AtomicInteger nextHashCode = new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);  // 生成的hash值为 1*HASH_INCREMENT、2*HASH_INCREMENT、3*HASH_INCREMENT...
    }
```

- <1> 由此可见，`ThreadLocal.set` -> `ThreadLocalMap.set`

```
        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);  // 根据hash计算应在数组中的位置

            // 采用线性探测法，寻找合适的插入位置
            for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // key 存在则直接覆盖
                if (k == key) {
                    e.value = value;
                    return;
                }
                
                // key = null，说明之前的ThreadLocal被GC回收了
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            
            // 不存在合适位置也没有旧元素，就重新创建一个
            tab[i] = new Entry(key, value);
            int sz = ++size;
            // 清除无效的entry,并当数组元素阈值大于threshold就进行扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();   // <1> 重新hash
        }
```

- `<1>`处，调用`#rehash()`方法，清理无效entry并根据数组大小进行扩容

```
        private void rehash() {
            expungeStaleEntries();  // <1.1> 清理无效数据

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();   // <1.2> 扩容
        }
```

- - `<1.1>`处，清理无效数据

```
        // 清理无效数据
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)   // 当entry!=null 且 key为null时(即ThreadLocal不再使用且被GC回收)
                    expungeStaleEntry(j);   // 清理下标为j的entry
            }
        }
        // 清理下标为stateSlot的Entry
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot 
            // 执行清理操作，value置空，entry置空，size--
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            // 遍历指定清理节点的后续所有节点
            for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {    // key 为 null，执行删除操作
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {    // 否则进行重新定位
                    int h = k.threadLocalHashCode & (len - 1);
                    // h：entry应该属于的下标位置，i：entry当前所在位置
                    if (h != i) {   // entry所属位置与它应该在的位置不一致，使用线性探测法找到它的位置，将原位置数据设置为null，插入新位置
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

- - `<1.2>`处，当数组大小 >= 四分之三阈值就进行扩容。避免迟滞

```
        // ThreadLocalMap 扩容
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;    // 设置新数组长度为原数组长度的二倍
            Entry[] newTab = new Entry[newLen]; // 创建新数组
            int count = 0;

            // <1.2.1> 遍历旧数据，插入到新数据
            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);   // 重新定位
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);   // 设置新的阈值，(size*1.5)
            size = count;   // 数组元素个数
            table = newTab; // 将新数组赋值给ThreadLocalMap的table
        }
```

- - `<1.2.1>`，将旧数据插入到新的数组中。发现无效数据，将value设置为null，帮助GC进行回收。重新定位Entry位置，利于查找。

## get

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);    // <2> 调用ThreadLocalMap.getEntry(ThreacLocal)
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();   // <1> 调用 #initialValue() 进行值的初始化并设置Thread的局部变量.
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

- `<1>` 处，当线程局部数据未进行初始化(为`null`)时，进行初始化，并设置Entry到局部数据中，逻辑和`#set` 方法一致。
- `<2>` 处，调用`#ThreadLocalMap.getEntry` 获取当前线程局部变量(即Thread内部的threadLocals保存的Entry)。然后获取当前ThreadLocal所管理的当前线程的具体指。

```
        // 获取 key(ThreadLocal对象) 所持有的当前线程的数据对象
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);   // 计算当前 key 应该位于 table 的下标位置
            Entry e = table[i];
            if (e != null && e.get() == key)    // 如果找到且数据时当前 ThreadLocal 持有的，直接返回
                return e;
            else
                return getEntryAfterMiss(key, i, e);    // <1.1> 当对应下标位置为null或者其保存的数据对象不是当前ThreadLocal持有时
        }
```

- `<1.1>` 处，当table中对应下标没找到当前ThreadLocal持有数据时，调用`#getEntryAfterMiss()`方法继续查找

```
        // 数组下标位置i 未找到 key 所持有的当前的数据对象时，继续查找。
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
            
            // 遍历 table
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)   // 找到当前对象持有数据对象，直接返回
                    return e;
                if (k == null)  // 遍历的数据对象key为null，执行清理
                    expungeStaleEntry(i);
                else    // 遍历下一数据
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;    // 找到e为null也未找到，返回null
        }
```

## remove

```
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

- 最终调用`ThreadLocalMap.remove` 代码如下：

```
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();  // Reference的clear方法,设置应用对象为null,方便 expungeStaleEntry(i); 进行数据清理
                    expungeStaleEntry(i);   // 清理数据,set方法中有讲，不再赘述
                    return;
                }
            }
        }
```

- remove方法常用于线程池中对象进行局部变量赋值。**线程池中核心线程执行完任务后，会放回线程池，****作为GC Root，线程状态不是Termainal，****不会作为可回收对象被GC回收，导致其内部ThreadLocalMap对象也不会被回收。造成内存泄漏。且下次如果使用线程没有初始化****`threadLocals`****，还会读取到上次使用时的值，造成逻辑错误（关于GC机制在JVM章节中再详细介绍），解决方法：**

1. 1. 复写线程池的 `#afterExecute()` 方法，当线程执行完成后，设置局部数据变量为null。即`Thread.currentThread().threadLocals = null;`
   2. 当ThreadLocal 线程局部数据使用结束后，调用其 `#remove` 方法，清除数据。即 `tl.remove();`



# 常见问题

## Q1: 为什么Entry要设置为WeakReference？

主要是为了方便GC回收ThreadLocal的entry中的key。后续使用ThreadLocal的方法(get、set)时可以进行value的回收。避免内存泄漏(此方法可以规避绝大部分内存泄漏，但是还存一些特殊情况需要程序支持)



## Q2：ThreadLocal 数据中的key已经设置为WeakReference为什么还会产生内存泄漏？

这里说的内存泄漏主要是指value。当线程池中的核心线程(主要是描述这个线程不会被销毁)执行任务时使用了ThreadLocal t1变量进行赋值，线程任务结束，线程被放回线程池中，不会被销毁，线程内部变量 `threadLocals`也不会被回收。 t1 不在了，指向ThreadLocal的强引用没了。只剩下key指向ThreadLocal，key是弱引用，会被GC回收，结果是key为null，value的值还在，这块数据无法使用也无法回收，造成内存泄漏。当然，后续如果当前线程继续使用ThreadLocal进行set或get时是会清理上述的无效数据的，但是在线程执行完任务和再次执行任务使用ThreadLocal之间。这个value就存在内存泄漏情况。

## Q3：为什么建议将ThreadLocal 设置为 `private static`

防止多次实例化，其内容都指向同一线程的局部数据(在同个线程内实例化)造成歧义且浪费资源。另外设置为static也是导致发生Q2内存泄漏的原因，即Static变量的生命周期和线程的生命周期一致。
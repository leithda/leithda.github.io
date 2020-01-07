---
title: Java-JUC概述
abbrlink: 2396729586
date: 2020-01-03 10:26:43
categories:
  - Java
  - Basic
tags:
  - JUC
  - 并发
  - 多线程
author: 长歌
---

{% cq %}
在 Java 5.0 提供了 java.util.concurrent （简称JUC ）包，在此包中增加了在并发编程中很常用的实用工具类，用于定义类似于线程的自定义子系统，包括线程池、异步 IO 和轻量级任务框架。提供可调的、灵活的线程池。还提供了设计用于多线程上下文中的 Collection 实现等。
{% endcq %}
<!-- More -->

## 并发理论
### 三大问题
- 原子性
  一个或多个操作，要么全部执行，要么都不执行。
  
- 可见性
  当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看到  
  java中普通变量不保证可见性,因为其在多线程情况下修改何时写入主存是不确定的  
  
  缓存优化或者硬件优化或指令重排以及编辑器的优化都可能导致一个线程修改不会立即被其他线程察觉  
  
  Java提供`volatile`关键字保证可见性,写时直接刷新到主存,读时从主存读取  
  
  加锁可以保证可见性,同一时刻只有一个线程可以操作,并在释放锁后将数据的修改刷新到主存  
  
- 有序性
  程序按照代码的先后顺序执行  
  
  为了性能(更少的寄存器操作等),编译器和运行时环境通常会对指令序列进行重排序破坏有序性  
  
  

### 重排序
​	重排序通常是编译器为了优化程序性能而对指令重新排序执行的手段。
-  避免指令重排序可以通过`synchronized`、`final`和`volatile`关键字实现。
  - synchronized 可以解决三大问题，但存在竞争情况造成线程切换，代价较大
  - `volatile`可以抑制重排序，规则如下：
    - `volatile`写之前的操作不会被重排序到`volatile`写之后
    - `volatile`读之后的操作不会被重排序到`volatile`读之前



### 内存模型JMM
>本节内容参考自:[CSDN博主「和尚要吃肉」的文章](https://blog.csdn.net/qq_39707130/article/details/93915667#JavaJMM_11)


#### 计算机内存模型
- 计算机的内存模型如下图：
{% asset_img 计算机内存模型.png 计算机内存模型 %}


#### Java 内存模型
- Java内存模型简称JMM(Java Mermory Model),是Java虚拟机所定义的一种抽象规范，用来屏蔽不同硬件和操作系统的内存访问差异，让java程序在各种平台下都能达到一致的内存访问效果。
- Java内存模型本身是一种`抽象的概念，并不真实存在`，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）（不包括局部变量与方法参数）的访问方式。

{% asset_img JMM.png Java内存模型 %}

##### 主内存和工作内存
- 主内存(Main Mermory):共享、类信息、常量、静态变量
- 本地内存(Working Memory): 工作内存——主内存中数据的副本

##### 内存间交互操作
| 指令         | 作用对象       | 作用                                                         |
| ------------ | -------------- | ------------------------------------------------------------ |
| lock(锁定)   | 主内存的变量   | 把一个变量标识为一个线程独占状态                             |
| unlock(解锁) | 主内存的变量   | 把锁定的变量释放出来，释放出来的变量才能被其他线程使用       |
| read(读取)   | 主内存的变量   | 把一个变量的值从主内存传输到线程的工作内存，后续被load使用   |
| load(载入)   | 工作内存的变量 | 把read操作得到的值放入到工作内存的变量副本中 eg：read是货车，工作内存是仓库，load把货车里东西放进仓库 |
| use(使用)    | 工作内存的变量 | 把变量值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时，将会执行这个操作 |
| assign(赋值) | 工作内存的变量 | 把use中执行引擎操作的结果赋值给工作内存的变量 eg：use好比站点进行快递员分配，站点说我把快递分给你了快递员A。快递员A接收到快递 assign开始派送。 |
| store(存储)  | 工作内存的变量 | 把工作内存的变量传送到主内存中，以便后面的write操作          |
| write(写入)  | 主内存的变量   | 把store操作中的变量值传送到主内存的变量中 eg：store和 write就好理解了，快递员A将快递送到你家门口（store），然后你得签收（write） |

以下行为具有原子性，在使用上互相依赖：

- read-load从主内存复制变量到当前工作内存
- use-assign执行代码改变共享变量值
- store-write用工作内存数据刷新主存相关内容，指令顺序不能变
- 上面操作指令顺序不能变，但是之间可以插入其他的指令，**eg**：read a，read b，load b， load a

##### 指令规则
1. 不允许read和load、store和write操作之一单独出现
2. 不允许一个线程丢弃它的最近assign的操作，即变量在工作内存中改变了之后必须同步到主内存中。
3. 不允许一个线程无原因地（没有发生过任何assign操作）把数据从工作内存同步回主内存中。
4. 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量。即就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
5. 一个变量在同一时刻只允许一条线程对其进行lock操作，lock和unlock必须成对出现
6. 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前需要重新执行load或assign操作初始化变量的值
7. 如果一个变量事先没有被lock操作锁定，则不允许对它执行unlock操作；也不允许去unlock一个被其他线程锁定的变量。
8. 对一个变量执行unlock操作之前，必须先把此变量同步到主内存中（执行store和write操作）

##### long和double型变量的特殊规则

- long和double是64位的数据类型，如果没有被volatile修饰，每次操作划分为2次32位进行操作，原子性对于这个不起作用，如果多个线程操作这样的数据，会出现数据错误。


### happens-before规则
- 先行发生原则(Happens-Before)是判断数据存在竞争、线程是否安全的主要依据
- 在Java内存模型中，Happens-Before可以翻译成：前一个操作的结果可以被后续的操作获取

- Java内存模型中存在天然的先行发生关系：

  1. 程序次序规则：`同一个线程内`，按照代码出现的顺序，前面的代码先行于后面的代码，准确的说是控制流顺序，因为要考虑到分支和循环结构。
  2. 管程锁定规则：一个unlock操作先行发生于后面（时间上）对同一个锁的lock操作。
  3. **volatile变量规则：对一个volatile变量的写操作先行发生于后面（时间上）对这个变量的读操作。**
  4. 线程启动规则：Thread的start( )方法先行发生于这个线程的每一个操作。
  5. 线程终止规则：线程的所有操作都先行于此线程的终止检测。可以通过Thread.join( )方法结束、Thread.isAlive( )的返回值等手段检测线程的终止。
  6. 线程中断规则：对线程interrupt( )方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt( )方法检测线程是否中断
  7. 对象终结规则：一个对象的初始化完成先行于发生它的finalize()方法的开始。
  8. 传递性：如果操作A先行于操作B，操作B先行于操作C，那么操作A先行于操作C。

  总结：一个操作“时间上的先发生”不代表这个操作先行发生；一个操作先行发生也不代表这个操作在时间上是先发生的(重排序的出现)



## [并发关键字](../detail/4169854470.html)

### volatile
> volatile是Java虚拟机提供的轻量级的同步机制。

- 被`volatile`修饰的共享变量对所有线程可见
- `volatile`可以抑制指令重排序.(volatile是Happens-Before的实现)
- 不具有原子性，存在线程安全问题



### final
> final 是 Java中的一个关键字，可以修饰类、方法、成员变量及局部变量。被final修饰的变量不能被修改，类不能被继承
- `final`变量的写操作不会重排序到构造函数之外（不出现`this`逃逸时）
- 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

### synchronized
> synchronized，互斥同步锁，可重入，保证了可见、有序以及原子性

## [原子操作](../detail/1710138375.html)
> jdk1.5以后，提供了`java.util.concurrent.atomic`包，里面一共提供了13个类，分为四种类型：

> 1. 基本类型
> 2. 数组
> 3. 引用
> 4. 属性工具

- AtomicInteger,AtomicIntegerArray,AtomicReference,AtomicIntegerFieldUpdater

## Lock体系
> - 在Lock接口出现之前，java靠`synchronized`关键字实现锁功能
> - Lock接口显示的拥有获取锁及释放锁的操作
> - synchronized同步块执行完成或遇到异常会自动释放，Lock需要手动执行`#lock.unlock()`方法

```java
Lock lock = new ReetrantLock();
try{
	lock.lock();
	//以下代码只有一个线程可以运行
	...
}finally{
	lock.unlock();//显式解锁
}
```

## 并发工具类
> 针对于多线程编程，juc提供以下工具类用于多线程程序的执行控制

| 工具类         | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| CountDownLatch | 倒计时器，可以保证一个或多个线程等待其他线程执行完成各自的工作后再执行 |
| CyclicBarrier  | 栅栏，阻塞一组线程直到某个事件的发生                         |
| Semaphore      | 信号量，是一个计数器，用来进行共享资源的访问控制，线程访问共享变量时需先获取信号量，信号量减一。当信号量的计数器为0时，信号量会将访问线程置为休眠直到计数器大于0. |
| Exchanger      | 数据交换，用于两个线程交换信息，当多个线程使用同个对象时，不保证信息交换次序 |
| Executors      | 线程工厂类，提供了一系列工厂方法用于创建线程池               |

## 并发容器
> - JUC提供了用于多线程中的容器类实现与高效的、可伸缩的、线程安全的非阻塞FIFO队列。
>- CAS算法(Compare And Swap):	乐观锁技术，CAS需要三个参数，变量(var)、旧的预期值(`expectedValue`)、新的更新值(`newValue`)
>   CAS指令执行时，只有变量值var=期望值expectedValue时，才将变量值更改为newValue，否则什么都不做

### List
 - **CopyOnWriteArrayList**
  - CopyOnWriteArrayList相当于线程安全的ArrayList。

### Set
- CopyOnWriteArraySet
  相当于线程安全的 HashSet，但是性能优于 HashSet
  - ConcurrentSkipListSet
    相当于线程安全的TreeSet基于 ConcurrentSkipListMap 的可缩放并发 NavigableSet 实现

### Map
- ConcurrentHashMap
  - 是线程安全的哈希表，相当于线程安全的HashMap
- ConcurrentSkipListMap
  - 是线程安全的有序的哈希表，相当于线程安全的TreeMap。


### Queue
- ArrayBlockingQueue
  是一个由基于数组的、线程安全的、有界阻塞队列
- LinkedBlockingQueue
  是一个基于单向链表的、可指定大小的阻塞队列
- LinkedBlockingDeque
  是一个基于单向链表的、可指定大小的双端阻塞队列
- ConcurrentLinkedDeque
  是一个基于双向链表的、无界的队列
- ConcurrentLinkedQueue
  是一个基于单向链表的、无界的队列

## Executor体系
> jdk1.5之后推出的Executor框架，主要目的是为了更方便的开发多线程应用，将执行单元与执行机制分离

- Executor框架包含3部分
  1. 任务，也就是工作单元，包括被执行任务需要实现的接口:`Runnable`和`Callable`。
  2. 任务的执行，执行机制，包括`Executor`接口及继承自`Executor`接口的`ExecutorService`接口。
  3. 异步计算的结果。包括`Future`接口及实现了`Future`接口的`FutureTask`类。

{% mermaid sequenceDiagram %}
participant main as 主线程
participant task as Runnable|Callable对象
participant thread as ExecutorService的实现类
participant ft as FutureTask对象

main->>task: 创建
task->>thread: execute()或者submit()
Note over thread: ThreadPoolExecuto<br>ScheduledThreadPoolExecutor
thread->>ft: return
ft->>main: get() 或者 cancel()
{% endmermaid %}
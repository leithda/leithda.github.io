---
title: Java-JUC-线程
abbrlink: 2466310928
date: 2020-01-15 10:52:08
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
内容基本照搬自: [并发番@Thread一文通(1.7版) by 黄志鹏kira](https://www.zybuluo.com/kiraSally/note/823674)  
自己手写一遍加深印象~
{% endcq %}
<!-- More -->

# 什么是线程
## 线程的概述
### 线程组成
- 一个运行程序就是一个进程,而线程是进程中独立运行的子任务
- 线程是操作系统执行流中的最小单位,一个进程可以有多个线程,这些线程与进程共享同一份内存空间
- 线程是系统独立调度和分派CPU的基本单位，通常有`就绪`、`运行`、`阻塞`三种基本状态

## 多线程的风险
### 上下文切换
> - **上下文切换**: CPU通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。但是，在切换前会保存上一个任务的状态，以便下次切换回这个任务时可以重新加载这个任务的状态。所有任务从保存到再加载的过程就是一次上下文切换
> - **多线程性能问题**:并发任务需要进行线程创建以及上下文的切换,所以效率不如串行块
> - **减少上下文切换**: 无锁并发编程,CAS算法,减少并发,使用最小线程,协程
>   - 无锁并发编程:避免使用锁,比如数据分段执行(MapReduce),尽可能使用无状态对象,避免竞争情况等
>   - CAS算法: `java.util.current`包中大量使用CAS算法,比如`Atomic`,`AQS`等等
>   - 减少并发:Java8 中新引入的`LongAdder`,`DoubleAdder`等新类,将CAS算法替换成value分担原则
>   - 使用最小线程:避免创建不必要的线程,当任务很少但线程很多时,会导致大量线程为等待状态
>   - 协程:在单线程里实现多任务的调度,并在单线程里维持多个任务间的切换
> - **补充**: 需要注意的是,Java的线程是映射到操作系统的原生线程上,因此需要阻塞或唤醒一个线程都需要操作系统的协助,这就意味着要从用户态转换到核心态,因此状态转化是非常耗费处理器时间的

### 竞争条件

> - 竞争条件指多个线程并发访问和操作同一数据且执行结果与线程访问的特定顺序有关
> - 竞争条件发生在当多个线程读写数据时,其最终的结果依赖于读个线程指令执行顺序
> - 由于竞争条件的指令顺序操作的不确定性甚至是错误的,可能造成结果的混乱.录入臭名昭著的i++的原子性问题

### 活跃性失败问题

> - **饥饿(Starvation)**: 饥饿是指某一个或多个线程因为种种原因无法获得所需要的资源,导致一直无法执行,如:
>   - 该线程优先级低,而高优先级的线程不断抢占它的资源,导致低优先级线程一直无法正常工作
>   - 比如某个线程长时间占用关键资源不释放,导致其他线程不可用
> - **活锁(Livelock)**:活锁是各个线程都会主动释放资源,导致资源不断在两个线程中跳动(同时线程状态也在不停地切换).而没有一个线程可以同时拿到所有的资源而正常执行(如`yield`方法)
> - **死锁(Deadlock)**: 线程间互相等到对象释放锁而形成死循环,产生死锁
> - **避免死锁的几个常见方法**:
>   - 避免一个线程同时获取多个锁
>   - 避免一个线程在锁内占用多个资源.尽量保证每个锁只占用一个资源
>   - 尝试使用定时锁,使用`Lock.tryLock(timeout)`来替代内部锁机制
>   - 对于数据库锁,加锁和解锁必须在一个数据库连接里,否则会出现解锁失败的情况

## 并发级别

### 阻塞(Blocking)

> - 线程若是阻塞的,那么在其他线程释放资源之前,当前线程会被挂起,无法继续执行
> - 在Java中,使用`synchronized`关键字或者`重入锁`时,得到的就是阻塞的线程
> - 无论是 `synchronized` 还是重入锁，都会试图在执行后续代码之前，竞争临界区的锁：
>   - 如果竞争成功，当前线程会获得锁并占用资源，从而继续往后执行
>   - 如果竞争失败，继续挂起阻塞，等待下次资源被释放后的再次竞争

### 无饥饿(Starvation-Free)

> - 若线程间区分优先级，那么线程调度通常会优先满足高优先级的线程（非公平原则），就可能产生饥饿
> - 对于非公平的锁来说，系统允许高优先级的线程插队，这样就可能导致低优先级线程产生饥饿
> - 对于公平的锁来说，不管新到的线程优先级多高都需要乖乖排队，所有线程都有机会执行，饥饿很难产生
> - 当一个任务非常耗时导致某线程一直占据关键资源不放，其他线程难以获取，此时其他线程也可以说是饥饿的

### 无障碍(Obstruction-Free)

> - 无障碍是最弱的非阻塞调度：当两个线程无障碍执行，那么就不会以为临界区问题导致一方被挂起，而是共享临界区
> - 此时若是一个线程对临界区共享数据进行修改操作，对于无障碍线程来说，一旦检测到这种情况，会立即对自己所做的修改进行回滚，确保数据安全（类似事务的概念）
> - 无障碍类似于乐观锁，线程在操作之前，先读取并保存一个"一致性标记"，操作完成后，再次读取，检查这个标记是否被变更过，如果两者一致，说明没有冲突；否则就有冲突，会重试或者回滚（跟Git差不多，区别是Git需要手动merge或者revert）

### 无锁(Lock-Free)

> - 无锁的并行都是无障碍。在无锁情况下，所有线程都能尝试对临界区进行访问，区别是无锁的并发保证必然有一个线程能够在有限步内完成操作并离开临界区
> - 如果线程一直尝试不成功，可能会出现饥饿的情况，这样常见于无穷循环调用

### 无等待(Wait-Free)

> - 无等待是在无锁的基础上拓展，它要求所有的线程都必须在有限步内完成，这样就不会有饥饿问题
> - 一个典型的无等待结构是`RCU(Read-Copy-Update)`。它的基本思想是：读操作都是无等待，写操作时，需要先获取原始数据副本，接着只修改副本数据，修改完后，在合适的时机回写数据
> - RCU在JAVA中应用的非常广泛，比如(JAVA的观察者模式的JDK实现)`Observable` 的 `notifyObservers()`方法，`ArrayList` 的 `resize()`方法等等

## Java线程状态的转换

### Java线程状态转换图

{% asset_img Java线程状态转换.jpg Java线程状态转换图 %}

### Java线程状态转换

> - **新建(new)**: 新创建一个线程对象,在Java中的表现就是`Thread thread = new Thread();`
> - **就绪(runnable)**: 线程创建后,其他线程调用该对象的`#start()`方法.该状态的线程位于可运行线程池中,`等待被线程调度选中,获取CPU时间分片使用权`,在Java中的表现是`thread.start();`
> - **运行(running)**:就绪态线程获取到CUP时间分片之后,就可以执行任务,在Java中的表现就是`thread.run()`,需要注意的是线程不一定是立即执行,这与系统调度有关
> - **阻塞(block)**:阻塞状态是指线程因为某种原因`放弃CPU使用权,让出CPU时间分片,暂时停止运行`,直到线程进入就绪态,才有机会再次获取时间片转为运行态.阻塞分三种情况:
>   - **常规阻塞**:运行态线程在`发出I/O请求`、执行`Thread.sleep()`方法，`t.join方法`时，JVM会将该线程设置为阻塞状态；当`I/O处理完毕并发出响应`、`sleep方法超时`、`join等到线程终止或超时`线程重新转入就绪态
>   - **同步阻塞**：运行太线程阿紫获取对象的同步锁时，当锁被其他线程占用时，JVM会将该线程暂时放入同步队列中，当其他线程释放锁且该线程竞争到该同步锁时，该线程转为就绪态
>   - **等待阻塞**：运行态执行`#wait()方法`，JVM会将该线程放入等待队列中，直到被其他线程唤醒或其他线程中断或超时，再次获取同步锁表示，重新进入就绪态
> - **死亡(dead)**:线程`main方法`、`run方法`执行完毕或者出现异常，则该线程生命周期结束。处于死亡或终结状态的线程不可再次被调度，不可被分配到时间片，此时其寄存器上下文和栈都将被释放。可以使用`线程池机制`来提高线程的复用性，避免线程被直接杀死；

### Java线程状态枚举

```java
/** 
  * JAVA对于线程状态的枚举类，使用jstack查看dump文件可以看到相对应的线程状态
  * 注意：状态的转换要以状态图为参照标准，枚举类只是用来统一记录某种状态以方便JAVA代码编写！
  * 对应JVM中对线程的监控的4种核心抽象状态：
  *     运行(running)，休眠(Sleeping)，等待(Wait)，监视(Monitor)
  */
public enum State {
    /**
      * Thread state for a thread which has not yet started.
      *     新建：线程创建但还不是就绪态（Thread还没有执行start方法）
      */
        NEW,
    /**
      * Thread state for a runnable thread.  A thread in the runnable
      * state is executing in the Java virtual machine but it may be waiting  
      * for other resources from the operating system such as processor.
      *     运行状态：Java将就绪态和运行态统一设置为RUNNABLE
      *     笔者认为这可能与Thread执行start方法之后会立即执行run方法有关
      */
    RUNNABLE,
    /**
      * Thread state for a thread blocked waiting for a monitor lock.
      * A thread in the blocked state is waiting for a monitor lock to enter 
      * a synchronized block/method or reenter a synchronized block/method
      * after calling {@link Object#wait() Object.wait}.
      *     阻塞：线程正等待获取监视锁(同步锁)，调用wait方法就会阻塞当前线程
      *     只有获取到锁的线程才能进入或重入同步方法或同步代码
      */
    BLOCKED,
    /**
      * Thread state for a waiting thread.
      * A thread is in the waiting state due to calling one of the following methods:
      * <ul>
      *   <li>{@link Object#wait() Object.wait} with no timeout</li>
      *   <li>{@link #join() Thread.join} with no timeout</li>
      *   <li>{@link LockSupport#park() LockSupport.park}</li>
      * </ul>
      *     调用以上方法会使线程进入等待状态
      * <p>A thread in the waiting state is waiting for another thread to
      * perform a particular action.
      *     进入等待状态的线程，需要等待其他线程的唤醒才能继续运行
      * For example, a thread that has called <tt>Object.wait()</tt>
      * on an object is waiting for another thread to call
      * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
      * that object. A thread that has called <tt>Thread.join()</tt>
      * is waiting for a specified thread to terminate.
      *     比如线程被wait方法执行等待，需要被notify或notifyAll唤醒
      *     再比如join方法会等待一个指定线程结束之后才会继续运行
      */
    WAITING,
    /**
      * Thread state for a waiting thread with a specified waiting time.
      * A thread is in the timed waiting state due to calling one of
      * the following methods with a specified positive waiting time:
      * <ul>
      *   <li>{@link #sleep Thread.sleep}</li>
      *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
      *   <li>{@link #join(long) Thread.join} with timeout</li>
      *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
      *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
      * </ul>
      *     调用以上方法会使线程进入等待状态，但会超时返回
      */
    TIMED_WAITING,
    /**
      * Thread state for a terminated thread.
      * The thread has completed execution.
      *     终止：当线程任务完成之后，就终止了
      */
    TERMINATED;
    }
```

- 枚举类图表如下图所示

| 状态名称     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NEW          | 初始状态，线程被构建，但是还没有调用`start()`方法            |
| RUNNABLE     | 运行状态，Java线程将操作系统中的就绪和运行两种状态笼统地统称为“运行中” |
| BLOCKED      | 阻塞状态，表示线程阻塞于锁                                   |
| WAITING      | 等待状态，表示线程进入等待状态，进入该状态表示当前线程需要其他线程做出一些特定动作(通知或中断) |
| TIME_WAITING | 超时等待状态，它可以在指定的时间自行返回                     |
| TERMINATED   | 终止状态，表示当前线程已经执行完毕                           |

- 调用与线程有关的方法是造成线程状态改变的主要原因，枚举类的转换如下图所示：

{% asset_img 线程转换.png 线程转换 %}

### Synchronized Vs Lock

> - 阻塞状态是线程阻塞在进入同步方法或同步块(获取锁)时的状态
> - `concurrent`包下`Lock`接口的线程状态是等待状态，因为阻塞实现是基于`LockSupport类`方法

### Dump文件中的线程状态

> - **死锁: DeadLock(重点关注)**
> - **等待资源：Waiting on condition(重点关注)**
> - **等待获取监视器: Waiting on monitor entry(重点关注)**
> - **阻塞: Blocked(重点关注)**
> - 执行中： Runnable
> - 暂停: Suspended
> - 对象等待中: Object.wait() 或 TIMED_WAITING
> - 停止： Parked

### JVM监控工具

> - Java VisualVM 是 Java自带的工具(1.6开始)，同时提供可视化的GUI界面操作，位于bin包下
> - 可以通过该工具查看整个JVM状态，尤其是线程状态(Thread Dump)
> - 此工具可以实现远程监控，通过指定端口以及IP远程查看JVM线程状态

{% asset_img Java_VisualVM.png JVM监控工具 %}

**致谢：**[Java线程Dump分析工具--jstack](http://www.cnblogs.com/nexiyi/p/java_thread_jstack.html)

# 什么是Thread

- Java中实现多线程编程主要有两种方式： 一是继承`Thread类`，二是实现`Runnable接口`
- `Thread`所示JDK对线程的抽象实现类，封装了有关线程的一些基本信息和操作
- JVM允许一个应用有多个线程并发执行，优先级高的线程优先执行
- JVM启动后，会创建一些守护线程来进行自身的常规管理(垃圾回收，终结处理)，以及一个运行`main`函数的主线程
- 当满足以下其中一个条件时，JVM将停止线程运行：
  1. Runtime执行`exit()`方法同时安全管理器允许该操作被执行
  2. 所有非守护线程全部死亡或`run`方法返回或在执行`run`方法是抛出异常并传播
- Bruce Eckel(《Java编程思想》作者认为`Runnable`应该更名为`Task`)

# Thread的数据结构

## 类定义

```java
//Thread类
public class Thread implements Runnable
//Runnable接口
public interface Runnable {
    public abstract void run();
}
```



## 构造器

```java
/**
  * 默认构造器
  * 其中name规则为 "Thread-" + nextThreadNum()
  */
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
/**
  * 创建一个指定Runnable的线程
  * 其中name规则为 "Thread-" + nextThreadNum()
  */
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
/**
  * 创建一个指定所属线程组和Runnable的线程
  * 其中name规则为 "Thread-" + nextThreadNum()
  */
public Thread(ThreadGroup group, Runnable target) {
    init(group, target, "Thread-" + nextThreadNum(), 0);
}
/**
  * 创建一个指定name线程
  */
public Thread(String name) {
    init(null, null, name, 0);
}
/**
  * 创建一个指定所属线程组和name的线程
  */
public Thread(ThreadGroup group, String name) {
    init(group, null, name, 0);
}
/**
  * 创建一个指定Runnable和name的线程
  */
public Thread(Runnable target, String name) {
    init(null, target, name, 0);
}
/**
  * Allocates a new {@code Thread} object so that it has {@code target}
  * as its run object, has the specified {@code name} as its name,
  * and belongs to the thread group referred to by {@code group}.
  *     创建一个新的Thread对象，同时满足以下条件：
  *         1.该线程拥有一个指定的Runnable对象用于方法执行
  *         2.该线程具有一个指定的名称
  *         3.该线程属于一个指定的线程组ThreadGroup
  * <p>If there is a security manager, its
  * {@link SecurityManager#checkAccess(ThreadGroup) checkAccess}
  * method is invoked with the ThreadGroup as its argument.
  *     若这里有个安全管理器，则ThreadGroup将调用checkAccess方法进而触发SecurityManager的checkAccess方法
  * <p>In addition, its {@code checkPermission} method is invoked with
  * the {@code RuntimePermission("enableContextClassLoaderOverride")}
  * permission when invoked directly or indirectly by the constructor of a subclass which
  * overrides the {@code getContextClassLoader} or {@code setContextClassLoader} methods.
  *     当enableContextClassLoaderOverride被开启时，checkPermission将被重写子类直接或间接地调用
  * <p>The priority of the newly created thread is set equal to the
  * priority of the thread creating it, that is, the currently running
  * thread. The method {@linkplain #setPriority setPriority} may be
  * used to change the priority to a new value.
  *     新创建的Thread的优先级等同于创建它的线程的优先级，调用setPriority会变更其优先级
  * <p>The newly created thread is initially marked as being a daemon
  * thread if and only if the thread creating it is currently marked
  * as a daemon thread. The method {@linkplain #setDaemon setDaemon}
  * may be used to change whether or not a thread is a daemon.
  *     当且仅当线程创建时被显示地标记为守护线程，新创建的线程才会被初始化为一个守护线程
  *     setDaemon方法可以设置当前线程是否为守护线程
  */
public Thread(ThreadGroup group, Runnable target, String name) {
    init(group, target, name, 0);
}
/**
  * Allocates a new {@code Thread} object so that it has {@code target} as its run object,
  * has the specified {@code name} as its name, and belongs to the thread group referred to
  * by {@code group}, and has the specified <i>stack size</i>.
  *     创建一个新的Thread对象，同时满足以下条件：
  *         1.该线程拥有一个指定的Runnable对象用于方法执行
  *         2.该线程具有一个指定的名称
  *         3.该线程属于一个指定的线程组ThreadGroup
  *         4.该线程拥有一个指定的栈容量
  * <p>This constructor is identical to {@link #Thread(ThreadGroup,Runnable,String)}
  * with the exception of the fact that it allows the thread stack size to be specified. 
  * The stack size is the approximate number of bytes of address space that the virtual machine
  * is to allocate for this thread's stack. <b>The effect of the {@code stackSize} parameter,
  * if any, is highly platform dependent.</b> 
  *     栈容量指的是JVM分配给该线程的栈的地址(内存)空间大小，这个参数的效果高度依赖于JVM运行平台 
  * <p>On some platforms, specifying a higher value for the {@code stackSize} parameter may allow
  * a thread to achieve greater  recursion depth before throwing a {@link StackOverflowError}.
  * Similarly, specifying a lower value may allow a greater number of threads to exist
  * concurrently without throwing an {@link OutOfMemoryError} (or other internal error).
  * The details of the relationship between the value of the <tt>stackSize</tt> parameter
  * and the maximum recursion depth and concurrency level are platform-dependent.
  * <b>On some platforms, the value of the {@code stackSize} parameter
  * may have no effect whatsoever.</b>
  *     在一些平台上，栈容量越高,(会在栈溢出之前)允许线程完成更深的递归(换句话说就是栈空间更深) 
  *     同理，若栈容量越小，(在抛出内存溢出之前)允许同时存在更多的线程数
  *     对于栈容量、最大递归深度和并发水平之间的关系依赖于平台
  * <p>The virtual machine is free to treat the {@code stackSize} parameter as a suggestion. 
  * If the specified value is unreasonably low for the platform,the virtual machine may instead 
  * use some platform-specific minimum value; if the specified value is unreasonably  high, 
  * the virtual machine may instead use some platform-specific maximum. 
  * Likewise, the virtual machine is free to round the specified value up or down as it sees fit
  * (or to ignore it completely).
  *     JVM会将指定的栈容量作为一个参考依据，但当小于平台最小值时会直接使用最小值，最大值同理
  *     同样，JVM会动态调整栈空间的大小以适应程序的运行或者甚至直接就忽视该值的设置
  * <p><i>Due to the platform-dependent nature of the behavior of this constructor, extreme care
  * should be exercised in its use.The thread stack size necessary to perform a given computation 
  * will likely vary from one JRE implementation to another. In light of this variation, 
  * careful tuning of the stack size parameter may be required,and the tuning may need to
  * be repeated for each JRE implementation on which an application is to run.</i>
  *     简单总结一下就是：这个值严重依赖平台，所以要谨慎使用，多做测试验证
  * @param  group
  *         the thread group. If {@code null} and there is a security
  *         manager, the group is determined by {@linkplain
  *         SecurityManager#getThreadGroup SecurityManager.getThreadGroup()}.
  *         If there is not a security manager or {@code
  *         SecurityManager.getThreadGroup()} returns {@code null}, the group
  *         is set to the current thread's thread group.
  *             当线程组为null同时有个安全管理器，该线程组由SecurityManager.getThreadGroup()决定
  *             当没有安全管理器或getThreadGroup为空，该线程组即为创建该线程的线程所属的线程组
  * @param  target
  *         the object whose {@code run} method is invoked when this thread
  *         is started. If {@code null}, this thread's run method is invoked.
  *             若该值为null，将直接调用该线程的run方法（等同于一个空方法）
  * @param  name
  *         the name of the new thread
  * @param  stackSize
  *         the desired stack size for the new thread, or zero to indicate
  *         that this parameter is to be ignored.
  *             当栈容量被设置为0时，JVM就会忽略该值的设置
  * @throws  SecurityException
  *          if the current thread cannot create a thread in the specified thread group
  *             如果当前线程在一个指定的线程组中不能创建一个新的线程时将抛出 安全异常
  * @since 1.4
  */    
public Thread(ThreadGroup group, Runnable target, String name,long stackSize) {
    init(group, target, name, stackSize);
}
```



## JVM栈异常分类

​	根据栈异常的不同，主要有两种分类：

​	栈溢出：若线程请求的栈深度大于虚拟机所允许的深度，将抛出`StackOverflowError`异常  

​	内存溢出： 若虚拟机栈可以动态扩展(大部分Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈)，如果扩展时无法申请到足够的内存，就会跑出`OutOfMemoryError`异常



## 重要变量

```java
//线程名，用char来保存（String底层实现就是char）
private char        name[];
//线程优先级
private int         priority;
//不明觉厉
private Thread      threadQ;
//不明觉厉
private long        eetop;
/* Whether or not to single_step this thread. 不明觉厉*/
private boolean     single_step;
/* Whether or not the thread is a daemon thread. 是否是守护线程，默认非守护线程*/
private boolean     daemon = false;
/* JVM state. 是否一出生就领便当，默认false*/
private boolean     stillborn = false;
/* What will be run. Thread的run方法最终会调用target的run方法*/
private Runnable target;
/* The group of this thread. 当前线程的所属线程组*/
private ThreadGroup group;
/* The context ClassLoader for this thread 当前线程的ClassLoader*/
private ClassLoader contextClassLoader;
/* The inherited AccessControlContext of this thread 当前线程继承的AccessControlContext*/
private AccessControlContext inheritedAccessControlContext;
/* For autonumbering anonymous threads. 给匿名线程自动编号，并按编号起名字*/
private static int threadInitNumber;
/* ThreadLocal values pertaining to this thread. This map is maintained by the ThreadLocal class. 
 * 当前线程附属的ThreadLocal，而ThreadLocalMap会被ThreadLocal维护（ThreadLocal会专门分析）
 */
ThreadLocal.ThreadLocalMap threadLocals = null;
/*
 * InheritableThreadLocal values pertaining to this thread. This map is
 * maintained by the InheritableThreadLocal class.
 * 主要作用：为子线程提供从父线程那里继承的值
 * 在创建子线程时，子线程会接收所有可继承的线程局部变量的初始值，以获得父线程所具有的值
 * 创建一个线程时如果保存了所有 InheritableThreadLocal 对象的值，那么这些值也将自动传递给子线程
 * 如果一个子线程调用 InheritableThreadLocal 的 get() ，那么它将与它的父线程看到同一个对象
 */
 ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
/*
 * The requested stack size for this thread, or 0 if the creator did not specify a stack size. 
 * It is up to the VM to do whatever it likes with this number; some VMs will ignore it.
 * 栈容量：当设置为0时，JVM会忽略该值；该值严重依赖于JVM平台，有些VM甚至会直接忽视该值
 * 该值越大，线程栈空间变大，允许的并发线程数就越少；该值越小，线程栈空间变小，允许的并发线程数就越多
 */
private long stackSize;
/* JVM-private state that persists after native thread termination.*/
private long nativeParkEventPointer;
/* Thread ID. 每个线程都有专属ID，但名字可能重复*/
private long tid;
/* For generating thread ID 用于ID生成，每次+1*/
private static long threadSeqNumber;
/* 
 * Java thread status for tools,initialized to indicate thread 'not yet started'
 * 线程状态 0仅表示已创建
 */
private volatile int threadStatus = 0;
/**
  * The argument supplied to the current call to java.util.concurrent.locks.LockSupport.park.
  * Set by (private) java.util.concurrent.locks.LockSupport.setBlocker
  * Accessed using java.util.concurrent.locks.LockSupport.getBlocker
  * 主要是提供给 java.util.concurrent.locks.LockSupport该类使用
  */
volatile Object parkBlocker;
/* The object in which this thread is blocked in an interruptible I/O operation, if any. 
 * The blocker's interrupt method should be invoked after setting this thread's interrupt status.
 * 中断阻塞器：当线程发生IO中断时，需要在线程被设置为中断状态后调用该对象的interrupt方法
 */
private volatile Interruptible blocker;
//阻塞器锁，主要用于处理阻塞情况
private final Object blockerLock = new Object();
 /* The minimum priority that a thread can have. 最小优先级*/
public final static int MIN_PRIORITY = 1;
/* The default priority that is assigned to a thread. 默认优先级*/
public final static int NORM_PRIORITY = 5;
/* For generating thread ID 最大优先级*/
public final static int MAX_PRIORITY = 10;
/* 用于存储堆栈信息 默认是个空的数组*/
private static final StackTraceElement[] EMPTY_STACK_TRACE = new StackTraceElement[0];
private static final RuntimePermission SUBCLASS_IMPLEMENTATION_PERMISSION =
            new RuntimePermission("enableContextClassLoaderOverride");
// null unless explicitly set 线程异常处理器，只对当前线程有效
private volatile UncaughtExceptionHandler uncaughtExceptionHandler;
// null unless explicitly set 默认线程异常处理器，对所有线程有效
private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
```



## 本地方法

```java
/* 
 * Make sure registerNatives is the first thing <clinit> does. 
 * 确保clinit最先调用该方法：所有该方法是类中的最靠前的一个静态方法
 * clinit：在JVM第一次加载class文件时调用，用于静态变量初始化语句和静态块的执行
 * 所有的类变量初始化语句和类型的静态初始化语句都被Java编译器收集到该方法中
 * 
 * registerNatives方法被native修饰，即是本地方法，将由C/C++去完成，并被编译成了.dll，供JAVA调用
 * 其主要作用是将C/C++中的方法映射到Java中的native方法，实现方法命名的解耦
 */
private static native void registerNatives();
static {
    registerNatives();
}
/** 主动让出CPU资源,当时可能又立即抢到资源 **/
public static native void yield();
/** 休眠一段时间，让出资源但是并不会释放对象锁 **/
public static native void sleep(long millis) throws InterruptedException;
/** 检查 线程是否存活 **/
public final native boolean isAlive();
/** 检查线程是否中断 isInterrupted() 内部使用 **/
private native boolean isInterrupted(boolean ClearInterrupted);
/** 返回当前执行线程 **/
public static native Thread currentThread();
public static native boolean holdsLock(Object obj);
private native void start0();
private native void setPriority0(int newPriority);
private native void stop0(Object o);
private native void suspend0();
private native void resume0();
private native void interrupt0();
private native void setNativeName(String name);
```



# 线程运行

## init方法

```java
/**
  * Initializes a Thread.
  *     初始化一个线程
  * @param g the Thread group
  * @param target the object whose run() method gets called
  * @param name the name of the new Thread
  * @param stackSize the desired stack size for the new thread, or
  *        zero to indicate that this parameter is to be ignored.
  */
private void init(ThreadGroup g, Runnable target, String name,long stackSize) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
    //返回当前线程，即创建该hread的线程 currentThread是个本地方法
    Thread parent = currentThread();
    //安全管理器根据Java安全策略文件决定将哪组权限授予类
    //如果想让应用使用安全管理器和安全策略，可在启动JVM时设定-Djava.security.manager选项
    //还可以同时指定安全策略文件
    //如果在应用中启用了Java安全管理器，却没有指定安全策略文件，那么Java安全管理器将使用默认的安全策略
    //它们是由位于目录$JAVA_HOME/jre/lib/security中的java.policy定义的
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
        /* Determine if it's an applet or not */
        /* If there is a security manager, ask the security manager what to do. */
        if (security != null) {
            g = security.getThreadGroup();
        }
        /* If the security doesn't have a strong opinion of the matter use the parent thread group. */
        if (g == null) {
            g = parent.getThreadGroup();
        }
    }
    /* checkAccess regardless of whether or not threadgroup is explicitly passed in. */
    //判断当前运行线程是否有变更其线程组的权限
    g.checkAccess();
    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }
    //新建线程数量计数+1 或者说未就绪线程数+1==> nUnstartedThreads++;
    g.addUnstarted();
    this.group = g;
    this.daemon = parent.isDaemon();//若当前运行线程是守护线程，新建线程也是守护线程
    this.priority = parent.getPriority();//默认使用当前运行线程的优先级
    this.name = name.toCharArray();
    //设置contextClassLoader
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext = AccessController.getContext();
    this.target = target;
    setPriority(priority);//若有指定的优先级，使用指定的优先级
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    /* Stash the specified stack size in case the VM cares */
    this.stackSize = stackSize;
    /* Set thread ID 线程安全，序列号每次同步+1*/
    tid = nextThreadID();
}
private static synchronized long nextThreadID() {
    return ++threadSeqNumber;
}
```

## start方法

```java
/**
  * Causes this thread to begin execution; the Java Virtual Machine
  * calls the <code>run</code> method of this thread.
  *     线程进入就绪态，随后JVM将会调用这个线程run方法
  *     当获取到CPU时间片时，会立即执行run方法，此时线程会直接变成运行态
  * <p>
  * The result is that two threads are running concurrently: the
  * current thread (which returns from the call to the
  * <code>start</code> method) and the other thread (which executes its
  * <code>run</code> method).
  * <p>
  * It is never legal to start a thread more than once.
  * In particular, a thread may not be restarted once it has completed execution.
  *     一个线程只能被start一次，特别是线程不会在执行完毕后重新start
  *     当线程已经start了，再次执行会抛出IllegalThreadStateException异常
  * @exception  IllegalThreadStateException  if the thread was already started.
  * @see        #run()
  * @see        #stop()
  */
public synchronized void start() {
    /**
      * This method is not invoked for the main method thread or "system"
      * group threads created/set up by the VM. Any new functionality added
      * to this method in the future may have to also be added to the VM.
      * 该方法不会被主线程或系统线程组调用，若未来有新增功能，也会被添加到VM中
      * A zero status value corresponds to state "NEW".
      * 0对应"已创建"状态 -> 用常量或枚举标识多好
      */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();
    /* Notify the group that this thread is about to be started
     * so that it can be added to the group's list of threads
     * and the group's unstarted count can be decremented. */
    //通知所属线程组该线程已经是就绪状态，因而可以被添加到该线程组中
    //同时线程组的未就绪线程数需要-1，对应init中的+1
    group.add(this);
    boolean started = false;
    try {
        //调用本地方法，将内存中的线程状态变更为就绪态
        //同时JVM会立即调用run方法,获取到CPU之后，线程变成运行态并立即执行run方法
        start0();
        started = true;//标记为已开启
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);//如果变更失败，要回滚线程和线程组状态
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
                 it will be passed up the call stack */
            //如果start0出错，会被调用栈直接通过
        }
    }
}
-------------
//start之后会立即调用run方法
Thread t = new Thread( new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
},"kira");
t.start();//kira
```



## run方法

```java
 /**
   * If this thread was constructed using a separate <code>Runnable</code> run object,
   * then that <code>Runnable</code> object's <code>run</code> method is called;
   * otherwise, this method does nothing and returns.
   * Subclasses of <code>Thread</code> should override this method.
   *    若Thread初始化时有指定Runnable就执行其的run方法，否则doNothing
   *    该方法必须被子类实现
   */
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```



## isAlive方法

```java
/**
  * Tests if this thread is alive. A thread is alive if it has
  * been started and has not yet died.
  *     测试线程是否处于活动状态
  *     活动状态：线程处于正在运行或者准备开始运行状态
  * @return  <code>true</code> if this thread is alive;
  *          <code>false</code> otherwise.
  */
public final native boolean isAlive();
```



# 线程暂停

## Java线程暂停的方法

> - 暂停线程意味着线程还可以重新恢复
> - Java提供多种暂停的方式:`suspend()和resume()`组合、`sleep()`、`join()`、`yield()`、`wait()和notify()组合`等
> - 其中`join()`与`wait()和notify()`将放在`线程间通信`中进一步讲解
> - `suspend()和resume()组合`作为线程间的通信方式已被官方弃用，官方推荐使用`wait()和notify()组合`替换前者



### suspend方法和resume方法

```java
/**
 *  暂停当前线程
 *   - 若当前线程还活着，将被暂停同时直到被恢复之前都不会做任何事
 *   - 在操作在执行之前必须获取锁
 *   - 该操作在暂停线程的同时并不会释放锁
 */
@Deprecated
public final void suspend() {
    checkAccess();
    suspend0();
}
/**
 * 恢复一个被暂停的线程
 *   - 若线程活着同时被暂停，恢复它同时允许它继续执行后续行为
 *   - 由于执行之前suspend会先持有锁，当resume发生在加锁后但suspend前，
 *     会产生死锁 -> suspend会一直占据锁资源；同时线程状态仍为RUNNABLE
 */
@Deprecated
public final void resume() {
    checkAccess();
    resume0();
}
```



### suspend和resume的问题

- 独占问题

```java
public class TestThread {
    synchronized void func1(){
        System.out.println("开始任务");
        if(Thread.currentThread().getName().equals("leithda")){
            System.out.println("leithda线程执行suspend方法并永远暂停");
            Thread.currentThread().suspend();
        }
    }
    
    /*
    suspend和resume使用不当容易造成公共的同步对象的独占，使其他线程无法访问公共同步对象
     */
    private static void test1() {
        final TestThread thread = new TestThread();

        Thread t1 = new Thread(thread::func1,"leithda");
        t1.start();
        Thread t2 = new Thread(()->{
            System.out.println("test线程启动，但无法执行echo方法");
            // echo方法被leithda线程锁定
            thread.func1();;
        },"test");
        t2.start();
    }
}
```

- 不同步问题

```java
public class TestThread {

    private String value = "战狼1";
    public String getValue(){
        return value;
    }

    public void setValue(String value){
        if(Thread.currentThread().getName().equals("leithda")){
            System.out.println("停止leithda线程");
            Thread.currentThread().suspend();
        }
        this.value = value;
    }
    
    /**
     * 由于线程暂停导致的不同步问题
     */
    private static void test2(){
        final TestThread thread = new TestThread();
        new Thread(()->{
            thread.setValue("战狼2");
        },"leithda").start();

        new Thread(()->{
            System.out.println(thread.getValue());
        },"test").start();
    }
```



### sleep方法

```java
/**
 * 使线程睡眠一段毫秒时间，但线程并不会丢失已有的任何监视器    
 */
public static void sleep(long millis, int nanos) throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException("nanosecond timeout value out of range");
    }
    //换算用
    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }
    sleep(millis);
}
/** 我们一般会直接调用native方法，这或许是我们主动使用的最多次的native方法了 **/
public static native void sleep(long millis) throws InterruptedException;
```



### yield方法

```java
/**
  *  暗示线程调度器当前线程将释放自己当前占用的CPU资源
  *  - 线程调度器会自由选择是否忽视此暗示
  *  - 该方法会放弃当前的CPU资源，将它让给其他的任务去占用CPU执行时间
  *  - 但放弃的时间不确定，可能刚刚放弃又获得CPU时间片
  *  该方法的适合使用场景比较少，主要用于Debug，比如Lock包设计
  */
public static native void yield();
```










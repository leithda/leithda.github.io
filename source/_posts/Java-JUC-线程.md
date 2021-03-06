---
title: Java-JUC-线程
abbrlink: 2466310928
date: 2020-01-15 10:52:08
categories:
  - Java
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
},"leithda");
t.start();//leithda
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

# 线程中断
## 终止方式
Java终止线程主要有三个手段：
- **自动终止:** 使用退出标识，使线程正常退出，即当`run()`方法完成之后线程自动终止
- **强行终止:** 使用`stop()`方法强行终止,该方法已经弃用,因为使用这个方法可能产生不可预料的后果.
- **手动终止:** 使用`interrupt()`方法中断线程,并在获取到线程中断时结束(退出)任务

## 中断机制
由于Java中无法立即停止一个线程,而停止操作很重要,因此Java提供了一种用于停止线程的机制,即**中断机制**
- **中断状态:** 在Java中每个线程会维护一个Boolean中断状态位,用来表明当前线程是否被中断,默认非中断false
- **中断方法:** 中断仅仅只是一种协作方式,JDK仅提供设置中断状态和判断是否中断的方法,如`interrupted()`和`isInterrupted()`
- **中断过程:** 由于JDK只负责检测和更新中断状态,因此中断过程必须有程序员自己实现,包括中断捕获、中断处理，因此如何优雅的处理中断变得尤为重要


## 中断方法
JDK仅提供检测和更新状态位的方法，注意静态方法将清除中断状态
-  Thread.currentThread.isInterrupted(): 判断是否中断，对象方法，不会清除中断状态
-  Thread.interrupted(): 判断是否中断，静态方法，会清除中断状态
-  Thread.currentThread().interrupt(): 中断操作，对象方法，会将线程的中断状态设置为ture，仅此而已(不会真正中断线程)，捕获和处理中断由程序员自行实现

**中断后: 线程中断后的结果是死亡、等待新的任务或是继续运行至下一步，取决于程序本身

### interrupt方法
```java
/**
 * 中断一个线程(实质是设置中断标志位，标记中断状态)
 *   - 线程只能被自己中断，否则抛出SecurityException异常
 *   - 特殊中断处理如下：
 *     1.若中断线程被如下方法阻塞，会抛出InterruptedException同时清除中断状态：
 *     Object.wait()、Thread.join() or Thread.sleep()
 *
 *     2.若线程在InterruptibleChannel上发生IO阻塞，该通道要被关闭并将设置中断状态同时抛出ClosedByInterruptException异常
 *
 *     3.若线程被NIO多路复用器Selector阻塞，会设置中断状态且从select方法中立即返回一个非0值(当wakeup方法正好被调用时)
 *
 *   - 非上述情况都会将线程状态设置为中断
 *   - 中断一个非活线程不会有啥影响
 */
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();
    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            // Just to set the interrupt flag
            // 调用interrupt方法仅仅是在当前线程中打了一个停止的标记，并不是真的停止线程！
            interrupt0();           
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```

### isInterrupted方法
```java
/**
 *   检测线程是否是中断状态，但不清除状态标志
 */
public boolean isInterrupted() {
    //会调用本地isInterrupted方法，同时不清除状态标志
    return isInterrupted(false);
}
/**
  * @param ClearInterrupted 是否清除状态标注，false不清除，true清除
  */
private native boolean isInterrupted(boolean ClearInterrupted);
-------------
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0 ;i < 10000;i++){}
        System.out.println(Thread.currentThread().getName());
    }
},"leithda");
t2.start();
t2.interrupt();
System.out.println("是否停止 1 ?= " + t2.isInterrupted());//是否停止 1 ?=true
System.out.println("是否停止 2 ?= " + t2.isInterrupted());//是否停止 2 ?=true
```

### interrupted方法
```java
/**
 *  检测当前线程是否是中断状态，执行后将清除中断状态
 *  当连续两次调用该方法时，第二次会返回false(除非该线程再次被中断)
 */
public static boolean interrupted() {
    //当前线程会调用本地isInterrupted方法，同时不清除状态标志
    //注意比isInterrupted方法多了个currentThread
    return currentThread().isInterrupted(true);
}
-------------
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0 ;i < 10000;i++){}
        System.out.println(Thread.currentThread().getName());
    }
},"leithda");
t2.start();
t2.interrupt();//此时currentThread指的是主线程，主线程并没有被中断(执行了out方法)，因此返回false
System.out.println("是否停止 1 ?= " + t2.interrupted());//是否停止 1 ?=fasle
System.out.println("是否停止 2 ?= " + t2.interrupted());//是否停止 2 ?=fasle
//将t2.interrupt()变成下面的，则主线程被中断
Thread.currentThread().interrupt();
System.out.println("是否停止 1 ?= " + t2.interrupted());//是否停止 1 ?=true
```

## 使用中断
### 设置中断监听
监听原则： 需要在可能发生中断的线程中循环监听线程中断状态，一旦发生中断就会执行中断逻辑，否则继续正常执行
处理方案： 处理中断的方案可参见[6.6 处理中断](#处理中断)
```java
Thread thread = new Thread(() -> {
    //循环监听中断状态
    while(!Thread.currentThread().isInterrupted()) {
        //正常执行任务
    }
    //处理中断
}).start();
```

### 触发中断
中断处理原则： 当调用中断后，会在`下一次`循环判断中退出循环，进而执行中断逻辑
```java
//调用JDK提供的中断方法即可
thread.interrupt();
```

## 中断情况
中断线程分以下三种情况：
1. **中断非阻塞线程:** volatile共享变量或使用`interrupt()`,前者需要自己实现,后者是JDK提供的
2. **中断阻塞线程:** 当处于阻塞状态的线程调用`interrupt()`时会抛出中断异常,并且会清除线程中断标志(设置为`false`);由于中断标志被清除,若像继续中断,需要在捕获中断异常后重新调用`interrupt()`重置中断标志位`true`
3. **不可中断线程:** `synchroinzed`和`aquire()`不可被中断,但AQS提供了`acquireInterruptibly()`方法响应中断

**补充:**中断阻塞线程测试用例
```java
Thread thread = new Thread(() -> {
    while(!Thread.currentThread().isInterrupted()) {
        System.out.println(Thread.currentThread().getName() + " while run ");
        try {
            System.out.println(Thread.currentThread().getName() + " sleep begin");
            Thread.sleep(500);
            System.out.println(Thread.currentThread().getName() + " sleep end");
        } catch (InterruptedException e) {
            //sleep方法会清空中断标志，若不重新中断，线程会继续执行
            Thread.currentThread().interrupt();
            e.printStackTrace();
        }
    }
    if (Thread.currentThread().isInterrupted()) {
        System.out.println(Thread.currentThread().getName() + "is interrupted");
    }
});
thread.start();
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
thread.interrupt();
```

## 处理中断

**注意**: `interrupt()`只是建议中断而不是真正中断,所以需要用以下方法真正中断线程
### 异常法
```java
//直接选择在run方法中抛出异常，从而停止线程
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            throw new InterruptedException();//主动抛出异常
        } catch (InterruptedException e) {
            System.out.println("成功捕获 InterruptedException异常");
            e.printStackTrace();
        }
    }
},"leithda");
t2.start();
t2.interrupt();
-------------
//输出：成功捕获 InterruptedException异常
//打印：java.lang.InterruptedException
```

### 阻塞法
```java
//沉睡
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            //若是wait()、join()、sleep()方法需要获取或者选择向上抛出
            Thread.sleep(1000);//沉睡
        } catch (InterruptedException e) {
            System.out.println("在沉睡中 成功捕获 InterruptedException异常");
            /此时若想中断任务，需要再次触发中断以停止任务
            Thread.currentThread().interrupt();
        }
    }
},"leithda");
t2.start();
t2.interrupt();
-------------
//输出：在沉睡中 成功捕获 InterruptedException异常
//打印：java.lang.InterruptedException: sleep interrupted
```


### Return 方法
```java
//将interrupt方法与return结合使用也能停止线程
Thread t2 = new Thread(new Runnable() {
@Override
public void run() {
    int i = 0;
    while (true){
        if (Thread.currentThread().isInterrupted()){
            System.out.println("停止");
            return;
        }
        System.out.println(System.currentTimeMillis());
    }
}
},"leithda");
t2.start();
Thread.sleep(1000);
t2.interrupt();
-------------
//输出：
.....
1502463392035
1502463392035
1502463392035
停止
//然后就停止打印了
```
### 暴力法
```java
//使用stop()方法停止线程是非常暴力的，不能用！
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        int i = 0;
        while (true){
            System.out.println(i++);
        }
    }
},"leithda");
t2.start();
Thread.sleep(1000);
t2.stop();
-------------
//输出：
.....
154192
154193
154194
154195
//然后就停止打印了
```

## 禁用stop方法
**禁用`stop()`的原因:**
  - **立即抛出异常:** 即刻抛出`ThreadDeath异常`,在线程的`run()`内,任何一点都有可能抛出`ThreadDeath Error`,包括在`cache或finally`语句中
  - **立即释放所有锁:** 释放该线程所持有的所有的锁,会导致数据无法同步处理,出现数据不一致的问题
```java
//情况1：使用stop()方法时会抛出ThreadDeath异常，但一般不需要显示捕获
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.currentThread().stop();
        }catch (ThreadDeath death){
            System.out.println("捕获到ThreadDeath异常");
            death.printStackTrace();
        }
    }
},"leithda");
t2.start();
-------------
//输出：捕获到ThreadDeath异常
//打印：java.lang.ThreadDeath
```
```java
//情况2：使用stop()方法释放锁会给数据造成不一样的问题
final Object lock = new Object();
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName() + "获取锁");
                Thread.sleep(1000);
                System.out.println(Thread.currentThread().getName() + "释放锁");
            }
        } catch (Throwable ex) {
            System.out.println("获取异常： " + ex);
            ex.printStackTrace();
        }
    }
},"leithda");
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName() + "获取锁");
        }
    }
},"mellofly");
t1.start();
Thread.sleep(1000);
//        t1.stop();
t2.start();
-------------
//当t1.stop()未执行时，两个线程竞争锁的顺序是固定的，打印结果如下：
leithda 获取锁
leithda 释放锁
mellofly 获取锁
//当t1.stop()执行时，t1线程抛出了ThreadDeath异常并且t1线程释放了它所占有的锁，打印结果如下：
mellofly
java.lang.ThreadDeath
获取异常： java.lang.ThreadDeath
//由上述可得，stop方法非线性安全，再加上是过期方法，一定不能用！
```

# 线程优先级
## setPriority方法
```java
/**
  * Changes the priority of this thread.
  *     变更线程优先级 默认优先级为NORM_PRIORITY = 5
  */
public final void setPriority(int newPriority) {
    ThreadGroup g;
    checkAccess();
    //优先级区间时MIN_PRIORITY - MAX_PRIORITY
    //注意：不同操作系统的优先级可能有些区别
    if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
        throw new IllegalArgumentException();
    }
    if((g = getThreadGroup()) != null) {
        if (newPriority > g.getMaxPriority()) {
            newPriority = g.getMaxPriority();
        }
        setPriority0(priority = newPriority);
    }
}
```

## 优先级规则
- 继承性
```java
//线程t1启动线程t2，则t2与t1具有相同的优先级
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("当前线程" + Thread.currentThread().getName() + "的优先级为：" + Thread.currentThread().getPriority());
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("当前线程" + Thread.currentThread().getName() + "的优先级为：" + Thread.currentThread().getPriority());
            }
        },"leithda").start();
    }
}, "mellofly");
t1.setPriority(Thread.MAX_PRIORITY);//注意默认优先级是5，这里我们改成10
t1.start();
-------------
//输出：
当前线程leithda的优先级为：10
当前线程mellofly的优先级为：10

```
- 规则性
> - CPU会尽量将执行资源让给优先级比较高的线程
> - 高优先级的线程总是大部分先执行完,但不意味着高优先级的线程会先全部执行完
> - 当线程优先级的等级差距很大时,谁先执行完和代码的调用顺序无关

- 随机性
> - 优先级高的线程不一定每次都先执行完,具有随机性和不确定性
> - 但通常来说,优先级高的线程都会运行的比优先级较低的要快(大概率事件)

# 线程异常处理
## 线程中异常处理
**线程中异常处理方法:**
  - `setDefaultUncaughtExceptionHandler()`:全局默认线程异常处理器,用于给所有线程对象设置默认的异常处理器
  - `setUncaughtException()`: 线程私有异常处理器,给指定线程对象设置的异常处理器,优先级高于全局默认线程异常处理器

### setDefaultUncaughtExceptionHandler方法
```java
Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println(t.getName() + "线程出现异常");
        e.printStackTrace();
    }
});
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        Object object = null;
        System.out.println(object.toString());
    }
},"leithda");
t2.start();
-------------
//输出：
leithda线程出现异常
java.lang.NullPointerException
    at concurrent.Main$1.run(Main.java:12)
    at java.lang.Thread.run(Thread.java:748)
```

### setUncaughtExceptionHandler方法
```java
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        Object object = null;
        System.out.println(object.toString());
    }
},"leithda");
t2.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println(t.getName() + "线程出现异常");
        e.printStackTrace();
    }
});
t2.start();
-------------
//输出：
leithda线程出现异常
java.lang.NullPointerException
    at concurrent.Main$1.run(Main.java:12)
    at java.lang.Thread.run(Thread.java:748)
```

## 线程组异常处理
**线程组异常特点**  
  - 默认情况下,同意个线程组的线程出现异常,该线程组并不会收到影响并被终止,而是继续正常运行
  - 若像实现同一个线程组中一旦有一个线程出现异常要终止该线程组的所有线程时,需要集成`ThreadGroup`同时重写`ThreadGroup`的`uncaughtException()`方法

 - 继承ThreadGroup
```java
/**
  * 继承ThreadGroup自定义一个新的线程组，并重写uncaughtException方法
  * 实现：当线程组内一个线程出现异常，该线程组下的所有线程也会全部停止
  */
public class leithdaThreadGroup extends ThreadGroup {
    public leithdaThreadGroup(String name) {
        super(name);
    }
    //重写线程组的uncaughtException方法
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        super.uncaughtException(t, e);
        this.interrupt();//出现异常，就终止该线程
    }
}
-------------
leithdaThreadGroup leithdaThreadGroup = new leithdaThreadGroup("leithdaGroup");
Thread [] threads = new Thread[5];
for (int i =0;i<5;i++){
    threads[i] = new Thread(leithdaThreadGroup,new Runnable() {
    @Override
    public void run() {
        Thread t = Thread.currentThread();
        if ("leithda4".equals(t.getName())){
            throw new NullPointerException();//随便抛出一个异常
        }
        while (t.isInterrupted() == false){
            System.out.println("线程" + t.getName() + "死循环中");
        }
    }
},"leithda" + i);
threads[i].start();
-------------
//输出：
.....
线程leithda1死循环中
线程leithda1死循环中
线程leithda0死循环中
线程leithda3死循环中
Exception in thread "leithda4" java.lang.NullPointerException
    at concurrent.Main$1.run(Main.java:17)
    at java.lang.Thread.run(Thread.java:748)
//然后所有线程都终止了，结束打印   
```

# 守护线程
## 守护线程
- **分类**:在Java中分成两种线程: **用户线程**和**守护线程**
- **特性**: 当进程中不存在非守护线程时,则全部的守护线程会自动销毁
- **应用**: JVM在启动后会生成一系列守护线程,最有名的当属GC(垃圾回收器)

## 守护线程的使用
```java
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("守护线程运行了");
        for (int i = 0; i < 500000;i++){
            System.out.println("守护线程计数：" + i);
        }
    }
}, "leithda");
t2.setDaemon(true);
t2.start();
Thread.sleep(500);
-------------
//输出：
......
守护线程计数：113755
守护线程计数：113756
守护线程计数：113757
守护线程计数：113758
//结束打印：会发现守护线程并没有打印500000次，因为主线程已经结束运行了
```

# 线程间通信
## 线程间通信
线程与线程之间不是独立的个体,彼此之间可以互相通信和协作:
  **Java提供多种线程间通信方案**: 轮询机制、等待/通知机制、join()、ThreadLocal、Synchronized、Volatile等

## 轮询机制
- sleep+while(true)
```java
final List<Integer> list = new ArrayList<Integer>();
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            for (int i = 0 ; i < 6 ;i++){
                list.add(i);
                System.out.println("添加了" + (i+1) + "个元素");
            }
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
},"leithda");
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            while (true){
                if (list.size() == 3){
                    System.out.println("已添加5个元素,mellofly 线程需要退出");
                    throw new InterruptedException();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();//这里只是打印堆栈信息，不是真的停止执行
        }
    }
},"mellofly");
t1.start();
t2.start();
-------------
//输出：
添加了1个元素
添加了2个元素
添加了3个元素
已添加3个元素,mellofly 线程需要退出
java.lang.InterruptedException
    at concurrent.Main$2.run(Main.java:98)
    at java.lang.Thread.run(Thread.java:748)
添加了4个元素
添加了5个元素
添加了6个元素
```
  - 轮训机制的弊端
    1. 难以保证及时性：睡眠时，基本不消耗资源，但睡眠时间过长(轮询间隔时间较大)，就不能及时发现条件变更
    2. 难以降低开销：减少睡眠时间(轮询间隔时间较小)，能更迅速发现变化，但会消耗更多资源，造成无端浪费
    3. Java引入等待/通知机制来减少CPU的资源浪费，同时还可以实现在多个线程间的通信

## 等待/通知机制
### Object类的wait和notify方法

| 方法名称       | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| notify()       | 通知一个在对象上等待的线程，使其从wait()方法返回，而返回的前提是该线程获取到了对象的锁 |
| notifyAll()    | 通知所有等待在该对象上的线程                                 |
| wait()         | 调用该方法的线程进入WAITING状态，只有等待另外线程的通知或被中断才会返回，需要注意，调用wait()方法后，会释放对象的锁 |
| wait(long)     | 超时等待一段时间，这里的参数时间是毫秒，也就是等待长达n毫秒，如果没有通知就超时返回 |
| wait(long,int) | 对于超时时间更细粒度的控制，可以达到纳秒                     |



### Object.wait方法

- wait方法**使当前线程进行阻塞等待**,该方法是Object类的方法,用来将当前线程**放入"等待队列"**中,并在wait所在的代码处停止执行,直到收到通知**被唤醒或中断或超时**
- **调用wait方法之前,线程必须获得该对象的对象级别锁**,即只能在同步方法或同步块中调用wait方法
- 在执行wait方法后,**当前线程释放锁**,在从wait方法返回前,线程与其他线程竞争重新获得锁
- 如果调用wait方法时没有持有适当的锁,则抛出运行期异常类`IllegalMonitorStateException`

### Object.notify方法

- notify方法**使线程被唤醒**,该方法是Object类的方法,用来将当前线程**从"等待队列中"移出到"同步队列中"**线程状态重新编程**阻塞状态**,notify方法所在同步快释放锁后,从wait方法返回继续执行
- **调用notify方法之前,线程必须获得该对象的对象级别锁**,即只能在同步方法或同步块中调用notify方法
- 该方法用来通知可能等待该对象的对象锁的其他线程,如果有多个线程等待,则由线程规划器从等待队列中随机选择一个WAITTING状态线程,对其发出通知转入同步队列并使它等待获取该对象的对象锁
- 在执行notify方法之后,**当前线程不会马上释放对象所锁**,等待线程也并不能马上获取该对象锁,需要等到执行notify方法的线程将程序执行完,即**退出同步代码块之后当前线程才能释放锁**,而等待线程才可以有机会获取该对象锁
- 如果调用notify方法时没有持有适当的锁,则抛出运行期异常类`IllegalMonitorStateException`

### wait/notify机制

- **wait()使线程停止运行，notify()使停止的线程继续运行
1. 使用`wait()`、`notify()`、`notifyAll()`需要 **先对调用对象加锁**，即只能在同步方法或同步代码块中调用这些方法
2. 使用`wait()`方法后，**线程状态由RUNNING编程WAITING**，并将当前线程放入对象的 **等待队列**中
3. 调用`notify()`或`notifyAll()`方法之后，等待线程不会从`wait()`返回，需要`notify()`方法所在 **同步代码块执行完毕而释放锁**之后，等待线程才可以 **获取到该对象锁并从`wait()`返回**
4. `notify()`方法将 **随机选择一个等待线程**从等待队列移动到同步队列中；`notifyAll()`方法会将等待队列中的 **所有等待线程**全部移到同步队列中，被移动线程状态由`WAITING`变成`BLOCKED`

### 等待/通知的经典范式(生产者-消费者模式)
- 等待方(生产者)遵循如下规则
- 1. 获取对象的锁
- 2. 如果条件不满足，那么调用对象的wait方法，被通知后仍要检查条件
- 3. 条件满足则执行对应处理逻辑

```java
// 对应伪代码
synchronized(对象){
  while(条件不满足){
    对象.wait();
  }
  对应处理逻辑
}
```

- 通知方(消费者)遵循如下规则
- 1. 获取对象的锁
- 2. 改变条件
- 3. 通知所有等待在对象上的线程

```java
// 对应伪代码
synchronized(对象){
  改变条件
  对象.notifyAll();
}
```

### wait/notify的使用
```java
final List<Integer> list = new ArrayList<Integer>();
Object lock = new Object();
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        //加锁，拥有lock的Monitor
        //wait方法必须在synchronized方法或方法块中执行，否则抛出IllegalMonitorStateException
        synchronized (lock){
            System.out.println(Thread.currentThread().getName() + "线程开始执行");
            if (list.size() != 3){
                System.out.println("wait 开始:" + System.currentTimeMillis());
                try {
                    lock.wait();//将当前leithda线程放入等待队列中，进入等待状态
                    System.out.println("wait 结束:" + System.currentTimeMillis());
                    System.out.println(Thread.currentThread().getName() + "线程结束执行");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
},"leithda");
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        //加锁，拥有lock的Monitor
        //notify方法必须在synchronized方法或方法块中执行，否则抛出IllegalMonitorStateException
        //由于wait方法会释放锁，所以mellofly线程可以获取到lock同步锁
        //同时notify方法不会释放锁，直到该同步块执行完毕
        synchronized (lock){
            try {
                System.out.println(Thread.currentThread().getName() + "线程开始执行");
                for (int i = 0; i< 6;i++){
                    list.add(i);
                    if (list.size() == 3){
                        System.out.println("notify 开始:" + System.currentTimeMillis());
                        //随机唤醒一个等待线程，这里因为就只有leithda线程被等待，所有就唤醒它
                        lock.notify();
                        // lock.notifyAll(); 会一次性唤醒所有的等待线程
                        System.out.println("notify方法已执行，发出通知");
                    }
                    System.out.println("已添加" + ( i +1 ) + "个元素");
                    Thread.sleep(500);//为了让效果明显一些，我们先暂停500毫秒
                }
                System.out.println("notify 结束:" + System.currentTimeMillis());
                System.out.println(Thread.currentThread().getName() + "线程结束执行");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
},"mellofly");
t1.start();
t2.start();
-------------
//输出：
leithda线程开始执行
wait 开始:1502699526681
执行wait方法    
mellofly线程开始执行 //注意：wait方法会释放锁，所有mellofly线程获得锁从而执行
已添加1个元素
已添加2个元素
notify 开始:1502699527682
notify已方法，发出通知
已添加3个元素
已添加4个元素
已添加5个元素
已添加6个元素 //注意：notify方法不会释放锁，直到该同步方法/块执行完毕才会释放锁
notify 结束:1502699529682
mellofly线程结束执行
wait 结束:1502699529682
leithda线程结束执行
```

### wait/notify的陷阱
> - **通知过早**： 当notify执行的过早，或者说通知过早，很容易造成逻辑混乱，比如wait失效
> - **等待条件变化**： 当wait等待的条件发生了变化，很容易造成逻辑混乱，比如异常

## join方法
### join方法

- **作用**: 等待线程对象销毁，可以使得一个线程在另一个线程结束后再执行，底层使用`wait()`实现
- **常用于**： 将两个交替执行的线程合并为顺序执行的线程
- **场景**： 在很多情况下，主线程创建并启动子线程，当子线程中药进行大量耗时运算，主线程往往将遭遇子线程结束之前结束。这时，如果主线程想等待子线程执行完成之后再结束，比如子线程处理一个数据，主线程要取得这个数据中的值，就可以使用`join()`方法

### join方法解析

```java
/**
  * Waits at most {@code millis} milliseconds for this thread to
  * die. A timeout of {@code 0} means to wait forever.
  *     后续线程需要等待当前线程至多运行millis毫秒（超过millis当前线程会自动死亡，结束等待）
  *     若millis表示0，表示后续线程需要永远等待（直到当前线程运行完毕）
  * <p> This implementation uses a loop of {@code this.wait} calls conditioned on 
  * {@code this.isAlive}. As a thread terminates the {@code this.notifyAll} method is invoked. 
  * It is recommended that applications not use {@code wait}, {@code notify}, or
  * {@code notifyAll} on {@code Thread} instances.
  *     该方法的原理是循环调用wait方法阻塞后续线程直到当前线程已经不是存活状态了
  * @param  millis
  *         the time to wait in milliseconds
  * @throws  IllegalArgumentException
  *          if the value of {@code millis} is negative
  * @throws  InterruptedException
  *          if any thread has interrupted the current thread. The
  *          <i>interrupted status</i> of the current thread is
  *          cleared when this exception is thrown.
  */
//注意 join方法被synchronized修改，即是个同步方法，也是此处获取到同步锁，为wait做好前提准备
//同时lock指的就是调用join方法的对象
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
    //当millis为0时，说明后续线程需要被无限循环等待，直到当前线程结束运行
    if (millis == 0) {
        while (isAlive()) {
            wait(0);//wait的超时时间为0
        }
    } else {
        //当millis>0时，在millis毫秒内后续线程需要循环等待，直到超时当前线程自动死亡
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);//wait的超时时间为delay
            now = System.currentTimeMillis() - base;
        }
    }
}
/**
  * Thread类还提供一个等待时间为0的join方法
  * 用于将后续线程无限循环等待，直到当前线程结束运行
  */
public final void join() throws InterruptedException {
    join(0);
}
```

### join方法的使用
```java
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0;i<100;i++){
            System.out.println(Thread.currentThread().getName()+"线程值为:leithda" + i);                    }
    }
},"leithda");
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0;i<2;i++){
            System.out.println(Thread.currentThread().getName()+"线程值为:mellofly" + i);
        }
    }
},"mellofly");
t1.start();
t1.join();//让t2线程和后续线程无限等待直到leithda线程执行完毕
t2.start();
-------------
//输出：
......
leithda线程值为:leithda97
leithda线程值为:leithda98
leithda线程值为:leithda99
mellofly线程值为:mellofly0 //可以发现直到leithda线程执行完毕，mellofly线程才开始执行
mellofly线程值为:mellofly1
```

### join(long)方法的使用

```java
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            System.out.println("beginTime=" + System.currentTimeMillis());
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
},"leithda");
t1.start();
t1.join(2000);//当主线程等待2000毫秒
System.out.println("endTime=" + System.currentTimeMillis());
-------------
//输出：
......
beginTime=1502693592670
endTime  =1502693594670
//endTime - beginTime = 2000 正好相当于等待的时间
```

### join VS sleep

- `join(long`方法在内部使用`wait(long`方法进行等待，因此`join(long)`方法能够释放锁
- `Thread.sleep(long)`方法不会释放锁

### join VS synchronized

- `join()`方法在内部使用`wait()`方法进行等待
- `synchronized`使用的是对象监视器原理作为同步

## ThreadLocal

### ThreadLocal
- **变量共享**：变量值的共享可以使用public static变量，所有线程都使用同一个静态变量
- **作用**： ThreadLocal 实现每个线程都有自己的共享变量，从而别的线程修改静态变量并不会对自身产生影响
- **原理**： ThreadLocal相当于全局存放数据的盒子，存储每个线程的私有数据
- **重点**： ThreadLocal 并不能解决并发带来的数据不一致问题，而主要用于备份共享变量作为私有变量
- **补充**：关于 ThreadLocal 请参见笔者的 并发番@ThreadLocal一文通，这里只是简单介绍

### ThreadLocal的使用

```java
ThreadLocal threadLocal = new ThreadLocal();
Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0;i<2;i++){
            threadLocal.set("leithda" + i);
            System.out.println(Thread.currentThread().getName() + 
                "线程的ThreadLocal值为:" + threadLocal.get());
            }
        }
    },"leithda");
Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0;i<2;i++){
            threadLocal.set("mellofly" + i);
            System.out.println(Thread.currentThread().getName() + 
                "线程的ThreadLocal值为:" + threadLocal.get());
            }
        }
    },"mellofly");
t1.start();
t2.start();
for (int i = 0;i<2;i++){
    threadLocal.set("Main" + i);
    System.out.println(Thread.currentThread().getName() + 
        "线程的ThreadLocal值为:" + threadLocal.get());
}
//输出：
main线程的ThreadLocal值为:Main0
mellofly线程的ThreadLocal值为:mellofly0
leithda线程的ThreadLocal值为:leithda0
mellofly线程的ThreadLocal值为:mellofly1
main线程的ThreadLocal值为:Main1
leithda线程的ThreadLocal值为:leithda1
```

# 致谢
[并发番@Thread一文通(1.7版)](https://www.zybuluo.com/kiraSally/note/823674) 由 [黄志鹏kira](https://juejin.im/user/59716ee96fb9a06b9c744c67) 创作，采用 [知识共享 署名-非商业性使用 4.0 国际 许可协议](http://creativecommons.org/licenses/by-nc/4.0/) 进行许可。

本站文章除注明转载/出处外，均为本站原创或翻译，转载前请务必署名。
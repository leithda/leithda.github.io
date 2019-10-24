---
title: 高并发之线程池
categories:
  - Java
  - Core
tags:
  - 多线程
  - 并发
author: 长歌
abbrlink: 2990170790
date: 2019-10-24 00:00:00
---

构造一个线程池为什么需要几个参数？如果避免线程池出现OOM？Runnable和Callable的区别是什么？

<!-- More -->

## 线程
### 概念
- 线程和进程的主要差别在于它们是不同的操作系统资源管理方式。进程在执行过程中拥有独立的内存单元，而多个线程共享内存。线程有自己的堆栈和局部变量，但线程没有单独的地址空间
- 线程池

### 线程的创建与管理
#### 继承 Thread
- 通过继承`Thread`类来创建一个线程
```java
public class OneThread extends Thread {
    private Thread thread;
    private String threadName;

    OneThread(String threadName) {
        this.threadName = threadName;
    }

    public void run() {
        System.out.println("执行线程 " + threadName);
        try {
            for (int i = 4; i > 0; i--) {
                System.out.println("线程: " + threadName + ", " + i);
                // 让线程睡眠一会
                Thread.sleep(50);
            }
        } catch (InterruptedException e) {
            System.out.println("线程 " + threadName + " interrupted.");
        }
        System.out.println("线程 " + threadName + " exiting.");
    }

    public void start() {
        System.out.println("创建线程 " + threadName);
        if (thread == null) {
            thread = new Thread(this, threadName);
            thread.start();
        }
    }
}
```

#### 实现 Runnable
- `Runnable`是一个接口，实现该接口并重写`#run()`方法，`new Thread(runnable).start()`，线程启动时就会自动调用该对象的`#run()`方法
```java
public class TwoThread implements Runnable {
    private Thread thread;
    private String threadName;

    public TwoThread(String threadName) {
        this.threadName = threadName;
        System.out.println("创建线程" + threadName);
    }


    @Override
    public void run() {
        System.out.println("执行线程 " + threadName);
        try {
            for (int i = 4; i > 0; i--) {
                System.out.println("线程: " + threadName + ", " + i);
                // 让线程睡眠一会
                Thread.sleep(50);
            }
        } catch (InterruptedException e) {
            System.out.println("线程 " + threadName + " interrupted.");
        }
        System.out.println("线程 " + threadName + " exiting.");
    }

    public void start() {
        System.out.println("开启线程" + threadName);
        if (thread == null) {
            thread = new Thread(this, threadName);
            thread.start();
        }
    }
}
```

#### 使用 FutureTask
- `Callable`是个接口，实现`#call()`方法，使用`FutureTask`类来包装`Callable`对象，使用`FutureTask`对象作为`Thread`对象的`target`创建并启动新线程；也可以使用线程池启动
```java
public class ThirdThread implements Callable<Integer> {
    private String threadName;
    private Thread thread;
    private FutureTask<Integer> futureTask;

    public ThirdThread(String threadName) {
        this.threadName = threadName;
    }

    public void start() {
        futureTask = new FutureTask<>(this);
        for (int i = 0; i < 100; i++) {

            System.out.println(Thread.currentThread().getName() + " 的循环变量i的值" + i);
            if (i == 20) {
                if (thread == null) {
                    System.out.println("创建线程 " + threadName);
                    thread = new Thread(futureTask, threadName);
                    thread.start();
                }
            }
        }
    }

    public Integer get() {
        try {
            int result = futureTask.get();
            System.out.println("当前线程" + threadName + "的返回值: " + result);
            return result;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        return -1;
    }

    @Override
    public Integer call() throws Exception {
        System.out.println("执行线程 " + threadName);
        int i = 0;
        for (; i < 20; i++) {
            System.out.println(Thread.currentThread().getName() + " " + i);
        }
        return i;
    }
}
```
- `Runnable`和`Callable`的区别
    1. `Callable`规定的方法是`#call()`,`Runnable`规定的方法是`#run()`
    2. `Callable`的任务执行后可返回值，而`Runnable`的任务不能返回值
    3. `#call()`方法可以抛出异常，而`#run()`方法不可以
    4. 运行`Callable`任务可以拿到一个`Future`对象，`Future`表示异步计算的结果，它提供了检查计算是否完成的接口。计算完成后调用`#get()`方法获取结果，如果线程没有执行完，`#get()`方法会阻塞当前线程的执行；如果线程出现异常，`#Future.get()`会抛出异常。如果线程已经取消，会抛出`CancletationException`。由`#cancle()`方法取消线程执行。`#isDone()`确定任务是正常完成还是取消了。计算完成后不允许取消。如果为了取消执行而使用`Fuure`而又不提供可用的结果，则可以声明`Future<?>`形式类型，并返回`null`作为底层任务的结果。

### 线程状态
{% asset_img JavaThreadState.png Java 线程状态 %}
线程共包含以下五种状态：
1. **新建状态(New)**: 线程对象被创建后，就进入了新建状态。例如，Thread thread = new Thread()。
2. **就绪状态(Runnable)**: 也被称为“可执行状态”。线程对象被创建后，其它线程调用了该对象的start()方法，从而来启动该线程。例如，thread.start()。处于就绪状态的线程，随时可能被CPU调度执行。
3. **运行状态(Running)**: 线程获取CPU权限进行执行。需要注意的是，线程只能从就绪状态进入到运行状态。
4. **阻塞状态(Blocked)**: 阻塞状态是线程因为某种原因放弃CPU使用权，暂时停止运行。直到线程进入就绪状态，才有机会转到运行状态。阻塞的情况分三种：
    - 等待阻塞 -- 通过调用线程的wait()方法，让线程等待某工作的完成。
    - 同步阻塞 -- 线程在获取synchronized同步锁失败(因为锁被其它线程所占用)，它会进入同步阻塞状态。
    - 其他阻塞 -- 通过调用线程的sleep()或join()或发出了I/O请求时，线程会进入到阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入就绪状态。
5. **死亡状态(Dead)**: 线程执行完了或者因异常退出了run()方法，该线程结束生命周期。

## 线程池
### 使用线程池的好处
- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的损耗。
- 提高响应速度。当任务到达时，不需要创建线程就能立即执行
- 提高线程的可管理性。使用线程池可以进行统一的分配、调优和监控

### 线程池的创建与管理
> 线程池：`Executors`类提供了方便的工厂方法来创建不同类型的`executor services`。无论是`Runnable`接口的实现类还是`Callable`接口的实现类，都可以被`ThreadPoolExecutor`或者`ScheduledThreadPoolExecutor`执行
1. public static ExecutorService newCachedThreadPool() 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若没有可回收线程，则新建线程。之前创建线程可用时会重用。对于执行很多短期异步程序的程序而言，这些线程池通常可以提高性能。
2. public static ExecutorService newFixedThreadPool(int nThreads) 创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程，超出的线程会在队列中等待。
3. public static ExecutorService newSingleThreadExecutor() 创建一个使用单个 worker 线程的 Executor,以无界队列方式来运行该线程，它只会用唯一的工作线程来执行任务，保证所有的任务按照顺序(FIFO,LIFO,优先级)执行
4. public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) 创建一个定长线程池，支持定时及周期性任务执行。
5. newWorkStealingPool 它是新的线程池类`ForkJoinPool`的扩展，由于能够合理的使用CPU进行对任务操作(并行操作)，所以很适合使用在很耗时的任务中。

#### ForkJoinPool
- `ForkJoinPool`的每个工作线程都维护着一个工作队列`WorkQueue`,这是一个双端队列，里面存放的对象是任务(`ForkJoinTask`)
1. 每个工作线程在运行中产生的新的任务(通常是因为调用`#fork()`)时，会放入工作队列的队尾，工作线程处理时采用`LIFO`方式，即：每次从队尾取出任务执行。
2. 每个工作线程执行自己的任务时，会尝试窃取一个任务(刚刚提交或者其他工作线程的任务)，窃取的任务位于其他线程的工作队列队首，窃取使用FIFO模式
3. 在遇到`#join()`时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。
4. 在既没有自己的任务，也没有可以窃取的任务时，进入休眠

#### ExecutorCompletionService 
- 内部管理着一个已完成任务的阻塞队列，如果阻塞队列中有已完成的任务，`#take()`方法就返回任务的结果，否则阻塞等待任务完成，基于`FutureTask`实现。

## 线程池原理
### ThreadPoolExecutor
#### corePoolSize
- 核心线程数量，当有新任务在`#execute()`方法提交时，会执行以下判断:
1.  如果运行的线程少于`corePoolSize`，创建新线程来处理任务，即使线程池中有其他的空闲线程
2.  如果运行的线程数量大于等于`corePoolSize`且小于`maximumPoolSize`,则只有当`workQueue`满时才创建新的线程去处理任务
3.  如果设置的`corePoolSize`和`maximumPoolSize`相等，则创建的线程池的大小是固定的，这时如果有新任务提交，若`workQueue`未满，则将请求放入`workQueue`中，等待有空闲的线程从`workQueue`中取任务并执行
4.  如果运行的线程数量大于等于`maximumPoolSize`，这时如果`workQueue`已经满了，则通过`handler`制定的策略来处理任务。
5.  所以，任务提交时，判断的顺序为: `corePoolSize` --> `workQueue` --> `maximumPoolSize`

#### maximumPoolSize
- 最大线程数量

#### workQueue
- 等待队列， 当任务提交时，如果线程池中的线程数量大于等于`corePoolSize`时，把该任务封装成一个`Worker`对象放入队列等待
- 保存等待执行的任务的阻塞队列，当提交一个新的任务到线程池后，线程池会根据当前线程池中正在执行的线程的数量来决定对该任务的处理方式，主要有以下几种处理方式：
1. 直接切换: 这种方式常用的队列是`SynchronousQueue`，TODO 后续介绍
2. 使用无界队列：一般使用基于链表的阻塞队列`LinkedBlockingQueue`。如果使用这种方式，那么线程池中能够创建的最大线程数就是`corePoolSize`，而`maximumPoolSize`就不会起作用，当线程池中的核心线程都是 RUNNING 状态时，新的任务被提交会放入等待队列中。
3. 使用有界队列：一般使用`ArrayBlockingQueue`。使用该方式可以讲线程池的最大线程数量限制为`maximumPoolSize`，这样能够降低资源的消耗，但这种方式使得线程池对线程的调度变得更困难，因为线程池和队列的容量都是有限的值，所以想要使线程池处理任务的吞吐率达到一个相对合理的范围，又想使线程调度相对简单，并且还要尽可能的降低线程池对资源的消耗，就需要合理的设置这两个数量。
4. keepAliveTime：线程池维护线程所允许的空闲时间。当线程池中的线程数量大于`corePoolSize`的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是等待直到时间超过了`keepAliveTime`;
5. threadFactory：它是`ThreadFactory`类型的变量，用来创建新线程。默认使用`Executors.defaultThreadFactory()`来创建线程。使用默认的`ThreadFactory`来创建线程时，会使新创建的线程具有相同的`NORM_PRIORITY`优先级并且是非守护线程，同时也设置了线程的名称。
6. handler：它是`RejectedExecutionHandler`类型的变量，表示线程池的饱和策略。如果阻塞队列满了且没有空闲的线程，这时如果继续提交任务，就需要采取一种策略处理该任务。线程池提供了4中策略：
    1. AbortPolicy：直接抛出异常，默认策略;
    2. CallerRunsPolicy：用调用者所在的线程来执行任务
    3. DiscardOldestPollicy：丢弃阻塞队列中最靠前的任务，并执行当前任务；
    4. DiscardPolicy：直接丢弃任务；

### 核心源码
#### 线程池执行源码
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /**
         * clt 记录着 runState 和 workCount
         */
        int c = ctl.get();
        /**
         * workerCountOf 方法取出低29位的值，表示当前活动的线程数
         */
        if (workerCountOf(c) < corePoolSize) {
            /**
             * addWorker方法的主要工作是在线程池中创建一个新的线程并执行
             * firstTask参数 用于指定新增的线程执行的第一个任务，
             * core参数为
             *     ture: 新增线程检查当前活动线程数是否少于corePoolSize
             *     false:新增线程检查当前活动线程数收少于maximumPoolSize
             */
            if (addWorker(command, true))
                return;
            /**
             * 如果添加失败，则重新获取 ctl 值
             */
            c = ctl.get();
        }
        /**
         * 如果当前线程池是运行状态并且任务添加到队列成功
         */
        if (isRunning(c) && workQueue.offer(command)) {
            // 重新获取 ctl 值
            int recheck = ctl.get();
            /**
             * 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了
             * ，这时需要移除该command。执行过后通过handler使用拒绝策略对该任务执行处理，整个方法返回
             */
            if (! isRunning(recheck) && remove(command))
                reject(command);
            /**
             * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
             * 这里传入的参数表示：
             * 1. 第一个参数为 null ,表示在线程池中创建一个线程，但不去启动
             * 2. 第二个参数为 false ,将线程池的有限线程数量的上限设置为 maximumPoolSize,添加线程时根据maximumPoolSize 判断。
             * 如果判断 workCount 大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行
             */
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        /**
         * 如果执行到这里，有两种情况：
         * 1. 线程池已经不是RUNNING状态；
         * 2. 线程池时RUNNIING状态，但workerCount >= corePoolSize 并且 workQueue已满。
         * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize
         * 如果失败则拒绝该任务
         */
        else if (!addWorker(command, false))
            reject(command);
    }
```

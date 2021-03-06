---
title: Java-JUC-工具类
abbrlink: 3943993470
date: 2020-03-23 12:40:16
categories:
  - Java
tags:
  - JUC
  - 并发
  - 多线程
author: 长歌
---

{% cq %}
从java1.5开始，jdk提供了java.util.concurrent包，这个包中包含多个供并发使用的工具类
{% endcq %}
<!-- More -->

## CountDownLatch（闭锁）
`CountDownLatch`又叫闭锁，可以让一个线程等待其他一组线程执行结束后再执行。

### 常用方法
- public CountDownLatch(int count) 其中`count`是计数个数，表示要等待的线程个数
- public void await()   在需要阻塞的线程中执行这个方法，表示将当前线程阻塞至计数器减为0
- public void countDown() 执行一次这个方法会将计数器减1，当计数器为0表示所有线程执行完了。**注意：同一线程调用多次会导致计数器持续递减**

### 使用

```java
// 统计多个线程的访问时间
public class TestCountDownLatch {
    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(10);    // 10个线程
        CountDownLatchDemo ld = new CountDownLatchDemo(latch);
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10; i++) {
            new Thread(ld).start();
        }

        try{
            latch.await();
        }catch (Exception e){}

        long end = System.currentTimeMillis();
        System.out.println("耗时 : " + (end - start) + "ms");
    }
}

class CountDownLatchDemo implements Runnable{

    private CountDownLatch latch;

    public CountDownLatchDemo(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        synchronized (this){
            try{
                for (int i = 0; i < 50000; i++) {
                    if(i%2 == 0){
                        System.out.println(i);
                    }
                }
            }finally {
                latch.countDown();	// 计数器减一
            }
        }
    }
}
```



## CyclicBarrier（循环栅栏）

`CyclicBarrier`又叫循环栅栏，允许一组线程互相等待，直到某个到达某个公共屏障前。它看上去和`CountDownLatch`一样，但是`CountDownLatch`无法被重置。

### 常用方法

- public int await()	阻塞当前线程，让当前线程等待其他线程
- public CyclicBarrier(int parties)  构造方法，其中`parties`表示等待的线程个数
- public CyclicBarrier(int parties, Runnable barrierAction) 传入一个Runnable对象，当所有线程到达栅栏处时，在等待线程中挑选一个线程执行Runnable任务，然后所有线程同时往下执行

### 使用

```java
public class TestCyclicBarrier {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3); // 3个工作线程阻塞后才开始执行

        for (int i = 0; i < 10; i++) {
            System.out.println("创建工作线程" + i);
            new Thread(new CyclicBarrierDemo(cyclicBarrier)).start();
        }
    }
}
class CyclicBarrierDemo implements Runnable{

    private CyclicBarrier cyclicBarrier;

    public CyclicBarrierDemo(CyclicBarrier cyclicBarrier) {
        this.cyclicBarrier = cyclicBarrier;
    }

    @Override
    public void run() {
        try{
            System.out.println(Thread.currentThread().getName() + "开始等待其他线程");
            cyclicBarrier.await();
            System.out.println(Thread.currentThread().getName() + "开始执行");
            // 工作线程开始处理，这里用Thread.sleep()来模拟业务处理
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName() + "执行完毕");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```



## Exchanger

主要用于两个线程间交换数据，调用exchange等待线程匹配成功，然后交换数据并一起执行。一个线程执行exchange会一直阻塞，多个线程使用同一个交换器时会随机两个线程进行配对。



```java
public class TestExchanger {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();
        new Thread(new ExchangerDemo(exchanger, "我是线程A"),"线程A").start();
        try{
            Thread.sleep(200);
        }catch (Exception e){
            e.printStackTrace();
        }
        new Thread(new ExchangerDemo(exchanger, "我是线程B"),"线程B").start();
    }
}

class ExchangerDemo implements Runnable{
    private Exchanger<String> exchanger;
    private String info;

    public ExchangerDemo(Exchanger<String> exchanger, String info) {
        this.exchanger = exchanger;
        this.info = info;
    }

    @Override
    public void run() {
        try{
            System.out.println(Thread.currentThread().getName() + "等待交换消息");
            String msg = exchanger.exchange(info);  // 阻塞等待,对方交换数据后才往下进行数据交换
            System.out.println(Thread.currentThread().getName() + "收到消息 : "+ msg);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

- Exchanger必须成对出现，否则会导致有的线程一直等待，可以通过`exchange(V x, long timeout, TimeUnit unit)`设置最大等待时长



## Semphore

信号量，资源访问控制，维护了一个许可数量，每次线程想要执行使用`require`进行许可申请，如果没有则阻塞，每次使用执行`release`进行许可释放。

```java
public class TestSemphore {
    public static void main(String[] args) throws Exception {
        // 现在允许操作的资源一共有2两个
        Semaphore sem = new Semaphore(2);
        // 模拟每一个用户办理业务的时间
        for (int i = 0; i < 10; i++) {
            new Thread(new SemphoreDemo(sem),"顾客-"+i).start();
        }
    }
}

class SemphoreDemo implements Runnable {

    Semaphore sem;

    public SemphoreDemo(Semaphore sem) {
        this.sem = sem;
    }

    @Override
    public void run() {
        Random rand = new Random();
        // 现在有空余窗口
        if (sem.availablePermits() > 0) {
            System.out.println("【" + Thread.currentThread().getName() + "】进入银行，此时银行没有人排队");
        } else {   // 没有空余位置
            System.out.println("【" + Thread.currentThread()
                    .getName() + "】排队等待办理业务。");
        }
        try {
            // 从信号量中获得操作许可
            sem.acquire();
            System.out.println("【" + Thread.currentThread().getName() + "】｛start｝开始办理业务。");
            // 模拟办公延迟
            TimeUnit.SECONDS.sleep(rand.nextInt(10));
            System.out.println("【" + Thread.currentThread().getName() + "】｛end｝结束办理业务。");
            // 当前线程离开办公窗口
            sem.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```




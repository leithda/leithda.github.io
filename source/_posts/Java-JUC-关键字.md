---
title: Java-JUC-关键字
abbrlink: 4169854470
date: 2020-01-06 18:17:13
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
java提供的并发关键字主要如下:`volatile`,`synchronized`和`final`
{% endcq %}
<!-- More -->

# volatile
> volatile变量，用来确保将变量的可见性。当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。
>
> 在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。
>
>  
>
> 推荐阅读 [并发番@Java内存模型&Volatile一文通（1.7版）]([https://www.zybuluo.com/kiraSally/note/850631#%E5%B9%B6%E5%8F%91%E7%95%AAjava%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bvolatile%E4%B8%80%E6%96%87%E9%80%9A17%E7%89%88](https://www.zybuluo.com/kiraSally/note/850631#并发番java内存模型volatile一文通17版))

## volatile的特性
1. 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。（实现可见性）
2. 对于单个volatile变量的读或者写具有原子性.
    如`i++`,其变量的操作为read;inc.write.变量的写操作依赖与当前值,所以不能保证线程安全
3. 禁止进行指令重排序。（实现有序性）

## volatile的实现原理
### volatile的可见性实现原理
> volatile的可见性通过内存屏障（Memory Barrier）实现.

- 内存屏障，又称内存栅栏，是一个 CPU 指令。
- 在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM 为了保证在不同的编译器和 CPU 上有相同的结果，通过插入特定类型的内存屏障来禁止特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。


### volatile 有序性实现原理
#### volatile 的 happens-before 关系
> happens-before 规则中有一条是 **volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。**

```java
//假设线程A执行writer方法，线程B执行reader方法
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    
    public void writer() {
        a = 1;              // 1 线程A修改共享变量
        flag = true;        // 2 线程A写volatile变量
    } 
    
    public void reader() {
        if (flag) {         // 3 线程B读同一个volatile变量
        int i = a;          // 4 线程B读共享变量
        //……
        }
    }
}
```
- 根据 [happens-before](../detail/2396729586.html#happens-before规则) 规则，上面过程会建立 3 类 happens-before 关系。
    + 根据程序次序规则：1 happens-before 2 且 3 happens-before 4。
    + 根据 volatile 规则：2 happens-before 3。
    + 根据 happens-before 的传递性规则：1 happens-before 4。
{% asset_img volatile_1.png volatile有序性 %}
- 因为以上规则，当线程 A 将 volatile 变量 flag 更改为 true 后，线程 B 能够迅速感知。

#### volatile 禁止重排序
- 为了性能优化，JMM 在不改变正确语义的前提下，会允许编译器和处理器对指令序列进行重排序。JMM 提供了内存屏障阻止这种重排序。
- Java 编译器会在生成指令系列时在适当的位置会插入内存屏障指令来禁止特定类型的处理器重排序。
- JMM 会针对编译器制定 volatile 重排序规则表。

{% asset_img volatile重排序规则.png vloatile重排序规则 %}

- " NO " 表示禁止重排序。
- 为了实现 volatile 内存语义时，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。
- 对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎是不可能的，为此，JMM 采取了保守的策略。
    - 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。
    - 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。
    - 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
    - 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。
- volatile 写是在前面和后面分别插入内存屏障，而 volatile 读操作是在后面插入两个内存屏障。

| 内存屏障        | 说明                                                        |
| -------------- | ----------------------------------------------------------- |
| StoreStore 屏障 | 禁止上面的普通写和下面的 volatile 写重排序。                |
| StoreLoad 屏障  | 防止上面的 volatile 写与下面可能有的 volatile 读/写重排序。 |
| LoadLoad 屏障   | 禁止下面所有的普通读操作和上面的 volatile 读重排序。        |
| LoadStore 屏障  | 禁止下面所有的普通写操作和上面的 volatile 读重排序。        |

{% asset_img volatile写插入内存屏障.png volatile写插入内存屏障 %}

{% asset_img volatile读插入内存屏障.png volatile读插入内存屏障 %}



## volatile的应用场景

### 使用volatile应具备的条件

- 对变量的写操作不依赖于当前值。
- 该变量没有包含在具有其他变量的不变式中。
- 只有在状态真正独立于程序内其他内容时才能使用 volatile。

### 模式#1 状态

- 也许实现 volatile 变量的规范使用仅仅是使用一个布尔状态标志，用于指示发生了一个重要的一次性事件，例如完成初始化或请求停机。

```java
volatile boolean isShutdown;	// 停机
......
public void shutdown() { isShutdown = true; }
public void doWork() { 
    while (!isShutdown) { 
        // do stuff
    }
}
```

### 模式#2 一次性安全发布(one-time safe publication)

- 在缺乏同步的情况下，可能会遇到某个对象引用的更新值（由另一个线程写入）和该对象状态的旧值同时存在。（这就是造成著名的双重检查锁定（double-checked-locking）问题的根源）

```java
//基于volatile的解决方案
public class SafeDoubleCheckSingleton {
    //通过volatile声明，实现线程安全的延迟初始化
    private volatile static SafeDoubleCheckSingleton singleton;
    private SafeDoubleCheckSingleton(){
    }
    public static SafeDoubleCheckSingleton getInstance(){
        if (singleton == null){
            synchronized (SafeDoubleCheckSingleton.class){
                if (singleton == null){
                    //原理利用volatile在于 禁止 "初始化对象"(2) 和 "设置singleton指向内存空间"(3) 的重排序
                    singleton = new SafeDoubleCheckSingleton();
                }
            }
        }
        return singleton;
    }
}
```

- 未设置为`volatile`可能导致执行顺序如下:

  ```bash
  memory = allocate();	// 1.分配对象的内存空间
  instance = memory;		// 3. 设置instance指向刚分配的空间
  ctorInstance(memory);	// 2. 初始化对象
  ```

### 模式#3 独立观察(independent observation)

- 安全使用 volatile 的另一种简单模式是定期 发布 观察结果供程序内部使用。例如，假设有一种环境传感器能够感觉环境温度。一个后台线程可能会每隔几秒读取一次该传感器，并更新包含当前文档的 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值

```java
public class UserManager {
    public volatile String lastUser;
 
    public boolean authenticate(String user, String password) {
        boolean valid = passwordIsValid(user, password);
        if (valid) {
            User u = new User();
            activeUsers.add(u);
            lastUser = user;
        }
        return valid;
    }
}
```

### 模式 #4开销较低的读－写锁策略

- 当读远多于写，结合使用内部锁和 volatile 变量来减少同步的开销
  利用volatile保证读取操作的可见性；利用synchronized保证复合操作的原子性

```java
public class Counter {
    private volatile int value;
    //利用volatile保证读取操作的可见性, 读取时无需加锁
    public int getValue() { return value; }
    // 使用 synchronized 加锁
    public synchronized int increment() { 
        return value++;
    }
}
```

# synchronized
> Java的保留关键字,是Java同步机制的一种实现,即互斥锁机制.
> 
> 推荐阅读 [并发番@Synchronized一文通（1.8版）](https://www.zybuluo.com/kiraSally/note/857726)

## synchronized 的使用

| 作用域       | 锁对象            | 同步原理                                          | Demo                                                         |
| ------------ | ----------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| 普通同步方法 | 当前实例          | 依靠方法修饰符上的ACC_SYNCHRONIZED                | synchronized void func(){}                                   |
| 静态同步方法 | 当前类的class对象 | 依靠方法修饰符上的ACC_SYNCHRONIZED                | static synchronized void func(){}                            |
| 同步代码块   | 括号内对象        | 同步代码块使用了`monitorenter`和`monitorexit`实现 | Object o = new Object();<br>void func(){<br>  synchronized(o/this){}<br>} |

- **补充：** 使用同步代码块的好处在于其他线程仍可以访问非synchronized(this)的同步代码块

## synchronized 的使用规则
### 测试类
```java
public class SyncDemo {
    public synchronized static void staticMethod(){
        System.out.println(Thread.currentThread().getName() + "访问了静态方法 staticMethod");
        try{
            Thread.sleep(1000);
        }catch (Exception ignored){}
        System.out.println(Thread.currentThread().getName() + "结束访问静态方法 staticMethod");
    }

    public synchronized static void staticMethod2(){
        System.out.println(Thread.currentThread().getName() + "访问了静态方法 staticMethod2");
        try{
            Thread.sleep(1000);
        }catch (Exception ignored){}
        System.out.println(Thread.currentThread().getName() + "结束访问静态方法 staticMethod2");
    }

    private synchronized void syncmethod(){
        System.out.println(Thread.currentThread().getName() + "访问了方法 syncmethod");
        try{
            Thread.sleep(1000);
        }catch (Exception ignored){}
        System.out.println(Thread.currentThread().getName() + "结束访问方法 syncmethod");
    }

    private synchronized void syncmethod2(){
        System.out.println(Thread.currentThread().getName() + "访问了方法 syncmethod2");
        try{
            Thread.sleep(1000);
        }catch (Exception ignored){}
        System.out.println(Thread.currentThread().getName() + "结束访问方法 syncmethod2");
    }

    private void method(){
        System.out.println(Thread.currentThread().getName() + "访问了方法 method");
        try{
            Thread.sleep(1000);
        }catch (Exception ignored){}
        System.out.println(Thread.currentThread().getName() + "结束访问方法 method");
    }

    private final Object lock = new Object();
    private void chunkMethod(){
        System.out.println(Thread.currentThread().getName() + "访问了chunkMethod方法");
        synchronized (lock){
            System.out.println(Thread.currentThread().getName() + "在chunkMethod方法中获取了lock");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(Thread.currentThread().getName() + "结束访问 chunkMethod方法");
    }
    private void chunkMethod2(){
        System.out.println(Thread.currentThread().getName() + "访问了chunkMethod2方法");
        synchronized (lock){
            System.out.println(Thread.currentThread().getName() + "在chunkMethod2方法中获取了lock");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(Thread.currentThread().getName() + "结束访问chunkMethod2 方法");
    }
    public void chunkMethod3(){
        System.out.println(Thread.currentThread().getName() + "访问了chunkMethod3方法");
        //同步代码块
        synchronized (this){
            System.out.println(Thread.currentThread().getName() + "在chunkMethod3方法中获取了this");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(Thread.currentThread().getName() + "结束访问 chunkMethod3方法");
    }
    public void stringMethod(String lock){
        synchronized (lock){
            while (true) {
                System.out.println(Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### 同步方法与普通方法
#### 代码
```java
    /**
     * 普通方法和同步方法
     */
    private static void methodAndsyncMethod(){
        SyncDemo sd = new SyncDemo();
        new Thread(sd::method).start();
        new Thread(sd::syncmethod).start();
    }
```

#### 结果
```terminal
Thread-0访问了方法 method
Thread-1访问了方法 syncmethod
Thread-0结束访问方法 method
Thread-1结束访问方法 syncmethod
```
- 当一个线程进入同步方法时，其他线程可以正常访问其他非同步方法

### 同步方法
#### 代码
```java
    /**
     * 同一个锁(相同实例)的同步方法
     */
    private static void syncMethodWithOneLock() {
        SyncDemo sd = new SyncDemo();
        new Thread(() -> {
            sd.syncmethod();
            try {
                Thread.sleep(1000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            sd.syncmethod2();
        }).start();

        new Thread(() -> {
            sd.syncmethod();
            sd.syncmethod2();
        }).start();
    }
```

#### 结果
```bash
Thread-0访问了方法 syncmethod
Thread-0结束访问方法 syncmethod
// 线程0释放锁
Thread-1访问了方法 syncmethod
Thread-1结束访问方法 syncmethod
Thread-1访问了方法 syncmethod2
Thread-1结束访问方法 syncmethod2
// 线程1释放锁
Thread-0访问了方法 syncmethod2
Thread-0结束访问方法 syncmethod2
```
- 任务阻塞,线程需要等待线程0执行完成后才继续执行

### 同步代码块
#### 代码
```java
    /**
     * 同一个锁下的同步代码块同一时刻只能被一个线程访问
     */
    private static void chunkMethodWithOneLock(){
        SyncDemo sd = new SyncDemo();
        new Thread(()->{
            sd.chunkMethod();
            sd.chunkMethod2();
        }).start();

        new Thread(()->{
            sd.chunkMethod2();
            sd.chunkMethod();
        }).start();
    }
```

#### 结果
```java
Thread-0访问了chunkMethod方法
Thread-0在chunkMethod方法中获取了lock
Thread-1访问了chunkMethod2方法
// 线程1等待
Thread-0结束访问 chunkMethod方法
Thread-0访问了chunkMethod2方法
Thread-0在chunkMethod2方法中获取了lock
Thread-0结束访问chunkMethod2 方法
Thread-1在chunkMethod2方法中获取了lock
Thread-1结束访问chunkMethod2 方法
Thread-1访问了chunkMethod方法
Thread-1在chunkMethod方法中获取了lock
Thread-1结束访问 chunkMethod方法
```


# final
> 在JMM中,`final`能确保初始化过程的安全性,共享final对象时无须同步

## final的使用场景
final 可以用于修饰变量,方法和类

- final 修饰成员变量
1. 类变量
2. 实例变量
3. 引用变量
4. 局部变量

- final 修饰成员方法
该方法不可被子类复写,但是可以重载

- final 修饰类
该类不可以被继承,

## final 代码
```java
public class FinalDemo {
//public final class FinalDemo {    // 3 final 修饰的类不能被继承

    static final int classVar;  // 1.1 final 修饰类变量

    static{
        classVar = 10;
    }

    final int instanceVar;  // 1.2 final 修饰成员变量

    public FinalDemo() {
//        classVar = 10;    // 错误: 类实例化时,类变量classVar对应内存空间已分配,JVM发现类变量没初始化会报错
        this.instanceVar = 20;
    }


    public void print(){
        final int innerVar; // 1.3 final 修饰局部变量

        // 其他处理(不包含对 innerVar 的初始化)

        innerVar = 30;
        System.out.println(innerVar);
    }


    /**
     * final 修饰的方法不可以被子类复写
     */
    public final void func(){
        System.out.println("finao void func()");
    }

    public void testReference(){
        final FinalObject object = new FinalObject();
        object.number = 10; // 1.4 final 修饰引用变量时,其引用(指向地址)不可变,引用对象内容可变
    }
}

class FinalExtenderDemo extends FinalDemo{


    // 可重写普通方法
    @Override
    public void print() {
        super.print();
        func(); // 可以调用 final 方法
    }
//  重写 final 方法报错
//    public void func(){
//
//    }

    // 可以重载 final 方法
    public void func(String content){
        System.out.println(content);
    }
}

class FinalObject {
    public int number;
}
```

## 并发中的final
### final 的重排序规则
#### 基础数据类型
- 写final域重排序规则
> 写final域的重排序规则：禁止对final域的写，重排序到构造函数之外

    1. JMM禁止编译器把final域的写，重排序到构造函数之外；

    2. 编译器会在final域写之后，构造函数return之前，插入一个storestore屏障（关于内存屏障可以看上篇文章）。这个屏障可以禁止处理器把final域的写，重排序到构造函数之外。

- 读 final 域重排序规则
> 读final域重排序规则为：在一个线程中，初次读对象的引用，和初次读该对象包含的final域，JMM会禁止这两个操作的重排序。（注意，这个规则仅仅是针对处理器），处理器会在读final域操作的前面插入一个LoadLoad屏障.

#### 引用类型
> 禁止在构造函数对一个final修饰的对象的成员域的写入与随后将这个被构造的对象的引用赋值给引用变量 重排序

### final引用溢出问题
```java
class FinalReferenceEscapeDemo {
    private final int a;
    private FinalReferenceEscapeDemo referenceDemo;

    public FinalReferenceEscapeDemo() {
        a= 1;  //1
        referenceDemo= this; //2
    }

    public void writer() {
        new FinalReferenceEscapeDemo();
    }

    public void reader() {
        if (referenceDemo!= null){  //3
            int temp=referenceDemo.a; //4
        }
    }
}
```
- 1和2之间不存在依赖性,可以重排序,先执行2,此时读取这个对象会出现问题


# Reflence
{% blockquote 童云兰 https://book.douban.com/subject/10484692 豆瓣 %}
Java并发编程实战(原作名: Java Concurrency in Practice)
{% endblockquote %}

---
{% blockquote 羽杰 https://www.jianshu.com/p/ccfe24b63d87 简书 %}
【Java 并发笔记】volatile 相关整理
{% endblockquote %}

---

{% blockquote @kiraSally https://www.zybuluo.com/kiraSally/note/850631 www.zybuluo.com %}
并发番@Java内存模型&Volatile一文通（1.7版）
{% endblockquote %}
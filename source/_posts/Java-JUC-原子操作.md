---
title: Java-JUC-原子操作
abbrlink: 1710138375
date: 2020-01-06 13:33:16
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
从java1.5开始，jdk提供了java.util.concurrent.atomic包，这个包中的原子操作类，提供了一种用法简单，性能高效，线程安全的更新一个变量的方式。
{% endcq %}
<!-- More -->

## 概述
atomic包里面一共提供了13个类，分为4种类型，分别是：原子更新基本类型，原子更新数组，原子更新引用，原子更新属性，这13个类都是使用Unsafe实现的包装类。



## 原子更新基本类型
> 原子更新基本类型分别是: AtomicInteger,AtomicLong,AtomicBoolean.下面测试以AtomicInteger为例:

### 测试代码
```java
    /**
     * 测试 AtomInteger
     */
    public static void testAtomInteger() {
        CountDownLatch latch = new CountDownLatch(100);
        CountDownLatch latchAtomic = new CountDownLatch(100);
        NormalNumber nn = new NormalNumber(latch);
        AtomicNumber an = new AtomicNumber(latchAtomic);
        for (int i = 0; i < 100; i++) {
            new Thread(nn, "线程" + i).start();
            new Thread(an, "线程" + i).start();
        }
        try {
            latch.await();
            latchAtomic.await();
        } catch (Exception e) {
        }
        System.out.println("普通变量结果是:" + nn.getNumber());
        System.out.println("原子变量结果是:" + an.getNumber());
    }

/**
 * 普通变量 int
 */
class NormalNumber implements Runnable {

    private int number = 0;
    CountDownLatch latch;

    public NormalNumber(CountDownLatch latch) {
        this.latch = latch;
    }

    public int getNumber() {
        return number;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            number++;
        }
        latch.countDown();
    }
}

/**
 * 原子操作变量 AtomicInteger
 */
class AtomicNumber implements Runnable {

    private AtomicInteger number = new AtomicInteger(01);
    CountDownLatch latch;

    public AtomicNumber(CountDownLatch latch) {
        this.latch = latch;
    }

    public int getNumber() {
        return number.get();
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            number.incrementAndGet();
        }
        latch.countDown();
    }
}
```

### 结果
```bash
普通变量结果是:9984
原子变量结果是:10000
```

## 原子数组
### 测试代码
```java
    private static void testAtomicIntegerArray() {
        int[] values = new int[10];
        CountDownLatch latch = new CountDownLatch(100);
        CountDownLatch latchAtomic = new CountDownLatch(100);
        NormalArray na = new NormalArray(values, latch);
        AtomicArray aa = new AtomicArray(new AtomicIntegerArray(values), latchAtomic);
        for (int i = 0; i < 100; i++) {
            new Thread(na, "普通线程" + i).start();
            new Thread(aa, "原子线程" + i).start();
        }
        try {
            latch.await();
            latchAtomic.await();
        } catch (Exception ignored) {
        }
        aa.print();
        na.print();
    }
class NormalArray implements Runnable {

    private int[] values;
    private CountDownLatch latch;

    NormalArray(int[] values, CountDownLatch latch) {
        this.latch = latch;
        this.values = values;
    }

    @Override
    public void run() {
        synchronized (this) {
            for (int i = 0; i < 100; i++) {
                for (int j = 0; j < 10; j++) {
                    values[j]++;
                }
            }
            latch.countDown();
        }
    }

    void print() {
        System.out.print("普通数组:");
        for (int value : values) {
            System.out.print(value + " ");
        }
    }
}

class AtomicArray implements Runnable {

    private AtomicIntegerArray values;
    private CountDownLatch latch;

    AtomicArray(AtomicIntegerArray value, CountDownLatch latch) {
        this.latch = latch;
        this.values = value;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            for (int j = 0; j < 10; j++) {
                values.incrementAndGet(j);
            }
        }
        latch.countDown();
    }

    void print() {
        System.out.print("原子数组:");
        for (int i = 0; i < 10; i++) {
            System.out.print(values.get(i) + " ");
        }
    }
}
```

### 结果
```bash
不知道是程序原因还是倒霉,没运行出想要的结果,后续补充...
```

## 原子引用
### 测试代码
```java
    private static void testAtomicReference() {
        Book book = new Book(1, "测试书");
        AtomicReference<Book> bookAtomicReference = new AtomicReference<>(book);
        Book updateBook = new Book(10, "更新书");
        bookAtomicReference.compareAndSet(book, updateBook);
        System.out.println(bookAtomicReference.get().getId());
        System.out.println(bookAtomicReference.get().getName());
    }

class Book {
    volatile int id;
    private String name;

    Book(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

### 结果
```bash
10
更新书
```

## 原子属性更新工具
### 测试代码
```java
    private static void testAtomicIntegerFieldUpdater() {
        Class<Book> cls = Book.class;

        AtomicIntegerFieldUpdater<Book> mAtoInt = AtomicIntegerFieldUpdater.newUpdater(cls, "id");
        Book book = new Book(1000, "测试书");

        System.out.println("初始值=" + mAtoInt.get(book));

        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.getAndIncrement(book) ->返回值 " + mAtoInt.getAndIncrement(book) + " -> 结果值" + mAtoInt.get(book));// 自增 获取旧值
        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.incrementAndGet(book) ->返回值 " + mAtoInt.incrementAndGet(book) + " -> 结果值" + mAtoInt.get(book));// 自增 获取新值

        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.decrementAndGet(book) ->返回值 " + mAtoInt.decrementAndGet(book) + " -> 结果值" + mAtoInt.get(book));// 自减 获取新值
        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.getAndDecrement(book) ->返回值 " + mAtoInt.getAndDecrement(book) + " -> 结果值" + mAtoInt.get(book));// 自减 获取旧值

        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.addAndGet(book, 10) ->返回值 " + mAtoInt.addAndGet(book, 10) + " -> 结果值" + mAtoInt.get(book));
        System.out.println("原值" + mAtoInt.get(book) + " + mAtoInt.getAndAdd(book, 10) ->返回值 " + mAtoInt.getAndAdd(book, 10) + " -> 结果值" + mAtoInt.get(book));

    }
```

### 结果
```bash
初始值=1000
原值1000 + mAtoInt.getAndIncrement(book) ->返回值 1000 -> 结果值1001
原值1001 + mAtoInt.incrementAndGet(book) ->返回值 1002 -> 结果值1002
原值1002 + mAtoInt.decrementAndGet(book) ->返回值 1001 -> 结果值1001
原值1001 + mAtoInt.getAndDecrement(book) ->返回值 1001 -> 结果值1000
原值1000 + mAtoInt.addAndGet(book, 10) ->返回值 1010 -> 结果值1010
原值1010 + mAtoInt.getAndAdd(book, 10) ->返回值 1010 -> 结果值1020
```




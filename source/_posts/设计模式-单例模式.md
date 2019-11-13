---
title: 单例模式
categories:
  - 设计模式
tags:
  - 设计模式
author: 长歌
date: 2019-9-12
abbrlink: 4049607742
---

单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。
<!-- More -->
## 单例模式实现方式
### 懒汉式
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:09
 * Description: 单例模式1，懒汉式，线程不安全，Lazy加载
 */
public class Singleton {
    private static Singleton instance;

    // 设置构造函数为私有，防止类被实例化
    private Singleton(){}
    /* 可以通过下面代码避免反射多次实例化 单例对象
    private Singleton(){
        if(instance != null){
            throw new IllegalArgumentException("单例构造器不能重复使用");
        }
    }
    */

    public static Singleton getInstance(){
        if(instance == null){
            instance = new Singleton();
        }
        return instance;
    }

    public void test(){
        System.out.println(this);
    }

}
```

### 懒汉式2
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:13
 * Description: 单例模式2，懒汉式，线程安全，Lazy加载，性能低
 */
public class Singleton2 {
    private static Singleton2 instance;

    private Singleton2() {
    }

    // 加锁 synchronized  保证单例，影响效率
    public static synchronized Singleton2 getInstance() {
        if (instance == null) {
            instance = new Singleton2();
        }
        return instance;
    }

    public void test(){
        System.out.println(this);
    }
}
```

### 饿汉式
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:16
 * Description: 单例模式3，饿汉式，线程安全，无Lazy加载，浪费内存
 */
public class Singleton3 {
    // 类装载时对象就会实例化，浪费内存
    private static Singleton3 instance = new Singleton3();

    private Singleton3(){}

    public static Singleton3 getInstance(){
        return instance;
    }

    public void test(){
        System.out.println(this);
    }
}
```

### 双重锁验锁
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:18
 * Description: 单例模式4，双重锁验锁（DCL，即 double-checked locking）,线程安全，Lazy加载，性能较高，实现复杂
 */
public class Singleton4 {
    private volatile static Singleton4 instance;

    private Singleton4(){}

    public static Singleton4 getInstance(){
        // 当对象没有加载时，拿锁进行对象实例化保证单例。对象加载时，直接返回，性能较好
        if(instance == null){
            synchronized (Singleton4.class){
                if(instance == null){
                    instance = new Singleton4();
                }
            }
        }
        return instance;
    }
    public void test(){
        System.out.println(this);
    }
}

```

### 内部类
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:21
 * Description: 单例模式5，内部类实现，线程安全，Lazy加载
 */
public class Singleton5 {
    // 优化了Singleton3，使用静态内部类实现了Lazy加载
    private static class SingleHolder{
        private static final Singleton5 INSTANCE  = new Singleton5();
    }
    private Singleton5(){}

    public static Singleton5 getInstance(){
        return SingleHolder.INSTANCE;
    }
    public void test(){
        System.out.println(this);
    }
}
```

### 枚举方式
```java
package cn.leithda.snail.dp.singlePattern;

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 2019/2/18
 * Time: 16:23
 * Description: 单例模式6
 */
public enum Singleton6 {
    // 这种方式可以防止通过 reflection attact 调用类的私有构造方法
    INSTANCE;
    public void test(){
        System.out.println(this);
    }
}

```
---
title: 适配器模式
categories:
  - 设计模式
tags:
  - 设计模式
author: 长歌
abbrlink: 4123342314
date: 2019-10-11 00:00:00
---

将一个接口转换成客户希望的另一个接口，使接口不兼容的那些类可以一起工作，其别名为包装器(Wrapper)。适配器模式既可以作为类结构型模式，也可以作为对象结构型模式
<!-- More -->

适配器模式
===
## 使用场景
1. 系统需要使用一些现有的类，而这些类的接口（如方法名）不符合系统的需要，甚至没有这些类的源代码。
2. 想创建一个可以重复使用的类，用于与一些彼此之间没有太大关联的一些类，包括一些可能在将来引进的类一起工作。


## 优点
1. 可以让任何两个没有关联的类一起运行。 
2. 提高了类的复用。 
3. 增加了类的透明度。 
4. 灵活性好。

## 缺点
1. 过多的使用适配器，会让系统很凌乱，明明调用的这个接口的方法，里面的实现却是另一个方法
2. java 最多继承一个类，所以至多只能适配一个适配者类，而且目标类必须是抽象类。

## 实现

### 类适配器

#### 被适配类

```java
public class Adaptee {
    public void adapteeRequest() {
        System.out.println("被适配者的方法");
    }
}
```

#### 目标接口
```java
public interface Target {
    void request();
}

```

#### 适配器类

```java
public class Adapter extends Adaptee implements Target {
    @Override
    public void request() {
        // ... 一些操作

        super.adapteeRequest();

        // ... 一些操作
    }
}
```

### 对象适配器

> 和类适配器的区别是一个使用继承，一个使用组合方式

```java
public class Adapter2 implements Target {
    Adaptee adaptee = new Adaptee();
    @Override
    public void request() {
        // ... 一些操作

        adaptee.adapteeRequest();
        // ... 一些操作
    }
}
```

## 示例
- 再来一个好理解的例子，我们国家的民用电都是 220V，日本是 110V，而我们的手机充电一般需要 5V，这时候要充电，就需要一个电压适配器，将 220V 或者 100V 的输入电压变换为 5V 输出

### 电压接口

```java
public interface AC {
    /**
     * 输出电压
     * @return 电压值
     */
    int outputAC();
}
```

### 电压实例
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-10
 * Description: 110伏电压
 */
public class AC110 implements AC {

    @Override
    public int outputAC() {
        int ac = 110;
        return ac;
    }
}

/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-10
 * Description: 220伏电压
 */
public class AC220 implements AC {

    @Override
    public int outputAC() {
        int ac = 220;
        return ac;
    }
}
```

### 适配器接口
```java
/**
 * Created with IntelliJ IDEA.
 * User: 长歌
 * Date: 19-10-10
 * Description: 适配器接口
 */
public interface DC5Adapter {

    /**
     * 是否可以适配
     * @param ac 电压
     * @return 是否
     */
    boolean support(AC ac);

    /**
     * 适配处理
     * @param ac 输入电压
     * @return 适配后电压
     */
    int outputDC5V(AC ac);
}
```

### 适配器实例
```java
public class ChinaPowerAdapter implements DC5Adapter {

    public static final int voltage = 220;

    @Override
    public boolean support(AC ac) {
        return (voltage == ac.outputAC());
    }

    @Override
    public int outputDC5V(AC ac) {
        int adapterInput = ac.outputAC();
        //变压器...
        int adapterOutput = adapterInput / 44;
        System.out.println("使用ChinaPowerAdapter变压适配器，输入AC:" + adapterInput + "V" + "，输出DC:" + adapterOutput + "V");
        return adapterOutput;
    }
}

public class JapanPowerAdapter implements DC5Adapter {
    public static final int voltage = 110;

    @Override
    public boolean support(AC ac) {
        return (voltage == ac.outputAC());
    }

    @Override
    public int outputDC5V(AC ac) {
        int adapterInput = ac.outputAC();
        //变压器...
        int adapterOutput = adapterInput / 22;
        System.out.println("使用JapanPowerAdapter变压适配器，输入AC:" + adapterInput + "V" + "，输出DC:" + adapterOutput + "V");
        return adapterOutput;
    }
}
```

### 手机充电器类
```java
public class Charger {

    private List<DC5Adapter> adapters = new LinkedList<>();

    public Charger() {
        this.adapters.add(new ChinaPowerAdapter());
        this.adapters.add(new JapanPowerAdapter());
    }

    /**
     * 转换电压
     * @param ac 输入电压
     * @return 输出电压
     */
    public int convertAC(AC ac) {
        DC5Adapter adapter = null;
        for (DC5Adapter ad : this.adapters) {
            if (ad.support(ac)) {
                adapter = ad;
                break;
            }
        }
        if (adapter == null){
            throw new  IllegalArgumentException("没有找到合适的变压适配器");
        }
        return adapter.outputDC5V(ac);
    }
}
```

## 测试如下
```java
public class AdapterTest {

    /**
     * 输出如下：
     *     被适配者的方法
     */
    @Test
    public void testBasic1() throws Exception{
        Adapter adapter = new Adapter();

        adapter.request();
    }


    /**
     * 输出如下：
     *     被适配者的方法
     */
    @Test
    public void testBasic2()throws Exception{
        Adapter2 adapter = new Adapter2();

        adapter.request();

    }

    /**
     * 输出如下：
     *     使用ChinaPowerAdapter变压适配器，输入AC:220V，输出DC:5V
     *     220V电压 充电 转换后的电压为:5
     *     使用JapanPowerAdapter变压适配器，输入AC:110V，输出DC:5V
     *     110V电压 充电 转换后的电压为:5
     */
    @Test
    public void testCharger() throws Exception{

        AC chinaAC = new AC220();
        Charger charger = new Charger();
        System.out.println("220V电压 充电 转换后的电压为:"+charger.convertAC(chinaAC));


        // 去日本旅游，电压是 110V
        AC japanAC = new AC110();
        System.out.println("110V电压 充电 转换后的电压为:"+charger.convertAC(japanAC));
    }
}
```

- 对应结构如图:

{% asset_img adapter.png 适配器模式 UML图 %}

## 个人理解
> 想复用现有接口或者方法签名，但是需要增加额外功能时使用适配器模式
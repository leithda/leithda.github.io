---
title: Java-面向对象
abbrlink: 2742829166
date: 2019-12-19 20:32:20
categories:
  - 理论
  - Java
tags:
  - 面向对象思想
author: 长歌
---

{% cq %}
Java是一门面向对象的编程语言
{% endcq %}
<!-- More -->

## 三大特性

### 封装

> 封装是指隐藏对象的属性和实现细节，仅对外提供公共访问方式。封装最主要的功能是我们可以修改自己的实现代码，而不用去修改调用我们代码的程序.

- 封装的好处
  1. 提高了数据的安全性
  2. 减少耦合
  3. 可以对成员变量进行更精确的控制
  4. 隐藏实现，程序实现过程对调用者不可见
- 实现封装

```java
public class Person {

    private String name;
    private int gender;
    private int age;

    public String getName() {
        return name;
    }

    public String getGender() {
        return gender == 0 ? "man" : "woman";
    }

    public void work() {
        if (18 <= age && age <= 50) {
            System.out.println(name + " is working very hard!");
        } else {
            System.out.println(name + " can't work any more!");
        }
    }
}
```

- 封装通过关键字`private`实现，可以有属性的封装、方法的封装

### 继承

>  继承实现了 **IS-A** 关系，例如 Cat 和 Animal 就是一种 IS-A 关系，因此 Cat 可以继承自 Animal，从而获得 Animal 非 private 的属性和方法。   

 继承应该遵循里氏替换原则，子类对象必须能够替换掉所有父类对象。   

 Cat 可以当做 Animal 来使用，也就是说可以使用 Animal 引用 Cat 对象。父类引用指向子类对象称为 **向上转型** 。 

```java
Animal animal = new Cat();
```



### 多态

> 多态主要分为编译时多态和运行时多态:
>
> - 编译时多态主要指方法的重载
> - 运行时多态指程序中定义的对象引用所指向的具体类型在运行期间才确定

运行时多态有三个条件:

- 继承
- 覆盖(重写)
- 向上转型

```java
public class Animal{
    public void call(){
        System.out.println("Animal's call");
    }
}
public class Dog extends Animal{
    public void call(){
        System.out.println("汪..汪.!!!");
    } 
}

public class Cat extends Animal{
    public void call(){
        System.out.println("喵.喵喵..!!!");
    }
}
public class Zoo{
    public static void main(Strin[] args){
        Animal dog = new Dog();
        Animal cat = new Cat();
        dog.call();
        cat.call();
    }
}
```

```tex
汪..汪.!!!
喵.喵喵..!!!
```





## 五大设计原则
| 简写 | 全拼                                | 中文翻译     |
| ---- | ----------------------------------- | ------------ |
| SRP  | The Single Responsibility Principle | 单一责任原则 |
| OCP  | The Open Closed Principle           | 开放封闭原则 |
| LSP  | The Liskov Substitution Principle   | 里氏替换原则 |
| ISP  | The Interface Segregation Principle | 接口分离原则 |
| DIP  | The Dependency Inversion Principle  | 依赖倒置原则 |

### 单一责任原则

> 修改一个类的原因应该只有一个

一个类只负责一件事，一个类承担的职责过多，就等于把这些职责耦合在了一起

### 开放封闭原则

>  类应该对扩展开放，对修改关闭。 

对类进行功能扩展时，不需要修改原代码。最符合开放封闭原则模式的是[装饰者模式]()

### 里氏替换原则

> 子类对象必须能够替换掉所有父类对象

子类必须可以当成父类来使用，保证继承体系的统一性

### 接口分离原则

> 不应该强迫客户依赖于它们不用的方法。

每个接口专用，而不是将所有功能聚合到一个接口里

### 依赖倒置原则

> 高层模块不应该依赖于低层模块，二者都应该依赖于抽象
>
> 抽象不应该依赖于细节，细节应该依赖于抽象

高层模块包含一个应用程序中重要的策略选择和业务模块，如果高层模块依赖于低层模块，低层模块的改动就会直接影响到高层模块。从而使高层模块也需要改动   

依赖于抽象意味着:

-  任何变量都不应该持有一个指向具体类的指针或者引用； 
-  任何类都不应该从具体类派生； 
-  任何方法都不应该覆写它的任何基类中的已经实现的方法。 

## 其他原则
| 简写 | 全拼                              | 中文翻译     |
| ---- | --------------------------------- | ------------ |
| LOD  | The Law of Demeter                | 迪米特法则   |
| CRP  | The Composite Reuse Principle     | 合成复用原则 |
| CCP  | The Common Closure Principle      | 共同封闭原则 |
| SAP  | The Stable Abstractions Principle | 稳定抽象原则 |
| SDP  | The Stable Dependencies Principle | 稳定依赖原则 |

### 迪米特法则

 迪米特法则又叫作最少知识原则（Least Knowledge Principle，简写 LKP），就是说一个对象应当对其他对象有尽可能少的了解，不和陌生人说话。 

### 合成复用原则

 尽量使用对象组合，而不是通过继承来达到复用的目的。 

### 共同封闭原则

 一起修改的类，应该组合在一起（同一个包里）。如果必须修改应用程序里的代码，我们希望所有的修改都发生在一个包里（修改关闭），而不是遍布在很多包里。 

### 稳定抽象原则

 最稳定的包应该是最抽象的包，不稳定的包应该是具体的包，即包的抽象程度跟它的稳定性成正比。 

### 稳定依赖原则

 包之间的依赖关系都应该是稳定方向依赖的，包要依赖的包要比自己更具有稳定性。 
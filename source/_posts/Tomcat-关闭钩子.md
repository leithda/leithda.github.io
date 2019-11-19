---
title: Tomcat-关闭钩子
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
abbrlink: 966063575
date: 2019-11-19 20:00:00
---

> 为避免应用程序非自然关闭导致清理代码未能正常执行，Java提供了一种优雅的方式供使用。
在Java中，虚拟机遇到两种事件会关闭虚拟机：
- 应用程序正常退出如`System.exit`或者最后一个非守护退出
- 用户强制终止虚拟机，例如键入`Ctrl`+`C`或者在关闭Java程序之前从系统中注销。
<!-- More -->

## 虚拟机优雅关闭
- 当虚拟机关闭时，虚拟机有以下两个步骤:
    1. 虚拟机启动所有注册的关闭钩子。关闭钩子是在Runtime上注册的线程。所有关闭钩子被同时执行，直到完成.
    2. 虚拟机调用所有未被调用的`finalizers`


## 创建关闭钩子
1. 写一个类继承`Thread`类
2. 编写类的`#run()`方法。该方法为应用程序关闭时要调用的代码，无论正常退出还是异常退出
3. 在应用程序中初始化一个钩子
4. 在当前的`Runtime`上使用`#addShutdownHook`方法来注册该关闭钩子

## 编写关闭钩子案例
```java
public class ShutdownHookDemo {
    public void start(){
        System.out.println("Demo");
        ShutdownHook shutdownHook = new ShutdownHook();
        Runtime.getRuntime().addShutdownHook(shutdownHook);
    }

    public static void main(String[] args) {
        ShutdownHookDemo demo = new ShutdownHookDemo();
        demo.start();
        try{
            System.in.read();
        }catch(Exception e){

        }

    }

    class ShutdownHook extends Thread{
        @Override
        public void run() {
            super.run();
            System.out.println("Shutdown Down");
        }
    }
}
```
- 用户键入任意字符后，控制台会打印`Shutdown Down`

## Tomcat中的关闭钩子
```java
    // Catalina.java

    protected class CatalinaShutdownHook extends Thread {
        protected CatalinaShutdownHook() {
        }

        public void run() {
            if (Catalina.this.server != null) {
                try {
                    ((Lifecycle)Catalina.this.server).stop();
                } catch (LifecycleException var2) {
                    System.out.println("Catalina.stop: " + var2);
                    var2.printStackTrace(System.out);
                    if (var2.getThrowable() != null) {
                        System.out.println("----- Root Cause -----");
                        var2.getThrowable().printStackTrace(System.out);
                    }
                }
            }

        }
    }
```
- 关闭钩子在应用启动时，加入到`Runtime`中



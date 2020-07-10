---
title: JVM之入门及类文件结构
abbrlink: 1204858154
date: 2020-07-10 10:35:38
categories:
  - Java
  - JVM
tags:
  - JVM
author: 长歌
---

> JVM是Java Virtual Machine（Java虚拟机）的缩写，JVM是一种用于计算设备的规范，它是一个虚构出来的计算机，是通过在实际的计算机上仿真模拟各种计算机功能来实现的。
> 引入Java语言虚拟机后，Java语言在不同平台上运行时不需要重新编译。Java语言使用Java虚拟机屏蔽了与具体平台相关的信息，使得Java语言编译程序只需生成在Java虚拟机上运行的目标代码（字节码），就可以在多种平台上不加修改地运行。

<!-- More -->

# 1.1 java从编码到执行



{% asset_img java从编码到执行 java从编码到执行.png %}


> JIT即时编译器： 执行次数多的代码会被即时编译器编译，下次不需要字节码解释器一句一句解释执行，可以直接交给OS执行



# 1.2 常见的JVM实现



## Hotspot



- oracle官方，我们做实验用的JVM
- java -version



## Jrockit



- BEA,曾经号称世界上最快的JVM
- 被Oracle收购，合并于hotspot



## TaobaoVM



- hotspot 深度定制版



## LiquidVM



- 直接针对硬件



## azul zing



- 最新垃圾回收的业界标杆
- www.azul.com



## J9 - IBM



## Microsoft VM



# 1.3 JDK&JRE&JVM
{% asset_img JDK&JRE&JVM JDK&JRE&JVM.png %}


# 1.4 Class 文件



文件格式如下:
{% asset_img Class格式 Class文件格式.png %}




### 编写测试类如下:



```
public class Code_1_4_ByteCode {
}
```



### 编译后字节码文件可以通过如下方式查看:



- javap
- JBE 可以直接修改
- jClassLib - IDEA插件



#### javap


```
javap -v Code_1_4_ByteCode.class


#================================================================================#
Classfile /D:/work/Github/jvm/target/classes/cn/leithda/jvm/class01/Code_1_4_ByteCode.class
  Last modified 2020-5-27; size 322 bytes
  MD5 checksum f7a96335a5a359d43a1c590b45b7f512
  Compiled from "Code_1_4_ByteCode.java"
public class cn.leithda.jvm.class01.Code_1_4_ByteCode
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#13         // java/lang/Object."<init>":()V
   #2 = Class              #14            // cn/leithda/jvm/class01/Code_1_4_ByteCode
   #3 = Class              #15            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               Lcn/leithda/jvm/class01/Code_1_4_ByteCode;
  #11 = Utf8               SourceFile
  #12 = Utf8               Code_1_4_ByteCode.java
  #13 = NameAndType        #4:#5          // "<init>":()V
  #14 = Utf8               cn/leithda/jvm/class01/Code_1_4_ByteCode
  #15 = Utf8               java/lang/Object
{
  public cn.leithda.jvm.class01.Code_1_4_ByteCode();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/leithda/jvm/class01/Code_1_4_ByteCode;
}
SourceFile: "Code_1_4_ByteCode.java"
```



#### jClassLib
##### 安装 jClassLib
在IDEA 中安装jClassLib，图中为安装好之后，安装在marketplace中查找插件即可。
{% asset_img IDEA中安装jClassLib jClassLib安装.png %}

##### 使用jClassLib
> 构建项目后，点击view->show ByteCode with JClassLib

{% asset_img jClassLib示例 jClassLib.png %}

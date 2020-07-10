---
title: JVM之类文件加载过程
abbrlink: 581636265
date: 2020-07-10 16:19:09
tags:
---

> Class文件从硬盘或网络加载到虚拟机经历了什么？Java 使用什么样的机制去加载Class文件？
> 
<!-- More -->

# ![类加载器.png](https://cdn.nlark.com/yuque/0/2020/png/1625785/1592913985853-bce4edfd-e68a-43fa-b0b9-51793d9a7f2a.png?x-oss-process=image%2Fresize%2Cw_1500)

# 2.1 类加载器

1. Bootstrap 加载lib/rt.jar charset.jar等核心类，C++实现
2. Extension 加载扩展jar包 jre/lib/ext/*.jar.或由`-Djava.ext.dirs`指定
3. App  加载classpath指定内容
4. Custom ClassLoader自定义类加载器

```
public class Code_2_ClassLoaderLevel {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());  // null
        System.out.println(sun.awt.datatransfer.ClipboardTransferable.class.getClassLoader());   // null
        System.out.println(sun.net.spi.nameservice.dns.DNSNameService.class.getClassLoader());   // sun.misc.Launcher$ExtClassLoader@677327b6
        System.out.println(Code_2_ClassLoaderLevel.class.getClassLoader()); // sun.misc.Launcher$AppClassLoader@18b4aac2

        System.out.println(sun.net.spi.nameservice.dns.DNSNameService.class.getClassLoader().getClass().getClassLoader()); // null
        System.out.println(Code_2_ClassLoaderLevel.class.getClassLoader().getClass().getClassLoader()); // null
    }
}
```

- `null`表示类通过`BootStrapClassLoader`加载，BootStrap通过C++实现，没有Class与它对应

## 2.1.1双亲委派模型

> 由子到父进行检查，由父到子进行实际查找和加载
>
> 主要是为了安全，保证Java的核心类库由JDK自己决定。
>
> 举例：自己编写String类型，赋值时通过网络采集String的值，会导致安全问题。采用双亲委派模型会先向上进行检索，BootStrap加载了java.lang.String会直接返回。



## 2.1.2类加载器范围

> `sun.misc.Launcher` 源码中体现

```
public class Code_2_1_2_ClassLoaderScope {
    public static void main(String[] args) {
        String pathBoot = System.getProperty("sun.boot.class.path");
        System.out.println(pathBoot.replaceAll(";", System.lineSeparator()));

        System.out.println("---------------");
        String pathExt = System.getProperty("java.ext.dirs");
        System.out.println(pathExt.replaceAll(";", System.lineSeparator()));

        System.out.println("---------------");
        String pathApp = System.getProperty("java.class.path");
        System.out.println(pathApp.replaceAll(";", System.lineSeparator()));
    }
}
```



## 2.1.3 自定义类加载器

> ClassLoader源码
>
> findCache -> parent.loadClass -> findClass()
>
> 自定义类加载器使用场景：
>
> 1. Tomcat加载自己定义的类，通过自定义加载器完成
> 2. Spring 实现了自己的ClassLoader
> 3. 热部署
> 4. 定义加密的类，对class文件进行加密，加载时实现解密逻辑，然后进行加载

自定义类加载器需要实现的方法`findClass` ，使用**模板模式**。相关源码`ClassLoader` 。最终调用defineClass将二进制流转换为`Class`对象

```
public class Code_2_1_3_MyClassLoader extends ClassLoader{

    private String filePath;

    public Code_2_1_3_MyClassLoader(String filePath) {
        this.filePath = filePath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        File file = new File(filePath, name.replace(".", "/").concat(".class"));
        try {
            FileInputStream fis = new FileInputStream(file);
            ByteArrayOutputStream byteArrayOS = new ByteArrayOutputStream();
            int b = 0;

            while ((b=fis.read()) !=0) {
                byteArrayOS.write(b);
            }

            byte[] bytes = byteArrayOS.toByteArray();
            byteArrayOS.close();
            fis.close();//可以写的更加严谨

            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.findClass(name); //throws ClassNotFoundException
    }

    public void hello(){
        System.out.println("hello JVM");
    }

    public static void main(String[] args) throws Exception {
        Code_2_1_3_MyClassLoader cl = new Code_2_1_3_MyClassLoader(System.getProperty("user.dir"));
        Class<?> clazz = cl.loadClass("cn.leithda.jvm.class02.Code_2_1_3_MyClassLoader");
        Class<?> clazz1 = cl.loadClass("cn.leithda.jvm.class02.Code_2_1_3_MyClassLoader");
        System.out.println("类是否相等 : "+(clazz == clazz1));   // true
//        Code_2_1_3_MyClassLoader clInstance = (Code_2_1_3_MyClassLoader) clazz.newInstance(); // 类加载器没有无参构造方法，不能直接实例化
        Constructor<?> constructor = clazz.getConstructor(String.class);
        Code_2_1_3_MyClassLoader clInstance = (Code_2_1_3_MyClassLoader) constructor.newInstance("abc");
        clInstance.hello(); // hello JVM
    }
}
```

- 这个例子不严谨，在自定义加载器加载`Code_2_1_3_MyClassLoader`前，AppClassLoader已经加载了。只作演示，不要纠结。

## 2.1.4 实现loadClass破坏双亲委派模型



1. JDK1.2之前，自定义ClassLoader都必须重写loadClass()
2. ThreadContextClassLoader可以实现基础类调用实现类代码，通过thread.setContextClassLoader指定
3. 热启动，热部署

1. 1. osgi tomcat 都有自己的模块指定classloader（可以加载同一类库的不同版本）

## 2.1.5 类的懒加载

> JVM规范没有规定类何时加载，但是规定了什么时候必须初始化
>
> - new getstatic putstatic invokestatic指令，访问final变量除外
> - java.lang.reflect对类进行反射调用
> - 初始化子类的时候，父类必须初始化
> - 虚拟机启动时，被执行的主类必须初始化
> - 动态语言支持java.lang.invoke.MethodHandle解析的结果为REF_getstatuc REF_putstatic REF_invokestatic  的方法句柄时，该类必须初始化

```
public class Code_2_1_5_LasyLoading {
    public static void main(String[] args) throws Exception {
        //P p;
        //X x = new X();
        //System.out.println(P.i);
        //System.out.println(P.j);
        Class.forName("cn.leithda.jvm.class02.Code_2_1_5_LasyLoading$P");

    }

    public static class P {
        final static int i = 8;
        static int j = 9;
        static {
            System.out.println("P");
        }
    }

    public static class X extends P {
        static {
            System.out.println("X");
        }
    }

}
```



# 2.2 加载过程

> 类加载时，将class二进制文件加载到内存，同时生成Class的一个对象指向内存。后续使用该Class时通过**Class对象地址**使用`Class对象`

## 2.2.1 Loading(加载)

> 把一个class文件加载到内存
>
> - 此阶段可以通过自定义类加载器去完成字节流的自定义方式获取
> - 通过使用不同的类加载器，可以从不同来源处加载Class的二进制字节流，通常有以下几种情况
>
> - - 从本地文件系统加载
>   - 从一个ZIP、JAR、CAB或者其他某种归档文件中提取Java Class文件。比如JDBC的驱动包就在JAR中。
>   - 通过网络加载Class文件，最典型的应用就是 Applet
>   - 把一个Java源文件动态编译、并执行加载
>   - 运行时动态生成，使用场景最多的就是动态代理技术， 在 java.lang.reflect.Proxy中 , 就是用了 ProxyGenerator.generateProxyClass来为特定接口生成形式为“*$Proxy”的代理类的二进制字节流 

1. 通过类的"全限定名"来获取定义此类的二进制字节流
2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
3. 将类的class文件读入内存，并为之创建一个java.lang.Class对象，也就是说当程序中使用任何类时，系统都会为之建立一个java.lang.Class对象, 作为方法区这个类的各种数据的访问入口

## 2.2.2 Linking(链接)

### 2.2.2.1 Verification(验证)

> 验证文件是否符合JVM规范，属性与方法是否合理，方法体是否正确等。主要有以下四个步骤：
>
> 1. 文件格式验证
>
> 验证文件是否符合Class文件的规范，且能否被当前虚拟机支持
>
> - - 是否以魔数CAFEBABE开头
>   - 主、次版本号是否在被当前虚拟机支持
>   - 常量池中的常量是否有不被支持的常量类型(检查常量TAG标志)
>   - 指向常量的各种索引值是否有指向不存在的常量或不符合类型的常量
>   - CONSTANT_Utf8_info型的常量中是否有不符合 UTF8编码的数据
>   - Class 文件中各个部分及文件本身是否有被删除的或附加的其他信息等
>
> 1. 元数据验证
>
> - - 这个类是否有父类(除了`java.lang.Object`都应该有父类)
>   - 这个类(以及父类)是否继承了不允许继承的类(被 `final` 修饰)
>   - 如果这个类不是抽象类，是否实现了父类(或者接口)中需要实现的所有方法
>   - 类中的字段、方法是否与父类产生了矛盾。(覆盖了父类的final字段，重载方法但是只有返回值不一致等)
>
> 1. 字节码验证
>
> 第三阶段是整个验证过程中最复杂的一个阶段, 主要目的是通过数据流和控制流的分析，确定语义是合法的。符号逻辑的。在第二阶段对元数据信息中的数据类型做完校验后，这阶段将对类的方法体进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的行为，例如：
>
> - - 保证任意时刻操作数栈的数据装型与指令代码序列都能配合工作, 例如不会出现类似这样的情况:在操作栈中放置了一个 int类型的数据, 使用时却按long类型来加载入本地变量表中
>   - 保证跳转指令不会跳转到方法体以外的字节码指令上
>   - 保证方法体中的类型转换是有效的。父类指针可以指向子类对象，反过来是不正确的。
>
> 1. 符号引用验证
>
> 最后一个阶段的校验发生在虚拟机将符号引用转化为直接引用的时候 , 这个转化动作将在连接的第三个阶段——解析阶段中发生。符号引用验证可以看做是对类自身以外(常量池中的各种符号引用) 的信息进行匹配性的校验, 通常需要校验以下内容:
>
> - - 符号引用中通过字将串描述的全限定名是否能找到对应的类
>   - 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段  
>   - 符号引用中的类、字段和方法的访问性(private、 protected、 public、 default)是否可被当前类访问
>
> **验证阶段对JVM虚拟机来说是重要的，但不是必须的，如果可以确保代码安全，可以使用****`-Xverify:none`** **参数关闭类的大部分的验证。**

### 2.2.2.2 Preparation(准备)

> 对类的**类变量**分配内存并设置默认值，这时操作的内存为方法区内存。此过程中会将**类变量(static修饰的变量且没被final修饰)**设置默认值。被`static final` 修饰的变量经过准备阶段会被JVM设置为初始值。

### 2.2.2.3 Resolution(解析)

> - **将虚拟机常量池中的各种符号引用解析为指针、偏移量等内存地符号址的直接引用**
>
> - - **符号引用**：符号引用是一组符号来描述所引用的目标对象，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标对象并不一定已经加载到内存中。
>   - **直接引用**：直接引用可以是直接指向目标对象的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是与虚拟机内存布局实现相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同，如果有了直接引用，那引用的目标必定已经在内存中存在。
>
> - 解析动作主要针对类或接口、字段、类方法和接口方法的符号引用进行。分别对应常量池的以下内容：
>
> 1 CONSTANT_Class_Info 2 CONSTANT_Fieldref_Info 3 CONSTANT_Methodef_Info 4 CONSTANT_InterfaceMethoder_Info



## 2.2.3 Initializing(初始化)

> **调用类初始化代码，给静态变量赋正确的初始值**
>
> **
> **
>
> 1. 如果该类的直接父类没有初始化，先初始化父类
> 2. 如果类中有初始化语句，则JVM先依次执行初始化语句



**类的初始化细节： 引自：**[**Java类加载的过程_持之以恒！---- 类的初始化一节**](https://blog.csdn.net/xuemengrui12/article/details/82707473/#类的初始化)

- <clinit>()方法是由编译器自动收集类中的所有类变量的赋值动作和静志语句块(static{}块)中的语句合并产生的, 编译器收集的顺序是由语句在源文件中出现的顺序所决定的, 静态语句块中只能访问到定义在静态语句块之前的变量, 定义在它之后的変量 , 在前面的静态语句块可以赋值 , 但是不能访问
- <clinit>()方法与类的构造函数 (或者说实例构造器<init>()方法)不同，它不需要显式地调用父类构造器, 虚期机会保证在子类的<clinit>()方法执行之前, 父类的<clinit>()方法已经执行完毕, 因此在虚期机中第一个被执行的<clinit>()方法的类肯定是 java,lang.Object

- 由于父类的<clinit>()方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作

- <clinit>()方法对于类或接口来说并不是必须的, 如果一个类中没有静态语句块,也没有对变量的赋值操作, 那么编译器可以不为这个类生成<clinit>()方法

- 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作, 因此接口与类一样都会生成<clinit>()方法。 但接口与类不同的是, 执行接口的<clinit>()方法不需要先执行父接口的<clinit>()方法。只有当父接口中定义的变量被使用时, 父接口才会被初始化。 另外, 接口的实现类在初始化时也一样不会执行接口的<clinit>()方法

- 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁和同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()法,其他线程部需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作, 那就可能造成多个进程阻塞, 在实际应用中这种阻塞往往是隐蔽的。

## 2.2.4 Usage(使用)

## 2.2.5 Unloading(卸载)



# 2.3 编译

## 解释器

bytecode intepreter

## JIT

Just In-Time compiler

## 混合模式

- 混合使用解释器 + 热点代码编译
- 起始阶段采用解释执行
- 热点代码检测  `**-XX:CompileThreshold = 10000**`

- - 多次被调用的方法(方法计数器：监测方法执行频率)
  - 多次被调用的循环体(循环计数器：监测循环执行频率)
  - 进行编译

## 配置

- `-Xmixed` 默认为混合模式：开始解释执行，启动速度较快，对热点代码实行检测和编译
- `-Xint` 使用解释模式，启动很快，执行稍慢
- `-Xcomp` 使用纯编译模式，启动很慢，执行很快

```
public class Code_2_3_WayToRun {
    public static void main(String[] args) {
        for (int i = 0; i < 10_0000; i++)
            m();

        long start = System.currentTimeMillis();
        for (int i = 0; i < 10_0000; i++) {
            m();
        }
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }

    public static void m() {
        for (long i = 0; i < 10_0000L; i++) {
            long j = i % 3;
        }
    }
}
```





# 参考文章

- [深入详细讲解JVM原理_lzhpo-CSDN博客_jvm](https://blog.csdn.net/know9163/article/details/80574488)


## 1- JVM简介



## 字节码

Jvm字节码（包含Java字节码...）

![image-20201225132439776](https://gitee.com/clancy/images/raw/master/img/image-20201225132439776.png)



混合编程，系统不再只使用一种语言。

学习最好的方法就是去实践：《自己动手写Java虚拟机》（go语言，写需要20多天）

## 虚拟机

虚拟的计算机，来执行虚拟计算机指令。

分系统虚拟机和程序虚拟机

专门运行Java的字节码的虚拟机称为Java虚拟机。

特点：

- 一次编译，到处运行
- 自动内存管理
- 自动垃圾回收功能

了解JVM封装内容的原理。



## jvm的位置

![image-20201225135759966](https://gitee.com/clancy/images/raw/master/img/image-20201225135759966.png)



## JVM的整体结构

简单版：

![image-20201225140135843](https://gitee.com/clancy/images/raw/master/img/image-20201225140135843.png)

详细版：

![image-20201225140120729](https://gitee.com/clancy/images/raw/master/img/image-20201225140120729.png)

## Java代码执行流程



详细版

![image-20201225140642077](https://gitee.com/clancy/images/raw/master/img/image-20201225140642077.png)



## JVM的架构模型



![image-20201228125651530](https://gitee.com/clancy/images/raw/master/img/image-20201228125651530.png)



例子：

```java
public class StackTest {

    public static void main(String[] args) {
        int a = 10;
        int b = 20;
        int c = a + b;
    }

}
```



使用 `javap -c StackTest.class` 反编译

```c
Compiled from "StackTest.java"
public class com.chaocode.jvm.learn.StackTest {
  public com.chaocode.jvm.learn.StackTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10 // 入栈a
       2: istore_1 // 存储在栈的1号位
       3: bipush        20 // 入栈b
       5: istore_2 // 存储在栈的2号位
       6: iload_1 // 加载栈的1号位
       7: iload_2 // 加载栈的1号位
       8: iadd // 执行相加操作
       9: istore_3 // 把结果存储在栈的3号位
      10: return
}
```



由于跨平台性的设计，Java的指令都是根据栈来设计的。

栈：

- 跨平台性
- 指令集小
- 指令多
- 执行性能比寄存器差



## 生命周期



###  JVM启动

Java虚拟机的启动是通过引导类加载器（bootstarp classloader）创建一个初始类（initial class）来完成的，这个类是由虚拟机的具体实现指定的。



### 虚拟机的执行
- 一个运行中的Java虚拟机有着一个清晰的任务：执行Java程序。
- 程序开始执行时他才运行，程序结束时他就停止。
- 执行一个所谓的Java程序的时候,真真正正在执行的是一个叫做Java虚拟机的进程



### 虚拟机的退出

有如下的几种情况

- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而异常终止
- 由于操作系统出现错误而导致Java虚拟机进程终止
- 某线程调用 Runtime类或 System类的exit方法，或 Runtime类的halt方法,并且Java安全管理器也允许这次exit或halt操作。
- 除此之外，JNI ( Java Native interface)规范描述了用 JNI Invocation API来加载或卸载Java虚拟机时，Java虚拟机的退出情况。



## JVM发展历程

### Sun Classic VM

第一款商用的Java虚拟机。只提供了解释器。（启动快，运行慢。步行，不用等公交）不支持即时编译器。（启动慢，运行快。公交，需要等和换乘）（即时编译器，能提高效率。（缓存））。二者搭配使用最好。

### Exact VM

jdk1.2提供此JVM

Exact Memory Management：准确式内存管理

- 也可以叫Mon- Conservative/ ccurate Memory Management
- 虚拟机可以知道内存中某个位置的数据具体是什么类型

具备现代高性能虚拟机的雏形

- 热点探
- 编译器与解释器混合工作模式

只在 Solaris平台短暂使用,其他平台上还是 classic vm，英雄气短,终被 Hotspot虛拟机替换

### HotSpot VM（商用三款之一）

武林霸主。（有方法区）

应用桌面，服务器，移动端。

### BEA 的 JRockit（商用三款之二）

应用服务器端。不包含解析器，全部代码靠即时编译后执行。所以是最快的 JVM。

JMC（Java Mission Control）监控管理套件。

2008年，BEA被Oracle收购。

### IBM 的 J9（商用三款之三）

全称：IBM Technology for Java Virtual Machine, 简称IT4J，内部代号： J9

在IBM的机器上性能比较好。

### Azul VM

与硬件高度绑定耦合。

### Liquid VM

BEA公司开发，直接运行在自家Hypervisor系统上。

### Apache Harmony

IBM和Intel联合开发的开源JVM，受到同样开源的OpenJDK的压制。

大部分被Android SDK吸纳使用。

### Microsoft JVM

只能在windows下运行。性能很好。被Sun告侵权，从windows抹掉了。

### TaobaoJVM

由AliJVM团队发布。基于OpenJDK，开发了自己的定投版本AlibabaJDK，简称AJDK。是整个阿里Java体系的基石。

基于OpentJDK HotSpot VM发布的国内第一个优化、深度定投且开源的高性能服务器版Java虚拟机

- 创新的GcIH( Gc invisible heap)技术实现了off-heap,即将生命周期较长的Java对象从heap中移到heap之外,并且GC不能管理GCIH内部的Java对象,以此达到降低GC的回收频率和提升GC的回收效率的目的
- GCIH中的对象还能够在多个Java虚拟机进程中实现共享
- 使用cxc32指令实现 JVM intrinsic降低JNI的调用开销
- PMU hardware的 Java profiling too1和诊断协助功能
- 针对大数据场景的 ZenGO

注：学习技术，怎么看技术的发展趋势。看大公司使用的技术，大公司是风向标。（如阿里的Flink）

### Dalvik VM

执行的dex（Dalvik Executable）文件。

### Graal VM

"Run Programs Faster Anywhere"号称。

跨语言全栈虚拟机，可以作为“任何语言”的运行平台。

最有可能的下一代虚拟机。




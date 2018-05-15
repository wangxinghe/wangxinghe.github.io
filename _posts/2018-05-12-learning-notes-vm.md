---
layout: post
comments: true
title: "Android学习笔记——虚拟机"
description: "Android学习笔记——虚拟机"
category: Android
tags: [Android]
---

**1、虚拟机分类**    
**（1）JVM**    
**（2）Dalvik**    
**（3）ART**    
**2、参考文档**

<!--more-->

### 1、虚拟机分类    

#### （1）虚拟机类型    
`JVM`、`Dalvik`、`ART(Android Runtime)`

为什么需要Java虚拟机？    
实现`平台无关性`和`语言无关性`。    
平台无关性指的是，程序语言不需要关心操作系统、机器指令集，程序语言只需要编译成.class字节码就可以在不同系统上运行。    
语言无关性指的是，不区分编程语言，只要是能编译成.class字节码就可以运行，如除Java外，Clojure,Groovy,JRuby,Jython,Scala等也可以在JVM上运行

#### （2）执行方式    
`解释执行`、`编译执行`和`混合模式`。    
`解释执行`：逐行读入，逐行解释并翻译成机器码执行。    
`编译执行`：对于所有的方法，无论是否热点代码，都会编译执行，常见的编译器有JIT编译器。    
`混合模式`：部分代码解释执行，部分代码编译执行。一般热点代码编译执行，非热点代码解释执行。        

解释执行的效率比较低。

**编译方式：**    
`JIT`：Just-in-time，即时编译器。      
`AOT`：Ahead-of-time，预编译（安装期间）；或者也可以叫All-Of-the-Time，全时段编译（运行期间）。    


Interpret+JIT混合编译模式如下：    

![](/image/2018-05-12-learning-notes-vm/mixed-compile.png)

即对于热代码

Interpret+JIT+AOT混合编译模式是在Interpret+JIT的基础上


JavaScript：使用Interpret解释执行方式。    
Java平台：HotSpot JVM使用Interpret+JIT混合模式。    
Android平台：Dalvik（2.x~4.x）使用Interpret+JIT混合编译，ART（5.x~6.x）使用AOT编译，ART（7.x）使用Interpret+JIT+AOT混合编译  


#### （3）JVM、Dalvik、ART对比    

![](/image/2018-05-12-learning-notes-vm/jvm-dvm.png)

![](/image/2018-05-12-learning-notes-vm/jvm-dalvik-art.svg)
  

JVM vs. Dalvik／ART：    
（1）JVM是用于普通Java程序的虚拟机，Dalvik是Android系统的虚拟机    
（2）JVM基于栈的架构（stack based），Dalvik基于寄存器的架构（reg based）    
（3）JVM处理.class字节码，Dalvik处理.dex文件    
（4）HotSpot JVM使用Interpret+JIT混合模式，Dalvik（2.x~4.x）使用Interpret+JIT混合编译，ART（5.x~6.x）使用AOT编译，ART（7.x）使用Interpret+JIT+AOT混合编译    
（5）JVM只能运行一个实例，也就是所有应用都运行在同一个JVM中；Dalvik允许运行多个虚拟机实例，每一个应用启动都运行一个单独的虚拟机，并且运行在一个独立的进程中。

Dalvik vs. ART：    
（1）Android 2.x~4.x使用的是Dalvik虚拟机，Android 5.x以上是ART虚拟机    
（2）编译方式。Dalvik（2.x~4.x）使用Interpret+JIT编译，ART（5.x~6.x）使用AOT编译，ART（7.x）使用Interpret+JIT+AOT混合编译    
（3）安装期间。Dalvik（2.x~4.x）安装期间dex->odex，ART（5.x~6.x）安装期间dex->oat，ART（7.x）安装期间啥也不做    
（4）Dalvik（2.x~4.x）运行速度慢，占用内存少；ART（5.x~6.x）安装时间长，占用内存多，但是启动和运行速度快；ART（7.x）安装速度快，前几次运行速度慢，到后面运行速度快，占用内存不多。    

PS：补充下Android打包流程：

![](/image/2018-05-12-learning-notes-vm/android-package.png)


#### （4）JVM、Dalvik、ART特点：    

参考：[https://www.zhihu.com/question/55652975](https://www.zhihu.com/question/55652975)    

（1）Android 4.x(Interpreter + JIT)    
原理：平时代码走解释器，但热点trace会执行JIT进行即时编译    
优点：占用内存少    
缺点：耗电(退出App下次启动还会重复编译)，卡顿(JIT编译时)    

（2）Android 5.0/5.1/6.0(interpreter + AOT)    
原理: 在AOT模式下，App在安装过程时， 就会完成所有编译。    
优点: 性能好    
缺点: App安装时间长，占用存储空间多。

（3）Android 7.0/7.1的ART引入了全新的Hybrid模式(Interpreter + JIT + AOT)    
原理:

- App在安装时不编译， 所以安装速度快。
- 在运行App时， 先走解释器， 然后热点函数会被识别，并被JIT进行编译， 存储在jit code cache， 并产生profile文件(记录热点函数信息)。
- 等手机进入charging和idle状态下， 系统会每隔一段时间扫描App目录下profile文件，并执行AOT编译(Google官方称之为profile-guided compilation)。
- 不论是jit编译的binary code, 还是AOT编译的binary code, 它们之间的性能差别不大， 因为它们使用同一个optimizing compiler进行编译。    

优点: App安装速度快，占用存储少(只编译热点函数)。    
缺点: 前几次运行会较慢， 只有用户操作得次数越多，jit 和AOT编译后， 性能才会跟上来。综上，Android 7.0/7.1上的ART是将Android 4.x的JIT和Android 5.x/6.0上的AOT结合，取长补短，从而在performance和battery之间取得某种trade off.

可以结合官网来理解： [https://source.android.com/devices/tech/dalvik/jit-compiler](https://source.android.com/devices/tech/dalvik/jit-compiler)    

### 2、参考文档

（1）[深入浅出JIT编译器](https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html)    
（2）[Android系统下编译模式的演变](http://panwei.info/2017/07/05/Android%E7%B3%BB%E7%BB%9F%E4%B8%8B%E7%BC%96%E8%AF%91%E6%A8%A1%E5%BC%8F%E7%9A%84%E6%BC%94%E5%8F%98/)    
（3）[实现 ART 即时(JIT)编译器](https://source.android.com/devices/tech/dalvik/jit-compiler)    
（4）[关于Dalvik、ART、DEX、ODEX、JIT、AOT、OAT](http://rangerzhou.top/2017/06/30/%E5%85%B3%E4%BA%8EDalvik%E3%80%81ART%E3%80%81DEX%E3%80%81ODEX%E3%80%81JIT%E3%80%81AOT%E3%80%81OAT/)    




---
layout: post
comments: true
title: "Android学习笔记——进程和线程"
description: "Android学习笔记——进程和线程"
category: Android
tags: [Android]
---


**1、进程和线程的关系**        
**2、进程**    
**（1）进程分类**    
**（2）进程间通信**    
**（3）进程保活**    
**3、线程**    
**（1）线程分类**    
**（2）线程调度**    
**（3）线程间通信**    
**4、多线程编程**    
**（1）线程协作**    
**（2）生产者消费者**    
**5、参考文档**     


<!--more-->


### 1、进程和线程的关系        

### 2、进程    

#### （1）进程分类    

按优先级分为5类：`前台进程`、`可见进程`、`服务进程`、`后台进程`、`空进程`。

这个分类是根据进程中正在运行的组件的状态进行分类的。

`前台进程`：用户当前操作所必需的进程。如托管Activity（已调用onResume()）、Service（和前台正在交互的Activity绑定，或已调用startForeground()，或执行了onCreate()/onStart()/onDestroy()）、BroadcastReceiver（正执行onReceive()）等前台运行的组件的进程。

`可见进程`：没有任何前台组件，但仍会影响用户在屏幕上所见内容的进程。如托管Activity（已调用onPause()）、Service（绑定到可见onPause()的Activity）

`服务进程`：正在运行已使用startService()方法启动的服务，且不属于上述两个更高级别进程的进程。如Service（后台播放音乐、从网络下载数据场景）    

`后台进程`：包含目前对用户不可见的Activity的进程。如Activity（已调用onStop()）。后台进程会保存在一个LRU列表中，以确保最近查看的Activity的进程最后一个被终止。    

`空进程`：不包含任何活动应用组件的进程，系统往往会终止这些进程。作用：用作缓存，以缩短下次在其中运行组件所需的启动时间。    

从上面可以看出，运行Service的进程至少是服务进程，运行Activity的进程至少是后台进程。

#### （2）进程间通信    

Linux进程间通信的方式：    
（1）`管道`（Pipeline）：`无名管道`，半双工通信，数据单向通信，且在父子进程间通信；`有名管道`，半双工通信，允许无亲缘关系进程间通信。管道的实质是一个内核缓冲区。    
（2）`信号量`（Semophore）：是一个计数器，用来控制多个进程对共享资源的访问。    
（3）`消息队列`（Message Queue）：存放在内核中的消息链表，每个消息队列由消息队列标志符表示。    
（4）`信号`（Signal）：用于通知接收进程某个事件已经发生。    
（5）`共享内存`（Shared Memory）：映射一段能被其他进程访问的内存，这段内存由一个进程创建，但多个进程都可以访问。    
（6）`套接字`（Socket）：与其他通信机制的区别是，可用于不同机器间的进程通信。

Android系统进程间通信方式：    
（1）`文件共享`：不适用于并发场景。    
（2）`Socket`：网络数据交换场景。    
（3）`Binder`：AIDL、Messageer、BroadcastReceiver、ContentProvider通信本质上都是基于Binder实现。    

Zygote和system_server进程，Zygote和App进程的通信都是Socket通信；其他情况是Binder通信。    


#### （3）进程保活    

### 3、线程    

#### （1）线程分类    

#### （2）线程调度    

#### （3）线程间通信    

### 4、多线程编程    

#### （1）线程协作    

#### （2）生产者消费者    

### 5、参考文档     

（1）[进程和线程](https://developer.android.com/guide/components/processes-and-threads?hl=zh-cn)    
（2）[进程间通信IPC (InterProcess Communication)](https://www.jianshu.com/p/c1015f5ffa74)    





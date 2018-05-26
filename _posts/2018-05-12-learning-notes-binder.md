---
layout: post
comments: true
title: "Android学习笔记——Binder"
description: "Android学习笔记——Binder"
category: Android
tags: [Android]
---

**1、为什么采用Binder机制**    
**2、Binder架构图**     
**3、Binder通信过程**     
**4、Java层Binder分析**     
**5、参考文档**    

<!--more-->

### 1、为什么采用Binder机制    

#### （1）Binder能够很好的实现Client-Server架构    

**管道／消息队列／共享内存／信号量** 等IPC通信手段不支持Client-Server架构    

**Socket** 支持Client-Server架构但主要用于网络间通信和本机进程间的低速通信，传输效率低。

#### （2）Binder的传输效率和可操作性很好    

**管道／消息队列** 采用存储－转发方式，需要经过2次内存拷贝，即先从发送方的缓存区（Linux中的用户存储空间）拷贝到内核的缓存区（Linux中的内核存储空间），再从内核缓存区拷贝到接收方的缓存区（Linux中的用户存储空间）。    

**共享内存** 内存拷贝次数为0，但操作复杂。

**Socket** 传输效率低。    

**Binder** 只需要1次内存拷贝，即从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的。

#### （3）Binder机制的安全性很高    

**管道／消息队列／共享内存／信号量／Socket** 接收方无法获得发送方可靠的UID／PID（用户ID／进程ID），无法鉴别对方身份。    

**Binder** 为每个进程分配UID／PID作为鉴别身份的标识，并且会利用UID／PID进行有效性检测。    


### 2、Binder架构图     

![](/image/2018-05-12-learning-notes-binder/binder_frame.png)    

### 3、Binder通信过程     

![](/image/2018-05-12-learning-notes-binder/IPC-Binder.jpg)    


![](/image/2018-05-12-learning-notes-binder/binder_4relationship.jpg)    

### 4、Java层Binder分析     



### 5、参考文档    

[Android Binder机制(一) Binder的设计和框架](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)    

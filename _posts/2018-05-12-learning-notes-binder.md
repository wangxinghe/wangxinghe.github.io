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

Android系统是基于Linux的操作系统，内存最大为4G，0～3G为`用户空间`，3～4G为`内核空间`。    

应用程序运行在用户空间，Kernel和驱动运行在内核空间。不同应用程序之间的通信需要通过内核进行中转。    

用户空间：ServiceManager、Server、Client。    
内核空间：Binder驱动。    

PS：所谓用户空间并不是说Framework层以上的应用层，Kernel层之上的Native／Framework层都属于用户空间。

### 3、Binder通信过程     

#### 简略版的通信过程：

![](/image/2018-05-12-learning-notes-binder/IPC-Binder.jpg)    

（1）`注册服务`(addService)：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。    
（2）`获取服务`(getService)：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。    
（3）`使用服务`：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。    

以上这个过程并不是Framework层之上的通信，而是Native层和Kernel层之间的通信。Framework层将通信。

看一下[Binder系列7—framework层分析](http://gityuan.com/2015/11/21/binder-framework/) 可能理解的更好。

#### 详细版的通信过程：

![](/image/2018-05-12-learning-notes-binder/binder_4relationship.jpg)    

基本元素介绍：    
（1）`Binder实体`:是各个Server以及ServiceManager在内核空间的存在形式，是一个binder_node结构体对象。    
（2）`Binder引用`：是Binder实体的引用，即binder_ref结构体对象。    
（3）`远程服务`：如果将Server本身看做“本地服务”的话，Client中的“远程服务”就是本地服务的代理。    

具体通信过程：    
1 `注册服务`(addService)：    
（1）Server首先向Binder驱动发起注册请求。告诉Binder驱动这个请求交给`0号Binder引用`对应的进程处理。    
（2）Binder驱动转发请求给ServiceManager。转发之前做的事情：a.新建该Server对应的`Binder实体`；b.在ServiceManager的保存Binder引用的红黑树中查找是否存在该Server的`Binder引用`，不存在的话新建一个引用添加到红黑树中；    
2 `获取服务`(getService)：    
（1）Client向Binder驱动发起获取服务的请求。    
（2）Binder驱动转发请求给ServiceManager。    
（3）ServiceManager通过Binder驱动将Server信息反馈给Client。做的事情：ServiceManager从Binder引用的单链表中，根据Server的服务名查找Server对应的Binder实体的引用，并返回。        
3 `使用服务`：    
（1）Client根据返回的Binder引用，创建一个Server对应的远程服务，通过这个远程服务调用Server的服务接口。        

#### 交互模型：    

![](/image/2018-05-12-learning-notes-binder/binder_communication.jpg)    


- (01) Server进程启动之后，会进入中断等待状态，等待Client的请求。
- (02) 当Client需要和Server通信时，会将请求发送给Binder驱动。
- (03) Binder驱动收到请求之后，会唤醒Server进程。
- (04) 接着，Binder驱动还会反馈信息给Client，告诉Client：它发送给Binder驱动的请求，Binder驱动已经收到。
- (05) Client将请求发送成功之后，就进入等待状态。等待Server的回复。
- (06) Binder驱动唤醒Server之后，就将请求转发给Server进程。
- (07) Server进程解析出请求内容，并将回复内容发送给Binder驱动。
- (08) Binder驱动收到回复之后，唤醒Client进程。
- (09) 接着，Binder驱动还会反馈信息给Server，告诉Server：它发送给Binder驱动的回复，Binder驱动已经收到。
- (10) Server将回复发送成功之后，再次进入等待状态，等待Client的请求。
- (11) 最后，Binder驱动将回复转发给Client。

### 4、Java层Binder分析     



### 5、参考文档    

（1）[Android Binder机制(一) Binder的设计和框架](http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/)    
（2）[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)    

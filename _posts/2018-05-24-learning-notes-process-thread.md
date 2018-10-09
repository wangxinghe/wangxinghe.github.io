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
**（1）线程优先级**    
**（2）线程调度**    
**（3）线程间通信**    
**4、多线程编程**    
**（1）线程协作**    
**（2）生产者消费者**    
**5、参考文档**     


<!--more-->


### 1、进程和线程的关系        

进程是系统进行资源分配和调度的一个独立单位，拥有独立的内存单元。线程是比进程更小的可独立运行的基本单位。

区别：    
`调度方面`：进程是资源的拥有单位，线程是独立的调度和分派单位，可以显著的提高并发度和减少切换开销。    
`并发性`：进程间可以并发，同一进程的线程间也可以并发，有效提高系统资源的利用率和吞吐量。    
`拥有资源`：进程是基本的资源拥有单位，线程只拥有很少的资源，但是线程可以访问所属进程的资源。    
`系统开销`：创建或者撤销进程的时候，系统要为之创建或回收PCB，系统资源等，切换时也需要保存和恢复CPU环境。而线程的切换只需要保存和恢复少量的寄存器，不涉及存储器管理方面的工作，所以开销较小。     

### 2、进程    

#### （1）进程分类    

按优先级分为5类：`前台进程`、`可见进程`、`服务进程`、`后台进程`、`空进程`。

这个分类是根据进程中正在运行的组件的状态进行分类的。

`前台进程`：用户当前操作所必需的进程。如托管Activity（已调用onResume()）、Service（和前台正在交互的Activity绑定，或已调用startForeground()，或执行了onCreate()/onStart()/onDestroy()）、BroadcastReceiver（正执行onReceive()）等前台运行的组件的进程。

`可见进程`：没有任何前台组件，但仍会影响用户在屏幕上所见内容的进程。如托管Activity（已调用onPause()）、Service（绑定到可见onPause()的Activity）

`服务进程`：正在运行已使用startService()方法启动的服务，且不属于上述两个更高级别进程的进程。如Service（后台播放音乐、从网络下载数据场景）    

`后台进程`：包含目前对用户不可见的Activity的进程。如Activity（已调用onStop()）。后台进程会保存在一个LRU列表中，以确保最近查看的Activity的进程最后一个被终止。守护进程也属于后台进程。    

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

Android系统中，Zygote和system_server进程，Zygote和App进程的通信都是Socket通信；其他情况是Binder通信。    


#### （3）进程保活    

进程保活方式：`厂商白名单`、`启动前台服务`、`进程相互唤醒`、`利用系统漏洞`。

`厂商白名单`：如有些手机厂商把知名App加入白名单，保证进程不死来提高用户体验。    

`启动前台服务`：开启一个前台服务，如启动一个前台Activity或Service。开启一个像素的Activity，如手Q方案；启动一个前台Service，在系统通知栏生成一个Notification，哪怕App退到后台，进程也会一直运行。    

`进程相互唤醒`：主要包括3种场景，即利用系统进程唤醒、第三方SDK唤醒、同体系App唤醒。    

（1）系统进程唤醒：开机、网络切换（CONNECTIVITY_ACTION）、拍照（ACTION_NEW_PICTURE）、拍视频（ACTION_NEW_VIDEO）时，利用系统产生的广播唤醒App。开机广播有些定制ROM的厂商已将其去掉，Android N之后已取消网络切换、拍照、拍视频3种广播        
（2）第三方SDK唤醒：比如某个App中集成微信SDK，则微信SDK会唤醒微信App；比如使用信鸽或友盟SDK的App，相互之间会相互唤醒等。    
（3）同体系App唤醒：如阿里系的App，比如淘宝、天猫、支付宝等之间会相互唤醒。    

`利用系统漏洞`：在`启动前台服务`这种方式上，想办法把通知栏去掉。    
原理如下：
对于 API level < 18 ：调用startForeground(ID， new Notification())，`发送空的Notification` ，图标则不会显示。    
对于 API level >= 18：在需要提优先级的Service A启动一个InnerService，两个服务同时startForeground，且绑定同样的 notificationId，然后Stop掉InnerService ，这样通知栏图标即被移除，但是Service A还活着。    

从大的方面，根据优先级进程可以分为：`前台进程`、`可见进程`、`服务进程`、`后台进程`、`空进程`。    
从更具体方面，每个进程都有一个`oom_adj`，代表进程优先级值，这个值越小优先级越高，比如前台进程oom_adj的值为0，空进程最大。    

系统往往会杀`优先级低且内存占用大`的进程。先按照oom_adj杀，oom_adj相同的话，按照内存占用杀。    

参考：    
[Android进程保活的一般套路](https://www.jianshu.com/p/1da4541b70ad)    
[微信Android客户端后台保活经验分享](https://mp.weixin.qq.com/s?__biz=MzUxMzcxMzE5Ng==&mid=2247488488&amp;idx=1&amp;sn=f76fb0a8f88f6958d6a7ecfe6658b5a5&source=41#wechat_redirect)    
[Android进程保活的一般套路](https://www.jianshu.com/p/1da4541b70ad)    

### 3、线程    

#### （1）线程优先级    

Java线程优先级使用1～10的整数表示，值越大优先级越高。

Thread.MIN_PRIORITY（1）、Thread.NORM_PRIORITY（5）、Thread.MAX_PRIORITY（10）

线程优先级只在局部起作用，不保证线程执行的顺序，真正的执行顺序和操作系统及虚拟机版本有关。只能说优先级高的线程获取CPU资源的概率高。    

#### （2）线程调度    

![](/image/2018-05-24-learning-notes-process-thread/thread_schedule.png)

图片来自《深入理解Java虚拟机》

#### （3）线程间通信    

`主线程和子线程`：AsyncTask、Handler。    

从其他线程访问UI线程（本质还是Handler机制）：    

Activity.runOnUiThread(Runnable)    
View.post(Runnable)    
View.postDelayed(Runnable, long)    

    public void onClick(View v) {
        new Thread(new Runnable() {
            public void run() {
                final Bitmap bitmap =
                        loadImageFromNetwork("http://example.com/image.png");
                mImageView.post(new Runnable() {
                    public void run() {
                        mImageView.setImageBitmap(bitmap);
                    }
                });
            }
        }).start();
    }


`子线程和子线程`：Handler

### 4、多线程编程    

#### （1）线程协作    

死锁

wait()/notify()、sleep()/interrupt()

#### （2）生产者消费者    

[生产者/消费者问题的多种Java实现方式](https://blog.csdn.net/monkey_d_meng/article/details/6251879)    
[Java实现生产者和消费者的5种方式](https://juejin.im/entry/596343686fb9a06bbd6f888c)    


### 5、参考文档     

（1）[Processes and threads overview](https://developer.android.com/guide/components/processes-and-threads)    
（2）[进程间通信IPC (InterProcess Communication)](https://www.jianshu.com/p/c1015f5ffa74)    
    





---
layout: post
comments: true
title: "Android学习笔记——Android体系结构总结"
description: "Android学习笔记——Android体系结构总结"
category: Android
tags: [Android]
---

**1、Android系统框架**    
**2、App启动过程**     
**3、Activity启动过程**     
**4、参考文档**

<!--more-->

### 1、Android系统框架    


### 2、App启动过程     

![](/image/2018-05-16-learning-notes-android-architect/start_activity_process.jpg)

#### 启动流程：

（1）点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；    
（2）system_server进程接收到请求后，向zygote进程发送创建进程的请求；    
（3）Zygote进程fork出新的子进程，即App进程；    
（4）App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；    
（5）system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；    
（6）App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；    
（7）主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。    

到此，App便正式启动，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染结束后便可以看到App的主界面。 

参考：[http://gityuan.com/2016/03/12/start-activity/](http://gityuan.com/2016/03/12/start-activity/)

### 3、Activity启动过程     

![](/image/2018-05-16-learning-notes-android-architect/start_activity.svg)

上面是App内部Activity A启动Activity B的工作过程，finish过程也类似。

Binder客户端和Binder服务端是相对的。startActivity时，ActivityManagerProxy是客户端，ActivityManagerService是服务端；scheduleLaunchActivity时，ApplicationThreadProxy是客户端，ApplicationThread是服务端。

### 4、参考文档

---
layout: post
comments: true
title: "Android学习笔记——插件化"
description: "Android学习笔记——插件化"
category: Android
tags: [Android]
---



1、类加载器

继承关系：    
`PathClassLoader/DexClassLoader -> BaseDexClassLoader -> ClassLoader`

`PathClassLoader`只能加载/data/app中的`apk`，也就是`已经安装`到手机中的apk。    
`DexClassLoader`可以加载`任何路径`的`apk/dex/jar`    
`URLClassLoader`可以加载java中的jar


双亲委派模型：    

CustomClassLoader -> AppClassLoader -> ExtClassLoader -> BootStrapClassLoader    


[Android中插件开发篇之—-类加载器](http://www.520monkey.com/archives/336)    
[Java高新技术第一篇：类加载器详解](http://www.520monkey.com/archives/151)    
[Android中的动态加载机制](http://www.520monkey.com/archives/146)    



dynamic-load-apk    

VirtualAPK    

[《Android插件化技术——原理篇》](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653579547&idx=1&sn=9f782f6c91c20fd0b17a6c3762b6e06a&chksm=84b3bb1cb3c4320ad660e3a4a274aa2e433bf0401389f38be337d01d2ba604714303e169d48a&mpshare=1&scene=23&srcid=0111lAPa4UGPssFMoc05pgLP#rd)    

[Android开源框架源码鉴赏：VirtualAPK](https://blog.souche.com/untitled-9/)    


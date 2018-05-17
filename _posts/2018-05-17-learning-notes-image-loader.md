---
layout: post
comments: true
title: "Android学习笔记——图片加载"
description: "Android学习笔记——图片加载"
category: Android
tags: [Android]
---

**1、ImageLoader框架**    
**（1）整体流程**    
**（2）数据存取**    
**（3）ImageDecoder**    
**2、各框架比较**    
**3、参考文档**    

<!--more-->

### 1、ImageLoader框架    

#### （1）整体流程    

![](/image/2018-05-17-learning-notes-image-loader/UIL_Flow.png)

**调用方式：**

    ImageLoader imageLoader = ImageLoader.getInstance();
    imageLoader.displayImage(imageUri, imageView);        
    
**缓存机制：**    
主要包括三级缓存：`内存缓存`，`磁盘缓存`，`网络`。    

**数据加载方式：**    
（1）先判断内存是否有数据，如果有则依次走post-process -> Bitmap Display    
（2）如果内存没有数据再判断磁盘是否有数据，如果有则走Image Decoder -> pre-process -> Memory Cache -> post-process -> Bitmap Display     
（3）如果内存和磁盘都没有数据，则从网络获取，依次走Image Downloader -> Disk Cache -> Image Decoder -> pre-process -> Memory Cache -> post-process -> Bitmap Display    

在Bitmap存入Memory Cache和真正显示之前，都会经过Bitmap Processor进行处理（如果外部设置了的话）    

接下来还有几件事情需要搞清楚：    
（1）Bitmap怎么存到磁盘和内存，以及分别怎么从磁盘和内存取？        
（2）存到磁盘和内存前是存原始数据吗？存之前做了哪些优化处理？    

#### （2）数据存取    



#### （3）ImageDecoder    

### 2、各框架比较    

### 3、参考文档    

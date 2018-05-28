---
layout: post
comments: true
title: "Android学习笔记——性能优化"
description: "Android学习笔记——性能优化"
category: Android
tags: [Android]
---

**1、性能优化分类**    
**（1）启动速度优化**    
**（2）View渲染优化**    
**（3）内存优化**    
**（4）网络优化**    
**（5）电量优化**    
**（6）包大小优化**      
**2、参考文档**

<!--more-->

### 1、性能优化分类    

#### （1）启动速度优化    

#### （2）View渲染优化    

主要包括`布局优化`和`绘制优化`两部分：    
`布局优化`：减少层数，也就是减少measure、layout、draw过程的时间。    
`绘制优化`：一个是少绘制不必要的背景图片；还有就是绘制方法里不做耗时操作。    
上面两点都会引出`60fps问题`，从而导致画面卡顿。

`60fps`即一秒绘制60帧，View的绘制频率为60fps时，可以达到流畅画面，即每一帧绘制时间`16ms`。Android系统每隔16ms会发出`VSYNC`信号，触发UI渲染，如果每一帧画面时间过长的话，会出现卡顿。    

（1）`减少布局的层数，使用性能更好的控件`。    
一般来说布局层数越少越好；当使用LinearLayout和RelativeLayout的布局层数相同时，优先选用LinearLayout，因为RelativeLayout布局过程更复杂，效率更低。  
  
（2）`使用<merge>或<ViewStub>标签`。    
比如我们需要在LinearLayout/FrameLayout里面嵌入一个layout，而这个layout根节点也是LinearLayout/FrameLayout，这时候可以使用<merge>标签；<ViewStub>提供了按需加载的功能，调用inflate之后才会将布局加载到内存。    

（3）`避免过度绘制，移除冗余的背景图片`    
过度绘制，指屏幕上的某个像素在同一帧的事件内被绘制了多次（一帧，指一副静止的画面）    
移除不必要的背景，如Window主题的默认背景色或布局冗余背景。    

（4）`onDraw()方法不要new对象、不要有循环操作、不要做耗时任务`    
因为onDraw()方法可能会被频繁调用，频繁创建对象会导致内存占用过多，GC更加频繁，降低程序的执行效率；onDraw()中如果有耗时任务，会导致绘制一帧所占的时间过长，造成画面卡顿。


过度绘制图：    

![](/image/2018-05-12-learning-notes-android-performance-optimization/overdraw.png)

60fps图：

![](/image/2018-05-12-learning-notes-android-performance-optimization/draw_per_16ms.png)

`使用工具`：    

- `Hierarchy Viewer`查看层级    
- `Lint`查看布局优化建议（Android -> Lint -> Performance）    
- 查看过度绘制，`设置 -> 开发者选项 -> Show GPU Overdraw`    

#### （3）内存优化    

内存优化，就是解决`内存溢出`、`内存泄露`、`内存抖动`三个问题。    

（1）内存溢出

（2）内存泄漏    

（3）内存抖动    


#### （4）网络优化    

#### （5）电量优化    

#### （6）包大小优化      

### 2、参考文档    

（1）[http://hukai.me/blog/archives/](http://hukai.me/blog/archives/)    


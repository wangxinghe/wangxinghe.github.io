---
layout: post
comments: true
title: "Android学习笔记——View的Touch事件"
description: "Android学习笔记——View的Touch事件"
category: Android
tags: [Android]
---


**1、Touch事件**    
**（1）事件分发／事件消费**    
**（2）案例分析**    
**2、滑动事件**    
**（1）滑动冲突**    
**3、参考文档**    

<!--more-->

### 1、Touch事件    

官方文档：Manage touch events in a ViewGroup    
[https://developer.android.com/training/gestures/viewgroup](https://developer.android.com/training/gestures/viewgroup)

`ACTION_CANCEL`触发时机    
If you return true from onInterceptTouchEvent(), the child view that was previously handling touch events receives an ACTION_CANCEL, and the events from that point forward are 
sent to the parent's onTouchEvent() method for the usual handling    

`TouchDelegate`: Extend a child view's touchable area    

多指触控：    
[https://developer.android.com/training/gestures/multi](https://developer.android.com/training/gestures/multi)    
[https://developer.android.com/training/gestures/scale](https://developer.android.com/training/gestures/scale)    

`ACTION_DOWN`—For the first pointer that touches the screen. This starts the gesture. The pointer data for this pointer is always at index 0 in the MotionEvent.    
`ACTION_POINTER_DOWN`—For extra pointers that enter the screen beyond the first. The pointer data for this pointer is at the index returned by getActionIndex().    
`ACTION_MOVE`—A change has happened during a press gesture.    
`ACTION_POINTER_UP`—Sent when a non-primary pointer goes up.    
`ACTION_UP`—Sent when the last pointer leaves the screen.    

ACTION_POINTER_UP事件处理：    
In the ACTION_POINTER_UP case, the example extracts this index and ensures that the active pointer ID is not referring to a pointer that is no longer touching the screen. If it is, the app selects a different pointer to be active and saves its current X and Y position. Since this saved position is used in the ACTION_MOVE case to calculate the distance to move the onscreen object, the app will always calculate the distance to move using data from the correct pointer.

参考ScrollView的源码。

#### （1）事件分发／事件消费    

![](/image/2018-05-28-learning-notes-view-touch-event/touch1.jpg)    


#### （2）案例分析    



### 2、滑动事件    

#### （1）滑动冲突    

PS：之前写过一篇[View事件分发及消费源码分析](http://mouxuejie.com/blog/2016-05-01/view-touch-event-source-analysis/)，缺少流程图，导致整个流程不够清晰。    

### 3、参考文档    
（1）[Android事件分发机制](http://gityuan.com/2015/09/19/android-touch/)    
（2）[Android 事件分发机制源码和实例解析](https://www.jianshu.com/p/7daf0feb6c2d)    
（3）[Android View体系之Touch事件传递源码解析(8.0)](http://yummylau.com/2018/03/05/Android_2018-03-05_View%E4%BD%93%E7%B3%BB%E4%B9%8BTouch%E4%BA%8B%E4%BB%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)    
（4）[Android事件分发机制 详解攻略，您值得拥有](https://blog.csdn.net/carson_ho/article/details/54136311)    



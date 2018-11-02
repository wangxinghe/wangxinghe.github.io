---
layout: post
comments: true
title: "Android学习笔记——SharedPreference"
description: "Android学习笔记——SharedPreference"
category: Android
tags: [Android]
---

SharedPreference使用不当会引起ANR。

涉及几个知识点：    
（1）commit() & apply()区别    
（2）apply()都干了啥，涉及到QueuedWork入队    
（3）ActivityThread handleStopActivity的时候QueuedWork.waitToFinish()导致阻塞    

看过别人写的代码，通过重写SharedPreference来缓解ANR


网上有几篇文章也有讲到：    

[全面剖析SharedPreferences](http://gityuan.com/2017/06/18/SharedPreferences/)    
[SharedPreference如何阻塞主线程](https://www.jianshu.com/p/63ee8587de3f)    
[Android的两种数据存储方式分析（一）](https://www.jianshu.com/p/1b0f067afa11)    

还有一篇有意思的文章：    
[请不要滥用SharedPreference](http://weishu.me/2016/10/13/sharedpreference-advices/)


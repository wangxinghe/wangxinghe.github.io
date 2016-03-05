---
layout: post
comments: true
title: "TraceView性能优化工具使用"
description: "TraceView性能优化工具使用"
category: android
tags: [Android]
---

### 一、面板组成

#### 1. Timeline Panel

（1）每一行表示一个线程    
（2）每一种不同颜色代表一个不同的方法    
（3）每一个bar对应一个方法执行过程，bar的宽度表示方法执行的时长    
（4）对于缩小图，bar表示方法开始执行时间；对于放大图，每个方法对应的bar会扩大成一个有色的U型，U型的左边  表示方法开始时间，右边表示方法结束时间    
（5）小技巧：双击上面的时间轴可以缩小图，鼠标选中线程的颜色部分轻微水平拉动可以放大图    

补充（参考4）：    
（1）颜色脉冲bar的高度表示cpu的利用率，高度越高表示cpu利用率越高    
（2）白色gap空白块表示该线程目前没有占CPU，被其他线程占用    
（3）黑块表示系统空闲(system idle)    

#### 2. Profile Panel

（1）表示执行的方法列表    
（2）任意点击一个方法展开，Parent表示调用该方法的方法，Children表示该方法调用的方法    
（3）颜色和Timeline中的颜色含义相同，且选中一个方法会在Timeline中同步高亮    
（4）参数解释：    
`Incl Cpu Time(%)` －方法自身及该方法调用的所有子方法所占时间和（百分比）    
`Excl Cpu Time(%)` －方法自身所占时间（百分比）    
`Incl Real Time(%)` －类似于Incl Cpu Time(%)    
`Excl Real Time(%)` －类似于Excl Cpu Time(%)    
`Calls + Recur Calls/Total` －调用次数＋递归调用次数。或某子方法被该父方法调用的次数／某子方法被所有方法调用的总次数。参考5。    

`Cpu Time/Call` －方法调用一次所占Cpu Time。[Calls + Recur Calls/Total] * [Cpu Time/Call] = [Incl Cpu Time]    
`Real Time/Call` －方法调用一次所占Real Time。[Calls + Recur Calls/Total] * [Real Time/Call] = [Incl Real Time]    
关于Cpu Time和Real Time的区别（参考6）：    
`Cpu Time` 指的是方法执行所占用的cpu时间，不包括中间的等待时间    
`Real Time` 指的是方法从开始执行到结束执行总的时间，包括中间的等待时间    

### 二、产生trace logs的两种方式

（1）更精确方式：Debug.startMethodTracing()，Debug.stopMethodTracing()，生成.trace文件，
         导出到PC目录adb pull /sdcard/*.trace /tmp， 然后DDMS -> Open file
         
（2）次精确方式：打开Tools->Android->Android Device Monitor。在Devices下选中进程，然后选中Start Method Profiling图标，然后在设备上进行可能存在性能问题的操作，然后点击Stop Method Profiling图标

### 三、关于分析方法

性能问题通常分为两类：    
（1）调用次数不多，但每次耗时长的方法    
（2）自身耗时不长，但频繁调用的方法    
关于第一种，通常做法是先按Cpu Time/Call降序排序，然后看Incl Cpu Time的大小，综合起来越大的性能问题越严重    
关于第二种，通常做法是按Calls + Recur Calls/Total降序排序，然后看Incl Cpu Time的大小，综合起来越大的性能问题越严重    

总的来说，还是要自己多试验多体会，具体问题具体分析，我不便于给出绝对或误导人的结论，这里给出我认为很好的两篇博文，见参考文献2和3。    

#### 参考文献：
(0) [http://developer.android.com/intl/zh-cn/tools/performance/traceview/index.html](http://developer.android.com/intl/zh-cn/tools/performance/traceview/index.html)    
(1) [http://developer.android.com/intl/zh-cn/tools/debugging/debugging-tracing.html](http://developer.android.com/intl/zh-cn/tools/debugging/debugging-tracing.html)    
(2) [http://jwzhangjie.cn/2015/07/14/android%E7%B3%BB%E7%BB%9F%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98%E5%B7%A5%E5%85%B7%E4%BB%8B%E7%BB%8D/](http://jwzhangjie.cn/2015/07/14/android%E7%B3%BB%E7%BB%9F%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98%E5%B7%A5%E5%85%B7%E4%BB%8B%E7%BB%8D/)    
(3) [http://myeyeofjava.iteye.com/blog/2250801](http://myeyeofjava.iteye.com/blog/2250801)    
(4) [http://stackoverflow.com/questions/6476991/android-eclipse-traceview-i-just-dont-get-it](http://stackoverflow.com/questions/6476991/android-eclipse-traceview-i-just-dont-get-it)    
(5) [http://stackoverflow.com/questions/28125441/how-to-understanding-callsrecurcalls-total-in-android-traceview-tool/28149711#28149711](http://stackoverflow.com/questions/28125441/how-to-understanding-callsrecurcalls-total-in-android-traceview-tool/28149711#28149711)    
(6) [http://stackoverflow.com/questions/15760447/what-is-the-meaning-of-incl-cpu-time-excl-cpu-time-incl-real-cpu-time-excl-re](http://stackoverflow.com/questions/15760447/what-is-the-meaning-of-incl-cpu-time-excl-cpu-time-incl-real-cpu-time-excl-re)





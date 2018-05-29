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

App启动方式：冷启动、热启动、温启动。    

`冷启动`：App没有启动过或App进程退出时被主动杀死。    

`热启动`：App从后台切换到前台，进程和Activity还活着。    

`温启动`：介于冷启动和热启动之间。    
场景：
（1）用户退出App，然后又启动。App进程还在运行，但Activity需要重建。    
（2）用户退出App，之后`由于内存原因进程被系统杀死`，进程和Activity都需要重建，但是可以在onCreate中恢复之前保存的状态(saved instance state)。    

预防措施：    
Application的onCreate中不要做太多事情。将初始化任务分成不同优先级，实现lazy load。    
首屏Activity尽量简化。

工具：Traceview。

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

`内存溢出`：就是应用程序所占内存超出Dalvik Heap Size的最大值，这个最大值可以通过`ActivityManager.getMemoryClass()`查询。

`内存泄漏`：没被用到的对象由于错误的引用不能及时回收。

`内存抖动`：是由于短时间内有大量对象进出Young Generation区域导致，它伴随着频繁的GC。    
场景：比如循环体内创建对象；比如String拼接创建大量小的对象。    

#### 预防措施：    

（1）使用更加轻量的数据结构。如ArrayMap/SparseArray优于HashMap（当然并不是绝对）    
（2）减少使用Enum。static constants优于enum，当然对于static变量的话不能滥用。    
（3）减少Bitmap对象的内存占用。图片载入内存前，inSampleSize缩放比例；Bitmap的Config配置一般情况选用RBG_565（恰当选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8）。临时的Bitmap对象要回收。    
关于Bitmap可以参考：[玩转Android Bitmap](https://www.jianshu.com/p/3950665e93e6)    
（4）使用更小的图片。或使用tinypng等对图片进行压缩处理。    
（5）对象的复用。如复用系统自带的资源、ListView/GridView中子控件ConvertView的复用及ViewHolder使用、Bitmap使用LRU算法复用。    
（6）避免频繁创建对象。如在onDraw()创建对象、StringBuilder优于＋    
（7）涉及到非UI控件的情况优先使用Application Context，避免Activity泄漏。    
（8）避免数据容器中对象的泄漏。如各种监听器的注销、Cursor对象的关闭。    
（9）IntentService优于Service。    
（10）使用nano protobuf序列化数据。    
（11）谨慎使用第三方框架。如RoboGuice、或者可能有的第三方框架里面会包含重复的库。    


#### 相关理论：

`垃圾回收机制`：从GC Roots出发的引用树的可达性，不可达的将被回收。    

`GC Root`:    
总结下来就4点：System Class相关、JNI相关、Thread相关、Finalize相关。    
`System Class`，Bootstrap/System ClassLoader加载的类，如rt.jar    
`JNI`，JNI中的本地／全局变量或代码（包括用户自定义的和JVM内部的）    
`Thread`，运行中的线程。    
`Thread Block`，运行中的线程引用的对象。    
`Java Local／Native Stack`，Thread方法栈中的传参或局部变量，Native代码中的传参。    
`Busy Monitor`，所有调用了wait()/notify()的对象，或调用了synchronized(Object)/synchronized方法的对象或类（如果是static方法就是类，如果非static方法就是对象）    
`Finalizable`，队列中等待finalizer的对象，或不在队列中但有finalize()方法的对象。    

具体看文档：[http://help.eclipse.org/luna/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Fconcepts%2Fgcroots.html&cp=37_2_3](http://help.eclipse.org/luna/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Fconcepts%2Fgcroots.html&cp=37_2_3)    

`JVM垃圾收集算法`：    
`标记－清除算法`：分为标记和清除2个步骤。缺点：效率都不高，且标记清除后产生大量不连续的内存碎片，当需要分配大对象时可能会提前触发GC操作。        
`复制算法`：将可用内存分为大小相等的两块，每次只使用其中的一块。缺点：内存使用率低。    
`标记－整理算法`：所有存活的对象都向一端移动 ，然后直接清理掉边界以外的内存。    
`分代收集算法`：将内存划分为新生代和老年代，新生代使用复制算法，老年代使用标记－清理或标记整理算法。    

`内存分配与回收策略`：    
Android的`Generational Heap Memory`分代堆内存模型：    

![](/image/2018-05-12-learning-notes-android-performance-optimization/memory_mode_generation.png)

- `新生代（Young Generation）`：新创建的对象，使用小GC（PS：若new对象所占空间大于XX:PretenureSizeThreshold时，会直接放入老年代）    
- `老年代（Old Generation）`：新生代中执行小GC幸存下来的老对象，使用大GC    
- `永久代（Permanent Generation）`：包含应用的类／方法信息、JRE库的类／方法信息。    

小GC执行很频繁而且速度快，大GC一般会比小GC慢10倍以上。    
大小GC都会发出`Stop the World`事件，中断程序运行，直到GC完成。    

具体GC过程可以参考 [GC那些事儿--Android内存优化第一弹](https://www.jianshu.com/p/5db05db4f5ab)    
同时参考：深入《理解Java虚拟机》的 内存分配与回收策略 这一小节。    

工具：LeakCanary查看内存泄漏；Android Memory Monitor／MAT／命令行 查看内存使用情况    

#### （4）网络优化    

网络请求可能造成的影响：流量、电量（受网络连接的Radio影响）、用户体验。    

（1）减少网络请求的次数。在API设计上，注册接口隐藏登录功能；或提供合并几个小接口的数据的接口。    
（2）减少数据包大小。gzip压缩、protocol buffer代替JSON。    
（3）加入缓存机制。    
（4）通过监听设备的不同状态按需加载，如是否充电／是否弱网。比如Splash闪屏图片可以在WiFi下下载，新闻类App可以在WIFI或充电状态下离线缓存。    

工具：Android Studio内置的`Network Monitor`。

#### （5）电量优化    

主要的耗电因素：网络请求、WakeLock、GPS等。

网络请求：因为手机是通过内置的射频模块Radio和基站链接从而上网的，这个射频模块非常耗电。这就是为啥飞行模式耗电很少的原因。     
WakeLock：WakeLock可以用来保持CPU长期运行, 或是防止屏幕变暗/关闭，如果没有及时释放的话会非常耗电。如视频播放时android:keepScreenOn=true    
GPS：定位模块。    

#### （6）包大小优化      

参考：[Android APP终极瘦身指南](http://jayfeng.com/2016/03/01/Android-APP%E7%BB%88%E6%9E%81%E7%98%A6%E8%BA%AB%E6%8C%87%E5%8D%97/)    
[Android App包瘦身优化实践](https://tech.meituan.com/android-shrink-overall-solution.html)    

Res资源层面：    
（1）使用同一套资源文件，如一般使用xhdpi即可。    
（2）使用tinypng有损压缩    
（3）使用webp图片格式。带透明度，仅限于4.0+以上设备，webp > jpg > png。    
（4）gradle中启用minifyEnabled开启代码混淆、shrinkResources去除无用资源。    
（5）使用第三方资源压缩工具。如微信提供的7zip    
（6）<shape>标签代替图片    

库文件层面：    
（1）保留armable和armable-x86，删除armable-v7包下的so（仅限于对图形渲染要求不高的情况）    
（2）避免重复库    
（3）使用更小的库    

其他：    
插件化／业务精简    

#### （7）ANR    

ANR，Application Not Responding。

`产生原因`：    
`5s`内无法响应用户输入事件(例如键盘输入, 触摸屏幕等)；BroadcastReceiver在`10s`内无法结束    

`如何避免`：    
主线程不要执行耗时任务，如CPU计算、IO操作放到子线程。    
Activity/Service/Broadcast Receiver／设置MainLoop的Handler 都是运行在主线程中的。    
子线程有Thread／AsyncTask／HandlerThread／IntentService／Loader等


### 2、参考文档    

（1）[http://hukai.me/blog/archives/](http://hukai.me/blog/archives/)    
（2）[https://www.jianshu.com/p/f7006ab64da7](https://www.jianshu.com/p/f7006ab64da7)

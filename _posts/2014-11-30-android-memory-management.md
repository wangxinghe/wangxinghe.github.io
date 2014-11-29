---
layout: post
title: "Android内存管理"
description: "Android内存管理"
category: android
tags: [Android]
---

前一段时间自己学习了下Android内存管理相关的东西，在部门内部做了一次技术分享。作为一个在公开场合会腼腆，上不了台面，表达能力也不行的人来说是一个不小的进步。一直很佩服墙内墙外的牛人们坚持写博客和大家分享的精神。嗯，今天就将上次的技术分享写在博客里面，希望对大家有帮助。

下面分为4个部分来阐述： 
 
-	Android内存管理机制  
-	Android App中内存管理优化  
-	检测内存使用情况  
-	内存泄漏  


###一、Android系统内存管理机制

1.Linux vs. Windows  
两者都能将物理内存中长时间不使用的内容转移到磁盘空间上，再次访问时才加载回内存。  
Windows只在应用程序需要的时候才会分配内存，不能充分充分利用内存。  
Linux充分利用物理内存空间（RAM）及其高速读写特性，将程序调用过程中的硬盘数据读入内存，提高数据访问性能。  

Android是基于Linux系统的，继承了Linux的很多优秀特性。但是Android没有可交换磁盘空间（swap space）,所有的内存都存在于RAM中，这使得Android系统释放内存的唯一方式就是释放引用的对象。

2.Android进程和内存

Android进程包含2种：  
（1）Native进程：采用C/C++实现，不包含dalvik实例的进程。/system/bin/目录下面的文件运行后都是以native进程存在的。  
（2）Java进程：Android中运行于dalvik虚拟机上的进程。dalvik虚拟机的宿主进程由fork()系统调用创建，所以每一个Java进程都存在于一个Native进程中。由于每个Java存在一个虚拟机实例，因此Java进程内存分配要比Native进程复杂。  

3.Android中的java程序为啥容易OOM？  
Android对dalvik的vm heapsize作了限制，当Java进程申请的内存超过阈值时，会抛出OOM异常。  
程序发生OOM并不能说明RAM不足，只能说明是Java heap超出了dalvik vm heapsize的阈值。  
当RAM内存不足时，memory killer会杀掉低优先级进程，保证高优先级进程有更多内存。  

Google之所以这样设计，处于以下考虑：  
为了让比较多的进程同时常驻内存，这样程序启动时不需要每次都加载到内存，能够给用户更快的响应。通过限制每个应用程序的内存，使得Android系统内存中同时常驻多个进程。

4.如何绕过dalvik vm heapsize的限制  
（1）创建子进程  
（2）使用JNI在native heap上申请内存（推荐）  
（3）使用RAM中的显存空间

###二、Android App中内存管理优化

1.Use services sparingly  

- 尽量少用`Service`，当后台任务运行完成时及时关闭`Service`  
- 用`IntentService`取代`Service`，当后台任务完成时自动结束服务自身

原因：Service启动时，Service所在进程会保持运行状态，Service所占用的内存不会释放，使得进程占用资源过多，因此系统LRU cache中能同时保持的进程数减少，应用程序的切换变得更低效。由于没有足够的进程来处理系统中的Service，也会导致系统稳定性变差。

2.当UI不可见或内存紧张时，释放内存：  
在`Activity`的回调方法`onTrimMemory(int level)`中根据level的不同释放内存。  
进程不在缓存中：

- `TRIM_MEMORY_RUNNING_MODERATE` 应用程序正在运行，并且处于非killable状态，此时设备内存低（low），系统主动杀LRU缓存中的进程
- `TRIM_MEMORY_RUNNING_LOW` 应用程序正在运行，并且处于非killable状态，此时设备内存很低（much lower），需要释放没用的资源
- `TRIM_MEMORY_RUNNING_CRITICAL` 应用程序正在运行，但是系统已杀死LRU缓存中的大部分进程，此时需要释放所有不至关重要的资源。如果系统不能回收足够的内存，就会清掉LRU中所有的进程以及服务进程。  

进程在LRU缓存中：

- `TRIM_MEMORY_BACKGROUND` 系统低内存下运行，程序进程位于LRU缓存列表的开头位置。虽然程序进程被kill的概率不大，但是系统可能正在杀LRU中的进程。你需要释放容易恢复的资源以便程序进程还在LRU list中，当从其他App返回时，能快速恢复现场。
- `TRIM_MEMORY_MODERATE` 系统低内存下运行，程序进程位于LRU缓存列表的中间位置。你的进程被杀掉的可能性变大。
- `TRIM_MEMORY_COMPLETE` 系统低内存下运行，程序进程最先容易被系统杀死。你需要释放所有对于恢复程序状态不至关重要的资源。  

API14开始有`onTrimMemory()`回调；API 14以下使用的是`onLowMemory()`，等价于`TRIM_MEMORY_COMPLETE`

3.恰当使用Bitmap  
加载bitmap时，尽量保证bitmap分辨率和屏幕分辨率匹配，对于大分辨率的bitmap需要进行压缩处理。
![](/image/2014-11-30-android-memory-management/bitmap_2.3.x.png) 
![](/image/2014-11-30-android-memory-management/bitmap_3.0.png)  
（1） Android 2.3.x(API 10)及以下的系统  
bitmap像素数据实际存储于native内存中，在java heap中只是保留对象的引用，因此在java heap中内部都显示同一个大小。内存回收需主动调用recycle()，GC失效  
（2） 在Android 3.0(API 11)及以上的系统  
bitmap像素数据存储于java heap中，无需主动调用recycle()，由GC管理内存。  
（3） 通过内存分析工具调试bitmap内存时，在Android 3.0的系统上进行，因为大部分内存分析工具只能分析java内存

4.使用`SparseArray`，`SparseBooleanArray`和`LongSparseArray`等优化的数据容器代替HashMap  

5.使用static const代替`enum`

6.非必要情况下，少用抽象

7.对于序列化数据，使用nano protobuf

8.尽量少使用依赖注入框架

9.谨慎使用第三方库

10.使用ProGuard去除不必要的代码

11.apk打包签名时，使用zipalign工具对齐

12.使用多进程  
一般情况下，大多数应用程序都不需要使用多进程。只有对于需要在后台和前台同时运行，并且前后台单独管理的程序才涉及到多进程（如音乐播放器）。  
使用多进程方法：

	<service android:name=".PlaybackService"
         android:process=":background" />

一个空进程占用大概1.4MB内存，如果在该进程中操作UI，内存占用将是原来的好几倍。因此如果使用多进程，一般一个进程用于UI，其他进程不要有任何UI相关操作。

###三、检测内存使用情况

1.分析Logcat信息

	D/dalvikvm:<GC_Reason><Amount_freed>,<Heap_stats>,<External_memory_stats>,<Pause_time>
参数解释：

- GC_Reason  发生GC的原因（GC_CONCURRENT,GC_FOR_MALLOC,GC_HPROF_DUMP_HEAP,GC_EXPLICIT,GC_EXTERNAL_ALLOC）
- Amount_freed 垃圾回收释放的内存
- Heap_stats java堆内存可用百分比(number of live objects)/(total heap size)
- External_memory_stats  (amount of allocated memory)/(limit at which collection will occur)
- Pausing_time 取决于heap大小，对于并发操作可能会有2个pausing time

2.利用Eclipse的DDMS视图自带的内存分析工具

- 观察heap内存变化情况  
 DDMS -> Update heap -> Cause GC
- 跟踪内存分配情况  
DDMS -> Allocation Tracker -> Start Tracking -> GetAllocations  
跟踪内存分配的方法，可用定位到具体某个实例，某个线程，某个类，某个文件和某一行。

3.使用adb命令行  

	adb shell dumpsys meminfo <package_name>

![](/image/2014-11-30-android-memory-management/memory_usage_info.png)  

4.使用MAT内存分析工具  
下载eclipse插件memory analyze tool  
操作DDMS -> Dump HPROF file -> save,分析Histigram view和Dominator tree内容。

![](/image/2014-11-30-android-memory-management/shallow_heap_and_retained_heap.png)  

重要概念：shallow heap和retained heap

- shallow heap 对象本身占用内存大小
- retained heap 通过释放这个对象总共可以回收的内存


###四、内存泄漏

1.GC原理  
![](/image/2014-11-30-android-memory-management/gc_0.png)  
![](/image/2014-11-30-android-memory-management/gc_1.png)  

GC会选择一些还存活的对象作为内存遍历的根节点GC Roots（如thread stack中的变量，JNI中的全局变量，zygote中的class loader对象等），对heap进行遍历，没有被直接或间接遍历到的引用会被GC回收，能被遍历到的不能被回收。  
内存泄露：某些不再使用的对象被GC Roots引用，导致不能回收，使实际可使用内存变小。

2.引起内存泄露的因素  
（1）长时间保持对`Activity`,`Context`,`View`,`Drawable`和其他对象的引用  
（2）非静态内部类  
（3）持有对象的时间超出需要的时间

3.常见的内存泄露  
（1）非静态内部类的静态实例  
（2）`Activity`使用静态成员  
（3）`Handler`、`HandlerThread`使用时的问题  
（4）register某个对象后缺少对应的unregister操作  
（5）集合对象未清理，资源对象未关闭  
（6）Dialog或PopupWindow未关闭引起的window leak  
（7）不良代码造成的压力。如Bitmap使用不当；构造adapter时，没有使用缓存的convertView;在循环方法中创建对象。

4.改进建议  

- 与View无关的操作尽量使用Application Context
- 使用静态内部类
- 灵活使用弱引用`WeakReference`
- 对对象的持有时间不要超过其生命周期

###参考文档

1.[https://developer.android.com/tools/debugging/debugging-memory.html#TriggerLeaks][triggerLeaks]  
2.[https://developer.android.com/training/articles/memory.html][memory]  
3.[http://csdn.net/gemmem/article/details/8920039/][memory ref1]  
4.[http://blog.csdn.net/gemmem/article/details/13017999/][memory ref2]  
5.Google IO 2011: Memory management for Android Apps "Google IO 2011"

[triggerLeaks]: https://developer.android.com/tools/debugging/debugging-memory.html#TriggerLeaks "TriggerLeaks"
[memory]: https://developer.android.com/training/articles/memory.html "Memory"
[memory ref1]: http://csdn.net/gemmem/article/details/8920039/ "Memory Ref1"
[memory ref2]: http://blog.csdn.net/gemmem/article/details/13017999/ "Memory Ref2"
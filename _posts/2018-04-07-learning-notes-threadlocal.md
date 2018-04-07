---
layout: post
comments: true
title: "Java学习笔记——ThreadLocal"
description: "Java学习笔记——ThreadLocal"
category: Java
tags: [Java]
---


**1、ThreadLocal设计图**    
**2、应用场景分析**    
**3、关键类ThreadLocalMap**    
**（1）数据结构**    
**（2）怎么存/取数据**    
**（3）内存回收和扩容问题**    
**4、参考文档**

<!--more-->


### 1、ThreadLocal设计图

![](/image/2018-04-07-learning-notes-threadlocal/threadlocal.jpg)

设计图分析：    

一个Thread维护一个ThreadLocalMap的实例，ThreadLocalMap内部包含一个Entry[]数组，而Entry是一个key为WeakReference<ThreadLocal>，value为Object的数据对，或者说包含一个key到ThreadLocal对象的弱引用，value到Object的强引用。当GC发生时，若ThreadLocalRef到ThreadLocal实例的引用也断开，此时由于key到ThreadLocal实例为弱引用，则ThreadLocal实例会被释放。

是否会发生内存泄漏，其核心是是否有GC Roots开始的引用链，对于ThreadLocal要考虑的问题：    
若往Entry[]数组中，put的数据越来越多，会导致ThreadLocal实例和Object value越来越多，假设ThreadLocalRef到ThreadLocal实例的引用又不断开的话，Thread也不结束的话，就会导致ThreadLocal实例和Object value不能释放，从而可能引发内存溢出或内存泄漏。(活跃的Thread可以作为GC Roots，如果Thread结束，则从Thread中所有对象都会释放。

解决思路：    
（1）Thread活跃期间，定期查找key引用断开的废弃Entry，断开这个Entry到value的引用，等GC回收    
（2）断开ThreadLocalRef到ThreadLocal实例的引用

不过很多用ThreadLocal的场景，每个线程的Entry[]只存了一个数据，且ThreadLocal定义成 static final类型，所以不容易出现内存溢出。如Android中的Looper就是一个线程对应一个Looper，JDBC数据库连接中也是一个线程对应一个连接。

### 2、应用场景分析

ThreadLocal简单说来就是：    
一个Thread维护自己的一个Map，每个Thread只拿自己Map中的数据，各个Thread互不干扰，不存在什么线程间数据共享问题。

比如Android中的Looper机制就是，主线程和各子线程都有自己独立的Looper实例。    
比如Hibernate的JDBC数据库连接，每个线程有自己独立的Connection，互不干扰。    

ThreadLocal和synchronized的比较：    
都和多线程有关。区别是，ThreadLocal里各线程间的数据是没啥关联，互不干扰的。而synchronized里是数据共享的数据，同一时刻只能一个线程访问。而且ThreadLocal里需要维护一个数组，synchronized不需要维护数组。所以ThreadLocal略占内存但效率高，synchronized不占内存但效率低。也就是网上有人总结的`空间换时间`和`时间换空间`的差别。

### 3、关键类ThreadLocalMap

#### （1）数据结构

**ThreadLocalMap三要素：**    
Entry[] table;  //数组    
int size;  //元素个数    
int threshold;  //阈值，2/3数组长度，当size >= 3/4 * threshold时需要对数组扩容    

Entry是<WeakReference<ThreadLocal>, Object>的键值对。    
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
#### （2）怎么存/取数据

数据存取，主要是关注set/get方法。

**取数据get()**    

1、获取当前线程    
2、获取当前线程的ThreadLocalMap对象    
3、当前ThreadLocal实例作为key，从ThreadLocalMap中查找对应的value，并返回结果。     
   具体查找过程：    
 （1）先计算索引，int i = key.threadLocalHashCode & (table.length - 1);    
 （2）如果table[i]元素非空且table[i].key == key，说明找到了，返回table[i].value    
 （3）否则，从i开始遍历一轮每个元素e（即从包括i开始往后直到第一个空元素，算作1轮，i有可能就是第一个空元素）。    
   （3.1）如果e.key == key，则说明`找到`并返回e.value；    
   （3.2）如果e.key == null，则说明是`垃圾数据`，调用`expungeStaleEntry(i)`进行垃圾数据逻辑处理即可    
   （3.3）如果e.key != null && e.key != key，则说明存在`hash冲突`，这个位置被别的对象先占了，根据`线性探测法`往后寻找，循环上面步骤。        
4、如果没有查到，则调用初始化方法并返回初始值。

`expungeStaleEntry(int staleSlot)`为垃圾数据处理逻辑：    
`简单来说干了两件事：跑1轮，清空里面的垃圾数据，非垃圾数据重新hash。`    
（1）staleSlot位置的数据清空，size-1    
（2）从staleSlot+1开始遍历一轮每个元素e，位置为i（即从staleSlot+1开始往后直到第一个空元素，算作1轮）    
    （2.1）如果e.key == null，则说明是`垃圾数据`，将e清空，size-1    
    （2.2）如果e.key != null 但  [h = e.key.threadLocalHashCode & (len - 1)] != i，说明元素e本来不应该放在位置i，当时只是因为发生`hash冲突`h位置被别人占了，将第i个位置置空，然后根据`线性探测法`重新放置元素e的位置，即找到从h开始的第一个空元素位置放进去即可。


**存数据set()**    

    
#### （3）内存回收和扩容问题    



### 4、参考文档

（1）[深入JDK源码之ThreadLocal类](https://my.oschina.net/xianggao/blog/392440)    
（2）[正确理解ThreadLocal](http://lujh99.iteye.com/blog/103804)    





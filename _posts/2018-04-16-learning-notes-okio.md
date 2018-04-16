---
layout: post
comments: true
title: "Android学习笔记——Okio"
description: "Android学习笔记——Okio"
category: Android
tags: [Android]
---

**1、OKio框架**    
**2、OKio的设计思想**    
**（1）Sink / Source**    
**（2）Buffer机制--Segment和SegmentPool**    
**（3）超时机制--Timeout和AsyncTimeout**    
**3、参考文档**

<!--more-->

### 1、OKio框架

![](/image/2018-04-16-learning-notes-okio/okio_framework.png)

![](/image/2018-04-16-learning-notes-okio/java_io_framework.png)

上面两张图是OKio和Java IO的类图。我们可以很直观的看出两者之间的异同。
    
**相同点：**    
本质一样，都是对流的操作，Source类似于InputStream／Reader，Sink类似于OutputStream／Writer。    
除了基本的read/write外，还有支持特定需求的各种子类。

**不同点：**    
（1）OKio精简了输入输出流的类个数    
（2）OKio的Buffer机制更加优秀，引入Segment和SegmentPool复用机制    
（3）OKio支持Timeout超时机制    
（4）OKio支持md5、sha、base64等数据处理    

### 2、OKio的设计思想    

下面重点讲下OKio框架设计中关键的几个部分。

#### （1）Source / Sink   

`Source`的意思是`水源`，`Sink`的意思是**水槽**，非常形象的表明了流的输入输出。    
其中`Source`对应输入流，`Sink`对应输出流。

以`Source`为例，其代码如下：

    public interface Source extends Closeable {
      /**
       * Removes at least 1, and up to {@code byteCount} bytes from this and appends
       * them to {@code sink}. Returns the number of bytes read, or -1 if this
       * source is exhausted.
       */
      long read(Buffer sink, long byteCount) throws IOException;

      /** Returns the timeout for this source. */
      Timeout timeout();

      /**
       * Closes this source and releases the resources held by this source. It is an
       * error to read a closed source. It is safe to close a source more than once.
       */
      @Override void close() throws IOException;
    }

继承关系如图：
 
 ![](/image/2018-04-16-learning-notes-okio/source.png)
 
`Source`和`BufferedSource`都是接口，真正的实现类是`RealBufferedSource`，真正的read操作还是通过`Buffer`来执行，Buffer类似于BufferedInputStream/BufferedReader里的byte[]/char[]缓冲数组，不过Buffer里面是一个链表的数据结构。

`Sink`结构和Source类似。

#### （2）Buffer机制--Segment和SegmentPool    



#### （3）超时机制--Timeout和AsyncTimeout    



### 3、参考文档

（1）[深入理解okio的优化思想](https://blog.csdn.net/zoudifei/article/details/51232711)    


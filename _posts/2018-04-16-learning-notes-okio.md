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
 
`Source`和`BufferedSource`都是接口，BufferedSource在Source基础上扩充了一些方法，实现类是`RealBufferedSource`，真正的read操作还是通过`Buffer`来执行，Buffer类似于BufferedInputStream/BufferedReader，只不过传统的Java IO里缓存区是byte[]/char[]缓冲数组，而Buffer里面是一个链表的数据结构。

`Sink`结构和Source类似。

#### （2）Buffer机制--Segment和SegmentPool    



#### （3）超时机制--Timeout和AsyncTimeout    

在传统的Java IO类库中是没有超时检测的，如果我们要实现这样一个机制，是需要再封装一层的。

OKio框架中提供了`Timeout`和`AsyncTimeout`两个类，用于实现超时机制。Timeout是同步，AsyncTimeout是异步。在OKHttp框架里面，Socket流使用的是AsyncTimeout，其他流使用的是Timeout。

重点分析AsyncTimeout，先看这个类的基本结构：

    public class AsyncTimeout extends Timeout {
      //单链表，static全局变量
      static AsyncTimeout head;
      //当前结点是否在单链表中
      private boolean inQueue;
      //单链表后继结点
      private @Nullable AsyncTimeout next;
    
      //流操作开始时调用
      public final void enter()｛...｝
      //流操作结束时调用
      public final boolean exit() {...}
      //流操作超时时调用
      protected void timedOut() {...}
      
      //监听单链表中结点对应的流是否超时的WatchDog线程
      private static final class Watchdog extends Thread {
        Watchdog() {
          super("Okio Watchdog");
          setDaemon(true);
        }

        public void run() {
          while (true) {
            try {
              AsyncTimeout timedOut;
              synchronized (AsyncTimeout.class) {
                timedOut = awaitTimeout();

                // Didn't find a node to interrupt. Try again.
                if (timedOut == null) continue;

                // The queue is completely empty. Let this thread exit and let another watchdog thread
                // get created on the next call to scheduleTimeout().
                if (timedOut == head) {
                  head = null;
                  return;
                }
              }
    
              //超时回调
              timedOut.timedOut();
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    ｝

总结下AsyncTimeout的特点：    
（1）AsyncTimeout是一个单链表，结点按等待时间从小到大排序，head是一个头结点，起占位作用。    
（2）提供了3个方法enter(), exit(), timeout()，分别用于流操作开始、结束、超时三种情况调用    
（3）有一个WatchDog的守护线程，在while(true)死循环里不停检测单链表里第一个结点对应的流是否超时，若超时进行回调    

再去细看源码的话，我们会知道：    
（1）enter()将当前流对应的AsyncTimeout结点根据等待时机顺序插入到单链表中    
（2）exit()将当前流对应的AsyncTimeout结点从单链表中移除    

根据下面使用AsyncTimeout的地方，我们知道，一个AsyncTimeout结点专门检测一个Socket流是否超时，在这个流的read/write操作的开始调用enter()，结束调用exit()。在OKhttp里面，timeout()回调方法中会关闭流。

关于设计成单链表的个人思考：    
（1）看Handler源码的时候，我们会发现MessageQueue就是一个按发送时间排序的Message单链表，和AsyncTimeout设计思想有异曲同工之妙。    
（2）在我们之前看过的很多代码里，比如有一系列对象，需要监听每个对象的状态，我们一般会考虑使用HashMap<Subject, Listener>来实现，和这里使用单链表的思路和以前是不同的。    
（3）再延伸一下，如果遇到涉及列表对象，且有时间先后顺序的场景，可以考虑下使用单链表的数据结构


附上AsyncTime调用的地方：


      #<Okio.java>
      public static Source source(Socket socket) throws IOException {
        if (socket == null) throw new IllegalArgumentException("socket == null");
        if (socket.getInputStream() == null) throw new IOException("socket's input stream == null");
        AsyncTimeout timeout = timeout(socket);
        Source source = source(socket.getInputStream(), timeout);
        return timeout.source(source);
      }

      #<AsyncTimeout.java>
      public final Source source(final Source source) {
        return new Source() {
          @Override public long read(Buffer sink, long byteCount) throws IOException {
            boolean throwOnTimeout = false;
            enter();
            try {
              long result = source.read(sink, byteCount);
              throwOnTimeout = true;
              return result;
            } catch (IOException e) {
              throw exit(e);
            } finally {
              exit(throwOnTimeout);
            }
          }

          @Override public void close() throws IOException {
            boolean throwOnTimeout = false;
            try {
              source.close();
              throwOnTimeout = true;
            } catch (IOException e) {
              throw exit(e);
            } finally {
              exit(throwOnTimeout);
            }
          }

          @Override public Timeout timeout() {
            return AsyncTimeout.this;
          }

          @Override public String toString() {
            return "AsyncTimeout.source(" + source + ")";
          }
        };
      }
      
从上面可以知道，一个Socket流设置一个AsyncTimeout，在read/write操作的开始调用enter()，结束调用exit()，很显然说明一个AsyncTimeout结点专门检测一个Socket流是否超时。



### 3、参考文档

（1）[深入理解okio的优化思想](https://blog.csdn.net/zoudifei/article/details/51232711)    


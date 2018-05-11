---
layout: post
comments: true
title: "Java学习笔记——Netty"
description: "Java学习笔记——Netty"
category: Java
tags: [Java]
---


**1、整体框架**    
**（1）Server工作原理**    
**（2）Client工作原理**    
**2、基本流程**    
**（1）Server绑定端口**    
**（2）Client建立连接**    
**（3）Client/Server读数据**    
**（4）Client/Server写数据**    
**3、重要元素**    
**（1）NioSocketChannel/NioServerSocketChannel**    
**（2）NioEventLoopGroup**    
**（3）NioEventLoop**    
**（4）Pipeline**    
**（5）ChannelHandlerContext**    
**（6）ByteBuf**    
**4、总结**    
**（1）链式结构**    
**（2）缓存结构**    
**5、参考文档**    

<!--more-->


### 1、整体框架    

#### （1）Server工作原理    

![](/image/2018-05-08-learning-notes-netty/netty-server.svg)

#### （2）Client工作原理    

![](/image/2018-05-08-learning-notes-netty/netty-client.svg)

### 2、基本流程    

整个流程无非就4步：    
（1）Server绑定端口以及监听新连接    
（2）Client发起建立连接    
（3）连接成功后，C/S读数据    
（4）连接成功后，C/S写数据    

#### （1）Server绑定端口    

![](/image/2018-05-08-learning-notes-netty/netty-bind.svg)

#### （2）Client建立连接    

![](/image/2018-05-08-learning-notes-netty/netty-connect.svg)

#### （3）Client/Server读数据

![](/image/2018-05-08-learning-notes-netty/netty-read.svg)
    
#### （4）Client/Server写数据    

![](/image/2018-05-08-learning-notes-netty/netty-write.svg)

### 3、重要元素    

#### （1）NioSocketChannel/NioServerSocketChannel

Channel可以理解成java中的Stream，既可以输入又可以输出，可以是设置成阻塞和非阻塞。

NioSocketChannel是Client端的流，NioServerSocketChannel是Server端的流。

一个NioSocketChannel对应一个连接，一个NioSocketChannel／NioServerSocketChannel对应一个Pipeline。    

#### （2）NioEventLoopGroup

NioEventLoopGroup相当于是一个线程池，维护一个NioEventLoop[]数组。

NioEventLoopGroup可以设置线程个数，线程默认个数是NettyRuntime.availableProcessors() * 2。

对于Server来说，parent Group一般只有1个线程，child Group有NettyRuntime.availableProcessors() * 2个线程。    
对于Client来说，一般也是一个线程。    

每次有新Channel注册时，NioEventLoopGroup会按顺序选择一个NioEventLoop来处理这个Channel。对于服务端，NioServerSocketChannel是扔给parent Group 里唯一的线程处理的，而新接入的NioSocketChannel则是由child Group按顺序选择的NioEventLoop来处理。客户端类似。

#### （3）NioEventLoop

NioEventLoop相当于一个线程，一个NioEventLoop包含一个Selector。    

NioEventLoop是用来执行任务的。主要包括3种类型的任务：定时任务、普通任务、IO任务。    

在需要执行任务的时候，会启动NioEventLoop线程，NioEventLoop的run()实现如下：

    // NioEventLoop
    protected void run() {
        while(true) {
            switch(hasTasks() ? selector.selectNow() : -1) {
                case CONTINUE: //-2
                    continue;
                case SELECT: //-1
                    select(wakenUp.getAndSet(false));
                    if(this.wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
                    processSelectedKeys();
                    runAllTasks();
                    break;
            }
        }
    }

1.select():    
轮询是否有`到点的定时任务`或`普通任务`或`IO任务`可以执行。如果有任一种任务可执行就会跳出循环。如果任何任务都没有，则会调用selector.select(timeoutMillis)等待可执行的IO任务。

定时任务：PriorityQueue scheduledTaskQueue，按timeout时间排序    
普通任务：LinkedBlockingQueue taskQueue    
IO任务：SelectionKey

2.processSelectedKeys():    
处理IO任务。    

    // NioEventLoop
    public void processSelectedKeys() {
        for(int i = 0; i < selectedKeys.size; ++i) {
            SelectionKey k = selectedKeys.keys[i];
            AbstractNioChannel ch = k.attachment();
            NioUnsafe unsafe = ch.unsafe();
            
            //处理连接事件
            int readyOps = k.readyOps();
            if ((readyOps & OP_CONNECT) != 0) {
                unsafe.finishConnect();
            }
            
            //处理写事件
            if((readyOps & SelectionKey.OP_WRITE) != 0) {
                unsafe.forceFlush();
            }

            //处理读事件或处理NioServerSocketChannel的ACCEPT事件
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        }
    }

3.runAllTasks():    
处理到点的定时任务和普通任务。    
对于到点的定时任务，会添加到普通任务队列，并执行普通任务队列的任务。

IO任务执行时间比例：ioRatio ＝ 50    
其他任务执行时间比例：100 - ioRatio     
如果ioRatio设置为0的话，则是顺序执行，先执行完IO任务再执行完其他任务。    

#### （4）Pipeline

Pipeline相当于一个管道，维护一个ChannelHandlerContext的双向链表，头结点是HeadContext，尾结点是TailContext。

HeadContext，既是ChannelInBoundHandler也是ChannelOutBoundHandler，是outBound类型。    
TailContext，是ChannelInBoundHandler，是inBound类型。

当进行读操作，即数据从外部读到自己这边的Channel时，数据会先传给Pipeline，然后从HeadContext开始依次传给inBound类型的ChannelInBoundHandler，直到TailContext结束。    
当进行写操作，即数据从自己这边的Channel往外部写时，数据会先传给Pipeline，然后从TailContext开始依次传给outBound类型的ChannelOutBoundHandler，直到HeadContext结束。    

异常事件是从HeadContext往TailContext传递。

数据传递过程是一个`链式传递`，因为之前稍微看过OkHttp源码，有点类似。

#### （5）ChannelHandlerContext

unsafe：    
NioServerSocketChannel对应NioMessageUnsafe
NioSocketChannel对应NioByteUnsafe

HeadContext的工作：    
1、通过unsafe处理bind/connect/read/write/flush/deregister/disconnect/close等具体事件    
2、链式传递exceptionCaught/channelRegistered/channelUnregistered/channelActive/channelInactive/channelRead/channelReadComplete等事件。    

TailContext的工作：    
负责userEventTriggered/exceptionCaught/channelRead等一些异常捕获或未处理消息的日志提醒和资源释放工作。    

用户自定义的ChannelHandlerContext可以选择是否将消息继续往下传递，默认是传递的，如果不传递将ctx.fireChannelXXX();这行代码覆盖掉即可。比如ServerBootstrapAcceptor在channelRead的时候就没有继续往下传。

传递的参数可以是Channel，比如在Server端监听到OP_ACCEPT事件时，ServerSocketChannel.accept()会得到一个NioSocketChannel，这个连接会放到list，然后经由ServerSocketChannel对应的Pipeline的channelRead递归传给ServerBootstrapAcceptor，NioSocketChannel进行一些初始化处理后再交由child Group进行下一步操作。

#### （6）ByteBuf    

ByteBuf是数据缓存。

### 4、总结    

#### （1）链式结构    

OkHttp网络框架和这个类似，可能有相互借鉴。    

#### （2）缓存结构    

可以对比下okio、nio中的Buffer

### 5、参考文档    

（1）[https://www.jianshu.com/u/4fdc8c2315e8](https://www.jianshu.com/u/4fdc8c2315e8)    
（2）[https://www.jianshu.com/p/03bb8a945b37](https://www.jianshu.com/p/03bb8a945b37)    
（3）[https://blog.csdn.net/zxhoo/article/category/1800249](https://blog.csdn.net/zxhoo/article/category/1800249)    

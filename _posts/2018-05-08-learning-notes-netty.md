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
    
#### （2）NioEventLoopGroup
    
#### （3）NioEventLoop
    
#### （4）Pipeline
    
#### （5）ChannelHandlerContext
    
#### （6）ByteBuf    


NioEventLoop里面有一个线程thread，这个线程在NioEventLoop创建的时候创建，在需要执行任务的时候才启动；还有一个任务队列LinkedBlockingQueue<Runnable> taskQueue
startThread()是启动这个thread，在run()

    // NioEventLoop
    while(true) {
        switch() {
            case CONTINUE:
            
            case SELECT:
            select(this.wakenUp.getAndSet(false));
            if(this.wakenUp.get()) {
                this.selector.wakeup();
            }
            default:
            this.processSelectedKeys();
            this.runAllTasks();
        }
    }

select();
做的事情为：从PriorityQueue scheduledTaskQueue优先队列中取出第一个Task，得到对应的deadline
然后在while(true)中：
（1）如果scheduledTaskQueue取的task的timeoutMillis到了，则break，同时调一下selector.selectNow();
（2）如果LinkedBlockingQueue taskQueue非空，则break，同时调一下selector.selectNow();
（3）执行阻塞操作 int selectedKeys = selector.select(timeoutMillis);
（4）阻塞操作接受后，如果发现selectedKeys或scheduledTaskQueue或taskQueue不为空则break

总结来说，就是有3种类型的任务：定时任务scheduledTaskQueue、普通任务taskQueue、IO任务selectedKeys

select干的事情就是：从scheduledTaskQueue中拿一个任务出来，在一个while(true)循环中：如果检测到这个任务timeout时间到了则调一下selector.selectNow()然后立即跳出循环；如果有普通任务taskQueue非空，则调一下selector.selectNow()然后立即跳出循环；如果要执行的延时任务和普通任务都没有，则在这段timeout时间内执行IO阻塞操作selector.select(timeoutMillis)，完了后判断是否有要执行的延时任务和普通任务和selectKeys，如果有任何一个则break，否则的话还在这个 while(true)中走

轮询到可执行的操作后，接下来就是处理了，处理的顺序是：先处理IO，再处理Task


processSelectedKeys();执行比例ioRatio默认50（总数100）

    public void processSelectedKeys() {
        for(int i = 0; i < selectedKeys.size; ++i) {
            SelectionKey k = selectedKeys.keys[i];
            AbstractNioChannel ch = k.attachment();
            NioUnsafe unsafe = ch.unsafe();
            
            int readyOps = k.readyOps();
            if ((readyOps & OP_CONNECT) != 0) {
                unsafe.finishConnect();
            }
            
            if((readyOps & SelectionKey.OP_WRITE) != 0) {
                unsafe.forceFlush();
            }

            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        }
    }

对于服务端来说，上面的channel是NioServerSocketChannel，
unsafe.read();做的事情是

    public void read() {
        List<Object> buf;
        
        do {
            SocketChannel ch = SocketUtils.accept(this.javaChannel());
            buf.add(new NioSocketChannel(this, ch));
        } while(readBuf.size() < maxMessagesPerRead);
        
        for(int i = 0; i < readBuf.size(); ++i) {
            pipeline.fireChannelRead(readBuf.get(i));
        }
        
        pipeline.fireChannelReadComplete();
    }



pipeline.fireChannelRead(readBuf.get(i));
执行的操作：先经过HeadContext -> unsafe -> handler -> ServerBootstrapAcceptor -> ...ChannelInBoundHandler... -> TailContext

pipeline.fireChannelRead() -> head.invokeChannelRead(msg) -> head.channelRead() -> ctx.fireChannelRead(msg);递归往后传，前提是inboundHandler

pipeline.fireChannelReadComplete() -> head.invokeChannelReadComplete() -> head.channelReadComplete -> ctx.fireChannelReadComplete()递归往后传,前提是inboundHandler。传完后调this.readIfIsAutoRead();

readIfIsAutoRead()：
pipeline.read() -> tail.read() -> 递归到head.read()，前提是findContextOutbound。然后head.read(）又走unsafe.beginRead()，设置一些selectionKey.interestOps(interestOps | this.readInterestOp);


runAllTasks();//执行比例100-ioRatio
做的事情：在do while()循环里，每次从scheduledTaskQueue里取出timeout时间到点的任务放到taskQueue里，然后执行taskQueue里的所有任务。循环跳出的条件是scheduledTaskQueue里没有到点的任务为止。

关于unsafe：
NioServerSocketChannel对应NioMessageUnsafe
NioSocketChannel对应NioByteUnsafe

Pipeline的读和写（DefaultChannelPipeline）：


#### 异常处理

调用链`当前发生异常的ChannelHandlerContext.exceptionCaught() -> ctx.fireExceptionCaught(cause) -> ...ChannelHandlerContext(不区分in和out)... -> TailContext.exceptionCaught()`

    //TailContext
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        DefaultChannelPipeline.this.onUnhandledInboundException(cause);
    }

打log并释放Throwable对象


### 4、总结    

#### （1）链式结构    

okhttp类似、tcp/ip协议模型

#### （2）缓存结构    

okio、nio类似

### 5、参考文档    

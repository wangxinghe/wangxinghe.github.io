---
layout: post
comments: true
title: "Android消息循环机制源码分析"
description: "Android消息循环机制源码分析"
category: android
tags: [Android]
---

## 概述

搞`Android`的不懂`Handler`消息循环机制，都不好意思说自己是Android工程师。面试的时候一般也都会问这个知识点，但是我相信大多数码农肯定是没有看过相关源码的，顶多也就是网上搜搜，看看别人的文章介绍。学姐不想把那个万能的关系图拿出来讨论。

<!--more-->

学姐先从我们平时的使用方法引出这个机制，再结合源码进行分析。

我们平时使用是这样的：
        
        //1. 主线程
        Handler handler = new MyHandler();

        //2. 非主线程
        HandlerThread handlerThread = new HandlerThread("handlerThread");
        handlerThread.start();
        Handler handler = new Handler(handlerThread.getLooper());

        //发送消息
        handler.sendMessage(msg);
        
        //接收消息
        static class MyHandler extends Handler {
            //对于非主线程处理消息需要传Looper，主线程有默认的sMainLooper
            public MyHandler(Looper looper) {
                super(looper);
            }

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
            }
        }

那么为什么初始化的时候，我们执行了1或2，后面只需要sendMessage就可处理任务了呢？学姐这里以非主线程为例进行介绍，`handlerThread.start()`的时候，实际上创建了一个用于消息循环的`Looper`和消息队列`MessageQueue`，同时启动了消息循环，并将这个循环传给`Handler`，这个循环会从`MessageQueue`中依次取任务出来执行。用户若要执行某项任务，只需要调用`handler.sendMessage`即可，这里做的事情是将消息添加到`MessaeQueue`中。对于主线程也类似，只是主线程`sMainThread`和`sMainLooper`不需要我们主动去创建，程序启动的时候`Application`就创建好了，我们只需要创建`Handler`即可。

我们这里提到了几个概念：

`HandlerThread` 支持消息循环的线程

`Handler` 消息处理器

`Looper` 消息循环对象

`MessageQueue` 消息队列

`Message` 消息体

对应关系是：**一对多，即（一个）HandlerThread、Looper、MessageQueue －> （多个）Handler、Message** 

## 源码解析

### 1. Looper

（1）创建消息循环

`prepare()`用于创建`Looper`消息循环对象。`Looper`对象通过一个成员变量`ThreadLocal`进行保存。

（2）获取消息循环对象
 
 `myLooper()`用于获取当前消息循环对象。`Looper`对象从成员变量`ThreadLocal`中获取。

（3）开始消息循环

`loop()`开始消息循环。循环过程如下：

每次从消息队列`MessageQueue`中取出一个`Message`

使用`Message`对应的`Handler`处理`Message`

已处理的`Message`加到本地消息池，循环复用

循环以上步骤，若没有消息表明消息队列停止，退出循环

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
    
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
    
### 2. Handler

（1）发送消息

`Handler`支持2种消息类型，即`Runnable`和`Message`。因此发送消息提供了`post(Runnable r)`和`sendMessage(Message msg)`两个方法。从下面源码可以看出`Runnable`赋值给了`Message`的`callback`，最终也是封装成`Message`对象对象。学姐个人认为外部调用不统一使用`Message`，应该是兼容Java的线程任务，学姐认为这种思想也可以借鉴到平常开发过程中。发送的消息都会入队到`MessageQueue`队列中。

（2）处理消息

Looper循环过程的时候，是通过dispatchMessage(Message msg)对消息进行处理。处理过程：先看是否是Runnable对象，如果是则调用handleCallback(msg)进行处理，最终调到Runnable.run()方法执行线程；如果不是Runnable对象，再看外部是否传入了Callback处理机制，若有则使用外部Callback进行处理；若既不是Runnable对象也没有外部Callback，则调用handleMessage(msg)，这个也是我们开发过程中最常覆写的方法了。

（3）移除消息

removeCallbacksAndMessages()，移除消息其实也是从MessageQueue中将Message对象移除掉。

    public void handleMessage(Message msg) {
    }
        
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    
    private static void handleCallback(Message message) {
        message.callback.run();
    }
    
    public final Message obtainMessage()
    {
        return Message.obtain(this);
    }
    
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
    
    public final void removeCallbacksAndMessages(Object token) {
        mQueue.removeCallbacksAndMessages(this, token);
    }


### 3. MessageQueue

（1）消息入队

消息入队方法enqueueMessage(Message msg, long when)。其处理过程如下：

待入队的Message标记为InUse，when赋值

若消息链表mMessages为空为空，或待入队Message执行时间小于mMessage链表头，则待入队Message添加到链表头

若不符合以上条件，则轮询链表，根据when从低到高的顺序，插入链表合适位置

（2）消息轮询

next()依次从MessageQueue中取出Message

（3）移除消息

removeMessages()可以移除消息，做的事情实际上就是将消息从链表移除，同时将移除的消息添加到消息池，提供循环复用。

    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
    
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
    
    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

### 4. Message

（1）消息创建

Message.obtain()创建消息。若消息池链表sPool不为空，则从sPool中获取第一个，flags标记为UnInUse，同时从sPool中移除，sPoolSize减1；若消息池链表sPool为空，则new Message()

（2）消息释放

recycle()将消息释放，从内部实现recycleUnchecked()可知，将flags标记为InUse，其他各种状态清零，同时将Message添加到sPool，且sPoolSize加1

    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    /**
     * Return a Message instance to the global pool.
     * <p>
     * You MUST NOT touch the Message after calling this function because it has
     * effectively been freed.  It is an error to recycle a message that is currently
     * enqueued or that is in the process of being delivered to a Handler.
     * </p>
     */
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }
    
    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
    
### 5. HandlerThread

由于Java中的Thread是没有消息循环机制的，run()方法执行完，线程则结束。HandlerThread通过使用Looper实现了消息循环，只要不主动调用HandlerThread或Looper的quit()方法，循环就是一直走下去。

    public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
    
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    }
    
## 总结

1. 关键类：HandlerThread、Handler、Looper、MessageQueue、Messaga

2. MessageQueue数据结构，链表。


## 欢迎大家关注我的公众号：学姐的IT专栏

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)

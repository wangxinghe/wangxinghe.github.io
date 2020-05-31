---
layout: post
comments: true
title: "理解Choreographer"
description: "理解Choreographer"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析

<!--more-->

本文以TextView#setText为切入口, 分析Choreographer的实现机制.  

**调用链:**  
`TextView#setText -> checkForRelayout -> requestLayout -> invalidate -> invalidateInternal -> ViewGroup#invalidateChild -> ViewGroup#onDescendantInvalidated(硬件加速) -> ... -> DecorView#onDescendantInvalidated -> ViewRootImpl#onDescendantInvalidated -> ViewRootImpl#invalidate -> ViewRootImpl#scheduleTraversals`  

我们再来看`ViewRootImpl#scheduleTraversals`的实现:   
[ -> frameworks/base/core/java/android/view/ViewRootImpl.java ]

	void scheduleTraversals() {
	    if (!mTraversalScheduled) {
	        mTraversalScheduled = true;
	        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
	        mChoreographer.postCallback(
	                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
	        if (!mUnbufferedInputDispatch) {
	            scheduleConsumeBatchedInput();
	        }
	        notifyRendererOfFramePending();
	        pokeDrawLockIfNeeded();
	    }
	}

根据上面代码, 引出Choreographer:  
ViewRootImpl#scheduleTraversals -> Choreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, ...)

## 1.Choreographer
[ -> frameworks/base/core/java/android/view/Choreographer.java ]

### 1.1 Choreographer是什么?  
中文翻译是**编舞者**. 用于协调animations, input和drawing的时间.  
Choreographer通过接收显示系统的时间脉冲(如垂直同步信号), 来完成下一帧的渲染工作.  
通常App层通过高层抽象的API调用和Choreographer交互, 如ValueAnimator#start / View#postOnAnimation / View#postInvalidateOnAnimation / View#invalidate等.  

### 1.2 重要组成元素  

	// 每一帧的时间间隔(通常是16ms), 帧率为60fps
	private long mFrameIntervalNanos;
	// 帧率系数因子(通常是1)
	private int mFPSDivisor = 1;

	// Callback队列, 是一个元素为CallbackRecord的单向链表.
	private final CallbackQueue[] mCallbackQueues;

	// CALLBACK_INPUT(如Touch事件) 
	public static final int CALLBACK_INPUT = 0;
	// CALLBACK_ANIMATION&CALLBACK_INSETS_ANIMATION(如各种动画)
	public static final int CALLBACK_ANIMATION = 1;
	public static final int CALLBACK_INSETS_ANIMATION = 2;
	// CALLBACK_TRAVERSAL(如layout/draw等绘制操作)
	public static final int CALLBACK_TRAVERSAL = 3;
	// CALLBACK_COMMIT(当前帧时间延迟的上报和时间戳调整)
	public static final int CALLBACK_COMMIT = 4;

接下来分析Choreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, ...)过程.

### 1.3 postCallback

    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType, Runnable action, Object token, long delayMillis) {
        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

	private void postCallbackDelayedInternal(int callbackType, Object action, Object token, long delayMillis) {
	    synchronized (mLock) {
	        final long now = SystemClock.uptimeMillis();
	        final long dueTime = now + delayMillis;
	        // 根据dueTime时间序, 将callback添加到CallbackQueue队列
	        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

			// 根据dueTime决定是否立即执行callback
	        if (dueTime <= now) {
	            scheduleFrameLocked(now);
	        } else {
	            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
	            msg.arg1 = callbackType;
	            // 异步消息, 不受同步栅栏SyncBarrier影响
	            msg.setAsynchronous(true);
	            mHandler.sendMessageAtTime(msg, dueTime);
	        }
	    }
	}

**Choreographer#postCallback逻辑:**  
(1) 封装CallbackRecord对象.  mCallbackPool为用于循环复用的对象池.  

    private CallbackRecord obtainCallbackLocked(long dueTime, Object action, Object token) {
        CallbackRecord callback = mCallbackPool;
        if (callback == null) {
            callback = new CallbackRecord();
        } else {
            mCallbackPool = callback.next;
            callback.next = null;
        }
        callback.dueTime = dueTime;
        callback.action = action;
        callback.token = token;
        return callback;
    }

CallbackRecord数据结构为:  
- `next`: 下一个CallbackRecord元素
- `callbackType`: callback类型  
- `dueTime`: 到期时间
- `action`: 下一帧要做的事情  
- `token`: callback令牌  

(2) 执行callback操作.  
如果dueTime时间到了, 则调用scheduleFrameLocked立即执行  
如果dueTime时间没到, 则封装并向FrameHandler发送异步消息`MSG_DO_SCHEDULE_CALLBACK`.  

FrameHandler是一个Handler, 用于执行Choreographer中的操作:  

	private final class FrameHandler extends Handler {
	    public FrameHandler(Looper looper) {
	        super(looper);
	    }
 
	    @Override
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case MSG_DO_FRAME: // 执行帧操作
	                doFrame(System.nanoTime(), 0);
	                break;
	            case MSG_DO_SCHEDULE_VSYNC: // 执行vsync
	                doScheduleVsync();
	                break;
	            case MSG_DO_SCHEDULE_CALLBACK: // 执行callback
	                doScheduleCallback(msg.arg1);
	                break;
	        }
	    }
	}

执行callback操作:  

    void doScheduleCallback(int callbackType) {
        synchronized (mLock) {
            if (!mFrameScheduled) {
                final long now = SystemClock.uptimeMillis();
                // 检查dueTime到期时间
                if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                    scheduleFrameLocked(now);
                }
            }
        }
    }

不管同步还是异步, 最终都执行到scheduleFrameLocked.  

### 1.4 scheduleFrameLocked

	private void scheduleFrameLocked(long now) {
	    if (!mFrameScheduled) {
	        mFrameScheduled = true;
	        if (USE_VSYNC) {
		        // 判断当前运行的Looper是否为FrameHandler对应的Looper
	            if (isRunningOnLooperThreadLocked()) {
	                scheduleVsyncLocked();
	            } else {
	                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
	                msg.setAsynchronous(true);
	                mHandler.sendMessageAtFrontOfQueue(msg);
	            }
	        } else {
		        // sFrameDelay为10ms
	            final long nextFrameTime = Math.max(mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
	            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
	            msg.setAsynchronous(true);
	            mHandler.sendMessageAtTime(msg, nextFrameTime);
	        }
	    }
	}

**scheduleFrameLocked逻辑:**  
`USE_VSYNC`: 是否使用Vsync机制, 默认是true.  
**1.Vsync机制**  
首先确保执行的Looper是在FrameHandler对应的Looper.  
如果是则可以直接执行; 如果不是需要向FrameHandler发送类型为`MSG_DO_SCHEDULE_VSYNC`的异步消息

	void doScheduleVsync() {
	    synchronized (mLock) {
	        if (mFrameScheduled) {
	            scheduleVsyncLocked();
	        }
	    }
	}

最终都会执行到scheduleVsyncLocked

**2.非Vsync机制**  
通过消息循环机制, 定时向FrameHandler发送类型为`MSG_DO_FRAME`的异步消息. 执行doFrame从而刷新一帧数据.   

下面的过程都是分析Vsync机制.

### 1.5 scheduleVsyncLocked

	private void scheduleVsyncLocked() {
	    mDisplayEventReceiver.scheduleVsync();
	}

Choreographer通过FrameDisplayEventReceiver接收来自Native层显示系统的Vsync垂直同步脉冲信号.  
`class FrameDisplayEventReceiver extends DisplayEventReceiver`

首先看一下超类`DisplayEventReceiver`:  

	public abstract class DisplayEventReceiver {
	    private long mReceiverPtr;
 
	    @FastNative
	    private static native void nativeScheduleVsync(long receiverPtr);

		// 接收Vsync垂直同步信号, 在该方法中需要渲染一帧数据, 然后调用scheduleVsync启动下一个Vsync过程
		// timestampNanos为接收到Vsync信号的时间戳, frame表示帧序号
	    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
	    }
 
	    public void scheduleVsync() {
	        if (mReceiverPtr == 0) {
	            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
	                    + "receiver has already been disposed.");
	        } else {
	            nativeScheduleVsync(mReceiverPtr);
	        }
	    }
 
		// Called from native code.
	    private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame) {
	        onVsync(timestampNanos, physicalDisplayId, frame);
	    }
		...
	}

调用链:  DisplayEventReceiver#scheduleVsync -> nativeScheduleVsync  
由此可知Vsync机制是Native实现的.  

**1.Vsync机制Native层实现**:  
[ -> frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp ]

	static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
	    sp<NativeDisplayEventReceiver> receiver =
	            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
	    status_t status = receiver->scheduleVsync();
	    if (status) {
	        String8 message;
	        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
	        jniThrowRuntimeException(env, message.string());
	    }
	}

[ -> frameworks/base/libs/androidfw/DisplayEventDispatcher.cpp ]

	status_t DisplayEventDispatcher::scheduleVsync() {
	    if (!mWaitingForVsync) {
	        ALOGV("dispatcher %p ~ Scheduling vsync.", this);
 
	        // Drain all pending events.
	        nsecs_t vsyncTimestamp;
	        PhysicalDisplayId vsyncDisplayId;
	        uint32_t vsyncCount;
	        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
	            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "",
	                    this, ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
	        }
 
	        status_t status = mReceiver.requestNextVsync();
	        if (status) {
	            ALOGW("Failed to request next vsync, status=%d", status);
	            return status;
	        }
 
	        mWaitingForVsync = true;
	    }
	    return OK;
	}

[ -> frameworks/native/libs/gui/DisplayEventReceiver.cpp ]

	status_t DisplayEventReceiver::requestNextVsync() {
	    if (mEventConnection != nullptr) {
	        mEventConnection->requestNextVsync();
	        return NO_ERROR;
	    }
	    return NO_INIT;
	}

**2.启动Vsync过程**:  

	DisplayEventReceiver#scheduleVsync -> nativeScheduleVsync  -> JNI -> android_view_DisplayEventReceiver#nativeScheduleVsync -> DisplayEventDispatcher#scheduleVsync -> DisplayEventReceiver#requestNextVsync

更底层的实现细节暂时就不追究了.  

**3.接收Vsync垂直脉冲信号**  
调用链: `Native层 -> DisplayEventReceiver#dispatchVsync -> onVsync`  

再看一下实现类`FrameDisplayEventReceiver`:  

	private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
	    @Override
	    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
	        long now = System.nanoTime();

	        // Vsync产生的时间不能大于当前时间(如果大于, 说明Vsync机制出问题了)
	        if (timestampNanos > now) {
	            timestampNanos = now;
	        }
 
	        // 一次只能执行一个Vsync同步信号(如果上一个Vsync同步信号没执行完成, 则不能执行下一个)
	        if (mHavePendingVsync) {
	            Log.w(TAG, "Already have a pending vsync event.  There should only be one at a time.");
			} else {
	            mHavePendingVsync = true;
	        }
 
	        mTimestampNanos = timestampNanos;
	        mFrame = frame;
 
	        // 发送异步消息(异步消息执行不受SyncBarrier限制)
	        Message msg = Message.obtain(mHandler, this);
	        msg.setAsynchronous(true);
	        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
	    }
 
	    @Override
	    public void run() {
	        mHavePendingVsync = false;
	        doFrame(mTimestampNanos, mFrame);
	    }
	    ...
	}

Native层的Vsync垂直同步脉冲信号会回传到Java层的`FrameDisplayEventReceiver#onVsync`.  
该方法又向FrameHandler发送包含Runnable的异步信号, 在Runnable#run方法中又会触发doFrame.  至此才真正触发帧渲染的操作.  

**这里补充一点**:  
对于非Vsync机制, 是通过向FrameHandler发`MSG_DO_FRAME`异步消息, 触发doFrame, 实现刷新一帧的. 为了避免掉帧, 非Vsync是每隔`10ms`发送一次消息.   
不管是否Vsync机制, 都是每隔16ms刷新一帧. 只不过Vsync机制通过Native层的垂直脉冲控制帧频率; 而非Vsync机制通过Handler机制控制.  

### 1.6 doFrame

	// frameTimeNanos表示接收到Vsync垂直同步脉冲信号的时间戳
	void doFrame(long frameTimeNanos, int frame) {
	    final long startNanos;
	    synchronized (mLock) {
	        if (!mFrameScheduled) {
	            return; // no work to do
	        }
 
	        long intendedFrameTimeNanos = frameTimeNanos;
	        startNanos = System.nanoTime();
	        // 计算从接收到Vsync垂直同步脉冲信号到真正执行doFrame, 中间的延迟时间
	        final long jitterNanos = startNanos - frameTimeNanos;
	        // 检查掉帧
	        if (jitterNanos >= mFrameIntervalNanos) {
	            // 计算跳帧数
	            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
	            // 跳帧数超过30, Log提示主线程中做了过多工作.
	            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
	                Log.i(TAG, ...);
	            }
	            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
	            // 计算当前帧的时间戳(必须是mFrameIntervalNanos的整数倍)
	            frameTimeNanos = startNanos - lastFrameOffset;
	        }
 
	        // 若当前帧的时间戳小于上一帧的时间戳, 可能是VSync机制出了问题, 则再执行一次Vsync垂直同步信号
	        if (frameTimeNanos < mLastFrameTimeNanos) {
	            scheduleVsyncLocked();
	            return;
	        }
 
	        if (mFPSDivisor > 1) {
		        // 若距离上一帧的时间小于帧之间的时间间隔, 则再执行一次Vsync垂直同步信号
	            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
	            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
	                scheduleVsyncLocked();
	                return;
	            }
	        }
 
	        // 设置当前帧信息(包括接收Vsync垂直同步脉冲信号的时间戳 和 当前帧绘制的时间戳)
	        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
	        mFrameScheduled = false;
	        // 将当前帧的时间戳赋值给mLastFrameTimeNanos
	        mLastFrameTimeNanos = frameTimeNanos;
	    }
 
	    // 真正开始渲染当前帧数据, 执行时间戳为frameTimeNanos
	    try {
	        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
 
	        // 执行类型为CALLBACK_INPUT事件(如Touch事件)
	        mFrameInfo.markInputHandlingStart();
	        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
 
	        // 执行类型为CALLBACK_ANIMATION事件(如各种动画)
	        mFrameInfo.markAnimationsStart();
	        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
	        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
 
	        // 执行类型为CALLBACK_TRAVERSAL事件(如layout/draw等重绘操作)
	        mFrameInfo.markPerformTraversalsStart();
	        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
 
	        // 执行类型为CALLBACK_COMMIT事件, 作为结束标识(当前帧时间延迟的上报和时间戳调整)
	        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
	    } finally {
	        AnimationUtils.unlockAnimationClock();
	    }
	}

**doFrame逻辑**:  
(1) 确认当前帧执行的时间戳.  
由于接收到Vsync垂直同步脉冲信号的时间和真正调用doFrame的时间可能存在延迟, 因此可能会有跳帧或掉帧的可能.    
结合跳帧, 计算出当前帧执行的时间戳, 这个时间戳必须是帧时间间隔的整数倍.  
(2) 渲染当前帧数据.  
渲染一帧数据, 依次执行如下类型的doCallbacks操作:  
- `CALLBACK_INPUT`
- `CALLBACK_ANIMATION`
- `CALLBACK_INSETS_ANIMATION`
- `CALLBACK_TRAVERSAL`
- `CALLBACK_COMMIT`  

### 1.7 doCallbacks

	void doCallbacks(int callbackType, long frameTimeNanos) {
	    CallbackRecord callbacks;
	    synchronized (mLock) {
	        final long now = System.nanoTime();
	        // 获取dueTime小于当前时间戳的所有callback记录
	        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(now / TimeUtils.NANOS_PER_MS);
	        if (callbacks == null) {
	            return;
	        }
	        mCallbacksRunning = true;
 
	        if (callbackType == Choreographer.CALLBACK_COMMIT) {
		        // 计算从CALLBACK_INPUT到CALLBACK_TRAVERSAL中间执行过程的时间
	            final long jitterNanos = now - frameTimeNanos;
	            // 如果多于2帧的时间间隔, 则调整当前帧的时间戳, 并赋值给mLastFrameTimeNanos
	            if (jitterNanos >= 2 * mFrameIntervalNanos) {
	                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos + mFrameIntervalNanos;
	                frameTimeNanos = now - lastFrameOffset;
	                mLastFrameTimeNanos = frameTimeNanos;
	            }
	        }
	    }
	    try {
	        // 遍历并执行上述所有callback记录
	        for (CallbackRecord c = callbacks; c != null; c = c.next) {
	            c.run(frameTimeNanos);
	        }
	    } finally {
	        // 回收已执行完的callback记录
	        synchronized (mLock) {
	            mCallbacksRunning = false;
	            do {
	                final CallbackRecord next = callbacks.next;
	                recycleCallbackLocked(callbacks);
	                callbacks = next;
	            } while (callbacks != null);
	        }
	    }
	}

**doCallbacks逻辑**:  
(1) 从CallbackQueue[]数组中取出所有类型为callbackType且满足dueTime到期时间的CallbackRecord记录(单链表).  
(2) 遍历上述CallbackRecord单链表, 执行对应的Runnable操作  
(3) 回收上述CallbackRecord单链表, 也就是放到mCallbackPool对象池  

当callbackType为`CALLBACK_COMMIT`时:  
如果从CALLBACK_INPUT到CALLBACK_TRAVERSAL中间执行过程的时间超过2帧的时间间隔, 则调整当前帧的时间戳, 并赋值给mLastFrameTimeNanos.  

### 1.8 帧操作  
[ -> frameworks/base/core/java/android/view/ViewRootImpl.java ]

**1.CALLBACK_INPUT**  
对应的Runnable为ConsumeBatchedInputRunnable:  
    
    final ConsumeBatchedInputRunnable mConsumedBatchedInputRunnable = new ConsumeBatchedInputRunnable();

**2.CALLBACK_ANIMATION**  
对应的Runnable为InvalidateOnAnimationRunnable:  

	final class InvalidateOnAnimationRunnable implements Runnable {
        @Override
        public void run() {
            final int viewCount;
            final int viewRectCount;
            synchronized (this) {
                mPosted = false;

                viewCount = mViews.size();
                if (viewCount != 0) {
                    mTempViews = mViews.toArray(mTempViews != null
                            ? mTempViews : new View[viewCount]);
                    mViews.clear();
                }

                viewRectCount = mViewRects.size();
                if (viewRectCount != 0) {
                    mTempViewRects = mViewRects.toArray(mTempViewRects != null
                            ? mTempViewRects : new AttachInfo.InvalidateInfo[viewRectCount]);
                    mViewRects.clear();
                }
            }

            for (int i = 0; i < viewCount; i++) {
                mTempViews[i].invalidate();
                mTempViews[i] = null;
            }

            for (int i = 0; i < viewRectCount; i++) {
                final View.AttachInfo.InvalidateInfo info = mTempViewRects[i];
                info.target.invalidate(info.left, info.top, info.right, info.bottom);
                info.recycle();
            }
        }
	}

**3.CALLBACK_INSETS_ANIMATION**  
对应的Runnable为InsetsController#mAnimCallback  

        mAnimCallback = () -> {
            mAnimCallbackScheduled = false;
            if (mAnimationControls.isEmpty()) {
                return;
            }

            mTmpFinishedControls.clear();
            InsetsState state = new InsetsState(mState, true /* copySources */);
            for (int i = mAnimationControls.size() - 1; i >= 0; i--) {
                InsetsAnimationControlImpl control = mAnimationControls.get(i);
                if (mAnimationControls.get(i).applyChangeInsets(state)) {
                    mTmpFinishedControls.add(control);
                }
            }

            WindowInsets insets = state.calculateInsets(mFrame, mLastInsets.isRound(),
                    mLastInsets.shouldAlwaysConsumeSystemBars(), mLastInsets.getDisplayCutout(),
                    mLastLegacyContentInsets, mLastLegacyStableInsets, mLastLegacySoftInputMode,
                    null /* typeSideMap */);
            mViewRoot.mView.dispatchWindowInsetsAnimationProgress(insets);

            for (int i = mTmpFinishedControls.size() - 1; i >= 0; i--) {
                dispatchAnimationFinished(mTmpFinishedControls.get(i).getAnimation());
            }
        };

**4.CALLBACK_TRAVERSAL**  
对应的Runnable为TraversalRunnable:  
	
	final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }



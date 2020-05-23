---
layout: post
comments: true
title: "理解ViewRootImpl"
description: "理解ViewRootImpl"
category: Efficiency
tags: [Efficiency]
---

<!--more-->

本文以View的requestLayout/invalidate/postInvalidate为切入口, 引申到ViewRootImpl过程分析.

主要按照下面3个步骤:  
1.requestLayout / invalidate / postInvalidate的异同  
2.ViewRootImpl过程  
3.常见问题分析  

## 1.View的绘制调用方式  
[ -> frameworks/base/core/java/android/view/View.java ]

### 1.1 requestLayout

    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

		// 步骤1
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

		// 步骤2
        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;
        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }

调用时机:  
当某些变化导致原有View树的layout发生变化, 可以调用requestLayout重新布局View树.  

**1.关于requestLayout的叠加**  
(1) 尽量不要在layout过程中又调用requestLayout.  
(2) 如果非要叠加调用requestLayout, 则将当前作出该请求的View先放入mLayoutRequesters列表, 等到调用ViewRootImpl#performLayout的时候再取出mLayoutRequesters列表依次调用各个View#requestLayout. 如果当前正在处理mLayoutRequesters列表, 则不要再次触发requestLayout.  

**2.mParent.requestLayout()**  
(1) 我们先搞清楚mParent是啥?   
mParent的类型是ViewParent, 通过代码溯源View#assignParent, 可以得出如下结论:  
DecorView(即根View)对应的mParent是ViewRootImpl, 普通子View(非根View)对应的mParent是子View的父View(即ViewGroup)  

(2) requestLayout调用过程?    
该过程是一个从 子View -> 父View -> DecorView -> ViewRootImpl 层层往上的递归调用过程.  
`View#requestLayout -> ViewGroup#requestLayout -> ... -> DecorView#requestLayout -> ViewRootImpl#requestLayout`  

### 1.2 invalidate

[ -> frameworks/base/core/java/android/view/View.java ]

    public void invalidate() {
        invalidate(true);
    }

    public void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
        if (skipInvalidate()) {
            return;
        }
        ...
        // Propagate the damage rectangle to the parent view.
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            p.invalidateChild(this, damage);
        }
        ...
    }

[ -> frameworks/base/core/java/android/view/ViewGroup.java ]

    @Override
    public final void invalidateChild(View child, final Rect dirty) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null && attachInfo.mHardwareAccelerated) {
            // HW accelerated fast path
            onDescendantInvalidated(child, child);
            return;
        }

        ViewParent parent = this;
        if (attachInfo != null) {
            final int[] location = attachInfo.mInvalidateChildLocation;
            location[CHILD_LEFT_INDEX] = child.mLeft;
            location[CHILD_TOP_INDEX] = child.mTop;
			...
            do {
                ...
                parent = parent.invalidateChildInParent(location, dirty);
                ...
            } while (parent != null);
        }
    }

调用时机:  
当前View树需要重绘时. 如果当前View可见, 则会调到onDraw方法.  
该方法必须在UI线程调用.  

View#invalidate的调用逻辑:  
从子View -> 父View -> DecorView -> ViewRootImpl 从子到父层层调用.  
`View#invalidateChild -> ViewGroup#invalidateChild -> ViewGroup#invalidateChildInParent -> ... -> DecorView#invalidateChildInParent -> ViewRootImpl#invalidateChildInParent`

### 1.3 postInvalidate  
[ -> frameworks/base/core/java/android/view/View.java ]

    public void postInvalidate() {
        postInvalidateDelayed(0);
    }

    public void postInvalidateDelayed(long delayMilliseconds) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
        }
    }

[ -> frameworks/base/core/java/android/view/ViewRootImpl.java ]

    public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
        Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
        mHandler.sendMessageDelayed(msg, delayMilliseconds);
    }

调用时机:  
和invalidate类似.   
不同的是, postInvalidate通过发送一个消息到主线程的消息队列, 在消息队列中执行invalidate方法.  
因此postInvalidate可以在非UI线程调用.  

### 1.4 requestLayout & invalidate & postInvalidate

**相同点:**  
(1) 都是从 子View -> 父View -> DecorView -> ViewRootImpl 从子到父层层调用.  
(2) 都会导致View树重绘, 最终都会调用到ViewRootImpl#scheduleTraversals  

**不同点**:  
(1) requestLayout 在UI线程调用  
(2) invalidate在UI线程调用. 当View树可见时, 会调用到onDraw  
(3) postInvalidate可以在非UI线程调用. 逻辑和invalidate类似, 不同点是postInvalidate通过发消息方式, 使invalidate操作在消息队列里有序执行.  

## 2.ViewRootImpl  
[ -> frameworks/base/core/java/android/view/ViewRootImpl.java ]





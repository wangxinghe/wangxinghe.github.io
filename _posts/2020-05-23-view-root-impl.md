---
layout: post
comments: true
title: "理解ViewRootImpl"
description: "理解ViewRootImpl"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析

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

**1. 调用时机**:  
当前View树需要重绘时. 如果当前View可见, 则会调到onDraw方法.  
该方法必须在UI线程调用.  

**2. View#invalidate的调用逻辑**:  
从子View -> 父View -> DecorView -> ViewRootImpl 从子到父层层调用.  

(1) 硬件加速:  
`ViewGroup#onDescendantInvalidated -> ...-> DecorView#onDescendantInvalidated ->  ViewRootImpl#onDescendantInvalidated`

(2) 非硬件加速:  
`ViewGroup#invalidateChild -> ViewGroup#invalidateChildInParent -> ... -> DecorView#invalidateChildInParent -> ViewRootImpl#invalidateChildInParent`

**3. skipInvalidate逻辑**  

    private boolean skipInvalidate() {
        return (mViewFlags & VISIBILITY_MASK) != VISIBLE && mCurrentAnimation == null &&
                (!(mParent instanceof ViewGroup) ||
                        !((ViewGroup) mParent).isViewTransitioning(this));
    }

如果当前View不可见并且当前没有动画时, 则不会invalidate执行重绘.  

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

### 2.1 scheduleTraversals

	void scheduleTraversals() {
	    if (!mTraversalScheduled) {
	        mTraversalScheduled = true;
	        // 添加同步屏障
	        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
	        // 发送并执行遍历操作
	        mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
	        if (!mUnbufferedInputDispatch) {
	            scheduleConsumeBatchedInput();
	        }
	        notifyRendererOfFramePending();
	        pokeDrawLockIfNeeded();
	    }
	}

	final class TraversalRunnable implements Runnable {
	    @Override
	    public void run() {
	        doTraversal();
	    }
	}

	void doTraversal() {
	    if (mTraversalScheduled) {
	        mTraversalScheduled = false;
	        // 移除同步屏障
	        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
 
	        performTraversals();
	    }
	}

**1.同步屏障SyncBarrier**  
当调用postSyncBarrier后, MessageQueue中的同步消息将不能执行, 直到removeSyncBarrier才会执行. 这个不影响异步消息.  

在设置了同步屏障后, 发送一个`CALLBACK_TRAVERSAL`类型消息到Choreographer的消息队列.  
在移除了同步屏障后, 执行performTraversals  

### 2.2 performTraversals

	private void performTraversals() {
	    ...
	    // 如果当前View树中包含SurfaceView, 则执行surfaceCreated/surfaceChanged回调
	    if (mSurfaceHolder != null) {
	        if (mSurface.isValid()) {
	            mSurfaceHolder.mSurface = mSurface;
	        }
	        mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
	        if (mSurface.isValid()) {
	            if (!hadSurface) {
	                mSurfaceHolder.ungetCallbacks();
 
	                mIsCreating = true;
	                SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
	                if (callbacks != null) {
	                    for (SurfaceHolder.Callback c : callbacks) {
	                        c.surfaceCreated(mSurfaceHolder);
	                    }
	                }
	                surfaceChanged = true;
	            }
	            if (surfaceChanged || surfaceGenerationId != mSurface.getGenerationId()) {
	                SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
	                if (callbacks != null) {
	                    for (SurfaceHolder.Callback c : callbacks) {
	                        c.surfaceChanged(mSurfaceHolder, lp.format, mWidth, mHeight);
	                    }
	                }
	            }
	            mIsCreating = false;
	        }
	    }
 
	    ...
	    // mWidth&mHeight为Frame宽高, lp为setView传进来的WindowManager.LayoutParams参数
	    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
	    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
	    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	    ...
	    performLayout(lp, mWidth, mHeight);
	    ...
	    // 调用OnGlobalLayoutListener#onGlobalLayout
	    if (triggerGlobalLayoutListener) {
	        mAttachInfo.mRecomputeGlobalAttributes = false;
	        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
	    }
	    ...
	    performDraw();
	    ...
	}

主要做了2件事:  
(1) 如果View树中当前View为SurfaceView, 则执行surfaceCreated/surfaceChanged相关回调.  
(2) 依次执行performMeasure -> performLayout -> performDraw  
(3) OnGlobalLayoutListener#onGlobalLayout回调中可以获取到View的真是宽高. 因为该方法在performMeasure -> performLayout后面执行.  

## 3.performMeasure过程  

### 3.1 ViewRootImpl#performMeasure

	private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
	    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	}

### 3.2 View#measure

	public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	    boolean optical = isLayoutModeOptical(this);
	    if (optical != isLayoutModeOptical(mParent)) {
	        Insets insets = getOpticalInsets();
	        int oWidth  = insets.left + insets.right;
	        int oHeight = insets.top  + insets.bottom;
	        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
	        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
	    }
 
	    // Suppress sign extension for the low bytes
	    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
	    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
 
	    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
 
	    // Optimize layout by avoiding an extra EXACTLY pass when the view is
	    // already measured as the correct size. In API 23 and below, this
	    // extra pass is required to make LinearLayout re-distribute weight.
	    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
	            || heightMeasureSpec != mOldHeightMeasureSpec;
	    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
	            && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
	    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
	            && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
	    final boolean needsLayout = specChanged
	            && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
 
	    if (forceLayout || needsLayout) {
	        // first clears the measured dimension flag
	        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
 
	        resolveRtlPropertiesIfNeeded();
	 
	        int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
	        if (cacheIndex < 0 || sIgnoreMeasureCache) {
	            // measure ourselves, this should set the measured dimension flag back
	            onMeasure(widthMeasureSpec, heightMeasureSpec);
	            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	        } else {
	            long value = mMeasureCache.valueAt(cacheIndex);
	            // Casting a long to int drops the high 32 bits, no mask needed
	            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
	            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	        }
 
	        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
	    }
 
	    mOldWidthMeasureSpec = widthMeasureSpec;
	    mOldHeightMeasureSpec = heightMeasureSpec;
 
	    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
	            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
	}

### 3.3 View#onMeasure

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
	            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
	}

### 3.4 View#setMeasuredDimension

	protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
	    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
	}

	private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
	    mMeasuredWidth = measuredWidth;
	    mMeasuredHeight = measuredHeight;
 
	    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
	}

**1. Measure过程调用链路:**    
`ViewRootImpl#performMeasure -> View#measure -> View#onMeasure -> View#setMeasureDimension`  

**2. 缓存策略**  
为了避免每次重复测量, 采用了缓存策略. 测量缓存数据结构`LongSparseLongArray`.  
其中key为MeasureSpec, value为MeasuredWidth/MeasuredHeight. 高32位代表宽度, 低32位代表高度.   

	// key
	(long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL  
	// value
	(long) mMeasuredWidth << 32 |  (long) mMeasuredHeight & 0xffffffffL

**3.测量过程**  
如果缓存中没有, 则需要测量方法View#onMeasure. 具体的测量宽高方式参考getDefaultSize.    
其中size为建议大小getSuggestedMinimumWidth/Height, measureSpec为测量规范.  

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
    
	public static int getDefaultSize(int size, int measureSpec) {
	    int result = size;
	    int specMode = MeasureSpec.getMode(measureSpec);
	    int specSize = MeasureSpec.getSize(measureSpec);
 
	    switch (specMode) {
	    case MeasureSpec.UNSPECIFIED:
	        result = size;
	        break;
	    case MeasureSpec.AT_MOST:
	    case MeasureSpec.EXACTLY:
	        result = specSize;
	        break;
	    }
	    return result;
	}

**4.MeasureSpec**  

上面提到了int measureSpec为测量规范. 怎么理解这个测量规范呢? 可以用类`MeasureSpec`来描述.   

MeasureSpec由**mode**和**size**组成, 前2位为mode, 后30位为size.  之所以这么设计, 是出于节省内存考虑.  

**3种mode类型**:  
`UNSPECIFIED`  父View没有对子View做限制  
`EXACTLY`  父View指定了子View的精确大小  
`AT_MOST`  父View指定了子View的最大值, 子View最大到达这个最大值  

    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        public static final int AT_MOST     = 2 << MODE_SHIFT;

        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}

        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size, @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }

        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }

        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }

## 4.performLayout过程

### 4.1 ViewRootImpl#performLayout

	private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
	    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
	}

### 4.2 View#layout

	public void layout(int l, int t, int r, int b) {
	    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
	        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
	        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	    }
 
	    int oldL = mLeft;
	    int oldT = mTop;
	    int oldB = mBottom;
	    int oldR = mRight;
 
	    boolean changed = isLayoutModeOptical(mParent) ?
	            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
 
	    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
	        onLayout(changed, l, t, r, b);
 
	        if (shouldDrawRoundScrollbar()) {
	            if(mRoundScrollbarRenderer == null) {
	                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
	            }
	        } else {
	            mRoundScrollbarRenderer = null;
	        }
 
	        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
 
	        ListenerInfo li = mListenerInfo;
	        if (li != null && li.mOnLayoutChangeListeners != null) {
	            ArrayList<OnLayoutChangeListener> listenersCopy =
	                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
	            int numListeners = listenersCopy.size();
	            for (int i = 0; i < numListeners; ++i) {
	                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
	            }
	        }
	    }
 
	    final boolean wasLayoutValid = isLayoutValid();
 
	    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
	    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
 
	    if (!wasLayoutValid && isFocused()) {
	        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
	        if (canTakeFocus()) {
	            // We have a robust focus, so parents should no longer be wanting focus.
	            clearParentsWantFocus();
	        } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
	            // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
	            // layout. In this case, there's no guarantee that parent layouts will be evaluated
	            // and thus the safest action is to clear focus here.
	            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
	            clearParentsWantFocus();
	        } else if (!hasParentWantsFocus()) {
	            // original requestFocus was likely on this view directly, so just clear focus
	            clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
	        }
	        // otherwise, we let parents handle re-assigning focus during their layout passes.
	    } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
	        mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
	        View focused = findFocus();
	        if (focused != null) {
	            // Try to restore focus as close as possible to our starting focus.
	            if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
	                // Give up and clear focus once we've reached the top-most parent which wants
	                // focus.
	                focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
	            }
	        }
	    }
 
	    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
	        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
	        notifyEnterOrExitForAutoFillIfNeeded(true);
	    }
	}

**1. Layout过程调用链路:**    
`ViewRootImpl#performLayout -> View#layout -> View#onLayout`

## 5.performDraw过程

### 5.1 ViewRootImpl#performDraw

	private void performDraw() {
	    ...
	    final Canvas canvas = mSurface.lockCanvas(dirty);
	    canvas.setDensity(mDensity);
	    canvas.translate(-xoff, -yoff);
	    canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
	    ...
	    mView.draw(canvas);
	    surface.unlockCanvasAndPost(canvas);
	}

performDraw调用流程:  
draw(fullRedrawNeeded) -> drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty, surfaceInsets) -> mView.draw(canvas)

### 5.2 View#draw

	public void draw(Canvas canvas) {
	    final int privateFlags = mPrivateFlags;
	    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
 
	    // Step 1, draw the background, if needed
	    int saveCount; 
	    drawBackground(canvas);
 
	    // skip step 2 & 5 if possible (common case)
	    final int viewFlags = mViewFlags;
	    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
	    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
	    if (!verticalEdges && !horizontalEdges) {
	        // Step 3, draw the content
	        onDraw(canvas);
 
	        // Step 4, draw the children
	        dispatchDraw(canvas);
 
	        drawAutofilledHighlight(canvas);
 
	        // Overlay is part of the content and draws beneath Foreground
	        if (mOverlay != null && !mOverlay.isEmpty()) {
	            mOverlay.getOverlayView().dispatchDraw(canvas);
	        }
 
	        // Step 6, draw decorations (foreground, scrollbars)
	        onDrawForeground(canvas);
 
	        // Step 7, draw the default focus highlight
	        drawDefaultFocusHighlight(canvas);
 
	        if (debugDraw()) {
	            debugDrawFocus(canvas);
	        }
 
	        // we're done...
	        return;
	    }
 
	    boolean drawTop = false;
	    boolean drawBottom = false;
	    boolean drawLeft = false;
	    boolean drawRight = false;
 
	    float topFadeStrength = 0.0f;
	    float bottomFadeStrength = 0.0f;
	    float leftFadeStrength = 0.0f;
	    float rightFadeStrength = 0.0f;
 
	    // Step 2, save the canvas' layers
	    int paddingLeft = mPaddingLeft;
 
	    final boolean offsetRequired = isPaddingOffsetRequired();
	    if (offsetRequired) {
	        paddingLeft += getLeftPaddingOffset();
	    }
 
	    int left = mScrollX + paddingLeft;
	    int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
	    int top = mScrollY + getFadeTop(offsetRequired);
	    int bottom = top + getFadeHeight(offsetRequired);
 
	    if (offsetRequired) {
	        right += getRightPaddingOffset();
	        bottom += getBottomPaddingOffset();
	    }
 
	    final ScrollabilityCache scrollabilityCache = mScrollCache;
	    final float fadeHeight = scrollabilityCache.fadingEdgeLength;
	    int length = (int) fadeHeight;
 
	    // clip the fade length if top and bottom fades overlap
	    // overlapping fades produce odd-looking artifacts
	    if (verticalEdges && (top + length > bottom - length)) {
	        length = (bottom - top) / 2;
	    }
 
	    // also clip horizontal fades if necessary
	    if (horizontalEdges && (left + length > right - length)) {
	        length = (right - left) / 2;
	    }
 
	    if (verticalEdges) {
	        topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
	        drawTop = topFadeStrength * fadeHeight > 1.0f;
	        bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
	        drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
	    }
 
	    if (horizontalEdges) {
	        leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
	        drawLeft = leftFadeStrength * fadeHeight > 1.0f;
	        rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
	        drawRight = rightFadeStrength * fadeHeight > 1.0f;
	    }
 
	    saveCount = canvas.getSaveCount();
	    int topSaveCount = -1;
	    int bottomSaveCount = -1;
	    int leftSaveCount = -1;
	    int rightSaveCount = -1;
 
	    int solidColor = getSolidColor();
	    if (solidColor == 0) {
	        if (drawTop) {
	            topSaveCount = canvas.saveUnclippedLayer(left, top, right, top + length);
	        }
 
	        if (drawBottom) {
	            bottomSaveCount = canvas.saveUnclippedLayer(left, bottom - length, right, bottom);
	        }
 
	        if (drawLeft) {
	            leftSaveCount = canvas.saveUnclippedLayer(left, top, left + length, bottom);
	        }
 
	        if (drawRight) {
	            rightSaveCount = canvas.saveUnclippedLayer(right - length, top, right, bottom);
	        }
	    } else {
	        scrollabilityCache.setFadeColor(solidColor);
	    }
 
	    // Step 3, draw the content
	    onDraw(canvas);
 
	    // Step 4, draw the children
	    dispatchDraw(canvas);
 
	    // Step 5, draw the fade effect and restore layers
	    final Paint p = scrollabilityCache.paint;
	    final Matrix matrix = scrollabilityCache.matrix;
	    final Shader fade = scrollabilityCache.shader;
 
	    // must be restored in the reverse order that they were saved
	    if (drawRight) {
	        matrix.setScale(1, fadeHeight * rightFadeStrength);
	        matrix.postRotate(90);
	        matrix.postTranslate(right, top);
	        fade.setLocalMatrix(matrix);
	        p.setShader(fade);
	        if (solidColor == 0) {
	            canvas.restoreUnclippedLayer(rightSaveCount, p);
	        } else {
	            canvas.drawRect(right - length, top, right, bottom, p);
	        }
	    }
 
	    if (drawLeft) {
	        matrix.setScale(1, fadeHeight * leftFadeStrength);
	        matrix.postRotate(-90);
	        matrix.postTranslate(left, top);
	        fade.setLocalMatrix(matrix);
	        p.setShader(fade);
	        if (solidColor == 0) {
	            canvas.restoreUnclippedLayer(leftSaveCount, p);
	        } else {
	            canvas.drawRect(left, top, left + length, bottom, p);
	        }
	    }
 
	    if (drawBottom) {
	        matrix.setScale(1, fadeHeight * bottomFadeStrength);
	        matrix.postRotate(180);
	        matrix.postTranslate(left, bottom);
	        fade.setLocalMatrix(matrix);
	        p.setShader(fade);
	        if (solidColor == 0) {
	            canvas.restoreUnclippedLayer(bottomSaveCount, p);
	        } else {
	            canvas.drawRect(left, bottom - length, right, bottom, p);
	        }
	    }
 
	    if (drawTop) {
	        matrix.setScale(1, fadeHeight * topFadeStrength);
	        matrix.postTranslate(left, top);
	        fade.setLocalMatrix(matrix);
	        p.setShader(fade);
	        if (solidColor == 0) {
	            canvas.restoreUnclippedLayer(topSaveCount, p);
	        } else {
	            canvas.drawRect(left, top, right, top + length, p);
	        }
	    }
 
	    canvas.restoreToCount(saveCount);
 
	    drawAutofilledHighlight(canvas);
 
	    // Overlay is part of the content and draws beneath Foreground
	    if (mOverlay != null && !mOverlay.isEmpty()) {
	        mOverlay.getOverlayView().dispatchDraw(canvas);
	    }
 
	    // Step 6, draw decorations (foreground, scrollbars)
	    onDrawForeground(canvas);
 
	    if (debugDraw()) {
	        debugDrawFocus(canvas);
	    }
	}

**1.调用链路**  
`ViewRootImpl#performDraw -> View#draw`  

**2.draw绘制顺序**  
由下到上的绘制顺序.  
(1) `drawBackground`  绘制背景  
(2) save layer (当有水平或垂直fading edges时)  
(3) `onDraw` 绘制当前View的内容  
(4) `dispatchDraw`  绘制当前View的子View  
(5) 绘制fading edges和restore layers(当有水平或垂直fading edges时)  
(6) `onDrawForeground`  绘制前景或滚动条  
(7) `drawDefaultFocusHighlight`  绘制获取焦点的View的Focus高亮  

## 6.常见问题  

**1. 什么时候获取View的测量宽高? **  

	private void performTraversals() {
	    ...
	    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	    ...
	    performLayout(lp, mWidth, mHeight);
	    ...
	    if (triggerGlobalLayoutListener) {
	        mAttachInfo.mRecomputeGlobalAttributes = false;
	        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
	    }
	    ...
	    performDraw();
	    ...
	}

根据ViewRootImpl#scheduleTraversals的调用逻辑. dispatchOnGlobalLayout在performMeasure -> performLayout之后调用, 通过该回调可以获取到测量的宽高.  

App层可以通过注册OnGlobalLayoutListener方式获取.  

	view.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener() {
	    @Override
	    public void onGlobalLayout() {
			// view.getMeasuredWidth()/view.getMeasuredHeight()
			...           
	    }
	});

**2. 在子线程中可以更新UI吗?**  

在Activity#onResume之前, 可以在子线程中更新UI.  
checkThread线程检查是在ViewRootImpl的方法中进行的 (如invalidate和requestLayout)  
而通过溯源代码, ViewRootImpl是在ActivityThread#handleResumeActivity创建的. 在Activity#onResume之前ViewRootImpl还没创建, 所以也不会检查线程和绘制UI.  

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    
	@Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        checkThread();
        ...
    }

    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }


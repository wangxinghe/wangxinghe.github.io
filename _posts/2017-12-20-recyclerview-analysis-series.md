---
layout: post
comments: true
title: "RecyclerView源码分析（续）"
description: "RecyclerView源码分析（续）"
category: android
tags: [Android]
---


去年写过一篇RecyclerView的源码分析，时隔一年又看了一遍RecyclerView和ListView的源码，有一些新的体会。出于篇幅考虑本篇还是写RecyclerView，下一篇写ListView。

本文还是先按照`measure -> layout -> draw`的基本流程过代码。

### 0x00 基本代码结构

#### 0. Measure

`onMeasure`调用方法`defaultOnMeasure(int widthSpec, int heightSpec)`，这部分没啥好讲的，代码如下：

```java
private void defaultOnMeasure(int widthSpec, int heightSpec) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    final int widthSize = MeasureSpec.getSize(widthSpec);
    final int heightSize = MeasureSpec.getSize(heightSpec);

    int width = 0;
    int height = 0;

    switch (widthMode) {
        case MeasureSpec.EXACTLY:
        case MeasureSpec.AT_MOST:
            width = widthSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            width = ViewCompat.getMinimumWidth(this);
            break;
    }

    switch (heightMode) {
        case MeasureSpec.EXACTLY:
        case MeasureSpec.AT_MOST:
            height = heightSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            height = ViewCompat.getMinimumHeight(this);
            break;
    }

    setMeasuredDimension(width, height);
}
```
#### 1. Layout

`onLayout`部分逻辑主要集中在`dispatchLayout()`方法。这部分做的工作依次是`preLayout -> onLayoutChildren -> postLayout`，下面是源码中注释：

// Step 0: Find out where all non-removed items are, pre-layout

// Step 1: run prelayout: This will use the old positions of items. The layout manager is expected to layout everything, even removed items (though not to add removed items back to the container). This gives the pre-layout position of APPEARING views which come into existence as part of the real layout.

// Step 2: Run layout
mLayout.onLayoutChildren(mRecycler, mState);

// Step 3: Find out where things are now, post-layout

// Step 4: Animate DISAPPEARING and REMOVED items
// Step 5: Animate APPEARING and ADDED items
// Step 6: Animate PERSISTENT items
// Step 7: Animate CHANGING items
最后调用mLayout.removeAndRecycleScrapInt(mRecycler);

我们重点关注`mLayout.onLayoutChildren(mRecycler, mState)`，这部分代码是布局各个child View的。

以`LinearLayoutManager`为例，如下是`onLayoutChildren`代码注释：
 // layout algorithm:
 // 1) by checking children and other variables, find an anchor coordinate and an anchor item position.
 // 2) fill towards start, stacking from bottom
 // 3) fill towards end, stacking from top
 // 4) scroll to fulfill requirements like stack from bottom.
 // create layout state
 
 从上面注释可以看出流程依次是：
 1. 找出锚点
 2. 从锚点出发依次往上布局子View
 3. 从锚点出发依次往下布局子View
 4. 滚动时位置修正处理

##### 锚点位置和坐标的确定（mPosition, mCoordinate）
我们分析`updateAnchorInfoForLayout(state, mAnchorInfo)`源码，锚点会根据mPendingSavedState确定或从现有的子View中寻找并确定。一般是选取离父View的start或end最接近的item作为锚点，因此index一般是0或getChildCount() - 1。

##### 以锚点作为起始点填充子View
这部分逻辑主要集中在`fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable)`，基本思路是从锚点往上或往下填充子View，直至父View没有更多可见空间。
核心代码如下：
```java
public int fill(RecyclerView.Recycler recycler, LayoutState layoutState, 
			RecyclerView.State state, boolean stopOnFocusable) {
	...
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
	LayoutChunkResult layoutChunkResult = new LayoutChunkResult();
	while (remainingSpace > 0) && layoutState.hasMore(state) {
	  layoutChunkResult.resetInternal();
	  layoutChunk(recycler, state, layoutState, layoutChunkResult);
	  remainingSpace -= layoutChunkResult.mConsumed;
	}
	...
}
```
不难推断`layoutChunk`方法的作用是布局每一个子View，我们跟进其代码：
```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    ...
    measureChildWithMargins(view, 0, 0);
    ...
    layoutDecorated(view, left + params.leftMargin, top + params.topMargin,
       right - params.rightMargin, bottom - params.bottomMargin);
}
```
`layoutChunk`具体逻辑如下：
（1）确定下一个用来显示的子View
（2）测量子View的大小
（3）布局子View

关键代码：
```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
	View view = layoutState.next(recycler);
	...
	measureChildWithMargins(view, 0, 0);
	...
	layoutDecorated(view, left + params.leftMargin, top + params.topMargin,
                right - params.rightMargin, bottom - params.bottomMargin);
}
```

##### 确定下一个用来显示的子View
主要逻辑集中在`layoutState.next(recycler)`，`Recycler.getViewForPosition(int position)`实现View的复用。
子View根据优先级从下面数据源复用：
（1）从`mAttachedScrap/mChangedScrap`中获取
（2）从`mCachedViews`中获取
（3）从`mViewCacheExtension`中获取
（4）从`RecycledViewPool`中获取
（5）从`mAdapter.createViewHolder()`中获取

以上优先根据position匹配，如果hasStableId==true，则根据itemId匹配。

上面4个数据源，都是`Recycler`的成员变量。
```java
ArrayList< ViewHolder > mAttachedScrap;
ArrayList< ViewHolder > mChangedScrap;
ArrayList< ViewHolder > mCachedViews;
ViewCacheExtension mViewCacheExtension;
RecycledViewPool mRecyclerPool;

public static final int DEFAULT_CACHE_SIZE = 2;
int mViewCacheMax = DEFAULT_CACHE_SIZE; 
```
`mAttachScrap`和`mChangedScrap`：在调用fill填充子View前，将当前前台显示的子View添加到`mAttachScrap/mChangedScrap`。这两个数据源生命周期仅存在于锚点确定后，子View布局前。不同点是`mChangedScrap`中添加的是子View内容发生变化（FLAG_CHANGED）且支持change动画的ViewHolder列表。
`mViewCacheExtension`：可动态设置的外部接口。
`mCachedViews`：当子View从父View中remove掉时，会将子View添加到这个列表。这种情况出现在调用setAdapter或者onLayoutChildren时移除disappering view时。`mCachedViews`最大容量为mViewCacheMax，默认最大容量是2，超出最大容量时，会移除列表靠前项并将其添加到`RecycledViewPool`。
`mRecyclerPool`：当setAdapter时，会清空mCachedViews和mRecyclerPool中的内容，是4级缓存中最后一级。
`RecycledViewPool`数据结构如下：
```java
private SparseArray<ArrayList<ViewHolder>> mScrap = new SparseArray<ArrayList<ViewHolder>>();
private SparseIntArray mMaxScrap = new SparseIntArray();
private static final int DEFAULT_MAX_SCRAP = 5;
```
`mScrap`是一个映射，其中key为viewType，value为ArrayList< ViewHolder >
`mMaxScrap`是一个int映射，key为viewType，value为mScrap.get(viewType)得到的列表的最大容量，默认最大容量为5

注意：以上数据源之间的数据没有重复的，同一个数据只会存在于一个数据源。

若以上4个数据源都没有获取到，则通过`RecyclerView.createViewHolder()`创建。

找到要展示的View后，接下来就是数据绑定，将对应的item数据项绑定到对应的ViewHolder。
```java
if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
	mAdapter.bindViewHolder(holder, offsetPosition);
}
```

##### 测量子View的大小

```java
public void measureChildWithMargins(View child, int widthUsed, int heightUsed) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();

    final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);
    widthUsed += insets.left + insets.right;
    heightUsed += insets.top + insets.bottom;

    final int widthSpec = getChildMeasureSpec(getWidth(),
            getPaddingLeft() + getPaddingRight() +
                    lp.leftMargin + lp.rightMargin + widthUsed, lp.width,
            canScrollHorizontally());
    final int heightSpec = getChildMeasureSpec(getHeight(),
            getPaddingTop() + getPaddingBottom() +
                    lp.topMargin + lp.bottomMargin + heightUsed, lp.height,
            canScrollVertically());
    child.measure(widthSpec, heightSpec);
}

Rect getItemDecorInsetsForChild(View child) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    if (!lp.mInsetsDirty) {
        return lp.mDecorInsets;
    }

    final Rect insets = lp.mDecorInsets;
    insets.set(0, 0, 0, 0);
    final int decorCount = mItemDecorations.size();
    for (int i = 0; i < decorCount; i++) {
        mTempRect.set(0, 0, 0, 0);
        mItemDecorations.get(i).getItemOffsets(mTempRect, child, this, mState);
        insets.left += mTempRect.left;
        insets.top += mTempRect.top;
        insets.right += mTempRect.right;
        insets.bottom += mTempRect.bottom;
    }
    lp.mInsetsDirty = false;
    return insets;
}

public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
    getItemOffsets(outRect, 
    ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(), parent);
}

public void getItemOffsets(Rect outRect, int itemPosition, RecyclerView parent) {
    outRect.set(0, 0, 0, 0);
}

public static int getChildMeasureSpec(int parentSize, int padding, int childDimension,
        boolean canScroll) {
    int size = Math.max(0, parentSize - padding);
    int resultSize = 0;
    int resultMode = 0;

    if (canScroll) {
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else {
            // MATCH_PARENT can't be applied since we can scroll in this dimension, wrap
            // instead using UNSPECIFIED.
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
    } else {
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.FILL_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

通过上面代码，我们发现：
（1）orientation = vertical，垂直可滚动
若childDimension > 0，则宽高就是childDimension
若childDimension <= 0，则：
当childDimension = MATCH_PARENT时， child view的宽度 = parent width - padding horizontal
当childDimension = WRAP_CONTENT时，child view的最大宽度是parent - padding horizontal
child view的高度根据resultSize = 0 resultMode = MeasureSpec.UNSPECIFIED 来计算
（2）orientation = horizontal，水平可滚动：
若childDimension > 0，则宽高就是childDimension
若childDimension <= 0，则：
当childDimension = MATCH_PARENT时， child view的高度 = parent width - padding vertical
当childDimension = WRAP_CONTENT时，child view的最大高度是parent - padding vertical
child view的宽度根据resultSize = 0 resultMode = MeasureSpec.UNSPECIFIED 来计算

关于子View和父View间的padding：
padding包括了parent padding、lp.mDecorInsets、lp.margin三部分。
水平padding = getPaddingLeft() + getPaddingRight() + insets.left + insets.right  + lp.marginLeft + lp.marginRight
垂直padding = getPaddingTop() + getPaddingBottom() + insets.top + insets.bottom  + lp.marginTop + lp.marginBottom
mDecorInsets可以理解成子View与父View的外边距，不属于子View内间距，和margin类似。
mDecorInsets可以根据ItemDecorations.getItemOffsets()进行调整，默认情况下为0。

##### 布局子View

前面已经分析了父View和子View之间的间距，计算出相对于父View的(left, top, right, bottom)后，调用chid.layout(left, top, right, bottom)就可以布局了。

#### 2. Draw

接下来讲`draw`部分：

回顾一下，普通View的绘制流程draw(canvas)：
1. draw the background
2. If necessary, save the canvas' layers to prepare for fading
3. Draw view's content: onDraw(canvas)
4. Draw children:  dispatchDraw(canvas)
5. If necessary, draw the fading edges and restore layers
6. Draw decorations (scrollbars for instance)

RecyclerView的`draw`过程用代码描述大致如下：

```java
public void draw(Canvas c) {
	ItemDecoration.onDraw();
	dispatchDraw(canvas); //绘制子View 
	ItemDecoration.onDrawOver();
}
```

因此如果要在RecyclerView更上层显示一个控件，是不需要另外新写一个独立于RecyclerView的控件的，只需要复写ItemDecoration.onDrawOver()即可。

### 0x01 复用原理

上面一节整体过了一遍代码流程，下面着重总结下复用原理。
`Recycler`这个类专门用来复用管理，考虑如下场景：

#### 0. RecyclerView滚动时：

列表里的数据集可能会超出一屏，而每次只会绘制显示在屏幕前台的子View。

当RecyclerView滚动时，调用路线依次是：
```java
onTouchEvent()
scrollByInternal(int x, int y, MotionEvent ev)
consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
scrollBy(dy, recycler, state);
final int consumed = freeScroll + fill(recycler, mLayoutState, state, false);
layoutChunk(recycler, state, layoutState, layoutChunkResult);
```
每次滚动都会触发layout -> draw这个过程。
如果每次这个过程都创建新View的话，会相当耗费资源，Recycler作了复用处理：

对于滑出屏幕的View，会添加到mCachedViews或RecyclerViewPool
对于滑入屏幕的View，会从mCachedViews或RecyclerViewPool中获取并复用
对于既没滑入也没滑出屏幕的View，会先从原来的mAttachScrap/mChangedScrap中根据position或者itemId去查找，没找到的情况下才会依次从其他3级缓存找。
如果没有找到才会调用mAdapter.createViewHolder()进行创建。


#### 1. 数据集发生变化时：

数据集的变化分为整体变化和局部变化。

（1）整体变化的调用路线：
```java
public final void notifyDataSetChanged() {
    mObservable.notifyChanged();
}
private class RecyclerViewDataObserver extends AdapterDataObserver {
    @Override
    public void onChanged() {
        mState.mStructureChanged = true;
        setDataSetChangedAfterLayout();
        requestLayout();
    }
	...
}
```
（2）局部变化（增删查改）的调用：
```java
notifyItemChanged
notifyItemRangeChanged
notifyItemInserted
notifyItemRangeInserted
notifyItemMoved
notifyItemRemoved
notifyItemRangeRemoved
```

以notifyItemInserted为例，调用路线如下：
```java
public final void notifyItemInserted(int position) {
    mObservable.notifyItemRangeInserted(position, 1);
}

public void notifyItemRangeInserted(int positionStart, int itemCount) {
    for (int i = mObservers.size() - 1; i >= 0; i--) {
        mObservers.get(i).onItemRangeInserted(positionStart, itemCount);
    }
}

public void onItemRangeInserted(int positionStart, int itemCount) {
    assertNotInLayoutOrScroll(null);
    if (mAdapterHelper.onItemRangeInserted(positionStart, itemCount)) {
        triggerUpdateProcessor();
    }
}
void triggerUpdateProcessor() {
    requestLayout();
}
```
最终都调到requestLayout()，进行再次布局。

#### 2. 重新设置数据适配器Adapter：

调用路线如下：
```java
public void setAdapter(Adapter adapter) {
    // bail out if layout is frozen
    setLayoutFrozen(false);
    setAdapterInternal(adapter, false, true);
    requestLayout();
}

private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious,
            boolean removeAndRecycleViews) {
    mLayout.removeAndRecycleAllViews(mRecycler);
    mLayout.removeAndRecycleScrapInt(mRecycler);
    mRecycler.clear();
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
}

void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,
        boolean compatibleWithPrevious) {
    clear();
    getRecycledViewPool().onAdapterChanged(oldAdapter, newAdapter, compatibleWithPrevious);
}

public void clear() {
    mAttachedScrap.clear();
    recycleAndClearCachedViews();
}

void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,
        boolean compatibleWithPrevious) {
    if (oldAdapter != null) {
        detach();
    }
    if (!compatibleWithPrevious && mAttachCount == 0) {
        clear();
    }
    if (newAdapter != null) {
        attach(newAdapter);
    }
}
```

基本意思就是：
由于RecyclerViewPool可以同时供多个RecyclerView共用，每个RecyclerView都对应有一个Adapter。
当只有一个RecyclerView使用RecyclerViewPool时，调用setAdapter会清空4级缓存里的所有数据，包括mAttachScrap/mChangedScrap，mCachedViews，RecyclerViewPool等。即设置了新的Adapter后，旧的ViewHolder缓存数据相当于作废。
当多个RecyclerView共用RecyclerViewPool时，调用setAdapter会将原来在前台显示的子View和原来前3级缓存里的数据都加到RecyclerViewPool进行复用，然后前3级缓存清空，子View清空。

这个地方可以这么理解：如果几个RecyclerView共用一个时，其中一个RecyclerView的Adapter被替换时，剩下的那些RecyclerViewPool的缓存数据源还必须存在，不能说因为其中一个变化，影响别人。
如果只有一个RecyclerView使用，那直接全部清空，重新来一遍就行了。


### 0x02 其他细节

count区别：
getChildCount() 指当前显示在前台的View个数
getItemCount()  指全部数据集大小

position区别：
getLayoutPosition() 站在LayoutManager角度
getAdapterPosition() 站在Aadpter角度

Item的5种基本状态：
```java
PERSISTENT: animateMove
REMOVED: animateRemove
ADDED: animateAdd
DISAPPEARING
APPEARING
```

ViewHolder的具体状态，通过flag标记：

```java
static final int FLAG_BOUND = 1 << 0;
static final int FLAG_UPDATE = 1 << 1;
static final int FLAG_INVALID = 1 << 2;
static final int FLAG_REMOVED = 1 << 3;
static final int FLAG_NOT_RECYCLABLE = 1 << 4;
static final int FLAG_RETURNED_FROM_SCRAP = 1 << 5;
static final int FLAG_CHANGED = 1 << 6;
static final int FLAG_IGNORE = 1 << 7;
static final int FLAG_TMP_DETACHED = 1 << 8;
static final int FLAG_ADAPTER_POSITION_UNKNOWN = 1 << 9;
```

本文暂时就记录这么多，更多内容请期待下篇。

#### 参考链接：
[https://www.jianshu.com/p/a9f42289fd04](RecyclerView源码解析) RecyclerView源码解析
[https://dev.qq.com/topic/5811d3e3ab10c62013697408](Android ListView与RecyclerView对比浅析--缓存机制) Android ListView与RecyclerView对比浅析--缓存机制

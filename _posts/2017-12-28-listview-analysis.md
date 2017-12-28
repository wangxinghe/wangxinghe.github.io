---
layout: post
comments: true
title: "ListView源码分析"
description: "ListView源码分析"
category: android
tags: [Android]
---


上篇分析了下`RecyclerView`源码，本文继续讲ListView源码。

<!--more-->

本文目录如下：	
0x00 基本代码结构	
（1）`measure`过程		
（2）`layout`过程	
（3）`draw`过程	
0x01 复用原理	
（1）ListView滚动时		
（2）数据集发生变化时		
（3）重新设置数据适配器Adapter	
0x02 总结	

### 0x00 基本代码结构

#### 0. `measure`过程

```java
@Override
public void onMeasure() {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);
    
    //宽度
    if (widthMode == MeasureSpec.UNSPECIFIED) {
        widthSize = mListPadding.left + mListPadding.right + childWidth +
                getVerticalScrollbarWidth();
    } else {
        widthSize |= (childState&MEASURED_STATE_MASK);
    }

	//高度。如果是UNSPECIFIED，则设置为itemHeight + ListPadding，如果是AT_MOST，则取Min(count * (itemHeight + dividerHeight), heightSize)
	if (heightMode == MeasureSpec.UNSPECIFIED) {
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;
	} else if (heightMode == MeasureSpec.AT_MOST) {
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
	}
	
	setMeasuredDimension(widthSize , heightSize); 
}
```

#### 1. `layout`过程

`onLayout`关键代码如下：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    super.onLayout(changed, l, t, r, b);
    final int childCount = getChildCount();
    if (changed) {
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).forceLayout();
        }
        mRecycler.markChildrenDirty();
    }
    ...
	layoutChildren();
	...
}
```
当数据发生变化时，首先界面上显示的子View重新layout，然后调用layoutChildren()布局子View。

`layoutChildren()`代码如下：

```java
@Override
protected void layoutChildren() {
	invalidate();
    ...
    final int childrenTop = mListPadding.top;
    final int childrenBottom = mBottom - mTop - mListPadding.bottom;
    final int childCount = getChildCount();
	...
    boolean dataChanged = mDataChanged;
    if (dataChanged) {
        handleDataChanged();
    }
	...
    // Pull all children into the RecycleBin.
    // These views will be reused if possible
    final int firstPosition = mFirstPosition;
    final RecycleBin recycleBin = mRecycler;
    if (dataChanged) {
        for (int i = 0; i < childCount; i++) {
            recycleBin.addScrapView(getChildAt(i), firstPosition+i);
        }
    } else {
        recycleBin.fillActiveViews(childCount, firstPosition);
    }
	...
    // Clear out old views
    detachAllViewsFromParent();
    recycleBin.removeSkippedScrap();
	...
	//填充：fillUp() / fillFromTop() / fillSpecific();
	if (childCount == 0) {
        if (!mStackFromBottom) {
            final int position = lookForSelectablePosition(0, true);
            setSelectedPositionInt(position);
            sel = fillFromTop(childrenTop);
        } else {
            final int position = lookForSelectablePosition(mItemCount - 1, false);
            setSelectedPositionInt(position);
            sel = fillUp(mItemCount - 1, childrenBottom);
        }
    } else {
        if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
            sel = fillSpecific(mSelectedPosition,
                    oldSel == null ? childrenTop : oldSel.getTop());
        } else if (mFirstPosition < mItemCount) {
            sel = fillSpecific(mFirstPosition,
                    oldFirst == null ? childrenTop : oldFirst.getTop());
        } else {
            sel = fillSpecific(0, childrenTop);
        }
    }
	...
    // Flush any cached views that did not get reused above
    recycleBin.scrapActiveViews();
}
```
上面代码拆分成3个部分来看：	
（1）填充子View前	
（2）填充子View	
（3）填充子View后	

##### （1）填充子View前：	
（1）首次Layout，此时childCount == 0，不需要做特殊处理。		
（2）非首次Layout：		
若数据发生变化：如果是瞬态子View，将屏幕上显示的子View添加到mTransientStateViewsById/mTransientStateViews/mSkippedScrap；如果是稳态子View，将屏幕上显示的子View添加到mCurrentScrap/mScrapViews列表。		
若数据未变化：将屏幕上显示的子View添加到mActiveViews列表。	

##### （2）填充子View：	
（1）首次Layout，若填充方向为从下往上（mStackFromBottom = true），则调用fillUp(mItemCount - 1, childrenBottom)，若填充方向为从上往下（mStackFromBottom = false），则调用fillFromTop(childrenTop)		
（2）非首次Layout，则从指定位置先依次往上往下填充子View，见fillSpecific(int position, int top)		

三种填充方式：	
`fillUp()`   从下往上填充		
`fillFromTop()`   从上往下填充		
`fillSpecific(position)`  从某个位置先往上填充再往下填充		

以`fillFromTop(int nextTop)`为例，主要代码如下：

```java
private View fillFromTop(int nextTop) {
    mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
    mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
    if (mFirstPosition < 0) {
        mFirstPosition = 0;
    }
    return fillDown(mFirstPosition, nextTop);
}
```
继续关注`fillDown(mFirstPosition, nextTop)`，

```java
private View fillDown(int pos, int nextTop) {
    View selectedView = null;

    int end = (mBottom - mTop);
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
        end -= mListPadding.bottom;
    }

    while (nextTop < end && pos < mItemCount) {
        // is this the selected item?
        boolean selected = pos == mSelectedPosition;
        View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

        nextTop = child.getBottom() + mDividerHeight;
        if (selected) {
            selectedView = child;
        }
        pos++;
    }

    setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
    return selectedView;
}
```
从mFirstPosition位置从上往下填充子View直至屏幕填满，子View的获取方法见`makeAndAddView(pos, nextTop, true, mListPadding.left, selected)`，主要代码如下：

```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
        boolean selected) {
    View child;

    if (!mDataChanged) {
        // Try to use an existing view for this position
        child = mRecycler.getActiveView(position);
        if (child != null) {
            // Found it -- we're using an existing child
            // This just needs to be positioned
            setupChild(child, position, y, flow, childrenLeft, selected, true);

            return child;
        }
    }

    // Make a new view for this position, or convert an unused view if possible
    child = obtainView(position, mIsScrap);

    // This needs to be positioned and measured
    setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

    return child;
}
```
数据未变化时，先从mActiveViews中复用。若数据发生变化或者从mActiveViews中未获取到子View，则调用`obtainView(position, mIsScrap)`获取，获取到之后再调用`setupChild`来measure、layout子View。

`obtainView(position, mIsScrap)`按照优先级依次从transientViews、mScrapViews中根据position匹配View，并将其作为convertView，然后调用mAdapter.getView(position, convertView, this)绑定数据

```java
View obtainView(int position, boolean[] isScrap) {
    final View transientView = mRecycler.getTransientStateView(position);
    if (transientView != null) {
        final LayoutParams params = (LayoutParams) transientView.getLayoutParams();
        // If the view type hasn't changed, attempt to re-bind the data.
        if (params.viewType == mAdapter.getItemViewType(position)) {
            final View updatedView = mAdapter.getView(position, transientView, this);
        }
        return transientView;
    }
    ...
    final View scrapView = mRecycler.getScrapView(position);
    final View child = mAdapter.getView(position, scrapView, this);
    ...
	return child;
}
```

`setupChild`做的事情就是重新调整子View的位置，会涉及到子View的measure、layout操作

```java
//Add a view as a child and make sure it is measured (if necessary) and positioned properly.
private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft, boolean selected, boolean recycled) {
    ...
    int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
            mListPadding.left + mListPadding.right, p.width);
    int lpHeight = p.height;
    int childHeightSpec;
    if (lpHeight > 0) {
        childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
    } else {
        childHeightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
    }
    child.measure(childWidthSpec, childHeightSpec);
	...
    final int childRight = childrenLeft + w;
    final int childBottom = childTop + h;
    child.layout(childrenLeft, childTop, childRight, childBottom);
}
```

##### （3）填充子View后：
调用`mRecycler.scrapActiveViews()`，将其余还存在于mActiveViews中的子View移到mScrapViews中。

`RecycleBin`负责layout过程的子View复用机制，复用的详细过程可以跳到后面小节的`0x01 复用原理`。

#### 2. `draw`过程

主要代码如下：

```java
@Override
public void draw(Canvas canvas) {
    super.draw(canvas);
    //绘制上下边缘
	mEdgeGlowTop.draw(canvas);
	mEdgeGlowBottom.draw(canvas);
}

@Override
protected void dispatchDraw(Canvas canvas) {
	if (!mStackFromBottom) {//从上往下
		//绘制OverscrollHeader或header divider
		drawOverscrollHeader(canvas, overscrollHeader, bounds) or drawDivider(canvas, bounds, -1);
		//绘制元素间的divider
		for (int i = 0; i < count; i++) {
			drawDivider(canvas, bounds, i);
		}
		//绘制OverscrollFooter
		drawOverscrollFooter(canvas, overscrollFooter, bounds);
	} else {//从下往上
		//绘制OverscrollHeader
		drawOverscrollHeader(canvas, overscrollHeader, bounds);
		//绘制元素间的divider
		for (int i = 0; i < count; i++) {
			drawDivider(canvas, bounds, i);
		}
		//绘制OverscrollFooter或footer divider
		drawOverscrollFooter(canvas, overscrollFooter, bounds) or drawDivider(canvas, bounds, -1);
	}

	//drawSelector等
	
	//绘制子View
	super.dispatchDraw(canvas);
}
```
绘制的主要逻辑集中在`dispatchDraw(Canvas canvas)`，绘制顺序是：		
（1）绘制OverscrollHeader或header divider		
（2）绘制元素间的divider		
（3）绘制OverscrollFooter		
（4）绘制子View		

### 0x01 复用原理

`RecycleBin`数据结构如下：

```java
class RecycleBin {
	private View[] mActiveViews;
	private ArrayList<View>[] mScrapViews;
	private int mViewTypeCount;
	ArrayList<View> mCurrentScrap;
	private SparseArray<View> mTransientStateViews;
	private LongSparseArray<View> mTransientStateViewsById;
}
```

`mActiveViews`：子View数组。若数据集没发生变化，在布局子View之前，先将当前界面显示的子View添加到mActiveViews。mActiveViews生命周期只存在于onLayout过程。	
`mScrapViews`：key为viewtype，value为ArrayList< View >的键值对。在子View布局前，若数据集发生变化，将当前界面显示的子View添加到mScrapViews；在子View布局之后，将mAcviteViews中剩余的子View添加到`mScrapViews`。		
`mTransientStateViews/mTransientStateViewsById`：瞬态View列表的键值对，key为position/itemId，value为List< View >。如果hasStableId=true，则key为itemId。		
填充子View的过程中，子View复用逻辑：	
若数据集没变化，则先从mActiveViews中匹配，然后直接布局子View，子VIew不需要重新绑定数据。	
若数据集发生变化，则从transientView或mScrapViews中匹配，并将匹配到的子View作为convertView，重新绑定数据后，再布局子View。如果没匹配到子View，则在getView中新创建一个子View。		
首次Layout时，mActiveViews和mScrapViews都为空。	

继续结合以下场景分析：	
（1）ListView滚动时		
（2）数据集发生变化时		
（3）重新设置数据适配器Adapter	

#### 0. ListView滚动时：	
ListView滚动时，调用路线依次为：	
```java
onTouchEvent()
scrollIfNeeded()
trackMotionScroll()
fillGap(down);
fillDown(mFirstPosition + count, startOffset); 或fillUp()  注：往上滑是fillUp，往下滑是fillDown
makeAndAddView(pos, nextTop, true, mListPadding.left, selected);
child = obtainView(position, mIsScrap);
mAdapter.getView(position, transientView, this);
```
对于滑出屏幕的View，会添加到mScrapViews，见`trackMotionScroll`代码。	
对于滑入屏幕的View，会从mScrapViews或mTransientStateViews中匹配出convertView，然后调用getView进行数据绑定；若没匹配到则调用getView创建View并绑定数据。	
对于既没滑入也没滑出屏幕的View，且数据集未发生变化时，直接从mActiveViews中获取现成的View进行展现，不需要进行数据绑定。反之若数据集发生变化则还是走mScrapViews、getView()复用那一套逻辑。	

```java
boolean trackMotionScroll(int deltaY, int incrementalDeltaY) {
	...
	final boolean down = incrementalDeltaY < 0;
    if (down) {//从下向上滑动时
        int top = -incrementalDeltaY;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            top += listPadding.top;
        }
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            if (child.getBottom() >= top) {
                break;
            } else {
                count++;
                int position = firstPosition + i;
                if (position >= headerViewsCount && position < footerViewsStart) {
                    // The view will be rebound to new data, clear any
                    // system-managed transient state.
                    child.clearAccessibilityFocus();
                    mRecycler.addScrapView(child, position);
                }
            }
        }
    } else {//从上向下滑动时
        int bottom = getHeight() - incrementalDeltaY;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            bottom -= listPadding.bottom;
        }
        for (int i = childCount - 1; i >= 0; i--) {
            final View child = getChildAt(i);
            if (child.getTop() <= bottom) {
                break;
            } else {
                start = i;
                count++;
                int position = firstPosition + i;
                if (position >= headerViewsCount && position < footerViewsStart) {
                    // The view will be rebound to new data, clear any
                    // system-managed transient state.
                    child.clearAccessibilityFocus();
                    mRecycler.addScrapView(child, position);
                }
            }
        }
    }
	...
}
```

#### 1. 数据集发生变化时：

以`class BaseAdapter implements ListAdapter`为例，当数据集发生变化时，会调用mAdapter.notifyDataSetChanged()或mAdapter.notifyDataSetInvalidated()。

调用路线依次是：

`BaseAdapter.notifyDataSetChanged() -> DataSetObservable.notifyChanged() -> DataSetObserver.onChanged() -> AdapterDataSetObserver.onChanged() -> requestLayout()`

所以还是会走layout那一套逻辑，整体刷新ListView。ListView没直接提供局部刷新的方法。

#### 2. 重新设置数据适配器Adapter：

重新设置数据适配器时，会调到`requestLayout()`，重新走一遍layout逻辑。	
```java
@Override
public void setAdapter(ListAdapter adapter) {
	mAdapter = adapter;
	...
	requestLayout();
}
```

### 0x02 其他

getChildCount() vs. mAdapter.getCount()		
getChildCount()屏幕上显示的子View个数		
mAdapter.getCount() Adapter数据集的个数		


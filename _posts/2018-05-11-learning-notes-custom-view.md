---
layout: post
comments: true
title: "Android学习笔记——自定义View"
description: "Android学习笔记——自定义View"
category: Android
tags: [Android]
---

**1、自定义View**    
**（1）整体结构及工作流程**    
**（2）measure**    
**（3）layout**    
**（4）draw**    
**2、measure过程**    
**（1）MeasureSpec(measureSpec/mode/size)**    
**（2）父View宽高的测量方法**    
**（3）子View宽高的测量方法**    
**3、layout过程**    
**（1）left/top/right/bottom含义**    
**（2）left/top/right/bottom计算**    
**4、draw过程**    
**（1）绘制顺序**    
**5、源码举例**    
**（1）LinearLayout**    
**（2）ListView**    
**（3）ScrollView**    
**6、参考文档**

<!--more-->

### 1、自定义View    

#### （1）整体结构及工作流程    

**Activity、Window、DecorView之间的关系：**    

![](/image/2018-05-11-learning-notes-custom-view/Window.svg)


`Activity`：相当于一个Controller，具备生命周期。    

`Window`：相当于窗口，是视图的承载器，PhoneWindow是唯一实现类。    

`DecorView`：顶级View，是一个Framelayout，包含StatusBar、TitleBar＋ContentView、NavigationBar三个部分。    
StatusBar是状态栏；    
TitleBar对应各种ActionBar；    
ContentView对应R.id.content，setContentView设置的View被添加到R.id.content对应的View上，可通过findViewById(android.id.content)得到ContentView，findViewById(android.id.content).getChildAt(0)得到设置进去的View；    
NavigationBar是虚拟按键。        

`ViewRoot`：实现类是ViewRootImpl，它是连接WindowManager和DecorView的纽带，View的三大流程是通过ViewRoot来完成的。    


**View的工作流程：**    

![](/image/2018-05-11-learning-notes-custom-view/view_render.png)    


1. 最先从ViewRoot.performTraversals()方法开始    
2. DecorView的绘制：调用DecorView的performMeasure，performLayout，performDraw三个方法。DecorView的measure顺序：performMeasure -> measure -> onMeasure；DecorView的layout顺序：performLayout -> layout -> onLayout；DecorView的draw顺序：performDraw -> draw -> onDraw。然后onMeasure／onLayout/onDraw又会调用child.measure()/child.layout()/child.onDraw()将这个过程传给child。        
3. 子View的绘制：measure过程 measure -> onMeasure；layout -> onLayout；draw -> onDraw    
4. 依次往下传递。

measure过程：得到所有View体系的宽高    
layout过程：得到所有View的坐标    
draw过程：绘制所有View

整个过程的简略版代码，以LinearLayout为例：    

#### （2）measure    

`measure(int widthMeasureSpec, int heightMeasureSpec)`做的事情：    

1. 保存widthMeasureSpec和heightMeasureSpec    
2. 调用onMeasure(widthMeasureSpec, heightMeasureSpec)方法

`onMeasure(int widthMeasureSpec, int heightMeasureSpec)`做的事情：    

1. 调用child.measure(childWidthMeasureSpec, childHeightMeasureSpec)测量子View的宽高    
2. 得到当前View的总宽高，并调用setMeasuredDimension(totalWidth, totalHeight)保存总宽高

`setMeasuredDimension(int measuredWidth, int measuredHeight)`做的事情：    

1. 保存当前View的总宽高 

具体代码如下：

    //LinearLayout
    
    @Override
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        ...
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
    
    void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
        int totalWidth = ...;
        int totalHeight = 0;
        for (int i = 0; i < getChildCount(); i++) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
            totalHeight += child.getMeasureHeight();
        }
        ...
        setMeasuredDimension(totalWidth, totalHeight);
    }

    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

    private void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
    }

#### （3）layout    

`layout(int l, int t, int r, int b)`做的事情：    

1. 保存当前View的坐标    
2. 调用onLayout(changed, l, t, r, b)方法    

`onLayout(boolean changed, int l, int t, int r, int b)`做的事情：    

1. 计算子View的坐标    
2. 调用child.layout(childLeft, childTop, childRight, childBottom)将子View的坐标传给子View

具体代码如下：

    //LinearLayout
    
    @Override
    public void layout(int l, int t, int r, int b) {
        ...        
        mLeft = l;
        mTop = t;
        mRight = r;
        mBottom = b;
        ...
        onLayout(changed, l, t, r, b);    
        ...
    ｝
    
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
    
    void layoutVertical(int left, int top, int right, int bottom) {
        for (int i = 0; i < getChildCount(); i++) {
            ...
            childLeft = ...;
            childTop = ...;
            childRight = childLeft + child.getMeasuredWidth();            
            childBottom = childTop + child.getMeasuredHeight();

            child.layout(childLeft, childTop, childRight, childBottom);
        }
    
    }

#### （4）draw    

`draw(Canvas canvas)`做的事情：    

1. 调用drawBackground(canvas)绘制背景    
2. save canvas‘ layers
3. 调用onDraw(canvas)绘制内容    
4. 调用dispatchDraw(canvas)绘制子View    
5. 绘制fade effect和restore layers    
6. 调用onDrawScrollBars(canvas)绘制滚动条

`onDraw(canvas)`做的事情：    

1. 绘制当前View的内容

`dispatchDraw(Canvas canvas)`做的事情：    

1. 调用child.draw(canvas)绘制子View

具体代码如下：

    //LinearLayout
    
    public void draw(Canvas canvas) {
        // Step 1, draw the background, if needed
        drawBackground(canvas);
        
        // Step 2, save the canvas' layers（非必需）
        
        // Step 3, draw the content
        onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers（非必需）
        
        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);      
    }

    @Override
    protected void dispatchDraw(Canvas canvas) {
        for (int i = 0; i < getChildCount(); i++) {
            drawChild(canvas, child);
        }
    }
    
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }


### 2、measure过程    
    
#### （1）MeasureSpec(measureSpec/mode/size)    

`measureSpec`：32位的int值，高2位代表mode，低30位 代表size。    

`mode`是测量模式，包括：    
`UNSPECIFIED`：父View对当前View没啥限制，当前View可以是任意大小        
`EXACTLY`：父View决定了当前View的精确大小        
`AT_MOST`：当前View最高可以到某个大小    

`size`指某种测量模式下的规格大小。

    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        public static final int AT_MOST     = 2 << MODE_SHIFT;

        public static int makeMeasureSpec(int size, int mode) {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }

        public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }

        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }
    

如果不复写onMeasure的话，模式实现是：

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }


#### （2）父View宽高的测量方法    

#### （3）子View宽高的测量方法    

    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

childMeasureSpec和parentMeasureSpec的关系：    

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }


### 3、layout过程    

#### （1）left/top/right/bottom含义    

#### （2）left/top/right/bottom计算    

### 4、draw过程    

#### （1）绘制顺序    

### 5、源码举例    

#### （1）LinearLayout    

#### （2）ListView    

#### （3）ScrollView    

### 6、参考文档
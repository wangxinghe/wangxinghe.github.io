---
layout: post
comments: true
title: "View事件分发及消费源码分析"
description: "View事件分发及消费源码分析"
category: android
tags: [Android]
---


## 1.概述
学姐觉得View事件这块是特别烧脑的，看了好久，才自认为看明白。中间上网查了下singwhatiwanna粉丝的读书笔记，有种茅塞顿开的感觉。

**很重要的学习方法：化繁为简，只抓重点。**

源码一坨，不要指望每一行代码都看懂。首先是没必要，其次大量非关键代码会让你模糊真正重要的部分。
以下也只是学姐的学习成果，各位同学要想理解深刻，还需要自己亲自去看源码。

<!--more-->

## 2.源码分析

由于源码实在太长，而且也不容易看懂，学姐这里就不贴出来了，因为没必要。

以下是学姐简化版源码。

#### （1）ViewGroup.dispatchTouchEvent(event)

    boolean dispatchTouchEvent(MotionEvent event) {
        int action = event.getAction();
        
        //判断ViewGroup是否拦截touch事件。当为ACTION_DOWN或者找到能够接收touch事件的子View
        时，由onInterceptTouchEvent(event)决定是否拦截。其他情况，即ACTION_MOVE/ACTION_UP且
        没找到能够接收touch事件的子View时，直接拦截。
        boolean intercepted;
        if (action == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
            intercepted = onInterceptTouchEvent(event);
        } else {
            intercepted = true;
        }

        //如果ViewGroup不拦截touch事件。在ACTION_DOWN时遍历所有子View，查找能够接收touch事件的
        子View。如果找到则设置mFirstTouchTarget，并跳出循环。
        boolean alreadyDispatchedToNewTouchTarget =  false;
        if (!intercepted) {
            if (action == MotionEvent.ACTION_DOWN) {
               for (int i = childrenCount - 1; i >= 0; i--) {
                    if (!canViewReceivePointerEvents(child) ||
                     !isTransformedTouchPointInView(x, y, child, null)) {
                         continue;
                    }
                    if (dispatchTransformedTouchEvent(event, child)) {
                       //找到mFirstTouchTarget
                       newTouchTarget = addTouchTarget(child);
                       alreadyDispatchedToNewTouchTarget = true;
                       break;
                    }
                 }
             }
        }

        //事件下发及消费。如果没找到能够接收touch事件的子View，则由ViewGroup自己处理及消费。
        如果找到能够接收touch事件的子View，则由子View递归处理touch事件及消费。
        boolean handled = false;
        if (mFirstTouchTarget == null) {
            handled = dispatchTransformedTouchEvent(event, null);
        } else {
            if (alreadyDispatchedToNewTouchTarget) {
                handled = true;
            } else {
                while (touchTarget) {
                    handled = dispatchTransformedTouchEvent(event, child);
                }
            }
        }

        return handled;
    }


    //ViewGroup事件下发。如果无接收touch事件的子View，则由ViewGroup的父类（即View）下发touch事件
    如果child非空，则交由子View下发touch事件，子View可以是ViewGroup或View。
    boolean dispatchTransformedTouchEvent(MotionEvent event, View child) {
       boolean handled;
       if (child == null) {
            handled = super.dispatchTouchEvent(event);
       } else {
            handled = child.dispatchTouchEvent(event);
       }
       return handled;
    }

#### （2）View.dispatchTouchEvent(event)

    //View的Touch事件分发。当外部设置了mOnTouchListener时，先交由mOnTouchListener.onTouch(event)消费。
    若未消费，则交给View的onTouchEvent(event)消费。onTouchEvent的实现是，如果设置了mOnClickListener，
    则执行mOnClickListener.onClick()点击事件。返回值为true，表示消费，否则未消费。
    boolean dispatchTouchEvent(MotionEvent event) {
       boolean result = false;
       if (mOnTouchListener != null && mOnTouchListener.onTouch(this, event)) {
             result = true;
       }
       if (!result && onTouchEvent(event)) {
            result = true;
       }
       return result;
    }


    boolean onTouchEvent(MotionEvent event) {
       performClick();
    }

## 3.总结

总结下ViewGroup的事件分发及消费过程：

0. 整个过程包括3个部分：`判断是否拦截 -> 查找接收touch事件的子View -> 事件下发及消费`

1. 判断是否拦截：

    (1) ACTION_DOWN 或者 非ACTION_DOWN且找到接收touch事件的子View时，由onInterceptTouchEvent(event)决定是否拦截
    
    (2) 非ACTION_DOWN，且未找到接收touch事件的子View时，标明需要拦截touch事件
    
      这里解释下，影响ViewGroup是否能拦截touch事件有2个因素：是否 找到了接收touch事件的子View 和 onInterceptTouchEvent(event). 而查找接收touch事件的子View这一过程只需要在ACTION_DOWN的时候确定好就行。如果ACTION_DOWN的时候没找到，那么ACTION_MOVE和ACTION_UP肯定也找不到，因此touch事件直接被ViewGroup拦截。如果找到了接收touch事件的子View，那么ACTION_MOVE和ACTION_UP情况下还是要检查下ViewGroup的onInterceptTouchEvent(event)，看下是否拦截。
      
2. 查找接收touch事件的子View：

    (1) 两种情况下查找：ACTION_DOWN且ViewGroup不拦截的情况下。
    
    (2) 查找方法：遍历所有子View，如果touch事件的xy坐标在该ViewGroup的某个子View范围内，则针对该子View执行递归分发touch事件操作，如果找到有子View处理touch事件(return true)，则跳出循环。
    
    这里解释下查找条件。查找接收touch事件的子View，显然只需要ACTION_DOWN情况下即可，没必要ACTION_MOVE和ACTION_UP都检查，否则重复操作。如果ViewGroup都已经拦截了，显然不需要再去考虑子View怎么样了。
    
3. 事件下发及消费：

    (1)两种情况：ViewGroup下发及消费 或者 ViewGroup的子View下发及消费

    (2)如果经过以上两步，没找到接收Touch事件的子View，那么由ViewGroup进行下发及消费，下发及调用流程是：ViewGroup.dispatchTouchEvent -> View.dispatchTouchEvent -> mOnTouchListener.onTouch -> onTouchEvent -> onClick

    (3)如果找到接收touch事件的子View，则针对该子View执行touch事件递归下发及消费的操作

**补充：**

(1) 源码中，mFirstTouchEvent表示接收touch事件的子View

(2) 步骤2和3，都有执行dispatchTransformedTouchEvent(event, child)的操作，步骤2中只是为了查找接收touch事件的子View，步骤3主要目的是进行事件分发及消费。如果步骤2中针对某个子View已经执行了该方法，则步骤3中不再重复执行。个人理解，不知道是否有误。

## 4.结论

(1) 回调方法

ViewGroup：dispatchTouchEvent -> onInterceptTouchEvent -> onTouchEvent

View: dispatchTouchEvent -> onTouch

(2) 调用顺序

Action执行顺序：ACTION_DOWN -> ACTION_MOVE -> ACTION_UP

ViewGroup：dispatchTouchEvent -> onInterceptTouchEvent -> onTouchEvent（）

View: dispatchTouchEvent -> onTouchEvent

事件分发传递顺序：Parent View －> Child View

ViewGroup1.dispatchTouchEvent -> ViewGroup2.dispatchTouchEvent -> View3.dispatchTouchEvent
（紧跟着是View3.onTouchEvent）

事件消费传递顺序：Child View -> Parent View

View3.onTouchEvent -> ViewGroup2.onTouchEvent -> ViewGroup1.onTouchEvent

个人理解这种传递顺序，是由dispatchTransformedTouchEvent引起的，这里就是递归调用，整个事件的入口就是ViewGroup.dispatchTouchEvent.

内容预告：下篇文章会补充Activity及touch事件处理的相关示例说明，请大家继续关注～


## 欢迎大家关注我的公众号：学姐的IT专栏

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)
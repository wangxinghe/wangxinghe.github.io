---
layout: post
comments: true
title: "Input事件分发"
description: "Input事件分发"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析

<!--more-->

当手指触摸到屏幕时, 屏幕硬件一行行不断地扫描每个像素点, 获取到触摸事件后, 从底层产生中断上报.  

Native底层细节可参考[http://gityuan.com/2016/12/31/input-ipc/](http://gityuan.com/2016/12/31/input-ipc/),  本文暂不讨论, 直接从Java层分析起.  

## 1.ViewRootImpl
[ -> frameworks/base/core/java/android/view/ViewRootImpl.java ]

### 1.1 WindowInputEventReceiver

#### 1.1.1 dispatchInputEvent

	public abstract class InputEventReceiver {
		...
	    // Called from native code.
	    private void dispatchInputEvent(int seq, InputEvent event) {
	        mSeqMap.put(event.getSequenceNumber(), seq);
	        onInputEvent(event);
	    }
	    ...
	}

#### 1.1.2 onInputEvent

	final class WindowInputEventReceiver extends InputEventReceiver {
	    public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
	        super(inputChannel, looper);
	    }

	    @Override
	    public void onInputEvent(InputEvent event) {
	        List<InputEvent> processedEvents = mInputCompatProcessor.processInputEventForCompatibility(event);
        
	        if (processedEvents != null) {
	            if (processedEvents.isEmpty()) {
	                // InputEvent consumed by mInputCompatProcessor
	                finishInputEvent(event, true);
	            } else {
	                for (int i = 0; i < processedEvents.size(); i++) {
	                    enqueueInputEvent(
	                            processedEvents.get(i), this,
	                            QueuedInputEvent.FLAG_MODIFIED_FOR_COMPATIBILITY, true);
	                }
	            }
	        } else {
	            enqueueInputEvent(event, this, 0, true);
	        }
	    }
		...
	}

### 1.2 enqueueInputEvent

	void enqueueInputEvent(InputEvent event, InputEventReceiver receiver, int flags, boolean processImmediately) {
	    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

	    // Always enqueue the input event in order, regardless of its time stamp.
	    // We do this because the application or the IME may inject key events
	    // in response to touch events and we want to ensure that the injected keys
	    // are processed in the order they were received and we cannot trust that
	    // the time stamp of injected events are monotonic.
	    QueuedInputEvent last = mPendingInputEventTail;
	    if (last == null) {
	        mPendingInputEventHead = q;
	        mPendingInputEventTail = q;
	    } else {
	        last.mNext = q;
	        mPendingInputEventTail = q;
	    }
	    mPendingInputEventCount += 1;

	    if (processImmediately) {
	        doProcessInputEvents();
	    } else {
	        scheduleProcessInputEvents();
	    }
	}

### 1.3 doProcessInputEvents

	void doProcessInputEvents() {
	    // Deliver all pending input events in the queue.
	    while (mPendingInputEventHead != null) {
	        QueuedInputEvent q = mPendingInputEventHead;
	        mPendingInputEventHead = q.mNext;
	        if (mPendingInputEventHead == null) {
	            mPendingInputEventTail = null;
	        }
	        q.mNext = null;

	        mPendingInputEventCount -= 1;

	        long eventTime = q.mEvent.getEventTimeNano();
	        long oldestEventTime = eventTime;
	        if (q.mEvent instanceof MotionEvent) {
	            MotionEvent me = (MotionEvent)q.mEvent;
	            if (me.getHistorySize() > 0) {
	                oldestEventTime = me.getHistoricalEventTimeNano(0);
	            }
	        }
	        mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

	        deliverInputEvent(q);
	    }

	    // We are done processing all input events that we can process right now
	    // so we can clear the pending flag immediately.
	    if (mProcessInputEventsScheduled) {
	        mProcessInputEventsScheduled = false;
	        mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
	    }
	}

### 1.4 deliverInputEvent

	private void deliverInputEvent(QueuedInputEvent q) {
	    if (mInputEventConsistencyVerifier != null) {
	        mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
	    }

	    InputStage stage;
	    if (q.shouldSendToSynthesizer()) {
	        stage = mSyntheticInputStage;
	    } else {
	        stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
	    }

	    if (q.mEvent instanceof KeyEvent) {
	        mUnhandledKeyManager.preDispatch((KeyEvent) q.mEvent);
	    }

	    if (stage != null) {
	        handleWindowFocusChanged();
	        stage.deliver(q);
	    } else {
	        finishInputEvent(q);
	    }
	}

### 1.5 ViewPostImeInputStage

#### 1.5.1 deliver

	abstract class InputStage {
	    private final InputStage mNext;

	    protected static final int FORWARD = 0;
	    protected static final int FINISH_HANDLED = 1;
	    protected static final int FINISH_NOT_HANDLED = 2;

	    /**
	     * Creates an input stage.
	     * @param next The next stage to which events should be forwarded.
	     */
	    public InputStage(InputStage next) {
	        mNext = next;
	    }

	    /**
	     * Delivers an event to be processed.
	     */
	    public final void deliver(QueuedInputEvent q) {
	        if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
	            forward(q);
	        } else if (shouldDropInputEvent(q)) {
	            finish(q, false);
	        } else {
	            apply(q, onProcess(q));
	        }
	    }
		...
	}

#### 1.5.2 onProcess

	/**
	 * Delivers post-ime input events to the view hierarchy.
	 */
	final class ViewPostImeInputStage extends InputStage {
	    public ViewPostImeInputStage(InputStage next) {
	        super(next);
	    }

	    @Override
	    protected int onProcess(QueuedInputEvent q) {
	        if (q.mEvent instanceof KeyEvent) {
	            return processKeyEvent(q);
	        } else {
	            final int source = q.mEvent.getSource();
	            if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
	                return processPointerEvent(q);
	            } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
	                return processTrackballEvent(q);
	            } else {
	                return processGenericMotionEvent(q);
	            }
	        }
	    }

	    @Override
	    protected void onDeliverToNext(QueuedInputEvent q) {
	        if (mUnbufferedInputDispatch
	                && q.mEvent instanceof MotionEvent
	                && ((MotionEvent)q.mEvent).isTouchEvent()
	                && isTerminalInputEvent(q.mEvent)) {
	            mUnbufferedInputDispatch = false;
	            scheduleConsumeBatchedInput();
	        }
	        super.onDeliverToNext(q);
	    }
		...
	}

#### 1.5.3 processPointerEvent

	final class ViewPostImeInputStage extends InputStage {
		...
		private int processPointerEvent(QueuedInputEvent q) {
	    	final MotionEvent event = (MotionEvent)q.mEvent;

	    	mAttachInfo.mUnbufferedDispatchRequested = false;
	    	mAttachInfo.mHandlingPointerEvent = true;
	    	boolean handled = mView.dispatchPointerEvent(event);
	    	maybeUpdatePointerIcon(event);
	    	maybeUpdateTooltip(event);
	    	mAttachInfo.mHandlingPointerEvent = false;
	    	if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
	        	mUnbufferedInputDispatch = true;
	        	if (mConsumeBatchedInputScheduled) {
	            	scheduleConsumeBatchedInputImmediately();
	        	}
	    	}
	    	return handled ? FINISH_HANDLED : FORWARD;
		}
		...
	}

## 2.DecorView
[ -> frameworks/base/core/java/com/android/internal/policy/DecorView.java ]

### 2.1 dispatchPointerEvent

DecorView继承ViewGroup, DecorView#dispatchPointerEvent 也就是 ViewGroup#dispatchPointerEvent

	public final boolean dispatchPointerEvent(MotionEvent event) {
	    if (event.isTouchEvent()) {
	        return dispatchTouchEvent(event);
	    } else {
	        return dispatchGenericMotionEvent(event);
	    }
	}

### 2.2 dispatchTouchEvent

	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
	    final Window.Callback cb = mWindow.getCallback();
	    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
	            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
	}

溯源`PhoneWindow#setCallback`可知, mCallback为Activity.  

## 3.Activity  
[ -> frameworks/base/core/java/android/app/Activity.java ]

### 3.1 dispatchTouchEvent

	/**
	 * Called to process touch screen events.  You can override this to
	 * intercept all touch screen events before they are dispatched to the
	 * window.  Be sure to call this implementation for touch screen events
	 * that should be handled normally.
	 *
	 * @param ev The touch screen event.
	 *
	 * @return boolean Return true if this event was consumed.
	 */
	public boolean dispatchTouchEvent(MotionEvent ev) {
	    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
	        onUserInteraction();
	    }
	    if (getWindow().superDispatchTouchEvent(ev)) {
	        return true;
	    }
	    return onTouchEvent(ev);
	}

    /**
     * Called when a touch screen event was not handled by any of the views
     * under it.  This is most useful to process touch events that happen
     * outside of your window bounds, where there is no view to receive it.
     *
     * @param event The touch screen event being processed.
     *
     * @return Return true if you have consumed the event, false if you haven't.
     * The default implementation always returns false.
     */
    public boolean onTouchEvent(MotionEvent event) {
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }

        return false;
    }

## 4.PhoneWindow
[ -> frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java ]

	@Override
	public boolean superDispatchTouchEvent(MotionEvent event) {
	    return mDecor.superDispatchTouchEvent(event);
	}

## 5.DecorView

	public boolean superDispatchTouchEvent(MotionEvent event) {
	    return super.dispatchTouchEvent(event);
	}

## 6.ViewGroup

### 6.1 dispatchTouchEvent

	@Override
	public boolean dispatchTouchEvent(MotionEvent ev) {
	    if (mInputEventConsistencyVerifier != null) {
	        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
	    }

	    // If the event targets the accessibility focused view and this is it, start
	    // normal event dispatch. Maybe a descendant is what will handle the click.
	    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
	        ev.setTargetAccessibilityFocus(false);
	    }

	    boolean handled = false;
	    if (onFilterTouchEventForSecurity(ev)) {
	        final int action = ev.getAction();
	        final int actionMasked = action & MotionEvent.ACTION_MASK;

	        // Handle an initial down.
	        if (actionMasked == MotionEvent.ACTION_DOWN) {
	            // Throw away all previous state when starting a new touch gesture.
	            // The framework may have dropped the up or cancel event for the previous gesture
	            // due to an app switch, ANR, or some other state change.
	            cancelAndClearTouchTargets(ev);
	            resetTouchState();
	        }

	        // Check for interception.
	        final boolean intercepted;
	        if (actionMasked == MotionEvent.ACTION_DOWN
	                || mFirstTouchTarget != null) {
	            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
	            if (!disallowIntercept) {
	                intercepted = onInterceptTouchEvent(ev);
	                ev.setAction(action); // restore action in case it was changed
		        } else {
	                intercepted = false;
	            }
	        } else {
	            // There are no touch targets and this action is not an initial down
	            // so this view group continues to intercept touches.
	            intercepted = true;
	        }

	        // If intercepted, start normal event dispatch. Also if there is already
	        // a view that is handling the gesture, do normal event dispatch.
	        if (intercepted || mFirstTouchTarget != null) {
	            ev.setTargetAccessibilityFocus(false);
	        }

	        // Check for cancelation.
	        final boolean canceled = resetCancelNextUpFlag(this)
	                || actionMasked == MotionEvent.ACTION_CANCEL;

	        // Update list of touch targets for pointer down, if needed.
	        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
	        TouchTarget newTouchTarget = null;
	        boolean alreadyDispatchedToNewTouchTarget = false;
	        if (!canceled && !intercepted) {
	            // If the event is targeting accessibility focus we give it to the
	            // view that has accessibility focus and if it does not handle it
	            // we clear the flag and dispatch the event to all children as usual.
	            // We are looking up the accessibility focused host to avoid keeping
	            // state since these events are very rare.
	            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
	                    ? findChildWithAccessibilityFocus() : null;

	            if (actionMasked == MotionEvent.ACTION_DOWN
	                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
	                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
	                final int actionIndex = ev.getActionIndex(); // always 0 for down
	                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
	                        : TouchTarget.ALL_POINTER_IDS;

	                // Clean up earlier touch targets for this pointer id in case they
	                // have become out of sync.
	                removePointersFromTouchTargets(idBitsToAssign);

	                final int childrenCount = mChildrenCount;
	                if (newTouchTarget == null && childrenCount != 0) {
	                    final float x = ev.getX(actionIndex);
	                    final float y = ev.getY(actionIndex);
	                    // Find a child that can receive the event.
	                    // Scan children from front to back.
	                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
	                    final boolean customOrder = preorderedList == null
	                            && isChildrenDrawingOrderEnabled();
	                    final View[] children = mChildren;
	                    for (int i = childrenCount - 1; i >= 0; i--) {
	                        final int childIndex = getAndVerifyPreorderedIndex(
	                                childrenCount, i, customOrder);
	                        final View child = getAndVerifyPreorderedView(
	                                preorderedList, children, childIndex);

	                        // If there is a view that has accessibility focus we want it
	                        // to get the event first and if not handled we will perform a
	                        // normal dispatch. We may do a double iteration but this is
	                        // safer given the timeframe.
	                        if (childWithAccessibilityFocus != null) {
	                            if (childWithAccessibilityFocus != child) {
	                                continue;
	                            }
	                            childWithAccessibilityFocus = null;
	                            i = childrenCount - 1;
	                        }

	                        if (!child.canReceivePointerEvents()
	                                || !isTransformedTouchPointInView(x, y, child, null)) {
	                            ev.setTargetAccessibilityFocus(false);
	                            continue;
	                        }

	                        newTouchTarget = getTouchTarget(child);
	                        if (newTouchTarget != null) {
	                            // Child is already receiving touch within its bounds.
	                            // Give it the new pointer in addition to the ones it is handling.
	                            newTouchTarget.pointerIdBits |= idBitsToAssign;
	                            break;
	                        }

	                        resetCancelNextUpFlag(child);
	                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
	                            // Child wants to receive touch within its bounds.
	                            mLastTouchDownTime = ev.getDownTime();
	                            if (preorderedList != null) {
	                                // childIndex points into presorted list, find original index
	                                for (int j = 0; j < childrenCount; j++) {
	                                    if (children[childIndex] == mChildren[j]) {
	                                        mLastTouchDownIndex = j;
	                                        break;
	                                    }
	                                }
	                            } else {
	                                mLastTouchDownIndex = childIndex;
	                            }
	                            mLastTouchDownX = ev.getX();
	                            mLastTouchDownY = ev.getY();
	                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
	                            alreadyDispatchedToNewTouchTarget = true;
	                            break;
	                        }

	                        // The accessibility focus didn't handle the event, so clear
	                        // the flag and do a normal dispatch to all children.
	                        ev.setTargetAccessibilityFocus(false);
	                    }
	                    if (preorderedList != null) preorderedList.clear();
	                }

	                if (newTouchTarget == null && mFirstTouchTarget != null) {
	                    // Did not find a child to receive the event.
	                    // Assign the pointer to the least recently added target.
	                    newTouchTarget = mFirstTouchTarget;
	                    while (newTouchTarget.next != null) {
	                        newTouchTarget = newTouchTarget.next;
	                    }
	                    newTouchTarget.pointerIdBits |= idBitsToAssign;
	                }
	            }
	        }

	        // Dispatch to touch targets.
	        if (mFirstTouchTarget == null) {
	            // No touch targets so treat this as an ordinary view.
	            handled = dispatchTransformedTouchEvent(ev, canceled, null,
	                    TouchTarget.ALL_POINTER_IDS);
	        } else {
	            // Dispatch to touch targets, excluding the new touch target if we already
	            // dispatched to it.  Cancel touch targets if necessary.
	            TouchTarget predecessor = null;
	            TouchTarget target = mFirstTouchTarget;
	            while (target != null) {
	                final TouchTarget next = target.next;
	                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
	                    handled = true;
	                } else {
	                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
	                            || intercepted;
	                    if (dispatchTransformedTouchEvent(ev, cancelChild,
	                            target.child, target.pointerIdBits)) {
	                        handled = true;
	                    }
	                    if (cancelChild) {
	                        if (predecessor == null) {
	                            mFirstTouchTarget = next;
	                        } else {
	                            predecessor.next = next;
	                        }
	                        target.recycle();
	                        target = next;
	                        continue;
	                    }
	                }
	                predecessor = target;
	                target = next;
	            }
	        }

	        // Update list of touch targets for pointer up or cancel, if needed.
	        if (canceled
	                || actionMasked == MotionEvent.ACTION_UP
	                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
	            resetTouchState();
	        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
	            final int actionIndex = ev.getActionIndex();
	            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
	            removePointersFromTouchTargets(idBitsToRemove);
	        }
	    }

	    if (!handled && mInputEventConsistencyVerifier != null) {
	        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
	    }
	    return handled;
	}

### 6.2 onInterceptTouchEvent

	public boolean onInterceptTouchEvent(MotionEvent ev) {
	    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
	            && ev.getAction() == MotionEvent.ACTION_DOWN
	            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
	            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
	        return true;
	    }
	    return false;
	}

## 7.View

### 7.1 dispatchTouchEvent

	public boolean dispatchTouchEvent(MotionEvent event) {
	    // If the event should be handled by accessibility focus first.
	    if (event.isTargetAccessibilityFocus()) {
	        // We don't have focus or no virtual descendant has it, do not handle the event.
	        if (!isAccessibilityFocusedViewOrHost()) {
	            return false;
	        }
	        // We have focus and got the event, then use normal event dispatch.
	        event.setTargetAccessibilityFocus(false);
	    }

	    boolean result = false;

	    if (mInputEventConsistencyVerifier != null) {
	        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
	    }

	    final int actionMasked = event.getActionMasked();
	    if (actionMasked == MotionEvent.ACTION_DOWN) {
	        // Defensive cleanup for new gesture
	        stopNestedScroll();
	    }

	    if (onFilterTouchEventForSecurity(event)) {
	        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
	            result = true;
	        }
	        // noinspection SimplifiableIfStatement
	        ListenerInfo li = mListenerInfo;
	        if (li != null && li.mOnTouchListener != null
	                && (mViewFlags & ENABLED_MASK) == ENABLED
	                && li.mOnTouchListener.onTouch(this, event)) {
	            result = true;
	        }

	        if (!result && onTouchEvent(event)) {
	            result = true;
	        }
	    }

	    if (!result && mInputEventConsistencyVerifier != null) {
	        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
	    }

	    // Clean up after nested scrolls if this is the end of a gesture;
	    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
	    // of the gesture.
	    if (actionMasked == MotionEvent.ACTION_UP ||
	            actionMasked == MotionEvent.ACTION_CANCEL ||
	            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
	        stopNestedScroll();
	    }

	    return result;
	}

### 7.2 onTouchEvent

	public boolean onTouchEvent(MotionEvent event) {
	    final float x = event.getX();
	    final float y = event.getY();
	    final int viewFlags = mViewFlags;
	    final int action = event.getAction();

	    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
	            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
	            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

	    if ((viewFlags & ENABLED_MASK) == DISABLED) {
	        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
	            setPressed(false);
	        }
	        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
	        // A disabled view that is clickable still consumes the touch
	        // events, it just doesn't respond to them.
	        return clickable;
	    }
	    if (mTouchDelegate != null) {
	        if (mTouchDelegate.onTouchEvent(event)) {
	            return true;
	        }
	    }

	    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
	        switch (action) {
	            case MotionEvent.ACTION_UP:
	                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
	                if ((viewFlags & TOOLTIP) == TOOLTIP) {
	                    handleTooltipUp();
	                }
	                if (!clickable) {
	                    removeTapCallback();
	                    removeLongPressCallback();
	                    mInContextButtonPress = false;
	                    mHasPerformedLongPress = false;
	                    mIgnoreNextUpEvent = false;
	                    break;
	                }
	                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
	                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
	                    // take focus if we don't have it already and we should in
	                    // touch mode.
	                    boolean focusTaken = false;
	                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
	                        focusTaken = requestFocus();
	                    }

	                    if (prepressed) {
	                        // The button is being released before we actually
	                        // showed it as pressed.  Make it show the pressed
	                        // state now (before scheduling the click) to ensure
	                        // the user sees it.
	                        setPressed(true, x, y);
	                    }

	                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
	                        // This is a tap, so remove the longpress check
	                        removeLongPressCallback();

	                        // Only perform take click actions if we were in the pressed state
	                        if (!focusTaken) {
	                            // Use a Runnable and post this rather than calling
	                            // performClick directly. This lets other visual state
	                            // of the view update before click actions start.
	                            if (mPerformClick == null) {
	                                mPerformClick = new PerformClick();
	                            }
	                            if (!post(mPerformClick)) {
	                                performClickInternal();
	                            }
	                        }
	                    }

	                    if (mUnsetPressedState == null) {
	                        mUnsetPressedState = new UnsetPressedState();
	                    }

	                    if (prepressed) {
	                        postDelayed(mUnsetPressedState,
	                                ViewConfiguration.getPressedStateDuration());
	                    } else if (!post(mUnsetPressedState)) {
	                        // If the post failed, unpress right now
	                        mUnsetPressedState.run();
	                    }

	                    removeTapCallback();
	                }
	                mIgnoreNextUpEvent = false;
	                break;
	            case MotionEvent.ACTION_DOWN:
	                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
	                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
	                }
	                mHasPerformedLongPress = false;

	                if (!clickable) {
	                    checkForLongClick(
	                            ViewConfiguration.getLongPressTimeout(),
	                            x,
	                            y,
	                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
	                    break;
	                }

	                if (performButtonActionOnTouchDown(event)) {
	                    break;
	                }

	                // Walk up the hierarchy to determine if we're inside a scrolling container.
	                boolean isInScrollingContainer = isInScrollingContainer();

	                // For views inside a scrolling container, delay the pressed feedback for
	                // a short period in case this is a scroll.
	                if (isInScrollingContainer) {
	                    mPrivateFlags |= PFLAG_PREPRESSED;
	                    if (mPendingCheckForTap == null) {
	                        mPendingCheckForTap = new CheckForTap();
	                    }
	                    mPendingCheckForTap.x = event.getX();
	                    mPendingCheckForTap.y = event.getY();
	                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
	                } else {
	                    // Not inside a scrolling container, so show the feedback right away
	                    setPressed(true, x, y);
	                    checkForLongClick(
	                            ViewConfiguration.getLongPressTimeout(),
	                            x,
	                            y,
	                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
	                }
	                break;
	            case MotionEvent.ACTION_CANCEL:
	                if (clickable) {
	                    setPressed(false);
	                }
	                removeTapCallback();
	                removeLongPressCallback();
	                mInContextButtonPress = false;
	                mHasPerformedLongPress = false;
	                mIgnoreNextUpEvent = false;
	                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
	                break;
	            case MotionEvent.ACTION_MOVE:
	                if (clickable) {
	                    drawableHotspotChanged(x, y);
	                }

	                final int motionClassification = event.getClassification();
	                final boolean ambiguousGesture =
	                        motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
	                int touchSlop = mTouchSlop;
	                if (ambiguousGesture && hasPendingLongPressCallback()) {
	                    final float ambiguousMultiplier =
	                            ViewConfiguration.getAmbiguousGestureMultiplier();
	                    if (!pointInView(x, y, touchSlop)) {
	                        // The default action here is to cancel long press. But instead, we
	                        // just extend the timeout here, in case the classification
	                        // stays ambiguous.
	                        removeLongPressCallback();
	                        long delay = (long) (ViewConfiguration.getLongPressTimeout() * ambiguousMultiplier);
	                        // Subtract the time already spent
	                        delay -= event.getEventTime() - event.getDownTime();
	                        checkForLongClick(
	                                delay,
	                                x,
	                                y,
	                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
	                    }
	                    touchSlop *= ambiguousMultiplier;
	                }

	                // Be lenient about moving outside of buttons
	                if (!pointInView(x, y, touchSlop)) {
	                    // Outside button
	                    // Remove any future long press/tap checks
	                    removeTapCallback();
	                    removeLongPressCallback();
	                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
	                        setPressed(false);
	                    }
	                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
	                }

	                final boolean deepPress =
	                        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
	                if (deepPress && hasPendingLongPressCallback()) {
	                    // process the long click action immediately
	                    removeLongPressCallback();
	                    checkForLongClick(
	                            0 /* send immediately */,
	                            x,
	                            y,
	                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
	                }
	                break;
	        }

	        return true;
	    }

	    return false;
	}


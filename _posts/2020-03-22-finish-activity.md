---
layout: post
comments: true
title: "Activity销毁过程"
description: "Activity销毁过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析Activity销毁过程.

## 0.文件结构

	frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java
	frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java
	frameworks/base/core/java/android/app/servertransaction/PauseActivityItem.java

	frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java

	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/Instrumentation.java
	frameworks/base/core/java/android/app/Activity.java
	frameworks/base/core/java/android/app/FragmentController.java

## 1.时序图

TODO

## 2.Activity.finish
[ -> frameworks/base/core/java/android/app/Activity.java ]

	public void finish() {
	    // finish当前Activity，但不finish所在的Task
	    finish(DONT_FINISH_TASK_WITH_ACTIVITY);
	}

	private void finish(int finishTask) {
	    if (mParent == null) { // 父Activity为空
	        int resultCode;
	        Intent resultData;
	        synchronized (this) {
	            resultCode = mResultCode;
	            resultData = mResultData;
	        }
	        if (resultData != null) {
	            resultData.prepareToLeaveProcess(this);
	        }
	        // 调用ATMS.finishActivity
	        if (ActivityTaskManager.getService().finishActivity(mToken, resultCode, resultData, finishTask)) {
	            mFinished = true;
	        }
	    } else { // 如果父Activity非空，直接finish整个Activity group
	        mParent.finishFromChild(this);
	    }
	    ...
	}

以下为finishTask: 

    // 当finish Activity时，其所在的Task不finish
    public static final int DONT_FINISH_TASK_WITH_ACTIVITY = 0;
    // 当finish root Activity时，其所在的Task也finish, 且Task从最近任务列表移除
    public static final int FINISH_TASK_WITH_ROOT_ACTIVITY = 1;
    // 当finish Activity时，其所在的Task也finish, 但Task不从最近任务列表移除
    public static final int FINISH_TASK_WITH_ACTIVITY = 2;

## 3.ActivityTaskManagerService.finishActivity
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java ]

	@Override
	public final boolean finishActivity(IBinder token, int resultCode, Intent resultData, int finishTask) {
	    synchronized (mGlobalLock) {
	        // 获取当前ActivityRecord
	        ActivityRecord r = ActivityRecord.isInStackLocked(token);
	        if (r == null) {
	            return true;
	        }
	        // 获取当前task
	        final TaskRecord tr = r.getTaskRecord();
	        // 获取当前task的根Activity
	        ActivityRecord rootR = tr.getRootActivity();
	        ...
	        boolean res;
	        // 若当前Activity为根Activity，判断是否finish根Activity所在的task
	        final boolean finishWithRootActivity = finishTask == Activity.FINISH_TASK_WITH_ROOT_ACTIVITY;
	        if (finishTask == Activity.FINISH_TASK_WITH_ACTIVITY || (finishWithRootActivity && r == rootR)) {
	            // finish当前task
	            res = mStackSupervisor.removeTaskByIdLocked(tr.taskId, false, finishWithRootActivity, "finish-activity");
	        } else {
	            // finish当前Activity
	            res = tr.getStack().requestFinishActivityLocked(token, resultCode, resultData, "app-request", true);
	        }
	        return res;
	    }
	}

## 4.ActivityStack
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java ]

### 4.1 requestFinishActivityLocked

	final boolean requestFinishActivityLocked(IBinder token, int resultCode, Intent resultData, String reason, boolean oomAdj) {
		// 先根据token获取ActivityRecord，然后判断该ActivityRecord是否在其所属Task的Activity列表并返回.
	    ActivityRecord r = isInStackLocked(token);
	    if (r == null) {
	        return false;
	    }
	 
	    finishActivityLocked(r, resultCode, resultData, reason, oomAdj);
	    return true;
	}

### 4.2 finishActivityLocked

	final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData, String reason, boolean oomAdj) {
	    // pauseImmediately为false
	    return finishActivityLocked(r, resultCode, resultData, reason, oomAdj, !PAUSE_IMMEDIATELY);
	}

	final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
	        String reason, boolean oomAdj, boolean pauseImmediately) {
	    mWindowManager.deferSurfaceLayout();   
	    try {
	        // 设置Activity的属性finishing=true
	        r.makeFinishingLocked();
	        // 获取当前task
	        final TaskRecord task = r.getTaskRecord();
	        // 获取当前task中所有Activity
	        final ArrayList<ActivityRecord> activities = task.mActivities;
	        // 获取当前Activity的索引
	        final int index = activities.indexOf(r);
	        if (index < (activities.size() - 1)) { // 如果当前不是最后一个Activity
	            task.setFrontOfTask();
	            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {
	                // If the caller asked that this activity (and all above it)
	                // be cleared when the task is reset, don't lose that information,
	                // but propagate it up to the next activity.
	                // 如果当前Activity的flags为FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET
	                // 则将该flags传递给下一个Activity
	                ActivityRecord next = activities.get(index+1);
	                next.intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET);
	            }
	        }
	 
	        // 暂停Key分发事件
	        r.pauseKeyDispatchingLocked();
	        // Move activity with its stack to front and make the stack focused
	        // 将当前Activity及其所在Task调整到最前面，使其focused
	        adjustFocusedActivityStack(r, "finishActivity");
	 
	        // 创建ActivityResult, 并将其添加到启动当前Activity的那个Activity的ActivityResult列表results
	        // 见【第4.3节】
	        finishActivityResultsLocked(r, resultCode, resultData);
	 
	        // 判断是否清除当前task: 当前Activity位于栈底且Task不复用时，则结束Task
	        final boolean endTask = index <= 0 && !task.isClearingToReuseTask();
	        // 设置转换模式
	        final int transit = endTask ? TRANSIT_TASK_CLOSE : TRANSIT_ACTIVITY_CLOSE;
	        if (mResumedActivity == r) { // 当前Activity为Resumed状态
	            if (endTask) {
	                // 通知listeners当前Task即将被移除
	                mService.getTaskChangeNotificationController().notifyTaskRemovalStarted(task.getTaskInfo());
	            }
	            getDisplay().mDisplayContent.prepareAppTransition(transit, false);
	 
	            // 设置Activity不可见，告诉WindowManager当前Activity即将被移除
	            r.setVisibility(false);
	 
	            if (mPausingActivity == null) {
	                // 执行pause操作. 见【第4.5节】
	                startPausingLocked(false, false, null, pauseImmediately);
	            }
	 
	            if (endTask) {
	                // 清除当前Task
	                mService.getLockTaskController().clearLockedTask(task);
	            }
	        } else if (!r.isState(PAUSING)) { // 当前Activity非Pausing状态
	            // If the activity is PAUSING, we will complete the finish once
	            // it is done pausing; else we can just directly finish it here.
	            if (r.visible) {
	                // 设置Activity为不可见
	                prepareActivityHideTransitionAnimation(r, transit);
	            }
	 
	            // 设置finishMode
	            final int finishMode = (r.visible || r.nowVisible) ? FINISH_AFTER_VISIBLE : FINISH_AFTER_PAUSE;
	            // finish当前Activity
	            final boolean removedActivity = finishCurrentActivityLocked(r, finishMode, oomAdj, "finishActivityLocked") == null;
	 
	            // The following code is an optimization. When the last non-task overlay activity
	            // is removed from the task, we remove the entire task from the stack. However,
	            // since that is done after the scheduled destroy callback from the activity, that
	            // call to change the visibility of the task overlay activities would be out of
	            // sync with the activitiy visibility being set for this finishing activity above.
	            // In this case, we can set the visibility of all the task overlay activities when
	            // we detect the last one is finishing to keep them in sync.
	            if (task.onlyHasTaskOverlayActivities(true /* excludeFinishing */)) {
	                for (ActivityRecord taskOverlay : task.mActivities) {
	                    if (!taskOverlay.mTaskOverlay) {
	                        continue;
	                    }
	                    prepareActivityHideTransitionAnimation(taskOverlay, transit);
	                }
	            }
	            return removedActivity;
	        }
	 
	        return false;
	    } finally {
	        mWindowManager.continueSurfaceLayout();
	    }
	}

### 4.3 finishActivityResultsLocked

	private void finishActivityResultsLocked(ActivityRecord r, int resultCode, Intent resultData) {
	    // resultTo为startActivity时启动当前Activity的那个Activity(当前Activity的结果发给这个Activity)
	    ActivityRecord resultTo = r.resultTo;
	    if (resultTo != null) {
	        if (resultTo.mUserId != r.mUserId) {
	            if (resultData != null) {
	                resultData.prepareToLeaveUser(r.mUserId);
	            }
	        }
	        if (r.info.applicationInfo.uid > 0) {
	            // 授予Uri权限
	            mService.mUgmInternal.grantUriPermissionFromIntent(r.info.applicationInfo.uid,
	                    resultTo.packageName, resultData,
	                    resultTo.getUriPermissionsLocked(), resultTo.mUserId);
	        }
	        // 创建ActivityResult, 并添加到resultTo的ActivityResult列表results. 见【第4.4节】
	        resultTo.addResultLocked(r, r.resultWho, r.requestCode, resultCode, resultData);
	        r.resultTo = null;
	    }
	 
	    // 当前Activity属性置空
	    r.results = null;
	    r.pendingResults = null;
	    r.newIntents = null;
	    r.icicle = null;
	}

### 4.4 ActivityRecord.addResultLocked
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java ]

    void addResultLocked(ActivityRecord from, String resultWho,
            int requestCode, int resultCode, Intent resultData) {
        ActivityResult r = new ActivityResult(from, resultWho,
                requestCode, resultCode, resultData);
        if (results == null) {
            results = new ArrayList<ResultInfo>();
        }
        results.add(r);
    }

### 4.5 startPausingLocked
	
	// resuming参数含义:
	// The activity we are currently trying to resume or null if this is not being called as part
    // of resuming the top activity, so we shouldn't try to instigate  a resume here if not null.
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
        ActivityRecord resuming, boolean pauseImmediately) {
	    // 如果当前有pausing状态的Activity(即mPausingActivity非空), 则完成Pause过程
	    if (mPausingActivity != null) {
	        if (!shouldSleepActivities()) {
	            completePauseLocked(false, resuming);
	        }
	    }
	    ActivityRecord prev = mResumedActivity;
	 
	    // Trying to pause when nothing is resumed
	    // 我理解这段代码含义: 当前没有resume状态的Activity, 说明这是一个resume过程
	    // 因此resume top activity, 然后直接return false. 正常的Pause走不到这里
	    if (prev == null && resuming == null) {
	        mRootActivityContainer.resumeFocusedStacksTopActivities();
	        return false;
	    }
	 
	    // Trying to pause activity that is in process of being resumed
	    if (prev == resuming) {
	        return false;
	    }
	 
	    // 将之前Resumed Activity设置为当前Paused Activity
	    mPausingActivity = prev;
	    mLastPausedActivity = prev;
	    mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
	            || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
	    // 设置Activity状态为Pausing
	    prev.setState(PAUSING, "startPausingLocked");
	    ...
	 
	    if (prev.attachedToProcess()) { // 进程非空
	        // 执行PauseActivityItem事务
	        mService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
	                prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
	                        prev.configChangeFlags, pauseImmediately));
	    } else { // 进程为空
	        mPausingActivity = null;
	        mLastPausedActivity = null;
	        mLastNoHistoryActivity = null;
	    }
	 
	    // If we are not going to sleep, we want to ensure the device is
	    // awake until the next activity is started.
	    if (!uiSleeping && !mService.isSleepingOrShuttingDownLocked()) {
	        mStackSupervisor.acquireLaunchWakelock();
	    }
	 
	    if (mPausingActivity != null) {
	        // Have the window manager pause its key dispatching until the new
	        // activity has started.  If we're pausing the activity just because
	        // the screen is being turned off and the UI is sleeping, don't interrupt
	        // key dispatch; the same activity will pick it up again on wakeup.
	        if (!uiSleeping) {
	            prev.pauseKeyDispatchingLocked();
	        }
	 
	        if (pauseImmediately) { // 立即pause
	            // If the caller said they don't want to wait for the pause, then complete the pause now.
	            completePauseLocked(false, resuming);
	            return false;
	        } else { // 500ms延时再pause
	            schedulePauseTimeout(prev);
	            return true;
	        }
	    } else { // Pause失败
	        // This activity failed to schedule the pause, so just treat it as being paused now.
	        if (resuming == null) {
	            mRootActivityContainer.resumeFocusedStacksTopActivities();
	        }
	        return false;
	    }
	}

## 5.Pause过程-PauseActivityItem
[ -> frameworks/base/core/java/android/app/servertransaction/PauseActivityItem.java ]

	public class PauseActivityItem extends ActivityLifecycleItem {
	    @Override
	    public void execute(ClientTransactionHandler client, IBinder token, PendingTransactionActions pendingActions) {
	        // 参考【第7.1节】
	        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions, "PAUSE_ACTIVITY_ITEM");
	    }

	    @Override
	    public int getTargetState() {
	        return ON_PAUSE;
	    }

	    @Override
	    public void postExecute(ClientTransactionHandler client, IBinder token, PendingTransactionActions pendingActions) {
	        ActivityTaskManager.getService().activityPaused(token);
	    }
	}

### 5.1 execute

handlePauseActivity. 参考【第7.1节】

### 5.2 postExecute

ActivityTaskManager.getService().activityPaused(token);

### 5.3 ActivityTaskManagerService.activityPaused
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java ]

	@Override
	public final void activityPaused(IBinder token) {
	    synchronized (mGlobalLock) {
	        ActivityStack stack = ActivityRecord.getStackLocked(token);
	        if (stack != null) {
	            stack.activityPausedLocked(token, false);
	        }
	    }
	}

### 5.4 ActivityStack
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java ]

#### 5.4.1 activityPausedLocked

	final void activityPausedLocked(IBinder token, boolean timeout) {
	    final ActivityRecord r = isInStackLocked(token);
	 
	    if (r != null) {
	        mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
	        if (mPausingActivity == r) { // 继续完成pause操作
	            completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
	            return;
	        } else {
	            if (r.isState(PAUSING)) { // pause失败的Activity，直接finish掉
	                r.setState(PAUSED, "activityPausedLocked");
	                if (r.finishing) {
	                    finishCurrentActivityLocked(r, FINISH_AFTER_VISIBLE, false, "activityPausedLocked");
	                }
	            }
	        }
	    }
	    mRootActivityContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
	}

#### 5.4.2 completePauseLocked

	private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
	    ActivityRecord prev = mPausingActivity;

	    if (prev != null) {
	        prev.setWillCloseOrEnterPip(false);
	        // wasStopping为false
	        final boolean wasStopping = prev.isState(STOPPING);
	        // 设置Activity状态为PAUSED
	        prev.setState(PAUSED, "completePausedLocked");
	        if (prev.finishing) { // 会走到这里
	            // finish Activity
	            prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false, "completePausedLocked");
	        } else if (prev.hasProcess()) {
	            if (prev.deferRelaunchUntilPaused) { // Re-launching after pause
	                // Complete the deferred relaunch that was waiting for pause to complete.
	                prev.relaunchActivityLocked(false /* andResume */, prev.preserveWindowOnDeferredRelaunch);
	            } else if (wasStopping) {
	                // We are also stopping, the stop request must have gone soon after the pause.
	                // We can't clobber it, because the stop confirmation will not be handled.
	                // We don't need to schedule another stop, we only need to let it happen.
	                prev.setState(STOPPING, "completePausedLocked");
	            } else if (!prev.visible || shouldSleepOrShutDownActivities()) {
	                // Clear out any deferred client hide we might currently have.
	                prev.setDeferHidingClient(false);
	                // If we were visible then resumeTopActivities will release resources before
	                // stopping.
	                addToStopping(prev, true /* scheduleIdle */, false /* idleDelayed */, "completePauseLocked");
	            }
	        } else { // App died during pause, not stopping
	            prev = null;
	        }
	        // It is possible the activity was freezing the screen before it was paused.
	        // In that case go ahead and remove the freeze this activity has on the screen
	        // since it is no longer visible.
	        if (prev != null) {
	            prev.stopFreezingScreenLocked(true /*force*/);
	        }
	        mPausingActivity = null;
	    }
	 
	    if (resumeNext) { // 是否恢复下一个Activity
	        final ActivityStack topStack = mRootActivityContainer.getTopDisplayFocusedStack();
	        if (!topStack.shouldSleepOrShutDownActivities()) {
	            // 如果没有 休眠和关机时，恢复顶层Activity
	            mRootActivityContainer.resumeFocusedStacksTopActivities(topStack, prev, null);
	        } else {
	            // 等待休眠完成，再恢复顶层Activity
	            checkReadyForSleep();
	            ActivityRecord top = topStack.topRunningActivityLocked();
	            if (top == null || (prev != null && top != prev)) {
	                mRootActivityContainer.resumeFocusedStacksTopActivities();
	            }
	        }
	    }
	 
	    if (prev != null) {
	        // 恢复Activity的Key事件分发
	        prev.resumeKeyDispatchingLocked();
	        // 更新进程在前台运行的时间
	        if (prev.hasProcess() && prev.cpuTimeAtResume > 0) {
	            final long diff = prev.app.getCpuTime() - prev.cpuTimeAtResume;
	            if (diff > 0) {
	                final Runnable r = PooledLambda.obtainRunnable(
	                        ActivityManagerInternal::updateForegroundTimeIfOnBattery,
	                        mService.mAmInternal, prev.info.packageName,
	                        prev.info.applicationInfo.uid,
	                        diff);
	                mService.mH.post(r);
	            }
	        }
	        // 重置resume时间
	        prev.cpuTimeAtResume = 0; // reset it
	    }
	 
	    // 通知Task栈发生变化
	    if (mStackSupervisor.mAppVisibilitiesChangedSinceLastPause || (getDisplay() != null && getDisplay().hasPinnedStack())) {
	        mService.getTaskChangeNotificationController().notifyTaskStackChanged();
	        mStackSupervisor.mAppVisibilitiesChangedSinceLastPause = false;
	    }
	 
	    mRootActivityContainer.ensureActivitiesVisible(resuming, 0, !PRESERVE_WINDOWS);
	}

#### 5.4.3 finishCurrentActivityLocked

	final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj, String reason) {
	    // First things first: if this activity is currently visible,
	    // and the resumed activity is not yet visible, then hold off on
	    // finishing until the resumed one becomes visible.
	 
	    // The activity that we are finishing may be over the lock screen. In this case, we do not
	    // want to consider activities that cannot be shown on the lock screen as running and should
	    // proceed with finishing the activity if there is no valid next top running activity.
	    // Note that if this finishing activity is floating task, we don't need to wait the
	    // next activity resume and can destroy it directly.
	    final ActivityDisplay display = getDisplay();
	    // 获取下一个Activity
	    final ActivityRecord next = display.topRunningActivity(true /* considerKeyguardState */);
	    final boolean isFloating = r.getConfiguration().windowConfiguration.tasksAreFloating();
	 
		// 如果mode为FINISH_AFTER_VISIBLE且下一个Activity不可见(默认即为此),
		// 则先将当前Activity添加到stopping列表，并将其状态设置为STOPPING
		// 等下一个Activity可见后再finish当前Activity
	    if (mode == FINISH_AFTER_VISIBLE && (r.visible || r.nowVisible)
	            && next != null && !next.nowVisible && !isFloating) {
	        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
	            // 添加到stopping列表. 参考【第5.4.4节】
	            addToStopping(r, false /* scheduleIdle */, false /* idleDelayed */, "finishCurrentActivityLocked");
	        }
	 
	        // Activity状态设置为STOPPING
	        r.setState(STOPPING, "finishCurrentActivityLocked");
	        if (oomAdj) {
	            mService.updateOomAdj();
	        }
	        return r;
	    }
	 
	    // make sure the record is cleaned out of other places.
	    mStackSupervisor.mStoppingActivities.remove(r);
	    mStackSupervisor.mGoingToSleepActivities.remove(r);
	 
	    // 获取当前状态
	    final ActivityState prevState = r.getState();
	 
	    // Activity状态设置为FINISHING
	    r.setState(FINISHING, "finishCurrentActivityLocked");
	 
	    // Don't destroy activity immediately if the display contains home stack, although there is
	    // no next activity at the moment but another home activity should be started later. Keep
	    // this activity alive until next home activity is resumed then user won't see a temporary
	    // black screen.
	    // 如果没有要显示的下一个Activity且当前显示的栈中包含Home栈，则先等HomeActivity恢复之后再销毁当前Activity
	    // 这样做的目的是避免出现临时黑屏
	    final boolean noRunningStack = next == null && display.topRunningActivity() == null && display.getHomeStack() == null;
	    final boolean noFocusedStack = r.getActivityStack() != display.getFocusedStack();
	    final boolean finishingInNonFocusedStackOrNoRunning = mode == FINISH_AFTER_VISIBLE && prevState == PAUSED && (noFocusedStack || noRunningStack);
	 
	    if (mode == FINISH_IMMEDIATELY
	            || (prevState == PAUSED && (mode == FINISH_AFTER_PAUSE || inPinnedWindowingMode()))
	            || finishingInNonFocusedStackOrNoRunning
	            || prevState == STOPPING
	            || prevState == STOPPED
	            || prevState == ActivityState.INITIALIZING) { // 满足立即销毁的条件
	        // 设置finishing=true
	        r.makeFinishingLocked();
	        // 销毁当前Activity
	        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm:" + reason);
	 
	        if (finishingInNonFocusedStackOrNoRunning) {
	            // Finishing activity that was in paused state and it was in not currently focused
	            // stack, need to make something visible in its place. Also if the display does not
	            // have running activity, the configuration may need to be updated for restoring
	            // original orientation of the display.
	            mRootActivityContainer.ensureVisibilityAndConfig(next, mDisplayId, false /* markFrozenIfConfigChanged */, true /* deferResume */);
	        }
	        if (activityRemoved) {
	            mRootActivityContainer.resumeFocusedStacksTopActivities();
	        }
	        return activityRemoved ? null : r;
	    }
	 
	    // Need to go through the full pause cycle to get this
	    // activity into the stopped state and then finish it.
	    if (DEBUG_ALL) Slog.v(TAG, "Enqueueing pending finish: " + r);
	    mStackSupervisor.mFinishingActivities.add(r);
	    r.resumeKeyDispatchingLocked();
	    mRootActivityContainer.resumeFocusedStacksTopActivities();
	    // If activity was not paused at this point - explicitly pause it to start finishing
	    // process. Finishing will be completed once it reports pause back.
	    if (r.isState(RESUMED) && mPausingActivity != null) {
	        startPausingLocked(false /* userLeaving */, false /* uiSleeping */, next /* resuming */, false /* dontWait */);
	    }
	    return r;
	}

以下为finishMode:

    // 立刻finish当前Activity
    static final int FINISH_IMMEDIATELY = 0;
    // 当前Activity pause后finish
    static final int FINISH_AFTER_PAUSE = 1;
    // 下一个要显示的Activity可见后再finish当前Activity
    static final int FINISH_AFTER_VISIBLE = 2;

一般面试会问的一个问题:  
为啥A -> B进行Activity切换的时候，生命周期是: A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop -> A.onDestroy. 而不是先A.onPause -> A.onStop -> A.onDestroy -> B.onCreate -> B.onStart -> B.onResume.

这段代码给出了解释:   
First things first: if this activity is currently visible, and the resumed activity is not yet visible,   
then hold off on finishing until the resumed one becomes visible.

The activity that we are finishing may be over the lock screen. In this case, we do not  
want to consider activities that cannot be shown on the lock screen as running and should  
proceed with finishing the activity if there is no valid next top running activity.  
Note that if this finishing activity is floating task, we don't need to wait the  
next activity resume and can destroy it directly.  

Don't destroy activity immediately if the display contains home stack, although there is  
no next activity at the moment but another home activity should be started later. Keep  
this activity alive until next home activity is resumed then user won't see a temporary black screen.  

#### 5.4.4 addToStopping

    private void addToStopping(ActivityRecord r, boolean scheduleIdle, boolean idleDelayed,
            String reason) {
        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
            // 将当前Activity添加到stopping列表
            mStackSupervisor.mStoppingActivities.add(r);
        }

        // If we already have a few activities waiting to stop, then give up
        // on things going idle and start clearing them out. Or if r is the
        // last of activity of the last task the stack will be empty and must
        // be cleared immediately.
        // 如果stopping列表超出最大值，或者当前Activity为所在Task的根Activity且任务历史列表只剩这一个Task
        // 则先保持idle，暂时不做什么
        boolean forceIdle = mStackSupervisor.mStoppingActivities.size() > MAX_STOPPING_TO_FORCE
                || (r.frontOfTask && mTaskHistory.size() <= 1);
        if (scheduleIdle || forceIdle) {
            if (!idleDelayed) {
                mStackSupervisor.scheduleIdleLocked();
            } else {
                mStackSupervisor.scheduleIdleTimeoutLocked(r);
            }
        } else { // sleep
            checkReadyForSleep();
        }
    }

#### 5.4.5 checkReadyForSleep

    void checkReadyForSleep() {
        if (shouldSleepActivities() && goToSleepIfPossible(false /* shuttingDown */)) {
            mStackSupervisor.checkReadyForSleepLocked(true /* allowDelay */);
        }
    }

#### 5.4.6 goToSleepIfPossible

    boolean goToSleepIfPossible(boolean shuttingDown) {
        boolean shouldSleep = true;

        if (mResumedActivity != null) {
            // Still have something resumed; can't sleep until it is paused.
            startPausingLocked(false, true, null, false);
            shouldSleep = false ;
        } else if (mPausingActivity != null) {
            // Still waiting for something to pause; can't sleep yet.
            shouldSleep = false;
        }

        if (!shuttingDown) {
            // 如果包含要stopping的Activity，则先做stop操作
            if (containsActivityFromStack(mStackSupervisor.mStoppingActivities)) {
                // Still need to tell some activities to stop; can't sleep yet.
                mStackSupervisor.scheduleIdleLocked();
                shouldSleep = false;
            }

            if (containsActivityFromStack(mStackSupervisor.mGoingToSleepActivities)) {
                // Still need to tell some activities to sleep; can't sleep yet.
                shouldSleep = false;
            }
        }

        if (shouldSleep) {
            goToSleep();
        }

        return shouldSleep;
    }

#### 5.4.7 ActivityStackSupervisor.scheduleIdleLocked

    final void scheduleIdleLocked() {
        mHandler.sendEmptyMessage(IDLE_NOW_MSG);
    }

#### 5.4.8 ActivityStackSupervisorHandler

    private final class ActivityStackSupervisorHandler extends Handler {

        public ActivityStackSupervisorHandler(Looper looper) {
            super(looper);
        }

        void activityIdleInternal(ActivityRecord r, boolean processPausingActivities) {
            synchronized (mService.mGlobalLock) {
                activityIdleInternalLocked(r != null ? r.appToken : null, true /* fromTimeout */,
                        processPausingActivities, null);
            }
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                ...
                case IDLE_TIMEOUT_MSG: {
                    // We don't at this point know if the activity is fullscreen,
                    // so we need to be conservative and assume it isn't.
                    activityIdleInternal((ActivityRecord) msg.obj,
                            true /* processPausingActivities */);
                } break;
                case IDLE_NOW_MSG: {
                    activityIdleInternal((ActivityRecord) msg.obj,
                            false /* processPausingActivities */);
                } break;
                ...
            }
        }

#### 5.4.9 activityIdleInternalLocked

TODO：  
这里比较复杂，我的理解是：  
这里有个线程机制，当线程空闲时会执行stopping Activity操作，最终又会走到ActivityStack#finishCurrentActivityLocked  
参考【第5.4.3节】 之后在执行这段代码过程中又会执行到destroyActivityLocked

#### 5.4.10 destroyActivityLocked

	final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
	    if (r.isState(DESTROYING, DESTROYED)) {
	        return false;
	    }

	    boolean removedFromHistory = false;
	 
	    cleanUpActivityLocked(r, false, false);
	 
	    final boolean hadApp = r.hasProcess();
	 
	    if (hadApp) {
	        if (removeFromApp) {
	            r.app.removeActivity(r);
	            if (!r.app.hasActivities()) {
	                mService.clearHeavyWeightProcessIfEquals(r.app);
	            }
	            if (!r.app.hasActivities()) {
	                // Update any services we are bound to that might care about whether
	                // their client may have activities.
	                // No longer have activities, so update LRU list and oom adj.
	                r.app.updateProcessInfo(true /* updateServiceConnectionActivities */,
	                        false /* activityChange */, true /* updateOomAdj */);
	            }
	        }
	 
	        boolean skipDestroy = false;
	 
	        // 将最终状态设置为DestroyActivityItem, 并执行ON_PAUSE~ON_DESTROY的过程
	        mService.getLifecycleManager().scheduleTransaction(r.app.getThread(), r.appToken,
	                DestroyActivityItem.obtain(r.finishing, r.configChangeFlags));
	         
	        r.nowVisible = false;
	 
	        // If the activity is finishing, we need to wait on removing it
	        // from the list to give it a chance to do its cleanup.  During
	        // that time it may make calls back with its token so we need to
	        // be able to find it on the list and so we don't want to remove
	        // it from the list yet.  Otherwise, we can just immediately put
	        // it in the destroyed state since we are not removing it from the
	        // list.
	        if (r.finishing && !skipDestroy) {
	            // Activity状态设置为DESTROYING
	            r.setState(DESTROYING, "destroyActivityLocked. finishing and not skipping destroy");
	            Message msg = mHandler.obtainMessage(DESTROY_TIMEOUT_MSG, r);
	            mHandler.sendMessageDelayed(msg, DESTROY_TIMEOUT);
	        } else {
	            r.setState(DESTROYED, "destroyActivityLocked. not finishing or skipping destroy");
	            r.app = null;
	        }
	    } else {
	        // remove this record from the history.
	        if (r.finishing) {
	            removeActivityFromHistoryLocked(r, reason + " hadNoApp");
	            removedFromHistory = true;
	        } else {
	            r.setState(DESTROYED, "destroyActivityLocked. not finishing and had no app");
	            r.app = null;
	        }
	    }
	 
	    r.configChangeFlags = 0;
	 
	    return removedFromHistory;
	}

接下来就是执行Stop和Destroy过程了.

## 6.Stop & Destroy过程

### 6.1 TransactionExecutor.performLifecycleSequence  
[ -> frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java ]

	private void performLifecycleSequence(ActivityClientRecord r, IntArray path, ClientTransaction transaction) {
	    final int size = path.size();
	    for (int i = 0, state; i < size; i++) {
	        state = path.get(i);
	        switch (state) {
	            case ON_CREATE:
	                mTransactionHandler.handleLaunchActivity(r, mPendingActions, null /* customIntent */);
	                break;
	            case ON_START:
	                mTransactionHandler.handleStartActivity(r, mPendingActions);
	                break;
	            case ON_RESUME:
	                mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */, r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
	                break;
	            case ON_PAUSE:
	                // 参考【第7.1节】
	                mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
	                        false /* userLeaving */, 0 /* configChanges */, mPendingActions, "LIFECYCLER_PAUSE_ACTIVITY");
	                break;
	            case ON_STOP:
	                // 参考【第7.2节】
	                mTransactionHandler.handleStopActivity(r.token, false /* show */,
	                        0 /* configChanges */, mPendingActions, false /* finalStateRequest */, "LIFECYCLER_STOP_ACTIVITY");
	                break;
	            case ON_DESTROY:
	                // 参考【第7.3节】
	                mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
	                        0 /* configChanges */, false /* getNonConfigInstance */, "performLifecycleSequence. cycling to:" + path.get(size - 1));
	                break;
	            case ON_RESTART:
	                mTransactionHandler.performRestartActivity(r.token, false /* start */);
	                break;
	            default:
	                throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
	        }
	    }
	}

## 7.ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 7.1 handlePauseActivity

	@Override
	public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
	        int configChanges, PendingTransactionActions pendingActions, String reason) {
	    ActivityClientRecord r = mActivities.get(token);
	    if (r != null) {
	        if (userLeaving) {
	            performUserLeavingActivity(r);
	        }
	 
	        r.activity.mConfigChangeFlags |= configChanges;
	        performPauseActivity(r, finished, reason, pendingActions);
	 
	        // Make sure any pending writes are now committed.
	        if (r.isPreHoneycomb()) {
	            QueuedWork.waitToFinish();
	        }
	        mSomeActivitiesChanged = true;
	    }
	}

#### 7.1.1 performPauseActivity

	private Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason, PendingTransactionActions pendingActions) {
	    ...
	    if (finished) {
	        r.activity.mFinished = true;
	    }
	 
	    final boolean shouldSaveState = !r.activity.mFinished && r.isPreHoneycomb();
	    if (shouldSaveState) {
		    // Android API版本<11, 在onPause之前先调用Activity.OnSaveInstanceState
	        callActivityOnSaveInstanceState(r);
	    }
	 
	    performPauseActivityIfNeeded(r, reason);
	 
	    // 通知外部注册的listeners
	    ArrayList<OnActivityPausedListener> listeners;
	    synchronized (mOnPauseListeners) {
	        listeners = mOnPauseListeners.remove(r.activity);
	    }
	    int size = (listeners != null ? listeners.size() : 0);
	    for (int i = 0; i < size; i++) {
	        listeners.get(i).onPaused(r.activity);
	    }
	 
	    final Bundle oldState = pendingActions != null ? pendingActions.getOldState() : null;
	    if (oldState != null) {
	        // We need to keep around the original state, in case we need to be created again.
	        // But we only do this for pre-Honeycomb apps, which always save their state when
	        // pausing, so we can not have them save their state when restarting from a paused
	        // state. For HC and later, we want to (and can) let the state be saved as the
	        // normal part of stopping the activity.
	        if (r.isPreHoneycomb()) {
	            r.state = oldState;
	        }
	    }
	 
	    return shouldSaveState ? r.state : null;
	}

	private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
	    if (r.paused) {
	        // You are already paused silly...
	        return;
	    }
	 
	    // Always reporting top resumed position loss when pausing an activity. If necessary, it
	    // will be restored in performResumeActivity().
	    reportTopResumedActivityChanged(r, false /* onTop */, "pausing");
	 
	    r.activity.mCalled = false;
	    // 参考【第8.1节】
	    mInstrumentation.callActivityOnPause(r.activity);
	    // 设置ActivityClientRecord状态为ON_PAUSE
	    r.setState(ON_PAUSE);
	}

### 7.2 handleStopActivity

	@Override
	public void handleStopActivity(IBinder token, boolean show, int configChanges,
	        PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
	    // 获取当前Activity
	    final ActivityClientRecord r = mActivities.get(token);
	    r.activity.mConfigChangeFlags |= configChanges;
	 
	    final StopInfo stopInfo = new StopInfo();
	    // 执行stop操作. 参考【第7.2.1节】
	    performStopActivityInner(r, stopInfo, show, true /* saveState */, finalStateRequest, reason);
	 
	    // 设置当前Activity的mDecor可见性为INVISIBLE
	    updateVisibility(r, show);
	 
	    // Make sure any pending writes are now committed.
	    if (!r.isPreHoneycomb()) {
	        QueuedWork.waitToFinish();
	    }
	 
	    // 设置和保存stopInfo
	    stopInfo.setActivity(r);
	    stopInfo.setState(r.state);
	    stopInfo.setPersistentState(r.persistentState);
	    pendingActions.setStopInfo(stopInfo);
	    mSomeActivitiesChanged = true;
	}

#### 7.2.1 performStopActivityInner

	private void performStopActivityInner(ActivityClientRecord r, StopInfo info, boolean keepShown, boolean saveState, boolean finalStateRequest, String reason) {
	    if (r != null) {
	        ...
	        // 先确保已经pause了
	        performPauseActivityIfNeeded(r, reason);
	         
	        if (!keepShown) {
	            // 调用Activity.OnStop
	            callActivityOnStop(r, saveState, reason);
	        }
	    }
	}

#### 7.2.2 callActivityOnStop

	private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
	    // Before P onSaveInstanceState was called before onStop, starting with P it's
	    // called after. Before Honeycomb state was always saved before onPause.
	    final boolean shouldSaveState = saveState && !r.activity.mFinished && r.state == null && !r.isPreHoneycomb();
	    final boolean isPreP = r.isPreP();
	    if (shouldSaveState && isPreP) {
	        // Android P以前版本, 调用Activity.OnSaveInstanceState
	        callActivityOnSaveInstanceState(r);
	    }
	     
	    // 执行stop操作. 参考【第9.2节】
	    r.activity.performStop(r.mPreserveWindow, reason);
	     
	    // 设置ActivityClientRecord状态为ON_STOP
	    r.setState(ON_STOP);
	 
	    if (shouldSaveState && !isPreP) {
	        // Android P及以后版本, 调用Activity.OnSaveInstanceState
	        callActivityOnSaveInstanceState(r);
	    }
	}

### 7.3 handleDestroyActivity

    @Override
    public void handleDestroyActivity(IBinder token, boolean finishing, int configChanges,
            boolean getNonConfigInstance, String reason) {
        ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance, reason);
        if (r != null) {
	        // 清除Window列表
            cleanUpPendingRemoveWindows(r, finishing);
            WindowManager wm = r.activity.getWindowManager();
            View v = r.activity.mDecor;
            if (v != null) {
                if (r.activity.mVisibleFromServer) {
                    // 可见Activity数减1
                    mNumVisibleActivities--;
                }
                IBinder wtoken = v.getWindowToken();
                if (r.activity.mWindowAdded) {
                    if (r.mPreserveWindow) {
                        // 保存当前Window，直到新启动的Activity的Window added
                        r.mPendingRemoveWindow = r.window;
                        r.mPendingRemoveWindowManager = wm;
                        // 清除ContentView
                        r.window.clearContentView();
                    } else {
                        // 清除View
                        wm.removeViewImmediate(v);
                    }
                }
                // 清除View
                if (wtoken != null && r.mPendingRemoveWindow == null) {
                    WindowManagerGlobal.getInstance().closeAll(wtoken,
                            r.activity.getClass().getName(), "Activity");
                } else if (r.mPendingRemoveWindow != null) {
                    WindowManagerGlobal.getInstance().closeAllExceptView(token, v,
                            r.activity.getClass().getName(), "Activity");
                }
                // DecorView置空
                r.activity.mDecor = null;
            }
            if (r.mPendingRemoveWindow == null) {
                WindowManagerGlobal.getInstance().closeAll(token,
                        r.activity.getClass().getName(), "Activity");
            }

            Context c = r.activity.getBaseContext();
            if (c instanceof ContextImpl) {
                ((ContextImpl) c).scheduleFinalCleanup(
                        r.activity.getClass().getName(), "Activity");
            }
        }
        if (finishing) {
            ActivityTaskManager.getService().activityDestroyed(token);
        }
        mSomeActivitiesChanged = true;
    }

#### 7.3.1 performDestroyActivity

    ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        Class<? extends Activity> activityClass = null;
        if (r != null) {
            activityClass = r.activity.getClass();
            r.activity.mConfigChangeFlags |= configChanges;
            if (finishing) {
                r.activity.mFinished = true;
            }

            // 先确保已经Pause了
            performPauseActivityIfNeeded(r, "destroy");

            if (!r.stopped) {
                // 如果没有stop，先调用Activity.onStop
                callActivityOnStop(r, false /* saveState */, "destroy");
            }
            if (getNonConfigInstance) {
                r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
            }
            r.activity.mCalled = false;
            // 参考【第8.3节】
            mInstrumentation.callActivityOnDestroy(r.activity);
            if (r.window != null) {
                r.window.closeAllPanels();
            }
            r.setState(ON_DESTROY);
        }
        schedulePurgeIdler();
        synchronized (mResourcesManager) {
            // 移除token
            mActivities.remove(token);
        }
        StrictMode.decrementExpectedActivityCount(activityClass);
        return r;
    }

#### 7.3.2 ActivityTaskManagerService.activityDestroyed

    @Override
    public final void activityDestroyed(IBinder token) {
        synchronized (mGlobalLock) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityDestroyedLocked(token, "activityDestroyed");
            }
        }
    }

#### 7.3.3 ActivityStack.activityDestroyedLocked

    final void activityDestroyedLocked(IBinder token, String reason) {
        final long origId = Binder.clearCallingIdentity();
        try {
            activityDestroyedLocked(ActivityRecord.forTokenLocked(token), reason);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }

	final void activityDestroyedLocked(ActivityRecord record, String reason) {
        if (record != null) {
            mHandler.removeMessages(DESTROY_TIMEOUT_MSG, record);
        }

        // 从栈中清除Activity
        if (isInStackLocked(record) != null) {
            if (record.isState(DESTROYING, DESTROYED)) {
                cleanUpActivityLocked(record, true, false);
                removeActivityFromHistoryLocked(record, reason);
            }
        }

        // resume栈中位于顶部的Activity
        mRootActivityContainer.resumeFocusedStacksTopActivities();
    }

## 8.Instrumentation
[ -> frameworks/base/core/java/android/app/Instrumentation.java ]

### 8.1 callActivityOnPause

    public void callActivityOnPause(Activity activity) {
        // 参考【第9.1节】
        activity.performPause();
    }

### 8.2 callActivityOnStop

    public void callActivityOnStop(Activity activity) {
	    // 参考【第9.2节】
        activity.onStop();
    }

### 8.3 callActivityOnDestroy

    public void callActivityOnDestroy(Activity activity) {
        // 参考【第9.3节】
        activity.performDestroy();
	}

## 9.Activity
[ -> frameworks/base/core/java/android/app/Activity.java ]

### 9.1 performPause

    final void performPause() {
        dispatchActivityPrePaused();
        mFragments.dispatchPause();
        mCalled = false;
        onPause();
        mResumed = false;
        dispatchActivityPostPaused();
    }

#### 9.1.1 dispatchActivityPrePaused

    private void dispatchActivityPrePaused() {
        getApplication().dispatchActivityPrePaused(this);
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPrePaused(this);
            }
        }
    }

#### 9.1.2 FragmentController.dispatchPause

参考【第10节】

#### 9.1.3 onPause

    protected void onPause() {
        dispatchActivityPaused();
		...
        mCalled = true;
    }

#### 9.1.4 dispatchActivityPaused

    private void dispatchActivityPaused() {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPaused(this);
            }
        }
        getApplication().dispatchActivityPaused(this);
    }

#### 9.1.5 dispatchActivityPostPaused

    private void dispatchActivityPostPaused() {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostPaused(this);
            }
        }
        getApplication().dispatchActivityPostPaused(this);
    }

### 9.2 performStop

	final void performStop(boolean preserveWindow, String reason) {
	    ...
	    if (!mStopped) {
	        dispatchActivityPreStopped();
	        if (mWindow != null) {
	            mWindow.closeAllPanels();
	        }
	        ...
	        mFragments.dispatchStop();
	        mCalled = false;
	        mInstrumentation.callActivityOnStop(this);
	        ...
	        mStopped = true;
	        dispatchActivityPostStopped();
	    }
	    mResumed = false;
	}

#### 9.2.1 dispatchActivityPreStopped

	private void dispatchActivityPreStopped() {
	    getApplication().dispatchActivityPreStopped(this);
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = callbacks.length - 1; i >= 0; i--) {
	            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPreStopped(this);
	        }
	    }
	}

#### 9.2.2 FragmentController.dispatchStop

参考【第10节】

#### 9.2.3 Instrumentation.callActivityOnStop

参考【第8.2节】

#### 9.2.4 dispatchActivityPostStopped

	private void dispatchActivityPostStopped() {
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = callbacks.length - 1; i >= 0; i--) {
	            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostStopped(this);
	        }
	    }
	    getApplication().dispatchActivityPostStopped(this);
	}

### 9.3 performDestroy

    final void performDestroy() {
        dispatchActivityPreDestroyed();
        mDestroyed = true;
        mWindow.destroy();
        mFragments.dispatchDestroy();
        onDestroy();
        ...
        dispatchActivityPostDestroyed();
    }

#### 9.3.1 dispatchActivityPreDestroyed

    private void dispatchActivityPreDestroyed() {
        getApplication().dispatchActivityPreDestroyed(this);
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i])
                        .onActivityPreDestroyed(this);
            }
        }
    }

#### 9.3.2 FragmentController.dispatchDestroy

参考【第10节】

#### 9.3.3 onDestroy

    protected void onDestroy() {
        mCalled = true;

        // dismiss any dialogs we are managing.
        if (mManagedDialogs != null) {
            final int numDialogs = mManagedDialogs.size();
            for (int i = 0; i < numDialogs; i++) {
                final ManagedDialog md = mManagedDialogs.valueAt(i);
                if (md.mDialog.isShowing()) {
                    md.mDialog.dismiss();
                }
            }
            mManagedDialogs = null;
        }

        // close any cursors we are managing.
        synchronized (mManagedCursors) {
            int numCursors = mManagedCursors.size();
            for (int i = 0; i < numCursors; i++) {
                ManagedCursor c = mManagedCursors.get(i);
                if (c != null) {
                    c.mCursor.close();
                }
            }
            mManagedCursors.clear();
        }

        // Close any open search dialog
        if (mSearchManager != null) {
            mSearchManager.stopSearch();
        }

        if (mActionBar != null) {
            mActionBar.onDestroy();
        }

        dispatchActivityDestroyed();
        ...
    }

#### 9.3.4 dispatchActivityDestroyed

    private void dispatchActivityDestroyed() {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityDestroyed(this);
            }
        }
        getApplication().dispatchActivityDestroyed(this);
    }

#### 9.3.5 dispatchActivityPostDestroyed

    private void dispatchActivityPostDestroyed() {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = callbacks.length - 1; i >= 0; i--) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i])
                        .onActivityPostDestroyed(this);
            }
        }
        getApplication().dispatchActivityPostDestroyed(this);
    }

## 10.FragmentController
[ -> frameworks/base/core/java/android/app/FragmentController.java ]

### 10.1 dispatchPause/dispatchStop/dispatchDestroyView/dispatchDestroy

    public void dispatchPause() {
        mHost.mFragmentManager.dispatchPause();
    }

    public void dispatchStop() {
        mHost.mFragmentManager.dispatchStop();
    }

    public void dispatchDestroyView() {
        mHost.mFragmentManager.dispatchDestroyView();
    }

    public void dispatchDestroy() {
        mHost.mFragmentManager.dispatchDestroy();
    }

## 11.FragmentManager
[ -> frameworks/base/core/java/android/app/FragmentManager.java ]

### 11.1 dispatchPause/dispatchStop/dispatchDestroyView/dispatchDestroy

    public void dispatchPause() {
        dispatchMoveToState(Fragment.STARTED);
    }
    
    public void dispatchStop() {
        dispatchMoveToState(Fragment.STOPPED);
    }
    
    public void dispatchDestroyView() {
        dispatchMoveToState(Fragment.CREATED);
    }

    public void dispatchDestroy() {
        mDestroyed = true;
        execPendingActions();
        dispatchMoveToState(Fragment.INITIALIZING);
        mHost = null;
        mContainer = null;
        mParent = null;
    }

## 12.ActivityLifecycleCallbacks
[ -> frameworks/base/core/java/android/app/Application.java ]

	public interface ActivityLifecycleCallbacks {
	
	    default void onActivityPreCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState) {
	    }
	
	    void onActivityCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState);
	
	    default void onActivityPostCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState) {
	    }
	
	    default void onActivityPreStarted(@NonNull Activity activity) {
	    }
	
	    void onActivityStarted(@NonNull Activity activity);
	
	    default void onActivityPostStarted(@NonNull Activity activity) {
	    }
	
	    default void onActivityPreResumed(@NonNull Activity activity) {
	    }
	
	    void onActivityResumed(@NonNull Activity activity);
	
	    default void onActivityPostResumed(@NonNull Activity activity) {
	    }
	
	    default void onActivityPrePaused(@NonNull Activity activity) {
	    }
	
	    void onActivityPaused(@NonNull Activity activity);
	
	    default void onActivityPostPaused(@NonNull Activity activity) {
	    }
	
	    default void onActivityPreStopped(@NonNull Activity activity) {
	    }
	
	    void onActivityStopped(@NonNull Activity activity);
	
	    default void onActivityPostStopped(@NonNull Activity activity) {
	    }
	
	    default void onActivityPreSaveInstanceState(@NonNull Activity activity, @NonNull Bundle outState) {
	    }
	
	    void onActivitySaveInstanceState(@NonNull Activity activity, @NonNull Bundle outState);
	
	    default void onActivityPostSaveInstanceState(@NonNull Activity activity, @NonNull Bundle outState) {
	    }
	
	    default void onActivityPreDestroyed(@NonNull Activity activity) {
	    }
	
	    void onActivityDestroyed(@NonNull Activity activity);
	
	    default void onActivityPostDestroyed(@NonNull Activity activity) {
	    }
	}

## 13.ActivityLifecycleItem
[ -> frameworks/base/core/java/android/app/servertransaction/ActivityLifecycleItem.java ]

	public abstract class ActivityLifecycleItem extends ClientTransactionItem {
	    public static final int UNDEFINED = -1;
	    public static final int PRE_ON_CREATE = 0;
	    public static final int ON_CREATE = 1;
	    public static final int ON_START = 2;
	    public static final int ON_RESUME = 3;
	    public static final int ON_PAUSE = 4;
	    public static final int ON_STOP = 5;
	    public static final int ON_DESTROY = 6;
	    public static final int ON_RESTART = 7;
	}


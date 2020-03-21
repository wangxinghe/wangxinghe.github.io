---
layout: post
comments: true
title: "Activity启动过程"
description: "Activity启动过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

Activity之间的启动分为`不同进程间`和`相同进程`的启动.   
由于进程间Activity启动流程包括了相同进程间Activity启动的流程. 因此本文只分析不同进程间, 即点击Launcher桌面图标启动Activity的这整个过程.

## 0.文件结构

	packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
	packages/apps/Launcher3/src/com/android/launcher3/touch/ItemClickHandler.java
 
	frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStartController.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java
	frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityDisplay.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
	frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java
 
	frameworks/base/core/java/android/app/Activity.java
	frameworks/base/core/java/android/app/Instrumentation.java
	frameworks/base/core/java/android/app/ActivityTaskManager.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/AppComponentFactory.java
	frameworks/base/core/java/android/app/FragmentController.java
 
	frameworks/base/core/java/android/app/ClientTransactionHandler.java
	frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java
	frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
	frameworks/base/core/java/android/app/servertransaction/TransactionExecutorHelper.java
	frameworks/base/core/java/android/app/servertransaction/ActivityLifecycleItem.java
	frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java

## 1.时序图
TODO

## 2.启动入口

### 2.1 Launcher.createShortcut
[ -> packages/apps/Launcher3/src/com/android/launcher3/Launcher.java ]

	public class Launcher extends BaseDraggingActivity implements LauncherExterns,
        LauncherModel.Callbacks, LauncherProviderChangeListener, UserEventDelegate,
        InvariantDeviceProfile.OnIDPChangeListener {
        ...
		public View createShortcut(ViewGroup parent, WorkspaceItemInfo info) {
	        BubbleTextView favorite = (BubbleTextView) LayoutInflater.from(parent.getContext())
			        .inflate(R.layout.app_icon, parent, false);
	        favorite.applyFromWorkspaceItem(info);
	        favorite.setOnClickListener(ItemClickHandler.INSTANCE);
	        favorite.setOnFocusChangeListener(mFocusHandler);
	        return favorite;
	    }
	}

### 2.2 ItemClickHandler
[ -> packages/apps/Launcher3/src/com/android/launcher3/touch/ItemClickHandler.java ]

    public class ItemClickHandler {
 
	    public static final OnClickListener INSTANCE = getInstance(null);
 
	    public static final OnClickListener getInstance(String sourceContainer) {
	        return v -> onClick(v, sourceContainer);
	    }
 
	    private static void onClick(View v, String sourceContainer) {
	        Launcher launcher = Launcher.getLauncher(v.getContext());
 
	        Object tag = v.getTag();
	        if (tag instanceof WorkspaceItemInfo) {
	            onClickAppShortcut(v, (WorkspaceItemInfo) tag, launcher, sourceContainer);
	        } else if (tag instanceof FolderInfo) {
	            ...
	        } else if (tag instanceof AppInfo) {
	            ...
	        } else if (tag instanceof LauncherAppWidgetInfo) {
	            ...
	        }
	    }
 
	    public static void onClickAppShortcut(View v, WorkspaceItemInfo shortcut,
			    Launcher launcher, @Nullable String sourceContainer) {
	        startAppShortcutOrInfoActivity(v, shortcut, launcher, sourceContainer);
	    }
 
	    private static void startAppShortcutOrInfoActivity(View v, ItemInfo item,
			    Launcher launcher, @Nullable String sourceContainer) {
	        Intent intent = item.getIntent();
	        launcher.startActivitySafely(v, intent, item, sourceContainer);
	    }
	}

### 2.3 Launcher.startActivitySafely

	public boolean startActivitySafely(View v, Intent intent, ItemInfo item, @Nullable String sourceContainer) {
	    boolean success = super.startActivitySafely(v, intent, item, sourceContainer);
	    return success;
	}
 
	public boolean startActivitySafely(View v, Intent intent, @Nullable ItemInfo item, @Nullable String sourceContainer) {
	    ...
	    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	    try {
	        ...
	        if (isShortcut) {
	            startShortcutIntentSafely(intent, optsBundle, item, sourceContainer);
	        } else if (user == null || user.equals(Process.myUserHandle())) {
	            startActivity(intent, optsBundle);
	        } else {
	            LauncherAppsCompat.getInstance(this).startActivityForProfile(
			            intent.getComponent(), user, intent.getSourceBounds(), optsBundle);
	        }
	        return true;
	    } catch (ActivityNotFoundException|SecurityException e) {
	        ...
	    }
	    return false;
	}

## 3.Activity
[ -> frameworks/base/core/java/android/app/Activity.java ]

### 3.1 startActivity

	@Override
	public void startActivity(Intent intent) {
	    this.startActivity(intent, null);
	}
 
	@Override
	public void startActivity(Intent intent, @Nullable Bundle options) {
	    if (options != null) {
	        startActivityForResult(intent, -1, options);
	    } else {
	        startActivityForResult(intent, -1);
	    }
	}
 
### 3.2 startActivityForResult
 
	public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
	    startActivityForResult(intent, requestCode, null);
	}

	public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
	    if (mParent == null) {
	        // 执行启动Activity，见【第4.1节】
	        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
	        if (ar != null) {
		        // 发送启动结果，见【第21.4节】
	            mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(), ar.getResultData());
	        }
	        if (requestCode >= 0) {
	            mStartedActivity = true;
	        }
	    } else {
	        ...
	    }
	}

## 4.Instrumentation
[ -> frameworks/base/core/java/android/app/Instrumentation.java ]

### 4.1 execStartActivity

Instrumentation#execStartActivity  
(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options)

各参数含义:  
- who: 启动Activity的上下文
- contextThread: 启动Activity的主线程, 即ApplicationThread
- token: 令牌, 标识着由谁启动, 系统唯一标识
- target: 在哪个Activity中启动, 会接收启动完成的结果
- intent: 启动意图
- requestCode: 请求码, 用于将结果和请求对应上
- options: 可选参数
  
代码实现如下:  

	public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
	    IApplicationThread whoThread = (IApplicationThread) contextThread;
 
	    // 遍历ActivityMonitor列表，匹配ActivityResult
	    if (mActivityMonitors != null) {
	        synchronized (mSync) {
	            final int N = mActivityMonitors.size();
	            for (int i=0; i<N; i++) {
	                final ActivityMonitor am = mActivityMonitors.get(i);
	                ActivityResult result = null;
	                // 如果ignoreMatchingSpecificIntents为true
	                if (am.ignoreMatchingSpecificIntents()) {
	                    // 返空
	                    result = am.onStartActivity(intent);
	                }
	                if (result != null) { // 命中结果，直接返回结果
	                    am.mHits++;
	                    return result;
	                } else if (am.match(who, null, intent)) {
	                    am.mHits++;
	                    // 当ActivityMonitor阻塞Activity启动时，直接返回结果
	                    if (am.isBlocking()) {
	                        return requestCode >= 0 ? am.getResult() : null;
	                    }
	                    // 否则跳出循环，走后面startActivity流程
	                    break;
	                }
	            }
	        }
	    }
 
	    // ActivityTaskManagerService.startActivity
	    int result = ActivityTaskManager.getService()
	        .startActivity(whoThread, who.getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
	    // 检查result, 判断Activity是否启动成功.
	    checkStartActivityResult(result, intent);
 
	    return null;
	}

## 5.ActivityTaskManager
[ -> frameworks/base/core/java/android/app/ActivityTaskManager.java ]

### 5.1 getService

	private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
	    new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                // 拿到BinderProxy对象
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                // 拿到new IActivityTaskManager.Stub.Proxy(new BinderProxy(handle))对象
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
 
	// 拿到new IActivityTaskManager.Stub.Proxy(new BinderProxy(handle))对象
	public static IActivityTaskManager getService() {
	    return IActivityTaskManagerSingleton.get();
	}

上面代码的含义:  
ActivityTaskManager.getService()得到IActivityTaskManager.Stub.Proxy对象   
ActivityTaskManager.getService().startActivity()等价于IActivityTaskManager.Stub.Proxy.startActivity()  
再根据AIDL分析过程可知，最终会走到ActivityTaskManagerService.startActivity()  

至此ActivityTaskManager.getService()拿到服务的代理对象new IActivityTaskManager.Stub.Proxy(new BinderProxy(handle))  
IActivityTaskManager.Stub.Proxy#startActivity通过AIDL最终调用到ActivityTaskManagerService#startActivity

## 6.ActivityTaskManagerService
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java ]

### 6.1 startActivity

ActivityTaskManagerService#startActivity  
(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions)

各参数含义:  
- caller: 启动Activity的线程, 即ApplicationThread
- callingPackage: 启动Activity的当前包名 
- intent: 启动意图
- resolvedType: 启动意图的MIME Type
- resultTo: 结果返回给谁(即启动Activity的当前Activity token), 类型为IBinder
- resultWho: 结果返回给谁(即启动Activity的当前Activity id), 类型为String
- requestCode: 请求码, 用于将request和response对应上
- startFlags: 启动flags
- profilerInfo: Profiler信息
- bOptions: 可选参数

代码实现如下:

	@Override
	public final int startActivity(IApplicationThread caller, String callingPackage,
	        Intent intent, String resolvedType, IBinder resultTo, String resultWho,
	        int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
	    // 指定启动Activity的userId
	    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
	            resultWho, requestCode, startFlags, profilerInfo, bOptions,
	            UserHandle.getCallingUserId());
	}
 
	@Override
	public int startActivityAsUser(IApplicationThread caller, String callingPackage,
	        Intent intent, String resolvedType, IBinder resultTo, String resultWho, 
	        int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions,
	        int userId) {
	    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
	            resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
	            true /*validateIncomingUser*/);
	}
 
	int startActivityAsUser(IApplicationThread caller, String callingPackage,
	        Intent intent, String resolvedType, IBinder resultTo, String resultWho, 
	        int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, 
	        int userId, boolean validateIncomingUser) {
	    // 确保调用进程不是isolated
	    enforceNotIsolatedCaller("startActivityAsUser");
	    // 检查userId的有效性, 并返回有效的userId
	    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
	            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
 
	    // ActivityStartController#obtainStarter()，拿到ActivityStarter
	    // ActivityStarter#execute()，启动Activity
	    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
	            .setCaller(caller)
	            .setCallingPackage(callingPackage)
	            .setResolvedType(resolvedType)
	            .setResultTo(resultTo)
	            .setResultWho(resultWho)
	            .setRequestCode(requestCode)
	            .setStartFlags(startFlags)
	            .setProfilerInfo(profilerInfo)
	            .setActivityOptions(bOptions)
	            .setMayWait(userId) // mRequest.mayWait = true
	            .execute();
	}

## 7.ActivityStarter
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java ]

### 7.1 execute

	int execute() {
	    try {
	        if (mRequest.mayWait) { // 需要等待请求返回结果
	            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
	                    mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
	                    mRequest.intent, mRequest.resolvedType,
	                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
	                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
	                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
	                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
	                    mRequest.inTask, mRequest.reason,
	                    mRequest.allowPendingRemoteAnimationRegistryLookup,
	                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
	        } else {
	            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
	                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
	                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
	                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
	                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
	                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
	                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
	                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
	                    mRequest.allowPendingRemoteAnimationRegistryLookup,
	                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
	        }
	    } finally {
	        // 执行完成
	        onExecutionComplete();
	    }
	}

### 7.2 startActivityMayWait

	private int startActivityMayWait(IApplicationThread caller, int callingUid,
	        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
	        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
	        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
	        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
	        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
	        int userId, TaskRecord inTask, String reason,
	        boolean allowPendingRemoteAnimationRegistryLookup,
	        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
	    ...
		ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
			computeResolveFilterUid(callingUid, realCallingUid, mRequest.filterCallingUid));
	    // Collect information about the target of the Intent.
	    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
	    ...
	    final ActivityRecord[] outRecord = new ActivityRecord[1];
	    int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
	            voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
	            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
	            ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
	            allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
	            allowBackgroundActivityStart);
 
	    return res;
	}

	private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
	        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
	        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
	        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
	        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
	        SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
	        ActivityRecord[] outActivity, TaskRecord inTask, String reason,
	        boolean allowPendingRemoteAnimationRegistryLookup,
	        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
	    ...
	    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
	            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
	            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid,startFlags,
	            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
	            inTask, allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
	            allowBackgroundActivityStart);
 
	    return getExternalResult(mLastStartActivityResult);
	}

	private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
	        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
	        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
	        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
	        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
	        SafeActivityOptions options,
	        boolean ignoreTargetSecurity, boolean componentSpecified,ActivityRecord[] outActivity,
	        TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
	        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
	    ...
	    // 下面这段代码是确定源Activity和目的Activity, 通常sourceRecord和resultRecord是同一个
	    // 源Activity, 启动Activity的那个Activity
	    ActivityRecord sourceRecord = null;
	    // 目的Activity, 将启动结果返回给它
	    ActivityRecord resultRecord = null;
	    // 根据启动Activity的页面token, 判断Activity Stack中是否有对应的源Activity
	    // 如果requestCode>=0且源Activity非空且非finish, 则目的Activity即为源Activity
	    if (resultTo != null) {
	        sourceRecord = mRootActivityContainer.isInAnyStack(resultTo);
	        if (sourceRecord != null) {
	            if (requestCode >= 0 && !sourceRecord.finishing) {
	                resultRecord = sourceRecord;
	            }
	        }
	    }
	    ...
	    // 拦截器处理
	    if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid, callingUid, checkedOptions)) {
	        ...
	    }
 
		// 创建ActivityRecord
	    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
		        callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
		        resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
		        mSupervisor, checkedOptions, sourceRecord);
	    // 启动Activity
	    final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
			    true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);
 
	    return res;
	}

	private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity, boolean restrictedBgActivity) {
	    int result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
	            startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
	    postStartActivityProcessing(r, result, startedActivityStack); 
	    return result;
	}
 
	private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
	        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
	        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
	        ActivityRecord[] outActivity, boolean restrictedBgActivity) {
	    ...
	    // 将要启动的Activity(即mStartActivity)添加到task里, 同时将该task移到task列表的最前面
	    // 这个task可能是新建的task, 也可能是启动Activity的当前Activity所在的task
	    int result = START_SUCCESS;
	    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
	        // 复用task或创建新的task
	        newTask = true;
	        result = setTaskFromReuseOrCreateNewTask(taskToAffiliate);
	    } else if (mSourceRecord != null) {
	        // 从mSourceRecord中获取task
	        result = setTaskFromSourceRecord();
	    } else if (mInTask != null) {
	        // 从mInTask中获取task
	        result = setTaskFromInTask();
	    } else {
	        // 从mHistoryTasks栈顶获取task或创建新的task
	        result = setTaskToCurrentTopOrCreateNewTask();
	    }
	    ...
	    // 启动Activity的准备工作.见【第8.1节】
	    mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition, mOptions);
	    if (mDoResume) { // resume操作, 此处为true
	        // 获取当前正在运行的位于栈顶的Activity
	        final ActivityRecord topTaskActivity = mStartActivity.getTaskRecord().topRunningActivityLocked();
	        
	        if (!mTargetStack.isFocusable() || (topTaskActivity != null && topTaskActivity.mTaskOverlay && mStartActivity != topTaskActivity)) {
		        // 设置启动的Activity(即mStartActivity)可见, 并通知WindowManager更新
	            mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);
	            mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
	        } else {
	            // 若mTargetStack栈当前没有显示在最上层并获取焦点, 则先将mTargetStack移到栈顶
	            if (mTargetStack.isFocusable() && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
	                mTargetStack.moveToFront("startActivityUnchecked");
	            }
	            // resume mStartActivity. 见【第8.2节】
	            mRootActivityContainer.resumeFocusedStacksTopActivities(mTargetStack, mStartActivity, mOptions);
	        }
	    } else if (mStartActivity != null) {
	        // 将启动的Activity所在的task添加到最近任务列表mRecentTasks里
	        mSupervisor.mRecentTasks.add(mStartActivity.getTaskRecord());
	    }
 
	    return START_SUCCESS;
	}

### 7.3 setTaskFromXXX

setTaskFromXXX主要做的事情就是:   
(1) `addOrReparentStartingActivity` 将要启动的Activity添加到Task的Activity列表栈顶
(2) `mTargetStack.moveToFront` 将Task移动到任务列表最上面

各方法含义:  
- `setTaskFromReuseOrCreateNewTask`  复用mReuseTask或新建TaskRecord
- `setTaskFromSourceRecord` 复用源Activity(mSourceRecord)所在的Task
- `setTaskFromInTask` 复用mInTask
- `setTaskToCurrentTopOrCreateNewTask` 新建TaskRecord

具体代码如下:

	private int setTaskFromReuseOrCreateNewTask(TaskRecord taskToAffiliate) {
	    if (mReuseTask == null) {
	        final TaskRecord task = mTargetStack.createTaskRecord(
	                mSupervisor.getNextTaskIdForUserLocked(mStartActivity.mUserId),
	                mNewTaskInfo != null ? mNewTaskInfo : mStartActivity.info,
	                mNewTaskIntent != null ? mNewTaskIntent : mIntent, mVoiceSession,
	                mVoiceInteractor, !mLaunchTaskBehind, mStartActivity, mSourceRecord,
	                mOptions);
	        addOrReparentStartingActivity(task, "setTaskFromReuseOrCreateNewTask-mReuseTask");
	    } else {
	        addOrReparentStartingActivity(mReuseTask, "setTaskFromReuseOrCreateNewTask");
	    }
 
	    if (taskToAffiliate != null) {
	        mStartActivity.setTaskToAffiliateWith(taskToAffiliate);
	    }
 
	    if (mDoResume) {
	        mTargetStack.moveToFront("reuseOrNewTask");
	    }
	    return START_SUCCESS;
	}

	private int setTaskFromSourceRecord() {
	    final TaskRecord sourceTask = mSourceRecord.getTaskRecord();
	    final ActivityStack sourceStack = mSourceRecord.getActivityStack();
 
	    if (mTargetStack == null) {
	        mTargetStack = sourceStack;
	    } else if (mTargetStack != sourceStack) {
	        sourceTask.reparent(mTargetStack, ON_TOP, REPARENT_MOVE_STACK_TO_FRONT, !ANIMATE, DEFER_RESUME, "launchToSide");
	    }
 
	    final TaskRecord topTask = mTargetStack.topTask();
	    if (topTask != sourceTask && !mAvoidMoveToFront) {
	        mTargetStack.moveTaskToFrontLocked(sourceTask, mNoAnimation, mOptions, mStartActivity.appTimeTracker, "sourceTaskToFront");
	    } else if (mDoResume) {
	        mTargetStack.moveToFront("sourceStackToFront");
	    }
	    addOrReparentStartingActivity(sourceTask, "setTaskFromSourceRecord");
	    return START_SUCCESS;
	}

	private int setTaskFromInTask() {
	    mTargetStack = mInTask.getStack();
	    ...
	    mTargetStack.moveTaskToFrontLocked(mInTask, mNoAnimation, mOptions, mStartActivity.appTimeTracker, "inTaskToFront");
	    addOrReparentStartingActivity(mInTask, "setTaskFromInTask");
 
	    return START_SUCCESS;
	}

	private int setTaskToCurrentTopOrCreateNewTask() {
	    ...
	    if (mDoResume) {
	        mTargetStack.moveToFront("addingToTopTask");
	    }
	    final ActivityRecord prev = mTargetStack.getTopActivity();
	    final TaskRecord task = (prev != null)
	            ? prev.getTaskRecord() : mTargetStack.createTaskRecord(
	            mSupervisor.getNextTaskIdForUserLocked(mStartActivity.mUserId),
	            mStartActivity.info,
	            mIntent, null, null, true, mStartActivity, mSourceRecord, mOptions);
	    addOrReparentStartingActivity(task, "setTaskToCurrentTopOrCreateNewTask");
	    ...
	    return START_SUCCESS;
	}

## 8.ActivityStack
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java ]

### 8.1 startActivityLocked

	void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity, boolean newTask, boolean keepCurTransition, ActivityOptions options) {
	    TaskRecord rTask = r.getTaskRecord();
	    final int taskId = rTask.taskId;
	    if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
	        // 将task添加到mTaskHistory任务列表栈顶
	        insertTaskAtTop(rTask, r);
	    }
	    TaskRecord task = null;
	    // 这段代码的含义: 如果是已有的task, 从mTaskHistory中查找目标task
	    // 并判断目标task中是否已经启动了Activity, 如果发现未启动则调用createAppWindowToken, 等待后续启动
	    if (!newTask) {
	        // If starting in an existing task, find where that is...
	        boolean startIt = true;
	        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
	            task = mTaskHistory.get(taskNdx);
	            if (task.getTopActivity() == null) {
	                // All activities in task are finishing.
	                continue;
	            }
	            if (task == rTask) {
	                // Here it is!  Now, if this is not yet visible to the
	                // user, then just add it without starting; it will
	                // get started when the user navigates back to it.
	                if (!startIt) {
	                    r.createAppWindowToken();
	                    ActivityOptions.abort(options);
	                    return;
	                }
	                break;
	            } else if (task.numFullscreen > 0) {
	                startIt = false;
	            }
	        }
	    }
 
		final TaskRecord activityTask = r.getTaskRecord();
	    task = activityTask;
	    task.setFrontOfTask();
	    ...
	    // 下面这段代码的含义: 做一些启动Activity的过渡工作, 比如动画之类的.
	    if (!isHomeOrRecentsStack() || numActivities() > 0) {
	        final DisplayContent dc = getDisplay().mDisplayContent;
	        if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
	            dc.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
	            mStackSupervisor.mNoAnimActivities.add(r);
	        } else {
	            int transit = TRANSIT_ACTIVITY_OPEN;
	            if (newTask) {
	                if (r.mLaunchTaskBehind) {
	                    transit = TRANSIT_TASK_OPEN_BEHIND;
	                } else {
	                    if (canEnterPipOnTaskSwitch(focusedTopActivity, null, r, options)) {
	                        focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
	                    }
	                    transit = TRANSIT_TASK_OPEN;
	                }
	            }
	            dc.prepareAppTransition(transit, keepCurTransition);
	            mStackSupervisor.mNoAnimActivities.remove(r);
	        }
	        boolean doShow = true;
	        if (newTask) {
	            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
	                resetTaskIfNeededLocked(r, r);
	                doShow = topRunningNonDelayedActivityLocked(null) == r;
	            }
	        } else if (options != null && options.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
	            doShow = false;
	        }
	        if (r.mLaunchTaskBehind) {
	            // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
	            // tell WindowManager that r is visible even though it is at the back of the stack.
	            r.setVisibility(true);
	            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
	        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
	            // Figure out if we are transitioning from another activity that is
	            // "has the same starting icon" as the next one.  This allows the
	            // window manager to keep the previous window it had previously
	            // created, if it still had one.
	            TaskRecord prevTask = r.getTaskRecord();
	            ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
	            if (prev != null) {
	                // We don't want to reuse the previous starting preview if:
	                // (1) The current activity is in a different task.
	                if (prev.getTaskRecord() != prevTask) {
	                    prev = null;
	                }
	                // (2) The current activity is already displayed.
	                else if (prev.nowVisible) {
	                    prev = null;
	                }
	            }
	            r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
	        }
	    } else {
	        // If this is the first activity, don't do any fancy animations,
	        // because there is nothing for it to animate on top of.
	        ActivityOptions.abort(options);
	    }
	}

### 8.2 RootActivityContainer.resumeFocusedStacksTopActivities
[ -> frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java ]

	boolean resumeFocusedStacksTopActivities(ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
	    ...
	    boolean result = false;
	    if (targetStack != null && (targetStack.isTopStackOnDisplay() || getTopDisplayFocusedStack() == targetStack)) {
	        result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
	    }
 
	    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
	        boolean resumedOnDisplay = false;
	        final ActivityDisplay display = mActivityDisplays.get(displayNdx);
	        for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
	            final ActivityStack stack = display.getChildAt(stackNdx);
	            final ActivityRecord topRunningActivity = stack.topRunningActivityLocked();
	            if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
	                continue;
	            }
	            if (stack == targetStack) {
	                // Simply update the result for targetStack because the targetStack had
	                // already resumed in above. We don't want to resume it again, especially in
	                // some cases, it would cause a second launch failure if app process was dead.
	                resumedOnDisplay |= result;
	                continue;
	            }
	            if (display.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
	                // Kick off any lingering app transitions form the MoveTaskToFront operation,
	                // but only consider the top task and stack on that display.
	                stack.executeAppTransition(targetOptions);
	            } else {
	                resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
	            }
	        }
	        if (!resumedOnDisplay) {
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                }
            }
	    }
	    return result;
	}

这段代码的目的是resumeTopActivity:   
先判断如果是顶层Task, 则直接调用resumeTopActivity方法.   
接着从mActivityDisplays列表从上往下依次遍历Task, 如果发现之前没有resume(即resumedOnDisplay为false), 则调用resumeTopActivity方法.  

最终都会走到resumeTopActivityUncheckedLocked

### 8.3 resumeTopActivityUncheckedLocked

	boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
	    if (mInResumeTopActivity) {
	        return false;
	    }
 
	    boolean result = false;
	    try {
	        // Protect against recursion.
	        mInResumeTopActivity = true;
	        // 见【第8.4节】
	        result = resumeTopActivityInnerLocked(prev, options);
         
	        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
	        if (next == null || !next.canTurnScreenOn()) {
	            checkReadyForSleep();
	        }
	    } finally {
	        mInResumeTopActivity = false;
	    }
 
	    return result;
	}

### 8.4 resumeTopActivityInnerLocked

	private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
	    ...
	    //从任务历史列表mTaskHistory里, 获取最上面运行的Activity
	    ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
	    final boolean hasRunningActivity = next != null;
	    if (!hasRunningActivity) {
	        // 如果没有运行的Activity, 则从其他task堆栈中查找Activity并resume
	        return resumeNextFocusableActivityWhenStackIsEmpty(prev, options);
	    }

	    // 如果topRunningActivity已经是resume状态, 则do nothing
	    if (mResumedActivity == next && next.isState(RESUMED) && display.allResumedActivitiesComplete()) {
	        executeAppTransition(options);
	        return false;
	    }
	    ...
	    // 如果没有获取到topRunningActivity，则先暂停所有Stack中的Activity. 见【第8.5节】
	    boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
	    // 暂停当前恢复态的Activity(mResumedActivity)
	    // 注意prev/next为要启动的目的Activity, 而mResumedActivity为当前Activity, 两者不是同一个
	    if (mResumedActivity != null) {
		    // 见【第8.6节】
	        pausing |= startPausingLocked(userLeaving, false, next, false);
	    }
	    if (pausing && !resumeWhilePausing) {
	        // 如果Activity正在Pausing状态,且不允许pausing过程中执行resume, 则先不执行resume
	        // 只是将即将启动的Activity所在的进程添加到mLruProcesses最前面, 避免被杀
	        if (next.attachedToProcess()) {
	            next.app.updateProcessInfo(false, true, false);
	        }
	        if (lastResumed != null) {
	            lastResumed.setWillCloseOrEnterPip(true);
	        }
	        return true;
	    } else if (mResumedActivity == next && next.isState(RESUMED) && display.allResumedActivitiesComplete()) {
	        // 如果要启动的Activity已经Resume了
	        // 则不需要做什么, 调一下executeAppTransition执行一下pending transitions即可.
	        executeAppTransition(options);
	        return true;
	    }
	    ...
	    // 如果下一个Activity已经可见了, 上一个Activity当前正处于finishing状态, 则直接让上一个Activity不可见
	    // 当上一个Activity不是finishing状态(比如当下一个Activity不是全屏状态时的场景), 上一个Activity还应该是可见的
	    if (prev != null && prev != next && next.nowVisible) {
	        if (prev.finishing) {
	            prev.setVisibility(false);
	        }
	    }
	    ...
	    if (next.attachedToProcess()) {
	        // Activity所在进程已启动, 和ActivityStackSupervisor.realStartActivityLocked做的事情类似.
	        // 上一个Activity是否半透明
	        final boolean lastActivityTranslucent = lastFocusedStack != null
	                && (lastFocusedStack.inMultiWindowMode()
	                || (lastFocusedStack.mLastPausedActivity != null
	                && !lastFocusedStack.mLastPausedActivity.fullscreen));
 
	        // 如果将要启动的Activity不可见, 则让该Activity可见
	        if (!next.visible || next.stopped || lastActivityTranslucent) {
	            next.setVisibility(true);
	        }
	        ...
	        // 设置Activity状态为RESUMED
	        next.setState(RESUMED, "resumeTopActivityInnerLocked");
	        // 更新进程信息
	        next.app.updateProcessInfo(false, true, true);
	        ...
	        final ClientTransaction transaction = ClientTransaction.obtain(next.app.getThread(), next.appToken);
	        // 分发所有pending结果
	        ArrayList<ResultInfo> a = next.results;
	        if (a != null) {
	            final int N = a.size();
	            if (!next.finishing && N > 0) {
	                transaction.addCallback(ActivityResultItem.obtain(a));
	            }
	        }
 
	        // 分发new intent
	        if (next.newIntents != null) {
	            transaction.addCallback(NewIntentItem.obtain(next.newIntents, true /* resume */));
	        }
 
	        // Well the app will no longer be stopped.
	        // Clear app token stopped state in window manager if needed.
	        next.notifyAppResumed(next.stopped);
	        next.sleeping = false;
	        mService.getAppWarningsLocked().onResumeActivity(next);
	        next.app.setPendingUiCleanAndForceProcessStateUpTo(mService.mTopProcessState);
	        next.clearOptionsLocked();
	        // 触发onResume
	       transaction.setLifecycleStateRequest(
			       ResumeActivityItem.obtain(next.app.getReportedProcState(), 
			       getDisplay().mDisplayContent.isNextTransitionForward()));
	        mService.getLifecycleManager().scheduleTransaction(transaction);
 
	        // From this point on, if something goes wrong there is no way
	        // to recover the activity.
	        next.completeResumeLocked();
	    } else { // 进程没启动
	        // Whoops, need to restart this activity!
	        if (!next.hasBeenLaunched) {
	            next.hasBeenLaunched = true;
	        } else {
	            if (SHOW_APP_STARTING_PREVIEW) {
	                next.showStartingWindow(null, false, false);
	            }
	        }
	        // 创建App进程并启动Activity. 见【第9.1节】
	        mStackSupervisor.startSpecificActivityLocked(next, true, true);
	    }
	    return true;
	}

### 8.5 ActivityDisplay.pauseBackStacks
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityDisplay.java ]

	boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
	    boolean someActivityPaused = false;
	    for (int stackNdx = mStacks.size() - 1; stackNdx >= 0; --stackNdx) {
	        final ActivityStack stack = mStacks.get(stackNdx);
	        final ActivityRecord resumedActivity = stack.getResumedActivity();
	        if (resumedActivity != null && (stack.getVisibility(resuming) != STACK_VISIBILITY_VISIBLE || !stack.isFocusable())) {
		        // 见【第8.6节】
	            someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming, dontWait);
	        }
	    }
	    return someActivityPaused;
	}

源码解释:  
Pause all activities in either all of the stacks or just the back stacks. This is done before  
resuming a new activity and to make sure that previously active activities are  
paused in stacks that are no longer visible or in pinned windowing mode. This does not  
pause activities in visible stacks, so if an activity is launched within the same stack/task,  
then we should explicitly pause that stack's top activity.  

### 8.6 startPausingLocked

    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity != null) {
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev = mResumedActivity;

		// Trying to pause when nothing is resumed
        if (prev == null && resuming == null) {
            mRootActivityContainer.resumeFocusedStacksTopActivities();
            return false;
        }

		// Trying to pause activity that is in process of being resumed
        if (prev == resuming) {
            return false;
        }

        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTaskRecord().touchActiveTime();
        clearLaunchTime(prev);

        mService.updateCpuStats();

        if (prev.attachedToProcess()) {
            mService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                    prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                            prev.configChangeFlags, pauseImmediately));
        } else {
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

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;
            } else {
                schedulePauseTimeout(prev);
                return true;
            }
        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            // Activity not running, resuming next.
            if (resuming == null) {
                mRootActivityContainer.resumeFocusedStacksTopActivities();
            }
            return false;
        }
    }

## 9.ActivityStackSupervisor
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java ]

### 9.1 startSpecificActivityLocked

	void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
	    // 获取Activity所在的进程
	    final WindowProcessController wpc = mService.getProcessController(r.processName, r.info.applicationInfo.uid);
 
	    // 如果Activity所在的进程已启动，则直接启动Activity. 见【第15节】
	    if (wpc != null && wpc.hasThread()) {
	        realStartActivityLocked(r, wpc, andResume, checkConfig);
	        return;
	    }
 
	    // 如果Activity所在的进程未启动，则先启动进程. 见【第10.1节】
	    final Message msg = PooledLambda.obtainMessage(
	            ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
	            r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
	    mService.mH.sendMessage(msg);
	}

## 10.ActivityManagerInternal
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

### 10.1 startProcess
	public final class LocalService extends ActivityManagerInternal {
	    @Override
	    public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead, String hostingType, ComponentName hostingName) {
	        synchronized (ActivityManagerService.this) {
	            // HostingRecord描述启动一个进程所需的信息
	            startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
	                    new HostingRecord(hostingType, hostingName),
	                    false /* allowWhileBooting */, false /* isolated */,
	                    true /* keepIfLarge */);
	        }
	    }
	}

	final ProcessRecord startProcessLocked(String processName,
	        ApplicationInfo info, boolean knownToBeDead, int intentFlags,
	        HostingRecord hostingRecord, boolean allowWhileBooting,
	        boolean isolated, boolean keepIfLarge) {
	    return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
	            hostingRecord, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
	            null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
	            null /* crashHandler */);
	}

## 11.ProcessList
[ -> frameworks/base/services/core/java/com/android/server/am/ProcessList.java ]

### 11.1 startProcessLocked

	final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
	        boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
	        boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
	        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
	    ProcessRecord app;
	    if (!isolated) { // isolated=false
	        // 获取进程对象ProcessRecord
	        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
	    } else {
	        // If this is an isolated process, it can't re-use an existing process.
	        app = null;
	    }
	    ...
	    if (app == null) { // 若进程为空，创建新进程
	        app = newProcessRecordLocked(info, processName, isolated, isolatedUid, hostingRecord);
	        app.crashHandler = crashHandler;
	        app.isolatedEntryPoint = entryPoint;
	        app.isolatedEntryPointArgs = entryPointArgs;
	    } else {
	        // If this is a new package in the process, add the package to the list
	        app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
	    }
	    ...
	    // 启动进程
	    final boolean success = startProcessLocked(app, hostingRecord, abiOverride);
	    return success ? app : null;
	}

	final boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord, String abiOverride) {
	    return startProcessLocked(app, hostingRecord, false /* disableHiddenApiChecks */, false /* mountExtStorageFull */, abiOverride);
	}

	boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
	        boolean disableHiddenApiChecks, boolean mountExtStorageFull, String abiOverride) {
	    ...
	    // 进程参数设置，包括uid/gids/mountMode等
	    int uid = app.uid;
	    int[] gids = null;
	    int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
	    if (!app.isolated) {
	        int[] permGids = null;
	        // 对于非isolated进程, 需要设置gids, 便于应用程序之间共享资源
	        if (ArrayUtils.isEmpty(permGids)) {
	            gids = new int[3];
	        } else {
	            gids = new int[permGids.length + 3];
	            System.arraycopy(permGids, 0, gids, 3, permGids.length);
	        }
	        gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
	        gids[1] = UserHandle.getCacheAppGid(UserHandle.getAppId(uid));
	        gids[2] = UserHandle.getUserGid(UserHandle.getUserId(uid));
 
	        // Replace any invalid GIDs
	        if (gids[0] == UserHandle.ERR_GID) gids[0] = gids[2];
	        if (gids[1] == UserHandle.ERR_GID) gids[1] = gids[2];
	    }
	    app.mountMode = mountExternal;
	    app.gids = gids;
	    app.setRequiredAbi(requiredAbi);
	    app.instructionSet = instructionSet;
	    ...
	    // Start the process.  It will either succeed and return a result containing
	    // the PID of the new process, or else throw a RuntimeException.
	    final String entryPoint = "android.app.ActivityThread";
	    return startProcessLocked(hostingRecord, entryPoint, app, uid, gids, runtimeFlags,
			    mountExternal, seInfo, requiredAbi, instructionSet, invokeWith, startTime);
	}

	boolean startProcessLocked(HostingRecord hostingRecord,
	        String entryPoint,
	        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
	        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
	        long startTime) {
	    // 设置进程参数, 并将进程添加到待启动进程列表mPendingStarts
	    app.pendingStart = true;
	    app.killedByAm = false;
	    app.removed = false;
	    app.killed = false;
	    final long startSeq = app.startSeq = ++mProcStartSeqCounter;
	    app.setStartParams(uid, hostingRecord, seInfo, startTime);
	    app.setUsingWrapper(invokeWith != null || SystemProperties.get("wrap." + app.processName) != null);
	    mPendingStarts.put(startSeq, app);
	    ...
	    if (mService.mConstants.FLAG_PROCESS_START_ASYNC) { // true 异步启动进程
	        mService.mProcStartHandler.post(() -> {
	            final Process.ProcessStartResult startResult = startProcess(app.hostingRecord,
	                    entryPoint, app, app.startUid, gids, runtimeFlags, mountExternal,
	                    app.seInfo, requiredAbi, instructionSet, invokeWith, app.startTime);
		            synchronized (mService) {
		                handleProcessStartedLocked(app, startResult, startSeq);
		            }
	        });
	        return true;
	    } else { // 同步启动进程
	        final Process.ProcessStartResult startResult = startProcess(hostingRecord,
			        entryPoint, app, uid, gids, runtimeFlags, mountExternal, seInfo, 
			        requiredAbi, instructionSet, invokeWith, startTime);
	        handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper, startSeq, false);
	        return app.pid > 0;
	    }
	}

	private Process.ProcessStartResult startProcess(HostingRecord hostingRecord,
			String entryPoint, ProcessRecord app, int uid, int[] gids, int runtimeFlags, 
			int mountExternal, String seInfo, String requiredAbi, String instructionSet, 
			String invokeWith, long startTime) {
	    final Process.ProcessStartResult startResult;
	    if (hostingRecord.usesWebviewZygote()) {
	        startResult = startWebView(entryPoint,
	                app.processName, uid, uid, gids, runtimeFlags, mountExternal,
	                app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
	                app.info.dataDir, null, app.info.packageName,
	                new String[] {PROC_START_SEQ_IDENT + app.startSeq});
	    } else if (hostingRecord.usesAppZygote()) {
	        final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);
	        startResult = appZygote.getProcess().start(entryPoint,
	                app.processName, uid, uid, gids, runtimeFlags, mountExternal,
	                app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
	                app.info.dataDir, null, app.info.packageName,
	                /*useUsapPool=*/ false,
	                new String[] {PROC_START_SEQ_IDENT + app.startSeq});
	    } else { // 默认是REGULAR_ZYGOTE
	        startResult = Process.start(entryPoint,
	                app.processName, uid, uid, gids, runtimeFlags, mountExternal,
	                app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
	                app.info.dataDir, invokeWith, app.info.packageName,
	                new String[] {PROC_START_SEQ_IDENT + app.startSeq});
	    }
	    return startResult;
	}

## 12.Process
[ -> frameworks/base/core/java/android/os/Process.java ]

### 12.1 start

Process#start各参数含义:  
- processClass: 进程入口类, 对于App进程是"android.app.ActivityThread"
- niceName: 进程名
- uid: 用户id
- gid: group id
- gids: group-ids 用于应用程序间共享资源
- runtimeFlags: 
- mountExternal
- targetSdkVersion
- seInfo: SELinux 信息
- abi: ABI
- instructionSet
- appDataDir: 应用程序的data目录
- invokeWith
- packageName: 包名
- zygoteArgs: zygote参数所需的参数

具体代码如下:  

	public static ProcessStartResult start(@NonNull final String processClass,
	                                       @Nullable final String niceName,
	                                       int uid, int gid, @Nullable int[] gids,
	                                       int runtimeFlags,
	                                       int mountExternal,
	                                       int targetSdkVersion,
	                                       @Nullable String seInfo,
	                                       @NonNull String abi,
	                                       @Nullable String instructionSet,
	                                       @Nullable String appDataDir,
	                                       @Nullable String invokeWith,
	                                       @Nullable String packageName,
	                                       @Nullable String[] zygoteArgs) {
	    return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, packageName,
                /*useUsapPool=*/ true, zygoteArgs);
	}

## 13.ZygoteProcess
[ -> frameworks/base/core/java/android/os/ZygoteProcess.java ]

### 13.1 start

	public final Process.ProcessStartResult start(@NonNull final String processClass,
	                                              final String niceName,
	                                              int uid, int gid, @Nullable int[] gids,
	                                              int runtimeFlags, int mountExternal,
	                                              int targetSdkVersion,
	                                              @Nullable String seInfo,
	                                              @NonNull String abi,
	                                              @Nullable String instructionSet,
	                                              @Nullable String appDataDir,
	                                              @Nullable String invokeWith,
	                                              @Nullable String packageName,
	                                              boolean useUsapPool,
	                                              @Nullable String[] zygoteArgs) {
	    // 通过Zygote启动新进程
	    return startViaZygote(processClass, niceName, uid, gid, gids,
	            runtimeFlags, mountExternal, targetSdkVersion, seInfo,
	            abi, instructionSet, appDataDir, invokeWith, /*startChildZygote=*/ false,
	            packageName, useUsapPool, zygoteArgs);
	}

### 13.2 startViaZygote

	private Process.ProcessStartResult startViaZygote(@NonNull final String processClass,
	                                                  @Nullable final String niceName,
	                                                  final int uid, final int gid,
	                                                  @Nullable final int[] gids,
	                                                  int runtimeFlags, int mountExternal,
	                                                  int targetSdkVersion,
	                                                  @Nullable String seInfo,
	                                                  @NonNull String abi,
	                                                  @Nullable String instructionSet,
	                                                  @Nullable String appDataDir,
	                                                  @Nullable String invokeWith,
	                                                  boolean startChildZygote,
	                                                  @Nullable String packageName,
	                                                  boolean useUsapPool,
	                                                  @Nullable String[] extraArgs)
	                                                  throws ZygoteStartFailedEx {
	    // Zygote进程参数设置argsForZygote
	    ArrayList<String> argsForZygote = new ArrayList<>();
	    argsForZygote.add("--runtime-args");
	    argsForZygote.add("--setuid=" + uid);
	    argsForZygote.add("--setgid=" + gid);
	    argsForZygote.add("--runtime-flags=" + runtimeFlags);
	    if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
	        argsForZygote.add("--mount-external-default");
	    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
	        argsForZygote.add("--mount-external-read");
	    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
	        argsForZygote.add("--mount-external-write");
	    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_FULL) {
	        argsForZygote.add("--mount-external-full");
	    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_INSTALLER) {
	        argsForZygote.add("--mount-external-installer");
	    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_LEGACY) {
	        argsForZygote.add("--mount-external-legacy");
	    }
	    argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
 
	    if (gids != null && gids.length > 0) {
	        StringBuilder sb = new StringBuilder();
	        sb.append("--setgroups=");
	        int sz = gids.length;
	        for (int i = 0; i < sz; i++) {
	            if (i != 0) {
	                sb.append(',');
	            }
	            sb.append(gids[i]);
	        }
	        argsForZygote.add(sb.toString());
	    }
 
	    argsForZygote.add("--nice-name=" + niceName);
	    argsForZygote.add("--seinfo=" + seInfo);
	    argsForZygote.add("--instruction-set=" + instructionSet);
	    argsForZygote.add("--app-data-dir=" + appDataDir);
	    argsForZygote.add("--invoke-with");
	    argsForZygote.add(invokeWith);
	    argsForZygote.add("--start-child-zygote");
	    argsForZygote.add("--package-name=" + packageName);
	    argsForZygote.add(processClass);
	    Collections.addAll(argsForZygote, extraArgs);
 
	    synchronized(mLock) {
	        // 先和Zygote进程通过socket通道建立连接, 然后将参数argsForZygote写入Zygote进程, 并等待结果返回
	        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), useUsapPool, argsForZygote);
	    }
	}

### 13.3 openZygoteSocketIfNeeded

	// 主Zygote进程zygote和从Zygote进程zygote_secondary. 优先和主Zygote进程建立连接.
	private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
	    attemptConnectionToPrimaryZygote();
 
	    if (primaryZygoteState.matches(abi)) {
	        return primaryZygoteState;
	    }
 
	    if (mZygoteSecondarySocketAddress != null) {
	        attemptConnectionToSecondaryZygote();
 
	        if (secondaryZygoteState.matches(abi)) {
	            return secondaryZygoteState;
	        }
	    }
	}

### 13.4 attemptZygoteSendArgsAndGetResult

	private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(
	        ZygoteState zygoteState, String msgStr) throws ZygoteStartFailedEx {
	    final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
	    final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
 
	    // 向Zygote进程输出流中写入参数信息
	    zygoteWriter.write(msgStr);
	    zygoteWriter.flush();
 
	    // 从Zygote进程输入流读取进程创建信息
	    Process.ProcessStartResult result = new Process.ProcessStartResult();
	    result.pid = zygoteInputStream.readInt();
	    result.usingWrapper = zygoteInputStream.readBoolean();
 
	    return result;
	}

## 14. App进程创建

参考文章 **[App进程创建过程](http://mouxuejie.com/blog/2020-03-08/zygote-fork-app-proc/)**

## 15.ActivityStackSupervisor
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java ]

### 15.1 realStartActivityLocked

	boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
	        boolean andResume, boolean checkConfig) throws RemoteException {
	    // 先等所有的暂停Activity操作完成
	    if (!mRootActivityContainer.allPausedActivitiesComplete()) {
	        return false;
	    }
	    ...
	    final TaskRecord task = r.getTaskRecord();
	    final ActivityStack stack = task.getStack();
 
	    proc.addActivityIfNeeded(r);
	    List<ResultInfo> results = null;
	    List<ReferrerIntent> newIntents = null;
	    if (andResume) {
	        // We don't need to deliver new intents and/or set results if activity is going to pause immediately after launch.
	        results = r.results;
	        newIntents = r.newIntents;
	    }
 
	    if (r.isActivityTypeHome()) { // 如果Activity为Home或Launcher
	        // 将task的第一个Activity所属进程设置为Home进程
	        updateHomeProcess(task.mActivities.get(0).app);
	    }
	    r.sleeping = false;
	    r.forceNewConfig = false;
	    r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
 
	    // Create activity launch transaction.
	    final ClientTransaction clientTransaction = ClientTransaction.obtain(proc.getThread(), r.appToken);
	    // 添加LaunchActivityItem消息到callback队列
	    final DisplayContent dc = r.getDisplay().mDisplayContent;
	    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
	            System.identityHashCode(r), r.info,
	            mergedConfiguration.getGlobalConfiguration(),
	            mergedConfiguration.getOverrideConfiguration(), r.compat,
	            r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
	            r.icicle, r.persistentState, results, newIntents,
	            dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(), r.assistToken));
 
	    // 设置执行完transaction操作后的最终生命周期状态, 可能是ResumeActivityItem或PauseActivityItem
	    final ActivityLifecycleItem lifecycleItem;
	    if (andResume) {
	        lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
	    } else {
	        lifecycleItem = PauseActivityItem.obtain();
	    }
	    clientTransaction.setLifecycleStateRequest(lifecycleItem);
 
	    // 执行transaction操作. 见【第16节】
	    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
	    ...
	    if (andResume && readyToResume()) {
	        // As part of the process of launching, ActivityThread also performs a resume.
	        stack.minimalResumeActivityLocked(r);
	    } else {
	        // This activity is not starting in the resumed state... which should look like we asked
	        // it to pause+stop (but remain visible), and it has done so and reported back the
	        // current icicle and other state.
	        r.setState(PAUSED, "realStartActivityLocked");
	    }
	    // Launch the new version setup screen if needed.  We do this -after-
	    // launching the initial activity (that is, home), so that it can have
	    // a chance to initialize itself while in the background, making the
	    // switch back to it faster and look better.
	    if (mRootActivityContainer.isTopDisplayFocusedStack(stack)) {
	        mService.getActivityStartController().startSetupActivity();
	    }
 
	    // Update any services we are bound to that might care about whether
	    // their client may have activities.
	    if (r.app != null) {
	        r.app.updateServiceConnectionActivities();
	    }
	    return true;
	}

## 16.ClientLifecycleManager
[ -> frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java ]

### 16.1 scheduleTransaction

	void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
	    final IApplicationThread client = transaction.getClient();
	    transaction.schedule();
	    if (!(client instanceof Binder)) {
	        // If client is not an instance of Binder - it's a remote call and at this point it is
	        // safe to recycle the object. All objects used for local calls will be recycled after
	        // the transaction is executed on client in ActivityThread.
	        transaction.recycle();
	    }
	}

## 17.ClientTransaction
[ -> frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java ]

### 17.1 schedule

	public void schedule() throws RemoteException {
	    mClient.scheduleTransaction(this);
	}

## 18.ApplicationThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 18.1 scheduleTransaction

	private class ApplicationThread extends IApplicationThread.Stub {
	    @Override
	    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
	        ActivityThread.this.scheduleTransaction(transaction);
	    }
	}

## 19.ClientTransactionHandler
[ -> frameworks/base/core/java/android/app/ClientTransactionHandler.java ]

### 19.1 scheduleTransaction

	/** Prepare and schedule transaction for execution. */
	void scheduleTransaction(ClientTransaction transaction) {
	    // 预处理
	    transaction.preExecute(this);
	    // 执行操作
	    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
	}

### 19.2 H.EXECUTE_TRANSACTION
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

	class H extends Handler {
	    public static final int EXECUTE_TRANSACTION = 159;
     
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case EXECUTE_TRANSACTION:
	                final ClientTransaction transaction = (ClientTransaction) msg.obj;
	                mTransactionExecutor.execute(transaction);
	                if (isSystem()) {
	                    // Client transactions inside system process are recycled on the client side
	                    // instead of ClientLifecycleManager to avoid being cleared before this
	                    // message is handled.
	                    transaction.recycle();
	                }
	                break;
	        }
	    }
	}

## 20.TransactionExecutor
[ -> frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java ]

### 20.1 execute

	public void execute(ClientTransaction transaction) {
	    // 检查当前事务的Activity是不是要destroy, 如果是则将其从activitiesToBeDestroyed列表移除
	    // 因为后面即将执行这个Activity的destroy操作.
	    final IBinder token = transaction.getActivityToken();
	    if (token != null) {
	        final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed = mTransactionHandler.getActivitiesToBeDestroyed();
	        final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
	        if (destroyItem != null) {
	            if (transaction.getLifecycleStateRequest() == destroyItem) {
	                // It is going to execute the transaction that will destroy activity with the
	                // token, so the corresponding to-be-destroyed record can be removed.
	                activitiesToBeDestroyed.remove(token);
	            }
	            if (mTransactionHandler.getActivityClient(token) == null) {
	                // The activity has not been created but has been requested to destroy, so all
	                // transactions for the token are just like being cancelled.
	                return;
	            }
	        }
	    }
     
	    // 执行callbacks列表中所有状态(startActivity过程执行ON_CREATE)
	    executeCallbacks(transaction);
	    // 执行到最终目标状态(startActivity过程执行ON_START和ON_RESUME)
	    executeLifecycleState(transaction);
	}

`executeCallbacks` 执行handleLaunchActivity (即LaunchActivityItem)   
`executeLifecycleState` 执行handleStartActivity -> handleResumeActivity

生命周期的遍历过程参考【第27节】**ActivityLifecycleItem**

### 20.2 executeCallbacks

	// 遍历并执行callbacks请求的所有状态
	public void executeCallbacks(ClientTransaction transaction) {
	    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
	    if (callbacks == null || callbacks.isEmpty()) {
	        return;
	    }
 
	    // 获取Activity令牌
	    final IBinder token = transaction.getActivityToken();
	    // 获取ActivityClientRecord
	    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
	    // 获取最终的目标状态
	    final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
	    final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState() : UNDEFINED;
	    // Index of the last callback that requests some post-execution state.
	    final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);
 
	    final int size = callbacks.size();
	    for (int i = 0; i < size; ++i) {
	        // 获取当前状态
	        final ClientTransactionItem item = callbacks.get(i);
	        // 获取下一个状态（只有NewIntentItem的getPostExecutionState为ON_RESUME, 其他都是UNDEFINED）
	        final int postExecutionState = item.getPostExecutionState();
	        // 获取离下一个状态最短路线的上一个状态（基本都是UNDEFINED，ON_RESUME除外）
	        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r, item.getPostExecutionState());
	        // 若最短路线的上一个状态非空，则执行从当前状态到最短路线的上一个状态中间的所有状态
	        // 只有当前为NewIntentItem时才会走到这里，其他情况都走不到
	        if (closestPreExecutionState != UNDEFINED) {
	            cycleToPath(r, closestPreExecutionState, transaction);
	        }
 
	        // 执行当前状态
	        item.execute(mTransactionHandler, token, mPendingActions);
	        // 当前状态postExecute
	        item.postExecute(mTransactionHandler, token, mPendingActions);
         
	        if (r == null) {
	            // Launch activity request will create an activity record.
	            r = mTransactionHandler.getActivityClient(token);
	        }
	        if (postExecutionState != UNDEFINED && r != null) {
	            // Skip the very last transition and perform it by explicit state request instead.
	            final boolean shouldExcludeLastTransition = i == lastCallbackRequestingState && finalState == postExecutionState;
	            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
	        }
	    }
	}

#### 20.2.1 LaunchActivityItem

[ -> frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java ]

	@Override
	public void execute(ClientTransactionHandler client, IBinder token, PendingTransactionActions pendingActions) {
	    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
	            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, 
	            mPersistentState, mPendingResults, mPendingNewIntents, mIsForward,
	            mProfilerInfo, client, mAssistToken);
	    // 执行launch操作. 见【第21.1节】
	    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
	}
 
	@Override
	public void postExecute(ClientTransactionHandler client, IBinder token, PendingTransactionActions pendingActions) {
	    // launching Activity减1
	    client.countLaunchingActivities(-1);
	}

### 20.3 executeLifecycleState

	/** Transition to the final state if requested by the transaction. */
	private void executeLifecycleState(ClientTransaction transaction) {
	    // 获取到最终状态
	    final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
	    if (lifecycleItem == null) {
	        return;
	    }
 
	    final IBinder token = transaction.getActivityToken();
	    final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
 
	    if (r == null) {
	        // Ignore requests for non-existent client records for now.
	        return;
	    }
 
	    // Cycle to the state right before the final requested state.
	    // 执行从当前状态到最终状态前一个状态之间的所有状态
	    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);
 
	    // Execute the final transition with proper parameters.
	    // 执行最终状态
	    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
	    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
	}

#### 20.3.1 cycleToPath

	private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState, ClientTransaction transaction) {
	    // 获取起始状态
	    final int start = r.getLifecycleState();
	    // 获取start~end之间的所有状态
	    final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
	    // 执行所有状态
	    performLifecycleSequence(r, path, transaction);
	}

#### 20.3.2 TransactionExecutorHelper.getLifecyclePath
[ -> frameworks/base/core/java/android/app/servertransaction/TransactionExecutorHelper.java ]

	public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
	    mLifecycleSequence.clear();
	    if (finish >= start) {
	        // just go there
	        for (int i = start + 1; i <= finish; i++) {
	            mLifecycleSequence.add(i);
	        }
	    } else { // finish < start, can't just cycle down
	        if (start == ON_PAUSE && finish == ON_RESUME) {
	            // Special case when we can just directly go to resumed state.
	            mLifecycleSequence.add(ON_RESUME);
	        } else if (start <= ON_STOP && finish >= ON_START) {
	            // Restart and go to required state.
	            // Go to stopped state first.
	            for (int i = start + 1; i <= ON_STOP; i++) {
	                mLifecycleSequence.add(i);
	            }
	            // Restart
	            mLifecycleSequence.add(ON_RESTART);
	            // Go to required state
	            for (int i = ON_START; i <= finish; i++) {
	                mLifecycleSequence.add(i);
	            }
	        } else { // Relaunch and go to required state
	            // Go to destroyed state first.
	            for (int i = start + 1; i <= ON_DESTROY; i++) {
	                mLifecycleSequence.add(i);
	            }
	            // Go to required state
	            for (int i = ON_CREATE; i <= finish; i++) {
	                mLifecycleSequence.add(i);
	            }
	        }
	    }
 
	    // Remove last transition in case we want to perform it with some specific params.
	    if (excludeLastState && mLifecycleSequence.size() != 0) {
	        mLifecycleSequence.remove(mLifecycleSequence.size() - 1);
	    }
 
	    return mLifecycleSequence;
	}

#### 20.3.3 performLifecycleSequence

执行`ON_START`和`ON_RESUME`

	private void performLifecycleSequence(ActivityClientRecord r, IntArray path, ClientTransaction transaction) {
	    final int size = path.size();
	    for (int i = 0, state; i < size; i++) {
	        state = path.get(i);
	        switch (state) {
	            case ON_CREATE:
	                mTransactionHandler.handleLaunchActivity(r, mPendingActions, null);
	                break;
	            case ON_START:
		            // 见【第21.2节】
	                mTransactionHandler.handleStartActivity(r, mPendingActions);
	                break;
	            case ON_RESUME:
	                // 见【第21.3节】
	                mTransactionHandler.handleResumeActivity(r.token, false, r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
	                break;
	            case ON_PAUSE:
	                mTransactionHandler.handlePauseActivity(r.token, false,
                        false, 0, mPendingActions, "LIFECYCLER_PAUSE_ACTIVITY");
	                break;
	            case ON_STOP:
	                mTransactionHandler.handleStopActivity(r.token, false,
                        0, mPendingActions, false, "LIFECYCLER_STOP_ACTIVITY");
	                break;
	            case ON_DESTROY:
	                mTransactionHandler.handleDestroyActivity(r.token, false,
                        0, false, "performLifecycleSequence. cycling to:" + path.get(size - 1));
	                break;
	            case ON_RESTART:
	                mTransactionHandler.performRestartActivity(r.token, false /* start */);
	                break;
	            default:
	                throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
	        }
	    }
	}

此处 start = 1, end = 3. 因此接下来会执行ON_START和ON_RESUME.

## 21.ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 21.1 handleLaunchActivity

	@Override
	public Activity handleLaunchActivity(ActivityClientRecord r, PendingTransactionActions pendingActions, Intent customIntent) {
	    ...
	    final Activity a = performLaunchActivity(r, customIntent);
 
	    if (a != null) {
	        // 更新Configuration
	        r.createdConfig = new Configuration(mConfiguration);
	        reportSizeConfigurations(r);
	        // 更新pendingActions
	        if (!r.activity.mFinished && pendingActions != null) {
	            pendingActions.setOldState(r.state);
	            pendingActions.setRestoreInstanceState(true);
	            pendingActions.setCallOnPostCreate(true);
	        }
	    } else { // 错误处理
	        ActivityTaskManager.getService().finishActivity(r.token, Activity.RESULT_CANCELED, null, Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
	    }
 
	    return a;
	}

#### 21.1.1 performLaunchActivity

	/**  Core implementation of activity launch. */
	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
	    ActivityInfo aInfo = r.activityInfo;
	    // 设置packageInfo
	    if (r.packageInfo == null) {
	        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo, Context.CONTEXT_INCLUDE_CODE);
	    }
 
	    // 设置component
	    ComponentName component = r.intent.getComponent();
	    if (component == null) {
	        component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
	        r.intent.setComponent(component);
	    }
 
	    if (r.activityInfo.targetActivity != null) {
	        component = new ComponentName(r.activityInfo.packageName, r.activityInfo.targetActivity);
	    }
 
	    // 创建ContextImpl
	    ContextImpl appContext = createBaseContextForActivity(r);
	    Activity activity = null;
	    try {
	        // 反射创建Activity
	        java.lang.ClassLoader cl = appContext.getClassLoader();
	        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
	        StrictMode.incrementExpectedActivityCount(activity.getClass());
	        r.intent.setExtrasClassLoader(cl);
	        r.intent.prepareToEnterProcess();
	        if (r.state != null) {
	            r.state.setClassLoader(cl);
	        }
	    } catch (Exception e) {
	        ...
	    }
 
	    try {
	        // 创建并启动Application实例， 如果mApplcation非空则直接返回
	        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
 
	        // 设置Activity各属性
	        if (activity != null) {
	            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
	            Configuration config = new Configuration(mCompatConfiguration);
	            if (r.overrideConfig != null) {
	                config.updateFrom(r.overrideConfig);
	            }
	            Window window = null;
	            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
	                window = r.mPendingRemoveWindow;
	                r.mPendingRemoveWindow = null;
	                r.mPendingRemoveWindowManager = null;
	            }
	            appContext.setOuterContext(activity);
	            activity.attach(appContext, this, getInstrumentation(), r.token,
	                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
	                    r.embeddedID, r.lastNonConfigurationInstances, config,
	                    r.referrer, r.voiceInteractor, window, r.configCallback,
	                    r.assistToken);
 
	            if (customIntent != null) {
	                activity.mIntent = customIntent;
	            }
	            r.lastNonConfigurationInstances = null;
	            checkAndBlockForNetworkAccess();
	            activity.mStartedActivity = false;
	            int theme = r.activityInfo.getThemeResource();
	            if (theme != 0) {
	                activity.setTheme(theme);
	            }
 
	            activity.mCalled = false;
	            // 调用Activity.onCreate(...)方法
	            if (r.isPersistable()) {
	                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
	            } else {
	                mInstrumentation.callActivityOnCreate(activity, r.state);
	            }
	            r.activity = activity;
	        }
	        // 设置Activity状态为ON_CREATE
	        r.setState(ON_CREATE);
 
	        synchronized (mResourcesManager) {
	            mActivities.put(r.token, r);
	        }
	    } catch (SuperNotCalledException e) {
	        ...
	    }
 
	    return activity;
	}

#### 21.1.2 createBaseContextForActivity

	// 创建Activity的上下文
	private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
	    final int displayId;
        displayId = ActivityTaskManager.getService().getActivityDisplayId(r.token);
 
	    ContextImpl appContext = ContextImpl.createActivityContext(this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
 
	    final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
	    // For debugging purposes, if the activity's package name contains the value of
	    // the "debug.use-second-display" system property as a substring, then show
	    // its content on a secondary display if there is one.
	    String pkgName = SystemProperties.get("debug.second-display.pkg");
	    if (pkgName != null && !pkgName.isEmpty()&&r.packageInfo.mPackageName.contains(pkgName)) {
	        for (int id : dm.getDisplayIds()) {
	            if (id != Display.DEFAULT_DISPLAY) {
	                Display display = dm.getCompatibleDisplay(id, appContext.getResources());
	                appContext = (ContextImpl) appContext.createDisplayContext(display);
	                break;
	            }
	        }
	    }
	    return appContext;
	}

#### 21.1.3 ContextImpl.createActivityContext

	static ContextImpl createActivityContext(ActivityThread mainThread,
	        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, 
	        int displayId, Configuration overrideConfiguration) {
	    String[] splitDirs = packageInfo.getSplitResDirs();
	    ClassLoader classLoader = packageInfo.getClassLoader();
	    if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
	        classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
	        splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
	    }
 
	    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName, activityToken, null, 0, classLoader, null);
 
	    // Clamp display ID to DEFAULT_DISPLAY if it is INVALID_DISPLAY.
	    displayId = (displayId != Display.INVALID_DISPLAY) ? displayId : Display.DEFAULT_DISPLAY;
	    final CompatibilityInfo compatInfo = (displayId == Display.DEFAULT_DISPLAY)
	            ? packageInfo.getCompatibilityInfo()
	            : CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO;
 
	    final ResourcesManager resourcesManager = ResourcesManager.getInstance();
 
	    // Create the base resources for which all configuration contexts for this Activity
	    // will be rebased upon.
	    context.setResources(resourcesManager.createBaseActivityResources(activityToken,
	            packageInfo.getResDir(),
	            splitDirs,
	            packageInfo.getOverlayDirs(),
	            packageInfo.getApplicationInfo().sharedLibraryFiles,
	            displayId,
	            overrideConfiguration,
	            compatInfo,
	            classLoader));
	    context.mDisplay = resourcesManager.getAdjustedDisplay(displayId, context.getResources());
	    return context;
	}

### 21.2 handleStartActivity

	@Override
	public void handleStartActivity(ActivityClientRecord r, PendingTransactionActions pendingActions) {
	    final Activity activity = r.activity;
	    // 执行Start操作
	    activity.performStart("handleStartActivity");
	    // 设置Activity状态为ON_START
	    r.setState(ON_START);
 
	    if (pendingActions == null) {
	        // No more work to do.
	        return;
	    }
	    // Restore instance state
	    // 调用RestoreInstanceState恢复状态
	    if (pendingActions.shouldRestoreInstanceState()) {
	        if (r.isPersistable()) {
	            if (r.state != null || r.persistentState != null) {
	                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state, r.persistentState);
	            }
	        } else if (r.state != null) {
	            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
	        }
	    }
 
	    // Call postOnCreate()
	    // 调用postOnCreate
	    if (pendingActions.shouldCallOnPostCreate()) {
	        activity.mCalled = false;
	        if (r.isPersistable()) {
	            mInstrumentation.callActivityOnPostCreate(activity, r.state, r.persistentState);
	        } else {
	            mInstrumentation.callActivityOnPostCreate(activity, r.state);
	        }
	    }
	}

#### 21.2.1 Activity.performStart

见【第23.2节】

###	21.3 handleResumeActivity

	@Override
	public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
	    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
	     
	    if (mActivitiesToBeDestroyed.containsKey(token)) {
	        return;
	    }
	 
	    final Activity a = r.activity;
	 
	    final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
	 
	    // 判断是否willActivityBeVisible
	    boolean willBeVisible = !a.mStartedActivity;
	    if (!willBeVisible) {
	        willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(a.getActivityToken());
	    }
	    // 如果Window还没添加到WindowManager，且要求Activity将要可见，则将Window添加到WindowManager
	    if (r.window == null && !a.mFinished && willBeVisible) {
	        r.window = r.activity.getWindow();
	        View decor = r.window.getDecorView();
	        decor.setVisibility(View.INVISIBLE);
	        ViewManager wm = a.getWindowManager();
	        WindowManager.LayoutParams l = r.window.getAttributes();
	        a.mDecor = decor;
	        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
	        l.softInputMode |= forwardBit;
	        if (r.mPreserveWindow) {
	            a.mWindowAdded = true;
	            r.mPreserveWindow = false;
	            ViewRootImpl impl = decor.getViewRootImpl();
	            if (impl != null) {
	                impl.notifyChildRebuilt();
	            }
	        }
	        if (a.mVisibleFromClient) {
	            if (!a.mWindowAdded) {
	                a.mWindowAdded = true;
	                wm.addView(decor, l);
	            } else {
	                a.onWindowAttributesChanged(l);
	            }
	        }
	    } else if (!willBeVisible) {
	        // 如果Window已经添加到WindowManager，但要求Activity将不可见，则设置隐藏
	        r.hideForNow = true;
	    }
	 
	    // Get rid of anything left hanging around.
	    // 清空将要移除的Windows
	    cleanUpPendingRemoveWindows(r, false /* force */);
	 
	    // The window is now visible if it has been added, we are not
	    // simply finishing, and we are not starting another activity.
	    // 设置Activity可见
	    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
	        if (r.newConfig != null) {
	            performConfigurationChangedForActivity(r, r.newConfig);
	            r.newConfig = null;
	        }
	        WindowManager.LayoutParams l = r.window.getAttributes();
	        if ((l.softInputMode & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) != forwardBit) {
	            l.softInputMode = (l.softInputMode & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)) | forwardBit;
	            if (r.activity.mVisibleFromClient) {
	                ViewManager wm = a.getWindowManager();
	                View decor = r.window.getDecorView();
	                wm.updateViewLayout(decor, l);
	            }
	        }
	 
	        r.activity.mVisibleFromServer = true;
	        mNumVisibleActivities++;
	        if (r.activity.mVisibleFromClient) {
	            r.activity.makeVisible();
	        }
	    }
	 
	    r.nextIdle = mNewActivities;
	    mNewActivities = r;
	    Looper.myQueue().addIdleHandler(new Idler());
	}

#### 21.3.1 performResumeActivity

	public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest, String reason) {
	    // 获取ActivityClientRecord
	    final ActivityClientRecord r = mActivities.get(token);
	    if (r == null || r.activity.mFinished) {
	        return null;
	    }
	 
	    if (finalStateRequest) {
	        r.hideForNow = false;
	        r.activity.mStartedActivity = false;
	    }
	    ...
	    // 调用Activity.onNewIntent
	    if (r.pendingIntents != null) {
	        deliverNewIntents(r, r.pendingIntents);
	        r.pendingIntents = null;
	    }
	    // 调用Activity.onActivityResult
	    if (r.pendingResults != null) {
	        deliverResults(r, r.pendingResults, reason);
	        r.pendingResults = null;
	    }
	    // 调用Activity.onResume
	    r.activity.performResume(r.startsNotResumed, reason);
	 
	    r.state = null;
	    r.persistentState = null;
	    // 设置Activity的状态为ON_RESUME
	    r.setState(ON_RESUME);
	 
	    return r;
	}

#### 21.3.2 deliverNewIntents

	private void deliverNewIntents(ActivityClientRecord r, List<ReferrerIntent> intents) {
	    final int N = intents.size();
	    for (int i=0; i<N; i++) {
	        ReferrerIntent intent = intents.get(i);
	        intent.setExtrasClassLoader(r.activity.getClassLoader());
	        intent.prepareToEnterProcess();
	        r.activity.mFragments.noteStateNotSaved();
	        mInstrumentation.callActivityOnNewIntent(r.activity, intent);
	    }
	}

#### 21.3.3 deliverResults

	private void deliverResults(ActivityClientRecord r, List<ResultInfo> results, String reason) {
	    final int N = results.size();
	    for (int i=0; i<N; i++) {
	        ResultInfo ri = results.get(i);
	        if (ri.mData != null) {
	            ri.mData.setExtrasClassLoader(r.activity.getClassLoader());
	            ri.mData.prepareToEnterProcess();
	        }
	        r.activity.dispatchActivityResult(ri.mResultWho, ri.mRequestCode, ri.mResultCode, ri.mData, reason);
	    }
	}

#### 21.3.4 Activity.performResume

见【第23.3节】

## 22.Instrumentation
[ -> frameworks/base/core/java/android/app/Instrumentation.java ]

### 22.1 newActivity

	public Activity newActivity(ClassLoader cl, String className, Intent intent)
	        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
	    String pkg = intent != null && intent.getComponent() != null ? intent.getComponent().getPackageName() : null;
	    return getFactory(pkg).instantiateActivity(cl, className, intent);
	}

	private AppComponentFactory getFactory(String pkg) {
	    if (pkg == null) {
	        return AppComponentFactory.DEFAULT;
	    }
	    if (mThread == null) {
	        return AppComponentFactory.DEFAULT;
	    }
	    LoadedApk apk = mThread.peekPackageInfo(pkg, true);
	    if (apk == null) apk = mThread.getSystemContext().mPackageInfo;
	    return apk.getAppFactory();
	}

#### 22.1.1 AppComponentFactory.instantiateActivity
[ -> frameworks/base/core/java/android/app/AppComponentFactory.java ]

	public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String className, @Nullable Intent intent)
	        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
	    return (Activity) cl.loadClass(className).newInstance();
	}

### 22.2 callActivityOnCreate

	public void callActivityOnCreate(Activity activity, Bundle icicle) {
	    prePerformCreate(activity);
	    activity.performCreate(icicle);
	    postPerformCreate(activity);
	}

#### 22.2.1 prePerformCreate

	// 更新mWaitingActivities列表状态
	private void prePerformCreate(Activity activity) {
	    if (mWaitingActivities != null) {
	        synchronized (mSync) {
	            final int N = mWaitingActivities.size();
	            for (int i=0; i<N; i++) {
	                final ActivityWaiter aw = mWaitingActivities.get(i);
	                final Intent intent = aw.intent;
	                if (intent.filterEquals(activity.getIntent())) {
	                    aw.activity = activity;
	                    mMessageQueue.addIdleHandler(new ActivityGoing(aw));
	                }
	            }
	        }
	    }
	}

#### 22.2.2 Activity.performCreate

见【第23.1节】

#### 22.2.3 postPerformCreate

	// 遍历mActivityMonitors， 并通知与Activity匹配的ActivityMonitor
	private void postPerformCreate(Activity activity) {
	    if (mActivityMonitors != null) {
	        synchronized (mSync) {
	            final int N = mActivityMonitors.size();
	            for (int i=0; i<N; i++) {
	                final ActivityMonitor am = mActivityMonitors.get(i);
	                am.match(activity, activity, activity.getIntent());
	            }
	        }
	    }
	}

### 22.3 callActivityOnStart

	public void callActivityOnStart(Activity activity) {
	    activity.onStart();
	}

#### 22.3.1 Activity.onStart 

见【第23.2节】

### 22.4 callActivityOnResume

	public void callActivityOnResume(Activity activity) {
	    activity.mResumed = true;
	    activity.onResume();
	 
	    if (mActivityMonitors != null) {
	        synchronized (mSync) {
	            final int N = mActivityMonitors.size();
	            for (int i=0; i<N; i++) {
	                final ActivityMonitor am = mActivityMonitors.get(i);
	                am.match(activity, activity, activity.getIntent());
	            }
	        }
	    }
	}

#### 22.4.1 Activity.onResume

见【第23.3节】

## 23.Activity
[ -> frameworks/base/core/java/android/app/Activity.java ]

### 23.1 performCreate

	final void performCreate(Bundle icicle) {
	    performCreate(icicle, null);
	}
 
	final void performCreate(Bundle icicle, PersistableBundle persistentState) {
	    // 调用Application生命周期ActivityLifecycleCallbacks方法onActivityPreCreated
	    dispatchActivityPreCreated(icicle);
	    ...
	    if (persistentState != null) {
	        onCreate(icicle, persistentState);
	    } else {
	        onCreate(icicle);
	    }
	    ...
	    // 告诉所有子fragments, 父Activity已经创建
	    mFragments.dispatchActivityCreated();
	    // 调用Application生命周期ActivityLifecycleCallbacks方法onActivityPostCreated
	    dispatchActivityPostCreated(icicle);
	}

#### 23.1.1 dispatchActivityPreCreated

	private void dispatchActivityPreCreated(@Nullable Bundle savedInstanceState) {
	    getApplication().dispatchActivityPreCreated(this, savedInstanceState);
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = 0; i < callbacks.length; i++) {
	            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPreCreated(this, savedInstanceState);
	        }
	    }
	}

#### 23.1.2 onCreate

	protected void onCreate(@Nullable Bundle savedInstanceState) {
	    ...
	    if (savedInstanceState != null) {
	        ...
	        Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
	        // 恢复子fragments的状态
	        mFragments.restoreAllState(p, mLastNonConfigurationInstances != null ? mLastNonConfigurationInstances.fragments : null);
	    }
	    // 通知子fragments执行Create方法
	    mFragments.dispatchCreate();
	    // 执行Application生命周期回调ActivityLifecycleCallbacks.onActivityCreated
	    dispatchActivityCreated(savedInstanceState);
	    ...
	    mRestoredFromBundle = savedInstanceState != null;
	    mCalled = true;
	}

#### 23.1.3 FramentController.dispatchCreate

见【第24节】

#### 23.1.4 dispatchActivityCreated

    private void dispatchActivityCreated(@Nullable Bundle savedInstanceState) {
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityCreated(this,
                        savedInstanceState);
            }
        }
    }

#### 23.1.5 FragmentController.dispatchActivityCreated

见【第24节】

#### 23.1.6 dispatchActivityPostCreated

    private void dispatchActivityPostCreated(@Nullable Bundle savedInstanceState) {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostCreated(this,
                        savedInstanceState);
            }
        }
        getApplication().dispatchActivityPostCreated(this, savedInstanceState);
    }

### 23.2 performStart

	final void performStart(String reason) {
	    // Application生命周期函数回调ActivityLifecycleCallbacks.onActivityPreStarted
	    dispatchActivityPreStarted();
	    ...
	    mInstrumentation.callActivityOnStart(this);
	    // 通知子fragments, 父Activity已经start
	    mFragments.dispatchStart();
	    ...
	    // Application生命周期函数回调ActivityLifecycleCallbacks.onActivityPostStarted
	    dispatchActivityPostStarted();
	}

#### 23.2.1 dispatchActivityPreStarted

	private void dispatchActivityPreStarted() {
	    getApplication().dispatchActivityPreStarted(this);
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = 0; i < callbacks.length; i++) {
	            ((Application.ActivityLifecycleCallbacks)callbacks[i]).onActivityPreStarted(this);
	        }
	    }
	}

#### 23.2.2 Instrumentation.callActivityOnStart

见【第22.3节】

#### 23.2.3 onStart

    protected void onStart() {
        mFragments.doLoaderStart();
        dispatchActivityStarted();
    }

#### 23.2.4 dispatchActivityStarted

	private void dispatchActivityStarted() {
	    getApplication().dispatchActivityStarted(this);
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = 0; i < callbacks.length; i++) {
	            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityStarted(this);
	        }
	    }
	}

#### 23.2.5 FramentController.dispatchStart

见【第24节】

#### 23.2.6 dispatchActivityPostStarted

	private void dispatchActivityPostStarted() {
	    Object[] callbacks = collectActivityLifecycleCallbacks();
	    if (callbacks != null) {
	        for (int i = 0; i < callbacks.length; i++) {
	            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostStarted(this);
	        }
	    }
	    getApplication().dispatchActivityPostStarted(this);
	}

### 23.3 performResume

	final void performResume(boolean followedByPause, String reason) {
	    // Application生命周期函数回调ActivityLifecycleCallbacks.onActivityPreResumed
	    dispatchActivityPreResumed();
	    ...
	    // 调用Activity.onResume
	    mInstrumentation.callActivityOnResume(this);
	    // 通知子fragments, 父Activity已经resume
	    mFragments.dispatchResume();
	    ...
	    onPostResume();
	    // Application生命周期函数回调ActivityLifecycleCallbacks.onActivityPostResumed
	    dispatchActivityPostResumed();
	}

#### 23.3.1 dispatchActivityPreResumed

    private void dispatchActivityPreResumed() {
        getApplication().dispatchActivityPreResumed(this);
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPreResumed(this);
            }
        }
    }

#### 23.3.1 Instrumentation.callActivityOnResume

见【第22.4节】

#### 23.3.3 onResume

    protected void onResume() {
        dispatchActivityResumed();
        ...
        mCalled = true;
    }
    
    private void dispatchActivityResumed() {
        getApplication().dispatchActivityResumed(this);
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityResumed(this);
            }
        }
    }

#### 23.3.4 FramentController.dispatchResume

见【第24节】

#### 23.3.5 onPostResume

    protected void onPostResume() {
        final Window win = getWindow();
        if (win != null) win.makeActive();
        if (mActionBar != null) mActionBar.setShowHideAnimationEnabled(true);
        mCalled = true;
    }

#### 23.3.6 dispatchActivityPostResumed

    private void dispatchActivityPostResumed() {
        Object[] callbacks = collectActivityLifecycleCallbacks();
        if (callbacks != null) {
            for (int i = 0; i < callbacks.length; i++) {
                ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityPostResumed(this);
            }
        }
        getApplication().dispatchActivityPostResumed(this);
    }

## 24.FragmentController
[ -> frameworks/base/core/java/android/app/FragmentController.java ]

### 24.1 dispatchCreate/dispatchStart/dispatchResume

	public void dispatchCreate() {
        mHost.mFragmentManager.dispatchCreate();
    }

    public void dispatchActivityCreated() {
        mHost.mFragmentManager.dispatchActivityCreated();
    }

    public void dispatchStart() {
        mHost.mFragmentManager.dispatchStart();
    }

    public void dispatchResume() {
        mHost.mFragmentManager.dispatchResume();
    }

## 25.FragmentManager
[ -> frameworks/base/core/java/android/app/FragmentManager.java ]

### 25.1 dispatchCreate/dispatchStart/dispatchResume

	public void dispatchCreate() {
        mStateSaved = false;
        dispatchMoveToState(Fragment.CREATED);
    }
    
    public void dispatchActivityCreated() {
        mStateSaved = false;
        dispatchMoveToState(Fragment.ACTIVITY_CREATED);
    }
    
    public void dispatchStart() {
        mStateSaved = false;
        dispatchMoveToState(Fragment.STARTED);
    }
    
    public void dispatchResume() {
        mStateSaved = false;
        dispatchMoveToState(Fragment.RESUMED);
    }

## 26.ActivityLifecycleCallbacks
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

## 27.ActivityLifecycleItem
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

## 28.ActivityThread发送启动结果
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

启动状态分为下面3种.  
ActivityThread.sendActivityResult最终会调到Fragment的onActivityResult.  
App开发过程中，一般都不对startActivity或startActivityForResult的启动结果做处理, 一般只在setResult之后才会处理.

    /** Standard activity result: operation canceled. */
    public static final int RESULT_CANCELED    = 0;
    /** Standard activity result: operation succeeded. */
    public static final int RESULT_OK           = -1;
    /** Start of user-defined activity results. */
    public static final int RESULT_FIRST_USER   = 1;

### 28.1 sendActivityResult

	public final void sendActivityResult(IBinder token, String id, int requestCode, int resultCode, Intent data) {
	    // 封装返回结果
	    ArrayList<ResultInfo> list = new ArrayList<ResultInfo>();
	    list.add(new ResultInfo(id, requestCode, resultCode, data));
	    final ClientTransaction clientTransaction = ClientTransaction.obtain(mAppThread, token);
	    // 添加ActivityResultItem事务
	    clientTransaction.addCallback(ActivityResultItem.obtain(list));
	    try {
	        mAppThread.scheduleTransaction(clientTransaction);
	    } catch (RemoteException e) {
	        // Local scheduling
	    }
	}

### 28.2 ActivityResultItem
[ -> frameworks/base/core/java/android/app/servertransaction/ActivityResultItem.java ]

	public class ActivityResultItem extends ClientTransactionItem {
	    private List<ResultInfo> mResultInfoList;
	 
	    @Override
	    public void execute(ClientTransactionHandler client, IBinder token,
	            PendingTransactionActions pendingActions) {
	        client.handleSendResult(token, mResultInfoList, "ACTIVITY_RESULT");
	    }
	    ...
	}

### 28.3 handleSendResult

    // token为目标Activity的令牌
    public void handleSendResult(IBinder token, List<ResultInfo> results, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            final boolean resumed = !r.paused;
            if (!r.activity.mFinished && r.activity.mDecor != null && r.hideForNow && resumed) {
                // 如果目标Activity为resumed状态，设置目标Activity可见
                updateVisibility(r, true);
            }
            if (resumed) {
                // 设置目标Activity状态为ON_PAUSE
                r.activity.mCalled = false;
                mInstrumentation.callActivityOnPause(r.activity);
            }
            checkAndBlockForNetworkAccess();
            // 发送返回结果
            deliverResults(r, results, reason);
            // 目标Activity执行OnResume操作
            if (resumed) {
                r.activity.performResume(false, reason);
            }
        }
    }

发送启动Activity结果这个过程， 目标Activity一般都是pause状态，因此resumed为false.

### 28.4 deliverResults

    private void deliverResults(ActivityClientRecord r, List<ResultInfo> results, String reason) {
        final int N = results.size();
        for (int i=0; i<N; i++) {
            ResultInfo ri = results.get(i);
            if (ri.mData != null) {
                ri.mData.setExtrasClassLoader(r.activity.getClassLoader());
                ri.mData.prepareToEnterProcess();
            }
            r.activity.dispatchActivityResult(ri.mResultWho, ri.mRequestCode, ri.mResultCode, ri.mData, reason);
        }
    }

### 28.5 Activity.dispatchActivityResult

	 void dispatchActivityResult(String who, int requestCode, int resultCode, Intent data,
            String reason) {
        mFragments.noteStateNotSaved();
        if (who == null) {
            onActivityResult(requestCode, resultCode, data);
        } else if (who.startsWith(REQUEST_PERMISSIONS_WHO_PREFIX)) {
            who = who.substring(REQUEST_PERMISSIONS_WHO_PREFIX.length());
            if (TextUtils.isEmpty(who)) {
                dispatchRequestPermissionsResult(requestCode, data);
            } else {
                Fragment frag = mFragments.findFragmentByWho(who);
                if (frag != null) {
                    dispatchRequestPermissionsResultToFragment(requestCode, data, frag);
                }
            }
        } else if (who.startsWith("@android:view:")) {
            ArrayList<ViewRootImpl> views = WindowManagerGlobal.getInstance().getRootViews(
                    getActivityToken());
            for (ViewRootImpl viewRoot : views) {
                if (viewRoot.getView() != null && viewRoot.getView().dispatchActivityResult(
                         who, requestCode, resultCode, data)) {
                    return;
                }
            }
        } else if (who.startsWith(AUTO_FILL_AUTH_WHO_PREFIX)) {
            Intent resultData = (resultCode == Activity.RESULT_OK) ? data : null;
            getAutofillManager().onAuthenticationResult(requestCode, resultData, getCurrentFocus());
        } else {
            Fragment frag = mFragments.findFragmentByWho(who);
            if (frag != null) {
                frag.onActivityResult(requestCode, resultCode, data);
            }
        }
    }

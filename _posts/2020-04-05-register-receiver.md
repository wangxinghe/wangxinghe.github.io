---
layout: post
comments: true
title: "registerReceiver过程"
description: "registerReceiver过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析广播接收器BroadcastReceiver注册过程.

## 0.文件结构

	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
	frameworks/base/services/core/java/com/android/server/am/BroadcastRecord.java
	
	frameworks/base/core/java/android/content/BroadcastReceiver.java
	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/LoadedApk.java
	frameworks/base/core/java/android/app/ActivityThread.java

## 1.时序图

TODO

## 2.ContextImpl
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

### 2.1 registerReceiver

	@Override
	public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
	    return registerReceiver(receiver, filter, null, null);
	}
	 
	@Override
	public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
	        String broadcastPermission, Handler scheduler) {
	    return registerReceiverInternal(receiver, getUserId(), filter, broadcastPermission, scheduler, getOuterContext(), 0);
	}

### 2.2 registerReceiverInternal

	private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
	        IntentFilter filter, String broadcastPermission,
	        Handler scheduler, Context context, int flags) {
	    // 获取InnerReceiver, 基本思路是从mReceivers映射表中查找和BroadcastReceiver一一对应的ReceiverDispatcher, 进而获取InnerReceiver. 如果没有则新建. 参考【第3节】
	    IIntentReceiver rd = null;
	    if (receiver != null) {
	        if (mPackageInfo != null && context != null) {
	            if (scheduler == null) {
	                scheduler = mMainThread.getHandler();
	            }
	            rd = mPackageInfo.getReceiverDispatcher(
	                receiver, context, scheduler,
	                mMainThread.getInstrumentation(), true);
	        } else {
	            if (scheduler == null) {
	                scheduler = mMainThread.getHandler();
	            }
	            rd = new LoadedApk.ReceiverDispatcher(receiver, context, scheduler, null, true).getIIntentReceiver();
	        }
	    }
	    try {
	        // AMS.registerReceiver注册广播. 参考【第4节】
	        final Intent intent = ActivityManager.getService().registerReceiver(
	                mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
	                broadcastPermission, userId, flags);
	        if (intent != null) {
	            intent.setExtrasClassLoader(getClassLoader());
	            intent.prepareToEnterProcess();
	        }
	        return intent;
	    } catch (RemoteException e) {
	        throw e.rethrowFromSystemServer();
	    }
	}

**ContextImpl.registerReceiver(BroadcastReceiver, IntentFilter)过程：**  
1.根据BroadcastReceiver获取对应的ReceiverDispatcher  
(1)先根据LoadedApk.getReceiverDispatcher, 从ArrayMap<BroadcastReceiver, ReceiverDispatcher>映射表中得到和BroadcastReceiver对应的ReceiverDispatcher  
(2)如果LoadedApk为空, 则新建LoadedApk.ReceiverDispatcher  
2.执行ReceiverDispatcher.getIIntentReceiver()得到InnerReceiver(即IIntentReceiver.Stub)  
3.调用AMS.registerReceiver注册广播.   

## 3.LoadedApk.getReceiverDispatcher
[ -> frameworks/base/core/java/android/app/LoadedApk.java ]

	public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
	        Context context, Handler handler,
	        Instrumentation instrumentation, boolean registered) {
	    synchronized (mReceivers) {
	        LoadedApk.ReceiverDispatcher rd = null;
	        ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
	        if (registered) {
	            map = mReceivers.get(context);
	            if (map != null) {
	                rd = map.get(r);
	            }
	        }
	        if (rd == null) {
	            rd = new ReceiverDispatcher(r, context, handler, instrumentation, registered);
	            if (registered) {
	                if (map == null) {
	                    map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
	                    mReceivers.put(context, map);
	                }
	                map.put(r, rd);
	            }
	        }
	        rd.mForgotten = false;
	        return rd.getIIntentReceiver();
	    }
	}

ReceiverDispatcher作用:  
(1) 首先BroadcastReceiver和ReceiverDispatcher是一一对应的, 一个广播接收器对应一个广播派发器.  
(2) 它是负责将广播Broadcast派发给BroadcastReceiver执行的广播派发器.  
(3) 它主要是通过内部类InnerReceiver和Args执行广播派发过程.  

## 4.ActivityManagerService.registerReceiver
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	public Intent registerReceiver(IApplicationThread caller, String callerPackage,
	        IIntentReceiver receiver, IntentFilter filter, String permission, int userId, int flags) {
	    // 不能从isolated进程调用, 否则会抛出异常
	    enforceNotIsolatedCaller("registerReceiver");
	 
	    ProcessRecord callerApp = null;
	    final boolean visibleToInstantApps = (flags & Context.RECEIVER_VISIBLE_TO_INSTANT_APPS) != 0;
	    int callingUid;
	    int callingPid;
	    boolean instantApp;
	    synchronized(this) {
	        if (caller != null) {
	            callerApp = getRecordForAppLocked(caller);
	            callingUid = callerApp.info.uid;
	            callingPid = callerApp.pid;
	        } else {
	            callerPackage = null;
	            callingUid = Binder.getCallingUid();
	            callingPid = Binder.getCallingPid();
	        }
	 
	        instantApp = isInstantApp(callerApp, callerPackage, callingUid);
	        userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
	                ALLOW_FULL_ONLY, "registerReceiver", callerPackage);
	 
	        // 先根据IntentFilter获取actions列表, 再从mStickyBroadcasts中找到包含action的所有Intent, 统一集中到stickyIntents
	        Iterator<String> actions = filter.actionsIterator();
	        if (actions == null) {
	            ArrayList<String> noAction = new ArrayList<String>(1);
	            noAction.add(null);
	            actions = noAction.iterator();
	        }
	        ArrayList<Intent> stickyIntents = null;
	        int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
	        while (actions.hasNext()) {
	            String action = actions.next();
	            for (int id : userIds) {
	                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
	                if (stickies != null) {
	                    ArrayList<Intent> intents = stickies.get(action);
	                    if (intents != null) {
	                        if (stickyIntents == null) {
	                            stickyIntents = new ArrayList<Intent>();
	                        }
	                        stickyIntents.addAll(intents);
	                    }
	                }
	            }
	        }
	    }
	 
	    // 遍历stickyIntents, 根据IntentFilter匹配action&category&schema&data各属性都匹配的intent, 统一集中到allSticky
	    ArrayList<Intent> allSticky = null;
	    if (stickyIntents != null) {
	        final ContentResolver resolver = mContext.getContentResolver();
	        for (int i = 0, N = stickyIntents.size(); i < N; i++) {
	            Intent intent = stickyIntents.get(i);
	            // instantApp过滤
	            if (instantApp && (intent.getFlags() & Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS) == 0) {
	                continue;
	            }
	 
	            if (filter.match(resolver, intent, true, TAG) >= 0) {
	                if (allSticky == null) {
	                    allSticky = new ArrayList<Intent>();
	                }
	                allSticky.add(intent);
	            }
	        }
	    }
	 
	    // The first sticky in the list is returned directly back to the client.
	    Intent sticky = allSticky != null ? allSticky.get(0) : null;
	 
	    synchronized (this) {
	        // mRegisteredReceivers中添加ReceiverList, mReceiverResolver中添加BroadcastFilter
	        ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
	        if (rl == null) {
	            rl = new ReceiverList(this, callerApp, callingPid, callingUid, userId, receiver);
	            if (rl.app != null) {
	                // 如果超出单个App允许注册的最大广播数1000，则抛出异常
	                final int totalReceiversForApp = rl.app.receivers.size();
	                if (totalReceiversForApp >= MAX_RECEIVERS_ALLOWED_PER_APP) {
	                    ...
	                }
	                rl.app.receivers.add(rl);
	            } else {
	                // 进程为空, 释放广播
	                receiver.asBinder().linkToDeath(rl, 0);
	                rl.linkedToDeath = true;
	            }
	            mRegisteredReceivers.put(receiver.asBinder(), rl);
	        }
	        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
	                permission, callingUid, userId, instantApp, visibleToInstantApps);
	        if (!rl.containsFilter(filter)) {
	            rl.add(bf);
	            mReceiverResolver.addFilter(bf);
	        }
	     
	        // 遍历allSticky列表, 根据intent封装BroadcastRecord广播记录, 并入队BroadcastQueue
	        if (allSticky != null) {
	            ArrayList receivers = new ArrayList();
	            receivers.add(bf);
	 
	            final int stickyCount = allSticky.size();
	            for (int i = 0; i < stickyCount; i++) {
	                Intent intent = allSticky.get(i);
	                // 根据intent的flags标记获取前台/后台/离线广播队列BroadcastQueue. 参考【第5节】
	                BroadcastQueue queue = broadcastQueueForIntent(intent);
	                // 创建BroadcastRecord
	                BroadcastRecord r = new BroadcastRecord(queue, intent, null,
	                        null, -1, -1, false, null, null, OP_NONE, null, receivers,
	                        null, 0, null, null, false, true, true, -1, false,
	                        false /* only PRE_BOOT_COMPLETED should be exempt, no stickies */);
	                // 添加到无序广播列表mParallelBroadcasts. 参考【第6节】
	                queue.enqueueParallelBroadcastLocked(r);
	                // 处理广播
	                queue.scheduleBroadcastsLocked();
	            }
	        }
	 
	        return sticky;
	    }
	}

## 5.broadcastQueueForIntent

	BroadcastQueue broadcastQueueForIntent(Intent intent) {
	    if (isOnOffloadQueue(intent.getFlags())) {
	        return mOffloadBroadcastQueue;
	    }
	 
	    final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
	    return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
	}

## 6.BroadcastQueue
[ -> frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java ]

### 6.1 enqueueParallelBroadcastLocked

	public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
	    mParallelBroadcasts.add(r);
	}

### 6.2 scheduleBroadcastsLocked

	public void scheduleBroadcastsLocked() {
	    if (mBroadcastsScheduled) {
	        return;
	    }
	    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
	    mBroadcastsScheduled = true;
	}

### 6.3 BroadcastHandler.BROADCAST_INTENT_MSG

	private final class BroadcastHandler extends Handler {
	    public BroadcastHandler(Looper looper) {
	        super(looper, null, true);
	    }
	 
	    @Override
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case BROADCAST_INTENT_MSG: {
	                processNextBroadcast(true);
	            } break;
	            case BROADCAST_TIMEOUT_MSG: {
	                synchronized (mService) {
	                    broadcastTimeoutLocked(true);
	                }
	            } break;
	        }
	    }
	}

### 6.4 processNextBroadcast

	final void processNextBroadcast(boolean fromMsg) {
	    synchronized (mService) {
	        processNextBroadcastLocked(fromMsg, false);
	    }
	}
	 
	final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
	    mService.updateCpuStats();
	 
	    if (fromMsg) {
	        mBroadcastsScheduled = false;
	    }
	 
	    // First, deliver any non-serialized broadcasts right away.
	    // 遍历无序广播列表mParallelBroadcasts, 获取每个广播对象BroadcastRecord的所有接收者BroadcastFilter，并发送给BroadcastFilter
	    // 同时将广播添加到历史广播列表
	    BroadcastRecord r;
	    while (mParallelBroadcasts.size() > 0) {
	        r = mParallelBroadcasts.remove(0);
	        r.dispatchTime = SystemClock.uptimeMillis();
	        r.dispatchClockTime = System.currentTimeMillis();
	 
	        final int N = r.receivers.size();
	        for (int i=0; i<N; i++) {
	            Object target = r.receivers.get(i);
		        // 将当前广播发送给广播接收器处理. 参考【第7.1节】
	            deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
	        }
	        addBroadcastToHistoryLocked(r);
	    }
	 
	    // 检查当前是否有进程正在处理广播mPendingBroadcast.如有则不往下继续执行, 先等广播处理完; 若无则继续往下执行, 处理下一个广播
	    if (mPendingBroadcast != null) {
	        boolean isDead;
	        if (mPendingBroadcast.curApp.pid > 0) {
	            synchronized (mService.mPidsSelfLocked) {
	                ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
	                isDead = proc == null || proc.isCrashing();
	            }
	        } else {
	            final ProcessRecord proc = mService.mProcessList.mProcessNames.get(
	                    mPendingBroadcast.curApp.processName, mPendingBroadcast.curApp.uid);
	            isDead = proc == null || !proc.pendingStart;
	        }
	        if (!isDead) {
	            // It's still alive, so keep waiting
	            return;
	        } else {
	            mPendingBroadcast.state = BroadcastRecord.IDLE;
	            mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
	            mPendingBroadcast = null;
	        }
	    }
	 
	    boolean looped = false;
	 
	    // 其他广播处理, 按先后顺序依次是:
	    // 1. mAlarmBroadcasts
	    // 2. the next "overdue" deferral
	    // 3. the next ordinary ordered broadcast
	    // 4. the next not-yet-overdue deferral
	    do {
	        final long now = SystemClock.uptimeMillis();
	        // 获取下一个广播
	        r = mDispatcher.getNextBroadcastLocked(now);
	 
	        // 如果广播都处理完了, 则做一些扫尾工作包括: 延迟广播检查和执行, GC, oomAdj更新等. 不再往下执行
	        if (r == null) {
	            mDispatcher.scheduleDeferralCheckLocked(false);
	            mService.scheduleAppGcsLocked();
	            if (looped) {
	                mService.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_START_RECEIVER);
	            }
	 
	            if (mService.mUserController.mBootCompleted && mLogLatencyMetrics) {
	                mLogLatencyMetrics = false;
	            }
	 
	            return;
	        }
	 
	        boolean forceReceive = false;
	 
	        // 处理并执行超时广播
	        int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
	        if (mService.mProcessesReady && !r.timeoutExempt && r.dispatchTime > 0) {
	            if ((numReceivers > 0) && (now > r.dispatchTime + (2 * mConstants.TIMEOUT * numReceivers))) {
	                broadcastTimeoutLocked(false); // forcibly finish this broadcast
	                forceReceive = true;
	                r.state = BroadcastRecord.IDLE;
	            }
	        }
	 
	        if (r.state != BroadcastRecord.IDLE) {
	            return;
	        }
	 
	        // Is the current broadcast is done for any reason?
	        // 当前广播处理完了, 则处理cross-BroadcastRecord refcount相关，并执行广播接收
	        if (r.receivers == null || r.nextReceiver >= numReceivers || r.resultAbort || forceReceive) {
	            if (r.resultTo != null) {
	                boolean sendResult = true;
	 
	                if (r.splitToken != 0) {
	                    int newCount = mSplitRefcounts.get(r.splitToken) - 1;
	                    if (newCount == 0) {
	                        mSplitRefcounts.delete(r.splitToken);
	                    } else {
	                        sendResult = false;
	                        mSplitRefcounts.put(r.splitToken, newCount);
	                    }
	                }
	                if (sendResult) {
	                    performReceiveLocked(r.callerApp, r.resultTo, new Intent(r.intent), r.resultCode,
	                            r.resultData, r.resultExtras, false, false, r.userId);
	                    r.resultTo = null;
	                }
	            }
	     
	            // 取消超时广播
	            cancelBroadcastTimeoutLocked();
	     
	            // 将当前广播添加到历史广播列表
	            addBroadcastToHistoryLocked(r);
	 
	            // 记录隐式广播
	            if (r.intent.getComponent() == null && r.intent.getPackage() == null
	                    && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
	                mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
	                        r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
	            }
	            // 当前广播处理完, mCurrentBroadcast置空
	            mDispatcher.retireBroadcastLocked(r);
	            r = null;
	            looped = true;
	            continue;
	        }
	 
	        // 如果下一个广播是延迟广播, 则针对mSplitRefcounts做相关处理, 同时将延迟广播添加到延迟广播列表
	        if (!r.deferred) {
	            final int receiverUid = r.getReceiverUid(r.receivers.get(r.nextReceiver));
	            if (mDispatcher.isDeferringLocked(receiverUid)) {
	                BroadcastRecord defer;
	                if (r.nextReceiver + 1 == numReceivers) {
	                    defer = r;
	                    mDispatcher.retireBroadcastLocked(r);
	                } else {
	                    defer = r.splitRecipientsLocked(receiverUid, r.nextReceiver);
	                    if (r.resultTo != null) {
	                        int token = r.splitToken;
	                        if (token == 0) {
	                            r.splitToken = defer.splitToken = nextSplitTokenLocked();
	                            mSplitRefcounts.put(r.splitToken, 2);
	                        } else {
	                            final int curCount = mSplitRefcounts.get(token);
	                            mSplitRefcounts.put(token, curCount + 1);
	                        }
	                    }
	                }
	                mDispatcher.addDeferredBroadcast(receiverUid, defer);
	                r = null;
	                looped = true;
	                continue;
	            }
	        }
	    } while (r == null);
	 
	    // 处理下一个广播, 并通过Handler延迟发送广播超时消息, 便于杀死超时广播
	    int recIdx = r.nextReceiver++;
	    r.receiverTime = SystemClock.uptimeMillis();
	    if (recIdx == 0) {
	        r.dispatchTime = r.receiverTime;
	        r.dispatchClockTime = System.currentTimeMillis();
	    }
	    if (!mPendingBroadcastTimeoutMessage) {
	        long timeoutTime = r.receiverTime + mConstants.TIMEOUT;
	        setBroadcastTimeoutLocked(timeoutTime);
	    }
	 
	    final BroadcastOptions brOptions = r.options;
	    final Object nextReceiver = r.receivers.get(recIdx);
	    if (nextReceiver instanceof BroadcastFilter) {
	        BroadcastFilter filter = (BroadcastFilter)nextReceiver;
	        // 将当前广播发送给广播接收器处理. 参考【第7.1节】
	        deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
	        // 当前广播处理完后, 继续处理下一个广播
	        if (r.receiver == null || !r.ordered) {
	            r.state = BroadcastRecord.IDLE;
	            scheduleBroadcastsLocked();
	        } else {
	            if (filter.receiverList != null) {
	                maybeAddAllowBackgroundActivityStartsToken(filter.receiverList.app, r);
	            }
	            if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
	                scheduleTempWhitelistLocked(filter.owningUid, brOptions.getTemporaryAppWhitelistDuration(), r);
	            }
	        }
	        return;
	    }
	 
	    // 各种检查包括：targetSdkVersion, Appop操作权限, 防火墙, 权限review, 用户运行状态, 后台执行检查等. 如果不符合条件将skip
	    boolean skip = false;
	    ...
	 
	    if (skip) {
	        // skip当前广播, 执行下一个广播
	        r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED;
	        r.receiver = null;
	        r.curFilter = null;
	        r.state = BroadcastRecord.IDLE;
	        r.manifestSkipCount++;
	        scheduleBroadcastsLocked();
	        return;
	    }

	    // 设置当前广播的Receiver为已接收状态DELIVERY_DELIVERED
	    ResolveInfo info = (ResolveInfo)nextReceiver;
	    ComponentName component = new ComponentName(info.activityInfo.applicationInfo.packageName, info.activityInfo.name);
	    r.manifestCount++;
	    r.delivery[recIdx] = BroadcastRecord.DELIVERY_DELIVERED;
	    r.state = BroadcastRecord.APP_RECEIVE;
	    r.curComponent = component;
	    r.curReceiver = info.activityInfo;
	 
	    if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
	        scheduleTempWhitelistLocked(receiverUid, brOptions.getTemporaryAppWhitelistDuration(), r);
	    }
	 
	    // Broadcast is being executed, its package can't be stopped.
	    AppGlobals.getPackageManager().setPackageStoppedState(r.curComponent.getPackageName(), false, r.userId);
	 
	    // 若当前进程活着, 则处理当前广播
	    if (app != null && app.thread != null && !app.killed) {
	        app.addPackage(info.activityInfo.packageName, info.activityInfo.applicationInfo.longVersionCode, mService.mProcessStats);
	        maybeAddAllowBackgroundActivityStartsToken(app, r);
	        processCurBroadcastLocked(r, app, skipOomAdj);
	        return;
	    }
	 
	    // 若当前进程未启动, 则先启动进程, 若进程启动失败, 则finishReceiver并准备执行下一个广播
	    if ((r.curApp=mService.startProcessLocked(targetProcess,
	            info.activityInfo.applicationInfo, true,
	            r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
	            new HostingRecord("broadcast", r.curComponent),
	            (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
	                    == null) {
	        finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
	        scheduleBroadcastsLocked();
	        r.state = BroadcastRecord.IDLE;
	        return;
	    }
	 
	    maybeAddAllowBackgroundActivityStartsToken(r.curApp, r);
	    mPendingBroadcast = r;
	    mPendingBroadcastRecvIndex = recIdx;
	}

**processNextBroadcast过程：**  
1.无序广播处理：  
(1)遍历无序广播列表mParallelBroadcasts  
(2)获取每个广播对象BroadcastRecord的全部广播接收者列表BroadcastFilter  
(3)调用BroadcastQueue.deliverToRegisteredReceiverLocked, 依次将当前广播发送给步骤1的广播接收者处理  
(4)将已处理的广播加入历史广播列表  

2.检查当前是否有进程正在处理广播mPendingBroadcast. 如有则不往下继续执行, 先等广播处理完; 若无则继续往下执行, 处理下一个广播  

3.获取有序或延时广播处理：执行do while循环, 直到查找到下一个广播r：  
(1)获取下一个广播r  
(2)若当前广播为空, 则做一些扫尾工作包括: 延迟广播检查和执行, GC, oomAdj更新等并return  
(3)若当前广播非空, 则判断广播处理是否超时, 若超时则调用broadcastTimeoutLocked超时处理并弹出ANR提示(超时时间是10秒), finish当前广播并设置广播状态为IDLE  
(4)若当前广播的所有广播接收器列表已完成处理:  
(4.1)更新cross-BroadcastRecord refcount计数，并调用performReceiveLocked执行BroadcastReceiver.onReceive广播接收操作和执行广播finish操作  
(4.2)取消超时广播  
(4.3)将当前广播添加到历史广播列表  
(4.4)若当前广播为隐式广播, 则记录隐式广播  
(4.5)当前广播处理完, mCurrentBroadcast置空, continue并执行下一轮do while循环  
(5)如果下一个广播是延迟广播, 则针对mSplitRefcounts做相关处理, 同时将延迟广播添加到延迟广播列表  

4.将步骤3的有序或延时广播r发送给当前索引nextReceiver的广播接收器处理(filter非空)：  
(1)获取广播r的当前广播接收器BroadcastFilter的索引nextReceiver. 并通过Handler超时机制发送消息, 便于杀死超时广播  
(2)获取索引nextReceiver的广播接收器BroadcastFilter:  
(2.1)调用deliverToRegisteredReceiverLocked将当前广播发送给广播接收器BroadcastFilter执行  
(2.2)若当前广播处理完, 设置广播状态为IDLE, 并调用scheduleBroadcastsLocked继续处理下一个广播并return  

5.若filter为空, 根据nextReceiver得到ResolveInfo且非空:  
(1)先执行各种检查包括：targetSdkVersion, Appop操作权限, 防火墙, 权限review, 用户运行状态, 后台执行检查等.  
(2)如果上述条件不符合, 则设置广播接收器状态为DELIVERY_SKIPPED, 设置广播状态为IDLE, 调用scheduleBroadcastsLocked处理下一个广播并return  
(3)根据ResolveInfo构造ComponentName对象设置给当前广播, 设置广播接收器状态为DELIVERY_DELIVERED, 设置广播状态为APP_RECEIVE  
(4)若当前进程活着, 则处理当前广播  
(5)若当前进程未启动, 则先启动进程, 若进程启动失败, 则finishReceiverLocked结束当前广播接收操作, 设置广播状态为IDLE, 并调用scheduleBroadcastsLocked执行下一个广播接收操作  

## 7.BroadcastQueue无序广播
[ -> frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java ]

### 7.1 deliverToRegisteredReceiverLocked

	private void deliverToRegisteredReceiverLocked(BroadcastRecord r, BroadcastFilter filter, boolean ordered, int index) {
	    // 各种检查：权限, 防火墙, 权限review. 如果不符合条件将skip
	    boolean skip = false;  
	    ...
	    if (skip) {
	        r.delivery[index] = BroadcastRecord.DELIVERY_SKIPPED;
	        return;
	    }
	 
	    // 设置当前广播的Receiver为已接收状态DELIVERY_DELIVERED
	    r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED;
	 
	    // 如果是有序广播, 则更新广播状态为CALL_IN_RECEIVE, 同时将当前广播添加到curReceivers列表和更新oomAdj
	    if (ordered) {
	        r.receiver = filter.receiverList.receiver.asBinder();
	        r.curFilter = filter;
	        filter.receiverList.curBroadcast = r;
	        r.state = BroadcastRecord.CALL_IN_RECEIVE;
	        if (filter.receiverList.app != null) {
	            r.curApp = filter.receiverList.app;
	            filter.receiverList.app.curReceivers.add(r);
	            mService.updateOomAdjLocked(r.curApp, true, OomAdjuster.OOM_ADJ_REASON_START_RECEIVER);
	        }
	    }
	    if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
	        // 对于有序广播, 如果是inFullBackup状态, 则跳过当前Receiver
	        if (ordered) {
	            skipReceiverLocked(r);
	        }
	    } else {
	        // 设置广播接收时间
	        r.receiverTime = SystemClock.uptimeMillis();
	        maybeAddAllowBackgroundActivityStartsToken(filter.receiverList.app, r);
	        // 广播接收器执行广播接收操作. 执行【第7.2节】
	        performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
	                new Intent(r.intent), r.resultCode, r.resultData,
	                r.resultExtras, r.ordered, r.initialSticky, r.userId);
	        // 无序广播是fire-and-forget处理方式, 不需要调用finishReceiverLocked(), 通过activity-start token方式管理无序广播
	        if (r.allowBackgroundActivityStarts && !r.ordered) {
	            postActivityStartTokenRemoval(filter.receiverList.app, r);
	        }
	    }

	    // 有序广播设置状态CALL_DONE_RECEIVE
	    if (ordered) {
	        r.state = BroadcastRecord.CALL_DONE_RECEIVE;
	    }
	}

**BroadcastQueue.deliverToRegisteredReceiverLocked过程:**  
入参为(BroadcastRecord r, BroadcastFilter filter, boolean ordered, int index), 其中r代表当前广播, filter代表广播接收器  
1.首先进行条件检查：权限, 防火墙, 权限review. 如果不符合条件将直接return并设置当前广播的接收器BroadcastReceiver状态为DELIVERY_SKIPPED  
2.如果符合条件, 设置当前广播的接收器BroadcastReceiver状态为DELIVERY_DELIVERED  
3.如果是有序广播, 更新当前广播状态为CALL_IN_RECEIVE(表示处理中), 同时将当前广播添加到curReceivers列表并更新oomAdj  
4.执行广播接收：  
(1)如果当前广播接收器的进程在inFullBackup状态, 且当前为有序广播, 则调用skipReceiverLocked跳过当前广播并处理下一个广播  
(2)如果进程为非inFullBackup状态, 则调用performReceiveLocked执行广播接收操作.   
(3)如果当前为无序广播,由于无序广播是fire-and-forget处理方式, 因此通过activity-start token方式管理无序广播  
5.如果是有序广播, 更新当前广播状态为CALL_DONE_RECEIVE(表示处理完毕)  

### 7.2 performReceiveLocked

	void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
	        Intent intent, int resultCode, String data, Bundle extras,
	        boolean ordered, boolean sticky, int sendingUser)
	        throws RemoteException {
	    // Send the intent to the receiver asynchronously using one-way binder calls.
	    if (app != null) {
	        if (app.thread != null) {
	            // If we have an app thread, do the call through that so it is
	            // correctly ordered with other one-way calls.
	            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
	                    data, extras, ordered, sticky, sendingUser, app.getReportedProcState());
	        }
	    } else {
			// 参考【第8节】
	        receiver.performReceive(intent, resultCode, data, extras, ordered, sticky, sendingUser);
	    }
	}

**BroadcastQueue.performReceiveLocked过程:**  
1.基本等价于执行ReceiverDispatcher.InnerReceiver.performReceive()方法.  

## 8.ReceiverDispatcher
[ -> frameworks/base/core/java/android/app/LoadedApk.java ]

### 8.1 InnerReceiver.performReceive

	static final class ReceiverDispatcher {
	 
	    final static class InnerReceiver extends IIntentReceiver.Stub {
	        final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
	        final LoadedApk.ReceiverDispatcher mStrongRef;
	 
	        InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
	            mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
	            mStrongRef = strong ? rd : null;
	        }
	 
	        @Override
	        public void performReceive(Intent intent, int resultCode, String data,
	                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
	            final LoadedApk.ReceiverDispatcher rd;
	            if (intent == null) {
	                rd = null;
	            } else {
	                rd = mDispatcher.get();
	            }
	            if (rd != null) {
	                rd.performReceive(intent, resultCode, data, extras, ordered, sticky, sendingUser);
	            } else {
	                // 结束当前广播
	                IActivityManager mgr = ActivityManager.getService();
	                if (extras != null) {
	                    extras.setAllowFds(false);
	                }
	                mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
	            }
	        }
	    }
	}

**ReceiverDispatcher.InnerReceiver.performReceive()过程:**  
1.若LoadedApk.ReceiverDispatcher对象非空, 则执行LoadedApk.ReceiverDispatcher.performReceive()方法处理广播  
2.若LoadedApk.ReceiverDispatcher对象为空,  则调用AMS.finishReceiver结束当前广播  

### 8.2 performReceive

    public void performReceive(Intent intent, int resultCode, String data,
            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
        final Args args = new Args(intent, resultCode, data, extras, ordered, sticky, sendingUser);
         
        if (intent == null || !mActivityThread.post(args.getRunnable())) {
            if (mRegistered && ordered) {
                IActivityManager mgr = ActivityManager.getService();
                args.sendFinished(mgr);
            }
        }
    }
**LoadedApk.ReceiverDispatcher.performReceive()过程：**  
1.封装Runnable对象Args, 执行Runnable, 若执行失败则调用AMS.finishReceiver  
2.Runnable执行过程：  
(1)若当前为有序广播且BroadcastReceiver或Intent为空, 则调用AMS.finishReceiver结束当前广播和广播接收器并return  
(2)调用BroadcastReceiver.onReceive接收广播  
(3)调用AMS.finishReceiver结束当前广播并继续处理下一个广播接收器操作  

## 9.Args
[ -> frameworks/base/core/java/android/app/LoadedApk.java ]

### 9.1 getRunnable

    public final Runnable getRunnable() {
        return () -> {
            final BroadcastReceiver receiver = mReceiver;
            final boolean ordered = mOrdered;

            final IActivityManager mgr = ActivityManager.getService();
            final Intent intent = mCurIntent;

            mCurIntent = null;
            mDispatched = true;
            mRunCalled = true;
            // 若当前receiver或intent为空, 则直接返回.
            // 对于已注册的有序广播, 需要先结束当前广播接收操作然后执行下一个广播接收操作. 参考【第9.2节】
            if (receiver == null || intent == null || mForgotten) {
                if (mRegistered && ordered) {
                    sendFinished(mgr);
                }
                return;
            }

            ClassLoader cl = mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            intent.prepareToEnterProcess();
            setExtrasClassLoader(cl);
            receiver.setPendingResult(this);
            // 调用BroadcastReceiver.onReceive方法
            receiver.onReceive(mContext, intent);

            // finish当前广播
            if (receiver.getPendingResult() != null) {
                finish();
            }
        };
    }

    public final void finish() {
        if (mType == TYPE_COMPONENT) {
            final IActivityManager mgr = ActivityManager.getService();
            if (QueuedWork.hasPendingWork()) {
                QueuedWork.queue(new Runnable() {
                    @Override
                    public void run() {
                        sendFinished(mgr);
                    }
                }, false);
            } else {
                sendFinished(mgr);
            }
        } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
            final IActivityManager mgr = ActivityManager.getService();
            sendFinished(mgr);
        }
    }

### 9.2 sendFinished

    public void sendFinished(IActivityManager am) {
        synchronized (this) {
            mFinished = true;

            if (mResultExtras != null) {
                mResultExtras.setAllowFds(false);
            }
            if (mOrderedHint) {
                am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras, mAbortBroadcast, mFlags);
            } else {
                am.finishReceiver(mToken, 0, null, null, false, mFlags);
            }
        }
    }

### 9.3 AMS.finishReceiver

	public void finishReceiver(IBinder who, int resultCode, String resultData,
	        Bundle resultExtras, boolean resultAbort, int flags) {
	    final long origId = Binder.clearCallingIdentity();
	    try {
	        boolean doNext = false;
	        BroadcastRecord r;
	        BroadcastQueue queue;
	 
	        synchronized(this) {
	            // 获取广播队列BroadcastQueue
	            if (isOnOffloadQueue(flags)) {
	                queue = mOffloadBroadcastQueue;
	            } else {
	                queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0 ? mFgBroadcastQueue : mBgBroadcastQueue;
	            }
	 
	            // 从广播队列中查找广播接收器(令牌为who)对应的广播BroadcastRecord
	            r = queue.getMatchingOrderedReceiver(who);
	            if (r != null) {
	                // 结束本次广播接收操作
	                doNext = r.queue.finishReceiverLocked(r, resultCode, resultData, resultExtras, resultAbort, true);
	            }
	            if (doNext) {
	                // 执行下一次广播接收操作
	                r.queue.processNextBroadcastLocked(/*fromMsg=*/ false, /*skipOomAdj=*/ true);
	            }
	            // 更新oomAdj
	            trimApplicationsLocked(OomAdjuster.OOM_ADJ_REASON_FINISH_RECEIVER);
	        }
	    } finally {
	        Binder.restoreCallingIdentity(origId);
	    }
	}

### 9.4 BroadcastReceiver.onReceive

## 10.BroadcastQueue延时或有序广播
[ -> frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java ]

### 10.1 BroadcastDispatcher.getNextBroadcastLocked

[ -> frameworks/base/services/core/java/com/android/server/am/BroadcastDispatcher.java ]

    public BroadcastRecord getNextBroadcastLocked(final long now) {
        if (mCurrentBroadcast != null) {
            return mCurrentBroadcast;
        }

        final boolean someQueued = !mOrderedBroadcasts.isEmpty();

        BroadcastRecord next = null;
        if (!mAlarmBroadcasts.isEmpty()) {
            next = popLocked(mAlarmBroadcasts);
        }

        if (next == null && !mDeferredBroadcasts.isEmpty()) {
            // We're going to deliver either:
            // 1. the next "overdue" deferral; or
            // 2. the next ordinary ordered broadcast; *or*
            // 3. the next not-yet-overdue deferral.

            for (int i = 0; i < mDeferredBroadcasts.size(); i++) {
                Deferrals d = mDeferredBroadcasts.get(i);
                if (now < d.deferUntil && someQueued) {
                    // stop looking when we haven't hit the next time-out boundary
                    // but only if we have un-deferred broadcasts waiting,
                    // otherwise we can deliver whatever deferred broadcast
                    // is next available.
                    break;
                }

                if (d.broadcasts.size() > 0) {
                    next = d.broadcasts.remove(0);
                    // apply deferral-interval decay policy and move this uid's
                    // deferred broadcasts down in the delivery queue accordingly
                    mDeferredBroadcasts.remove(i); // already 'd'
                    d.deferredBy = calculateDeferral(d.deferredBy);
                    d.deferUntil += d.deferredBy;
                    insertLocked(mDeferredBroadcasts, d);
                    break;
                }
            }
        }

        if (next == null && someQueued) {
            next = mOrderedBroadcasts.remove(0);
        }

        mCurrentBroadcast = next;
        return next;
    }

**getNextBroadcastLocked过程：**  
目的是获取下一个要处理的广播, 获取顺序一次是:  
1.mAlarmBroadcasts(Alarm广播)  
2.the next "overdue" deferral(延时广播)  
3.the next ordinary ordered broadcast(有序广播)  

### 10.2 broadcastTimeoutLocked

	final void broadcastTimeoutLocked(boolean fromMsg) {
	    if (fromMsg) {
	        mPendingBroadcastTimeoutMessage = false;
	    }
	 
	    if (mDispatcher.isEmpty() || mDispatcher.getActiveBroadcastLocked() == null) {
	        return;
	    }
	 
	    long now = SystemClock.uptimeMillis();
	    BroadcastRecord r = mDispatcher.getActiveBroadcastLocked();
	    if (fromMsg) {
	        // 忽略mProcessesReady前的广播超时
	        if (!mService.mProcessesReady) {
	            return;
	        }
	 
	        // 不考虑广播超时时
	        if (r.timeoutExempt) {
	            return;
	        }
	 
	        // 广播超时时间10秒
	        long timeoutTime = r.receiverTime + mConstants.TIMEOUT;
	        // 若当前未超时, 则取消超时处理
	        if (timeoutTime > now) {
	            setBroadcastTimeoutLocked(timeoutTime);
	            return;
	        }
	    }
	 
	    // 若当前广播正在等待后台服务处理, 则设置广播状态为IDLE, 处理下一个广播
	    if (r.state == BroadcastRecord.WAITING_SERVICES) {
	        r.curComponent = null;
	        r.state = BroadcastRecord.IDLE;
	        processNextBroadcast(false);
	        return;
	    }
	 
	    // 发送ANR信息(忽略debug场景的ANR)
	    final boolean debugging = (r.curApp != null && r.curApp.isDebugging());
	 
	    r.receiverTime = now;
	    if (!debugging) {
	        r.anrCount++;
	    }
	 
	    ProcessRecord app = null;
	    String anrMessage = null;
	 
	    Object curReceiver;
	    if (r.nextReceiver > 0) {
	        curReceiver = r.receivers.get(r.nextReceiver-1);
	        r.delivery[r.nextReceiver-1] = BroadcastRecord.DELIVERY_TIMEOUT;
	    } else {
	        curReceiver = r.curReceiver;
	    }
	    if (curReceiver != null && curReceiver instanceof BroadcastFilter) {
	        BroadcastFilter bf = (BroadcastFilter)curReceiver;
	        if (bf.receiverList.pid != 0 && bf.receiverList.pid != ActivityManagerService.MY_PID) {
	            synchronized (mService.mPidsSelfLocked) {
	                app = mService.mPidsSelfLocked.get(bf.receiverList.pid);
	            }
	        }
	    } else {
	        app = r.curApp;
	    }
	 
	    if (app != null) {
	        anrMessage = "Broadcast of " + r.intent.toString();
	    }
	 
	    if (mPendingBroadcast == r) {
	        mPendingBroadcast = null;
	    }
	 
	    // Move on to the next receiver.
	    finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
	    scheduleBroadcastsLocked();
	 
	    if (!debugging && anrMessage != null) {
	        // Post the ANR to the handler since we do not want to process ANRs while
	        // potentially holding our lock.
	        mHandler.post(new AppNotResponding(app, anrMessage));
	    }
	}

**BroadcastQueue.broadcastTimeoutLocked过程:**   
1.边界条件处理：排除进程还未Ready、timeExempt和等待后台服务处理等场景的超时  
2.如果广播从r.receiverTime到现在的时间超过10秒, 则认为超时并弹出ANR弹窗  

### 10.3 processCurBroadcastLocked

	private final void processCurBroadcastLocked(BroadcastRecord r,
	        ProcessRecord app, boolean skipOomAdj) throws RemoteException {
	    if (app.inFullBackup) {
	        skipReceiverLocked(r);
	        return;
	    }
	 
	    r.receiver = app.thread.asBinder();
	    r.curApp = app;
	    app.curReceivers.add(r);
	    app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_RECEIVER);
	    mService.mProcessList.updateLruProcessLocked(app, false, null);
	    if (!skipOomAdj) {
	        mService.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
	    }
	 
	    // Tell the application to launch this receiver.
	    r.intent.setComponent(r.curComponent);
	 
	    boolean started = false;
	    try {
	        mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
	                PackageManager.NOTIFY_PACKAGE_USE_BROADCAST_RECEIVER);
	        app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
	                mService.compatibilityInfoForPackage(r.curReceiver.applicationInfo),
	                r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
	                app.getReportedProcState());
	        started = true;
	    } finally {
	        if (!started) {
	            r.receiver = null;
	            r.curApp = null;
	            app.curReceivers.remove(r);
	        }
	    }
	}

**processCurBroadcastLocked调用链依次是：**  
ApplicationThread.scheduleReceiver -> H.RECEIVER -> handleReceiver -> BroadcastReceiver.onReceive

#### (1) ApplicationThread.scheduleReceiver  
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

	private class ApplicationThread extends IApplicationThread.Stub {
	    ...
	    public final void scheduleReceiver(Intent intent, ActivityInfo info,
	            CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
	            boolean sync, int sendingUser, int processState) {
	        updateProcessState(processState, false);
	        ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
	                sync, false, mAppThread.asBinder(), sendingUser);
	        r.info = info;
	        r.compatInfo = compatInfo;
	        sendMessage(H.RECEIVER, r);
	    }
	    ...
	}

#### (2) H.RECEIVER	 
	 
	class H extends Handler {
	    ...
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            ...
	            case RECEIVER:
	                handleReceiver((ReceiverData)msg.obj);
	                break;
	            ...
	        }
	    }
	}
	 
#### (3) handleReceiver  

	private void handleReceiver(ReceiverData data) {
	    // If we are getting ready to gc after going to the background, well
	    // we are back active so skip it.
	    unscheduleGcIdler();
	 
	    String component = data.intent.getComponent().getClassName();
	 
	    LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);
	 
	    IActivityManager mgr = ActivityManager.getService();
	 
	    Application app = packageInfo.makeApplication(false, mInstrumentation);
	    ContextImpl context = (ContextImpl) app.getBaseContext();
	    if (data.info.splitName != null) {
	        context = (ContextImpl) context.createContextForSplit(data.info.splitName);
	    }
	    java.lang.ClassLoader cl = context.getClassLoader();
	    data.intent.setExtrasClassLoader(cl);
	    data.intent.prepareToEnterProcess();
	    data.setExtrasClassLoader(cl);
	    BroadcastReceiver receiver = packageInfo.getAppFactory().instantiateReceiver(cl, data.info.name, data.intent);
	 
	    try {
	        sCurrentBroadcastIntent.set(data.intent);
	        receiver.setPendingResult(data);
	        receiver.onReceive(context.getReceiverRestrictedContext(), data.intent);
	    } catch (Exception e) {
	        ...
	    } finally {
	        sCurrentBroadcastIntent.set(null);
	    }
	 
	    if (receiver.getPendingResult() != null) {
	        data.finish();
	    }
	}

### 10.4 finishReceiverLocked

	public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
	        String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
	    final int state = r.state;
	    final ActivityInfo receiver = r.curReceiver;
	    final long finishTime = SystemClock.uptimeMillis();
	    final long elapsed = finishTime - r.receiverTime;
	    r.state = BroadcastRecord.IDLE;
	    // 如果广播运行时间超过10秒, 则立即移除该广播, 否则延迟移除该广播
	    if (r.allowBackgroundActivityStarts && r.curApp != null) {
	        if (elapsed > mConstants.ALLOW_BG_ACTIVITY_START_TIMEOUT) {
	            r.curApp.removeAllowBackgroundActivityStartsToken(r);
	        } else {
	            postActivityStartTokenRemoval(r.curApp, r);
	        }
	    }
	 
	    // 设置duration
	    if (r.nextReceiver > 0) {
	        r.duration[r.nextReceiver - 1] = elapsed;
	    }
	 
	    // 如果广播运行时间超过SLOW_TIME(5秒)，则使用延时策略
	    if (!r.timeoutExempt) {
	        if (mConstants.SLOW_TIME > 0 && elapsed > mConstants.SLOW_TIME) {
	            if (!UserHandle.isCore(r.curApp.uid)) {
	                if (r.curApp != null) {
	                    mDispatcher.startDeferring(r.curApp.uid);
	                }
	            }
	        }
	    }
	 
	    // 广播数据置空处理
	    r.receiver = null;
	    r.intent.setComponent(null);
	    if (r.curApp != null && r.curApp.curReceivers.contains(r)) {
	        r.curApp.curReceivers.remove(r);
	    }
	    if (r.curFilter != null) {
	        r.curFilter.receiverList.curBroadcast = null;
	    }
	    r.curFilter = null;
	    r.curReceiver = null;
	    r.curApp = null;
	    mPendingBroadcast = null;
	 
	    r.resultCode = resultCode;
	    r.resultData = resultData;
	    r.resultExtras = resultExtras;
	    if (resultAbort && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
	        r.resultAbort = resultAbort;
	    } else {
	        r.resultAbort = false;
	    }
	    // 当前有后台服务且需要等后台服务处理完后才能finishReceiver的情况处理
	    if (waitForServices && r.curComponent != null && r.queue.mDelayBehindServices
	            && r.queue.mDispatcher.getActiveBroadcastLocked() == r) {
	        ActivityInfo nextReceiver;
	        if (r.nextReceiver < r.receivers.size()) {
	            Object obj = r.receivers.get(r.nextReceiver);
	            nextReceiver = (obj instanceof ActivityInfo) ? (ActivityInfo)obj : null;
	        } else {
	            nextReceiver = null;
	        }
	        if (receiver == null || nextReceiver == null
	                || receiver.applicationInfo.uid != nextReceiver.applicationInfo.uid
	                || !receiver.processName.equals(nextReceiver.processName)) {
	            if (mService.mServices.hasBackgroundServicesLocked(r.userId)) {
	                r.state = BroadcastRecord.WAITING_SERVICES;
	                return false;
	            }
	        }
	    }
	 
	    r.curComponent = null;
	 
		// 返回值表示是否执行下一个广播接收操作, 只有当前广播完成接收操作时才会开始下一次
	    // 即当前广播状态码为APP_RECEIVE或CALL_DONE_RECEIVE
	    return state == BroadcastRecord.APP_RECEIVE || state == BroadcastRecord.CALL_DONE_RECEIVE;
	}

**BroadcastQueue.finishReceiverLocked过程：**  
入参为BroadcastRecord, 出参为bool值表示是否执行下一个广播接收操作    
1.如果当前广播BroadcastRecord运行时间超过10秒, 则移除当前广播; 否则发送消息延迟移除该广播  
2.设置当前广播的duration  
3.如果当前广播运行时间超过SLOW_TIME(5秒), 则使用延时策略处理该广播  
4.当前广播接收器数据置空处理  
5.如果要求waitForServices且当前有后台服务, 则先等后台服务处理, 设置广播状态WAITING_SERVICES并return  

### 10.5 scheduleBroadcastsLocked

参考【第6.2节】

## 11.总结

**ContextImpl.registerReceiver(BroadcastReceiver, IntentFilter)过程：**  
1.根据BroadcastReceiver获取对应的ReceiverDispatcher  
(1)先根据LoadedApk.getReceiverDispatcher, 从ArrayMap<BroadcastReceiver, ReceiverDispatcher>映射表中得到和BroadcastReceiver对应的ReceiverDispatcher  
(2)如果LoadedApk为空, 则新建LoadedApk.ReceiverDispatcher  
2.执行ReceiverDispatcher.getIIntentReceiver()得到InnerReceiver(即IIntentReceiver.Stub)  
3.调用AMS.registerReceiver注册广播.   

**AMS.registerReceiver过程(入参为InnerReceiver和IntentFilter)：**  
1.根据IntentFilter获取List<Intent>列表：  
(1)根据入参IntentFilter获取actions列表, 遍历action列表, 从mStickyBroadcasts中找到匹配action的所有Intent, 添加到stickyIntents  
(2)遍历stickyIntents, 根据入参IntentFilter.match精确匹配action&category&schema&data都符合的Intent, 添加到allSticky  
2.得到ReceiverList, 它是包含List<BroadcastFilter>的广播接收器:  
(1)检查HashMap<IBinder, ReceiverList> mRegisteredReceivers中是否包含key=InnerReceiver的value=ReceiverList, 如果有则命中  
(2)如果没有则新建ReceiverList并将<InnerReceiver, ReceiverList>加入该map  
3.封装BroadcastFilter对象,将其添加到上述ReceiverList和IntentResolver(Intent解析器):  
(1)根据入参IntentFilter和步骤2的ReceiverList封装BroadcastFilter, 并将其添加到上述ReceiverList和IntentResolver中  
(2)注意单个App允许注册的最大广播数1000，超出将抛出异常  
4.遍历步骤1的List<Intent>, 将Intent封装成BroadcastRecord广播记录, 并加入无序广播队列BroadcastQueue(根据Intent.flags决定是前台/后台/离线队列)  
5.调用BroadQueue.scheduleBroadcastsLocked处理广播  

**BroadcastQueue.scheduleBroadcastsLocked过程:**  
通过BroadcastHandler发送BROADCAST_INTENT_MSG, 在消息接收侧调用processNextBroadcast处理广播
processNextBroadcast过程：  
1.无序广播处理：  
(1)遍历无序广播列表mParallelBroadcasts  
(2)获取每个广播对象BroadcastRecord的全部广播接收者列表BroadcastFilter  
(3)调用BroadcastQueue.deliverToRegisteredReceiverLocked, 依次将当前广播发送给步骤1的广播接收者处理  
(4)将已处理的广播加入历史广播列表  
2.检查当前是否有进程正在处理广播mPendingBroadcast. 如有则不往下继续执行, 先等广播处理完; 若无则继续往下执行, 处理下一个广播  
3.获取有序或延时广播处理：执行do while循环, 直到查找到下一个广播r：  
(1)获取下一个广播r  
(2)若当前广播为空, 则做一些扫尾工作包括: 延迟广播检查和执行, GC, oomAdj更新等并return  
(3)若当前广播非空, 则判断广播处理是否超时, 若超时则调用broadcastTimeoutLocked超时处理并弹出ANR提示(超时时间是10秒), finish当前广播并设置广播状态为IDLE  
(4)若当前广播的所有广播接收器列表已完成处理:  
(4.1)更新cross-BroadcastRecord refcount计数，并调用performReceiveLocked执行BroadcastReceiver.onReceive广播接收操作和执行广播finish操作  
(4.2)取消超时广播  
(4.3)将当前广播添加到历史广播列表  
(4.4)若当前广播为隐式广播, 则记录隐式广播  
(4.5)当前广播处理完, mCurrentBroadcast置空, continue并执行下一轮do while循环  
(5)如果下一个广播是延迟广播, 则针对mSplitRefcounts做相关处理, 同时将延迟广播添加到延迟广播列表  
4.将步骤3的有序或延时广播r发送给当前索引nextReceiver的广播接收器处理(filter非空)：  
(1)获取广播r的当前广播接收器BroadcastFilter的索引nextReceiver. 并通过Handler超时机制发送消息, 便于杀死超时广播  
(2)获取索引nextReceiver的广播接收器BroadcastFilter:  
(2.1)调用deliverToRegisteredReceiverLocked将当前广播发送给广播接收器BroadcastFilter执行  
(2.2)若当前广播处理完, 设置广播状态为IDLE, 并调用scheduleBroadcastsLocked继续处理下一个广播并return  
5.若filter为空, 根据nextReceiver得到ResolveInfo且非空:  
(1)先执行各种检查包括：targetSdkVersion, Appop操作权限, 防火墙, 权限review, 用户运行状态, 后台执行检查等.  
(2)如果上述条件不符合, 则设置广播接收器状态为DELIVERY_SKIPPED, 设置广播状态为IDLE, 调用scheduleBroadcastsLocked处理下一个广播并return  
(3)根据ResolveInfo构造ComponentName对象设置给当前广播, 设置广播接收器状态为DELIVERY_DELIVERED, 设置广播状态为APP_RECEIVE  
(4)若当前进程活着, 则处理当前广播  
(5)若当前进程未启动, 则先启动进程, 若进程启动失败, 则finishReceiverLocked结束当前广播接收操作, 设置广播状态为IDLE, 并调用scheduleBroadcastsLocked执行下一个广播接收操作  

**广播处理过程总结：**  
1.无序广播处理逻辑：一次性将所有无序广播BroadcastRecord发给他们的所有广播接收器BroadcastFilter处理  
2.延时或有序广播处理：每次取一个广播BroadcastRecord及当前索引nextReceiver的广播接收器BroadcastFilter处理, 到下一次调用processNextBroadcast再获取下一个索引的广播接收器, 是串行执行的.  


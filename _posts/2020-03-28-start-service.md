---
layout: post
comments: true
title: "startService过程"
description: "startService过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析startService过程.

## 0.文件结构

	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
	frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java
	frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java

	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/Service.java

## 1.时序图

TODO

## 2.ContextImpl.startService
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

	@Override
	public ComponentName startService(Intent service) {
	    warnIfCallingFromSystemProcess();
	    // requireForeground=false表示调用startService启动非前台服务, 区别于startForegroundService
	    return startServiceCommon(service, false, mUser);
	}
 
	private ComponentName startServiceCommon(Intent service, boolean requireForeground, UserHandle user) {
	    validateServiceIntent(service);
	    service.prepareToLeaveProcess(this);
	    ComponentName cn = ActivityManager.getService().startService(
	        mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
	                    getContentResolver()), requireForeground,
	                    getOpPackageName(), user.getIdentifier());
	    return cn;
	}

注意：  
startService和startForegroundService差别是入参requireForeground的值，其他代码逻辑完全复用.

## 3.ActivityManagerService.startService
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	@Override
	public ComponentName startService(IApplicationThread caller, Intent service,
	        String resolvedType, boolean requireForeground, String callingPackage, int userId)
	        throws TransactionTooLargeException {
	    enforceNotIsolatedCaller("startService");
	    synchronized(this) {
	        final int callingPid = Binder.getCallingPid();
	        final int callingUid = Binder.getCallingUid();
	        final long origId = Binder.clearCallingIdentity();
	        ComponentName res;
	        try {
	            res = mServices.startServiceLocked(caller, service,
	                    resolvedType, callingPid, callingUid,
	                    requireForeground, callingPackage, userId);
	        } finally {
	            Binder.restoreCallingIdentity(origId);
	        }
	        return res;
	    }
	}

## 4.ActiveServices
[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

### 4.1 startServiceLocked

	ComponentName startServiceLocked(IApplicationThread caller, Intent service,
			String resolvedType, int callingPid, int callingUid, boolean fgRequired, 
			String callingPackage, final int userId)
	        throws TransactionTooLargeException {
		// fgRequired为false, 表示startService
		// allowBackgroundActivityStarts为false, 表示不允许后台Activity启动Service
		return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, 
			    fgRequired, callingPackage, userId, false);
	}

继续看该方法实现：

	ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
	        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
	        final int userId, boolean allowBackgroundActivityStarts)
	        throws TransactionTooLargeException {
	    final boolean callerFg;
	    if (caller != null) {
	        // 从mLruProcesses列表中找出指定进程
	        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
	        callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
	    } else {
	        callerFg = true;
	    }
 
	    // 各种权限检查，并从ServiceMap中根据ComponentName查找ServiceRecord，并封装成ServiceLookupResult
	    // 参考【第4.2节】
	    ServiceLookupResult res = retrieveServiceLocked(service, null, resolvedType,
			    callingPackage, callingPid, callingUid, userId, true, callerFg, false, false);
 
	    // 获取ServiceRecord
	    ServiceRecord r = res.record;
 
	    // If we're starting indirectly (e.g. from PendingIntent), figure out whether
	    // we're launching into an app in a background state.  This keys off of the same
	    // idleness state tracking as e.g. O+ background service start policy.
	    // 是否后台启动, Android 8.0+有后台启动限制
	    final boolean bgLaunch = !mAm.isUidActiveLocked(r.appInfo.uid);
 
	    // If the app has strict background restrictions, we treat any bg service
	    // start analogously to the legacy-app forced-restrictions case, regardless
	    // of its target SDK version.
	    // 如果后台启动Service，且app有后台启动限制，则forcedStandby设置为true
	    boolean forcedStandby = false;
	    if (bgLaunch && appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
	        forcedStandby = true;
	    }
 
	    // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
	    // 如果是startForegroundService启动前台服务，检查前台启动操作权限是否允许
	    boolean forceSilentAbort = false;
	    if (fgRequired) {
	        final int mode = mAm.mAppOpsService.checkOperation(
	                AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
	        switch (mode) {
	            case AppOpsManager.MODE_ALLOWED:
	            case AppOpsManager.MODE_DEFAULT:
	                // All okay.
	                break;
	            case AppOpsManager.MODE_IGNORED:
	                // Not allowed, fall back to normal start service, failing siliently
	                // if background check restricts that.
	                fgRequired = false;
	                forceSilentAbort = true;
	                break;
	        }
	    }
 
	    // If this isn't a direct-to-foreground start, check our ability to kick off an
	    // arbitrary service
	    // 如果是后台启动或者启动非前台服务，检查是否允许后台启动，如果不能的话返回null
	    if (forcedStandby || (!r.startRequested && !fgRequired)) {
	        // Before going further -- if this app is not allowed to start services in the
	        // background, then at this point we aren't going to let it period.
	        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
	                r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
	        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
	            if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
	                // In this case we are silently disabling the app, to disrupt as
	                // little as possible existing apps.
	                return null;
	            }
	            if (forcedStandby) {
	                // This is an O+ app, but we might be here because the user has placed
	                // it under strict background restrictions.  Don't punish the app if it's
	                // trying to do the right thing but we're denying it for that reason.
	                if (fgRequired) {
	                    return null;
	                }
	            }
	            // This app knows it is in the new model where this operation is not
	            // allowed, so tell it what has happened.
	            UidRecord uidRec = mAm.mProcessList.getUidRecordLocked(r.appInfo.uid);
	            return new ComponentName("?", "app is in background uid " + uidRec);
	        }
	    }
 
	    // At this point we've applied allowed-to-start policy based on whether this was
	    // an ordinary startService() or a startForegroundService().  Now, only require that
	    // the app follow through on the startForegroundService() -> startForeground()
	    // contract if it actually targets O+.
	    // Android 8.0以下启动前台服务不需要调用startForegroundService(), 即fgRequired为false
	    if (r.appInfo.targetSdkVersion < Build.VERSION_CODES.O && fgRequired) {
	        fgRequired = false;
	    }
 
	    NeededUriGrants neededGrants = mAm.mUgmInternal.checkGrantUriPermissionFromIntent(
	            callingUid, r.packageName, service, service.getFlags(), null, r.userId);
 
	    // 权限检查permission review
	    if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
	            callingUid, service, callerFg, userId)) {
	        return null;
	    }
 
	    if (unscheduleServiceRestartLocked(r, callingUid, false)) {
	        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
	    }
 
	    // 设置ServiceRecord启动参数, pendingStarts中添加启动项StartItem
	    r.lastActivity = SystemClock.uptimeMillis();
	    r.startRequested = true;
	    r.delayedStop = false;
	    r.fgRequired = fgRequired;
	    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(), service, neededGrants, callingUid));
 
	    if (fgRequired) {
	        // 如果是Android 8.0+调用的startForegroundService()启动前台服务的话, 更新ServiceState
	        // We are now effectively running a foreground service.
	        ServiceState stracker = r.getTracker();
	        if (stracker != null) {
	            stracker.setForeground(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
	        }
	        mAm.mAppOpsService.startOperation(AppOpsManager.getToken(mAm.mAppOpsService),
	                AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, true);
	    }
 
	    // 获取ServiceMap. 参考【第4.3节】
	    final ServiceMap smap = getServiceMapLocked(r.userId);
	    boolean addToStarting = false;
	    // 如果是调用startService且Service还没启动，且mStartedUsers列表包含userId
	    // 则更新addToStarting状态位为true
	    if (!callerFg && !fgRequired && r.app == null && mAm.mUserController.hasStartedUserState(r.userId)) {
	        // 从进程列表中获取启动Service所在的进程
	        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
	        if (proc == null || proc.getCurProcState() > ActivityManager.PROCESS_STATE_RECEIVER) {
	            // If this is not coming from a foreground caller, then we may want
	            // to delay the start if there are already other background services
	            // that are starting.  This is to avoid process start spam when lots
	            // of applications are all handling things like connectivity broadcasts.
	            // We only do this for cached processes, because otherwise an application
	            // can have assumptions about calling startService() for a service to run
	            // in its own process, and for that process to not be killed before the
	            // service is started.  This is especially the case for receivers, which
	            // may start a service in onReceive() to do some additional work and have
	            // initialized some global state as part of that.
	            // 如果delayed为true，延迟启动
	            if (r.delayed) {
	                // This service is already scheduled for a delayed start; just leave
	                // it still waiting.
	                return r.name;
	            }
	            // 如果后台启动服务数超过mMaxStartingBackground，则延迟启动
	            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
	                // Something else is starting, delay!
	                smap.mDelayedStartList.add(r);
	                r.delayed = true;
	                return r.name;
	            }
	            // addToStarting设置为true
	            addToStarting = true;
	        } else if (proc.getCurProcState() >= ActivityManager.PROCESS_STATE_SERVICE) {
	            // We slightly loosen when we will enqueue this new service as a background
	            // starting service we are waiting for, to also include processes that are
	            // currently running other services or receivers.
	            addToStarting = true;
	        }
	    }
 
	    // Called when the service is started with allowBackgroundActivityStarts set.
	    if (allowBackgroundActivityStarts) {
	        r.whitelistBgActivityStartsOnServiceStart();
	    }
 
		// 启动服务. 参考【第4.4节】
	    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
	    return cmp;
	}

### 4.2 retrieveServiceLocked

	private ServiceLookupResult retrieveServiceLocked(Intent service,
	        String instanceName, String resolvedType, String callingPackage,
	        int callingPid, int callingUid, int userId,
	        boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
	        boolean allowInstant) {
	    ServiceRecord r = null;
	    // 获取userId
	    userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
	            ActivityManagerInternal.ALLOW_NON_FULL_IN_PROFILE, "service", callingPackage);
 
	    // 根据userId获取ServiceMap. 参考【第4.3节】
	    ServiceMap smap = getServiceMapLocked(userId);
 
	    // 获取Component
	    final ComponentName comp;
	    if (instanceName == null) {
	        comp = service.getComponent();
	    } else {
	        final ComponentName realComp = service.getComponent();
	        comp = new ComponentName(realComp.getPackageName(), realComp.getClassName() + ":" + instanceName);
	    }
	    // 获取ServiceRecord
	    if (comp != null) {
	        r = smap.mServicesByInstanceName.get(comp);
	    }
	    if (r == null && !isBindExternal && instanceName == null) {
	        Intent.FilterComparison filter = new Intent.FilterComparison(service);
	        r = smap.mServicesByIntent.get(filter);
	    }
	    if (r != null && (r.serviceInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0
	            && !callingPackage.equals(r.packageName)) {
	        // If an external service is running within its own package, other packages should not bind to that instance.
	        r = null;
	    }
 
	    if (r == null) {
	        try {
	            int flags = ActivityManagerService.STOCK_PM_FLAGS | PackageManager.MATCH_DEBUG_TRIAGED_MISSING;
	            if (allowInstant) {
	                flags |= PackageManager.MATCH_INSTANT;
	            }
	            // 获取ResolveInfo
	            ResolveInfo rInfo = mAm.getPackageManagerInternalLocked().resolveService(service, resolvedType, flags, userId, callingUid);
	            ServiceInfo sInfo = rInfo != null ? rInfo.serviceInfo : null;
	            ...
	            ComponentName className = new ComponentName(sInfo.applicationInfo.packageName, sInfo.name);
	            ComponentName name = comp != null ? comp : className;
	            // 检查调起方和被调起方, 不同uid下的不同package能否建立联系. 默认情况下只要uid之一是SYSTEM_UID，都是允许建立练习的
	            if (!mAm.validateAssociationAllowedLocked(callingPackage, callingUid,name.getPackageName(), sInfo.applicationInfo.uid)) {
	                String msg = "association not allowed between packages " + callingPackage + " and " + name.getPackageName();
	                // Service查找失败, 返回null
	                return new ServiceLookupResult(null, msg);
	            }
 
	            // Store the defining packageName and uid, as they might be changed in
	            // the ApplicationInfo for external services (which run with the package name
	            // and uid of the caller).
	            // 这段代码检查是否允许Service运行在调用方的package和uid中
	            String definingPackageName = sInfo.applicationInfo.packageName;
	            int definingUid = sInfo.applicationInfo.uid;
	            if ((sInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0) {
	                if (isBindExternal) {
	                    // 各种检查：exported和flags & ServiceInfo.FLAG_ISOLATED_PROCESS
	                    ...
	                    // Run the service under the calling package's application.
	                    ApplicationInfo aInfo = AppGlobals.getPackageManager().getApplicationInfo(
	                            callingPackage, ActivityManagerService.STOCK_PM_FLAGS, userId);
	                    sInfo = new ServiceInfo(sInfo);
	                    sInfo.applicationInfo = new ApplicationInfo(sInfo.applicationInfo);
	                    sInfo.applicationInfo.packageName = aInfo.packageName;
	                    sInfo.applicationInfo.uid = aInfo.uid;
	                    name = new ComponentName(aInfo.packageName, name.getClassName());
	                    className = new ComponentName(aInfo.packageName,
	                            instanceName == null ? className.getClassName()
                                    : (className.getClassName() + ":" + instanceName));
	                    service.setComponent(name);
	                } else {
	                    throw new SecurityException("BIND_EXTERNAL_SERVICE required for " + name);
	                }
	            }
 
	            if (userId > 0) {
	                // Checks to see if the caller is in the same app as the singleton component, or the component is in a special app.
	                // 检查调用方和被调用方是否在同一个app中, 或者被调用方为SYSTEM_UID或PHONE_UID
	                if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo, sInfo.name, sInfo.flags)
	                        && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
	                    userId = 0;
	                    smap = getServiceMapLocked(0);
	                }
	                sInfo = new ServiceInfo(sInfo);
	                sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
	            }
	            // 从map中查找
	            r = smap.mServicesByInstanceName.get(name);
	            if (r == null && createIfNeeded) {
	                final Intent.FilterComparison filter = new Intent.FilterComparison(service.cloneFilter());
	                final ServiceRestarter res = new ServiceRestarter();
	                ...
	                // 创建ServiceRecord，并添加到Map中
	                r = new ServiceRecord(mAm, ss, className, name, definingPackageName, definingUid, filter, sInfo, callingFromFg, res);
	                res.setService(r);
	                smap.mServicesByInstanceName.put(name, r);
	                smap.mServicesByIntent.put(filter, r);
 
	                // 从pending列表移除该ServiceRecord
	                for (int i=mPendingServices.size()-1; i>=0; i--) {
	                    final ServiceRecord pr = mPendingServices.get(i);
	                    if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid && pr.instanceName.equals(name)) {
	                        mPendingServices.remove(i);
	                    }
	                }
	            }
	        } catch (RemoteException ex) {
	            // pm is in same process, this will never happen.
	        }
	    }
	    if (r != null) {
	        // 各种检查: 包括防火墙，权限等
	        ...
	        return new ServiceLookupResult(r, null);
	    }
	    return null;
	}

### 4.3 getServiceMapLocked

	private ServiceMap getServiceMapLocked(int callingUser) {
	    ServiceMap smap = mServiceMap.get(callingUser);
	    if (smap == null) {
	        smap = new ServiceMap(mAm.mHandler.getLooper(), callingUser);
	        mServiceMap.put(callingUser, smap);
	    }
	    return smap;
	}

### 4.4 startServiceInnerLocked

	ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
	        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
		// 更新ServiceState状态为started
	    ServiceState stracker = r.getTracker();
	    if (stracker != null) {
	        stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
	    }
	    r.callStart = false;
	    synchronized (r.stats.getBatteryStats()) {
	        r.stats.startRunningLocked();
	    }
	    // 拉起服务. 参考【第4.5节】
	    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
	    if (error != null) {
	        return new ComponentName("!!", error);
	    }
 
	    if (r.startRequested && addToStarting) {
	        boolean first = smap.mStartingBackground.size() == 0;
	        smap.mStartingBackground.add(r);
	        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
	        if (first) {
	            smap.rescheduleDelayedStartsLocked();
	        }
	    } else if (callerFg || r.fgRequired) {
	        smap.ensureNotStartingBackgroundLocked(r);
	    }
 
	    return r.name;
	}

### 4.5 bringUpServiceLocked

	private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
	        boolean whileRestarting, boolean permissionsReviewRequired)
	        throws TransactionTooLargeException {
	    if (r.app != null && r.app.thread != null) {
	        // 如果Service已经创建，则发送启动服务消息
	        sendServiceArgsLocked(r, execInFg, false);
	        return null;
	    }
 
	    if (!whileRestarting && mRestartingServices.contains(r)) {
	        // If waiting for a restart, then do nothing.
	        return null;
	    }
 
	    // We are now bringing the service up, so no longer in the restarting state.
	    if (mRestartingServices.remove(r)) {
	        clearRestartingIfNeededLocked(r);
	    }
 
	    // Make sure this service is no longer considered delayed, we are starting it now.
	    if (r.delayed) {
	        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
	        r.delayed = false;
	    }
 
	    // Make sure that the user who owns this service is started.  If not,
	    // we don't want to allow it to run.
	    if (!mAm.mUserController.hasStartedUserState(r.userId)) {
	        bringDownServiceLocked(r);
	        return msg;
	    }
 
	    // Service is now being launched, its package can't be stopped.
	    AppGlobals.getPackageManager().setPackageStoppedState(r.packageName, false, r.userId);
 
	    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
	    final String procName = r.processName;
	    HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
	    ProcessRecord app;
 
	    if (!isolated) { // 非isolated进程
	        // 从进程列表获取进程
	        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
	        if (app != null && app.thread != null) {
	            app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
	            // 真正启动服务. 参考【第4.6节】
	            realStartServiceLocked(r, app, execInFg);
	            return null;
 
	            // If a dead object exception was thrown -- fall through to restart the application.
	        }
	    } else { // isolated进程
	        // If this service runs in an isolated process, then each time
	        // we call startProcessLocked() we will get a new isolated
	        // process, starting another process if we are currently waiting
	        // for a previous process to come up.  To deal with this, we store
	        // in the service any current isolated process it is running in or
	        // waiting to have come up.
	        app = r.isolatedProc;
	        if (WebViewZygote.isMultiprocessEnabled() && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
	            hostingRecord = HostingRecord.byWebviewZygote(r.instanceName);
	        }
	        if ((r.serviceInfo.flags & ServiceInfo.FLAG_USE_APP_ZYGOTE) != 0) {
	            hostingRecord = HostingRecord.byAppZygote(r.instanceName, r.definingPackageName, r.definingUid);
	        }
	    }
 
	    // 如果进程为空，则启动进程并将ServiceRecord添加到服务队列中等待执行
	    if (app == null && !permissionsReviewRequired) {
	        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags, hostingRecord, false, isolated, false)) == null) {
	            bringDownServiceLocked(r);
	            return msg;
	        }
	        if (isolated) {
	            r.isolatedProc = app;
	        }
	    }
 
	    if (r.fgRequired) {
	        mAm.tempWhitelistUidLocked(r.appInfo.uid, SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
	    }
 
	    // mPendingServices列表中添加服务
	    if (!mPendingServices.contains(r)) {
	        mPendingServices.add(r);
	    }
 
	    if (r.delayedStop) {
	        // Oh and hey we've already been asked to stop!
	        r.delayedStop = false;
	        if (r.startRequested) {
	            stopServiceLocked(r);
	        }
	    }
 
	    return null;
	}

### 4.6 realStartServiceLocked

	private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
	    // 设置进程
	    r.setProcess(app);
	    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
 
	    // services列表中添加服务
	    final boolean newService = app.services.add(r);
	    // 更新ServiceRecord create阶段各属性. 参考【第4.8节】
	    bumpServiceExecutingLocked(r, execInFg, "create");
	    // 更新lru进程信息
	    mAm.updateLruProcessLocked(app, false, null);
	    // 更新进程中前台服务状态等信息
	    updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
	    // 更新oomAdj
	    mAm.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
 
	    boolean created = false;
	    synchronized (r.stats.getBatteryStats()) {
	        r.stats.startLaunchedLocked();
	    }
	    // Updates a package last used time.
	    mAm.notifyPackageUse(r.serviceInfo.packageName, PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
	    // 强制更新进程状态为PROCESS_STATE_SERVICE
	    app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
	    // 创建服务. 参考【5.1节】
	    app.thread.scheduleCreateService(r, r.serviceInfo,
	           mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
	           app.getReportedProcState());
	    r.postNotification();
	    created = true;
 
	    if (r.whitelistManager) {
	        app.whitelistManager = true;
	    }
 
		// 下面这段代码是bindService相关逻辑，此处只分析startService
	    requestServiceBindingsLocked(r, execInFg);
	    updateServiceClientActivitiesLocked(app, null, true);
	    if (newService && created) {
	        app.addBoundClientUidsOfNewService(r);
	    }
 
	    // If the service is in the started state, and there are no
	    // pending arguments, then fake up one so its onStartCommand() will be called.
	    // 如果pendingStarts列表为空，则将该服务对应的StartItem添加到该列表中
	    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
	        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(), null, null, 0));
	    }
 
	    // 发送启动服务消息. 参考【第4.7节】
	    sendServiceArgsLocked(r, execInFg, true);
 
	    if (r.delayed) {
	        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
	        r.delayed = false;
	    }
 
	    if (r.delayedStop) {
	        // Oh and hey we've already been asked to stop!
	        r.delayedStop = false;
	        if (r.startRequested) {
	            stopServiceLocked(r);
	        }
	    }
	}

realStartServiceLocked主要做的事情:   
(1) ApplicationThread.scheduleCreateService: 创建Service，并调用onCreate方法  
(2) sendServiceArgsLocked: 启动Service，通过ApplicationThread.scheduleServiceArgs最终调用到onStartCommand

### 4.7 sendServiceArgsLocked

	private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
	        boolean oomAdjusted) throws TransactionTooLargeException {
	    final int N = r.pendingStarts.size();
	    if (N == 0) {
	        return;
	    }
 
	    ArrayList<ServiceStartArgs> args = new ArrayList<>();
 
	    while (r.pendingStarts.size() > 0) {
	        // 从pendingStarts列表移除并取出StartItem
	        ServiceRecord.StartItem si = r.pendingStarts.remove(0);
 
	        // 当intent=null且pendingStarts列表存在多个待启动的Service时，直接跳过
	        if (si.intent == null && N > 1) {
	            continue;
	        }
 
	        // 更新deliver信息
	        si.deliveredTime = SystemClock.uptimeMillis();
	        r.deliveredStarts.add(si);
	        si.deliveryCount++;
	        if (si.neededGrants != null) {
	            mAm.mUgmInternal.grantUriPermissionUncheckedFromIntent(si.neededGrants, si.getUriPermissionsLocked());
	        }
	        mAm.grantEphemeralAccessLocked(r.userId, si.intent, UserHandle.getAppId(r.appInfo.uid), UserHandle.getAppId(si.callingId));
	        // 更新ServiceRecord start阶段各属性. 参考【第4.8节】
	        bumpServiceExecutingLocked(r, execInFg, "start");
	        // 更新oomAdj
	        if (!oomAdjusted) {
	            oomAdjusted = true;
	            mAm.updateOomAdjLocked(r.app, true, OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
	        }
	        if (r.fgRequired && !r.fgWaiting) {
	            if (!r.isForeground) {
	                // 如果启动前台服务，在调用了startForegroundService()后若当前还没有调用startForeground()
	                // 则要检查startForeground()是否在规定时间内调用，超时时间为10s
	                // 如果超时未调用则stopService，同时抛出ANR异常
	                scheduleServiceForegroundTransitionTimeoutLocked(r);
	            } else { // 如果已经调用了startForeground(), 则重置状态
	                r.fgRequired = false;
	            }
	        }
	        int flags = 0;
	        if (si.deliveryCount > 1) {
	            flags |= Service.START_FLAG_RETRY;
	        }
	        if (si.doneExecutingCount > 0) {
	            flags |= Service.START_FLAG_REDELIVERY;
	        }
	        // args列表中添加服务启动参数
	        args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
	    }
 
	    ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
	    // 设置列表最大数为4
	    slice.setInlineCountLimit(4);
	    Exception caughtException = null;
	    try {
	        // 启动Service.onStartCommand. 参考【第5.2节】
	        r.app.thread.scheduleServiceArgs(r, slice);
	    } catch (TransactionTooLargeException e) {
	        caughtException = e;
	    } catch (RemoteException e) {
	        caughtException = e;
	    } catch (Exception e) {
	        caughtException = e;
	    }
 
	    // 异常时，做一些数据清理工作
	    if (caughtException != null) {
	        // Keep nesting count correct
	        final boolean inDestroying = mDestroyingServices.contains(r);
	        for (int i = 0; i < args.size(); i++) {
	            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
	        }
	        if (caughtException instanceof TransactionTooLargeException) {
	            throw (TransactionTooLargeException)caughtException;
	        }
	    }
	}

### 4.8 bumpServiceExecutingLocked

	private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
	    // Services within the system server won't start until SystemServer
	    // does Looper.loop(), so we shouldn't try to start/bind to them too early in the boot
	    // process. However, since there's a little point of showing the ANR dialog in that case,
	    // let's suppress the timeout until PHASE_THIRD_PARTY_APPS_CAN_START.
 
	    // Too early to start/bind service in system_server: Phase < SystemService.PHASE_THIRD_PARTY_APPS_CAN_START
	    // 在Phase < PHASE_THIRD_PARTY_APPS_CAN_START之前，不检查服务启动超时
	    boolean timeoutNeeded = true;
	    if ((mAm.mBootPhase < SystemService.PHASE_THIRD_PARTY_APPS_CAN_START)
	            && (r.app != null) && (r.app.pid == android.os.Process.myPid())) {
	        timeoutNeeded = false;
	    }
 
	    // 设置ServiceRecord各属性
	    long now = SystemClock.uptimeMillis();
	    if (r.executeNesting == 0) {
	        r.executeFg = fg;
	        // 设置ServiceState状态为executing
	        ServiceState stracker = r.getTracker();
	        if (stracker != null) {
	            stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
	        }
		    // 添加到executingServices列表
	        if (r.app != null) {
	            r.app.executingServices.add(r);
	            r.app.execServicesFg |= fg;
	            if (timeoutNeeded && r.app.executingServices.size() == 1) {
		            // 检查服务执行时间是否超时，超时的话会抛出ANR异常.参考【第7.1节】
	                scheduleServiceTimeoutLocked(r.app);
	            }
	        }
	    } else if (r.app != null && fg && !r.app.execServicesFg) {
	        r.app.execServicesFg = true;
	        if (timeoutNeeded) {
		        // 检查服务执行时间是否超时，超时的话会抛出ANR异常.参考【第7.1节】
	            scheduleServiceTimeoutLocked(r.app);
	        }
	    }
	    r.executeFg |= fg;
	    r.executeNesting++;
	    r.executingStart = now;
	}

## 5.ApplicationThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 5.1 scheduleCreateService

    public final void scheduleCreateService(IBinder token, ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
        updateProcessState(processState, false);
        CreateServiceData s = new CreateServiceData();
        s.token = token;
        s.info = info;
        s.compatInfo = compatInfo;
 
        sendMessage(H.CREATE_SERVICE, s);
    }

### 5.2 scheduleServiceArgs

    // 遍历args列表并依次将每一项参数封装到ServiceArgsData，然后向H中发送消息并调用onStartCommand方法
    public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
        List<ServiceStartArgs> list = args.getList();
        for (int i = 0; i < list.size(); i++) {
            ServiceStartArgs ssa = list.get(i);
            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = ssa.taskRemoved;
            s.startId = ssa.startId;
            s.flags = ssa.flags;
            s.args = ssa.args;
            sendMessage(H.SERVICE_ARGS, s);
        }
    }

### 5.3 H - CREATE_SERVICE/SERVICE_ARGS

	class H extends Handler {
	    ...
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case CREATE_SERVICE:
	                handleCreateService((CreateServiceData)msg.obj);
	                break;
	            case SERVICE_ARGS:
	                handleServiceArgs((ServiceArgsData)msg.obj);
	                break;
	            ...
	        }
	    }
	}

### 5.4 handleCreateService

	private void handleCreateService(CreateServiceData data) {
	    // If we are getting ready to gc after going to the background, well we are back active so skip it.
	    // 服务创建过程不执行GC操作
	    unscheduleGcIdler();
 
	    // 反射机制创建Service实例
	    LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);
	    Service service = null;
	    java.lang.ClassLoader cl = packageInfo.getClassLoader();
	    service = packageInfo.getAppFactory().instantiateService(cl, data.info.name, data.intent);
 
	    // 创建ApplicationContext
	    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
	    context.setOuterContext(service);
 
	    // 返回或新建Application
	    Application app = packageInfo.makeApplication(false, mInstrumentation);
	    // attach服务到进程
	    service.attach(context, this, data.info.name, data.token, app, ActivityManager.getService());
	    // 调用Service.onCreate
	    service.onCreate();
	    // 服务添加到mServices列表
	    mServices.put(data.token, service);
 
	    // 做一些服务创建后的状态更新工作
	    ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	}

### 5.5 handleServiceArgs

	private void handleServiceArgs(ServiceArgsData data) {
	    Service s = mServices.get(data.token);
	    if (s != null) {
	        if (data.args != null) {
	            data.args.setExtrasClassLoader(s.getClassLoader());
	            data.args.prepareToEnterProcess();
	        }
	        int res;
	        if (!data.taskRemoved) {
	            // 调用onStartCommand
	            res = s.onStartCommand(data.args, data.flags, data.startId);
	        } else {
	            s.onTaskRemoved(data.args);
	            res = Service.START_TASK_REMOVED_COMPLETE;
	        }
 
	        // 等待执行完成
	        QueuedWork.waitToFinish();
 
	        // 将服务从deliver列表移除，并设置服务stopIfKilled属性, 以及做一些其他列表移除工作
	        ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
	    }
	}

## 6.ActivityManagerService
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

### 6.1 serviceDoneExecuting

	public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
	    synchronized(this) {
	        mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
	    }
	}

### 6.2 ActiveServices.serviceDoneExecutingLocked
[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

这段代码ServiceRecord的stopIfKilled解释：参考【第7.3节】

	void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
	    boolean inDestroying = mDestroyingServices.contains(r);
	    if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
	        // 若为启动服务，从deliveredStarts中查找指定服务，并设置服务stopIfKilled属性
	        r.callStart = true;
	        switch (res) {
	            case Service.START_STICKY_COMPATIBILITY:
	            case Service.START_STICKY: {
	                // We are done with the associated start arguments.
	                r.findDeliveredStart(startId, false, true);
	                // Don't stop if killed.
	                r.stopIfKilled = false;
	                break;
	            }
	            case Service.START_NOT_STICKY: {
	                // We are done with the associated start arguments.
	                r.findDeliveredStart(startId, false, true);
	                if (r.getLastStartId() == startId) {
	                    // There is no more work, and this service doesn't want to hang around if killed.
	                    r.stopIfKilled = true;
	                }
	                break;
	            }
	            case Service.START_REDELIVER_INTENT: {
	                // We'll keep this item until they explicitly call stop for it, but keep track of the fact that it was delivered.
	                ServiceRecord.StartItem si = r.findDeliveredStart(startId, false, false);
	                if (si != null) {
	                    si.deliveryCount = 0;
	                    si.doneExecutingCount++;
	                    // Don't stop if killed.
	                    r.stopIfKilled = true;
	                }
	                break;
	            }
	            case Service.START_TASK_REMOVED_COMPLETE: {
	                // Special processing for onTaskRemoved(). Don't impact normal onStartCommand() processing.
	                r.findDeliveredStart(startId, true, true);
	                break;
	            }
	        }
	        if (res == Service.START_STICKY_COMPATIBILITY) {
	            r.callStart = false;
	        }
	    } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
	        // destroy服务
	        ...
	    }
 
	    // 做一些Service清理工作，具体实现看下面代码       
	    serviceDoneExecutingLocked(r, inDestroying, inDestroying);
	}

继续看下面代码:  

	private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying, boolean finishing) {
	    r.executeNesting--;
	    if (r.executeNesting <= 0) {
	        if (r.app != null) {
	            r.app.execServicesFg = false;
	            r.app.executingServices.remove(r);
	            if (r.app.executingServices.size() == 0) {
	                // 如果当前没有正在执行的服务，则移除SERVICE_TIMEOUT_MSG消息
	                mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
	            } else if (r.executeFg) {
	                // Need to re-evaluate whether the app still needs to be in the foreground.
	                for (int i=r.app.executingServices.size()-1; i>=0; i--) {
	                    if (r.app.executingServices.valueAt(i).executeFg) {
	                        r.app.execServicesFg = true;
	                        break;
	                    }
	                }
	            }
	            if (inDestroying) { // 从mDestroyingServices列表移除服务
	                mDestroyingServices.remove(r);
	                r.bindings.clear();
	            }
	            // 更新oomAdj
	            mAm.updateOomAdjLocked(r.app, true, OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
	        }
	        // 重置前台执行标识位为false
	        r.executeFg = false;
	        if (r.tracker != null) {
	            final int memFactor = mAm.mProcessStats.getMemFactorLocked();
	            final long now = SystemClock.uptimeMillis();
	            // 更新ServiceState状态为非执行中和非前台
	            r.tracker.setExecuting(false, memFactor, now);
	            r.tracker.setForeground(false, memFactor, now);
	            if (finishing) {
	                r.tracker.clearCurrentOwner(r, false);
	                r.tracker = null;
	            }
	        }
	        if (finishing) {
	            // 从services列表移除服务
	            if (r.app != null && !r.app.isPersistent()) {
	                r.app.services.remove(r);
	                r.app.updateBoundClientUids();
	                if (r.whitelistManager) {
	                    updateWhitelistManagerLocked(r.app);
	                }
	            }
	            // process置空
	            r.setProcess(null);
	        }
	    }
	}

### 6.3 ServiceRecord.findDeliveredStart
[ -> frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java ]

	// 根据属性id和taskRemoved从deliveredStarts列表中匹配Service, 并根据remove标识决定要不要从列表移除
	public StartItem findDeliveredStart(int id, boolean taskRemoved, boolean remove) {
	    final int N = deliveredStarts.size();
	    for (int i=0; i<N; i++) {
	        StartItem si = deliveredStarts.get(i);
	        if (si.id == id && si.taskRemoved == taskRemoved) {
	            if (remove) deliveredStarts.remove(i);
	            return si;
	        }
	    }
 
	    return null;
	}

## 7.关注点

### 7.1 Service执行时间限制

前台服务执行时间限制是20秒，后台服务执行时间限制是200秒，否则超时会报ANR异常. 具体代码体现如下：  

**(1) ActiveServices.scheduleServiceTimeoutLocked**

    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
    
    // How long we wait for a service to finish executing.
    static final int SERVICE_TIMEOUT = 20*1000;
    static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;

**(2) MainHandler.SERVICE_TIMEOUT_MSG**

	final class MainHandler extends Handler {
	    public MainHandler(Looper looper) {
	        super(looper, null, true);
	    }
 
	    @Override
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	        ...
	        case SERVICE_TIMEOUT_MSG: {
	            mServices.serviceTimeout((ProcessRecord)msg.obj);
	        } break;
	        case SERVICE_FOREGROUND_TIMEOUT_MSG: {
	            mServices.serviceForegroundTimeout((ServiceRecord)msg.obj);
	        } break;
	        ...
	    }
	}

**(3) ActiveServices.serviceTimeout**

    void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;
        synchronized(mAm) {
            if (proc.isDebugging()) {
                // The app's being debugged, ignore timeout.
                return;
            }
            if (proc.executingServices.size() == 0 || proc.thread == null) {
                return;
            }
            final long now = SystemClock.uptimeMillis();
            final long maxTime =  now -
                    (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }
            if (timeout != null && mAm.mProcessList.mLruProcesses.contains(proc)) {
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortInstanceName;
            } else {
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        if (anrMessage != null) {
            proc.appNotResponding(null, null, null, null, false, anrMessage);
        }
    }

### 7.2 startService和startForegroundService

[https://developer.android.com/about/versions/oreo/android-8.0-changes](https://developer.android.com/about/versions/oreo/android-8.0-changes)

(1) Android 8.0之前一般通过startService启动后台服务, 当系统内存紧张情况下后台Service可能会被杀死.  
如果希望Service能够长时间运行而不被杀死, 可以在通过startService启动服务后, 在Service的onStartCommand方法中调用startForeground(id, notification)将服务设置为前台服务. 比如一般的音乐播放器都是这种做法.

(2) Android 8.0之后出于资源考虑对Service做了一些限制. 比如不允许后台调用startService, 否则会报错. 启动前台服务方式变成调用startForegroundService, 之后再在Service的onStartCommand方法中调用startForeground(id, notification)将服务设置为前台服务. 

(3) startService和startForegroundService共有一套代码实现, 都是调用startServiceCommon, 差别就是入参requireForeground前者为false后者为true.  

	// frameworks/base/core/java/android/app/ContextImpl.java
	@Override
	public ComponentName startService(Intent service) {
	    warnIfCallingFromSystemProcess();
	    return startServiceCommon(service, false, mUser);
	}
 
	@Override
	public ComponentName startForegroundService(Intent service) {
	    warnIfCallingFromSystemProcess();
	    return startServiceCommon(service, true, mUser);
	}
 
(4) Android 8.0之后启动前台服务, 会检查调用startForegroundService方法后, 是否在规定时间内调用了startForeground. 这个超时时间是10秒, 如果超时未调用则会stopService，同时抛出ANR异常. 具体代码体现如下：  

**4.1 ActiveServices.scheduleServiceForegroundTransitionTimeoutLocked**

[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

	void scheduleServiceForegroundTransitionTimeoutLocked(ServiceRecord r) {
	    ...
	    Message msg = mAm.mHandler.obtainMessage(ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG);
	    msg.obj = r;
	    r.fgWaiting = true;
	    mAm.mHandler.sendMessageDelayed(msg, SERVICE_START_FOREGROUND_TIMEOUT);
	}

**4.2 MainHandler.SERVICE_FOREGROUND_TIMEOUT_MSG**

[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	final class MainHandler extends Handler {
	    public MainHandler(Looper looper) {
	        super(looper, null, true);
	    }
 
	    @Override
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	        ...
	        case SERVICE_TIMEOUT_MSG: {
	            mServices.serviceTimeout((ProcessRecord)msg.obj);
	        } break;
	        case SERVICE_FOREGROUND_TIMEOUT_MSG: {
	            mServices.serviceForegroundTimeout((ServiceRecord)msg.obj);
	        } break;
	        ...
	    }
	}

**4.3 ActiveServices.serviceForegroundTimeout**

[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

	void serviceForegroundTimeout(ServiceRecord r) {
	    ProcessRecord app;
	    synchronized (mAm) {
	        if (!r.fgRequired || r.destroying) {
	            return;
	        }
 
	        app = r.app;
	        if (app != null && app.isDebugging()) {
	            // The app's being debugged; let it ride
	            return;
	        }
 
	        // startForeground()超时，stopService
	        r.fgWaiting = false;
	        stopServiceLocked(r);
	    }
 
	    // ANR调用
	    if (app != null) {
	        app.appNotResponding(null, null, null, null, false,
	                "Context.startForegroundService() did not then call Service.startForeground(): " + r);
	    }
	}

### 7.3 onStartCommand返回值

运行于主线程

[ -> frameworks/base/core/java/android/app/Service.java ]

	public @StartResult int onStartCommand(Intent intent, @StartArgFlags int flags, int startId) {
	    onStart(intent, startId);
	    return mStartCompatibility ? START_STICKY_COMPATIBILITY : START_STICKY;
	}

(1) 入参:  
**intent**:   通过startService传进来的意图. 如果Service重启时为null  
**flags**:  Additional data about this start request    
**startId**:  A unique integer representing this specific request to start  

(2) 返回值:  
描述了Service被杀后，应该怎么继续启动Service.  
具体体现在源码里，会影响ServiceRecord的stopIfKilled属性.  

	// 掩码
	public static final int START_CONTINUATION_MASK = 0xf;
	
	// 进程被杀后，不保证一定调用onStartCommand方法
	public static final int START_STICKY_COMPATIBILITY = 0;
	
	// Service启动后若所在进程被杀，则服务保留started状态但intent不保留
	// 等进程启动后会自动重建Service实例并调用onStartCommand, 且此时intent为null
	// 一般适用于音乐播放器等后台长时间运行场景
	public static final int START_STICKY = 1;
	
	// Service启动后若所在进程被杀，则不会自动重建Service, 需要外部主动调用startService才会新建Service实例
	// 适用场景：适用于当存在内存压力时允许所在进程被杀死，且有机制主动调用startService拉起服务的场景.
	// 如通过Alarm机制每隔N分钟启动一次服务这种场景.
	public static final int START_NOT_STICKY = 2;
	
	// Service启动后若所在进程被杀，则等进程启动后Service会被重建且intent会被继续转发
	// 除非手动调用stopService或intent为null
	public static final int START_REDELIVER_INTENT = 3;

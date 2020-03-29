---
layout: post
comments: true
title: "bindService过程"
description: "bindService过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析bindService过程.

## 0.文件结构

	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
	frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java
	frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java

	frameworks/base/core/java/android/app/LoadedApk.java
	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/Service.java

## 1.时序图

TODO

## 2.ContextImpl
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

### 2.1 bindService

	@Override
	public boolean bindService(Intent service, ServiceConnection conn, int flags) {
	    warnIfCallingFromSystemProcess();
	    return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null, getUser());
	}

### 2.2 bindServiceCommon

	private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
	        String instanceName, Handler handler, Executor executor, UserHandle user) {   
	    // 获取ServiceConnection对象
	    IServiceConnection sd;
	    if (mPackageInfo != null) {
	        if (executor != null) {
	            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
	        } else {
	            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
	        }
	    }
	    // 获取Activity令牌
	    IBinder token = getActivityToken();
	    if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
	            && mPackageInfo.getApplicationInfo().targetSdkVersion
	            < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
	        // 设置flags为不影响Service所在进程的调度和内存管理
	        flags |= BIND_WAIVE_PRIORITY;
	    }
	    service.prepareToLeaveProcess(this);
	    // 绑定isolated服务. 参考【第3节】
	    int res = ActivityManager.getService().bindIsolatedService(
	        mMainThread.getApplicationThread(), getActivityToken(), service,
	        service.resolveTypeIfNeeded(getContentResolver()),
	        sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
	    return res != 0;
	}

## 3.ActivityManagerService.bindIsolatedService
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
	        String resolvedType, IServiceConnection connection, int flags, String instanceName,
	        String callingPackage, int userId) throws TransactionTooLargeException {
	    // isolated进程不能调用
	    enforceNotIsolatedCaller("bindService");
 
	    // instanceName非空的话，进行字符校验. 此处instanceName为空，走不到这里
	    if (instanceName != null) {
	        for (int i = 0; i < instanceName.length(); ++i) {
	            char c = instanceName.charAt(i);
	            if (!((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')
	                        || (c >= '0' && c <= '9') || c == '_' || c == '.')) {
	                throw new IllegalArgumentException("Illegal instanceName");
	            }
	        }
	    }
 
	    synchronized(this) {
	        // 绑定服务. 参考【第4.1节】
	        return mServices.bindServiceLocked(caller, token, service,
	                resolvedType, connection, flags, instanceName, callingPackage, userId);
	    }
	}

## 4.ActiveServices
[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

### 4.1 bindServiceLocked

	int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
	        String resolvedType, final IServiceConnection connection, int flags,
	        String instanceName, String callingPackage, final int userId)
	        throws TransactionTooLargeException {
	    // 从mProcessList列表中获取调用进程
	    final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
 
	    ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
	    if (token != null) {
	        // 获取指定Activity的服务连接持有者ActivityServiceConnectionHolder
	        activity = mAm.mAtmInternal.getServiceConnectionsHolder(token);
	        // 如果holder为空, 直接返回
	        if (activity == null) {
	            return 0;
	        }
	    }
 
	    int clientLabel = 0;
	    PendingIntent clientIntent = null;
	    final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;
 
	    // 如果是系统进程调用, 则需要对intent进行标记
	    if (isCallerSystem) {
	        // Hacky kind of thing -- allow system stuff to tell us
	        // what they are, so we can report this elsewhere for
	        // others to know why certain services are running.
	        service.setDefusable(true);
	        clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);
	        if (clientIntent != null) {
	            clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
	            if (clientLabel != 0) {
	                // There are no useful extras in the intent, trash them.
	                // System code calling with this stuff just needs to know
	                // this will happen.
	                service = service.cloneFilter();
	            }
	        }
	    }
 
	    // flags检查
	    ...
 
	    final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
	    final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;
	    final boolean allowInstant = (flags & Context.BIND_ALLOW_INSTANT) != 0;
 
	    // 各种权限检查，并从ServiceMap中根据ComponentName查找ServiceRecord，并封装成ServiceLookupResult
	    // 参考【第4.2节】
	    ServiceLookupResult res =
	        retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
	                Binder.getCallingPid(), Binder.getCallingUid(), userId, true,
	                callerFg, isBindExternal, allowInstant);
	    ServiceRecord s = res.record;
 
	    // If permissions need a review before any of the app components can run,
	    // we schedule binding to the service but do not start its process, then
	    // we launch a review activity to which is passed a callback to invoke
	    // when done to start the bound service's process to completing the binding.
	    // 这段代码含义：如果在绑定服务之前需要权限检查，则先调起权限检查Activity，检查结果通过RemoteCallback回调
	    // 如果权限检查通过，则调用bringUpServiceLocked真正绑定服务, 否则调用unbindServiceLocked解绑服务
	    // 权限检查需要在前台执行
	    boolean permissionsReviewRequired = false;
	    if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
	            s.packageName, s.userId)) {
	        permissionsReviewRequired = true;
 
	        // Show a permission review UI only for binding from a foreground app
	        if (!callerFg) {
	            return 0;
	        }
 
	        final ServiceRecord serviceRecord = s;
	        final Intent serviceIntent = service;
 
	        RemoteCallback callback = new RemoteCallback(
	                new RemoteCallback.OnResultListener() {
	            @Override
	            public void onResult(Bundle result) {
	                synchronized(mAm) {
	                    final long identity = Binder.clearCallingIdentity();
	                    try {
	                        if (!mPendingServices.contains(serviceRecord)) {
	                            return;
	                        }
	                        // If there is still a pending record, then the service
	                        // binding request is still valid, so hook them up. We
	                        // proceed only if the caller cleared the review requirement
	                        // otherwise we unbind because the user didn't approve.
	                        if (!mAm.getPackageManagerInternalLocked()
	                                .isPermissionsReviewRequired(
	                                        serviceRecord.packageName, serviceRecord.userId)) {
	                            try {
	                                bringUpServiceLocked(serviceRecord,
	                                        serviceIntent.getFlags(), callerFg, false, false);
	                            } catch (RemoteException e) {
	                                /* ignore - local call */
	                            }
	                        } else {
	                            unbindServiceLocked(connection);
	                        }
	                    } finally {
	                        Binder.restoreCallingIdentity(identity);
	                    }
	                }
	            }
	        });
 
	        final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
	        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
	                | Intent.FLAG_ACTIVITY_MULTIPLE_TASK
	                | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
	        intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
	        intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);
 
	        mAm.mHandler.post(new Runnable() {
	            @Override
	            public void run() {
	                mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
	            }
	        });
	    }
 
	    final long origId = Binder.clearCallingIdentity();
 
	    try {
	        // 如果当前正在restart过程中调用bind服务, 则从mRestartingServices列表中移除该服务及重置restart相关属性
	        unscheduleServiceRestartLocked(s, callerApp.info.uid, false);
 
	        // 如果flags为BIND_AUTO_CREATE
	        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
	            s.lastActivity = SystemClock.uptimeMillis();
	            // 如果connections列表中没有flags为BIND_AUTO_CREATE的ConnectionRecord
	            if (!s.hasAutoCreateConnections()) {
	                // 设置ServiceState状态为bound
	                ServiceState stracker = s.getTracker();
	                if (stracker != null) {
	                    stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(), s.lastActivity);
	                }
	            }
	        }
 
	        // Ensures that the given package name has an explicit set of allowed associations.
	        if ((flags & Context.BIND_RESTRICT_ASSOCIATIONS) != 0) {
	            mAm.requireAllowedAssociationsLocked(s.appInfo.packageName);
	        }
 
	        // 根据入参从映射表中查找或创建Association对象，并更新各相关属性
		    // 参考【第4.3节】
	        mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
	                callerApp.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
	                s.instanceName, s.processName);
	        // Once the apps have become associated, if one of them is caller is ephemeral
	        // the target app should now be able to see the calling app
	        mAm.grantEphemeralAccessLocked(callerApp.userId, service,
	                UserHandle.getAppId(s.appInfo.uid), UserHandle.getAppId(callerApp.uid));
 
	        // 从bindings映射表中查找或新建AppBindRecord. 参考【第4.4节】
	        AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
	        // 创建ConnectionRecord
	        ConnectionRecord c = new ConnectionRecord(b, activity,
	                connection, flags, clientLabel, clientIntent,
	                callerApp.uid, callerApp.processName, callingPackage);
 
	        // 建立binder和ConnectionRecord列表的映射关系
	        IBinder binder = connection.asBinder();
	        s.addConnection(binder, c);
	        b.connections.add(c);
	        if (activity != null) {
	            activity.addConnection(c);
	        }
	        b.client.connections.add(c);
	        c.startAssociationIfNeeded();
	        if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
	            b.client.hasAboveClient = true;
	        }
	        if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
	            s.whitelistManager = true;
	        }
	        if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
	            s.setHasBindingWhitelistingBgActivityStarts(true);
	        }
	        // 从当前进程的services服务列表的ConnectionRecord列表中, 查找和其建立连接的Client, 看是否Client有Activity
	        // 并更新该进程状态信息和lru进程列表. 参考【第4.5节】
	        if (s.app != null) {
	            updateServiceClientActivitiesLocked(s.app, c, true);
	        }
	        ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
	        if (clist == null) {
	            clist = new ArrayList<>();
	            mServiceConnections.put(binder, clist);
	        }
	        clist.add(c);
 
	        // 如果flags为BIND_AUTO_CREATE, 真正执行绑定服务并返回. 参考【第4.6节】
	        if ((flags&Context.BIND_AUTO_CREATE) != 0) {
	            s.lastActivity = SystemClock.uptimeMillis();
	            if (bringUpServiceLocked(s, service.getFlags(), callerFg, false, permissionsReviewRequired) != null) {
	                return 0;
	            }
	        }
 
	        // 若为其他flags, 且Service所在的进程非空, 则更新lru进程列表和oomAdj. 这个可能会提高所在进程优先级
	        if (s.app != null) {
	            if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
	                s.app.treatLikeActivity = true;
	            }
	            if (s.whitelistManager) {
	                s.app.whitelistManager = true;
	            }
	            // This could have made the service more important.
	            mAm.updateLruProcessLocked(s.app,
	                   (callerApp.hasActivitiesOrRecentTasks() && s.app.hasClientActivities())
                           || (callerApp.getCurProcState() <= ActivityManager.PROCESS_STATE_TOP
                            && (flags & Context.BIND_TREAT_LIKE_ACTIVITY) != 0), b.client);
	            mAm.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
	        }
 
	        if (s.app != null && b.intent.received) {
	            // Service is already running, so we can immediately publish the connection.
	            // 如果服务正在运行, 则先建立连接
	            c.conn.connected(s.name, b.intent.binder, false);
 
	            // If this is the first app connected back to this binding,
	            // and the service had previously asked to be told when rebound, then do so.
	            // rebind服务
	            if (b.intent.apps.size() == 1 && b.intent.doRebind) {
	                requestServiceBindingLocked(s, b.intent, callerFg, true);
	            }
	        } else if (!b.intent.requested) {
	            // bind服务
	            requestServiceBindingLocked(s, b.intent, callerFg, false);
	        }
 
	        // 从mStartingBackground后台服务列表中移除当前Service, 取消后台启动
	        getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);
	    } finally {
	        Binder.restoreCallingIdentity(origId);
	    }
 
	    return 1;
	}

### 4.2 retrieveServiceLocked

参考文章: **[startService过程](http://mouxuejie.com/blog/2020-03-28/start-service/)**

### 4.3 AMS.startAssociationLocked  
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	// 映射关系Mapping is target uid -> target component -> source uid -> source process name -> association data.
	Association startAssociationLocked(int sourceUid, String sourceProcess,
			int sourceState, int targetUid, long targetVersionCode,
			ComponentName targetComponent, String targetProcess) {
	    if (!mTrackingAssociations) {
	        return null;
	    }
	    ArrayMap<ComponentName, SparseArray<ArrayMap<String, Association>>> components
	            = mAssociations.get(targetUid);
	    if (components == null) {
	        components = new ArrayMap<>();
	        mAssociations.put(targetUid, components);
	    }
	    SparseArray<ArrayMap<String, Association>> sourceUids = components.get(targetComponent);
	    if (sourceUids == null) {
	        sourceUids = new SparseArray<>();
	        components.put(targetComponent, sourceUids);
	    }
	    ArrayMap<String, Association> sourceProcesses = sourceUids.get(sourceUid);
	    if (sourceProcesses == null) {
	        sourceProcesses = new ArrayMap<>();
	        sourceUids.put(sourceUid, sourceProcesses);
	    }
	    Association ass = sourceProcesses.get(sourceProcess);
	    if (ass == null) {
	        ass = new Association(sourceUid, sourceProcess, targetUid, targetComponent, targetProcess);
	        sourceProcesses.put(sourceProcess, ass);
	    }
	    ass.mCount++;
	    ass.mNesting++;
	    if (ass.mNesting == 1) {
	        ass.mStartTime = ass.mLastStateUptime = SystemClock.uptimeMillis();
	        ass.mLastState = sourceState;
	    }
	    return ass;
	}

### 4.4 ServiceRecord.retrieveAppBindingLocked  
[ -> frameworks/base/services/core/java/com/android/server/am/ServiceRecord.java ]

	// 根据intent和app, 从bindings列表中匹配AppBindRecord, 没匹配到的话则新建一个
	public AppBindRecord retrieveAppBindingLocked(Intent intent, ProcessRecord app) {
	    Intent.FilterComparison filter = new Intent.FilterComparison(intent);
	    IntentBindRecord i = bindings.get(filter);
	    if (i == null) {
	        i = new IntentBindRecord(this, filter);
	        bindings.put(filter, i);
	    }
	    AppBindRecord a = i.apps.get(app);
	    if (a != null) {
	        return a;
	    }
	    a = new AppBindRecord(this, i, app);
	    i.apps.put(app, a);
	    return a;
	}

### 4.5 updateServiceClientActivitiesLocked

	private boolean updateServiceClientActivitiesLocked(ProcessRecord proc,
	        ConnectionRecord modCr, boolean updateLru) {
	    if (modCr != null && modCr.binding.client != null) {
	        if (!modCr.binding.client.hasActivities()) {
	            // This connection is from a client without activities, so adding and removing is not interesting.
	            // 如果和Service建立连接的Client进程没有Activity, 则直接返回
	            return false;
	        }
	    }
 
	    // 从当前进程的services服务列表的ConnectionRecord列表中, 查找和其建立连接的Client, 看是否Client有Activity(即anyClientActivities).
	    boolean anyClientActivities = false;
	    for (int i=proc.services.size()-1; i>=0 && !anyClientActivities; i--) {
	        ServiceRecord sr = proc.services.valueAt(i);
	        ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = sr.getConnections();
	        for (int conni = connections.size() - 1; conni >= 0 && !anyClientActivities; conni--) {
	            ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
	            for (int cri=clist.size()-1; cri>=0; cri--) {
	                ConnectionRecord cr = clist.get(cri);
	                if (cr.binding.client == null || cr.binding.client == proc) {
	                    // Binding to ourself is not interesting.
	                    continue;
	                }
	                if (cr.binding.client.hasActivities()) {
	                    anyClientActivities = true;
	                    break;
	                }
	            }
	        }
	    }

	    // 更新当前进程状态和lru进程列表
	    if (anyClientActivities != proc.hasClientActivities()) {
	        proc.setHasClientActivities(anyClientActivities);
	        if (updateLru) {
	            mAm.updateLruProcessLocked(proc, anyClientActivities, null);
	        }
	        return true;
	    }
	    return false;
	}

updateServiceClientActivitiesLocked方法含义:  
从当前进程的services服务列表的ConnectionRecord列表中, 查找和其建立连接的Client, 看是否Client有Activity, 并更新该进程状态信息和lru进程列表.

### 4.6 bringUpServiceLocked

参考文章: **[startService过程](http://mouxuejie.com/blog/2020-03-28/start-service/)**

bringUpServiceLocked方法在上一篇文章 **[startService过程](http://mouxuejie.com/blog/2020-03-28/start-service/)** 讲过. 会调到realStartServiceLocked方法.        
(1) 对于startService调用链为: realStartServiceLocked -> ApplicationThread.scheduleCreateService -> sendServiceArgsLocked   
(2) 对于bindService调用链为: realStartServiceLocked -> ApplicationThread.scheduleCreateService -> requestServiceBindingsLocked  

接下来我们看一下realStartServiceLocked方法的具体代码.  

### 4.7 realStartServiceLocked

	private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
	    ...
	    // 服务创建
	    app.thread.scheduleCreateService(r, r.serviceInfo,
	           mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
	           app.getReportedProcState());
	    r.postNotification();
	    created = true;
  
	    if (r.whitelistManager) {
	        app.whitelistManager = true;
	    }
  
	    // 请求绑定服务
	    requestServiceBindingsLocked(r, execInFg);
  
	    // 从当前进程的services服务列表的ConnectionRecord列表中, 查找和其建立连接的Client, 看是否Client有Activity
	    // 并更新该进程状态信息和lru进程列表
	    updateServiceClientActivitiesLocked(app, null, true);
  
	    // 将所有与Service绑定的Client端的uid添加到mBoundClientUids列表中
	    // 并将该列表信息更新到WindowProcessController中
	    if (newService && created) {
	        app.addBoundClientUidsOfNewService(r);
	    }
	    ...
	}

### 4.8 ApplicationThread.scheduleCreateService  

参考文章: **[startService过程](http://mouxuejie.com/blog/2020-03-28/start-service/)**

### 4.9 requestServiceBindingsLocked

	private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
	        throws TransactionTooLargeException {
	    for (int i=r.bindings.size()-1; i>=0; i--) {
	        IntentBindRecord ibr = r.bindings.valueAt(i);
	        if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
	            break;
	        }
	    }
	}

遍历bindings列表每一项, 执行bind操作.

	private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
	        boolean execInFg, boolean rebind) throws TransactionTooLargeException {
	    if (r.app == null || r.app.thread == null) {
	        // If service is not currently running, can't yet bind.
	        return false;
	    }
	    if ((!i.requested || rebind) && i.apps.size() > 0) {
	        try {
	            // 更新ServiceRecord bind阶段各属性
	            bumpServiceExecutingLocked(r, execInFg, "bind");
	            r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
	            // 调用ApplicationThread.scheduleBindService
	            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind, r.app.getReportedProcState());
	            if (!rebind) {
	                i.requested = true;
	            }
	            i.hasBound = true;
	            i.doRebind = false;
	        } catch (TransactionTooLargeException e) {
	            // Keep the executeNesting count accurate.
	            final boolean inDestroying = mDestroyingServices.contains(r);
	            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
	            throw e;
	        } catch (RemoteException e) {
	            // Keep the executeNesting count accurate.
	            final boolean inDestroying = mDestroyingServices.contains(r);
	            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
	            return false;
	        }
	    }
	    return true;
	}

## 5.ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 5.1 ApplicationThread.scheduleBindService

	private class ApplicationThread extends IApplicationThread.Stub {
	    ...
	    public final void scheduleBindService(IBinder token, Intent intent, boolean rebind, int processState) {
	        updateProcessState(processState, false);
	        BindServiceData s = new BindServiceData();
	        s.token = token;
	        s.intent = intent;
	        s.rebind = rebind;
 
	        sendMessage(H.BIND_SERVICE, s);
	    }
	}

### 5.2 H.BIND_SERVICE

	class H extends Handler {
	    ...
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case BIND_SERVICE:
	                handleBindService((BindServiceData)msg.obj);
	                break;
	            ...
	        }
	    }
	}

### 5.3 handleBindService

	private void handleBindService(BindServiceData data) {
	    Service s = mServices.get(data.token);
	    if (s != null) {
	        data.intent.setExtrasClassLoader(s.getClassLoader());
	        data.intent.prepareToEnterProcess();
 
	        if (!data.rebind) {
	            // 调用Service.onBind
	            IBinder binder = s.onBind(data.intent);
		        // 发布服务
	            ActivityManager.getService().publishService(data.token, data.intent, binder);
	        } else {
	            // 调用Service.onRebind
	            s.onRebind(data.intent);
	            // 做一些列表移除工作
	            ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	        }
	    }
	}

## 6.发布服务

### 6.1 AMS.publishService  
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	public void publishService(IBinder token, Intent intent, IBinder service) {
	    synchronized(this) {
	        mServices.publishServiceLocked((ServiceRecord)token, intent, service);
	    }
	}

### 6.2 ActiveServices.publishServiceLocked  
[ -> frameworks/base/services/core/java/com/android/server/am/ActiveServices.java ]

	// 这段代码的含义是: 根据intent封装filter对象并以此作为匹配条件
	// 从bindings列表中的所有ConnectionRecord中查找filter匹配的ConnectionRecord, 然后建立连接
	void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
	    final long origId = Binder.clearCallingIdentity();
	    try {
	        if (r != null) {
	            Intent.FilterComparison filter = new Intent.FilterComparison(intent);
	            IntentBindRecord b = r.bindings.get(filter);
	            if (b != null && !b.received) {
	                b.binder = service;
	                b.requested = true;
	                b.received = true;
	                ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
	                for (int conni = connections.size() - 1; conni >= 0; conni--) {
	                    ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
	                    for (int i=0; i<clist.size(); i++) {
	                        ConnectionRecord c = clist.get(i);
	                        if (!filter.equals(c.binding.intent.intent)) {
	                            continue;
	                        }
							
							// 建立连接. 参考【第7.1节】
	                        c.conn.connected(r.name, service, false);
	                    }
	                }
	            }
				
				// 发布服务后的收尾工作
	            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
	        }
	    } finally {
	        Binder.restoreCallingIdentity(origId);
	    }
	}

## 7.ServiceDispatcher  
[ -> frameworks/base/core/java/android/app/LoadedApk.java ]

### 7.1 InnerConnection

    private static class InnerConnection extends IServiceConnection.Stub {
        final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;
 
        InnerConnection(LoadedApk.ServiceDispatcher sd) {
            mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
        }
 
        public void connected(ComponentName name, IBinder service, boolean dead)
                throws RemoteException {
            LoadedApk.ServiceDispatcher sd = mDispatcher.get();
            if (sd != null) {
	            // 建立连接. 参考【第7.2节】
                sd.connected(name, service, dead);
            }
        }
    }

### 7.2 connected

	// 如果有线程池在Runnable中执行, 否则直接在当前线程执行, 最终都调到doConnected
	public void connected(ComponentName name, IBinder service, boolean dead) {
        if (mActivityExecutor != null) {
            mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
        } else if (mActivityThread != null) {
            mActivityThread.post(new RunConnection(name, service, 0, dead));
        } else {
            doConnected(name, service, dead);
        }
    }

### 7.3 RunConnection

    private final class RunConnection implements Runnable {
        RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
            mName = name;
            mService = service;
            mCommand = command;
            mDead = dead;
        }
 
        public void run() {
            if (mCommand == 0) {
	            // 建立连接. 参考【第7.4节】
                doConnected(mName, mService, mDead);
            } else if (mCommand == 1) {
                doDeath(mName, mService);
            }
        }
 
        final ComponentName mName;
        final IBinder mService;
        final int mCommand;
        final boolean mDead;
    }

### 7.4 doConnected

    // name为要bind的服务对应的ComponentName, service为onBind返回的Binder对象, dead为false表示绑定操作
    public void doConnected(ComponentName name, IBinder service, boolean dead) {
        ServiceDispatcher.ConnectionInfo old;
        ServiceDispatcher.ConnectionInfo info;
 
        synchronized (this) {
            if (mForgotten) {
                // We unbound before receiving the connection; ignore any connection received.
                return;
            }
            // 如果当前连接中已经含义目标Service的ComponentName, 且binder和onBind方法返回的binder相同
            // 则说明已经完成建立连接了, 直接返回
            old = mActiveConnections.get(name);
            if (old != null && old.binder == service) {
                // Huh, already have this one.  Oh well!
                return;
            }
 
            if (service != null) {
	            // 新建ConnectionInfo对象并设置binder, 将ConnectionInfo对象添加到mActiveConnections列表
                // A new service is being connected... set it all up.
                info = new ConnectionInfo();
                info.binder = service;
                info.deathMonitor = new DeathMonitor(name, service);

                service.linkToDeath(info.deathMonitor, 0);
                mActiveConnections.put(name, info);
            } else {
                // The named service is being disconnected... clean up.
                mActiveConnections.remove(name);
            }
 
            if (old != null) {
                old.binder.unlinkToDeath(old.deathMonitor, 0);
            }
        }
 
        // If there was an old service, it is now disconnected.
        if (old != null) {
            mConnection.onServiceDisconnected(name);
        }
        if (dead) {
            mConnection.onBindingDied(name);
        }
        // If there is a new viable service, it is now connected.
        if (service != null) {
	        // 回调ServiceConnection.onServiceConnected
            mConnection.onServiceConnected(name, service);
        } else {
            // The binding machinery worked, but the remote returned null from onBind().
            mConnection.onNullBinding(name);
        }
    }

## 8.ServiceConnection.onServiceConnected
[ -> frameworks/base/core/java/android/content/ServiceConnection.java ]

一般是开发者调用bindService时, 手动传入ServiceConnection的实现类.

## 9.总结

bindService和startService过程差不多, 差别主要是在创建Service实例后一个是执行bind操作, 一个是执行start操作.  


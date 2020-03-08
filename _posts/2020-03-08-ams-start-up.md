---
layout: post
comments: true
title: "AMS启动过程"
description: "AMS启动过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

SystemServer进程启动过程中会启动很多服务, 本文只分析启动AMS服务的过程.

## 0.文件结构

	frameworks/base/services/java/com/android/server/SystemServer.java
	
	frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java

## 1.时序图
TODO

## 2.SystemServer.run
[ -> frameworks/base/services/java/com/android/server/SystemServer.java ]

	private void run() {
	    ...
	    // 初始化系统上下文
        createSystemContext();
        // 初始化SystemServiceManager
	    mSystemServiceManager = new SystemServiceManager(mSystemContext);
	    ...
	    // 启动相互依赖关系复杂的服务
	    startBootstrapServices();
	    // 启动相互依赖性的基本服务
	    startCoreServices();
	    // 启动其他服务
	    startOtherServices();
	    ...
	}

    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);

        final Context systemUiContext = activityThread.getSystemUiContext();
        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
    }

### 2.1 startBootstrapServices

	private void startBootstrapServices() {
	    ...
	    // 启动ATMS服务. 见【第3节】
	    ActivityTaskManagerService atm = mSystemServiceManager
		    .startService(ActivityTaskManagerService.Lifecycle.class).getService();
	    // 启动AMS服务. 见【第4.1和4.2节】
	    mActivityManagerService = ActivityManagerService.Lifecycle
		    .startService(mSystemServiceManager, atm);
	    // 设置AMS服务的系统服务管理器
	    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	    // 设置AMS的App安装器
	    mActivityManagerService.setInstaller(installer);
	    // 初始化AMS相关的PMS. 见【第4.4节】
	    mActivityManagerService.initPowerManagement();
	    // 设置系统进程的应用程序实例. 见【第4.5节】
	    mActivityManagerService.setSystemProcess();
	    ...
	}

### 2.2 startCoreServices

	/**
	 * Starts some essential services that are not tangled up in the bootstrap process.
	 */
	private void startCoreServices() {
	    ...
	    mActivityManagerService.setUsageStatsManager(LocalServices.getService(UsageStatsManagerInternal.class));
	    ...
	}
	
### 2.3 startOtherServices

	/**
	 * Starts a miscellaneous grab bag of stuff that has yet to be refactored and organized.
	 */
	private void startOtherServices() {
	    ...
	    // 安装系统Provider. 见【第4.6节】
	    mActivityManagerService.installSystemProviders();
 
	    // 创建并设置WMS和IMS
	    inputManager = new InputManagerService(context);
	    wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
	            new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
	    ServiceManager.addService(Context.WINDOW_SERVICE, wm,
			    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
	    ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
			    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
	    mActivityManagerService.setWindowManager(wm);
	    wm.onInitReady();
	    ...
 
	    // 等待系统ready状态, 并执行相关操作. 见【第4.7节】
	    mActivityManagerService.systemReady(() -> {
	        // 启动状态 PHASE_ACTIVITY_MANAGER_READY
	        mSystemServiceManager.startBootPhase(SystemService.PHASE_ACTIVITY_MANAGER_READY);
	        // Native Crash监听
	        mActivityManagerService.startObservingNativeCrashes();   
 
	        // WebView初始化
	        final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
	        Future<?> webviewPrep = null;
	        if (!mOnlyCore && mWebViewUpdateService != null) {
	            webviewPrep = SystemServerInitThreadPool.get().submit(() -> {
	                ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
	                mZygotePreload = null;
	                mWebViewUpdateService.prepareWebViewInSystemServer();
	            }, WEBVIEW_PREPARATION);
	        }
 
	        // 启动系统UI服务"com.android.systemui/.SystemUIService"
	        startSystemUi(context, windowManagerF);
 
	        // 各种systemReady回调
	        networkManagementF.systemReady();
	        networkPolicyInitReadySignal = networkPolicyF.networkScoreAndNetworkManagementServiceReady();
	        ipSecServiceF.systemReady();
	        networkStatsF.systemReady();
	        connectivityF.systemReady();
	        networkPolicyF.systemReady(networkPolicyInitReadySignal);
 
	        // 确保WebView初始化完成
	        ConcurrentUtils.waitForFutureNoInterrupt(webviewPrep, WEBVIEW_PREPARATION);
 
	        // 启动状态 PHASE_THIRD_PARTY_APPS_CAN_START
	        mSystemServiceManager.startBootPhase(SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
 
	        // 启动第三方服务或App
	        NetworkStackClient.getInstance().start(context);
	        locationF.systemRunning();
	        countryDetectorF.systemRunning();
	        networkTimeUpdaterF.systemRunning();
	        inputManagerF.systemRunning();
	        telephonyRegistryF.systemRunning();
	        mediaRouterF.systemRunning();
	        mmsServiceF.systemRunning();
	        final IIncidentManager incident = IIncidentManager.Stub
		        .asInterface(ServiceManager.getService(Context.INCIDENT_SERVICE));
	        incident.systemRunning();
	    }, BOOT_TIMINGS_TRACE_LOG);
	}

	// 启动"com.android.systemui/.SystemUIService"服务
	private static void startSystemUi(Context context, WindowManagerService windowManager) {
	    Intent intent = new Intent();
	    intent.setComponent(new ComponentName("com.android.systemui", "com.android.systemui.SystemUIService"));
	    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
	    context.startServiceAsUser(intent, UserHandle.SYSTEM);
	    windowManager.onSystemUiStarted();
	}

## 3.SystemServiceManager
[ -> frameworks/base/services/core/java/com/android/server/SystemServiceManager.java ]

	public <T extends SystemService> T startService(Class<T> serviceClass) {
	    final String name = serviceClass.getName();
 
	    // Create the service.
	    if (!SystemService.class.isAssignableFrom(serviceClass)) {
	        throw new RuntimeException(...);
	    }
	    final T service;
	    try {
		    // 反射创建Service
	        Constructor<T> constructor = serviceClass.getConstructor(Context.class);
	        service = constructor.newInstance(mContext);
	    } catch (XXXException ex) {
	        ...
	    }
 
		// 启动Service
	    startService(service);
	    return service;
	}
	
	public void startService(@NonNull final SystemService service) {
	    // 将Service添加到服务列表
	    mServices.add(service);
	    try {
		    // 启动服务
	        service.onStart();
	    } catch (RuntimeException ex) {
	        ...
	    }
	}

## 4.ActivityManagerService
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

### 4.1 Lifecycle

	// 创建和管理AMS服务, 并提供服务的生命周期方法
	public static final class Lifecycle extends SystemService {
	    private final ActivityManagerService mService;
	    private static ActivityTaskManagerService sAtm;
 
	    public Lifecycle(Context context) {
	        super(context);
	        // 创建AMS. 见【第4.2节】
	        mService = new ActivityManagerService(context, sAtm);
	    }
 
	    public static ActivityManagerService startService(SystemServiceManager ssm,
			    ActivityTaskManagerService atm) {
	        sAtm = atm;
	        // SystemServiceManager启动服务. 见【第3节】
	        return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
	    }
 
	    @Override
	    public void onStart() {
		    // 启动服务. 见【第4.3节】
	        mService.start();
	    }
 
	    @Override
	    public void onBootPhase(int phase) { // 启动过程的每个阶段都会调用
	        mService.mBootPhase = phase;
	        if (phase == PHASE_SYSTEM_SERVICES_READY) {
	            mService.mBatteryStatsService.systemServicesReady();
	            mService.mServices.systemServicesReady();
	        } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
	            mService.startBroadcastObservers();
	        } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
	            mService.mPackageWatchdog.onPackagesReady();
	        }
	    }
 
	    @Override
	    public void onCleanupUser(int userId) {
	        mService.mBatteryStatsService.onCleanupUser(userId);
	    }
 
	    public ActivityManagerService getService() {
	        return mService;
	    }
	}

### 4.2 构造函数

	public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
	    mInjector = new Injector();
	    mContext = systemContext;
	    mSystemThread = ActivityThread.currentActivityThread();
	    mUiContext = mSystemThread.getSystemUiContext();
 
	    // 创建并启动名称为"ActivityManager"的线程
	    mHandlerThread = new ServiceThread(TAG, THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
	    mHandlerThread.start();
	    mHandler = new MainHandler(mHandlerThread.getLooper());
 
	    // 通过UiThread,创建名称为"android.ui"的线程,用于显示UI
	    mUiHandler = mInjector.getUiHandler(this);
 
	    // 创建并启动名称为"ActivityManager:procStart"的线程
	    mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
			    THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
	    mProcStartHandlerThread.start();
	    mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());
	    ...
	    // 创建前台广播队列, 并设置前台广播超时时间10s
	    final BroadcastConstants foreConstants = new BroadcastConstants(Settings.Global.BROADCAST_FG_CONSTANTS);
	    foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;
	    // 创建后台广播队列, 并设置后台广播超时时间60s
	    final BroadcastConstants backConstants = new BroadcastConstants(Settings.Global.BROADCAST_BG_CONSTANTS);
	    backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
	    // 创建离线广播队列, 并设置离线广播超时时间60s
	    final BroadcastConstants offloadConstants = new BroadcastConstants(Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
	    offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
	    // by default, no "slow" policy in this queue
	    offloadConstants.SLOW_TIME = Integer.MAX_VALUE;
 
	    mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", foreConstants, false);
	    mBgBroadcastQueue = new BroadcastQueue(this, mHandler, "background", backConstants, true);
	    mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler, "offload", offloadConstants, true);
	    mBroadcastQueues[0] = mFgBroadcastQueue;
	    mBroadcastQueues[1] = mBgBroadcastQueue;
	    mBroadcastQueues[2] = mOffloadBroadcastQueue;
 
	    // 启动ActiveServices, 设置后台最大服务启动数mMaxStartingBackground. 低内存时为1, 非低内存时为8
	    mServices = new ActiveServices(this);
	    mProviderMap = new ProviderMap(this);
	    mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
	    mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);
 
	    // 创建电池统计服务BatteryStatsService
	    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
			    BackgroundThread.get().getHandler());
	    mBatteryStatsService.getActiveStatistics().readLocked();
	    mBatteryStatsService.scheduleWriteToDisk();
	    mBatteryStatsService.getActiveStatistics().setCallback(this);
 
	    // 创建进程统计服务ProcessStatsService, 信息保存在目录/data/system/procstats  
	    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
 
	    // 创建AppOpsService服务
	    mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
	    ...
	    // 初始化ATMS服务
	    mActivityTaskManager = atm;
	    mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
			    DisplayThread.get().getLooper());
	    mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
 
	    // 创建名称为"CpuTracker"的线程
	    mProcessCpuThread = new Thread("CpuTracker") {
	        @Override
	        public void run() {
	            synchronized (mProcessCpuTracker) {
	                mProcessCpuInitLatch.countDown();
	                mProcessCpuTracker.init();
	            }
	            while (true) {
	                synchronized(this) {
	                    final long now = SystemClock.uptimeMillis();
	                    long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
	                    long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
	                    if (nextWriteDelay < nextCpuDelay) {
	                        nextCpuDelay = nextWriteDelay;
	                    }
	                    if (nextCpuDelay > 0) {
	                        mProcessCpuMutexFree.set(true);
	                        this.wait(nextCpuDelay);
	                    }
	                }
	                // 更新CPU状态信息
	                updateCpuStatsNow();
	            }
	        }
	    };
 
	    Watchdog.getInstance().addMonitor(this);
	    Watchdog.getInstance().addThread(mHandler);
	    ...
	}

### 4.3 start

	private void start() {
	    // 移除所有进程组
	    removeAllProcessGroups();
	    // 启动"CpuTracker"线程
	    mProcessCpuThread.start();
	    // 启动电池统计服务BatteryStatsService
	    mBatteryStatsService.publish();
	    // 启动AppOpsService
	    mAppOpsService.publish(mContext);
	    // 添加本地服务
	    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
	    // 确保"CpuTracker"线程启动完成
	    mProcessCpuInitLatch.await();
	}
 
### 4.4 initPowerManagement
 
	public void initPowerManagement() {
	    mActivityTaskManager.onInitPowerManagement();
	    mBatteryStatsService.initPowerManagement();
	    mLocalPowerManager = LocalServices.getService(PowerManagerInternal.class);
	}
 
### 4.5 setSystemProcess

	public void setSystemProcess() {
	    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
			    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
	    ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
	    ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
			    DUMP_FLAG_PRIORITY_HIGH);
	    ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
	    ServiceManager.addService("dbinfo", new DbBinder(this));
	    if (MONITOR_CPU_USAGE) {
	        ServiceManager.addService("cpuinfo", new CpuBinder(this),
			        /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
	    }
	    ServiceManager.addService("permission", new PermissionController(this));
	    ServiceManager.addService("processinfo", new ProcessInfoService(this));
 
	    ApplicationInfo info = mContext.getPackageManager()
			    .getApplicationInfo("android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
	    mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
 
	    synchronized (this) {
	        // 创建persistent进程, 设置其adj为系统级别
	        ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
			        false, 0, new HostingRecord("system"));
	        app.setPersistent(true);
	        app.pid = MY_PID;
	        app.getWindowProcessController().setPid(MY_PID);
	        app.maxAdj = ProcessList.SYSTEM_ADJ;
	        app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
	        mPidsSelfLocked.put(app);
	        // 更新lru进程列表
	        mProcessList.updateLruProcessLocked(app, false, null);
	        // 更新adj
	        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
	    }
 
	    // Start watching app ops after we and the package manager are up and running.
	    mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
            new IAppOpsCallback.Stub() {
                @Override 
                public void opChanged(int op, int uid, String packageName) {
                    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                        if (mAppOpsService.checkOperation(op, uid, packageName) !=
		                        AppOpsManager.MODE_ALLOWED) {
                            runInBackgroundDisabled(uid);
                        }
                    }
                }
            });
	}

### 4.6 installSystemProviders

	public final void installSystemProviders() {
	    List<ProviderInfo> providers;
	    synchronized (this) {
	        ProcessRecord app = mProcessList.mProcessNames.get("system", SYSTEM_UID);
	        providers = generateApplicationProvidersLocked(app);
	        if (providers != null) {
	            for (int i=providers.size()-1; i>=0; i--) {
	                ProviderInfo pi = (ProviderInfo)providers.get(i);
	                // 移除非系统Provider
	                if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
	                    providers.remove(i);
	                }
	            }
	        }
	    }
     
	    // 安装系统Provider
	    if (providers != null) {
	        mSystemThread.installSystemProviders(providers);
	    }
 
	    synchronized (this) {
	        mSystemProvidersInstalled = true;
	    }
	    ...
	}

### 4.7 systemReady

	public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
	    synchronized(this) {
	        if (mSystemReady) {
	            // 如果系统ready, 则直接回调并返回
	            if (goingCallback != null) {
	                goingCallback.run();
	            }
	            return;
	        }
 
	        mLocalDeviceIdleController = LocalServices
			        .getService(DeviceIdleController.LocalService.class);
	        mActivityTaskManager.onSystemReady();
	        // Make sure we have the current profile info,since it is needed for security checks.
	        mUserController.onSystemReady();
	        mAppOpsService.systemReady();
	        // 系统ready
	        mSystemReady = true;
	    }
	    ...
	    // 非persistent进程添加到procsToKill列表
	    ArrayList<ProcessRecord> procsToKill = null;
	    synchronized(mPidsSelfLocked) {
	        for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
	            ProcessRecord proc = mPidsSelfLocked.valueAt(i);
	            if (!isAllowedWhileBooting(proc.info)){
	                if (procsToKill == null) {
	                    procsToKill = new ArrayList<ProcessRecord>();
	                }
	                procsToKill.add(proc);
	            }
	        }
	    }
 
	    // 杀死procsToKill列表的进程
	    synchronized(this) {
	        if (procsToKill != null) {
	            for (int i=procsToKill.size()-1; i>=0; i--) {
	                ProcessRecord proc = procsToKill.get(i);
	                mProcessList.removeProcessLocked(proc, true, false, "system update done");
	            }
	        }
	        // 进程ready状态
	        mProcessesReady = true;
	    }
	    ...
	    if (goingCallback != null) goingCallback.run();
     
	    // 调用所有SystemService的onStartUser方法
	    final int currentUserId = mUserController.getCurrentUserId();
	    mSystemServiceManager.startUser(currentUserId);
	    synchronized (this) {
	        // 启动支持加密的persistent进程
	        startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
 
	        // 设置启动中
	        mBooting = true;
         
	        // 启动桌面Activity
	        mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
	        ...
	        final int callingUid = Binder.getCallingUid();
	        final int callingPid = Binder.getCallingPid();
	        long ident = Binder.clearCallingIdentity();
	        // 发送ACTION_USER_STARTED广播
	        Intent intent = new Intent(Intent.ACTION_USER_STARTED);
	        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY | Intent.FLAG_RECEIVER_FOREGROUND);
	        intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
	        broadcastIntentLocked(null, null, intent, null, null, 0, null, null, null, OP_NONE,
                null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid, currentUserId);       
	        // 发送ACTION_USER_STARTING广播
	        intent = new Intent(Intent.ACTION_USER_STARTING);
	        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
	        intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
	        broadcastIntentLocked(null, null, intent, null, new IIntentReceiver.Stub() {
                @Override
                public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                    }
                }, 0, null, null, new String[] {INTERACT_ACROSS_USERS}, OP_NONE, null, 
                true, false, MY_PID, SYSTEM_UID, callingUid, callingPid, UserHandle.USER_ALL);
	        Binder.restoreCallingIdentity(ident);
 
	        // 恢复栈顶的Activity
	        mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
	        mUserController.sendUserSwitchBroadcasts(-1, currentUserId);
	        ...
	    }
	}

## 5.ActivityTaskManagerService
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java ]

	// 创建和管理AMS服务, 并提供服务的生命周期方法
	public static final class Lifecycle extends SystemService {
	    private final ActivityTaskManagerService mService;
 
	    public Lifecycle(Context context) {
	        super(context);
	        mService = new ActivityTaskManagerService(context);
	    }
 
	    @Override
	    public void onStart() {
	        publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
	        mService.start();
	    }
 
	    @Override
	    public void onUnlockUser(int userId) {
	        synchronized (mService.getGlobalLock()) {
	            mService.mStackSupervisor.onUserUnlocked(userId);
	        }
	    }
 
	    @Override
	    public void onCleanupUser(int userId) {
	        synchronized (mService.getGlobalLock()) {
	            mService.mStackSupervisor.mLaunchParamsPersister.onCleanupUser(userId);
	        }
	    }
 
	    public ActivityTaskManagerService getService() {
	        return mService;
	    }
	}
 
	final class LocalService extends ActivityTaskManagerInternal {
	    @Override
	    public boolean startHomeOnAllDisplays(int userId, String reason) {
	        synchronized (mGlobalLock) {
	            return mRootActivityContainer.startHomeOnAllDisplays(userId, reason);
	        }
	    }
	}
 
## 6.RootActivityContainer
[ -> frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java ]

	boolean startHomeOnAllDisplays(int userId, String reason) {
	    boolean homeStarted = false;
	    for (int i = mActivityDisplays.size() - 1; i >= 0; i--) {
	        final int displayId = mActivityDisplays.get(i).mDisplayId;
	        homeStarted |= startHomeOnDisplay(userId, reason, displayId);
	    }
	    return homeStarted;
	}
	
	boolean startHomeOnDisplay(int userId, String reason, int displayId) {
	    return startHomeOnDisplay(userId, reason, displayId, 
			    false /* allowInstrumenting */, false /* fromHomeKey */);
	}
	
	boolean startHomeOnDisplay(int userId, String reason, int displayId,
			boolean allowInstrumenting, boolean fromHomeKey) {
	    Intent homeIntent = null;
	    ActivityInfo aInfo = null;
	    if (displayId == DEFAULT_DISPLAY) {
	        homeIntent = mService.getHomeIntent();
	        aInfo = resolveHomeActivity(userId, homeIntent);
	    } else if (shouldPlaceSecondaryHomeOnDisplay(displayId)) {
	        Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, displayId);
	        aInfo = info.first;
	        homeIntent = info.second;
	    }
 
	    homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName,
			    aInfo.name));
	    homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
	    if (fromHomeKey) {
	        homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
	    }
	    final String myReason = reason + ":" + userId + ":" +
			    UserHandle.getUserId(aInfo.applicationInfo.uid) + ":" + displayId;
	    mService.getActivityStartController()
			    .startHomeActivity(homeIntent, aInfo, myReason, displayId);
	    return true;
	}

## 7.总结

---
layout: post
comments: true
title: "App进程创建过程"
description: "App进程创建过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文只着重分析Zygote进程fork并启动App进程的过程. 暂不介绍Zygote进程的其他进程启动过程. 

## 0.文件结构

	frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
	frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
	frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
	frameworks/base/core/java/com/android/internal/os/Zygote.java
	frameworks/base/core/java/com/android/internal/os/RuntimeInit.java

	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/LoadedApk.java
	frameworks/base/core/java/android/app/Instrumentation.java
	
	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java
	frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

## 1.时序图
TODO

## 2.ZygoteInit
[ -> frameworks/base/core/java/com/android/internal/os/ZygoteInit.java ]

### 2.1 main

	public static void main(String argv[]) {
	    ZygoteServer zygoteServer = null;
 
	    Runnable caller;
	    try {
	        // 从argv中读取各参数信息
	        boolean startSystemServer = false;
	        String zygoteSocketName = "zygote";
	        String abiList = null;
	        boolean enableLazyPreload = false;
	        for (int i = 1; i < argv.length; i++) {
	            if ("start-system-server".equals(argv[i])) {
	                startSystemServer = true;
	            } else if ("--enable-lazy-preload".equals(argv[i])) {
	                enableLazyPreload = true;
	            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
	                abiList = argv[i].substring(ABI_LIST_ARG.length());
	            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
	                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
	            } else {
	                throw new RuntimeException("Unknown command line argument: " + argv[i]);
	            }
	        }
 
	        // 是否主Zygote进程
	        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
 
	        if (!enableLazyPreload) {
	            // 预加载资源和类
	            preload(bootTimingsTraceLog);
	        } else {
		        // 线程优先级设置为normal
	            Zygote.resetNicePriority();
	        }
 
	        // 启动之后做一次初始GC
	        gcAndFinalize();
	        // 初始化Zygote Native状态
	        Zygote.initNativeState(isPrimaryZygote);
 
	        // 创建Socket Server
	        zygoteServer = new ZygoteServer(isPrimaryZygote);
 
	        if (startSystemServer) {
	            // fork SystemServer进程
	            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
	            // 启动SystemServer进程
	            if (r != null) {
	                r.run();
	                return;
	            }
	        }
 
	        // while(true)循环监听并处理Socket连接，如果有fork子进程请求则fork子进程，并封装到Runnable中返回
	        // 否则一直循环下去
	        caller = zygoteServer.runSelectLoop(abiList);
	    } catch (Throwable ex) {
	        ...
	    } finally {
	        // 关闭Socket Server
	        if (zygoteServer != null) {
	            zygoteServer.closeServerSocket();
	        }
	    }
 
	    // 执行Runnable, 根据下文分析可知会调用ActivityThread.main()方法
	    if (caller != null) {
	        caller.run();
	    }
	}

### 2.2 zygoteInit

	public static final Runnable zygoteInit(int targetSdkVersion, String[] argv,
			ClassLoader classLoader) {
	    // 初始化
	    RuntimeInit.commonInit();
	    ZygoteInit.nativeZygoteInit();
	    // 见【第6.1节】
	    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
	}

## 3.ZygoteServer
[ -> frameworks/base/core/java/com/android/internal/os/ZygoteServer.java ]

### 3.1 runSelectLoop

	Runnable runSelectLoop(String abiList) {
	    ArrayList<FileDescriptor> socketFDs = new ArrayList<FileDescriptor>();
	    // Server Socket保存到socketFDs[0]
	    socketFDs.add(mZygoteSocket.getFileDescriptor());
	    // 填充null占位
	    peers.add(null);
 
	    while (true) {
	        StructPollfd[] pollFDs = null;
	        // 为poll struct数组分组分配空间.Zygote可能包含regular Zygote,WebView Zygote,AppZygote
	        if (mUsapPoolEnabled) {
	            usapPipeFDs = Zygote.getUsapPipeFDs();
                pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
	        } else {
	            pollFDs = new StructPollfd[socketFDs.size()];
	        }
 
	        int pollIndex = 0;
	        // 将socketFDs数组元素依次设置到pollFDs数组, 且pollIndex累加
	        for (FileDescriptor socketFD : socketFDs) {
	            pollFDs[pollIndex] = new StructPollfd();
	            pollFDs[pollIndex].fd = socketFD;
	            pollFDs[pollIndex].events = (short) POLLIN;
	            ++pollIndex;
	        }
	        // 关于mUsapPoolEventFD的解释： File descriptor used for communication
	        // between the signal handler and the ZygoteServer poll loop.
	        final int usapPoolEventFDIndex = pollIndex;

            // 将usapPipeFDs数组元素依次设置到pollFDs数组, pollIndex累加
            if (mUsapPoolEnabled) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = mUsapPoolEventFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;

                for (int usapPipeFD : usapPipeFDs) {
                    FileDescriptor managedFd = new FileDescriptor();
                    managedFd.setInt$(usapPipeFD);

                    pollFDs[pollIndex] = new StructPollfd();
                    pollFDs[pollIndex].fd = managedFd;
                    pollFDs[pollIndex].events = (short) POLLIN;
                    ++pollIndex;
                }
            }
	        // 轮询pollFDs文件描述符列表的状态，当pollFds有事件到来则往下执行，否则阻塞在这里
	        // 这里用到了IO多路复用机制
	        Os.poll(pollFDs, -1);
	        ...
	        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
	        peers.add(null);
	        while (--pollIndex >= 0) {
	            // 过滤非POLLIN的文件描述符
	            if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
	                continue;
	            }
 
	            if (pollIndex == 0) { // 处理socketFDs[0]，即ServerSocket 
	                // 监听Socket连接， 并创建一个ZygoteConnection
	                ZygoteConnection newPeer = acceptCommandPeer(abiList);
	                peers.add(newPeer);
	                socketFDs.add(newPeer.getFileDescriptor());
	            } else if (pollIndex < usapPoolEventFDIndex) { // 处理socketFDs数组除index=0的元素
	                // Session socket accepted from the Zygote server socket
	                ZygoteConnection connection = peers.get(pollIndex);
	                // 处理指令. 见【第4.1节】
	                final Runnable command = connection.processOneCommand(this);
	                if (mIsForkChild) { // fork子进程, 返回Runnable并跳出循环
	                    return command;
	                } else {
	                    ...
	                }
	                mIsForkChild = false;
	            } else { // 处理usapPipeFDs数组的元素
	                long messagePayload = -1;
                    byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                    int readBytes = Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);
                    if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                        DataInputStream inputStream =
                                new DataInputStream(new ByteArrayInputStream(buffer));
                        messagePayload = inputStream.readLong();
                    } else {
                        continue;
                    }
                    if (pollIndex > usapPoolEventFDIndex) {
                        Zygote.removeUsapTableEntry((int) messagePayload);
                    }
                    usapPoolFDRead = true;
	            }
	        }
	    }
	}

解释下这段代码：  
第一次while(true)时, socketFDs只有一个元素即ServerSocket, 因此会走到if (pollIndex == 0)分支, 结束时socketFDs有2个元素: ServerSocket和newPeer对应的Socket. peers也有2个元素: null和newPeer   
第一次while(true)时, socketFDs有2个元素, 当走到if (pollIndex < usapPoolEventFDIndex)分支时, 此时pollIndex=1, peers取出的元素为newPeer, 然后就会执行到connection.processOneCommand(...)   

### 3.2 acceptCommandPeer

    // 监听Socket连接， 并创建一个ZygoteConnection
    private ZygoteConnection acceptCommandPeer(String abiList) {
        return createNewConnection(mZygoteSocket.accept(), abiList);
    }
    
    protected ZygoteConnection createNewConnection(LocalSocket socket, String abiList) {
        return new ZygoteConnection(socket, abiList);
    }
    
## 4.ZygoteConnection
[ -> frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java ]

### 4.1 processOneCommand

	Runnable processOneCommand(ZygoteServer zygoteServer) {
	    // 从ServerSocket的输入流读取指令参数
	    String[] args = Zygote.readArgumentList(mSocketReader);
	    ZygoteArguments parsedArgs = new ZygoteArguments(args);
	    // 处理args指令, 如果是处理fork进程外的其他指令则返回值为null
	    ...
 
	    // fork进程, 返回值为0表示是子进程, -1表示错误. 见【第5.1节】
	    pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
            parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
            parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mTargetSdkVersion);
 
	    try {
	        if (pid == 0) { // 成功创建子进程
	            // 设置标志位, 关闭ServerSocket连接
	            zygoteServer.setForkChild();
	            zygoteServer.closeServerSocket();
	            IoUtils.closeQuietly(serverPipeFd);
	            serverPipeFd = null;
				// 处理子进程
	            return handleChildProc(parsedArgs, descriptors, childPipeFd,
			            parsedArgs.mStartChildZygote);
	        } else { // 错误处理
	            IoUtils.closeQuietly(childPipeFd);
	            childPipeFd = null;
	            handleParentProc(pid, descriptors, serverPipeFd);
	            return null;
	        }
	    } finally {
	        IoUtils.closeQuietly(childPipeFd);
	        IoUtils.closeQuietly(serverPipeFd);
	    }
	}

### 4.2 handleChildProc

	// isZygote表示创建的子进程也是Zygote进程
	private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor[] descriptors,
	        FileDescriptor pipeFd, boolean isZygote) {
	    if (!isZygote) { // 非Zygote进程
		    // 见【第2.2节】
	        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
	                parsedArgs.mRemainingArgs, null /* classLoader */);
	    } else {
	        return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
	                parsedArgs.mRemainingArgs, null /* classLoader */);
	    }
	}

## 5.Zygote
[ -> frameworks/base/core/java/com/android/internal/os/Zygote.java ]

### 5.1 forkAndSpecialize

	public static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
        int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir,
        int targetSdkVersion) {
	    ZygoteHooks.preFork();
	    // 重置线程优先级为NORM_PRIORITY.
	    resetNicePriority();
	    // native调用
	    int pid = nativeForkAndSpecialize(uid, gid, gids, runtimeFlags, 
			    rlimits, mountExternal, seInfo, niceName, fdsToClose,
	            fdsToIgnore, startChildZygote, instructionSet, appDataDir);
	    if (pid == 0) {
	        // targetSdkVersion<Q时, 设置library文件可读可执行
	        Zygote.disableExecuteOnly(targetSdkVersion);
	    }
	    ZygoteHooks.postForkCommon();
	    return pid;
	}

## 6.RuntimeInit
[ -> frameworks/base/core/java/com/android/internal/os/RuntimeInit.java ]

### 6.1 applicationInit

	protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
			ClassLoader classLoader) {
	    final Arguments args = new Arguments(argv);
	    // 查找进程入口类的静态main方法, 并封装成Runnable返回, 此处进程入口类是ActivityThread
	    return findStaticMain(args.startClass, args.startArgs, classLoader);
	}

### 6.2 findStaticMain

	protected static Runnable findStaticMain(String className, String[] argv,
			ClassLoader classLoader) {
	    Class<?> cl = Class.forName(className, true, classLoader);
	    Method m = cl.getMethod("main", new Class[] { String[].class });
	    return new MethodAndArgsCaller(m, argv);
	}

### 6.3 MethodAndArgsCaller

	static class MethodAndArgsCaller implements Runnable {
	    /** method to call */
	    private final Method mMethod;
 
	    /** argument array */
	    private final String[] mArgs;
 
	    public MethodAndArgsCaller(Method method, String[] args) {
	        mMethod = method;
	        mArgs = args;
	    }
 
	    public void run() {
	        try {
	            mMethod.invoke(null, new Object[] { mArgs });
	        } catch (IllegalAccessException ex) {
	            ...
	        }
	    }
	}

接下来会执行Runnable#run()方法, 开始ActivityThread.main()方法的调用.    

## 7.ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 7.1 main

	public static void main(String[] args) {
	    Looper.prepareMainLooper();
 
	    // 从args参数中获取startSeq
	    long startSeq = 0;
	    if (args != null) {
	        for (int i = args.length - 1; i >= 0; --i) {
	            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
	                startSeq = Long.parseLong(args[i].substring(PROC_START_SEQ_IDENT.length()));
	            }
	        }
	    }
	    ActivityThread thread = new ActivityThread();
	    thread.attach(false, startSeq);
 
	    if (sMainThreadHandler == null) {
	        sMainThreadHandler = thread.getHandler();
	    }
		// 在主线程中循环
	    Looper.loop();
	}

### 7.2 attach

	private void attach(boolean system, long startSeq) {
	    sCurrentActivityThread = this;
	    mSystemThread = system;
	    if (!system) { // 非系统进程
	        // 设置app name
	        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>", UserHandle.myUserId());
	        RuntimeInit.setApplicationObject(mAppThread.asBinder());
	        // AMS attach应用程序. 见【第8节】
	        final IActivityManager mgr = ActivityManager.getService();
	        mgr.attachApplication(mAppThread, startSeq);       
	        // GC监听
	        BinderInternal.addGcWatcher(new Runnable() {
	            @Override public void run() {
	                // 是否有Activity生命周期发生变化
	                if (!mSomeActivitiesChanged) {
	                    return;
	                }
	                Runtime runtime = Runtime.getRuntime();
	                long dalvikMax = runtime.maxMemory();
	                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
	                // 当虚拟机使用的内存超出最大内存的3/4时, 释放一些Activity
	                if (dalvikUsed > ((3*dalvikMax)/4)) {
	                    mSomeActivitiesChanged = false;
	                    ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
	                }
	            }
	        });
	    } else { // 系统进程
	        // Don't set application object here -- if the system crashes,
	        // we can't display an alert, we just want to die die die.
	        android.ddm.DdmHandleAppName.setAppName("system_process", UserHandle.myUserId());
	        mInstrumentation = new Instrumentation();
	        mInstrumentation.basicInit(this);
	        ContextImpl context = ContextImpl.createAppContext(this,
			        getSystemContext().mPackageInfo);
	        mInitialApplication = context.mPackageInfo.makeApplication(true, null);
	        mInitialApplication.onCreate();
	    }
 
		// 设置全局配置变化回调
	    ViewRootImpl.ConfigChangedCallback configChangedCallback = 
			    (Configuration globalConfig) -> {
	        synchronized (mResourcesManager) {
	            // We need to apply this change to the resources immediately,
	            // because upon returning the view hierarchy will be informed about it.
	            if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,null)) {
	                updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(), 
					        mResourcesManager.getConfiguration().getLocales());
 
	                // This actually changed the resources! Tell everyone about it.
	                if (mPendingConfiguration == null || 
			                mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
	                    mPendingConfiguration = globalConfig;
	                    sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
	                }
	            }
	        }
	    };
	    ViewRootImpl.addConfigCallback(configChangedCallback);
	}

## 8.ActivityManagerService.attachApplication
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	@Override
	public final void attachApplication(IApplicationThread thread, long startSeq) {
	    synchronized (this) {
	        int callingPid = Binder.getCallingPid();
	        final int callingUid = Binder.getCallingUid();
	        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
	    }
	}
 
	private final boolean attachApplicationLocked(IApplicationThread thread, int pid, 
			int callingUid, long startSeq) {
        ProcessRecord app;
        // 从mPidsSelfLocked列表根据pid查找进程
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        }

        // 若上述查找app为空，则从mPendingStarts列表根据startSeq查找进程
        if (app == null && startSeq > 0) {
            final ProcessRecord pending = mProcessList.mPendingStarts.get(startSeq);
            if (pending != null && pending.startUid == callingUid && pending.startSeq == startSeq
                    && mProcessList.handleProcessStartedLocked(pending, pid,
                    pending.isUsingWrapper(), startSeq, true)) {
                app = pending;
            }
        }
        ...
        // 进程是否处于正常模式
	    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
	    if (app.isolatedEntryPoint != null) { // isolated进程
	        // This is an isolated process which should just call an entry point instead of
	        // being bound to an application.
	        thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
	    } else if (instr2 != null) {
		    // 绑定App进程. 见【第9.1节】
	        thread.bindApplication(processName, appInfo, providers,
	                instr2.mClass,
	                profilerInfo, instr2.mArguments,
	                instr2.mWatcher,
	                instr2.mUiAutomationConnection, testMode,
	                mBinderTransactionTrackingEnabled, enableTrackAllocation,
	                isRestrictedBackupMode || !normalMode, app.isPersistent(),
	                new Configuration(app.getWindowProcessController().getConfiguration()),
	                app.compat, getCommonServicesLocked(app.isolated),
	                mCoreSettingsObserver.getCoreSettingsLocked(),
	                buildSerial, autofillOptions, contentCaptureOptions);
	    } else {
	        thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
	                null, null, null, testMode,
	                mBinderTransactionTrackingEnabled, enableTrackAllocation,
	                isRestrictedBackupMode || !normalMode, app.isPersistent(),
	                new Configuration(app.getWindowProcessController().getConfiguration()),
	                app.compat, getCommonServicesLocked(app.isolated),
	                mCoreSettingsObserver.getCoreSettingsLocked(),
	                buildSerial, autofillOptions, contentCaptureOptions);
	    }
 
	    // 绑定应用程序后, 设置app active, 并更新lru进程信息
	    app.makeActive(thread, mProcessStats);
	    mProcessList.updateLruProcessLocked(app, false, null);
	    app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
     
	    // Remove this record from the list of starting applications.
	    mPersistentStartingProcesses.remove(app);
	    mProcessesOnHold.remove(app);
 
	    boolean didSomething = false;
 
	    // See if the top visible activity is waiting to run in this process...
	    // 启动进程中最上面可见的Activity. 见【第12节】
	    if (normalMode) {
	        didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
	    }
 
	    // Find any services that should be running in this process...
	    // 启动进程中的服务. (启动服务过程本文暂不讲解)
	    didSomething |= mServices.attachApplicationLocked(app, processName);
 
	    // Check if a next-broadcast receiver is in this process...
	    // 发送pending中的广播. (发送广播过程本文暂不讲解)
	    if (isPendingBroadcastProcessLocked(pid)) {
	        didSomething |= sendPendingBroadcastsLocked(app);
	    }
 
	    if (!didSomething) {
		    // 进程启动，更新Adj
	        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_PROCESS_BEGIN);
	    }
 
	    return true;
	}

## 9.ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 9.1 ApplicationThread.bindApplication

	private class ApplicationThread extends IApplicationThread.Stub {
		public final void bindApplication(String processName, ApplicationInfo appInfo,
		        List<ProviderInfo> providers, ComponentName instrumentationName,
		        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
		        IInstrumentationWatcher instrumentationWatcher,
		        IUiAutomationConnection instrumentationUiConnection, int debugMode,
		        boolean enableBinderTracking, boolean trackAllocation,
		        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
		        CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
		        String buildSerial, AutofillOptions autofillOptions,
		        ContentCaptureOptions contentCaptureOptions) {
		    ...
		    // 封装App进程绑定数据AppBindData
		    AppBindData data = new AppBindData();
		    data.processName = processName;
		    data.appInfo = appInfo;
		    data.providers = providers;
		    data.instrumentationName = instrumentationName;
		    data.instrumentationArgs = instrumentationArgs;
		    data.instrumentationWatcher = instrumentationWatcher;
		    data.instrumentationUiAutomationConnection = instrumentationUiConnection;
		    data.debugMode = debugMode;
		    data.enableBinderTracking = enableBinderTracking;
		    data.trackAllocation = trackAllocation;
		    data.restrictedBackupMode = isRestrictedBackupMode;
		    data.persistent = persistent;
		    data.config = config;
		    data.compatInfo = compatInfo;
		    data.initProfilerInfo = profilerInfo;
		    data.buildSerial = buildSerial;
		    data.autofillOptions = autofillOptions;
		    data.contentCaptureOptions = contentCaptureOptions;
		    // 向H发送BIND_APPLICATION广播
		    sendMessage(H.BIND_APPLICATION, data);
		}
	}

### 9.2 H.handleBindApplication

	class H extends Handler {
	    public static final int BIND_APPLICATION        = 110;
 
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            case BIND_APPLICATION: // 处理App进程绑定
	                AppBindData data = (AppBindData)msg.obj;
	                handleBindApplication(data);
	                break;
	        }
	    }
	}

	private void handleBindApplication(AppBindData data) {
	    // 从AppBindData中取出InstrumentationInfo信息
	    final InstrumentationInfo ii;
	    if (data.instrumentationName != null) {
	        ii = new ApplicationPackageManager(null, 
			        getPackageManager()).getInstrumentationInfo(data.instrumentationName, 0);
	        mInstrumentationPackageName = ii.packageName;
	        mInstrumentationAppDir = ii.sourceDir;
	        mInstrumentationSplitAppDirs = ii.splitSourceDirs;
	        mInstrumentationLibDir = getInstrumentationLibrary(data.appInfo, ii);
	        mInstrumentedAppDir = data.info.getAppDir();
	        mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
	        mInstrumentedLibDir = data.info.getLibDir();
	    }
 
	    // 创建Context
	    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
 
		// InstrumentationInfo对象非空
	    if (ii != null) { // 从InstrumentationInfo中取出各参数, 基于此创建和初始化Instrumentation
	        ApplicationInfo instrApp = getPackageManager()
			        .getApplicationInfo(ii.packageName, 0, UserHandle.myUserId());
	        if (instrApp == null) {
	            instrApp = new ApplicationInfo();
	        }
	        // 将InstrumentationInfo里的参数拷贝到ApplicationInfo对象
	        ii.copyTo(instrApp);
	        instrApp.initForUser(UserHandle.myUserId());
	        final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo, 
			        appContext.getClassLoader(), false, true, false);
 
	        final ContextImpl instrContext = ContextImpl.createAppContext(this,
			        pi, appContext.getOpPackageName());
	        final ClassLoader cl = instrContext.getClassLoader();
	        mInstrumentation = (Instrumentation) cl
			        .loadClass(data.instrumentationName.getClassName()).newInstance();
	        final ComponentName component = new ComponentName(ii.packageName, ii.name);
	        mInstrumentation.init(this, instrContext, appContext, component, 
			        data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
	    } else { // 初始化Instrumentation
	        mInstrumentation = new Instrumentation();
	        mInstrumentation.basicInit(this);
	    }
 
	    // 创建Application对象. 见【第10节】
	    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
	    mInitialApplication = app;
 
	    // 调用Application.onCreate()方法. 见【第11节】
	    mInstrumentation.onCreate(data.instrumentationArgs);       
	    mInstrumentation.callApplicationOnCreate(app);
	    ...
	}

## 10.LoadedApk.makeApplication
[ -> frameworks/base/core/java/android/app/LoadedApk.java ]

LoadedApk: Local state maintained about a currently loaded .apk.

	public Application makeApplication(boolean forceDefaultAppClass, 
			Instrumentation instrumentation) {
	    if (mApplication != null) {
	        return mApplication;
	    }
 
	    String appClass = mApplicationInfo.className;
	    // 设置默认Application-"android.app.Application"
	    if (forceDefaultAppClass || appClass == null) {
	        appClass = "android.app.Application";
	    }
 
	    java.lang.ClassLoader cl = getClassLoader();
	    if (!mPackageName.equals("android")) {
	        initializeJavaContextClassLoader();
	    }
	    // 创建AppContext
	    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
	    // newApplication
	    Application app = mActivityThread.mInstrumentation
			    .newApplication(cl, appClass, appContext);
	    // 将Application设置到AppContext
	    appContext.setOuterContext(app);
    
	    // 将Application添加到应用程序列表
	    mActivityThread.mAllApplications.add(app);
	    mApplication = app;
 
	    if (instrumentation != null) { // 此处instrumentation=null
	        instrumentation.callApplicationOnCreate(app);
	    }
 
	    // Rewrite the R 'constants' for all library apks.
	    SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
	    final int N = packageIdentifiers.size();
	    for (int i = 0; i < N; i++) {
	        final int id = packageIdentifiers.keyAt(i);
	        if (id == 0x01 || id == 0x7f) {
	            continue;
	        }
 
	        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
	    }
 
	    return app;
	}

## 11.Instrumentation
[ -> frameworks/base/core/java/android/app/Instrumentation.java ]

	public void onCreate(Bundle arguments) {
	}
 
	public void callApplicationOnCreate(Application app) {
	    app.onCreate();
	}

## 12.ActivityTaskManagerService.attachApplication
[ -> frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java ]

	// 启动Activity
	public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
	    synchronized (mGlobalLockWithoutBoost) {
	        return mRootActivityContainer.attachApplication(wpc);
	    }
	}

## 13.RootActivityContainer.attachApplication
[ -> frameworks/base/services/core/java/com/android/server/wm/RootActivityContainer.java ]

	boolean attachApplication(WindowProcessController app) throws RemoteException {
	    final String processName = app.mName;
	    boolean didSomething = false;
	    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
	        final ActivityDisplay display = mActivityDisplays.get(displayNdx);
	        final ActivityStack stack = display.getFocusedStack();
	        if (stack != null) {
	            stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
	            final ActivityRecord top = stack.topRunningActivityLocked();
	            final int size = mTmpActivityList.size();
	            for (int i = 0; i < size; i++) {
	                final ActivityRecord activity = mTmpActivityList.get(i);
	                if (activity.app == null && app.mUid == activity.info.applicationInfo.uid
			                && processName.equals(activity.processName)) {
			            // 启动Activity
	                    if (mStackSupervisor.realStartActivityLocked(activity, app,
			                    top == activity /* andResume */, true /* checkConfig */)) {
	                        didSomething = true;
	                    }
	                }
	            }
	        }
	    }
	    if (!didSomething) {
	        ensureActivitiesVisible(null, 0, false /* preserve_windows */);
	    }
	    return didSomething;
	}

关于ActivityStackSupervisor#realStartActivityLocked 本文暂不分析, 后续放到startActivity部分讲解.

## 14.总结



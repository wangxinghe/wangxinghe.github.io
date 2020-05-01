---
layout: post
comments: true
title: "WMS启动过程"
description: "WMS启动过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文主要介绍WMS服务的启动过程, 以及WMS的重要组成元素.

## 0.文件结构

	frameworks/base/services/java/com/android/server/SystemServer.java
	frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
	frameworks/base/services/core/java/com/android/server/DisplayThread.java
	frameworks/base/services/core/java/com/android/server/wm/WindowState.java
	frameworks/base/services/core/java/com/android/server/wm/AppWindowToken.java
	frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
	frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
	frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java
	frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
	frameworks/base/services/core/java/com/android/server/wm/WindowAnimator.java
	frameworks/base/services/core/java/com/android/server/wm/WindowSurfacePlacer.java
	frameworks/base/services/core/java/com/android/server/wm/TaskSnapshotController.java
	frameworks/base/services/core/java/com/android/server/wm/TaskPositioningController.java
	frameworks/base/services/core/java/com/android/server/wm/DragDropController.java

## 1.SystemServer
[ -> frameworks/base/services/java/com/android/server/SystemServer.java ]

### 1.1 main
	/**
	 * The main entry point from zygote.
	 */
	public static void main(String[] args) {
	    new SystemServer().run();
	}
 
	private void run() {
	    ...
	    // 初始化系统上下文
	    createSystemContext();
	    // 初始化SystemServiceManager
	    mSystemServiceManager = new SystemServiceManager(mSystemContext);
	    ...
	    // 启动相互依赖关系复杂的服务
	    startBootstrapServices();
	    // 启动相互独立的基本服务
	    startCoreServices();
	    // 启动其他服务
	    startOtherServices();
	    ...
	}

### 1.2 startOtherServices

	private void startOtherServices() {
	    // 启动WMS前, 需要先启动SensorService
	    ConcurrentUtils.waitForFutureNoInterrupt(mSensorServiceStart, START_SENSOR_SERVICE);
	    mSensorServiceStart = null;
	    // 见【第2.2节】
	    WindowManagerService wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore, new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
	    ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
	    ...
	    // 所有WMS相关的实体对象初始化完成
	    wm.onInitReady();
	    ...
	    // Display ready
	    wm.displayReady();
	    ...
	    // WMS ready
	    wm.systemReady();
	    ...
	    final WindowManagerService windowManagerF = wm;
	    // 启动SystemUIService服务
	    startSystemUi(context, windowManagerF);
	    ...
	}
 
	private static void startSystemUi(Context context, WindowManagerService windowManager) {
	    Intent intent = new Intent();
	    intent.setComponent(new ComponentName("com.android.systemui", "com.android.systemui.SystemUIService"));
	    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
	    context.startServiceAsUser(intent, UserHandle.SYSTEM);
	    // System UI已启动
	    windowManager.onSystemUiStarted();
	}

## 2.WindowManagerService
[ -> frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java ]

### 2.1 组成元素

	// ActivityTaskManager服务相关, 管理Activity和其容器(如task/stacks/displays)的系统服务
	final IActivityTaskManager mActivityTaskManager;
	final ActivityTaskManagerService mAtmService;
  
	// DisplayManager服务相关, 管理Display属性
	final DisplayManagerInternal mDisplayManagerInternal;
	final DisplayManager mDisplayManager;
	final DisplayWindowSettings mDisplayWindowSettings;
  
	// ActivityManager服务相关, 用于和四大组件交互
	final IActivityManager mActivityManager;
	final ActivityManagerInternal mAmInternal;
  
	// PowerManager服务相关, 管理电源状态
	PowerManager mPowerManager;
	PowerManagerInternal mPowerManagerInternal;
  
	// 输入管理服务
	final InputManagerService mInputManager;
	// 包管理本地系统服务
	final PackageManagerInternal mPmInternal;
  
	// 根窗口容器
	RootWindowContainer mRoot;
	// 提供UI相关行为的策略类, 其实现类为PhoneWindowManager
	WindowManagerPolicy mPolicy;
	// 在一个单独的task中执行动画和Surface操作的类
	final WindowAnimator mAnimator;
	// 用来确定Window和Surface位置的类
	final WindowSurfacePlacer mWindowPlacerLocked;
	// 任务快照管理器(当App不可见时, 会将Task的快照以Bitmap形式存在缓存中)
	final TaskSnapshotController mTaskSnapshotController;
	// Task定位控制器
	final TaskPositioningController mTaskPositioningController;
	// View的拖/拉操作控制器
	final DragDropController mDragDropController;
  
	// 当前活跃状态的Session连接队列(通常一个进程中包含一个Session, 用于和WindowManager交互)
	final ArraySet<Session> mSessions = new ArraySet<>();
	// <IBinder, WindowState>客户端Window token和服务端WindowState的映射
	final WindowHashMap mWindowMap = new WindowHashMap();
  
	// AppWindowToken是一个窗口容器类, 可以理解为正在显示Window的App或Activity的窗口令牌(继承于WindowToken)
	// replace超时的AppWindowToken令牌列表
	final ArrayList<AppWindowToken> mWindowReplacementTimeouts = new ArrayList<>();
	// WindowState表示服务端描述的Window
	final ArrayList<WindowState> mResizingWindows = new ArrayList<>();
	final ArrayList<WindowState> mPendingRemove = new ArrayList<>();
	final ArrayList<WindowState> mDestroySurface = new ArrayList<>();
	final ArrayList<WindowState> mDestroyPreservedSurface = new ArrayList<>();
	final ArrayList<WindowState> mForceRemoves = new ArrayList<>();
	ArrayList<WindowState> mWaitingForDrawn = new ArrayList<>();
	private ArrayList<WindowState> mHidingNonSystemOverlayWindows = new ArrayList<>();
	WindowState[] mPendingRemoveTmp = new WindowState[20];
    
	// 主线程Handler
	final H mH = new H();

### 2.2 启动过程

#### 2.2.1 main

	public static WindowManagerService main(final Context context, final InputManagerService im,
	        final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,
	        ActivityTaskManagerService atm) {
	    return main(context, im, showBootMsgs, onlyCore, policy, atm, SurfaceControl.Transaction::new);
	}
 
	public static WindowManagerService main(final Context context, final InputManagerService im,
	        final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,
	        ActivityTaskManagerService atm, TransactionFactory transactionFactory) {
	    DisplayThread.getHandler().runWithScissors(() ->
	            sInstance = new WindowManagerService(context, im, showBootMsgs, onlyCore, policy, atm, transactionFactory), 0);
	    return sInstance;
	}
 
#### 2.2.2 构造方法
 
	private WindowManagerService(Context context, InputManagerService inputManager,
	        boolean showBootMsgs, boolean onlyCore, WindowManagerPolicy policy,
	        ActivityTaskManagerService atm, TransactionFactory transactionFactory) {
	    installLock(this, INDEX_WINDOW);
	    mGlobalLock = atm.getGlobalLock();
	    mAtmService = atm;
	    mContext = context;
	    mAllowBootMessages = showBootMsgs;
	    mOnlyCore = onlyCore;
	    // 各种变量读取
	    mLimitedAlphaCompositing = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_sf_limitedAlpha);
	    mHasPermanentDpad = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_hasPermanentDpad);
	    mInTouchMode = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_defaultInTouchMode);
	    mDrawLockTimeoutMillis = context.getResources().getInteger(
	            com.android.internal.R.integer.config_drawLockTimeoutMillis);
	    mAllowAnimationsInLowPowerMode = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_allowAnimationsInLowPowerMode);
	    mMaxUiWidth = context.getResources().getInteger(
	            com.android.internal.R.integer.config_maxUiWidth);
	    mDisableTransitionAnimation = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_disableTransitionAnimation);
	    mPerDisplayFocusEnabled = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_perDisplayFocusEnabled);
	    mLowRamTaskSnapshotsAndRecents = context.getResources().getBoolean(
	            com.android.internal.R.bool.config_lowRamTaskSnapshotsAndRecents);
	    mInputManager = inputManager; // Must be before createDisplayContentLocked.
	    mDisplayManagerInternal = LocalServices.getService(DisplayManagerInternal.class);
	    // Display设置
	    mDisplayWindowSettings = new DisplayWindowSettings(this);
 
	    mTransactionFactory = transactionFactory;
	    mTransaction = mTransactionFactory.make();
	    // PhoneWindowManager(继承于WindowManagerPolicy, 用来提供UI相关的一些行为)
	    mPolicy = policy;
	    // 在一个单独的task中执行动画和Surface操作的类
	    mAnimator = new WindowAnimator(this);
	    // 根Window容器
	    mRoot = new RootWindowContainer(this);
 
	    // 用来确定Window和Surface的位置
	    mWindowPlacerLocked = new WindowSurfacePlacer(this);
	    // 任务快照管理器(当App不可见时, 会将Task的快照以Bitmap形式存在缓存中)
	    mTaskSnapshotController = new TaskSnapshotController(this);

	    LocalServices.addService(WindowManagerPolicy.class, mPolicy);
 
	    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
 
	    // Keyguard处理器
	    mKeyguardDisableHandler = KeyguardDisableHandler.create(mContext, mPolicy, mH);
 
	    // PowerManager是控制设备电池状态的管理器
	    mPowerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);
	    // PowerManagerInternal是PowerMananger的本地服务
	    mPowerManagerInternal = LocalServices.getService(PowerManagerInternal.class);
 
	    if (mPowerManagerInternal != null) {
	        mPowerManagerInternal.registerLowPowerModeObserver(
	                new PowerManagerInternal.LowPowerModeListener() {
	            @Override
	            public int getServiceType() {
	                return ServiceType.ANIMATION;
	            }
 
	            @Override
	            public void onLowPowerModeChanged(PowerSaveState result) {
	                synchronized (mGlobalLock) {
	                    // 低电量模式发生变化时, 需要调整对应的动画
	                    final boolean enabled = result.batterySaverEnabled;
	                    if (mAnimationsDisabled != enabled && !mAllowAnimationsInLowPowerMode) {
	                        mAnimationsDisabled = enabled;
	                        dispatchNewAnimatorScaleLocked(null);
	                    }
	                }
	            }
	        });
	        // 获取是否允许动画
	        mAnimationsDisabled = mPowerManagerInternal.getLowPowerState(ServiceType.ANIMATION).batterySaverEnabled;
	    }
	    mScreenFrozenLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "SCREEN_FROZEN");
	    mScreenFrozenLock.setReferenceCounted(false);
 
	    // 获取IActivity.Stub.Proxy(new BinderProxy())
	    mActivityManager = ActivityManager.getService();
	    // 获取IActivityTaskManager.Stub.Proxy
	    mActivityTaskManager = ActivityTaskManager.getService();
	    // ActivityManagerInternal是ActivityManager的本地服务
	    mAmInternal = LocalServices.getService(ActivityManagerInternal.class);
	    // ActivityTaskManagerInternal是ActivityTaskManager的本地服务
	    mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
	    mAppOps = (AppOpsManager)context.getSystemService(Context.APP_OPS_SERVICE);
	    AppOpsManager.OnOpChangedInternalListener opListener =
	            new AppOpsManager.OnOpChangedInternalListener() {
	                @Override 
	                public void onOpChanged(int op, String packageName) {
	                    updateAppOpsState();
	                }
	            };
	    mAppOps.startWatchingMode(OP_SYSTEM_ALERT_WINDOW, null, opListener);
	    mAppOps.startWatchingMode(AppOpsManager.OP_TOAST_WINDOW, null, opListener);
 
	    // PackageManagerInternal是PackageManager的本地服务
	    mPmInternal = LocalServices.getService(PackageManagerInternal.class);
	    // 注册Package suspend/unsuspend广播
	    final IntentFilter suspendPackagesFilter = new IntentFilter();
	    suspendPackagesFilter.addAction(Intent.ACTION_PACKAGES_SUSPENDED);
	    suspendPackagesFilter.addAction(Intent.ACTION_PACKAGES_UNSUSPENDED);
	    context.registerReceiverAsUser(new BroadcastReceiver() {
	        @Override
	        public void onReceive(Context context, Intent intent) {
	            final String[] affectedPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
	            final boolean suspended = Intent.ACTION_PACKAGES_SUSPENDED.equals(intent.getAction());
	            updateHiddenWhileSuspendedState(new ArraySet<>(Arrays.asList(affectedPackages)), suspended);
	        }
	    }, UserHandle.ALL, suspendPackagesFilter, null, null);
 
	    // 获取并设置window scale设置
	    final ContentResolver resolver = context.getContentResolver();
	    mWindowAnimationScaleSetting = Settings.Global.getFloat(resolver,
			    Settings.Global.WINDOW_ANIMATION_SCALE, mWindowAnimationScaleSetting);
	    mTransitionAnimationScaleSetting = Settings.Global.getFloat(resolver,
			    Settings.Global.TRANSITION_ANIMATION_SCALE,
					    context.getResources().getFloat(
						    R.dimen.config_appTransitionAnimationDurationScaleDefault));
	    setAnimatorDurationScale(Settings.Global.getFloat(resolver,
			    Settings.Global.ANIMATOR_DURATION_SCALE, mAnimatorDurationScaleSetting));
		mForceDesktopModeOnExternalDisplays = Settings.Global.getInt(resolver,
			    DEVELOPMENT_FORCE_DESKTOP_MODE_ON_EXTERNAL_DISPLAYS, 0) != 0;
 
	    // 注册广播, 当DevicePolicyManager状态发生变化时设置keyguard属性是否可用
	    IntentFilter filter = new IntentFilter();
	    filter.addAction(ACTION_DEVICE_POLICY_MANAGER_STATE_CHANGED);
	    mContext.registerReceiverAsUser(mBroadcastReceiver, UserHandle.ALL, filter, null, null);
 
	    mLatencyTracker = LatencyTracker.getInstance(context);
 
	    mSettingsObserver = new SettingsObserver();
 
	    mHoldingScreenWakeLock = mPowerManager.newWakeLock(
			    PowerManager.SCREEN_BRIGHT_WAKE_LOCK | PowerManager.ON_AFTER_RELEASE, TAG_WM);
	    mHoldingScreenWakeLock.setReferenceCounted(false);
 
	    mSurfaceAnimationRunner = new SurfaceAnimationRunner(mPowerManagerInternal);
 
	    mAllowTheaterModeWakeFromLayout = context.getResources().getBoolean(
			    com.android.internal.R.bool.config_allowTheaterModeWakeFromWindowLayout);
 
	    // Task定位控制器
	    mTaskPositioningController = new TaskPositioningController(this, mInputManager,
			    mActivityTaskManager, mH.getLooper());
	    // View的拖/拉操作控制器
	    mDragDropController = new DragDropController(this, mH.getLooper());
 
	    mSystemGestureExclusionLimitDp = Math.max(MIN_GESTURE_EXCLUSION_LIMIT_DP,
	            DeviceConfig.getInt(DeviceConfig.NAMESPACE_WINDOW_MANAGER,
	                    KEY_SYSTEM_GESTURE_EXCLUSION_LIMIT_DP, 0));
	    mSystemGestureExcludedByPreQStickyImmersive =
	            DeviceConfig.getBoolean(DeviceConfig.NAMESPACE_WINDOW_MANAGER,
	                    KEY_SYSTEM_GESTURES_EXCLUDED_BY_PRE_Q_STICKY_IMMERSIVE, false);
	    DeviceConfig.addOnPropertiesChangedListener(DeviceConfig.NAMESPACE_WINDOW_MANAGER,
	            new HandlerExecutor(mH), properties -> {
	                synchronized (mGlobalLock) {
	                    final int exclusionLimitDp = Math.max(MIN_GESTURE_EXCLUSION_LIMIT_DP,
	                            properties.getInt(KEY_SYSTEM_GESTURE_EXCLUSION_LIMIT_DP, 0));
	                    final boolean excludedByPreQSticky = DeviceConfig.getBoolean(
	                            DeviceConfig.NAMESPACE_WINDOW_MANAGER,
	                            KEY_SYSTEM_GESTURES_EXCLUDED_BY_PRE_Q_STICKY_IMMERSIVE, false);
	                    if (mSystemGestureExcludedByPreQStickyImmersive != excludedByPreQSticky
	                            || mSystemGestureExclusionLimitDp != exclusionLimitDp) {
	                        mSystemGestureExclusionLimitDp = exclusionLimitDp;
	                        mSystemGestureExcludedByPreQStickyImmersive = excludedByPreQSticky;
	                        mRoot.forAllDisplays(DisplayContent::updateSystemGestureExclusionLimit);
	                    }
	                }
	            });
 
	    // 注册WindowManager的本地服务WindowManagerInternal
	    LocalServices.addService(WindowManagerInternal.class, new LocalService());
	}

#### 2.2.3 onInitReady

	public void onInitReady() {
	    // 初始化PhoneWindowManager(继承于WindowManagerPolicy, 用来提供UI相关的一些行为)
	    initPolicy();
 
	    // 添加Watchdog monitor
	    Watchdog.getInstance().addMonitor(this);
 
	    // 调用SurfaceControl.openTransaction(), 启动一个事务
	    openSurfaceTransaction();
	    // 创建水印
	    createWatermarkInTransaction();
	    // 结束事务
		closeSurfaceTransaction("createWatermarkInTransaction");
 
	    // 显示模拟器显示层
	    showEmulatorDisplayOverlayIfNeeded();
	}
 
#### 2.2.4 displayReady
 
	public void displayReady() {
	    synchronized (mGlobalLock) {
	        // 设置RootWindowContainer的Window列表的最大宽度
	        if (mMaxUiWidth > 0) {
	            mRoot.forAllDisplays(displayContent -> displayContent.setMaxUiWidth(mMaxUiWidth));
	        }
	        final boolean changed = applyForcedPropertiesForDefaultDisplay();
	        mAnimator.ready();
	        mDisplayReady = true;
	        if (changed) {
	            // 重新配置DiaplayContent属性
	            reconfigureDisplayLocked(getDefaultDisplayContentLocked());
	        }
	        mIsTouchDevice = mContext.getPackageManager().hasSystemFeature(PackageManager.FEATURE_TOUCHSCREEN);
	    }
 
	    // 1.修改当前configuration 2.确保当前Activity正在运行当前configuration
	    mActivityTaskManager.updateConfiguration(null);
	    // 更新CircularDisplayMask
	    updateCircularDisplayMaskIfNeeded();
	}

#### 2.2.5 systemReady
 
	public void systemReady() {
	    mSystemReady = true;
	    mPolicy.systemReady();
	    mRoot.forAllDisplayPolicies(DisplayPolicy::systemReady);
	    mTaskSnapshotController.systemReady();
	    // 是否支持色域
	    mHasWideColorGamutSupport = queryWideColorGamutSupport();
	    // 是否支持HDR渲染
	    mHasHdrSupport = queryHdrSupport();
	    UiThread.getHandler().post(mSettingsObserver::updateSystemUiSettings);
	    UiThread.getHandler().post(mSettingsObserver::updatePointerLocation);
	    // 获取IVrManager.Stub.Proxy, 并注册状态变化listener
	    IVrManager vrManager = IVrManager.Stub.asInterface(ServiceManager.getService(Context.VR_SERVICE));
	    if (vrManager != null) {
	        final boolean vrModeEnabled = vrManager.getVrModeState();
	        synchronized (mGlobalLock) {
	            vrManager.registerListener(mVrStateCallbacks);
	            if (vrModeEnabled) {
	                mVrModeEnabled = vrModeEnabled;
	                mVrStateCallbacks.onVrStateChanged(vrModeEnabled);
	            }
	        }
	    }
	}

### 2.3 主要功能

WMS主要用于窗口的添加和移除操作, 其对应的方法是addWindow和removeWindow.  
关于窗口的添加和删除过程, 放到后面文章去分析.

    public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
            InsetsState outInsetsState)
            
	void removeWindow(Session session, IWindow client)


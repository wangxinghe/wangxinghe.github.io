---
layout: post
comments: true
title: "创建和添加Window - SystemUI&Activity"
description: "创建和添加Window - SystemUI&Activity"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文主要分析内容:  
1.SystemUI(如StatusBar)和Activity中Window的创建和添加.  
2.Activity/PhoneWindow/DecorView/StatusBar/ViewRootImpl之间的关系.  

限于篇幅原因, WindowManager.addView具体的添加过程在下一篇文章分析.  

## 0.文件结构

	frameworks/base/packages/SystemUI/src/com/android/systemui/SystemBars.java
	frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java

	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/app/Activity.java
	frameworks/base/core/java/android/view/WindowManagerImpl.java
	frameworks/base/core/java/android/view/Window.java
	frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java
	frameworks/base/core/java/com/android/internal/policy/DecorView.java

	frameworks/base/packages/SystemUI/AndroidManifest.xml
	frameworks/base/packages/SystemUI/res/values/config.xml
	frameworks/base/packages/SystemUI/res/layout/super_status_bar.xml
	frameworks/base/packages/SystemUI/res/layout/status_bar.xml
	frameworks/base/core/res/res/layout/screen_simple.xml

## 1.时序图

TODO

## 2.SystemUI

SystemUI包括很多子类, 状态栏StatusBar是最常见的一种. 本文以StatusBar为例进行分析.

SystemUI运行的进程是SystemUIApplication.

### 2.1 SystemUIApplication
[ -> frameworks/base/packages/SystemUI/AndroidManifest.xml]

	<application
	    android:name=".SystemUIApplication"
	    android:persistent="true"
	    android:allowClearUserData="false"
	    android:allowBackup="false"
	    android:hardwareAccelerated="true"
	    android:label="@string/app_label"
	    android:icon="@drawable/icon"
	    android:process="com.android.systemui"
	    android:supportsRtl="true"
	    android:theme="@style/Theme.SystemUI"
	    android:defaultToDeviceProtectedStorage="true"
	    android:directBootAware="true"
	    android:appComponentFactory="androidx.core.app.CoreComponentFactory">
	    ...
	    <service android:name="SystemUIService" android:exported="true"/>
	    ...
	</application>

SystemUI调用链：  
`SystemUIApplication.onCreate -> startSecondaryUserServicesIfNeeded -> startServicesIfNeeded -> mServices[i].start() -> SystemBars.start`

### 2.2 SystemBars.start
[ -> frameworks/base/packages/SystemUI/src/com/android/systemui/SystemBars.java ]

	public class SystemBars extends SystemUI {
	    private SystemUI mStatusBar;
 
	    @Override
	    public void start() {
	        createStatusBarFromConfig();
	    }
 
	    private void createStatusBarFromConfig() {
	        // 从config.xml中读取className-"com.android.systemui.statusbar.phone.StatusBar"
	        final String clsName = mContext.getString(R.string.config_statusBarComponent);
	        // 反射创建StatusBar对象
	        Class<?> cls = mContext.getClassLoader().loadClass(clsName);
	        mStatusBar = (SystemUI) cls.newInstance();
	        mStatusBar.mContext = mContext;
	        mStatusBar.mComponents = mComponents;
	        // 将StatusBar注入系统UI的根组件
	        if (mStatusBar instanceof StatusBar) {
	            SystemUIFactory.getInstance().getRootComponent()
	                    .getStatusBarInjector()
	                    .createStatusBar((StatusBar) mStatusBar);
	        }
	        // 创建状态栏View, 并将其添加到WindowManager
	        mStatusBar.start();
	    }
	}

SystemBars#start做的事情:   
1.读取配置文件中的属性config_statusBarComponent, 得到字符串className为StatusBar的字符串.  
[ -> frameworks/base/packages/SystemUI/res/values/config.xml ]

	<string name="config_statusBarComponent" translatable="false">com.android.systemui.statusbar.phone.StatusBar</string>

2.利用反射机制创建StatusBar对象, 创建状态栏View并将其添加到WindowManager.  

### 2.3 StatusBar
[ -> frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java ]

#### 2.3.1 start

	public class StatusBar extends SystemUI implements DemoMode,
	        ActivityStarter, OnUnlockMethodChangedListener,
	        OnHeadsUpChangedListener, CommandQueue.Callbacks, ZenModeController.Callback,
	        ColorExtractor.OnColorsChangedListener, ConfigurationListener,
	        StatusBarStateController.StateListener, ShadeController,
	        ActivityLaunchAnimator.Callback, AmbientPulseManager.OnAmbientChangedListener,
	        AppOpsController.Callback {
	    protected WindowManager mWindowManager;
	    protected IWindowManager mWindowManagerService;
	    // 状态栏最外层View
	    protected StatusBarWindowView mStatusBarWindow;
		// CollapsedStatusBarFragment的根View(即StatusBarWindowView的childView)
	    protected PhoneStatusBarView mStatusBarView;
 
	    @Override
	    public void start() {
	        ...
	        // 获取WindowManagerImpl
	        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
	        // 获取IWindowManager.Stub.Proxy
	        mWindowManagerService = WindowManagerGlobal.getWindowManagerService();
	        ...
	        // 创建状态栏View, 并将其添加到WindowManager
	        createAndAddWindows(result);
	        ...
	    }
	    ...
	}

#### 2.3.2 createAndAddWindows

	// 创建状态栏View, 并将其添加到WindowManager
	public void createAndAddWindows(@Nullable RegisterStatusBarResult result) {
	    // 根据布局文件super_status_bar.xml创建StatusBarWindowView. 参考【第2.3.3节】
	    makeStatusBarView(result);
	    mStatusBarWindowController = Dependency.get(StatusBarWindowController.class);
	    // 参考
	    mStatusBarWindowController.add(mStatusBarWindow, getStatusBarHeight());
	}

    // 获取状态栏高度, App就是借鉴这个方法
    public int getStatusBarHeight() {
        if (mNaturalBarHeight < 0) {
            final Resources res = mContext.getResources();
            mNaturalBarHeight = res.getDimensionPixelSize(com.android.internal.R.dimen.status_bar_height);
        }
        return mNaturalBarHeight;
    }
 
 这里要提一下:  
 getStatusBarHeight这个方法, App开发中获取状态栏高度就是借鉴的这个方法, 只不过是反射获取该属性.  
 基本思路就是看源码中StatusBar对应的布局, 读取其高度属性.
 
#### 2.3.3 makeStatusBarView
 
    protected void makeStatusBarView(@Nullable RegisterStatusBarResult result) {
        ...
        // 根据布局文件super_status_bar.xml创建StatusBarWindowView
        inflateStatusBarWindow(context);
        mStatusBarWindow.setService(this);
        mStatusBarWindow.setOnTouchListener(getStatusBarWindowTouchListener());
        ...
        // status_bar_container位置填充CollapsedStatusBarFragment
        // CollapsedStatusBarFragment对应的布局文件为status_bar.xml
        FragmentHostManager.get(mStatusBarWindow).addTagListener(CollapsedStatusBarFragment.TAG, (tag, fragment) -> {
                // Fragment创建完成时, 初始化
                CollapsedStatusBarFragment statusBarFragment = (CollapsedStatusBarFragment) fragment;
                statusBarFragment.initNotificationIconArea(mNotificationIconAreaController);
                PhoneStatusBarView oldStatusBarView = mStatusBarView;
                mStatusBarView = (PhoneStatusBarView) fragment.getView();
                mStatusBarView.setBar(this);
                mStatusBarView.setPanel(mNotificationPanel);
                mStatusBarView.setScrimController(mScrimController);
                mStatusBarView.setBouncerShowing(mBouncerShowing);
                if (oldStatusBarView != null) {
                    float fraction = oldStatusBarView.getExpansionFraction();
                    boolean expanded = oldStatusBarView.isExpanded();
                    mStatusBarView.panelExpansionChanged(fraction, expanded);
                }
                ...
                mStatusBarWindow.setStatusBarView(mStatusBarView);
                ...
            }).getFragmentManager()
            .beginTransaction()
            .replace(R.id.status_bar_container, new CollapsedStatusBarFragment(), CollapsedStatusBarFragment.TAG)
            .commit();
        ...
        // 创建导航栏(仅针对CarStatusBar)
        createNavigationBar(result);
        ...
    }
 
#### 2.3.4 inflateStatusBarWindow
 
    protected void inflateStatusBarWindow(Context context) {
        // 从super_status_bar.xml布局文件创建StatusBarWindowView
        mStatusBarWindow = (StatusBarWindowView) mInjectionInflater.injectable(
                LayoutInflater.from(context)).inflate(R.layout.super_status_bar, null);
    }

#### 2.3.5 StatusBarWindowController.start  
[ -> frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarWindowController.java ]

	// Adds the status bar view to the window manager.
	public void add(ViewGroup statusBarView, int barHeight) {
	    mLp = new WindowManager.LayoutParams(
	            ViewGroup.LayoutParams.MATCH_PARENT,
	            barHeight,
	            WindowManager.LayoutParams.TYPE_STATUS_BAR,
	            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
	                    | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
	                    | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
	                    | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
	                    | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
	            PixelFormat.TRANSLUCENT);
	    mLp.token = new Binder();
	    mLp.gravity = Gravity.TOP;
	    mLp.softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
	    mLp.setTitle("StatusBar");
	    mLp.packageName = mContext.getPackageName();
	    mLp.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
	    mStatusBarView = statusBarView;
	    mBarHeight = barHeight;
	    mWindowManager.addView(mStatusBarView, mLp);
	    mLpChanged.copyFrom(mLp);
	    onThemeChanged();
	}

总结StatusBar#start做的事情:  
一句话总结就是创建状态栏View并将其添加到WindowManager.  

具体过程为:  
1.根据布局文件super_status_bar.xml创建StatusBarWindowView  
2.上述布局文件中id为status_bar_container的位置填充CollapsedStatusBarFragment  
3.创建导航栏(仅针对CarStatusBar)  
4.StatusBarWindowView添加到WindowManager.  

(1) 状态栏相关的布局文件为:  
[ -> frameworks/base/packages/SystemUI/res/layout/super_status_bar.xml ]

	<com.android.systemui.statusbar.phone.StatusBarWindowView
	    xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:sysui="http://schemas.android.com/apk/res-auto"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:fitsSystemWindows="true">
 
	    <com.android.systemui.statusbar.BackDropView
            android:id="@+id/backdrop"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="gone"
            sysui:ignoreRightInset="true">
	        
	        <ImageView android:id="@+id/backdrop_back"
                   android:layout_width="match_parent"
                   android:scaleType="centerCrop"
                   android:layout_height="match_parent" />
	        
	        <ImageView android:id="@+id/backdrop_front"
                   android:layout_width="match_parent"
                   android:layout_height="match_parent"
                   android:scaleType="centerCrop"
                   android:visibility="invisible" />
	    </com.android.systemui.statusbar.BackDropView>
 
	    <com.android.systemui.statusbar.ScrimView
	        android:id="@+id/scrim_behind"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:importantForAccessibility="no"
	        sysui:ignoreRightInset="true"/>
 
	    <FrameLayout
	        android:id="@+id/status_bar_container"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content" />
 
	    <include layout="@layout/status_bar_expanded"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:visibility="invisible" />
 
	    <include layout="@layout/brightness_mirror" />
 
	    <com.android.systemui.statusbar.ScrimView
	        android:id="@+id/scrim_in_front"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:importantForAccessibility="no"
	        sysui:ignoreRightInset="true"/>
 
	    <LinearLayout
	        android:id="@+id/lock_icon_container"
	        android:orientation="vertical"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_marginTop="@dimen/status_bar_height"
	        android:layout_gravity="top|center_horizontal">
	        
	        <com.android.systemui.statusbar.phone.LockIcon
	            android:id="@+id/lock_icon"
	            android:layout_width="@dimen/keyguard_lock_width"
	            android:layout_height="@dimen/keyguard_lock_height"
	            android:layout_gravity="center_horizontal"
	            android:layout_marginTop="@dimen/keyguard_lock_padding"
	            android:contentDescription="@string/accessibility_unlock_button"
	            android:src="@*android:drawable/ic_lock"
	            android:scaleType="center" />
	        
	        <com.android.keyguard.KeyguardMessageArea
	            android:id="@+id/keyguard_message_area"
	            style="@style/Keyguard.TextView"
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:layout_marginTop="@dimen/keyguard_lock_padding"
	            android:gravity="center"
	            android:singleLine="true"
	            android:ellipsize="marquee"
	            android:focusable="true" />
	    </LinearLayout>
	</com.android.systemui.statusbar.phone.StatusBarWindowView>

(2) CollapsedStatusBarFragment对应的布局文件为status_bar.xml  
[ -> frameworks/base/packages/SystemUI/res/layout/status_bar.xml ]

	<com.android.systemui.statusbar.phone.PhoneStatusBarView
	    xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:systemui="http://schemas.android.com/apk/res/com.android.systemui"
	    android:layout_width="match_parent"
	    android:layout_height="@dimen/status_bar_height"
	    android:id="@+id/status_bar"
	    android:background="@drawable/system_bar_background"
	    android:orientation="vertical"
	    android:focusable="false"
	    android:descendantFocusability="afterDescendants"
	    android:accessibilityPaneTitle="@string/status_bar">
 
	    <ImageView
	        android:id="@+id/notification_lights_out"
	        android:layout_width="@dimen/status_bar_icon_size"
	        android:layout_height="match_parent"
	        android:paddingStart="@dimen/status_bar_padding_start"
	        android:paddingBottom="2dip"
	        android:src="@drawable/ic_sysbar_lights_out_dot_small"
	        android:scaleType="center"
	        android:visibility="gone"/>
 
	    <LinearLayout android:id="@+id/status_bar_contents"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:paddingStart="@dimen/status_bar_padding_start"
	        android:paddingEnd="@dimen/status_bar_padding_end"
	        android:paddingTop="@dimen/status_bar_padding_top"
	        android:orientation="horizontal">
	        
	        <FrameLayout
	            android:layout_height="match_parent"
	            android:layout_width="0dp"
	            android:layout_weight="1">
 
	            <include layout="@layout/heads_up_status_bar_layout" />
 
	            <LinearLayout
	                android:id="@+id/status_bar_left_side"
	                android:layout_height="match_parent"
	                android:layout_width="match_parent"
	                android:clipChildren="false">
 
	                <ViewStub
	                    android:id="@+id/operator_name"
	                    android:layout_width="wrap_content"
	                    android:layout_height="match_parent"
	                    android:layout="@layout/operator_name" />
 
	                <com.android.systemui.statusbar.policy.Clock
	                    android:id="@+id/clock"
	                    android:layout_width="wrap_content"
	                    android:layout_height="match_parent"
	                    android:textAppearance="@style/TextAppearance.StatusBar.Clock"
	                    android:singleLine="true"
	                    android:paddingStart="@dimen/status_bar_left_clock_starting_padding"
	                    android:paddingEnd="@dimen/status_bar_left_clock_end_padding"
	                    android:gravity="center_vertical|start"/>
 
	                <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
	                    android:id="@+id/notification_icon_area"
	                    android:layout_width="0dp"
	                    android:layout_height="match_parent"
	                    android:layout_weight="1"
	                    android:orientation="horizontal"
	                    android:clipChildren="false"/>
	            </LinearLayout>
	        </FrameLayout>
 
	        <android.widget.Space
	            android:id="@+id/cutout_space_view"
	            android:layout_width="0dp"
	            android:layout_height="match_parent"
	            android:gravity="center_horizontal|center_vertical"/>
 
	        <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
	            android:id="@+id/centered_icon_area"
	            android:layout_width="wrap_content"
	            android:layout_height="match_parent"
	            android:orientation="horizontal"
	            android:clipChildren="false"
	            android:gravity="center_horizontal|center_vertical"/>
 
	        <com.android.keyguard.AlphaOptimizedLinearLayout android:id="@+id/system_icon_area"
	            android:layout_width="0dp"
	            android:layout_height="match_parent"
	            android:layout_weight="1"
	            android:orientation="horizontal"
	            android:gravity="center_vertical|end">
 
	            <include layout="@layout/system_icons" />
	        </com.android.keyguard.AlphaOptimizedLinearLayout>
	    </LinearLayout>
 
	    <ViewStub
	        android:id="@+id/emergency_cryptkeeper_text"
	        android:layout_width="wrap_content"
	        android:layout_height="match_parent"
	        android:layout="@layout/emergency_cryptkeeper_text"/>
	</com.android.systemui.statusbar.phone.PhoneStatusBarView>

## 3.Activity  
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 3.1 ActivityThread.handleLaunchActivity

    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
        final Activity a = performLaunchActivity(r, customIntent);
        ...
        return a;
    }
 
### 3.2 ActivityThread.performLaunchActivity
 
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo, Context.CONTEXT_INCLUDE_CODE);
        }
 
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }
 
        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName, r.activityInfo.targetActivity);
        }
 
        // 创建Activity实例
        ContextImpl appContext = createBaseContextForActivity(r);
        java.lang.ClassLoader cl = appContext.getClassLoader();
        Activity activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
 
        try {
            // 创建Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
 
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
            // 将Activity attach到Application
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
            // 调用Activity.onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            r.activity = activity;
        }
        r.setState(ON_CREATE);
 
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);
        }
 
        return activity;
    }

### 3.3 Activity.attach  
[ -> frameworks/base/core/java/android/app/Activity.java ]
 
	final void attach(Context context, ActivityThread aThread,
	        Instrumentation instr, IBinder token, int ident,
	        Application application, Intent intent, ActivityInfo info,
	        CharSequence title, Activity parent, String id,
	        NonConfigurationInstances lastNonConfigurationInstances,
	        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
	        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
	    attachBaseContext(context);
 
	    mFragments.attachHost(null /*parent*/);
	    // 创建PhoneWindow
	    mWindow = new PhoneWindow(this, window, activityConfigCallback);
	    mWindow.setWindowControllerCallback(this);
	    mWindow.setCallback(this);
	    mWindow.setOnWindowDismissedCallback(this);
	    mWindow.getLayoutInflater().setPrivateFactory(this);
	    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
	        mWindow.setSoftInputMode(info.softInputMode);
	    }
	    if (info.uiOptions != 0) {
	        mWindow.setUiOptions(info.uiOptions);
	    }
	    mUiThread = Thread.currentThread();
 
	    mMainThread = aThread;
	    mInstrumentation = instr;
	    mToken = token;
	    mAssistToken = assistToken;
	    mIdent = ident;
	    mApplication = application;
	    mIntent = intent;
	    mReferrer = referrer;
	    mComponent = intent.getComponent();
	    mActivityInfo = info;
	    mTitle = title;
	    mParent = parent;
	    mEmbeddedID = id;
	    mLastNonConfigurationInstances = lastNonConfigurationInstances;
	    if (voiceInteractor != null) {
	        if (lastNonConfigurationInstances != null) {
	            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
	        } else {
	            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this, Looper.myLooper());
	        }
	    }
 
	    // 设置WindowManagerImpl
	    mWindow.setWindowManager(
	            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
	            mToken, mComponent.flattenToString(),
	            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
	    // 设置父Window
	    if (mParent != null) {
	        mWindow.setContainer(mParent.getWindow());
	    }
 
	    // 获取本地创建的WindowManagerImpl(这个和上面setWindowManager设置进来不是同一个实例)
	    mWindowManager = mWindow.getWindowManager();
	    mCurrentConfig = config;
 
	    mWindow.setColorMode(info.colorMode);
 
	    setAutofillOptions(application.getAutofillOptions());
	    setContentCaptureOptions(application.getContentCaptureOptions());
	}

### 3.4 ActivityThread.handleResumeActivity

	@Override
	public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
	    unscheduleGcIdler();
	    mSomeActivitiesChanged = true;
 
	    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);   
	    final Activity a = r.activity;
 
	    final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
 
	    // 当Window还未添加到WindowManager, 且还未finish当前Activity或未启动新Activity时, 需要先添加到WindowManager
	    boolean willBeVisible = !a.mStartedActivity;
	    if (!willBeVisible) {
	        willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(a.getActivityToken());      
	    }
	    if (r.window == null && !a.mFinished && willBeVisible) {
	        // 拿到PhoneWindow
	        r.window = r.activity.getWindow();
	        // 拿到DecorView
	        View decor = r.window.getDecorView();
	        // 设置DecorView不可见
	        decor.setVisibility(View.INVISIBLE);
	        // 获取本地创建的WindowManagerImpl
	        ViewManager wm = a.getWindowManager();
	        // 设置Window各属性(如type为TYPE_BASE_APPLICATION)
	        WindowManager.LayoutParams l = r.window.getAttributes();
	        a.mDecor = decor;
	        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
	        l.softInputMode |= forwardBit;
	        if (r.mPreserveWindow) {
	            a.mWindowAdded = true;
	            r.mPreserveWindow = false;
	            // 获取DecorView的ViewPootImpl(每个View都有一个ViewRootImpl)
	            ViewRootImpl impl = decor.getViewRootImpl();
	            // 通知子View已经被重建(DecorView还是旧实例, Activity是新实例, 因此需要更新callbacks)
	            // 1.通常情况下, ViewRootImpl通过WindowManagerImpl#addView->ViewRootImpl#setView设置Activity的callbacks回调
	            // 2.但是如果DecorView复用时, 需要主动告诉ViewRootImpl callbacks可能发生变化
	            if (impl != null) {
	                impl.notifyChildRebuilt();
	            }
	        }
	        // 当Activity可见时, 添加DecorView到WindowManagerImpl或回调Window属性变化方法
	        if (a.mVisibleFromClient) {
	            if (!a.mWindowAdded) {
	                a.mWindowAdded = true;
	                wm.addView(decor, l);
	            } else {
	                a.onWindowAttributesChanged(l);
	            }
	        }
	    } else if (!willBeVisible) {
	        r.hideForNow = true;
	    }
 
	    // Get rid of anything left hanging around.
	    cleanUpPendingRemoveWindows(r, false /* force */);
 
	    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
	        // 调用Activity.onConfigurationChanged方法
	        if (r.newConfig != null) {
	            performConfigurationChangedForActivity(r, r.newConfig);
	            r.newConfig = null;
	        }
	        // 更新DecorView的LayoutParams
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
	        // 设置Activity可见
	        if (r.activity.mVisibleFromClient) {
	            r.activity.makeVisible();
	        }
	    }
 
	    r.nextIdle = mNewActivities;
	    mNewActivities = r;
	    Looper.myQueue().addIdleHandler(new Idler());
	}

在Activity创建过程中, 会创建和添加PhoneWindow.  

Activity中PhoneWindow的创建和添加过程:  
1.handleLaunchActivity过程  
(1)  创建Activity实例和Application实例, 调用Activity#attach将Activity attach到Application.  
(2)  在Activity中创建PhoneWindow, PhoneWindow又创建DecorView. 设置WindowManagerImpl到PhoneWindow.  
2.handleResumeActivity过程  
(1)  PhoneWindow中的DecorView添加到WindowManager  
(2)  设置DecorView可见  

## 4.PhoneWindow详解  


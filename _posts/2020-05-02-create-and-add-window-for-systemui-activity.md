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

## 2.系统窗口

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
	    // 将StatusBarWindowView添加到WindowManager. 参考【第2.4节】
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

### 2.4 StatusBarWindowController.start  
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

## 3.应用窗口    
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

### 3.1 ActivityThread

#### 3.1.1 handleLaunchActivity

    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        ...
        final Activity a = performLaunchActivity(r, customIntent);
        ...
        return a;
    }
 
#### 3.1.2 performLaunchActivity

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
            // 调用Activity.onCreate方法. 参考【第3.2.2节】
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

### 3.2 Activity

#### 3.2.1 attach  
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

#### 3.2.2 onCreate

在App开发过程中, Activity#onCreate方法会调用setContentView(layoutResID).  
那么我们来看下setContentView.  

#### 3.2.3 setContentView

    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

### 3.3 PhoneWindow.setContentView

    @Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
            transitionTo(newScene);
        } else {
	        // 将布局layoutResID添加到DecorView中id为R.id.content的位置
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
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
(3)  调用Activity#onCreate -> setContentView(layoutResID) 将layoutResId添加到DecorView中R.id.content位置.  
2.handleResumeActivity过程  
(1)  PhoneWindow中的DecorView添加到WindowManager  
(2)  设置DecorView可见  

## 4.PhoneWindow&DecorView详解  

第3节已经分析了Activity中PhoneWindow的创建和添加过程.  
在Activity#onCreate中会调用PhoneWindow#installDecor创建DecorView和子View.  

这一节来具体分析下PhoneWindow和DecorView的组成.  

### 4.1 PhoneWindow  
[ -> frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java ]

#### 4.1.1 installDecor

	// 创建DecorView及其子View
	private void installDecor() {
	    mForceDecorInstall = false;
	    if (mDecor == null) {
	        // 创建DecorView
	        mDecor = generateDecor(-1);
	        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
	        mDecor.setIsRootNamespace(true);
	        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
	            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
	        }
	    } else {
	        mDecor.setWindow(this);
	    }
	    if (mContentParent == null) {
	        // 根据设置的Window相关属性, 设置PhoneWindow特性. 给PhoneWindow的根DecorView添加子View, 并返回ContentView
	        mContentParent = generateLayout(mDecor);
 
	        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
	        mDecor.makeOptionalFitsSystemWindows();
 
	        // 对于R.layout.screen_simple没有该元素, 但对于R.layout.screen_action_bar包含该元素
	        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(R.id.decor_content_parent);
 
	        // 如果根View为ActionBarOverlayLayout, 则设置ActionBarOverlayLayout的title/icon/logo/menu等
	        if (decorContentParent != null) {
	            mDecorContentParent = decorContentParent;
	            mDecorContentParent.setWindowCallback(getCallback());
	            if (mDecorContentParent.getTitle() == null) {
	                mDecorContentParent.setWindowTitle(mTitle);
	            }
 
	            final int localFeatures = getLocalFeatures();
	            for (int i = 0; i < FEATURE_MAX; i++) {
	                if ((localFeatures & (1 << i)) != 0) {
	                    mDecorContentParent.initFeature(i);
	                }
	            }
 
	            mDecorContentParent.setUiOptions(mUiOptions);
 
	            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
	                    (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
	                mDecorContentParent.setIcon(mIconRes);
	            } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
	                    mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                mDecorContentParent.setIcon(getContext().getPackageManager().getDefaultActivityIcon());
	                mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
	            }
	            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
	                    (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
	                mDecorContentParent.setLogo(mLogoRes);
	            }
 
	            PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
	            if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
	                invalidatePanelMenu(FEATURE_ACTION_BAR);
	            }
	        } else {
	            // 如果有title元素, 更新title
	            mTitleView = findViewById(R.id.title);
	            if (mTitleView != null) {
	                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
	                    final View titleContainer = findViewById(R.id.title_container);
	                    if (titleContainer != null) {
	                        titleContainer.setVisibility(View.GONE);
	                    } else {
	                        mTitleView.setVisibility(View.GONE);
	                    }
	                    mContentParent.setForeground(null);
	                } else {
	                    mTitleView.setText(mTitle);
	                }
	            }
	        }
	        ...
	    }
	}

#### 4.1.2 generateDecor

	protected DecorView generateDecor(int featureId) {
	    Context context;
	    if (mUseDecorContext) {
	        Context applicationContext = getContext().getApplicationContext();
	        if (applicationContext == null) {
	            context = getContext();
	        } else {
	            context = new DecorContext(applicationContext, getContext());
	            if (mTheme != -1) {
	                context.setTheme(mTheme);
	            }
	        }
	    } else {
	        context = getContext();
	    }
	    return new DecorView(context, featureId, this, getAttributes());
	}

#### 4.1.3 generateLayout

	protected ViewGroup generateLayout(DecorView decor) {
	    // Apply data from current theme.
	    TypedArray a = getWindowStyle();
 
	    mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
	    int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR) & (~getForcedWindowFlags());
	    if (mIsFloating) {
	        setLayout(WRAP_CONTENT, WRAP_CONTENT);
	        setFlags(0, flagsToUpdate);
	    } else {
	        setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
	        requestFeature(FEATURE_NO_TITLE);
	    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
	        // Don't allow an action bar if there is no title.
	        requestFeature(FEATURE_ACTION_BAR);
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowActionBarOverlay, false)) {
	        requestFeature(FEATURE_ACTION_BAR_OVERLAY);
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowActionModeOverlay, false)) {
	        requestFeature(FEATURE_ACTION_MODE_OVERLAY);
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowSwipeToDismiss, false)) {
	        requestFeature(FEATURE_SWIPE_TO_DISMISS);
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
	        setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowTranslucentStatus, false)) {
	        setFlags(FLAG_TRANSLUCENT_STATUS, FLAG_TRANSLUCENT_STATUS & (~getForcedWindowFlags()));
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowTranslucentNavigation, false)) {
	        setFlags(FLAG_TRANSLUCENT_NAVIGATION, FLAG_TRANSLUCENT_NAVIGATION & (~getForcedWindowFlags()));
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowOverscan, false)) {
	        setFlags(FLAG_LAYOUT_IN_OVERSCAN, FLAG_LAYOUT_IN_OVERSCAN&(~getForcedWindowFlags()));
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowShowWallpaper, false)) {
	        setFlags(FLAG_SHOW_WALLPAPER, FLAG_SHOW_WALLPAPER&(~getForcedWindowFlags()));
	    }
 
	    if (a.getBoolean(R.styleable.Window_windowEnableSplitTouch,
	            getContext().getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.HONEYCOMB)) {
	        setFlags(FLAG_SPLIT_TOUCH, FLAG_SPLIT_TOUCH&(~getForcedWindowFlags()));
	    }
 
	    a.getValue(R.styleable.Window_windowMinWidthMajor, mMinWidthMajor);
	    a.getValue(R.styleable.Window_windowMinWidthMinor, mMinWidthMinor);
	    if (a.hasValue(R.styleable.Window_windowFixedWidthMajor)) {
	        if (mFixedWidthMajor == null) mFixedWidthMajor = new TypedValue();
	        a.getValue(R.styleable.Window_windowFixedWidthMajor, mFixedWidthMajor);
	    }
	    if (a.hasValue(R.styleable.Window_windowFixedWidthMinor)) {
	        if (mFixedWidthMinor == null) mFixedWidthMinor = new TypedValue();
	        a.getValue(R.styleable.Window_windowFixedWidthMinor, mFixedWidthMinor);
	    }
	    if (a.hasValue(R.styleable.Window_windowFixedHeightMajor)) {
	        if (mFixedHeightMajor == null) mFixedHeightMajor = new TypedValue();
	        a.getValue(R.styleable.Window_windowFixedHeightMajor, mFixedHeightMajor);
	    }
	    if (a.hasValue(R.styleable.Window_windowFixedHeightMinor)) {
	        if (mFixedHeightMinor == null) mFixedHeightMinor = new TypedValue();
	        a.getValue(R.styleable.Window_windowFixedHeightMinor, mFixedHeightMinor);
	    }
	    if (a.getBoolean(R.styleable.Window_windowContentTransitions, false)) {
	        requestFeature(FEATURE_CONTENT_TRANSITIONS);
	    }
	    if (a.getBoolean(R.styleable.Window_windowActivityTransitions, false)) {
	        requestFeature(FEATURE_ACTIVITY_TRANSITIONS);
	    }
 
	    mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);
 
	    final Context context = getContext();
	    final int targetSdk = context.getApplicationInfo().targetSdkVersion;
	    final boolean targetPreHoneycomb = targetSdk < android.os.Build.VERSION_CODES.HONEYCOMB;
	    final boolean targetPreIcs = targetSdk < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH;
	    final boolean targetPreL = targetSdk < android.os.Build.VERSION_CODES.LOLLIPOP;
	    final boolean targetPreQ = targetSdk < Build.VERSION_CODES.Q;
	    final boolean targetHcNeedsOptions = context.getResources().getBoolean(R.bool.target_honeycomb_needs_options_menu);
	    final boolean noActionBar = !hasFeature(FEATURE_ACTION_BAR) || hasFeature(FEATURE_NO_TITLE);
 
	    if (targetPreHoneycomb || (targetPreIcs && targetHcNeedsOptions && noActionBar)) {
	        setNeedsMenuKey(WindowManager.LayoutParams.NEEDS_MENU_SET_TRUE);
	    } else {
	        setNeedsMenuKey(WindowManager.LayoutParams.NEEDS_MENU_SET_FALSE);
	    }
 
	    if (!mForcedStatusBarColor) {
	        mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
	    }
	    if (!mForcedNavigationBarColor) {
	        mNavigationBarColor = a.getColor(R.styleable.Window_navigationBarColor, 0xFF000000);
	        mNavigationBarDividerColor = a.getColor(R.styleable.Window_navigationBarDividerColor, 0x00000000);
	    }
	    if (!targetPreQ) {
	        mEnsureStatusBarContrastWhenTransparent = a.getBoolean(R.styleable.Window_enforceStatusBarContrast, false);
	        mEnsureNavigationBarContrastWhenTransparent = a.getBoolean(R.styleable.Window_enforceNavigationBarContrast, true);
	    }
 
	    WindowManager.LayoutParams params = getAttributes();
 
	    // Non-floating windows on high end devices must put up decor beneath the system bars and
	    // therefore must know about visibility changes of those.
	    if (!mIsFloating) {
	        if (!targetPreL && a.getBoolean(R.styleable.Window_windowDrawsSystemBarBackgrounds, false)) {
	            setFlags(FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS, FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS & ~getForcedWindowFlags());
	        }
	        if (mDecor.mForceWindowDrawsBarBackgrounds) {
	            params.privateFlags |= PRIVATE_FLAG_FORCE_DRAW_BAR_BACKGROUNDS;
	        }
	    }
	    if (a.getBoolean(R.styleable.Window_windowLightStatusBar, false)) {
	        decor.setSystemUiVisibility(decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
	    }
	    if (a.getBoolean(R.styleable.Window_windowLightNavigationBar, false)) {
	        decor.setSystemUiVisibility(decor.getSystemUiVisibility() | View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR);
	    }
	    if (a.hasValue(R.styleable.Window_windowLayoutInDisplayCutoutMode)) {
	        int mode = a.getInt(R.styleable.Window_windowLayoutInDisplayCutoutMode, -1);
	        params.layoutInDisplayCutoutMode = mode;
	    }
 
	    if (mAlwaysReadCloseOnTouchAttr || getContext().getApplicationInfo().targetSdkVersion
	            >= android.os.Build.VERSION_CODES.HONEYCOMB) {
	        if (a.getBoolean(R.styleable.Window_windowCloseOnTouchOutside, false)) {
	            setCloseOnTouchOutsideIfNotSet(true);
	        }
	    }
 
	    if (!hasSoftInputMode()) {
	        params.softInputMode = a.getInt(R.styleable.Window_windowSoftInputMode, params.softInputMode);
	    }
 
	    if (a.getBoolean(R.styleable.Window_backgroundDimEnabled, mIsFloating)) {
	        /* All dialogs should have the window dimmed */
	        if ((getForcedWindowFlags()&WindowManager.LayoutParams.FLAG_DIM_BEHIND) == 0) {
	            params.flags |= WindowManager.LayoutParams.FLAG_DIM_BEHIND;
	        }
	        if (!haveDimAmount()) {
	            params.dimAmount = a.getFloat(android.R.styleable.Window_backgroundDimAmount, 0.5f);
	        }
	    }
 
	    if (params.windowAnimations == 0) {
	        params.windowAnimations = a.getResourceId(R.styleable.Window_windowAnimationStyle, 0);
	    }
 
	    // The rest are only done if this window is not embedded; otherwise,
	    // the values are inherited from our container.
	    if (getContainer() == null) {
	        if (mBackgroundDrawable == null) {
	            if (mFrameResource == 0) {
	                mFrameResource = a.getResourceId(R.styleable.Window_windowFrame, 0);
	            }
 
	            if (a.hasValue(R.styleable.Window_windowBackground)) {
	                mBackgroundDrawable = a.getDrawable(R.styleable.Window_windowBackground);
	            }
	        }
	        if (a.hasValue(R.styleable.Window_windowBackgroundFallback)) {
	            mBackgroundFallbackDrawable = a.getDrawable(R.styleable.Window_windowBackgroundFallback);
	        }
	        if (mLoadElevation) {
	            mElevation = a.getDimension(R.styleable.Window_windowElevation, 0);
	        }
	        mClipToOutline = a.getBoolean(R.styleable.Window_windowClipToOutline, false);
	        mTextColor = a.getColor(R.styleable.Window_textColor, Color.TRANSPARENT);
	    }
 
	    // Inflate the window decor.
 
	    int layoutResource;
	    int features = getLocalFeatures();
	    // System.out.println("Features: 0x" + Integer.toHexString(features));
	    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
	        layoutResource = R.layout.screen_swipe_dismiss;
	        setCloseOnSwipeEnabled(true);
	    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
	        if (mIsFloating) {
	            TypedValue res = new TypedValue();
	            getContext().getTheme().resolveAttribute(
	                    R.attr.dialogTitleIconsDecorLayout, res, true);
	            layoutResource = res.resourceId;
	        } else {
	            layoutResource = R.layout.screen_title_icons;
	        }
	        // XXX Remove this once action bar supports these features.
	        removeFeature(FEATURE_ACTION_BAR);
	        // System.out.println("Title Icons!");
	    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
	        // Special case for a window with only a progress bar (and title).
	        // XXX Need to have a no-title version of embedded windows.
	        layoutResource = R.layout.screen_progress;
	        // System.out.println("Progress!");
	    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
	        // Special case for a window with a custom title.
	        // If the window is floating, we need a dialog layout
	        if (mIsFloating) {
	            TypedValue res = new TypedValue();
	            getContext().getTheme().resolveAttribute(
	                    R.attr.dialogCustomTitleDecorLayout, res, true);
	            layoutResource = res.resourceId;
	        } else {
	            layoutResource = R.layout.screen_custom_title;
	        }
	        // XXX Remove this once action bar supports these features.
	        removeFeature(FEATURE_ACTION_BAR);
	    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
	        // If no other features and not embedded, only need a title.
	        // If the window is floating, we need a dialog layout
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title;
        }
        // System.out.println("Title!");
	    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
	        layoutResource = R.layout.screen_simple_overlay_action_mode;
	    } else {
	        // Embedded, so no decoration is needed.
	        layoutResource = R.layout.screen_simple;
	        // System.out.println("Simple!");
	    }
 
	    mDecor.startChanging();
	    // DecorView添加子View. 从父到子依次是:DecorView -> DecorCaptionView(不一定包含) -> root(即layoutResource对应的View)
	    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
 
	    // 获取R.id.content对应的ContentView
	    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
 
	    if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
	        ProgressBar progress = getCircularProgressBar(false);
	        if (progress != null) {
	            progress.setIndeterminate(true);
	        }
	    }
 
	    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
	        registerSwipeCallbacks(contentParent);
	    }
 
	    // 仅最顶层Window执行
	    if (getContainer() == null) {
	        mDecor.setWindowBackground(mBackgroundDrawable);
 
	        final Drawable frame;
	        if (mFrameResource != 0) {
	            frame = getContext().getDrawable(mFrameResource);
	        } else {
	            frame = null;
	        }
	        mDecor.setWindowFrame(frame);
 
	        mDecor.setElevation(mElevation);
	        mDecor.setClipToOutline(mClipToOutline);
 
	        if (mTitle != null) {
	            setTitle(mTitle);
	        }
 
	        if (mTitleColor == 0) {
	            mTitleColor = mTextColor;
	        }
	        setTitleColor(mTitleColor);
	    }
 
	    mDecor.finishChanging();
 
	    return contentParent;
	}

### 4.2 DecorView  
[ -> frameworks/base/core/java/com/android/internal/policy/DecorView.java ]

#### 4.2.1 onResourcesLoaded

	void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
	    if (mBackdropFrameRenderer != null) {
	        loadBackgroundDrawablesIfNeeded();
	        mBackdropFrameRenderer.onResourcesLoaded(
	                this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
	                mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
	                getCurrentColor(mNavigationColorViewState));
	    }
 
	    // 创建DecorCaptionView(即包含系统按钮如最大化,关闭等的标题)
	    mDecorCaptionView = createDecorCaptionView(inflater);
	    final View root = inflater.inflate(layoutResource, null);
	    if (mDecorCaptionView != null) {
	        // 从父到子依次是:DecorView -> DecorCaptionView -> root
	        if (mDecorCaptionView.getParent() == null) {
	            addView(mDecorCaptionView, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
	        }
	        mDecorCaptionView.addView(root, new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
	    } else {
	        // 从父到子依次是:DecorView -> root
	        addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
	    }
	    mContentRoot = (ViewGroup) root;
	    initializeElevation();
	}
 
	private DecorCaptionView createDecorCaptionView(LayoutInflater inflater) {
	    DecorCaptionView decorCaptionView = null;
	    for (int i = getChildCount() - 1; i >= 0 && decorCaptionView == null; i--) {
	        View view = getChildAt(i);
	        if (view instanceof DecorCaptionView) {
	            // The decor was most likely saved from a relaunch - so reuse it.
	            decorCaptionView = (DecorCaptionView) view;
	            removeViewAt(i);
	        }
	    }
	    final WindowManager.LayoutParams attrs = mWindow.getAttributes();
	    final boolean isApplication = attrs.type == TYPE_BASE_APPLICATION ||
	            attrs.type == TYPE_APPLICATION || attrs.type == TYPE_DRAWN_APPLICATION;
	    final WindowConfiguration winConfig = getResources().getConfiguration().windowConfiguration;
	    if (!mWindow.isFloating() && isApplication && winConfig.hasWindowDecorCaption()) {
	        if (decorCaptionView == null) {
	            decorCaptionView = inflateDecorCaptionView(inflater);
	        }
	        decorCaptionView.setPhoneWindow(mWindow, true /*showDecor*/);
	    } else {
	        decorCaptionView = null;
	    }
 
	    // Tell the decor if it has a visible caption.
	    enableCaption(decorCaptionView != null);
	    return decorCaptionView;
	}

**一句话总结PhoneWindow#installDecor**:   
该方法在Activity#setContentView(layoutResID)里调用. 用来在PhoneWindow创建DecorView及其子View.   

PhoneWindow#installDecor具体步骤:   
1.`generateDecor`:  创建DecorView(即一个FrameLayout)
2.`generateLayout`:  根据设置的Window相关属性, 设置PhoneWindow特性. 给PhoneWindow的根DecorView添加子View, 并返回ContentView  
(1)  读取`frameworks/base/core/res/res/values/attrs.xml`中Window相关的属性`< declare-styleable name="Window">`  
(2)  根据设置的Window各属性值, 设置PhoneWindow的特性(例如requestFeature/setFlags/WindowManager.LayoutParams), 以及选择对应的layout(默认为screen_simple)  
(3)  DecorView添加子View. 从父到子依次是:DecorView -> DecorCaptionView(不一定包含) -> root(即上述layout对应的View)  
(4)  返回root(即`R.layout.screen_simple`)中R.id.content对应的ContentView(即一个FrameLayout)  

DecorCaptionView即包含系统按钮如最大化,关闭等的标题, 此处不细纠.  

[ -> frameworks/base/core/res/res/layout/screen_simple.xml ]  

	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:fitsSystemWindows="true"
	    android:orientation="vertical">
	    <ViewStub
		     android:id="@+id/action_mode_bar_stub"
             android:inflatedId="@+id/action_mode_bar"
             android:layout="@layout/action_mode_bar"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:theme="?attr/actionBarTheme" />
	    <FrameLayout
	         android:id="@android:id/content"
	         android:layout_width="match_parent"
	         android:layout_height="match_parent"
	         android:foregroundInsidePadding="false"
	         android:foregroundGravity="fill_horizontal|top"
	         android:foreground="?android:attr/windowContentOverlay" />
	</LinearLayout>

## 5.Activity&PhoneWindow&DecorView关系图  

![PhoneWindow](/image/2020-05-02-create-and-add-window-for-systemui-activity/PhoneWindow.png)

上图是根据源码总结出来的各个类的关系图. 主要包含3部分:  
**1.`StatusBar`**  
状态栏(系统UI), 其根View为StatusBarWindowView, 通过WindowManagerImpl.addView添加到窗口管理器.  
**2.`DecorView`**  
(1)  应用程序View. 通过WindowManagerImpl.addView添加到窗口管理器.  一个页面对应一个Activity, 一个Activity包含一个PhoneWindow, PhoneWindow的根View即为DecorView, DecorView为FrameLayout.  
(2)  根据创建Activity时设置的Window属性的不同, 选择不同的layout布局, 并将该layout布局添加到DecorView中. 最简单的为R.layout.screen_simple.     
(3)  上述layout布局R.layout.screen_simple为LinearLayout, 包含2个元素: @id/action_mode_bar_stub和@android:id/content. 即标题栏和内容体. 在Activity#onCreate -> setContentView(layoutResID)调用时,  会将layoutResID填充到@android:id/content  
**3.`NavigationBar`**  
通常指的手机底部的虚拟按键.  

上面我们已经知道StatusBar和DecorView都会添加到WindowManagerImpl, 通过窗口管理器进行管理. 因此开发者可以定制状态栏和应用程序的UI样式及他们之间的显示关系.  

下面以沉浸式状态栏实现为例:  

![transparent_statusbar](/image/2020-05-02-create-and-add-window-for-systemui-activity/transparent_statusbar.png)

我们实现这样的效果需要遵循几个步骤:    
(1)  状态栏透明  
(2)  DecorView占满整个屏幕  
(3)  NavigationBar的控制  

实现代码如下:  

    private static void transparentStatusBar(final Activity activity) {
        // Android Android 4.4以下, 跳过
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
            return;
        }
        Window window = activity.getWindow();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            // Android 5.0及以上, 设置DecorView全屏和系统状态栏透明
            window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
            int option = View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN;
            window.getDecorView().setSystemUiVisibility(option);
            window.setStatusBarColor(Color.TRANSPARENT);
        } else {
	        // Android 4.4 设置系统状态栏透明(不能实现DecorView全屏)
	        // 由于Android 4.4不存在PhoneWindow#setStatusBarColor这个方法.
	        // 因此设置透明状态栏需要在DecorView添加一个AlphaStatusBarView
	        addStatusBarAlpha(activity, 0)
            window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        }
    }

	// 仅仅用于Android 4.4
    private static void addStatusBarAlpha(final Activity activity, final int alpha) {
        ViewGroup parent = (ViewGroup) activity.getWindow().getDecorView()
        View fakeStatusBarView = parent.findViewWithTag(TAG_ALPHA);
        if (fakeStatusBarView != null) {
            if (fakeStatusBarView.getVisibility() == View.GONE) {
                fakeStatusBarView.setVisibility(View.VISIBLE);
            }
            fakeStatusBarView.setBackgroundColor(Color.argb(alpha, 0, 0, 0));
        } else {
            parent.addView(createAlphaStatusBarView(parent.getContext(), alpha));
        }
    }
    
	// 仅仅用于Android 4.4
	private static View createAlphaStatusBarView(final Context context, final int alpha) {
        View statusBarView = new View(context);
        statusBarView.setLayoutParams(new LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, getStatusBarHeight()));
        statusBarView.setBackgroundColor(Color.argb(alpha, 0, 0, 0));
        statusBarView.setTag(TAG_ALPHA);
        return statusBarView;
    }


**系统UI的可见性** :  
系统UI(如StatusBar和NavigationBar), 可以在Activity中通过DecorView#setSystemUiVisibility控制.    
(1)  设置DecorView全屏  
View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN  
(2)  隐藏NavigationBar  
View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION  | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION  
(3)  其他参考View.SYSTEM_UI_FLAG_XXX


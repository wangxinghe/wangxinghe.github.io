---
layout: post
comments: true
title: "Window的类型和Z-Order"
description: "Window的类型和Z-Order"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

## 1.Window类型

常见的Window类型如下：  

![type](/image/2020-05-10-window-type-zorder/type.png)

完整的Window类型及解释, 可以参考源码WindowManager

## 2.Z-Order的确定

在Client端, WindowManager.LayoutParams的x & y & type用来确定Window的三维坐标位置.

    public static class WindowManager.LayoutParams extends ViewGroup.LayoutParams
		    implements Parcelable {
		// x坐标
	    public int x;
	    // y坐标
	    public int y;
	    // 类型(WMS根据type确定Z-Order)
	    public int type;
		...
    }

在WMS端, WindowState表示一个窗口实例, 其中属性mBaseLayer和mSubLayer用来确定Z-Order

[ -> frameworks/base/services/core/java/com/android/server/wm/WindowState.java ]

	// 计算主序时乘以10000, 目的是为同类型的多个窗口预留空间和Z-Order调整
	static final int TYPE_LAYER_MULTIPLIER = 10000;
	// 计算主序时加1000, 目的是确定相同主序的子窗口相对于父窗口的上下位置
	static final int TYPE_LAYER_OFFSET = 1000;
  
	// 表示主序
	final int mBaseLayer;
	// 表示子序
	final int mSubLayer;
 
	WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
            WindowState attachedWindow, int appOp, int seq, WindowManager.LayoutParams a,
            int viewVisibility, final DisplayContent displayContent) {
	    ...
	    // 若当前窗口类型为子窗口
	    if ((mAttrs.type >= FIRST_SUB_WINDOW && mAttrs.type <= LAST_SUB_WINDOW)) {
	        // 计算主序, 主序与父窗口一致，主序大的窗口位于主序小的窗口上面
	        mBaseLayer = mPolicy.getWindowLayerLw(parentWindow) * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
	        // 计算子序
	        mSubLayer = mPolicy.getSubWindowLayerFromTypeLw(a.type);
	        ...
	    } else { // 若当前窗口类型不是子窗口
	        // 计算主序
	        mBaseLayer = mPolicy.getWindowLayerLw(this) * TYPE_LAYER_MULTIPLIER + TYPE_LAYER_OFFSET;
	        // 子序为0
	        mSubLayer = 0;
	        ...
	    }
	    ...
	}

### 2.1 确定主序

[ -> frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java ]

	default int getWindowLayerLw(WindowState win) {
	    return getWindowLayerFromTypeLw(win.getBaseType(), win.canAddInternalSystemWindow());
	}

	default int getWindowLayerFromTypeLw(int type) {
	    if (isSystemAlertWindowType(type)) {
	        throw new IllegalArgumentException("Use getWindowLayerFromTypeLw() or getWindowLayerLw() for alert window types");
	    }
	    return getWindowLayerFromTypeLw(type, false /* canAddInternalSystemWindow */);
	}

	default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow) {
	    if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
	        return APPLICATION_LAYER; // 2
	    }
 
	    switch (type) {
	        case TYPE_WALLPAPER:
	            // wallpaper is at the bottom, though the window manager may move it.
	            return  1;
	        case TYPE_PRESENTATION:
	        case TYPE_PRIVATE_PRESENTATION:
	        case TYPE_DOCK_DIVIDER:
	        case TYPE_QS_DIALOG:
	            return  APPLICATION_LAYER;
	        case TYPE_PHONE:
	            return  3;
	        case TYPE_SEARCH_BAR:
	        case TYPE_VOICE_INTERACTION_STARTING:
	            return  4;
	        case TYPE_VOICE_INTERACTION:
	            // voice interaction layer is almost immediately above apps.
	            return  5;
	        case TYPE_INPUT_CONSUMER:
	            return  6;
	        case TYPE_SYSTEM_DIALOG:
	            return  7;
	        case TYPE_TOAST:
	            // toasts and the plugged-in battery thing
	            return  8;
	        case TYPE_PRIORITY_PHONE:
	            // SIM errors and unlock.  Not sure if this really should be in a high layer.
	            return  9;
	        case TYPE_SYSTEM_ALERT:
	            // like the ANR / app crashed dialogs
	            // Type is deprecated for non-system apps. For system apps, this type should be
	            // in a higher layer than TYPE_APPLICATION_OVERLAY.
	            return  canAddInternalSystemWindow ? 13 : 10;
	        case TYPE_APPLICATION_OVERLAY:
	            return  12;
	        case TYPE_DREAM:
	            // used for Dreams (screensavers with TYPE_DREAM windows)
	            return  14;
	        case TYPE_INPUT_METHOD:
	            // on-screen keyboards and other such input method user interfaces go here.
	            return  15;
	        case TYPE_INPUT_METHOD_DIALOG:
	            // on-screen keyboards and other such input method user interfaces go here.
	            return  16;
	        case TYPE_STATUS_BAR:
	            return  17;
	        case TYPE_STATUS_BAR_PANEL:
	            return  18;
	        case TYPE_STATUS_BAR_SUB_PANEL:
	            return  19;
	        case TYPE_KEYGUARD_DIALOG:
	            return  20;
	        case TYPE_VOLUME_OVERLAY:
	            // the on-screen volume indicator and controller shown when the user
	            // changes the device volume
	            return  21;
	        case TYPE_SYSTEM_OVERLAY:
	            // the on-screen volume indicator and controller shown when the user
	            // changes the device volume
	            return  canAddInternalSystemWindow ? 22 : 11;
	        case TYPE_NAVIGATION_BAR:
	            // the navigation bar, if available, shows atop most things
	            return  23;
	        case TYPE_NAVIGATION_BAR_PANEL:
	            // some panels (e.g. search) need to show on top of the navigation bar
	            return  24;
	        case TYPE_SCREENSHOT:
	            // screenshot selection layer shouldn't go above system error, but it should cover
	            // navigation bars at the very least.
	            return  25;
	        case TYPE_SYSTEM_ERROR:
	            // system-level error dialogs
	            return  canAddInternalSystemWindow ? 26 : 10;
	        case TYPE_MAGNIFICATION_OVERLAY:
	            // used to highlight the magnified portion of a display
	            return  27;
	        case TYPE_DISPLAY_OVERLAY:
	            // used to simulate secondary display devices
	            return  28;
	        case TYPE_DRAG:
	            // the drag layer: input for drag-and-drop is associated with this window,
	            // which sits above all other focusable windows
	            return  29;
	        case TYPE_ACCESSIBILITY_OVERLAY:
	            // overlay put by accessibility services to intercept user interaction
	            return  30;
	        case TYPE_SECURE_SYSTEM_OVERLAY:
	            return  31;
	        case TYPE_BOOT_PROGRESS:
	            return  32;
	        case TYPE_POINTER:
	            // the (mouse) pointer layer
	            return  33;
	        default:
	            return APPLICATION_LAYER;
	    }
	}

![type](/image/2020-05-10-window-type-zorder/base-layer.png)

从主序的确定过程可知:  
从大的分类上来说, 系统窗口 > 子窗口 > 应用窗口, 整体上type和主序存在正相关.  
但是在单个分类里, 单个窗口的type大小和主序大小不存在正相关或负相关.   
主序越大, 窗口Z-Order越靠上面.  

### 2.2 确定子序

	default int getSubWindowLayerFromTypeLw(int type) {
	    switch (type) {
	        case TYPE_APPLICATION_PANEL:
	        case TYPE_APPLICATION_ATTACHED_DIALOG:
	            return APPLICATION_PANEL_SUBLAYER; // 1
	        case TYPE_APPLICATION_MEDIA:
	            return APPLICATION_MEDIA_SUBLAYER; // -2
	        case TYPE_APPLICATION_MEDIA_OVERLAY:
	            return APPLICATION_MEDIA_OVERLAY_SUBLAYER; // -1
	        case TYPE_APPLICATION_SUB_PANEL:
	            return APPLICATION_SUB_PANEL_SUBLAYER; // 2
	        case TYPE_APPLICATION_ABOVE_SUB_PANEL:
	            return APPLICATION_ABOVE_SUB_PANEL_SUBLAYER; // 3
	    }
	    return 0;
	}

![type](/image/2020-05-10-window-type-zorder/sub-layer.png)

窗口子序，用来描述子窗口相对于父窗口的位置.  
子序>0表示位于父窗口上面，子序<0表示位于父窗口下面，子序=0表示不是子窗口.  
子序越大, 子窗口相对于父窗口越靠上面.  

从子序的确定过程可知:  
子窗口的type大小和子序大小不存在正相关或负相关.  
(1) 视频相关的子窗口类型(TYPE_APPLICATION_MEDIA和TYPE_APPLICATION_MEDIA_OVERLAY)位于父窗口的下面.  
(2) 其他类型子窗口位于父窗口上面.  

确定了主序和子序, 之后的过程后面再分析, 此处不细纠.

## 3.常见Window

App开发中常见的涉及Window的地方:  Activity / Dialog / PopupWindow / Toast / 输入法.

通过查看相关源码, 得出如下结论:  
**Activity**: 创建PhoneWindow, 类型为TYPE_BASE_APPLICATION  
**Dialog**:   创建PhoneWindow, 未指定类型  
**PopupWindow**:  未创建Window, 类型为TYPE_APPLICATION_PANEL  
**Toast**:  类型为TYPE_TOAST  
**输入法(IMM)**:  类型为TYPE_INPUT_METHOD


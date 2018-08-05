---
layout: post
comments: true
title: "Android学习笔记——compileSdkVersion、minSdkVersion、targetSdkVersion"
description: "Android学习笔记——消息推送"
category: Android
tags: [Android]
---

**1、sdkversion配置**    
**（1）compileSdkVersion、buildToolsVersion**    
**（2）minSdkVersion、maxSdkVersion**    
**（3）targetSdkVersion**    
**（4）总结**    
**2、参考文档**	

<!--more-->

## 1、sdkversion配置

    #build.gradle
	 android {
  	  compileSdkVersion 23
	  buildToolsVersion "23.0.1"
 
     defaultConfig {
       applicationId "com.mouxuejie.xxx"
       minSdkVersion 7
       targetSdkVersion 23
       versionCode 1
       versionName "1.0"
     }
    }

    #AndroidManifest.xml
    <uses-sdk android:targetSdkVersion="23" android:minSdkVersion="7" />

sdkversion可以在主module的build.gradle或主module的AndroidManifest.xml文件中配置。

但是gradle构建时，build.gradle会覆盖AndroidManifest.xml中的配置。

### （1）compileSdkVersion    

1、告诉Gradle用哪个Android SDK版本编译你的应用。使用任何新版本的API时就需要使用对应Level的Android SDK。      
2、只影响编译时的行为，不影响运行时的行为。        

Android SDK的版本升级了的话，会出现一些新的API，某些老的API会deprecated。如果我想使用新的API，则可以将compileSdkVersion升级到对应Level。然后编译的时候，可能会出现编译警告或编译错误。

建议compileSdkVersion使用最新的API Level编译。

如果使用 `Support Library` ，那么使用最新发布的 Support Library 就需要使用最新的 SDK 编译。例如要使用 23.1.1 版本的 Support Library ，compileSdkVersion 就必需至少是 23 （大版本号要一致！）

`buildToolsVersion` 编译工具集的版本号。

### （2）minSdkVersion、maxSdkVersion    

我理解的是，minSdkVersion和maxSdkVersion决定了APP可运行的Android SDK版本的下限和上限，即可以安装在哪个范围API Level的手机系统上。

`minSdkVersion`是应用可以运行的Android SDK的最低版本。Google Play 商店用来判断用户设备是否可以安装某个应用的标志之一。

APP使用的Library如果有自己的minSdkVersion，则APP本身设置的minSdkVersion必需大于等于这些库的 minSdkVersion。

`maxSdkVersion`是应用可以运行的Android SDK的最高版本。一般不指定，因为系统升级时如果超过maxSdkVersion可能会出现APP被移除的问题。

### （3）targetSdkVersion    

minSdkVersion <= targetSdkVersion <= maxSdkVersion

最重要的就是targetSdkVersion，targetSdkVersion 是 Android 提供向前兼容的主要依据。在应用的 targetSdkVersion 没有更新之前系统不会应用最新的行为变化。

比如系统升级到Android 4.4(API 19)之后，AlarmManager API设置时间的精确度行为发生了变化，由精确变成了非精确。假设APP的targetSdkVersion是17，那么还是会保持原来的精确行为，当targetSdkVersion>=19时才会使用新特性：	    

    #AlarmManager
    AlarmManager(IAlarmManager service, Context ctx) {
        mService = service;

        mPackageName = ctx.getPackageName();
        mTargetSdkVersion = ctx.getApplicationInfo().targetSdkVersion;
        mAlwaysExact = (mTargetSdkVersion < Build.VERSION_CODES.KITKAT);
        mMainThreadHandler = new Handler(ctx.getMainLooper());
    }

再来梳理一下整个流程。

假设手机系统升级到某个新版本newVersion，可能发生的变化：（1）有新的API，老的API有可能deprecated；（2）API外观没有变化，但是内部特性发生变化。

（1）有新的API

	#Android Oreo之前
	public abstract ComponentName startService(Intent service);

	#Android Oreo之后，API层面新增了startForegroundService
	public abstract ComponentName startService(Intent service);
	public abstract ComponentName startForegroundService(Intent service);
	
所以我的APP如果想使用前台服务，需要如下写：

	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
		startForegroundService(intent);//因为只有Android Oreo后系统才有这个API，不然编译和运行都不通过
	} else {
		startService(intent);
		//然后在Service里startForeground(...)
	}
	
（2）API内部特性发生变化

targetSDKVersion的版本大小可以决定是否采用新的特性还是维持老的特性。因为在手机系统升级之后，Android SDK在提供新特性的同时会兼容一下老特性，会根据APP targetSDKVersion的版本号来选择使用哪个特性。只要targetSDKVersion没有变化，即使手机系统升级了，还是采用老特性。上面AlarmManager就是一个典型的例子。

	#Android SDK中的代码形如
	if (ctx.getApplicationInfo().targetSdkVersion > Build.VERSION_CODES.KITKAT) {
		//新特性
	} else {
		//老特性
	}
	
还有一种情况是：假设APP的targetSDKVersion是API 26（对应Android 8.0系统）,然后运行的手机系统是Android 6.0，由于Android 6.0相对8.0来说比较老，API和特性都是老的，即使APP targetSDKVersion指定了高版本，APP在低版本的手机上还是老特性。

### （4）总结

所以compileSdkVersion决定了编译期间能否使用新版本的API。targetSDKVersion决定了运行期间使用哪种特性。

用较低的minSdkVersion来覆盖最大的人群，用最新的compileSdkVersion和targetSDKVersion来获得最好的外观和行为。

maxSdkVersion >= buildToolsVersion >= compileSdkVersion>= targetSdkVersion >= minSdkVersion

## 2、参考文档	

[Android targetSdkVersion 原理](https://www.race604.com/android-targetsdkversion/)

英文版 [Picking your compileSdkVersion, minSdkVersion, and targetSdkVersion](https://medium.com/google-developers/picking-your-compilesdkversion-minsdkversion-targetsdkversion-a098a0341ebd)

中文版 [如何选择 compileSdkVersion, minSdkVersion 和 targetSdkVersion](https://chinagdg.org/2016/01/picking-your-compilesdkversion-minsdkversion-targetsdkversion/)

官网（虽然讲半天不知道讲的什么） [https://developer.android.com/guide/topics/manifest/uses-sdk-element](https://developer.android.com/guide/topics/manifest/uses-sdk-element)


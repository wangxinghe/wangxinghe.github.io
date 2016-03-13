---
layout: post
comments: true
title: "AndroidManifest合并原理"
description: "AndroidManifest合并原理"
category: android
tags: [Android]
---


Android Studio工程通常包含多个`AndroidManifest`文件，最终构建成APK时，会合并成一个`AndroidManifest`文件。但是可能很多人应该都不知道是怎么合并的，本文将为大家揭开神秘面纱。

### 1. 合并冲突规则（merge conflict rules）

合并冲突，是指多个Manifest文件中含有同一属性但值不同时，默认合并规则解决不了从而导致的冲突。

当冲突发生时，高优先级的Manifest属性值会覆盖低优先级属性值。这个优先级规则由高到低依次是：

    buildType下的Manifest设置->productFlavor下的Manifest设置->主工程src/main->dependency&library
    
默认合并冲突规则如下：

![](/image/2016-02-05-manifest-merge/default_merge_conflict_rules.png)

当然还存在例外情况：

1. `uses-feature android:required`和`uses-library android:required`默认值都是true，根据`OR`规则合并
2. 如果不指定`uses-sdk`，默认的`minSdkVersion`和`targetSdkVersion`值为1，当发生冲突时将使用高优先级的值。若不指定`targetSdkVersion`，其值等于`targetSdkVersion`
3. 当library工程的`minSdkVersion`比主工程`src/main`中的`minSdkVersion`低时会产生冲突，此时需要添加`overLibrary`标记解决冲突
4. 当library工程的`targetSdkVersion`比主工程`src/main`中的大时，合并过程会增加一些权限保证library工程能正常运行
5. 每个Manifest文件只和其子Manifest文件的属性合并
6. `<intent-filter>`的合并规则是叠加而不是覆盖

### 2. 合并冲突标记和选择器（merge conflict marker&selector）

合并冲突标记，是android tools namespace中的一个属性，用来解决默认冲突规则解决不了的冲突。

主要包含以下几个：

1. merge
   
   默认合并操作。
   
2. replace

   高优先级替换低优先级Manifest文件中的属性
   
3. strict

   属性相同而值不同时会报错，除非通过冲突规则resolved
   
4. merge-only

    仅合并低优先级的属性
    
5. remove

    移除指定的低优先级的属性
    
6. remove-All

    移除相同节点类型下所有低优先级的属性

注：节点层面默认使用merge，属性层面默认使用strict


下面看几个例子：

（1）使用`replace`标记解决`androidk:icon`和`android:label`属性冲突

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.android.tests.flavorlib.app"
       xmlns:tools="http://schemas.android.com/tools">

       <application
           android:icon="@drawable/icon"
           android:label="@string/app_name"
           tools:replace="icon, label">
           ...
             
（2）以下代码块中，`src/main`会覆盖library的`<uses-sdk>`。(默认情况下是不允许低优先级的`minSdkVersion`大于高优先级的，否则会报错。)

src/main/ Manifest:

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.android.example.app"
       xmlns:tools="http://schemas.android.com/tools">
       ...
       <uses-sdk android:targetSdkVersion="22" android:minSdkVersion="2"
             tools:overrideLibrary="com.example.lib1, com.example.lib2"/>
       ...

Library manifest:

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.example.lib1">
        ...
        <uses-sdk android:minSdkVersion="4" />
        ...
    </manifest>
    
（3）以下代码块表示，移除library1中的`permissionOne`权限，而其他模块下该权限不受影响。

    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.android.example.app"
       xmlns:tools="http://schemas.android.com/tools">
       ...
       <permission
           android:name="permissionOne"
           tools:node="remove"
           tools:selector="com.example.lib1">
           ...
           
           
### 3. 向AndroidManifest文件注入build变量值

注入`build`变量值通常需要使用manifestPlaceholders，`applicationId`属性除外。另外支持部分注入，如`android:authority="com.acme.${localApplicationId}.foo"`.

直接看例子吧：

（1）注入`applicationId`

Manifest entry:

    <activity android:name=".Main">
        <intent-filter>
            <action android:name="${applicationId}.foo"></action>
        </intent-filter>
    </activity>

Gradle build file:

    android {
        compileSdkVersion 22
        buildToolsVersion "22.0.1"

        productFlavors {
            flavor1 {
                applicationId = "com.mycompany.myapplication.productFlavor1"
            }
        }

Merged manifest value:

    <action android:name="com.mycompany.myapplication.productFlavor1.foo">
    
（2）注入其他属性

Gradle build file:

    android {
        defaultConfig {
            manifestPlaceholders = [ activityLabel:"defaultName"]
        }
        productFlavors {
            free {
            }
            pro {
                manifestPlaceholders = [ activityLabel:"proName" ]
            }
        }

Placeholder in the manifest file:

    <activity android:name=".MainActivity" android:label="${activityLabel}" >
    
### 4. 基于product favors group的合并（貌似没咋看懂，有知道的朋友可以告诉我～）

![](/image/2016-02-05-manifest-merge/manifest_merge_across_product_flovor_group.png)

AndroidManifest合并顺序为：

    prod-paid -> API-22-> density-mdpi -> ABI-x86
    
### 5. 关于隐藏的权限

当导入的library工程里隐含一些默认的权限时，合并过程会把这些默认权限加进来，以保证不同版本的sdk能够正常运行。如：

![](/image/2016-02-05-manifest-merge/sdk_version_permission.png)

### 6. 关于merge错误处理

程序构建过程中，合并过程生成日志位于`build/outputs/logs`的`manifest-merger-<productFlavor>-report.txt`

Higher priority manifest declaration:

    <activity
        android:name="com.foo.bar.ActivityOne"
        android:screenOrientation="portrait"
        android:theme="@theme1"/>

Lower priority manifest declaration:

    <activity
        android:name="com.foo.bar.ActivityOne"
        android:screenOrientation="landscape"/>

Error log:

    /project/app/src/main/AndroidManifest.xml:3:9 Error:
    Attribute activity@screenOrientation value=(portrait) from AndroidManifest.xml:3:9
    is also present at flavorlib:lib1:unspecified:3:18 value=(landscape)
    Suggestion: add 'tools:replace="icon"' to element at AndroidManifest.xml:1:5 to override
    
### 参考文档

1.[http://developer.android.com/intl/zh-cn/tools/building/manifest-merge.html][manifest-merge]

[manifest-merge]: http://developer.android.com/intl/zh-cn/tools/building/manifest-merge.html "manifest-merge"

------------------------------------

**个人公众号：学姐的IT专栏**

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)
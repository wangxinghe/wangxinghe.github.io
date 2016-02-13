---
layout: post
title: "构建方法数超过65K的应用"
description: "构建方法数超过65K的应用"
category: android
tags: [Android]
---


#### 1. 关于65535限制    
通常一个APK文件中包含一个Dex文件，而一个Dex文件允许的最大方法数是65535。当APK很大，方法数很多时，可以将一个APK文件拆分为多个Dex的方式。
#### 2. 如何避免65535问题及MultiDex限制条件    
（1）使用MultiDex方式

 Android5.0之前的机制是Dalvik运行时(Dalvik Runtime)，依赖库multidex support library作为主Dex的一部分用于管理从Dex
 
 Android5.0及以后的机制是ART运行时(ART Runtime)，ART运行时在程序安装阶段进行预编译，将所有Dex文件预编译为.oat文件
 
 MultiDex使用方法如下：
 
 在build.gradle中进行如下配置
 
     android {
        compileSdkVersion 21
        buildToolsVersion "21.1.0"

        defaultConfig {
            ...
            minSdkVersion 14
            targetSdkVersion 21
            ...

            // Enabling multidex support.
            multiDexEnabled true
        }
        ...
    }
    dependencies {
      compile 'com.android.support:multidex:1.0.0'
    }

然后配置Application

    <?xml version="1.0" encoding="utf-8"?>
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.example.android.multidex.myapplication">
        <application
            ...
            android:name="android.support.multidex.MultiDexApplication">
            ...
        </application>
    </manifest>
    
或者extends MultiDexApplication，并在attachBaseContext() 方法中添加MultiDex.install(this)

（2）动态加载（后续研究）

##### 限制条件：    
multi support library在某些情况下可能存在问题，如：

* 当从Dex过大时，在某些设备上安装时会引起ANR
* 在Android4.0之前的设备上可能无法启动程序，必须减少方法数
* 在Android5.0之前如果存在过大内存分配的case，根据Dalvik线性内存分配限制，会导致crash
* 关于主Dex应包含哪些类会有复杂的需求

#### 3. 开发小技巧    
通常MultiDex都是很耗时的，因为构建系统会花较多时间来决定哪些类放在主Dex，哪些类放在从Dex。平常开发过程中，可以充分利用Android5.0的ART运行时方式进行开发，不过需要用5.0以上的设备。

ART运行时只进行如下操作：

* 预分包阶段，将每个module构建成一个Dex文件
* 将所有Dex文件按原样封装到APK
* 不需要花时间重新决定哪些放到主Dex，哪些放到从Dex这一过程，因此节约很多时间

配置过程如下，平常构建dev即可，发布时使用prod：

    android {
        productFlavors {
            // Define separate dev and prod product flavors.
            dev {
            // dev utilizes minSDKVersion = 21 to allow the Android gradle plugin
            // to pre-dex each module and produce an APK that can be tested on
            // Android Lollipop without time consuming dex merging processes.
            minSdkVersion 21
            }
            prod {
                // The actual minSdkVersion for the application.
                minSdkVersion 14
            }
        }
          ...
        buildTypes {
            release {
                runProguard true
                proguardFiles getDefaultProguardFile('proguard-android.txt'),
                                                 'proguard-rules.pro'
            }
        }
    }
    dependencies {
      compile 'com.android.support:multidex:1.0.0'
    }    
    
    
   IDE中可以先Sync一下工程，再在Build Variant选择特定的包进行构建：
   
  ![pic](/image/2014-02-04-android-methods-over-65k/studio-build-variant.png)

#### 4. 测试    

使用MultiDexTestRunner进行测试（在Gradle1.1之前，需要添加下面的dependencies），先进行配置：

    android {
      defaultConfig {
          ...
          testInstrumentationRunner "com.android.test.runner.MultiDexTestRunner"
      }
    }

    dependencies {
        androidTestCompile('com.android.support:multidex-instrumentation:1.0.1') {
            exclude group: 'com.android.support', module: 'multidex'
        }
    }

再在MultiDexTestRunner的onCreate方法中添加如下代码：

    public void onCreate(Bundle arguments) {
        MultiDex.install(getTargetContext());
        super.onCreate(arguments);
        ...
    }
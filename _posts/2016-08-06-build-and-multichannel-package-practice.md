---
layout: post
comments: true
title: "Android打包系列——多渠道打包及签名"
description: "Android打包系列——多渠道打包及签名"
category: Android
tags: [Android]
---

## 0x00 概述

上篇文章梳理了下打包流程，本文从实践的角度教你怎么打包？

## 0x01 Gradle配置

Android应用的编译和构建是基于Gradle，我们只需要在主Module的build.gradle中进行一些配置即可。

如下是楼主的配置文件，每个参数都进行了很详细的说明。

这里梳理出来，也是便于自己知识管理。

<!--more-->

    //应用Android插件
    apply plugin: 'com.android.application'

    android {
        //指定用来编译的Android API版本号
        compileSdkVersion COMPILE_SDK_VERSION as int
        //指定sdk构建工具版本号
        buildToolsVersion BUILD_TOOLS_VERSION

        defaultConfig {//默认配置
            //唯一标识一个app
            applicationId "com.mouxuejie.fun"
            //指定运行需要的最小API版本
            minSdkVersion MIN_SDK_VERSION
            //指定用来测试的API版本
            targetSdkVersion TARGET_SDK_VERSION
            //app版本号
            versionCode VERSION_CODE as int
            //app 版本名字
            versionName VERSION_NAME
            //分包
            multiDexEnabled true
        }
        lintOptions {
            //忽略Lint错误
            abortOnError false
        }
        signingConfigs {//签名
            release {
                keyAlias KEY_ALIAS
                keyPassword KEY_PASSWORD
                storeFile file(STORE_FILE)
                storePassword STORE_PASSWORD
            }
        }
        buildTypes {
            debug {
                //debug
            }
            release {
                //启用代码混淆
                minifyEnabled true
                //代码混淆使用的配置文件
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
                //签名
                signingConfig signingConfigs.release
                //移除无用的Resource
                shrinkResources true
                //资源文件对齐
                zipAlignEnabled true
            }
        }
        productFlavors {//多渠道打包
            dev {
                // applicationId不同，认为是两个不同的app，可以同时安装在一台设备上，
                // 因此为便于调试，通过修改applicationId或者后缀可以同时安装线上包和开发包
                applicationIdSuffix ".debug"
            }
            googlepay {}
            baidu {}
            qihoo360 {}
            xiaomi {}
            tencent {}
            anzhi {}
        }

        // 指定apk名字输出格式
        applicationVariants.all { variant ->
            variant.outputs.each { output ->
                output.outputFile = new File(
                        output.outputFile.parent,
                        "techexplore-${variant.buildType.name}-${variant.versionName}-$
                        {variant.productFlavors[0].name}.apk".toLowerCase())
            }
        }
    }

    dependencies {//指定依赖
        compile fileTree(dir: 'libs', include: ['*.jar'])
        compile 'com.android.support:appcompat-v7:23.1.0'
        compile 'io.reactivex:rxjava:1.1.0'
        compile 'io.reactivex:rxandroid:1.1.0'
        compile 'com.jakewharton.rxbinding:rxbinding:0.4.0'
        compile 'com.squareup.retrofit:retrofit:1.9.0'
        compile 'com.squareup.okhttp:okhttp:2.4.0'
    }

这里有几点说明：

1. 公共的常量放在`gradle.properties`文件，然后在`build.gradle`中直接引用即可，这样便于集中管理。
2. 需要进行签名，只有签名后的包才能安装在设备上。Debug包默认使用了debug签名，自动生成；release包使用release签名，需要开发者自己生成。
3. `productFlavors`设置渠道号，可以实现多渠道打包，通过修改`applicationId`可以让同个app的不同构建包同时安装在一台设备上。

`gradle.properties`文件中常量配置如下：

    VERSION_NAME=1.0.0
    VERSION_CODE=1

    COMPILE_SDK_VERSION=23
    BUILD_TOOLS_VERSION=23.0.1
    TARGET_SDK_VERSION=23
    MIN_SDK_VERSION=14

    KEY_ALIAS=mouxuejie
    KEY_PASSWORD=******
    STORE_FILE=/Users/wangxinghe/Works/Github_Mine/com.mouxuejie.jks
    STORE_PASSWORD=******

## 0x02 证书签名

### 1. 为什么要签名？

前面已经提到，任何一个安装包都需要有签名。为App签名的本质是说明这个App是我开发的，不是别人。通过签名可以在应用和开发者之间建立可信任的关联。

通过签名，Android系统可以保证如下：

1. 拿到一个应用的安装包，能够知道作者是谁
2. 当应用更新时，能够检测是不是作者本人提交的
3. 应用中的部分文件遭到修改时，能够检测到是否为作者本人做出的修改

因此如果签名发生变化，是没办法升级安装的。

### 2. debug证书&release证书

从上面的配置文件，我们知道Android签名需要配置4个参数：

- **key alias** key别名
- **key password** key密码
- **keystore file** 密钥文件
- **keystore password** 密钥密码

通过Run按钮或命令行生成debug包时，Gradle会自动使用debug证书进行签名。debug证书的keystore默认存储在`$HOME/.android/debug.keystore`目录下。这个debug证书是首次运行应用程序时产生的。

由于debug证书安全性较差，正常情况下渠道包都需要使用release签名。

通常一个公司或部门使用同一个证书。

### 3. 如何生成一个证书？

有2种方式来生成数字证书：

**（1）图形界面方式**

打开Android Studio，点击Build -> Generate Signed APK -> Create new...，弹出New Key Store窗口。

![/image/2016-08-06-build-and-multichannel-package-practice/new-key-store-gui.png](/image/2016-08-06-build-and-multichannel-package-practice/new-key-store-gui.png)

如上图所示，输入相关信息，然后点击OK即可生成数字证书。

**（2）命令行方式**

    $ keytool -genkey -v -keystore com.mouxuejie.jks
    -alias mouxuejie -keyalg RSA -keysize 2048 -validity 10000

各个参数含义：

- -keystore keystore
- -alias key别名
- -keyalg 加密方式，RSA加密
- -keysize key大小
- -validity 证书有效期

具体执行结果如下：

![/image/2016-08-06-build-and-multichannel-package-practice/new-key-store-cmd.png](/image/2016-08-06-build-and-multichannel-package-practice/new-key-store-cmd.png)

## 0x03 多渠道打包

我们可以使用图形界面方式进行多渠道打包。不过这里就只介绍使用`gradle`命令行打包方式。

那么到底怎么使用的呢？

我们执行`gradle tasks`命令，可以得到如下task：

![/image/2016-08-06-build-and-multichannel-package-practice/gradle-tasks.png](/image/2016-08-06-build-and-multichannel-package-practice/gradle-tasks.png)

这些task的含义如下：

- `gradle assemble`产生所有渠道的Debug和Release包

- `gradle assembleAndroidTest`产生所有渠道的测试包

- `gradle assembleDebug`产生所有渠道的Debug包

- `gradle assembleRelease`产生所有渠道的Release包

- `gradle assembleAnzhi`产生某个渠道（如anzhi）的Debug和Release包

这里并没有出现`stormzhang`的文章中说的形如`gradle assembleWandoujiaRelease`的写法。

执行`gradle assemble`后，在app/build/outputs/apk下面会生成一堆文件，如下图所示：

![/image/2016-08-06-build-and-multichannel-package-practice/gradle-assemble-pkgs.png](/image/2016-08-06-build-and-multichannel-package-practice/gradle-assemble-pkgs.png)

## 0x04 参考文档

1. [http://dev.xiaomi.com/doc/p=70/index.html](http://dev.xiaomi.com/doc/p=70/index.html)
2. [https://developer.android.com/studio/publish/app-signing.html?hl=zh-cn#debug-mode](https://developer.android.com/studio/publish/app-signing.html?hl=zh-cn#debug-mode)
3. [http://jayfeng.com/2015/11/07/Android%E6%89%93%E5%8C%85%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B/
](http://jayfeng.com/2015/11/07/Android%E6%89%93%E5%8C%85%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B/
)
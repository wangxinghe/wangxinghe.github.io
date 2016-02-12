---
layout: post
title: "关于Android Studio项目的Gradle构建"
description: "关于Android Studio项目的Gradle构建"
category: android
tags: [Android]
---


Gradle构建脚本使用`DSL`(Domain Specific Language)来描述构建逻辑，使用的语言是`Groovy`。想了解Android Studio工程的Gradle构建系统，可以先从`Project的settings.gradle`、`Project的build.gradle`、`Module的build.gradle`、`gradle/wrapper`这些文件分析起。

### 1. Project的settings.gradle

这个文件描述的是Project里包含哪些module。

    include ':app', ':lib'
    
### 2. Project的build.gradle

这个文件描述的是Gradle构建所引用的仓库和最基础的依赖。

    buildscript {
       repositories {  //支持的仓库包括jcenter, mavenCentral和Ivy
           jcenter()
       }
       dependencies {  //基础依赖
           classpath 'com.android.tools.build:gradle:1.0.1'

        // NOTE: Do not place your application dependencies here: they belong
        // in the individual module build.gradle files
       }
    }

    allprojects {
       repositories {
           jcenter()
       }
    }
    
### 3. Module的build.gradle

这个文件描述的是主Module的一些配置。

    apply plugin: ‘com.android.application’ //添加用于Gradle构建过程的Android插件

    android {   //所有android相关的构建参数
        compileSdkVersion 19  //编译版本号
        buildToolsVersion “19.0.0”//构建工具版本号。需要大于或等于compileSdkVersion和targetSdkVersion

        defaultConfig { //AndroidManifest.xml相关参数配置的入口，允许覆盖Manifest文件的配置
            applicationId “com.example.my.app” //不同package的唯一识别码，仅存在于build.grade文件中
            minSdkVersion 8
            targetSdkVersion 19
            versionCode 1
            versionName "1.0"
        }
        buildTypes {//构建和打包方式，默认含debug和release2种，其中debug包使用debug key签名，release包默认无签名
            release {
                minifyEnabled true
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro’//文件混淆。其中proguard-android.txt为默认的混淆配置，proguard-rules.pro为模块额外的混淆配置

            }
        }
    }

    dependencies {  //模块的依赖库
        compile project(":lib”) //library module依赖
        compile 'com.android.support:appcompat-v7:19.0.1’//远程库依赖，格式为group:name:version
        compile fileTree(dir: 'libs', include: ['*.jar’]) //本地库依赖，包含app/libs目录下的所有jar文件。因此当module想引用某个jar时，只需将jar拷贝到<moduleName>/libs即可
    }

applicationId也可以用于不同product flavour / build type使用不同applicationId的情况：

    productFlavors {
        pro {
            applicationId = "com.example.my.pkg.pro"
        }
        free {
            applicationId = "com.example.my.pkg.free"
        }
    }

    buildTypes {
        debug {
            applicationIdSuffix ".debug” //applicationId后缀
        }
    }
    ....
    
### 4. Gradle Wrapper

Gradle Wrapper字面理解即Gradle包装，Android Studio使用Gradle Wrapper来完全嵌入Gradle的Android插件。我理解的是Android Studio不能直接使用Gradle的Android插件，需要再包一层引用。

gradle/wrapper属于Project级别文件（NOTICE：需要添加到版本控制系统）。包含如下几个文件：
 
1. gradle-wrapper.jar文件
2. gradle-wrapper.properties文件
3. Windows平台下的shell脚本（可选）
4. Mac／Linux平台下的shell脚本（可选）

Android Studio读取properties文件的配置，且运行当前目录下的wrapper文件。

gradle-wrapper.properties文件内容如下所示，若想使用指定的gradle，修改distributionUrl即可。

    distributionBase=GRADLE_USER_HOME
    distributionPath=wrapper/dists
    zipStoreBase=GRADLE_USER_HOME
    zipStorePath=wrapper/dists
    distributionUrl=https\://services.gradle.org/distributions/gradle-2.4-all.zip

关于wrapper下的shell脚本，仅在命令行或非Android Studio环境下有效，Android Studio下不行。

### 5. 关于Build Variants

Build Variants我理解的是，基于同一份代码所构建出的不同的APK。通常基于`productFlavors`和`buildTypes`两个维度构建，当然也存在其他的维度（如CPU/ABI，可根据需要定义）。

若productFlavors含baidu和wandoujia两种，buildTypes含release和debug两种，则可构建出4个不同的包，分别为
baiduRelease, baiduDebug, wandoujiaRelease, wandoujiaDebug。

不同的包，若存在差异化功能，可能会出现形如这种风格的目录`src/main/`，`src/<buildType>/`，`src/<productFlavor>/`。最终打包merge时优先级由高到低为`src/<buildType>/ -> src/<productFlavor>/ -> src/main/ -> libraries/dependencies的src`，并且高优先级会覆盖低优先级内容。

### 6. Build Tasks

Android Studio构建系统定义了一系列构建Task，顶级Task包含：assemble, check, build, clean。

关于Task更详细介绍可以研究下Gradle官方文档。

### 7. 满足不同构建包的差异化需求

这部分内容也是基于Build Variants。

通常做法为：

1. 在`build.gradle`文件定义不同渠道的`product flavors`
2. 对于存在差异化的`product flavor`，创建对应的source目录`src/<productFlavor>`
3. 在`src/<productFlavor>`下增加差异化代码
4. 以上操作同样适用于`buildTypes`，如果有需要的话。实际开发中较少涉及，通常只存在productFlavors下加不同渠道的情况

假设baidu手机助手渠道包存在一些差异化功能：有2个页面，FirstActivity跳转到SecondActivity，其中FirstActivity相同，SecondActivity功能不同。

（1）首先在build.grade文件的productFlavors中添加baidu{}，代表百度手机助手渠道包。

    ...
    android {
        ...
        defaultConfig { ... }
        signingConfigs { ... }
        buildTypes { ... }
        productFlavors {
            baidu {
            }
            full {
            }
        }
    }
    ...

（2）在app/src目录下添加baidu包，baidu目录结构和main保持一致，根据实际情况只添加差异化部分。此处在baidu包下添加SecondActivity.java文件并自定义实现即可。

（3）由于baidu和main目录下的SecondActivity的package名称相同，因此在main的FirstActivity启动即可。
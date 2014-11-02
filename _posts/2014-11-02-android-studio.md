---
layout: post
title: "Android Studio学习笔记"
description: "Android Studio学习笔记"
category: jekyll
tags: [android]
---


### 1. Introduction
Android Studio是基于IntelliJ IDEA的android开发环境。在IntelliJ的基础上增加了一些新的特性：

- 基于Gradle的构建系统
- 支持多个apk的编译生成
- 支持多种Google Services和多种设备类型
- 支持主题编辑的强大layout editor
- 更强大的Lint工具，可以检测性能、使用率、版本兼容和其他问题
- 支持混淆（Proguard）和app签名
- 内置支持Google Cloud Platform,易于嵌入Google云消息和App引擎

##### Android Studio vs. Eclipse ADT 
![](http://ww4.sinaimg.cn/mw690/785c69ebgw1elx0fevdsrj20lb06fdht.jpg)

##### Update from older versions
Help -> Check for updates

### 2. Migrate from Eclipse
##### Export from Eclipse
1. 升级Eclipse ADT插件到22.0以上
2. 选择File -> Export
3. 选择Android -> Generate Gradle build files
4. 选择要导出的eclipse工程，点击finish，工程中会生成一个build.gradle文件
##### Import into Android Studio
1. 打开Android Studio，关闭其他Project
2. 点击Import Project
3. 展开要导入的eclipse工程，选中build.gradle文件，点击ok
4. 选择Use gradle wrapper,此时工程导入了Android Studio

### 3. Creating a Project
关于新建Project和eclipse类似

### 4. Tips and Tricks
基本操作类似于Eclispe.
File->Setting->Keymap可以修改快捷键。下图为常用的Studio快捷键指令：

![Programming](http://ww2.sinaimg.cn/mw690/785c69ebgw1elx1beg2cyj20lb0b4gqx.jpg)
![Project](http://ww3.sinaimg.cn/mw690/785c69ebgw1elx1bmmbv9j20la068wh3.jpg)

### 5. Using the Android Project View
Studio左侧导航栏可以选择试图。Project和Android视图结构不一样，
Android视图主要包含：

- java/ - Source files for the module.
- manifests/ - Manifest files for the module.
- res/ - Resource files for the module.

![Structure](https://developer.android.com/images/tools/projectview03.png)
### 6. Using the Layout Editor
布局编辑器比Eclipse要更强大：

- 可以实时预览布局效果图
- 布局文件中可以使用颜色选择器选择合适的颜色
- 可以拖拽控件到布局预览中
- 具备纠错能力

### 7. Building Your Project with Gradle
build.gradle文件可以配置构建参数

- Build variants. 同一个项目可以编译生成多个不同版本号的apk文件
- Dependencies. 构建系统自动管理项目的各种依赖
- Maanifest entries. 
- Signing.
- Proguard.
- Testing. 可以生成测试apk,不需要单独创建测试工程

#####关于Project和modules
一个Android Studio Project包含多个modules。
project是一个完整的Android app.
module是能独立编译，测试和调试的组件.
module主要包括这3类：

- JAVA library modules.构建后生成jar包
- Android library modules.构建后生成AAR包
- Android application modules.构建后生成apk包

Project顶层有一个build.gradle文件，每个module也分别含有对应的build.gradle

#####关于依赖
主要包含:

- Module Dependencies
- Loacal Dependencies
- Remote Dependencies

#####关于创建依赖库Module

- 默认module(app)
- 选择File -> New Module -> Android Lib ->Next（依赖module: lib）
- 添加依赖关系.在app/build.gradle中添加依赖关系

		dependencies {
			compile project(":lib")
		}

#####关于构建项目
有2种方式：

- 视图方式.Build -> Make Project
- 命令行方式.项目根目录下执行指令

		chmod +x gradlew
		./gradlew assembleDebug(若是release包，则执行assembleRelease)

		
#####关于build.gradle
	apply plugin: 'android'
	android {
    	compileSdkVersion 19
    	buildToolsVersion "19.0.0"

    	defaultConfig {//默认配置
        	minSdkVersion 8
       		targetSdkVersion 19
        	versionCode 1
        	versionName "1.0"
    	}

    	buildTypes {//混淆
        	release {
            	runProguard true
            	proguardFiles getDefaultProguardFile('proguard-android.txt'), \
            	'proguard-rules.txt'
        	}
    	}

    	signingConfigs {//签名
        	release {
            	storeFile file("myreleasekey.keystore")
            	storePassword "password"
            	keyAlias "MyReleaseKey"
            	keyPassword "password"
        	}
    	}
	}

	dependencies {//依赖
		// Module Dependencies
    	compile project(":lib")
		// Remote binary dependency
    	compile 'com.android.support:appcompat-v7:19.0.1'
		// Local binary dependecy
    	compile fileTree(dir: 'libs', include: ['*.jar'])
	}

签名中，将密码直接写在gradle文件的方式不够安全，以下方式可以避免这个问题：

从环境变量获取：

	storePassword System.getenv("KSTOREPWD")
	keyPassword System.getenv("KEYPWD")

命令行手动输入：

	storePassword System.console().readLine("\nKeystore password: ")
	keyPassword System.console().readLIne("\nKey password: ")

---未完待续
### 8. Debugging with Android Studio
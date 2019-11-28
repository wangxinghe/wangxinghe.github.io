---
layout: post
comments: true
title: "AOSP源码下载/编译/刷机/调试"
description: "AOSP源码下载/编译/刷机/调试"
category: AOSP
tags: [AOSP]
---

本文记录一下AOSP源码从下载->编译->刷机->调试的整个过程, 主要针对Framework层. 前前后后折腾了将近一周时间, 刷机过程踩了很多坑, 得到的经验教训就是一定要严格按照官网教程来做, 不然会出现各种诡异问题.

<!--more-->

### 1.下载

[https://source.android.com/setup/build/downloading](https://source.android.com/setup/build/downloading)

(1) 安装Repo

	// 创建bin目录
	mkdir ~/bin
	PATH=~/bin:$PATH
	// 下载repo启动器
	curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
	chmod a+x ~/bin/repo

Git / Repo源码控制工具参考:        
[https://source.android.com/setup/develop](https://source.android.com/setup/develop)

Repo使用参考:        
[https://source.android.com/setup/develop/repo](https://source.android.com/setup/develop/repo)

(2) 下载AOSP

	// 创建AOSP源码目录 
	mkdir aosp-10
	cd aosp-10
	// 初始化源码仓库, 选择与设备对应的源码tag版本
	repo init -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r1
	// 下载
	repo sync

选择源码版本:        
[https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds](https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds)        
以Pixel为例, 对应的Android 10源码tag是`android-10.0.0_r1`, build id是`QP1A.190711.019`.         
**PS: 这里版本不要选错,不然后面没办法成功刷机.**        
![aosp-tag](/image/2019-11-17-aosp-setup/aosp-tag.png)

(3)下载驱动        
[https://developers.google.com/android/drivers](https://developers.google.com/android/drivers)        
`Pixel`机型且build id为`QP1A.190711.019`对应的驱动是下面两个文件. **PS: 这里驱动版本不要选错.**        
![aosp-driver](/image/2019-11-17-aosp-setup/aosp-driver.png)

下载并解压, 得到两个脚本文件
	
	extract-google_devices-sailfish.sh
	extract-qcom-sailfish.sh

执行这两个脚本文件, 自动生成/vendor文件夹, 内容如下. 将/vendor文件夹拷贝到AOSP源码根目录下
		
	├── google_devices
	│   ├── marlin
	│   │   ├── BoardConfigVendor.mk
	│   │   └── device-vendor-sailfish.mk
	│   └── sailfish
	│       ├── android-info.txt
	│       ├── BoardConfigPartial.mk
	│       ├── device-partial.mk
	│       └── proprietary
	│           └── vendor.img
	└── qcom
	    └── sailfish
	        ├── BoardConfigPartial.mk
	        ├── device-partial.mk
	        └── proprietary
	            ├── ATT_profiles.xml
	            ├── lib64
	            │   ├── vendor.qti.atcmdfwd@1.0.so
	            │   └── vendor.qti.qcril.am@1.0.so
	            ├── pktlogconf
	            ├── qcrilhook.jar
	            ├── ROW_profiles.xml
	            └── VZW_profiles.xml

(4)删除之前编译生成的文件

	// 如果之前没编译过,可以不执行.
	make clobber

### 2.编译

[https://source.android.com/setup/build/building](https://source.android.com/setup/build/building)

	// 初始化编译环境
	source build/envsetup.sh
	// 选择与设备对应的编译版本
	lunch aosp_sailfish-userdebug
	// 选用8个线程并行编译 (或者make clean; make -j8)
	m -j8

编译版本选择:        
[https://source.android.com/setup/build/running#selecting-device-build](https://source.android.com/setup/build/running#selecting-device-build)        
Pixel对应的编译版本是`aosp_sailfish-userdebug`

![aosp-build-id](/image/2019-11-17-aosp-setup/aosp-build-id.png)

执行lunch aosp_sailfish-userdebug命令, 输出如下:

	============================================
	PLATFORM_VERSION_CODENAME=REL
	PLATFORM_VERSION=10
	TARGET_PRODUCT=aosp_sailfish
	TARGET_BUILD_VARIANT=userdebug
	TARGET_BUILD_TYPE=release
	TARGET_ARCH=arm64
	TARGET_ARCH_VARIANT=armv8-a
	TARGET_CPU_VARIANT=kryo
	TARGET_2ND_ARCH=arm
	TARGET_2ND_ARCH_VARIANT=armv8-a
	TARGET_2ND_CPU_VARIANT=kryo
	HOST_ARCH=x86_64
	HOST_2ND_ARCH=x86
	HOST_OS=linux
	HOST_OS_EXTRA=Linux-4.15.0-70-generic-x86_64-Ubuntu-16.04.5-LTS
	HOST_CROSS_OS=windows
	HOST_CROSS_ARCH=x86
	HOST_CROSS_2ND_ARCH=x86_64
	HOST_BUILD_TYPE=release
	BUILD_ID=QP1A.190711.019
	OUT_DIR=out
	PRODUCT_SOONG_NAMESPACES=device/google/marlin vendor/google/camera hardware/google/pixel
	============================================

关于`m -j8`和`make clean; make -j8`:        
经过实践发现, 使用`m`指令, 最终会生成生成IDEA相关的文件android.iml / android.ipr; 而使用`make clean; make -j8`指令需要再执行`mmm`才会生成.

	// 编译
	make clean; make -j8
	// 生成IDEA相关文件android.iml和android.ipr,方便导入Android Studio
	mmm development/tools/idegen/
	./development/tools/idegen/idegen.sh

源码编译是最耗时间的, 我这边3个多小时才编译完.

PS: 如果不想自己编译的话, 直接刷官方的镜像应该也是可以调试的.         
[https://developers.google.com/android/images](https://developers.google.com/android/images)        
找build id为`QP1A.190711.019`的官方镜像并下载, 大小1.3G左右        
![factory-image](/image/2019-11-17-aosp-setup/factory-image.png)

编译区分`整编`和`单编`,具体可参考:        
[http://liuwangshu.cn/framework/aosp/3-compiling-aosp.html](http://liuwangshu.cn/framework/aosp/3-compiling-aosp.html)

### 3.刷机

[https://source.android.com/setup/build/running](https://source.android.com/setup/build/running)

(1)进入fastboot模式

2种方式可以进入fastboot模式:        
方式1: 执行命令`adb reboot bootloader`        
方式2: 长按Power键开机屏幕亮的一瞬间, 同时长按`Volume Down + Power键`数秒

(2)bootloader解锁

这个比较麻烦, 淘宝买二手调试手机之前, 可以和店家说好要BL解锁.         
如果没有解锁的话, 自己网上搜关键字: Android Pixel BL解锁, 自己尝试:

[https://sspai.com/post/38319](https://sspai.com/post/38319)        
[https://www.itfanr.cc/2018/10/16/google-pixel-unlock-bl-and-root/](https://www.itfanr.cc/2018/10/16/google-pixel-unlock-bl-and-root/)        

关于OEM锁和BL锁, 有兴趣可以研究下.

(3)刷机

检查`ANDROID_PRODUCT_OUT`的值

	$ echo ${ANDROID_PRODUCT_OUT}
	/home/aosp-10/out/target/product/sailfish // echo输出

如果`ANDROID_PRODUCT_OUT`变量的值和编译后镜像路径不一致, 则需要在环境变量中配置:

	// 打开.bashrc文件
	vim .bashrc
	// 设置环境变量
	export ANDROID_PRODUCT_OUT=/home/aosp-10/out/target/product/sailfish
	// 保存文件修改
	source .bashrc

然后就可以刷机了, 这个时间很短.

	// 进入fastboot模式 (或长按Volume Down + Power键)
	adb reboot bootloader
	// 刷机
	fastboot flashall -w

我在刷机过程中, 踩过很多坑:        
比如下载的源码版本和机型不匹配, 导致刷机失败, 又从头开始下载和编译.        
比如没有下载Google驱动, 又要下载驱动,重新编译源码.        
比如编译的版本和机型不匹配, 又要重新编译源码.        
比如源码正常编译好了刷进手机, 又出现Google开屏界面和[Ramdump] writing to ext4 file界面无限循环重启的情况. 最后我刷了同版本的官方镜像, 让系统正常后再刷入自己编译的包, 总算没毛病了.        

得到的经验教训就是:        
一定要认真看官方文档, 有些可能没太明白的, 参考其他中文教程. 出现任何诡异问题, 有可能就是哪一步没做对.        
如果自己每一步都按照官方教程走的, 还是刷机失败的话, 刷同版本的官方镜像看有没有问题, 然后再刷一次.        
如果自己每一步都按照官方教程走的, 还是刷机失败的话, 多刷几次.        

不错的参考文章:        
[https://bbs.pediy.com/thread-218366.htm](https://bbs.pediy.com/thread-218366.htm)        
[https://bbs.pediy.com/thread-218513.htm](https://bbs.pediy.com/thread-218513.htm)

刷机后的系统:

![phone-0](/image/2019-11-17-aosp-setup/phone-0.png)        
![phone-1](/image/2019-11-17-aosp-setup/phone-1.png)


### 4.调试

由于源码过大,为了防止出现OOM和Indexing太久,需要先设置一下Android Studio

(1)设置Android Studio参数

参考 development/tools/idegen/README

Help -> Edit Custom Properties 修改idea.properties文件:        
idea.max.intellisense.filesize=100000        

Help -> Edit Custom VM Options 修改studio64.vmoptions文件:        
-Xms1g        
-Xmx8g        

File -> Invalidate Caches/Restart 让设置生效.

(2)过滤不必要的代码

修改android.iml文件,只加载framework和packages目录下的文件,过滤掉其他不关注的代码

      <excludeFolder url="file://$MODULE_DIR$/.repo" />
      <excludeFolder url="file://$MODULE_DIR$/abi" />
      <excludeFolder url="file://$MODULE_DIR$/art" />
      <excludeFolder url="file://$MODULE_DIR$/bionic" />
      <excludeFolder url="file://$MODULE_DIR$/bootable" />
      <excludeFolder url="file://$MODULE_DIR$/build" />
      <excludeFolder url="file://$MODULE_DIR$/cts" />
      <excludeFolder url="file://$MODULE_DIR$/dalvik" />
      <excludeFolder url="file://$MODULE_DIR$/developers" />
      <excludeFolder url="file://$MODULE_DIR$/development" />
      <excludeFolder url="file://$MODULE_DIR$/device" />
      <excludeFolder url="file://$MODULE_DIR$/external" />
      <excludeFolder url="file://$MODULE_DIR$/external/bluetooth" />
      <excludeFolder url="file://$MODULE_DIR$/external/chromium" />
      <excludeFolder url="file://$MODULE_DIR$/external/emma" />
      <excludeFolder url="file://$MODULE_DIR$/external/icu4c" />
      <excludeFolder url="file://$MODULE_DIR$/external/jdiff" />
      <excludeFolder url="file://$MODULE_DIR$/external/webkit" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
      <excludeFolder url="file://$MODULE_DIR$/hardware" />
      <excludeFolder url="file://$MODULE_DIR$/kernel" />
      <excludeFolder url="file://$MODULE_DIR$/libcore" />
      <excludeFolder url="file://$MODULE_DIR$/libnativehelper" />
      <excludeFolder url="file://$MODULE_DIR$/out" />
      <excludeFolder url="file://$MODULE_DIR$/out/eclipse" />
      <excludeFolder url="file://$MODULE_DIR$/out/host" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/common/docs" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/common/obj/JAVA_LIBRARIES/android_stubs_current_intermediates" />
      <excludeFolder url="file://$MODULE_DIR$/out/target/product" />
      <excludeFolder url="file://$MODULE_DIR$/pdk" />
      <excludeFolder url="file://$MODULE_DIR$/platform_testing" />
      <excludeFolder url="file://$MODULE_DIR$/prebuilt" />
      <excludeFolder url="file://$MODULE_DIR$/prebuilts" />
      <excludeFolder url="file://$MODULE_DIR$/sdk" />
      <excludeFolder url="file://$MODULE_DIR$/system" />
      <excludeFolder url="file://$MODULE_DIR$/test" />
      <excludeFolder url="file://$MODULE_DIR$/toolchain" />
      <excludeFolder url="file://$MODULE_DIR$/tools" />

如果还是觉得加载慢的话, 根据自己需求再过滤掉一些包.

(3)加载源码

打开Android Studio,打开android.ipr,大概10分钟就可以加载好Android源码        
源码加载好后, Android Studio的Project Structure视图选Java8和Android API 29(和下载的源码版本保持一致)

删除不必要的代码, 便于debug时跳转到项目对应的源码:        
android.iml中删除包含out路径的< sourceFolder />标签, 防止定位到编译后的out/soong的源码        
Android Studio -> SDKs删除Classpath和Sourcepath里的文件, 防止定位到SDK的源码        

解决Scan files to Index太久的问题:         
项目右键 -> Open module setting -> Modules -> 找到gen -> 右键Resources        

(4)调试

点击Attach Debugger to Android Process按钮, 选择system_process进程        
![attach](/image/2019-11-17-aosp-setup/attach.png)

在frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java类里打断点, 然后操作手机, 会执行到断点处.         
![debug](/image/2019-11-17-aosp-setup/debug.png)

也可以修改Framework源码, 重新编译后刷到手机, 具体后面再尝试.        

AOSP系列文章参考:        
[http://liuwangshu.cn/tags/AOSP%E5%9F%BA%E7%A1%80/](http://liuwangshu.cn/tags/AOSP%E5%9F%BA%E7%A1%80/)

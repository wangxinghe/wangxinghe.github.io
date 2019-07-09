---
layout: post
comments: true
title: "Chromium内核源码下载、编译和安装"
description: "Chromium内核源码下载、编译和安装"
category: Chromium
tags: [Chromium]
---

一直对Chromium内核有兴趣。因为之前做过浏览器项目，但是没有接触过内核，工作时间长了之后慢慢知道内核才是浏览器的核心；现在由于一些机缘能够接触到相关的前辈可以请教学习，当然不能错过这个机会。

<!--more-->

我问大佬我想Chromium内核，怎么才能入门，大佬扔给我一个网址，让我先学习。于是我准备按照官网提示将源码下载下来，先跑起来。源码下载、编译我折腾了一周，中间遇到各种奇怪的下载问题，我找技术小伙伴一起下载，终于搞定了，算是开启了我的Chromium内核学习之旅。

具体操作步骤：
[Checking out and building Chromium for Android](https://chromium.googlesource.com/chromium/src/+/master/docs/android_build_instructions.md)

要保证下载顺利进行，需要保证2点：	
（1）稳定的科学上网的工具        
（2）多一点耐心        

我的工作环境是ubuntu 16.04 LTS，使用的科学上网工具是ssr客户端，由于源码下载是通过terminal下载，因此terminal也需要设置代理，浏览器代理和terminal代理是分开设置的，除非在路由入口就已经做过处理。
先设置代理：        
ssr -> 服务器 -> 订阅管理 -> 添加 -> 输入url             
terminal设置代理：        
export http_proxy=http://127.0.0.1:xxxx        
export https_proxy=http://127.0.0.1:xxxx        

下载过程中可能会出现半天都没有反应的情况，只要不是报错，多点耐心，多等一会就好了。

整个过程可以分成几个步骤：        
- Install depot_tools
- Get the code
- Setting up the build
- Build Chromium
- Installing and Running Chromium on a device

		# Install depot_tools
		git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
		export PATH="$PATH:/path/to/depot_tools"
		
		# Get the code
		mkdir ~/chromium && cd ~/chromium
		fetch --nohooks android
		cd src
		gclient sync
		build/install-build-deps-android.sh
		gclient runhooks

		# Setting up the build
		gn args out/Default
		target_os = "android"
		target_cpu = "arm64"

		# Build Chromium
		autoninja -C out/Default chrome_public_apk

		# Installing and Running Chromium on a device
		out/Default/bin/chrome_public_apk install
		

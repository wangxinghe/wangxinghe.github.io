---
layout: post
comments: true
title: "jcenter vs. mavenCentral"
description: "jcenter vs. mavenCentral"
category: android
tags: [Android]
---

`jcenter()`和`mavenCentral()`是Android Studio中Gradle插件使用的仓库

Android Studio早期版本使用的是mavenCentral，从某个时候开始切换到jcenter了。

<!--more-->

这是因为jcenter在性能和占存储大小方面比mavenCentral更优：

1. jcenter是世界上最大的Java仓库
2. jcenter通过CDN服务，使用的是https协议，安全性更高，而Android Studio 0.8版本mavenCentral使用的是http协议
3. jcenter是mavenCentral的超集，包括许多额外的仓库
4. jcenter性能方面比mavenCentral更优
5. mavenCentral会自动下载很多与IDE相关的index，而这些用到的少，且不是必需

参考文献：
[Android Studio – Migration from Maven Central to JCenter](http://blog.bintray.com/2015/02/09/android-studio-migration-from-maven-central-to-jcenter/)
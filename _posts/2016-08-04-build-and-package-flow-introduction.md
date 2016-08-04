---
layout: post
comments: true
title: "Android打包系列——打包流程梳理"
description: "Android打包系列——打包流程梳理"
category: Android
tags: [Android]
---

## 0x00 前言

通常一个团队都会有一个人专门负责打包，对于其他同学并不需要关注具体打包的细节。然而，我们存在的意义不是为了仅仅满足工作的需求，而是要让自己掌握的东西更多、不断成长。

关于打包序列文字，我的计划分成几篇文章来介绍：

1. 打包流程梳理
2. 多渠道打包
3. 多渠道快速打包
4. 自动化构建

本文介绍基本打包流程。

<!--more-->

通常有2种打包方式：

- **Android Studio图形界面** 点击run按钮
- **命令行方式** gradlew assembleDebug, gradlew assembleRelease

方式1使用自动生成的debug keystore签名；方式2如果是Release包使用release keystore签名，如果是Debug包则使用debug keystore签名。

下面讲下从一堆源代码到产生apk文件的过程。

## 0x01 基本流程

![/image/2016-08-04-build-and-package-flow-introduction/build-simplified.png](/image/2016-08-04-build-and-package-flow-introduction/build-simplified.png)

上图是Android官方提供的打包简略流程图。清晰地展示了一个Android Project经过编译和打包后生成apk文件，然后再经过签名，就可以安装到设备上。

我们将一个实际的apk文件后缀改为zip并解压后，得到的内容如下：

![/image/2016-08-04-build-and-package-flow-introduction/apk_component_1.png](/image/2016-08-04-build-and-package-flow-introduction/apk_component_1.png)

和上图的描述一致。apk包内容包括：

- classes.dex...
- resources.arsc
- assets
- res
- AndroidManifest.xml
- META-INF

其中：

1. res中图片和raw文件下内容保持原样，res中其他xml文件内容均转化为二进制形式；assets文件内容保持原样
2. res中的文件会被映射到R.java文件中，访问的时候直接使用资源ID即R.id.filename；assets文件夹下的文件不会被映射到R.java中，访问的时候需要AssetManager类

## 0x02 详细流程

下面这张图是官网提供的详细流程图。

![/image/2016-08-04-build-and-package-flow-introduction/android_build.png](/image/2016-08-04-build-and-package-flow-introduction/android_build.png)

我们可以将整个打包过程概括为以下几步：

 1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
 2. 处理.aidl文件，生成对应的Java接口文件
 3. 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
 4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
 5. 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk
 6. 通过Jarsigner工具，对上面的apk进行debug或release签名
 7. 通过zipalign工具，将签名后的apk进行对齐处理。

以上步骤均为必须，否则不能在设备上安装。

关于zipalign工具，根据名字就知道是个zip文件对齐的工具。使得apk中的资源文件偏离文件起始位置4个字节，从而可以通过mmap()直接访问，从而减少RAM占用。

更详细的流程，可以看下图。

![/image/2016-08-04-build-and-package-flow-introduction/android_build_detail.png](/image/2016-08-04-build-and-package-flow-introduction/android_build_detail.png)

## 0x03 参考文档

- [https://developer.android.com/studio/build/index.html?hl=zh-cn#build-config](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-config)
- [https://developer.android.com/studio/command-line/zipalign.html](https://developer.android.com/studio/command-line/zipalign.html)

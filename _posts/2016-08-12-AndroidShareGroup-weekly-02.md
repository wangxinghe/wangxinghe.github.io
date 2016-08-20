---
layout: post
comments: true
title: "AndroidShareGroup技术周报（第二期）"
description: "AndroidShareGroup技术周报（第二期）"
category: AndroidShareGroup
tags: [AndroidShareGroup]
---

## 0x00前言

这周可爱的群友们又进行了一些技术讨论。

关于`AndroidShareGroup周报`并不一定每周都会写，关键还是看群讨论里是否有有价值的内容。

由于楼主目标明确，因此群里基本上不能容忍太多的扯淡，否则会被清除。

这样的好处就是这个群学习氛围很好，大家都是抱着学习心态加群的，大家有什么问题都会在群里问，因为会有热心的群友解答。

<!--more-->

## 0x01本周Topic

### 1.Apollo-基于编译时注解处理的RxBus实现

[https://zhuanlan.zhihu.com/p/21918727](https://zhuanlan.zhihu.com/p/21918727)

目前Android中的一些流行库，如Dagger2,ButterKnife等都使用了Compile-time的注解处理，在编译时生成注入代码，来实现注入功能。通过Java的反射机制，也可以实现同样的功能，而且实现更加简单方便，不过大量使用反射机制会导致严重的性能问题。

而编译时注解处理只会在编译的时候占用开发资源，生成额外的代码来实现功能，这些通过注解处理生成的Java源代码会同其他手写的源文件一同被编译进APK。

### 2.精通 Android Data Binding

[https://github.com/LyndonChin/MasteringAndroidDataBinding](https://github.com/LyndonChin/MasteringAndroidDataBinding)

Data Binding 解决了 Android UI 编程的一个痛点，官方原生支持 MVVM 模型可以让我们在不改变既有代码框架的前提下，非常容易地使用这些新特性。

Data Binding 框架如果能够推广开来，也许 RoboGuice、ButterKnife 这样的依赖注入框架会慢慢失去市场，因为在 Java 代码中直接使用 View 变量的情况会越来越少。

### 3.Android图片压缩工具，仿微信朋友圈压缩策略

[https://github.com/Curzibn/Luban](https://github.com/Curzibn/Luban)

通过在微信朋友圈发送近100张不同分辨率图片，对比原图与微信压缩后的图片逆向推算出来的压缩算法。

### 4.专注于配色方案和颜色搭配

[http://www.peise.net/](http://www.peise.net/)

由万能的群友推荐。作为程序员，也有画设计图和文档流程图的时候，收藏以备不时只需。

### 5.Java如何处理Word文档

[http://poi.apache.org/](http://poi.apache.org/)

Java提供了相关API，`org.apache.poi.hwpf.extractor.WordExtractor`。

### 6.Android LogCat 工具类

[https://github.com/ZhaoKaiQiang/KLog](https://github.com/ZhaoKaiQiang/KLog)

- 支持显示行号
- 支持显示Log所在函数名称
- 支持无Tag快捷打印
- 支持在Android Studio开发IDE中，点击函数名称，跳转至Log所在位置
- 支持JSON字符串解析打印
- 支持XML字符串解析打印
- 支持Log信息存储到文件
- 依赖库非常小，只有不到10K
- 支持无限长字符串打印，无Logcat4000字符限制
- 支持变长参数，任意个数打印参数
- 支持设置全局Tag

### 7.支持缩放和其他各种手势操作的ImageView

[https://github.com/chrisbanes/PhotoView](https://github.com/chrisbanes/PhotoView)

- Out of the box zooming, using multi-touch and double-tap.
- Scrolling, with smooth scrolling fling.
- Works perfectly when used in a scrolling parent (such as ViewPager).
- Allows the application to be notified when the displayed Matrix has changed. Useful for when you need to update your UI based on the current zoom/scroll position.
- Allows the application to be notified when the user taps on the Photo.

### 8.酷炫云标签View

[https://github.com/misakuo/3dTagCloudAndroid](https://github.com/misakuo/3dTagCloudAndroid)

TagCloudView是一个完全基于Android ViewGroup编写的控件。

支持将一组View展示为一个3D球形集合，并支持全方向滚动。

### 9.开阔视野

（1）**蓝牙**可以用于实现蓝牙打印、蓝牙开锁等功能。

（2）群里有位同学的工作是做中间件和构建框架的。比如调试器、统一存储、网络访问、插件化、热更新等。其中调试器就是，比如机房里有几百台手机，要让大家能连到上面做调试，且不能直接adb connect，要实时占用和释放，实现原理和VPN差不多，打个隧道。

（3）Python的xlrd库，读取csv文件

（4）ListView中的Item为WebView时，点击事件处理，可以先传到WebView的h5，再通过JSBridge回调到native层做处理

## 0x02博客推荐

1.[http://www.idtkm.com/](http://www.idtkm.com/)

Idtk同学，文章都是关于自定义View的。

2.[http://imxie.cc/](http://imxie.cc/)

谢三弟同学，90后，学生党。

###广告时间

如果想和我们一起学习技术，可以加入我们QQ群。**群号码：538266272**

![/image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg](/image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg)

互联网技术开发或人才相关，请扫码关注某学姐的微信公众号。**学姐的IT专栏**

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)

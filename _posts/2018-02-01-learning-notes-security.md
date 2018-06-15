---
layout: post
comments: true
title: "Android学习笔记——安全问题"
description: "Android学习笔记——安全问题"
category: Android
tags: [Android]
---


<!--more-->


组件安全：    
Activity、Service、Receiver、Provider访问权限设置android:exported = false    
LocalBroadcastManager替代BroadcastReceiver    
Application的debugable/allowBackup = false    

WebView安全：    
JsBridge

数据存储安全：    
敏感信息写到so

数据传输安全：    
https、TCP＋TLS

其他：    
日志输出关闭、混淆等

二次打包：    
native数字签名校验 ＋ 应用加固

页面劫持：    
在登录窗口等关键Activity的onPause方法中检测最前端Activity应用是不是自身或者是系统应用。

[移动APP安全测试要点](http://blog.nsfocus.net/mobile-app-security-security-test/)

[APK防二次打包解决方案](https://myslide.cn/slides/985?vertical=1)

[移动安全自动化审计之路](http://techshow.ctrip.com/wp-content/uploads/2017/07/4%E3%80%81%E7%A7%BB%E5%8A%A8%E5%AE%89%E5%85%A8%E8%87%AA%E5%8A%A8%E5%8C%96%E5%AE%A1%E8%AE%A1%E4%B9%8B%E8%B7%AF-%E8%B0%A2%E9%91%AB.pdf)

[Android逆向之旅—Android应用的安全的攻防之战](http://www.520monkey.com/archives/625)

[Android安全——客户端安全要点](https://www.jianshu.com/p/7f2202c18012)    




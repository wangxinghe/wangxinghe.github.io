---
layout: post
comments: true
title: "AndroidShareGroup技术周报（第三期）"
description: "AndroidShareGroup技术周报（第三期）"
category: AndroidShareGroup
tags: [AndroidShareGroup]
---

## 0x00前言

每次前言都不知道说什么。

所有的讨论都来源于可爱的群友，答案不具备官方权威性。

记录下来，是让你知道有这么回事，学习下别人思考问题的方式，了解下大家都在关注什么。

因为每一个知识点深入下去，都能写一篇很长很长的文章。

之所以坚持，是想做出一点不平凡的成绩。

<!--more-->

## 0x01本周Topic

### 1.UTF-8编码获取字符串长度

![/image/2016-08-20-AndroidShareGroup-weekly-03/utf-8.png](/image/2016-08-20-AndroidShareGroup-weekly-03/utf-8.png)

UTF-8编码是变长的，一个字符占1-4个字节，可以根据Unicode值先识别是不是汉字，汉字是在一个连续的范围内。或者参考如下代码段：

    str = str.replaceAll("[^\\x00-\\xff]", "**");
    return str.length();

关于UTF-8的说明：十分钟搞清字符集和字符编码 [http://cenalulu.github.io/linux/character-encoding/](http://cenalulu.github.io/linux/character-encoding/)

### 2.Scroll嵌套RecyclerView，高度不能自适应

可以重写RecyclerView的onMeasure方法。

不过，这种嵌套会导致Recycler失败，计算RecyclerView高度时会计算出所有item的高度之和，性能很差。RecyclerView的setMeasuredDimensionFromChildren方法，如果设置了autoMeasure的话，会遍历child然后计算高度，用来支持wrap_content属性。

所以，建议多搞几种viewType，把每个布局做成一个组合View，Adapter只负责type和分发数据，渲染交给组合View。

### 3.ViewPager搭配FragmentPagerAdapter使用，获取当前显示的Fragemnt

问题：ViewPager搭配FragmentPagerAdapter使用，初始设置setCurrentItem为非0，通过viewPager.getChildAt(i)获取的Fragment顺序和FragmentPagerAdapter里的顺序出现错位，导致获取当前Fragment不对的问题。

解决：初始setCurrentItem为2，则position为2的Fragment先添加到ViewPager，getChildAt(0)得到的就是Fragment2。可以通过复写FragmentPagerAdapter的setPrimaryItem方法记录当前Fragment，然后每次通过FragmentPagerAdapter去获取这个Fragment。

### 4.WebView回退失效问题

![/image/2016-08-20-AndroidShareGroup-weekly-03/webview-goback.png](/image/2016-08-20-AndroidShareGroup-weekly-03/webview-goback.png)

问题：canGoBack一直返回true，导致无法回退。

解决：url重定向了吧，通过webview.copyBackForwardList()可以拿到回退栈，对比下url看下。如果不需要在新页面打开url的话，可以试试shouldOverrideUrlLoading返回false。

### 5.字符串替换问题

问题：debug的时候，字符串没有替换掉

![/image/2016-08-20-AndroidShareGroup-weekly-03/string-replace.png](/image/2016-08-20-AndroidShareGroup-weekly-03/string-replace.png)

解答：String是不可变的。你应该return moneyText.replaceAll

### 6.使用系统功能裁剪图片，得到的Bitmap对象为空

问题：其他手机都正常，就是红米Note2返空

解答：有些手机（如三星）裁剪后横竖屏切换了，生命周期会重建。

### 7.Android 6.0权限问题

问题：在android4.4上可以访问网络，在6.0上加了权限判断，还是不能访问网络，权限判断那边一直不能得到权限

解答：设置里的权限管理可能被禁用了。

推荐[https://github.com/Karumi/Dexter](https://github.com/Karumi/Dexter)，Android library that simplifies the process of requesting permissions at runtime.

### 8.沉浸式状态栏
[https://developer.android.com/training/system-ui/immersive.html](https://developer.android.com/training/system-ui/immersive.html)

[http://www.jianshu.com/p/7dcfd243b1df](http://www.jianshu.com/p/7dcfd243b1df) Android右滑退出＋沉浸式（透明）状态栏

[https://github.com/laobie/StatusBarUtil](https://github.com/laobie/StatusBarUtil) A util for setting status bar style on Android App.

使用体验：StatusBarUtil这个库实现了透明状态栏，但是布局不能侵入状态栏，只有Activity的根viewgroup侵入了，再内层的布局无效，并且处理沉浸式状态栏会有兼容性问题。

### 9.Python网页微信API

[https://github.com/liuwons/wxBot](https://github.com/liuwons/wxBot)

wxBot 是用Python包装Web微信协议实现的微信机器人框架。

微信可以抓webapi的包，客户端的抓包我试过几次没成功，仅能抓到最新的未读消息，其他的抓不了

### 10.优雅配置Gradle，而不是写死

[http://yat3s.com/2016/04/08/gradle-config-elegant/](http://yat3s.com/2016/04/08/gradle-config-elegant/)

Android常用的Gradle配置和加速编译

### 11.魅族手机刷机之后，Toast不起作用

这个是因为新版本要一个权限了，新奇的魅族。具体参考[http://bbs.flyme.cn/thread-1131412-1-1.html](http://bbs.flyme.cn/thread-1131412-1-1.html)

### 12.ButterKnife真的能达到原生findViewById的效率么？

性能有损耗，但是基本忽略不计。编译时期通过apt生成代码。ViewBinder实例是反射创建的，只不过会缓存，并且用ViewBinder接口约束了实例的类型，变量和方法通过泛型参数传进去的。

### 13.开源库推荐

[https://github.com/lucasr/twoway-view](https://github.com/lucasr/twoway-view) RecyclerView made simple.

[https://github.com/Yat3s/BaseRecyclerViewAdapter](https://github.com/Yat3s/BaseRecyclerViewAdapter) BaseAdapter for RecyclerView,MultiViewType, ItemAnimation, HeaderView, ParallaxHeaderView, LoadingView etc

[https://github.com/Bigmercu/ACheckBox](https://github.com/Bigmercu/ACheckBox) This is a simple CheckBox for Android with cool animation.

![/image/2016-08-20-AndroidShareGroup-weekly-03/a-checkbox.png](/image/2016-08-20-AndroidShareGroup-weekly-03/a-checkbox.png)

### 14.开阔视野

（1）区块链

区块链（英语：Blockchain或Block chain）是一种分布式数据库，起源自比特币。区块链是一串使用密码学方法相关联产生的数据块，每一个数据块中包含了一次比特币网络交易的信息，用于验证其信息的有效性（防伪）和生成下一个区块。该概念在中本聪的白皮书[1]中提出，中本聪创造第一个区块，即“创世区块”。

区块链在网络上是公开的，可以在每一个离线比特币钱包数据中查询。比特币钱包的功能依赖于与区块链的确认，一次有效检验称为一次确认。通常一次交易要获得数个确认才能进行。轻量级比特币钱包使用在线确认，即不会下载区块链数据到设备存储中。

比特币的众多竞争币也使用同样的设计，只是在工作量证明上和算法上略有不同。如，采用权益证明和SCrypt算法等等。

更多参考：

[https://www.zhihu.com/question/27687960](https://www.zhihu.com/question/27687960) 区块链技术是什么？未来可能用于哪些方面？

[http://www.infoq.com/cn/articles/bitcoin-and-block-chain-part01](http://www.infoq.com/cn/articles/bitcoin-and-block-chain-part01) 揭秘比特币和区块链（一）：什么是区块链？

[http://wallstreetcn.com/node/234120](http://wallstreetcn.com/node/234120) 一文读懂颠覆式创新技术——区块链！

（2）马克飞象

[https://maxiang.io/](https://maxiang.io/) 马克飞象是一款专为印象笔记（Evernote）打造的Markdown编辑器

### 15.轻松娱乐

[https://yq.aliyun.com/articles/59084](https://yq.aliyun.com/articles/59084)

### 广告时间

如果想和我们一起学习技术，可以加入我们QQ群。**群号码：538266272**

![/image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg](/image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg)

互联网技术开发或人才相关，请扫码关注某学姐的微信公众号。**学姐的IT专栏**

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)


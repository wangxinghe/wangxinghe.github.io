---
layout: post
comments: true
title: "AndroidShareGroup周报第一期"
description: "AndroidShareGroup周报第一期"
category: AndroidShareGroup
tags: [AndroidShareGroup]
---


## 0x00前言

**AndroidShareGroup-ASG**，是楼主年初的时候创建的一个Android技术分享QQ群。

目的在于成为一个技术人之间交流的平台，集思广益，开阔视野。

自创建以来，这个群讨论氛围就很好，很少有扯淡，大家交流并不需要实名。其实并没有什么群规，大家都很自觉的在维护这个群。

群里鼓励除Android之外的其他技术交流，关于业内公司行情交流也是OK的。

因为群里每天都会自发有很多技术讨论话题，为了这些讨论不像流水一样消失，楼主于是决定每周将群里讨论的话题总结出来；由于群里有很多学生党，很多还是蛮厉害的，都有自己的博客和github，为了帮助同学们找工作找实习。

因此**AndroidShareGroup周报**由此诞生了，这是第一期。

**群号码：538266272**

![／image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg](／image/2016-07-30-AndroidShareGroup-weekly-01/AndroidShareGroup.jpg)

## 本周Topic

#### 1.脚本上传你的项目到JCenter/Maven

[https://github.com/Yat3s/bintrayUpload](https://github.com/Yat3s/bintrayUpload)

推荐理由：网上的流程需要好几步，官方的插件有一些Bug，对于License和Version需要hardcode，使用脚本方式规避了很多上传可能出现的问题，

#### 2.持续集成工具

- **hudson**   早期使用

- **jenkins**  绝大多数公司都使用的项目构建工具

- **Travis** github上项目构建工具

#### 3.关于toString()

![图1](／image/2016-07-30-AndroidShareGroup-weekly-01/to_string.png)

大家平常开发中对于JavaBean对象的toString()方法应该都是自己写的，其实Android Studio自带生成toString()方法，大家可以试试看，一个容易被忽略的问题。

#### 4.JSON转Model

[https://github.com/misakuo/JsonModelGenerator](https://github.com/misakuo/JsonModelGenerator)

![图2](／image/2016-07-30-AndroidShareGroup-weekly-01/json-model-generator.png)

推荐理由：Android Studio的GsonFormat插件可以很方便的实现JSON转Model。但是JSON Model Generator插件也有值得借鉴的地方，比如每个Object都会拿出来做一个独立的类，不生成静态内部类。因为静态内部类是个语法糖一样的东西，真正获取到内部类的应用需要通过编译期生成的一个方法，即反编译看到的$ClassName开头的那个方法。同时内部类访问外部类私有成员的时候也会增加access方法。

#### 5.Bug管理系统

- [BugTags](https://www.bugtags.com/) 移动时代首选Bug管理系统
- JIRA
- [kelude](http://kelude.aliyun.com/) 平台集成一体化产品管理、项目管理兼容验证流程、质量测试、团队管理，阿里云的一个东东。

#### 6.Library Module的Resource Id操作

![图3](／image/2016-07-30-AndroidShareGroup-weekly-01/library-resource-id.png)

Library Module中用不能使用Switch方式操作resource id。因为在SDK tools r14之后这些id是non final的，要操作需要使用if else，Android Studio有快捷键进行转换。

#### 7.关于加密

讨论：关于用户密码，是直接MD5然后保存到数据库？还是password+salt再加密保存？

直接MD5跟传明文差不多，RSA加密的话，那还要防止中间人攻击~~另外，服务器存储的密码是什么形式？

#### 8.Android Screencasting获取视频流

Android 5.0以上有相应API，但是不好用且权限敏感。

adb可以录屏大概3min，不过没有接口提供视频流，貌似可以借adb东风用app_process的方法获取。asm.jar可以投屏，但是帧率感人。

之前看过一个叫Cling的东东。[http://a.codekk.com/detail/Android/kevinshine/Cling%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90](http://a.codekk.com/detail/Android/kevinshine/Cling%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

#### 9.https安全性

https就是在http上加了SSL/TLS。推荐图书《图解密码技术》

也可以看Keegan小钢同学的博客系列讲解

[读《图解密码技术》(一):密码](http://keeganlee.me/post/reading/20160629)

[读《图解密码技术》(二):认证](http://keeganlee.me/post/reading/20160705)

[读《图解密码技术》(三):密钥、随机数和应用技术](http://keeganlee.me/post/reading/20160722)

#### 10.fq国际化工具

- **Shadowsocks**  Mac下专用
- **云梯VPN/Green VPN**  可以一键安装配置文件到PC和手机，想连哪就连哪
- **Lantern**    俗称蓝灯[https://getlantern.org/](https://getlantern.org/)
- **萌猫SS** 极客推荐的小众工具[http://www.sscat.cn/](http://www.sscat.cn/)
- **修改Hosts** 适用于Win/Mac/Android/iOS，不过Android需要root，iOS需要越狱

小常识：北京的移动联通网络屏蔽了iOS上VPN的PPTP连接方式端口，但是可以用L2TP

#### 11.Material图标生成工具

![图4](／image/2016-07-30-AndroidShareGroup-weekly-01/android-material-icon-generator.png)

[http://bitdroid.de/Android-Material-Icon-Generator/](http://bitdroid.de/Android-Material-Icon-Generator/)

## 博客推荐

1.[https://moxun.me/](https://moxun.me/)

相对论同学，90后，曾获湖北省优秀毕业论文，目前在淘宝工作。

2.[http://yat3s.com/](http://yat3s.com/)

Yat3s同学，90后，台湾人，目前在北京工作。

3.[http://keeganlee.me/](http://keeganlee.me/)

Keegan小钢同学，Android开发老司机。

4.[http://www.cnblogs.com/wondertwo/](http://www.cnblogs.com/wondertwo/)

哈皮小猿同学，90后，中南大学大三狗，目前找工作中，期望工作地点：北京／深圳

5.[http://dundunwen.com/](http://dundunwen.com/)

顿文同学，90后，重庆理工大学大三狗，目前找工作中，期望工作地点：广州

6.[http://anangryant.leanote.com/](http://anangryant.leanote.com/)

'不，我要去学习！'同学，90后，大连东软信息学院大三狗，目前找工作中，期望工作地点：北京／上海／杭州









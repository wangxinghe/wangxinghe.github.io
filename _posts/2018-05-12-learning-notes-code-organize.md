---
layout: post
comments: true
title: "Android学习笔记——关于代码的组织"
description: "Android学习笔记——关于代码的组织"
category: Android
tags: [Android]
---

**1、App组件化**    
**2、组件基本结构**     
**（1）MVC**     
**（2）MVP**     
**（3）MVVM**     
**（4）VIPER**     
**3、设计模式**     
**（1）单例模式**     
**（2）Buildr模式**     
**（3）......**     
**（4）源码举例**        
**4、参考文档**    

<!--more-->

### 1、App组件化   

之前写的一篇文章：[App架构之组件化理解](http://mouxuejie.com/blog/2016-03-20/architecture-componentization/) 

组件间通过`Router`通信，这个Router是业务无关的一个基础模块。可以通过Annotation实现。

组件化、模块化属于App级别的架构。

### 2、组件基本结构     

下面的架构是属于组件或模块级别，而不是整个App的架构。

#### （1）MVC     

![](/image/2018-05-12-learning-notes-code-organize/MVC.png)   

Model：包括Bean数据结构、业务逻辑处理（如数据库操作、网络请求等）    
View：就是展现出来的用户界面。   
Controller：Model和View可以直接通信，没有完全起到隔离作用，不算一个纯粹的控制器。     

存在的问题是：Model和View是耦合的。

#### （2）MVP     

![](/image/2018-05-12-learning-notes-code-organize/MVP.png)    

Model：包括Bean数据结构、业务逻辑处理    
View：就是展现出来的用户界面。   
Presenter：Model和View之间通信的桥梁，是一个控制器。    

特点：Model和View解耦。

Demo参考：[应用MVP模式写出可维护的优美Android应用
](http://blog.zhaiyifan.cn/2015/06/01/use-mvp-to-write-nice-code/)

#### （3）MVVM     

![](/image/2018-05-12-learning-notes-code-organize/MVVM.png)    

Model：包括Bean数据结构、业务逻辑处理    
View：就是展现出来的用户界面。   
ViewModel：就是与界面(view)对应的Model。因为，数据库结构往往是不能直接跟界面控件一一对应上的，所以，需要再定义一个数据对象专门对应view上的控件。而ViewModel的职责就是把model对象封装成可以显示和接受输入的界面数据对象。PS：不一定是Bean，里面可能也有业务逻辑处理，只不过这个是和View一一对应。和Presenter还是有差别。

View和ViewModel之间通过DataBinding进行双向绑定。

特点：和MVP相比，View和ViewModel通过DataBinding实现一一对应关系。

Demo参考：[选择恐惧症的福音！教你认清MVC，MVP和MVVM](http://zjutkz.net/2016/04/13/%E9%80%89%E6%8B%A9%E6%81%90%E6%83%A7%E7%97%87%E7%9A%84%E7%A6%8F%E9%9F%B3%EF%BC%81%E6%95%99%E4%BD%A0%E8%AE%A4%E6%B8%85MVC%EF%BC%8CMVP%E5%92%8CMVVM/)

关于ViewModel的理解参考：[MVVM模式中ViewModel和View、Model有什么区别](https://blog.csdn.net/AlbenXie/article/details/72771141)

#### （4）VIPER     

![](/image/2018-05-12-learning-notes-code-organize/VIPER.png)    

View：展现出来的用户界面。   
Presenter：View和Interactor之间的桥梁，相当于是一个控制器，不做具体业务逻辑。    
Entity：Bean数据结构，不包括业务逻辑处理。
Interactor：业务逻辑处理。    
Router：模块之间跳转，比如从当前A页面跳到B页面，就是由Router负责。    

VIPER模式，将MVP架构中原本由Model单独负责的Bean数据结构和业务逻辑处理分拆成2个类处理，Entity处理Bean数据结构，Interactor处理业务逻辑。此外在MVP基础上还增加了Router，负责模块跳转。    

特点：和上面几个相比，减轻了Model的工作量，避免了`胖Model`的问题（备注：胖Model和瘦Model）

参考：
[在 Android 上使用 VIPER 架构](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0329/7754.html)    
[Architecting iOS Apps with VIPER](https://www.objc.io/issues/13-architecture/viper/)    
[https://gist.github.com/marciogranzotto/1c96e87484a17bd914e159b2604e6469](https://gist.github.com/marciogranzotto/1c96e87484a17bd914e159b2604e6469)    

延伸阅读：[新版Uber App移动架构设计](https://zhuanlan.zhihu.com/p/24489480)

TODO：clean architecture

个人观点：    
MVP：适用于不太复杂的模块。    
MVVM：需要引入DataBinding框架，用起来比较复杂，感觉和MVP相比也没啥优势。    
VIPER：适用于逻辑比较复杂的模块，职责分明。    
本质上，上面那些架构都是在MVC基础上的优化，并不是死的，并不一定要去套上面的模式，遵循一些模块化的基本规则即可。    

### 3、设计模式     

#### （1）单例模式     

#### （2）Buildr模式     

#### （3）......     

#### （4）源码举例        

### 4、参考文档    
（1）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)    
（2）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)    
（3）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)    
（4）[iOS应用架构谈](https://casatwy.com/iosying-yong-jia-gou-tan-kai-pian.html)    
（5）[iOS 架构模式–解密 MVC，MVP，MVVM以及VIPER架构](http://www.codertopic.com/?p=247)    


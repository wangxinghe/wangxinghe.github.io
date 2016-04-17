---
layout: post
comments: true
title: "App架构之组件化理解"
description: "App架构之组件化理解"
category: android
tags: [Android]
---

### 概述

项目发展到一定阶段，随着需求的增加以及需求的频繁变更，项目会越来越大，耦合会越来越多，开发效率也会降低，这个时候需要做的就是进行模块拆分，官方的说法就是组件化。

<!--more-->

### App基本框架模型

![App_Framework](/image/2016-03-20-architecture-componentization/app_framework.jpg)

任意一个App抽象出来，可以得到上图模型。这个模型包括两大部分：基础框架和业务。下面从模型介绍、生命周期、组件间通信三个方面来进行阐述。

#### 1. 模型介绍

（1）一个完整的App由**基础框架**（App Framework）和**业务组件**（Business）组成。

（2）Business指的是具体业务线的业务。App Framework是基础框架，不存在业务逻辑，业务线必须基于这个框架开发，可以理解为所谓的壳工程。这个App Framework由3部分组成：**基础功能组件**、**基础业务组件**和**Framework**。基础功能组件指的是网络层、数据库、日志等基础服务；基础业务组件指的是上层业务需要用到的比较独立的基础业务服务，如登录组件、支付组件、评论组件等；Framework负责基础功能组件和基础业务组件的调度和生命周期管理，因为前面说的组件都是独立无连接的，因此运行不起来，要想形成一个有机整体正常运行起来，需要依托于Framework，实现组件生命周期管理和调度。

（3）遵循向下依赖关系。即Business整体依赖于App Framework。而App Framework中Framework类似于一个调度器，它是基于基础功能组件和基础业务组件的。基础业务组件依赖于基础功能组件。我认为这种向下依赖关系可以显式依赖，即可以直接调用。

（4）关于平级依赖。通常情况下App Framework应该尽量避免平级依赖。而对于上层业务组件平级之间的依赖是很常见的。这涉及到组件间通信，属于基础框架中的Framework的职责，通常做法是提供一个**Router**或者**Mediator**进行中转。

#### 2. 生命周期

曾经看过一位前辈的文章，加上自己的一点理解，来阐述生命周期。

Android中的四大组件Activity、Service、BroadcastReceiver、ContentProvider。这些组件可以构成一个完整的Application。四大组件都是有自己的生命周期的，以Activity为例，Activity可以认为是一个小型App，提供了onCreate()、onStart()、onResume()、onPause()、onStop()、onDestroy()这些生命周期方法，包括后来的Fragment也遵循这一原则提供生命周期方法的回调。

我们自己开发过程中的组件也借鉴同样的思想，即每一个组件必须有自己的生命周期方法，每一个业务组件能够像一个App一样独立运行起来。

#### 3. 组件间通信

Android中组件间通信是通过Intent实现。

组件间通信包括两个场景：（1）打开组件的某个页面（2）调用组件某个类的某个方法。组件A表示调用着，组件B表示被调用者，下面一一阐述。

##### (1) 打开一个页面

打开一个页面，通常指的是组件A要调起组件B的某个`Activity`或`Fragment`，这种情况往往采取的做法是URL Schema方式实现页面跳转，格式为`schema://host/action?param1=value1&param2=value2`。关于schema映射表维护以及schema解析和路由都是App Framework中的Framework来维护，需要上面提到的Router或Mediator这么一个角色。其中schema映射表可以做成后台配置文件形式。也可以代码维护，不过组件需要预先注册。

以上说的对于传递基本参数是没啥问题的。对于`Serializable`或`Parcel`对象传递也没啥问题，转成JSON String就可以了。而且通常也不会出现调起一个`Activity`的时候，需要传递一个非序列化对象，如ImageView过去的情况，真要处理`ImageView`的话很多公司直接把图片处理做成一个基础功能组件，业务组件是可以显式调用的。因此大部分公司组件间通信采用的是这种方式。

然而事实上只考虑这种情况是不全面的，我们要从更高的层面来考虑问题。

##### (2) 调用某个类的某个方法

假设组件A调用组件B的某个类的某个方法。由两种方案可以达到目的。

第一种，在Mediator中定义组件B的`interface`接口，组件B依赖`Mediator`并实现对应的interface接口方法。然后当组件A调用组件B的某个方法时，组件A依赖Mediator去调用即可。貌似蘑菇街的protocol-class方案就是这么搞的。

第二种，Mediator中不定义任何组件的interface接口，直接通过**反射机制**（OC中的runtime），将参数等穿进去，实现反射调用。这样的话组件B是不需要依赖Mediator的。不过建议是Mediator对于不同组件的反射调用能提供一组映射关系，而不是写到一起，这样会导致Mediator太大太杂。这个思路就是casa大神说的Category方案。

上面介绍的反射方式可以传任何类型的参数，嗯。

备注：Java中的interface相当于OC中的protocol。

##### (3) 小结

上面两条其实是从两种情况来考虑的，然而既然是架构，我们还是不希望出现，if else这种低级思考问题的方式。最好是能抽象出一个统一的方案。好，根据上面我们是能发现（2）能解决（1）的问题，而（1）不能解决（2）的问题，即（1）是（2）的子集，因此从技术咖层面来讲，最终得出的结论就是采用反射机制实现组件间通信这种方案比使用URL Schema这种方案更全面的。

### 总结

1.理清依赖关系，要有抽象思维

2.才疏学浅，有理解不到位的地方，还请多多指教

3.疑问：casa大神的去core化以及去model化还是不太能够理解，需要继续学习

### 参考文献

1. [http://casatwy.com/iOS-Modulization.html](http://casatwy.com/iOS-Modulization.html)

2. [http://blog.cnbang.net/tech/3080/](http://blog.cnbang.net/tech/3080/)

------------------------------------

**个人公众号：学姐的IT专栏**

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)
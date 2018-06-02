---
layout: post
comments: true
title: "Android学习笔记——热修复"
description: "Android学习笔记——热修复"
category: Android
tags: [Android]
---


**1、业界热修复方案**    
**2、代码热修复**    
**3、资源热修复**    
**4、SO库热修复**    
**5、参考文档**    


<!--more-->

### 1、业界热修复方案    

阿里系：Dexposed／Andfix，基于Xposed框架的AOP技术进行native hook，属于方法级粒度。优点是可以做到实时生效，缺点是需要考虑dalvik／art虚拟机适配、指令集的兼容问题，兼容性上会有一定的影响。    

腾讯系：QQ空间(Nuwa)／手Q(QFix)／Tinker，基于ClassLoader的方式加载multidex，下次启动时生效。缺点是需要反射更改DexElements，改变Dex的加载顺序，这使得patch需要在下次启动时才能生效，实时性就受到了影响。

美团：Robust，基于Instant Run方案，基于ClassLoader加载补丁包，不涉及到修改dexElements数组的情况，高兼容性，实时生效。    

Sophix：阿里系，综合以上方案的优点，同时包含native hook和ClassLoader方案。    
对于`较小程度的修改`，采用native hook方式实现实时生效。基于Andfix方案的改进，只不过忽略底层ArtMethod结构的差异，整体替换ArtMethod，而不是逐个属性替换。    
对于`较大程度的修改`，区分Dalvik和Art系统，Dalvik下全量dex方案，Art下补丁dex命名为classes.dex，基于Tinker方案的改进。    
`Dalvik`情况，在基线dex里去掉补丁dex中包含的Class，保证没有重复Class，然后dexmerge基线dex和补丁dex，从而保证dexopt的效果（只需要移除Class定义的入口即可，不删除具体内容）。    
`Art`情况，补丁dex命名为classes.dex，原来的dex依次命名为classes(2,3,4...).dex，然后一起打包为一个压缩文件；加载dex的时候调用DexFile.loadDexFiles得到dexFiles数组；然后整个替换掉旧的dexElements数组就可以了。    

上面主要是代码热修复的方案，资源热修复和so热修复后面有介绍。

### 2、代码热修复    

两大类方案：`底层替换方案`、`类加载方案`。

（1）底层替换方案（热部署）    

原理：在加载了的类中直接`替换原有方法`。

业务方案：Dexposed／Andfix等Hook方案

#### Dexposed
    
基于Xposed的AOP方案，通过修改Dalvik运行时的Zygote进程，使用Xposed Bridge来hook方法并注入自己的代码。只Hook App本身进程，不需要Root权限。    
缺点：不支持art系统。    

#### Andfix    

也是基于Xposed的AOP方案。

实现：（利用的是虚拟机调用方法的原理）    
（1）在native层定义和Android原生系统源码一样的类（如Art系统的ArtMethod）。    
（2）在Java层调用 native replaceMethod(Method src, Method dest)方法，逐个替换ArtMethod里的属性。（src为原来方法，dest为补丁方法）    

优点：补丁小，加载迅速，能够实时生效无需重新启动App。

缺点：    
无法实现对原有类进行方法和字段的增减。（否则`方法索引`和`字段索引`会发生变化）    
系统兼容性。Android是开源的，各家手机厂商可能对系统的ArtMethod进行修改，导致其结构发生变化，而Andfix方案的ArtMethod是自己参考源码写死的。    

适合场景：适合较小程度的修改，不适合大修改和被修复方法反射调用的情况。    

Sophix方案：忽略底层ArtMethod结构的差异，整体替换ArtMethod，而不是逐个属性替换。    

    memcpy(smeth, dmeth, sizeof(ArtMethod));

![](/image/2018-05-25-learning-notes-hotfix/ArtMethod.png)   

（2）类加载方案（冷启动）    

原理：在App重新启动后让ClassLoader去加载新的类。

业界方案：QQ空间／手机QQ／Tinker    

#### QQ空间

实现：    
（1）单独放一个帮助类在dalvikhack.dex中，让其他dex中的类去调用，阻止打上`CLASS_ISPREVERIFIED`标志。    
（2）将补丁dex插入dexElements数组的最前面    

关于CLASS_ISPREVERIFIED：   
（1）APK第一次安装的时候，会调用dvmVerifyClass(clazz)对原dex进行校验，防止类被篡改。如果被校验的类的所有方法中`直接引用`到的类和当前类在同一个dex的话，这个类就会被打上CLASS_ISPREVERIFIED标志。       
（2）在执行方法调用时，也会做校验。对于被打上CLASS_ISPREVERIFIED标记的类，会校验它的dex和被调用的类的dex是否属于同一个dex，如果不属于则会抛异常。    
存在问题：当原来的dex中类的某个方法调用到补丁dex中的某个类时，会抛异常。    
解决方案：一个单独无关的帮助类放到dalvikhack.dex中，侵入原dex打包流程，利用.class字节码修改技术，在所有.class文件的构造函数中引用这个帮助类，防止被打上CLASS_ISPREVERIFIED标志。（`插桩技术`）

缺点：    
（1）该方案侵入打包流程，同时添加一些臃肿的代码，实现起来不够优雅。    
（2）影响类加载阶段的性能。类加载的3个阶段`dvmResolveClass->dvmLinkClass->dvmInitClass`，打CLASS_ISPREVERIFIED/CLASS_ISOPTIMIZED标记是在dvmResolveClass中进行的，如果类没有打上标记，则会在dvmInitClass阶段执行verify和optimize操作，而verify是很重的操作，这样会影响类加载的性能。（相当于将verify/optimize 由原来的类安装阶段延迟到类加载阶段）    
（3）在ART模式下，如果类修改了结构，就会出现内存错乱的问题。为了解决这个问题，就必须把所有相关的调用类、父类、子类全部加载到patch.dex中，导致补丁包异常大，应用启动变慢。    

![](/image/2018-05-25-learning-notes-hotfix/dexopt.png)   

#### 手Q    

实现：    
基于QQ空间方案的改进，在dvmResolveClass阶段，保证dvmDexGetResolvedClass() != null，从而绕过后面的dex比较的过程。    
缺点：需要获取底层虚拟机的函数，不够稳定可靠。

#### Tinker

实现：    
利用dexmerge工具，将补丁dex和原来dex合并成一个完整的dex。然后用合并后的dex替换原来的dex。    
对于多dex的情况，假设有5个dex文件，分别修改了这5个dex，则需要产生5个patch.dex补丁包，进行5次patch合并动作，然后替换原来的5个dex文件。    
dex加载过程。Dalvik只加载classes.dex，其他dex被直接忽略；Art支持多dex加载。    

优点：节省空间

缺点：    
（1）从dex的方法和指令维度全量合成，实现复杂，性能消耗严重，总体性价比不高。    
（2）dexmerge会导致内存风暴，内存不足时merge会失败；同时多dex下dexmerge会有65535方法数超出的问题。    

#### Robust（Instant Run）    

（1）对每个方法在编译打包阶段自动插入一段代码。当changeQuickRedirect != null时，执行补丁包逻辑从而替换掉之前老的逻辑，达到fix的目的。    

    //修改前
    public class State {
        public long getIndex() {
            return 100;
        }
    }

    //修改后
    public class State {
        //每个Class增加静态成员变量    
        public static ChangeQuickRedirect changeQuickRedirect;
        public long getIndex() {
            //每个方法前插入判断逻辑
            if(changeQuickRedirect != null) {
                //PatchProxy中封装了获取当前className和methodName的逻辑，并在其内部最终调用了changeQuickRedirect的对应函数
                if(PatchProxy.isSupport(new Object[0], this, changeQuickRedirect, false)) {
                    return ((Long)PatchProxy.accessDispatch(new Object[0], this, changeQuickRedirect, false)).longValue();
                }
            }
            return 100L;
        }
    }

（2）生成patch.dex。包含StatePatch和PatchesInfoImpl两个类    
比如要修改getIndex()的返回值为106。

    public class StatePatch implements ChangeQuickRedirect {
        @Override
        public Object accessDispatch(String methodSignature, Object[] paramArrayOfObject) {
            String[] signature = methodSignature.split(":");
            if (TextUtils.equals(signature[1], "a")) {//long getIndex() -> a
                return 106;
            }
            return null;
        }

        @Override
        public boolean isSupport(String methodSignature, Object[] paramArrayOfObject) {
            String[] signature = methodSignature.split(":");
            if (TextUtils.equals(signature[1], "a")) {//long getIndex() -> a
                return true;
            }
            return false;
        }
    }

    public class PatchesInfoImpl implements PatchesInfo {
        public List<PatchedClassInfo> getPatchedClassesInfo() {
            List<PatchedClassInfo> patchedClassesInfos = new ArrayList<PatchedClassInfo>();
            //com.meituan.sample.d是com.meituan.sample.State混淆后的名字
            PatchedClassInfo patchedClass = new PatchedClassInfo("com.meituan.sample.d", StatePatch.class.getCanonicalName());
            patchedClassesInfos.add(patchedClass);
            return patchedClassesInfos;
        }
    }

（3）客户端用DexClassLoader加载patch.dex，反射调用PatchesInfoImpl.getPatchedClassesInfo()方法，得到需要被替换的Class列表信息即com.meituan.sample.d，将com.meituan.sample.d class中的changeQuickRedirect赋值为StatePatch.java。

优点：    
（1）实时生效
（2）方法级别的修复    

缺点：    
（1）代码是侵入式的，会在原有类中加入相关代码，且会增加方法数，会增大apk的大小。    
（2）暂不支持资源文件和so    


Sophix完整方案：Dalvik下全量dex方案，Art下补丁dex命名为classes.dex。

Dalvik全量dex方案：在基线dex里去掉补丁dex中包含的Class，保证没有重复Class，然后dexmerge基线dex和补丁dex，从而保证dexopt的效果（只需要移除Class定义的入口即可，不删除具体内容）。    
优点：不需要判断类的增加或减少，不需要处理合成时方法数超过65535的情况。    

![](/image/2018-05-25-learning-notes-hotfix/classdef.png)   

Art整体替换方案：补丁dex命名为classes.dex，原来的dex依次命名为classes(2,3,4...).dex，然后一起打包为一个压缩文件；加载dex的时候调用DexFile.loadDexFiles得到dexFiles数组；然后`整个替换`掉旧的dexElements数组就可以了。    

![](/image/2018-05-25-learning-notes-hotfix/tinker_dex.png)   

`Sophix方案：底层替换方案 + 类加载方案` 

### 3、资源热修复    

业界方案：Instant Run。    

（1）构造一个新的AssetManager，并通过反射调用addAssetPath，把这个`完整的新资源包`加入到AssetManager中。这样就得到了一个含有所有新资源的AssetManager。    
（2）找到所有之前引用到原有AssetManager的地方，通过反射把引用处替换为新的AssetManager。    

Sophix方案：    
构造一个package id为0x66的资源包（已加载的资源包package id为0x7f），这个包里只包含变化的资源项。然后直接在原有AssetManager中addAssetPath这个包即可。    
优点：    
（1）不侵入打包，直接对比新旧资源产生补丁资源；    
（2）不必下发完整包，补丁包中只包含变动的资源；    
（3）不需要在运行时合成完整包，不占用运行时计算和内存资源    

### 4、SO库热修复    

本质上是对native方法的修复和替换。

业界：多采用接口调用替换方式。

Sophix方案：类修复反射注入方式。启动期间，把补丁so库的路径插入到nativeLibraryDirectories数组的最前面，这时候加载的就是补丁so

### 5、参考文档    

（1）[Android热修复技术原理详解](https://www.cnblogs.com/popfisher/p/8543973.html)    
（2）[Android热更新方案Robust](https://tech.meituan.com/android_robust.html)    
（3）[Android热修复技术选型——三大流派解析](http://www.infoq.com/cn/articles/Android-hot-fix)    
（4）阿里出版的《深入探索Android热修复原理》    

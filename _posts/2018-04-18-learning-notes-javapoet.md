---
layout: post
comments: true
title: "Java学习笔记——JavaPoet"
description: "Java学习笔记——JavaPoet"
category: Java
tags: [Java]
---


**1、JavaPoet简介**    
**（1）为什么使用？**    
**（2）怎么使用？**    
**2、JavaPoet原理**    
**3、Annotation**    
**4、参考文档**    

<!--more-->

### 1、JavaPoet简介

#### （1）为什么使用？

`JavaPoet`主要用在编译期生成代码的场景。很多常见的开源库都有使用JavaPoet，比如：ButterKnife、EventBus、Retrofit、GreenDAO...

**使用JavaPoet的优势：**    
（1）节省开发阶段重复代码的编写。开发者通过使用注解，将重复代码（如findViewById）在编译阶段由javapoet生成，除此之外对程序并没有产生什么差别，如ButterKnife。    
（2）注解替代反射。一些需要用到反射的场景，可以考虑使用JavaPoet替代，提高性能。

#### （2）怎么使用？

javapoet需要配合auto-service库一起使用：

    compile 'com.squareup:javapoet:1.8.0'
    compile 'com.google.auto.service:auto-service:1.0-rc3'

基本的调用流程如下图所示：

![](/image/2018-04-18-learning-notes-javapoet/invoke-flow.png)
    
（1）在编译期，编译器先去查找auto-service.jar里`META-INF/services`下注册的接口及服务。发现文件对应的接口`javax.annotation.processing.Processor`及实现类`com.google.auto.service.processor.AutoServiceProcessor`    
（2）ServiceLoader加载AutoServiceProcessor，并调用它的process()方法    
（3）AutoServiceProcessor查找@AutoService标签标注的javax.annotation.processing.Processor接口的实现类`XXXProcessor`，并将其路径写到`META-INF/services/javax.annotation.processing.Processor`文件里    
（4）ServiceLoader加载XXXProcessor，并调用它的process()方法    
（5）XXXProcessor使用JavaPoet生成源代码

简单观摩下auto-service.jar的结构：

![](/image/2018-04-18-learning-notes-javapoet/auto-service.png)

再看下XXXProcessor是怎么使用JavaPoet生成代码文件的：

![](/image/2018-04-18-learning-notes-javapoet/xxxprocessor.png)

生成的代码文件如图：

    package com.mouxuejie.xxx.router;

    public class Router_Mapping {
      public static String func() {
          return "hello world";
      }
    }

### 2、JavaPoet原理

`利用JavaPoet生成Java类文件，本质上是将外界传进来的属性值按照Java类格式规范依次转换成String，并将String写进类文件。`

我们先看搞清楚Java类文件的基本结构：

![](/image/2018-04-18-learning-notes-javapoet/java-file.png)

之前有一篇Android编码规范的文章讲过怎么规范写一个类的代码：[Android编码规范](http://mouxuejie.com/blog/2016-08-28/android-code-style/)，结合起来看很容易理解上面的结构图。

我们结合上面结构图来看，发现主要有几个必不可少的元素：

- JavaFile Java文件
- TypeSpec 类
- FieldSpec 变量
- MethodSpec 方法
- CodeBlock 代码块
- AnnotationType/TypeName/Modifier等根据需求所需

以上元素存在包含关系，每个元素有一系列属性。我们可以使用`Builder模式`构建每一个元素。

**上面第一部分提到使用JavaPoet，整个过程结合起来看就是：**    
（1）外部先构造JavaFile对象    
（2）由JavaFile对象生成Java类文件    

看下JavaFile调用入口的实现：

![](/image/2018-04-18-learning-notes-javapoet/java-file-code1.png)

![](/image/2018-04-18-learning-notes-javapoet/java-file-code2.png)

**我们可以得出由JavaFile生成Java类文件的流程：**    
（1）根据`JavacFiler`结合Java类文件路径生成FilerOutputJavaFileObject对象    
（2）根据FilerOutputJavaFileObject对象，得到FilerWriter，FilerWriter extends java.io.FilterWriter    
（3）遍历JavaFile两次。第一次目的是得到suggestedImports，writer传的NULL_APPENDABLE，不进行写操作；第二次是结合suggestedImports和JavaFile，按照最上面结构图的顺序，利用CodeWriter将CodeBlock和String等写进类文件（暂时不讲修饰符／返回值等的处理……    

其中，suggestedImports ＝ importableTypes - referencedNames

**将CodeBlock写进Java类文件的流程：**    
（1）将代码串转换成CodeBlock对象。其中普通String和特殊字符（`$L、$N、$S、$T、$$、$>、$<、$[、$]、$W`）依次放到formats列表，args放到args列表    
（2）依次遍历formats列表，如果遇到上述特殊字符则要么从args列表中取值出来填充，要么做出其他相关处理。    
（3）最终得到的还是String，将String写进FilerWriter    

附上CodeBlock的构建方式：    

    public static CodeBlock of(String format, Object... args) {
        return new Builder().add(format, args).build();
    }

网上讲JavaPoet原理的文章比较少，主要参考源码和官方文档。

### 3、Annotation

这部分基础知识另外再单独总结。

### 4、参考文档

（1）[https://github.com/square/javapoet](https://github.com/square/javapoet)    
（2）[ServiceLoader实现原理](https://blog.csdn.net/is_zhoufeng/article/details/50722440)    
（3）[从0到1：实现 Android 编译时注解](https://www.jianshu.com/p/9e34defcb76f)




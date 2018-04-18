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


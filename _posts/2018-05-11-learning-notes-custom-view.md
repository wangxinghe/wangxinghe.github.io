---
layout: post
comments: true
title: "Android学习笔记——自定义View"
description: "Android学习笔记——自定义View"
category: Android
tags: [Android]
---

**1、自定义View**    
**（1）整体结构及绘制流程**    
**（2）onMeasure**    
**（3）onLayout**    
**（4）onDraw**    
**2、measure过程**    
**（1）MeasureSpec(mode/size)**    
**（2）父View宽高的测量方法**    
**（3）子View宽高的测量方法**    
**3、layout过程**    
**（1）left/top/right/bottom含义**    
**（2）left/top/right/bottom计算**    
**4、draw过程**    
**（1）绘制顺序**    
**5、源码举例**    
**（1）LinearLayout**    
**（2）ListView**    
**（3）ScrollView**    
**6、参考文档**

<!--more-->

### 1、自定义View    

#### （1）整体结构及绘制流程    

Activity、Window、DecorView之间的关系：    

![](/image/2018-05-11-learning-notes-custom-view/Window.svg)

View的绘制流程：    

![](/image/2018-05-11-learning-notes-custom-view/view_render.png)    


#### （2）onMeasure    

#### （3）onLayout    

#### （4）onDraw    

### 2、measure过程    
    
#### （1）MeasureSpec(mode/size)    

#### （2）父View宽高的测量方法    

#### （3）子View宽高的测量方法    

### 3、layout过程    

#### （1）left/top/right/bottom含义    

#### （2）left/top/right/bottom计算    

### 4、draw过程    

#### （1）绘制顺序    

### 5、源码举例    

#### （1）LinearLayout    

#### （2）ListView    

#### （3）ScrollView    

### 6、参考文档
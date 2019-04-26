---
layout: post
comments: true
title: "Kotlin静态代码检测调研"
description: "Kotlin静态代码检测调研"
category: Kotlin
tags: [Kotlin]
---

## Kotlin静态代码检测调研

和Java静态代码检测一样，Kotlin静态代码检测包含两方面的检查：`代码风格`和`代码质量`两方面。

<!--more-->

Java代码风格检查使用的是CheskStyle，代码质量检查使用的是FindBugs

Kotlin作为一门新兴语言，代码检测的方案大概有如下4种：    
1. Ktlint [https://github.com/pinterest/ktlint](https://github.com/pinterest/ktlint)    
2. Detekt [https://github.com/arturbosch/detekt](https://github.com/arturbosch/detekt)    
3. 增强Android Studio的Lint对Kotlin代码检测的能力    
4. 增加FindBugs对Kotlin代码检测的能力 [http://findbugs.sourceforge.net/](http://findbugs.sourceforge.net/)    

静态代码检测整体思路都差不多，使用Kotlin编译器解析Kotlin类分析语法树进行处理，最大差别在于RuleSet规则集的不一样。

### 方案分析

（1） `Ktlint`主要用来对代码风格进行检查，和Java中的CheckStyle类似，规则参考的是Kotlin官方代码风格规范 [https://developer.android.com/kotlin/style-guide](https://developer.android.com/kotlin/style-guide)

规则集在ktlint-ruleset-standard目录下    
![ktlint-rules](/image/2019-04-26-kotlin-static-code-analysis/ktlint-rules.png)

（2）`Detekt`包括代码风格和代码质量两个方面的检查。    
代码风格检查沿用了Ktlint的功能，在其基础上进行了一层包装。    
代码风格的规则集在detekt-formatting目录下    
![detekt-format-rules](/image/2019-04-26-kotlin-static-code-analysis/detekt-format-rules.png)

随便点一个类进去看，发现就是使用的Ktlint的Rule。    
![detekt-rule-sample](/image/2019-04-26-kotlin-static-code-analysis/detekt-rule-sample.png)

代码质量的规则集主要在detekt-rules目录下，一个包对应一个分类，每个包下的一个类对应一个rule case。    
bugs和performance包下是一些潜在bug和性能的规则集。    
![detekt-rules](/image/2019-04-26-kotlin-static-code-analysis/detekt-rules.png)

（3）`Lint`也支持Kotlin代码检查，但是看起来只是支持简单的代码风格检查，不够完善。    
![lint-kotlin](/image/2019-04-26-kotlin-static-code-analysis/lint-kotlin.png)

美团点评通过自定义Lint的方式实现Kotlin代码检查    
[https://tech.meituan.com/2018/07/05/kotlin-code-inspect.html](https://tech.meituan.com/2018/07/05/kotlin-code-inspect.html)

这个文档是2018年写的，到现在已经过去一年了。

（4）`FindBugs`暂时是不支持Kotlin代码检查的

### 总结分析

（1）`Ktlint`只支持代码风格检查，如果要支持代码性能检查的话，需要大量扩展代码性能规则集    
（2）`Detekt`支持代码风格检查和代码性能检查，代码风格检查完全复用Ktlint，代码性能检查规则集也比较完善，且支持规则集扩展，并且开源一直在维护中，代码更新比较频繁。    
（3）`Lint`暂时只支持简单的代码风格检查，想扩展的话需要把Lint检测原理搞懂，并且需要写大量规则集，这些东西Detekt其实已经帮我们做了    
（4）`FindBugs`完全不支持Kotlin检查，采用这种方案需要我们修改FindBugs，从零开始去搞清楚怎么检查Kotlin代码，以及从零开始去定义规则集    

### 最终结论

使用Detekt方案是最合适，性价比是最高的。

### 方案实施

（1）先接入`Detekt`，使用已有的代码检测能力。    
（2）随着Kotlin代码的深入使用，对于`Detekt`不支持的性能问题，可以参考如下一些性能点去扩展，达到和FindBugs一样的能力。    

Detekt自定义规则集扩展方法：    
[https://arturbosch.github.io/detekt/extensions.html](https://arturbosch.github.io/detekt/extensions.html)

Kotlin性能点参考：    
[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62)    
[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-2-324a4a50b70](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-2-324a4a50b70)    
[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4)    


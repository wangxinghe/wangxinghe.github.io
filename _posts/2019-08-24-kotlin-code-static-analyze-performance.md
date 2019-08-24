---
layout: post
comments: true
title: "Kotlin静态代码检测——性能检测"
description: "Kotlin静态代码检测——性能检测"
category: Kotlin
tags: [Kotlin]
---

前一段时间研究了下Kotlin基于Detekt实现性能检测，该做一个阶段性总结了。

目录：        
1. 为什么要做？        
2. 为什么要这么做？        
3. 怎么做？        
4. 方法论提炼        

###  1. 为什么要做？

从今年3月份开始，我们项目的新代码基本上都是Kotlin写的，Kotlin代码量占比已超过13%。        
由于大家都是新学习，基本上还处于使用和熟悉语法阶段。不管是从项目角度还是个人追求角度，针对Kotlin的性能优化都是一个避免不了的课题。        

对于工作了一段时间的人来说，我们学习一门新编程语言并不是从0开始的，我们有其他编程语言的基础，很多经验是可以借鉴平移的。由于Java静态代码性能检测使用findbugs，那么只要能找到或实现Kotlin版的就满足了我的诉求。

###  2. 为什么要这么做？

之前调研Kotlin静态代码检测的时候，就得出使用Detekt的结论。Detekt已经提供了如下检测bugs和performance的规则集，但是我觉得能力还不够强大，还需要扩充。

![Alt text](./1566647124200.png)

![Alt text](./1566647145544.png)

而且Detekt是支持扩展的，源码和官方文档都有相关范例参考。

### 3. 怎么做？

**（1）找到性能优化点**        
具体可参考：[http://mouxuejie.com/blog/2019-07-20/kotlin-performance/](http://mouxuejie.com/blog/2019-07-20/kotlin-performance/)

**（2）写扩展规则集**        
网上相关的资料基本上没有，唯一可以参考的就是源码和官方文档。

新建一个detektExt的android library        
![Alt text](./1566649828759.png)

由于官方demo是kotlin脚本，而我们项目用groovy脚本，因此build.gradle文件配置参考        https://github.com/vanniktech/kotlin-on-code-quality-tools/tree/master/custom-detekt-rules

Rule规则集参考Detekt自带的规则集源码，刚开始看的时候可能一脸懵逼，因为没有注释，不知道每个方法每个类代表的具体含义。没关系，多研究Detekt源码多打log日志慢慢就找到感觉了。遗憾的是，我想断点调试一直没成功，不然效率会高很多。

我们重点关注`implementation 'org.jetbrains.kotlin:kotlin-compiler-embeddable:1.3.21'`，这个基本上就是Kotlin/compiler目录下的源码。Detekt静态代码检测用到的API都在/org.jetbrains/kotlin/psi目录下，其中最重要的2个类是`KtTreeVisitorVoid`和`KtElement`：

`KtTreeVisitorVoid`：为外部提供了访问KtFile的每个元素（KtFunction/KtExpression/KtProperty/KtParameter等）的方法visitXXX        
`KtElement`：Kotlin语法树中代表一个元素的抽象类，具体的元素有KtFunction/KtExpression/KtProperty/KtParameter等        

Detekt只支持词法分析和语法分析，不支持语义分析。而且我也不打算细研究具体的语法，感觉性价比不高，知道怎么做就行了，毕竟还有很多重要的东西要研究。

下面是一个检查非空基本类型参数的示例，因为涉及到装箱的性能开销：

    /**
	 * <noncompliant>
	 * fun foo(a: Int?) {
	 *     a + 1
	 * }
	 * </noncompliant>
	 *
	 * <compliant>
	 * fun foo(a: Int) {
	 *     a + 1
	 * }
	 * </compliant>
	 */
	class PrimitiveParameterNotNullable(config: Config) : Rule(config) {

	    override val issue = Issue(javaClass.simpleName,
	        Severity.Performance,
	        "Primitive parameter type should not be nullable.",
	        Debt.FIVE_MINS)

	    private val nullablePrimitiveTypes = hashSetOf(
	        "Int?",
	        "Double?",
	        "Float?",
	        "Short?",
	        "Byte?",
	        "Long?",
	        "Char?"
	    )

	    override fun visitParameter(parameter: KtParameter) {
	        val typeReference = parameter.typeReference
	        if (typeReference != null && nullablePrimitiveTypes.contains(typeReference.text)) {
	            report(CodeSmell(issue, Entity.from(parameter), "Primitive parameter type should not be nullable."))
	        }
	        super.visitParameter(parameter)
	    }
	}

### 4. 方法论提炼

**（1）经验积累和触类旁通**        
我可以把Java的一些开发经验平移到Kotlin开发中。        
在计算机编程领域，很多经验应该都是共通的，我们学习任何新东西起点都不是0，经验越多学习新东西越快。        
如果善于归纳总结、触类旁通的话，哪怕是一些没做过的东西，也可以给出一个比较靠谱的方案。        

**（2）一个知识点可以延伸出很多东西**        
通过研究Detekt自定义规则集：        
我能推断出，Java静态代码检测的CheckStyle/FindBugs/Lint实现思路应该都差不多。        
我能推断出，一个静态代码检测工具需要提供的角色：元素访问者Visitor，规则集Rules，日志输出Reporter。写一个静态代码检测的工具应该不难。        
静态代码检测，可以引申出Java/Kotlin编译原理。        
......        

每一个点都不是一言两语可以讲清楚的，需要不断积累。

最后，大家如果有什么想法，可以和我交流。

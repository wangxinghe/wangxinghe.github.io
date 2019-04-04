---
layout: post
comments: true
title: "Kotlin学习笔记——基础篇"
description: "Kotlin学习笔记——基础篇"
category: Kotlin
tags: [Kotlin]
---

## Kotlin学习笔记——基础篇

之前Kotlin没有正式写过，年后组内听过一次分享，03月份开始真正用Kotlin写业务需求。

### 1.基础语法

英文官网：https://kotlinlang.org/docs/reference/
中文官网：https://www.kotlincn.net/docs/reference/

协程学习资料：
https://www.kotlincn.net/docs/reference/coroutines-overview.html

<!--more-->

主要涉及的知识点：
#### （1）基础语法
包 `package`
函数 `fun`
变量 `var, val`
字符串模块 `$a, ${a.b}`
条件表达式 `if else, when 二元条件优先使用if`
可空值及null检测 `Int? `
类型检测 `is, !is`
循环 `for, while`
区间 `for(x in 1..5)包含5, for(x in 1 until 5)不包含5, downTo/step`

#### （2）习惯用法
数据类 `data class Customer(val name: String, val email: String)`
具备以下能力：
所有属性的getters（对于var定义的还有setters）
equals()
hashCode()
toString()
copy()

过滤list `list.filter{ it > 0 }`

遍历 map/pair型list
	
	for ((k, v) in map) {
		println("$k -> $v")
	}

只读list `listOf("a", "b", "c")`
只读map `mapOf("a" to 1, "b" to 2, "c" to 3)`
延迟属性 

	val p: String by lazy {
	}
扩展函数 
	
	fun String.spaceToCamelCase() {...}
	"Convert this to camelcase".spaceToCamelCase()

创建单例

	object Resource {
		val name = "Name"
	}
if not null `?.`
if null执行语句 `?:`
if not null执行代码

	value?.let{
	   //如果value != null，执行到这里
	}
对一个对象实例调用多个方法 `with`
交换两个变量 `a = b.also { b = a }`

类布局：
属性声明与初始化块
次构造函数
方法声明
伴生对象

#### （3）基本类型

#### （4）类与对象
类与继承
属性与字段
接口
可见性修饰符
扩展
数据类
密封类
泛型
嵌套类
枚举类
对象
类型别名
内联类
委托
委托属性

#### （5）函数

#### （6）其他
解构声明
集合
区间
类型检查与转换
This表达式
相等性
操作符重载
空安全
异常
注解
反射
作用域函数

### 2.遇到的问题

开发过程中问题记录：
（1）非空成员变量，如果没有初始化，要用`lateinit`标识；对于可空的成员变量，不需要lateinit
（2）`const val`常量，只能在top level文件最外层、object单例类或companion object伴生对象中声明
（3）构造函数，分为主构造函数和次构造函数，主构造函数可以直接写在类后面，也可以使用`constructor`声明主次构造函数，`init`自动在主构造函数中执行
（4）字符串拼接，单变量字符`$a`，复杂表达式`${a.b}`，特殊字符需要转义，转义可以使用`\***`或`${'***'}`
参考：[Kotlin入门(5)字符串及其格式化](https://zhuanlan.zhihu.com/p/27789205)

（5）while语句中不能使用赋值语句，解决方法：
Assignments are not expressions,and only expressions are allowed in this context
	
	错误示范
	val br = BufferedReader(InputStreamReader(conn.inputStream))
	var output: String
	while ((output = br.readLine()) != null) {
		println(output)
	}

	正确示范1
	val reader = BufferedReader(reader)
	reader.lineSequence().forEach {
	    println(it)
	}
	正确示范2
	BufferedReader(reader).use { r ->
	    r.lineSequence().forEach {
	        println(it)
	    }
	}
	正确示范3
	generateSequence { br.readLine() }
          .forEach { println(output) }
		
参考：https://stackoverflow.com/questions/41537638/assignment-not-allowed-in-while-expression/41537792

（6）数组
	
	创建数组
	arrayOf(1, 2, 3)/intArrayOf()/longArrayOf()/...
	创建指定长度数组
	arrayOfNulls<Int>(6)/IntArray(3)
	创建空数组
	emptyArray<Int>()

	访问数组元素（元素遍历，索引遍历）
	val intArr = intArrayOf(1,2,3)
	for (item in intArr) {
	    println(item)
	}
	for (index in intArr.indices){
	    println(intArr[index])
	    println(intArr.get(index))
	}

	多维数组
	Array(3){IntArray(3)}
参考：https://www.jianshu.com/p/e75795be48c8

（7）this表达式引用外部类对象
在类的成员中，this 指的是该类的当前对象。
在扩展函数或者带有接收者的函数字面值中， this 表示在点左侧传递的 接收者 参数。
如果 this 没有限定符，它指的是最内层的包含它的作用域。要引用其他作用域中的 this，请使用 标签限定符：
	
	class A { // 隐式标签 @A
	    inner class B { // 隐式标签 @B
	        fun Int.foo() { // 隐式标签 @foo
	            val a = this@A // A 的 this
	            val b = this@B // B 的 this
	            val c = this // foo() 的接收者，一个 Int
	        }
	    }
	}
参考：https://www.kotlincn.net/docs/reference/this-expressions.html

（8）Kotlin调用Java方法时，注意基本类型的转换

	divider.setBackgroundColor(Color.parseColor("#FFF7F8FC")) 这样可以
	divider.setBackgroundColor(0xFFF7F8FC)不行
	divider.setBackgroundColor(0xFFF7F8FC.toInt()) 可以

（9）接口传递问题
如果接口和接口传递方法定义在Java文件中，在Kotlin文件调用这个方法时，可以使用lambda表达式或匿名对象的形式，因为Java会将lambda表达式自动转换成匿名对象。
如果接口和接口传递方法定义在Kotlin文件中，在Kotlin文件调用这个方法时，只能使用匿名对象的形式，因为Kotlin不会将lambda表达式自动转换成匿名对象。否则会报interface does not have constructors

	正确示范（定义在Java文件）
	public interface OnButtonClickListener {
	    void onClick(View v, ViewGroup pv)
	}

	public void setOnButtonClickListener(OnButtonClickListener listener) {
		// do sth
	}

	setOnButtonClickListener(object: OnButtonClickListener {
	    override fun onClick(v: View, pv: ViewGroup) {
			// do sth
		}
	})
	setOnButtonClickListener {
		// do sth
	}

	正确示范（定义在Kotlin文件）
	interface OnButtonClickListener {
	    fun onClick(v: View, pv: ViewGroup)
	}

	fun setOnButtonClickListener(listener: OnButtonClickListener) {
		// do sth
	}

	setOnButtonClickListener(object: OnButtonClickListener {
	    override fun onClick(v: View, pv: ViewGroup) {
			// do sth
		}
	})

参考：https://stackoverflow.com/questions/43737785/kotlin-interface-does-not-have-constructors

（10）遇到一个按钮点击失效问题
原因是没理清：setOnClickListener()传递对象，setOnClickListener{}执行操作

	itemView.setOnClickListener { // 点击失效（创建了一个对象）
	    object : OnClickListener {
	        override fun onClick(v: View?) {
	            clickListener?.onClick(v!!, this@TaskItemView)
	        }
	    }
	}
	itemView.setOnClickListener { // 点击有效
	    clickListener.onClick(it!!, this@TaskItemView)
	}

（11）接口、方法、变量定义在文件最外层，代表全局可用

（12）相等（是否忽略大小写）a.equals("b", ignoreCase = true)

（13）smart cast to 'XXX' is impossible, because xxx is a mutable property that could have been changed by this time
对于var成员变量，可能存在读写多线程并发问题。解决方法很多，要么定义成val，要么在使用时将var实例赋值给一个val临时变量，要么使用?.let{}
参考：
https://stackoverflow.com/questions/44595529/smart-cast-to-type-is-impossible-because-variable-is-a-mutable-property-tha
https://stackoverflow.com/questions/46701042/kotlin-smart-cast-is-impossible-because-the-property-could-have-been-changed-b

（14）smart cast xxx to 'XXX' is impossible, because xxx is a property that has open or custom getter
如果某个变量被open修饰或有自定义的getter()方法，那么这个变量前后两次读取的值可能不一样，因为可能会被复写或改变。在使用的时候可能就会存在问题，感觉和上面有点类似，都是同时改变的情况。
解决方法：定义一个val临时常量，统一使用这个val常量，或者使用?.let{...}
参考：https://stackoverflow.com/questions/41086296/smartcast-is-impossible-because-propery-has-open-or-custom-getter

（15）属性getter，setter
参考：
https://kotlinlang.org/docs/reference/properties.html
https://ebnbin.com/2017/11/06/kotlin-getter-setter/

（16）静态变量，静态方法`@JvmField`, `@JvmStatic`
参考：https://www.kotlincn.net/docs/reference/java-to-kotlin-interop.html

（17）Kotlin里的Any相当于Java里的Object
参考：https://jinguoliang.github.io/blog/2017/09/16/17-Kotlin%E7%9A%84Any/


### 3.Kotlin的warning处理

在开发过程中，发现build后会有很多warning，记录如下：
（1）Parameter 'xxx' is never used, could be renamed to _
解决方法：xxx改成_

（2）调用Deprecated API警告
解决方法：context.resources.getDrawable/getColor 改成 ContextCompat.getDrawable/getColor

（3）Unnecessary safe call on a non-null receiver of type XXX?
解决方法：大概意思是，如果根据上下文可以确定某个对象非空（如已判空），则不需要再使用?.安全调用

（4）The resulting type of this 'javaClass' call is Class<String.Companion> and not Class<String>. Please use the more clear '::class.java' syntax to avoid confusion
解决方法：这里涉及到Kotlin反射的相关知识，Kotlin有个KClass，Java里是Class，要区分.javaClass和::class.java的区别，而且和类定义在Java还是Kotlin文件中相关。
参考：
https://blog.csdn.net/u012947056/article/details/78925701
https://www.jianshu.com/p/a900ee71ae7f
https://stackoverflow.com/questions/46674787/instanceclass-java-vs-instance-javaclass

（5）命名遮蔽name shadow
解决方法：Kotlin里作用域不同的同名变量可能存在命名遮蔽问题，可以使用@Suppress("NAME_SHADOWING")屏蔽警告
参考：https://juejin.im/entry/5b0bd2a3f265da08e12f0b75

（6）should be replaced with an indexing operator[]
解决方法：数组/列表建议直接使用a[i]形式访问，而不需要a.get(i)这样访问

（7）对于getXXX()方法，访问时建议直接a.xXX访问，省略get

（8）Variable is never modified and can be declared immutable using val
解决方法：能用val的时候尽量用val，val相当于Java里的final，只能进行一次赋值操作。这种问题会在Android Studio右边标黄，但是build的时候并不会有warning提示。

（9）Kotlin中的Suppress
有些warning使用了@Suppress("xxx")屏蔽警告，自己用到的有`UNUSED_PARAMETER`, `UNUSED_EXPRESSION`, `NAME_SHADOWING`，不区分大小写。
更全面的参考：
https://stackoverflow.com/questions/40604446/what-are-the-possible-values-that-can-be-given-to-suppress-in-kotlin
https://github.com/JetBrains/kotlin/blob/master/compiler/frontend/src/org/jetbrains/kotlin/diagnostics/Errors.java
https://github.com/JetBrains/kotlin/blob/master/compiler/frontend/src/org/jetbrains/kotlin/diagnostics/rendering/DefaultErrorMessages.java
https://stackoverflow.com/questions/29046636/mark-unused-parameters-in-kotlin
https://stackoverflow.com/questions/43192281/kotlin-suppress-unused-property
https://stackoverflow.com/questions/54336836/how-do-i-get-rid-of-this-warning-the-expression-is-unused-warning
https://stackoverflow.com/questions/53102976/remove-warning-of-variable-shadowing-in-kotlin
https://stackoverflow.com/questions/49680040/why-kotlin-allows-to-declare-variable-with-the-same-name-as-parameter-inside-the
当然也可以在build.gradle文件里屏蔽所有warning，但一般不建议这么做。



### 4.未完待续
下一篇学习笔记是Kotlin协程，下下一篇学习笔记是原理剖析，4月份会交作业，敬请期待～


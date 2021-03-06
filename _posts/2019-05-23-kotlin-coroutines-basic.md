---
layout: post
comments: true
title: "Kotlin Coroutines基础和原理初探"
description: "Kotlin Coroutines基础和原理初探"
category: Kotlin
tags: [Kotlin]
---


### 1.协程概念
本质上，协程是轻量级的线程。

Kotlin 1.3开始正式支持Coroutines，Coroutines 并不是 Kotlin 的发明，很多语言（如 LISP、Python、Javascript 等）都有 Coroutine 这个概念。

<!--more-->

### 2.基本使用

	// 在后台启动一个新协程，不阻塞当前线程，返回值是Job
    val job = launch {
	    delay(1000L) // 非阻塞的等待1s 
	    println("hello world")
    }

	// 和launch类似，不阻塞当前线程，但返回值是Deferred，带结果值的Job（类Java中的Future）
	val defer = async { 
		delay(1000L)
		20
	}
	defer.await() // 等待执行结果

	// 惰性启动的async，当调用start()或await()才会启动协程
	val defer = async(start = CoroutineStart.LAZY) {
		delay(1000L)
		20
	}
	defer.start() // 启动协程
	defer.await() // 等待执行结果

	// 在当前线程启动一个协程，阻塞当前线程
	runBlocking {
		delay(1000L) // 阻塞的等待1s
	}

	// 挂起函数，只能在协程或其他suspend方法中调用
    suspend fun doSth() {
		delay(1000L)
	}

	// 协程的取消
	job.cancel() // 取消任务 
	job.join() // 等待任务执行结束
	job.cancelAndJoin() // 取消任务并等待它结束

	// 协程的超时，是一个挂起函数，只能在协程或其他suspend方法中调用
	withTimeout(1000L) { // 1s后取消任务，如果任务没执行完，会抛TimeoutCancellationException
		// do some suspend task
	}
	withTimeoutOrNull(1000L) { // 1s后取消任务，返回执行结果或返空，不抛TimeoutCancellationException
		// do some suspend task
	}

### 3.几个基本概念
#### （1）CoroutineScope / GlobalScope / MainScope 协程作用域
`CoroutineScope` 是一个接口，其子类有GlobalScope和MainScope

    public interface CoroutineScope {
	    public val coroutineContext: CoroutineContext
	}
`GlobalScope`  启动的是全局性的协程，不和具体的job绑定，生命周期是Application级别，一般不建议使用
A global [CoroutineScope] not bound to any job.
Global scope is used to launch top-level coroutines which are operating on the whole application lifetime
and are not cancelled prematurely.
Another use of the global scope is operators running in [Dispatchers.Unconfined], which don't have any job associated with them.
`MainScope` 作用域是UI线程，一般用于UI组件的操作

2种基本用法：

	class Activity1 : AppCompatActivity(), CoroutineScope by MainScope() {
		override fun onDestroy() {
	        cancel() // cancel is extension on CoroutineScope
	    }
 
        fun showSomeData() = launch { // extension on current activity, launched in main thread
            // here we can use suspending functions or coroutine builders with other dispatchers
            draw(data) // draw in the main thread
	    }
	}

	 class Activity2 : AppCompatActivity() {
		 private val scope = MainScope()
		 override fun onDestroy() {
			 super.onDestroy()
			 scope.cancel()
		 }
	 }

#### （2）CoroutineContext 协程上下文
CoroutineContext是一组Element实例的集合，其中一个Element对应一个上下文元素且有唯一的Key

协程上下文包含Job和协程调度器CoroutineDispatcher

Persistent context for the coroutine. It is an indexed set of [Element] instances.
An indexed set is a mix between a set and a map.
Every element in this set has a unique [Key]. Keys are compared _by reference_.

`withContext()`
`CoroutineScope.newCoroutineContext()`

	public suspend fun <T> withContext(
	    context: CoroutineContext,
	    block: suspend CoroutineScope.() -> T
	): T = {...}

#### （3）Coroutine Builder 协程构建器

	launch {...}
	async{...}
	runBlocking{...}

launch源码解析

	public fun CoroutineScope.launch(
	    context: CoroutineContext = EmptyCoroutineContext,
	    start: CoroutineStart = CoroutineStart.DEFAULT,
	    block: suspend CoroutineScope.() -> Unit
	): Job {
	    val newContext = newCoroutineContext(context)
	    val coroutine = if (start.isLazy)
	        LazyStandaloneCoroutine(newContext, block) else
	        StandaloneCoroutine(newContext, active = true)
	    coroutine.start(start, coroutine, block)
	    return coroutine
	}

#### （4）Dispatchers 协程调度器
协程调度器可以将协程的执行局限在指定的线程中，调度它运行在线程池中或让它不受限的运行。相关类Dispatchers和ExecutorCoroutineDispatcher，分为`非受限调度器`和`受限调度器`

`非受限调度器`：`Dispatchers.Unconfined`
协程调度器会在程序运行到第一个挂起点时，在调用者线程中启动。挂起后，它将在挂起函数执行的线程中恢复，恢复的线程完全取决于该挂起函数在哪个线程执行。

`受限调度器`：
`Dispatchers.Default`  运行在默认线程
`Dispatchers.Main`  运行在主线程
`Dispatchers.IO`  运行在IO线程
`newSingleThreadContext(name: String)`  运行在新线程上下文
`newFixedThreadPoolContext(nThreads: Int, name: String)`  运行在具有固定线程数的线程池上下文

基本使用：

	launch(Dispatchers.Unconfined) {
        println("Unconfined: I'm working in caller thread") // 运行在调起线程
        delay(500)
        println("Unconfined: After delay in thread kotlinx.coroutines.DefaultExecutor") // 运行在delay执行的线程
    }

线程切换：

    newSingleThreadContext("Ctx1").use { ctx1 ->
            newSingleThreadContext("Ctx2").use { ctx2 ->
                    runBlocking(ctx1) {
                        log("Started in ctx1")
                        withContext(ctx2) {
                            log("Working in ctx2")
                        }
                        log("Back to ctx1")
                    }
            }
    }

### 4.原理初探
#### （1）Coroutine Suspend vs. Callback
传统的Callback回调方式：

    requestToken {token ->
        createPost(token, item) {post ->
            processPost(post)
        }
    }

RxJava链式方式：

    Observable.just(item).subscribeOn(Schedulers.io())
        .map {
            val token = requestToken()
            val post = createPost(token, item)
            processPost(post)
        }
        .subscribe({}, Throwable::printStackTrace)

Coroutine Suspend方式：

    suspend fun postItem(item: Item) {
        val token = requestToken()
        val post = createPost(token, item)
        processPost(post)
    }

    suspend fun requestToken(): Token {
        return Token("token123")
    }

    suspend fun createPost(token: Token, item: Item): Post {
        return Post(token.tokenStr, item.name)
    }

    suspend fun processPost(post : Post) {
        println("${post.param1},${post.param2}")
    }

    launch {
		postItem(item)
	}

#### （2）Suspend原理分析
简化后的反编译代码：

    final void postItem(Item item, Continuation var2) {
		ContinuationImpl sm = new CoroutineImpl(var2) {
            Object result;
            int label;
            Object item;
            Object token;
            Object post;

            public final Object invokeSuspend(Object result) {
               this.result = result;
               this.label |= Integer.MIN_VALUE;
               return Test.this.postItem((Test.Item)null, this);
            }
		}
		switch (sm.label) {
		case 0:
			sm.item = item
			sm.label = 1
			token = requestToken(sm)
		case 1:
			sm.item = item
			sm.token = token
			sm.label = 2	
			post = createPost(token, item, sm)
		case 2:
			sm.item = item
			sm.token = token
			sm.post = post
			sm.label = 2	
			processPost(post, sm)
		}
	}

	final Object requestToken(Continuation var1) {...}
	final Object createPost(Token token, Item item, Continuation var2) {...}
	final void processPost(Post post, Continuation var3) {...}

相关特点：    
（1）每个suspend方法会增加一个Continuation入参    
（2）编译器会为父suspend方法创建一个ContinuationImpl实例，用于对这个suspend方法自身的回调，该方法执行完会回调到其上一级的父方法    
（3）父suspend方法会将子suspend方法调用转换为Switch状态机形式，每个case对应一个子suspend方法的调用    

![kotlin-cps](/image/2019-05-23-kotlin-coroutines-basic/kotlin-cps.png)

每一个挂起点和初始挂起点对应的 Continuation 都会转化为一种状态，协程恢复只是跳转到下一种状态中。挂起函数将执行过程分为多个 Continuation 片段，并且利用状态机的方式保证各个片段是顺序执行的。

`Continuation`：

	public interface Continuation<in T> {
	    public val context: CoroutineContext
	    public fun resumeWith(result: Result<T>) // 恢复暂停的Coroutine的执行
	}

`CPS - Continuation Passing Style`：

[https://en.wikipedia.org/wiki/Continuation-passing_style](https://en.wikipedia.org/wiki/Continuation-passing_style)    
In functional programming, continuation-passing style (CPS) is a style of programming in which control is passed explicitly in the form of a continuation. This is contrasted with direct style, which is the usual style of programming.

A function written in continuation-passing style takes an extra argument: an explicit "continuation", i.e. a function of one argument. When the CPS function has computed its result value, it "returns" it by calling the continuation function with this value as the argument. That means that when invoking a CPS function, the calling function is required to supply a procedure to be invoked with the subroutine's "return" value.

CPS是一种通过显式传递Continuation控制业务流程的编程风格，不同于常见的Direct Style，有点类似于Callback。

`State Machine`：
[https://en.wikipedia.org/wiki/Finite-state_machine#Concepts_and_terminology](https://en.wikipedia.org/wiki/Finite-state_machine#Concepts_and_terminology)    
状态机表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型

#### （3）Coroutine vs. Thread
[https://en.wikipedia.org/wiki/Coroutine#Comparison_with_threads](https://en.wikipedia.org/wiki/Coroutine#Comparison_with_threads)    
[https://stackoverflow.com/questions/1934715/difference-between-a-coroutine-and-a-thread](https://stackoverflow.com/questions/1934715/difference-between-a-coroutine-and-a-thread)    

协程可以看作是能被挂起、不阻塞线程的计算，协程的挂起几乎没有代价，没有上下文切换，不需要虚拟机和操作系统的支持。协程挂起通过suspend函数实现，suspend函数用状态机的方式用挂起点将协程的运算逻辑拆分为不同的片段，每次运行协程执行不同的逻辑片段。

线程上下文切换开销大，依赖于虚拟机和操作系统。

这是我的Kotlin Coroutines的初步学习成果和产出，关于Coroutines和Thread最本质的东西，我感觉自己还没看懂，需要进一步学习再总结成文章。

### 5.参考文档
[https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)    
[https://www.kotlincn.net/docs/tutorials/coroutines/async-programming.html](https://www.kotlincn.net/docs/tutorials/coroutines/async-programming.html)    

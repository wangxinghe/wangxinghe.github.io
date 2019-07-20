---
layout: post
comments: true
title: "Kotlin性能优化"
description: "Kotlin性能优化"
category: Kotlin
tags: [Kotlin]
---

昨天看了下，我们项目里Kotlin代码量已经占到13%了，是时候更深刻的了解自己写的代码都是一坨什么东西了。

就自己目前的认知，对于大部分程序员来说，所谓编程语言层面的优化，无非就是更好的掌握这门语言，掌握哪些用法是高效的，哪些用法存在性能损耗。一般的思路就是查看生成的字节码，看有没有在时间复杂度或空间复杂度上带来损耗。
Kotlin查看字节码的方式：Android Studio -> Tools -> Kotlin -> Show Kotlin Bytecode

<!--more-->

注意：如果觉得字节码看起来不太直观，也可以通过Decompile成Java代码的方式来看，但是反编译成Java代码可能看不出性能损耗点，所以最靠谱的方式还是字节码。（关于字节码的细节，AOP字节码插桩专题再详细讨论）

几类常见的Kotlin性能损耗点：
1. Companion Object-伴生对象
2. Higher-order functions and Lambda expressions-高阶函数和Lambda表达式
3. Local functions-本地函数
4. Null safety-空安全
5. Varargs-变长参数
6. Delegated properties-委托属性
7. Ranges
8. RecyclerView.ViewHolder
9. ......

### Topic 1: Companion Object-伴生对象
#### Case 1:  companion object访问OuterClass的成员变量
Kotlin代码：

    class Test {
	    private var hello = 0

	    companion object {
	        fun read() : Int {
	            return Test().hello
	        }
	        fun write() {
	            Test().hello = 1
	        }
	    }
	}

字节码：

    public final class com/mouxuejie/test/Test {
      private I hello
 
      public final static synthetic access$getHello$p(Lcom/mouxuejie/test/Test;)I
	    L0
	      LINENUMBER 3 L0
	      ALOAD 0
	      GETFIELD com/mouxuejie/test/Test.hello : I
	      IRETURN
	    L1
	      LOCALVARIABLE $this Lcom/mouxuejie/test/Test; L0 L1 0
	      MAXSTACK = 1
	      MAXLOCALS = 1

      public final static synthetic access$setHello$p(Lcom/mouxuejie/test/Test;I)V
        L0
          LINENUMBER 3 L0
	      ALOAD 0
	      ILOAD 1
	      PUTFIELD com/mouxuejie/test/Test.hello : I
	      RETURN
	    L1
	      LOCALVARIABLE $this Lcom/mouxuejie/test/Test; L0 L1 0
	      LOCALVARIABLE <set-?> I L0 L1 1
	      MAXSTACK = 2
	      MAXLOCALS = 2
	}

    public final class com/mouxuejie/test/Test$Companion {
      public final read()I
        L0
          LINENUMBER 12 L0
          NEW com/bikcom/mouxuejieest
          DUP
          INVOKESPECIAL com/mouxuejie/test/Test.<init> ()V
          INVOKEVIRTUAL com/mouxuejie/test/Test.getHello$app ()I
          IRETURN

	  public final write()V
	    L0
	      LINENUMBER 8 L0
          NEW com/mouxuejie/test/Test
          DUP
          INVOKESPECIAL com/mouxuejie/test/Test.<init> ()V
          ICONST_1
          INVOKEVIRTUAL com/mouxuejie/test/Test.setHello$app (I)V
    }

结论：    
外部类的`private成员变量`会生成对应的`static getter/setter静态方法`，companion object访问外部类的私有成员变量会调用其static getter/setter方法。    
外部类的`public或internal成员变量`会生成对应的`getter/setter实例方法`。    
（注意：只有companion object读时才会生成getter方法，写时才会生成setter方法，不读不写是不会生成这些方法的）

#### Case 2：OuterClass访问companion object的成员变量
Kotlin代码：

    class Test {
	    companion object {
	        private val HELLO_1 = 0
	        val HELLO_2 = 1
	    }
	
	    fun read1() : Int {
	        return HELLO_1
	    }

	    fun read2() : Int {
	        return HELLO_2
	    }
	}

字节码：

	public final class com/mouxuejie/test/Test {
	  private final static I HELLO_1 = 0
	  private final static I HELLO_2 = 0
	
	  public final read1()I
	   L0
	    LINENUMBER 9 L0
	    GETSTATIC com/mouxuejie/test/Test.HELLO_1 : I
	    IRETURN

	  public final read2()I
	   L0
	    LINENUMBER 9 L0
	    GETSTATIC com/mouxuejie/test/Test.HELLO_2 : I
	    IRETURN
	
	  public final static synthetic access$getHELLO_2$cp()I
	   L0
	    LINENUMBER 3 L0
	    GETSTATIC com/mouxuejie/test/Test.HELLO_2 : I
	    IRETURN
	}

	public final class com/mouxuejie/test/Test$Companion {
	  public final getHELLO_2()I
	   L0
	    LINENUMBER 5 L0
	    INVOKESTATIC com/mouxuejie/test/Test.access$getHELLO_2$cp ()I
	    IRETURN
	}

结论：    
companion object的`private val私有变量`，会在OuterClass生成对应的`private static final静态变量`，不生成额外方法。    
companion object的`public val变量`，会在OuterClass生成对应的`private static final静态变量`和`public static getter方法`，并且会在`Companion类中生成对应的getter实例方法`。其他类访问这个类的变量时，调用顺序是：Companion的getter方法 -> OuterClass的statuc getter方法 -> private static final静态变量    

#### 使用建议
综合以上各种情况，为了减少性能损耗：    
（1）能写成常量类型的尽量使用const val 或@JvmField val    
（2）尽量不要在伴生对象中访问外部类的成员变量    
（3）尽量不要在外部类中访问伴生对象的成员变量    

正确的示范：

	class Test {
	    @JvmField val HELLO_1 = "HELLO_1"
	    companion object {
	        const val HELLO_2 = "HELLO_2"
	    }
	}

注意：    
17年写的文章有可能过时了，下面这个描述目前并没有发现是这么回事    
For other types of constants you can’t, so if you need to access the constant repeatedly, you may want to cache the value in a local variable.

### Topic 2: Higher-order functions and Lambda expressions-高阶函数和Lambda表达式
#### Case 1: function objects函数对象
Kotlin代码：

    class Test {
	    fun transformIntToString(type: Int, body : (Int) -> String) : String {
	        return body(type)
	    }
	
	    fun test () {
	        val result0 = transformIntToString(1) {
	            it.toString()
	        }
	
	        val result1 = transformIntToString(1) {
	            it.toString()
	        }
	    }
	}

字节码：

	public final class com/mouxuejie/test/Test {
	  
	  public final transformIntToString(ILkotlin/jvm/functions/Function1;)Ljava/lang/String;
	   L0
	    ALOAD 2
	    LDC "body"
	    INVOKESTATIC kotlin/jvm/internal/Intrinsics.checkParameterIsNotNull (Ljava/lang/Object;Ljava/lang/String;)V
	   L1
	    LINENUMBER 8 L1
	    ALOAD 2
	    ILOAD 1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    INVOKEINTERFACE kotlin/jvm/functions/Function1.invoke (Ljava/lang/Object;)Ljava/lang/Object;
	    CHECKCAST java/lang/String
	    ARETURN
	
	  public final test()V
	   L0
	    LINENUMBER 16 L0
	    ALOAD 0
	    ICONST_1
	    GETSTATIC com/mouxuejie/test/Test$test$result0$1.INSTANCE : Lcom/mouxuejie/test/Test3$test$result0$1;
	    CHECKCAST kotlin/jvm/functions/Function1
	    INVOKEVIRTUAL com/mouxuejie/test/Test.transformIntToString (ILkotlin/jvm/functions/Function1;)Ljava/lang/String;
	    ASTORE 1
	   L1
	    LINENUMBER 20 L1
	    ALOAD 0
	    ICONST_1
	    GETSTATIC com/mouxuejie/test/Test$test$result1$1.INSTANCE : Lcom/mouxuejie/test/Test3$test$result1$1;
	    CHECKCAST kotlin/jvm/functions/Function1
	    INVOKEVIRTUAL com/mouxuejie/test/Test.transformIntToString (ILkotlin/jvm/functions/Function1;)Ljava/lang/String;
	    ASTORE 2
	}
	
	final class com/mouxuejie/test/Test$test$result0$1 extends kotlin/jvm/internal/Lambda  implements kotlin/jvm/functions/Function1  {

	  public synthetic bridge invoke(Ljava/lang/Object;)Ljava/lang/Object;
	   L0
	    LINENUMBER 5 L0
	    ALOAD 0
	    ALOAD 1
	    CHECKCAST java/lang/Number
	    INVOKEVIRTUAL java/lang/Number.intValue ()I
	    INVOKEVIRTUAL com/mouxuejie/test/Test3$test$result0$1.invoke (I)Ljava/lang/String;
	    ARETURN
	    MAXSTACK = 2
	    MAXLOCALS = 2
	
	  public final invoke(I)Ljava/lang/String;
	   L0
	    LINENUMBER 17 L0
	    ILOAD 1
	    INVOKESTATIC java/lang/String.valueOf (I)Ljava/lang/String;
	   L1
	    ARETURN
	   L2
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test3$test$result0$1; L0 L2 0
	    LOCALVARIABLE it I L0 L2 1
	    MAXSTACK = 1
	    MAXLOCALS = 2

	  public final static Lcom/mouxuejie/test/Test3$test$result0$1; INSTANCE
	}
	
	final class com/mouxuejie/test/Test$test$result1$1 extends kotlin/jvm/internal/Lambda  implements kotlin/jvm/functions/Function1  {
		// 和上面Function类似
		...
	}

结论：    
上面的例子`transformIntToString`是一个高阶函数，该函数的一个参数`body : (Int) -> String`是lambda表达式。    
每个调用高阶函数的地方，lambda表达式都会分别产生一个继承自`kotlin/jvm/functions/Function1的函数对象`。    
（注意：如果不希望每次调用lambda表达式的地方都创建对象，可以将lambda表达式赋值给一个变量，然后每次引用该变量）    

#### Case 2: Boxing overhead

Kotlin代码：

    fun transformIntToString(type: Int, body : (Int) -> String) : String {
        return body(type)
    }

字节码：

    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
    INVOKEINTERFACE kotlin/jvm/functions/Function1.invoke (Ljava/lang/Object;)Ljava/lang/Object;
    CHECKCAST java/lang/String

从上面的字节码可以看出，`lambda表达式调用存在装箱和拆箱开销`。    
基本调用过程：入参Int -> Integer，Function1.invoke(Object) : Object，出参Object -> String    

#### Case 3: inline functions
Kotlin代码：

	class Test {

	    inline fun transformIntToString(type: Int, body : (Int) -> String) : String {
	        return body(type)
	    }
	
	    val result = transformIntToString(1) {
	        it.toString()
	    }
	
	    fun test () {
	        val result0 = transformIntToString(1) {
	            it.toString()
	        }
	
	        val result1 = transformIntToString(1) {
	            it.toString()
	        }
	    }
	}

字节码：

	public final test()V
	  L4
	    LINENUMBER 17 L4
	    ILOAD 5
	    INVOKESTATIC java/lang/String.valueOf (I)Ljava/lang/String;
	  L12
	    LINENUMBER 21 L12
	    ILOAD 6
	    INVOKESTATIC java/lang/String.valueOf (I)Ljava/lang/String;

结论：    
如果高阶函数为`inline内联函数`，则lambda表达式不会生成kotlin/jvm/functions/Function1函数对象，会`直接执行lambda函数体`。    

当然内联函数也有一些限制，比如：    
（1）内联函数不能递归调用自己    
（2）内联函数只能访问所在类的public函数或属性    
（3）内联函数体代码应该精简一些    

关于内联函数介绍：    
[https://kotlinlang.org/docs/reference/inline-functions.html](https://kotlinlang.org/docs/reference/inline-functions.html)

#### 使用建议
（1）避免在每个地方都直接使用lambda表达式，可以将lambda表达式赋值给一个变量，然后每次引用该变量，这样既可以避免重复创建函数对象，也可以避免重复装箱拆箱开销。    
（2）inline内联函数可以避免高阶函数创建函数对象及装箱拆箱开销，但是要注意inline函数体不宜过大    

###  Topic 3: Local functions-本地函数
Kotlin代码：

	fun someMath(a: Int): Int {
	    fun sumSquare(b: Int) = (a + b) * (a + b)
	
	    return sumSquare(1) + sumSquare(2)
	}

字节码：

	  public final someMath(I)I
	   L0
	    LINENUMBER 5 L0
	    NEW com/mouxuejie/test/Test5$someMath$1
	    DUP
	    ILOAD 1
	    INVOKESPECIAL com/mouxuejie/test/Test5$someMath$1.<init> (I)V
	    ASTORE 2
	   L1
	    LINENUMBER 7 L1
	    ALOAD 2
	    ICONST_1
	    INVOKEVIRTUAL com/mouxuejie/test/Test5$someMath$1.invoke (I)I
	    ALOAD 2
	    ICONST_2
	    INVOKEVIRTUAL com/mouxuejie/test/Test5$someMath$1.invoke (I)I
	    IADD
	    IRETURN

	final class com/mouxuejie/test/Test5$someMath$1 extends kotlin/jvm/internal/Lambda implements kotlin/jvm/functions/Function1  {
	
	  public synthetic bridge invoke(Ljava/lang/Object;)Ljava/lang/Object;
	   L0
	    LINENUMBER 3 L0
	    ALOAD 0
	    ALOAD 1
	    CHECKCAST java/lang/Number
	    INVOKEVIRTUAL java/lang/Number.intValue ()I
	    INVOKEVIRTUAL com/mouxuejie/test/Test5$someMath$1.invoke (I)I
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    ARETURN
	    MAXSTACK = 2
	    MAXLOCALS = 2
	
	  public final invoke(I)I
	   L0
	    LINENUMBER 5 L0
	    ALOAD 0
	    GETFIELD com/mouxuejie/test/Test5$someMath$1.$a : I
	    ILOAD 1
	    IADD
	    ALOAD 0
	    GETFIELD com/mouxuejie/test/Test5$someMath$1.$a : I
	    ILOAD 1
	    IADD
	    IMUL
	    IRETURN
	   L1
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test5$someMath$1; L0 L1 0
	    LOCALVARIABLE b I L0 L1 1
	    MAXSTACK = 3
	    MAXLOCALS = 2
	}

稍微改变一下：

	fun someMath(a: Int): Int {
	    fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)
	
	    return sumSquare(a, 1) + sumSquare(a, 2)
	}

字节码：

	  public final someMath(I)I
	   L0
	    LINENUMBER 5 L0
	    GETSTATIC com/mouxuejie/test/Test6$someMath$1.INSTANCE : Lcom/mouxuejie/test/Test6$someMath$1;
	    ASTORE 2
	   L1
	    LINENUMBER 7 L1
	    ALOAD 2
	    ILOAD 1
	    ICONST_1
	    INVOKEVIRTUAL com/mouxuejie/test/Test6$someMath$1.invoke (II)I
	    ALOAD 2
	    ILOAD 1
	    ICONST_2
	    INVOKEVIRTUAL com/mouxuejie/test/Test6$someMath$1.invoke (II)I
	    IADD
	    IRETURN
	   L2
	    LOCALVARIABLE sumSquare$ Lcom/mouxuejie/test/Test6$someMath$1; L1 L2 2
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test6; L0 L2 0
	    LOCALVARIABLE a I L0 L2 1
	    MAXSTACK = 4
	    MAXLOCALS = 3

#### 使用建议
（1）如果不是必需，减少使用本地方法，因为本地方法会生成Function1函数对象，且使用不当会有频繁装箱拆箱开销。    
（2）如果一定要使用本地方法，建议遵循case1优于case2：    

	// case 1 每次调用外层方法，本地方法都会new一个function object且不会复用，同时会造成反复装箱拆箱
	fun someMath(a: Int): Int {
	   fun sumSquare(a: Int, b: Int) = (a + b) * (a + b)
	    return sumSquare(a, 1) + sumSquare(a, 2)
	}
	
	// case 2 每次调用外层方法，都会复用function object，减少装箱拆箱操作。
	fun someMath(a: Int): Int {
	    fun sumSquare(b: Int) = (a + b) * (a + b)
	    return sumSquare(1) + sumSquare(2)
	}

### Topic 4: Null safety-空安全

#### Case 1: Non-null arguments runtime checks
Kotlin代码：

	class Test {
	    fun sayHello(who: String) {
	        println("Hello $who")
	    }
	
	    fun test() {
	        sayHello(null) // 编译不通过，会报Null can not be a value of a non-null type String
	    }
	}

字节码：

	  public final sayHello(Ljava/lang/String;)V
	    @Lorg/jetbrains/annotations/NotNull;() // invisible, parameter 0
	   L0
	    ALOAD 1
	    LDC "who"
	    INVOKESTATIC kotlin/jvm/internal/Intrinsics.checkParameterIsNotNull (Ljava/lang/Object;Ljava/lang/String;)V
	   L1
	    LINENUMBER 5 L1
	    NEW java/lang/StringBuilder
	    DUP
	    INVOKESPECIAL java/lang/StringBuilder.<init> ()V
	    LDC "Hello "
	    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
	    ALOAD 1
	    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
	    INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
	    ASTORE 2
	   L2
	    ICONST_0
	    ISTORE 3
	   L3
	    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
	    ALOAD 2
	    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/Object;)V
	   L4
	   L5
	    LINENUMBER 6 L5
	    RETURN
	   L6
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test7; L0 L6 0
	    LOCALVARIABLE who Ljava/lang/String; L0 L6 1
	    MAXSTACK = 2
	    MAXLOCALS = 4

反编译代码：

	   public final void sayHello(@NotNull String who) {
	      Intrinsics.checkParameterIsNotNull(who, "who");
	      String var2 = "Hello " + who;
	      System.out.println(var2);
	   }

结论：    
对于`public方法`，如果入参为`引用参数且非空`（如String/Object等，即非基本类型），Kotlin编译器会为该入参生成`@NotNull注解`，同时会`增加判空校验代码`Intrinsics.checkParameterIsNotNull(who, "who");，在非法调用时编译不通过。    
当然这样的性能开销几乎可以忽略不计。    
对于`private方法`，编译器不会为入参生成@NotNull注解和判空校验代码，会默认私有方法为空安全的。    

#### Case 2: Nullable primitive types
Kotlin代码：

	class Test {
	    fun add0(a: Int, b: Int): Int {
	        return a + b
	    }
	    fun add1(a: Int?, b: Int?): Int {
	        return (a ?: 0) + (b ?: 0)
	    }
	
	    fun test() {
	        add0(1, 1)
	        add1(1, null)
	    }
	}

字节码：

	  public final add0(II)I
	   L0
	    LINENUMBER 5 L0
	    ILOAD 1
	    ILOAD 2
	    IADD
	    IRETURN
	   L1
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test; L0 L1 0
	    LOCALVARIABLE a I L0 L1 1
	    LOCALVARIABLE b I L0 L1 2
	    MAXSTACK = 2
	    MAXLOCALS = 3
	
	  public final add1(Ljava/lang/Integer;Ljava/lang/Integer;)I
	   L0
	    LINENUMBER 8 L0
	    ALOAD 1
	    DUP
	    IFNULL L1
	    INVOKEVIRTUAL java/lang/Integer.intValue ()I
	    GOTO L2
	   L1
	    POP
	    ICONST_0
	   L2
	    ALOAD 2
	    DUP
	    IFNULL L3
	    INVOKEVIRTUAL java/lang/Integer.intValue ()I
	    GOTO L4
	   L3
	    POP
	    ICONST_0
	   L4
	    IADD
	    IRETURN
	   L5
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test; L0 L5 0
	    LOCALVARIABLE a Ljava/lang/Integer; L0 L5 1
	    LOCALVARIABLE b Ljava/lang/Integer; L0 L5 2
	    MAXSTACK = 3
	    MAXLOCALS = 3
	
	  public final test()V
	   L0
	    LINENUMBER 12 L0
	    ALOAD 0
	    ICONST_1
	    ICONST_1
	    INVOKEVIRTUAL com/mouxuejie/test/Test.add0 (II)I
	    POP
	   L1
	    LINENUMBER 13 L1
	    ALOAD 0
	    ICONST_1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    ACONST_NULL
	    INVOKEVIRTUAL com/mouxuejie/test/Test.add1 (Ljava/lang/Integer;Ljava/lang/Integer;)I
	    POP
	   L2
	    LINENUMBER 14 L2
	    RETURN
	   L3
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test; L0 L3 0
	    MAXSTACK = 3
	    MAXLOCALS = 1
	
	  public <init>()V
	   L0
	    LINENUMBER 3 L0
	    ALOAD 0
	    INVOKESPECIAL java/lang/Object.<init> ()V
	    RETURN
	   L1
	    LOCALVARIABLE this Lcom/mouxuejie/test/Test; L0 L1 0
	    MAXSTACK = 1
	    MAXLOCALS = 1

结论：    
如果方法的入参为`primitive type基本数据类型且可空`，当外部调用该方法时，会先将基本数据类型装箱成Object，因此存在额外`装箱拆箱`的性能开销。因此建议尽量使用非空的基本数据类型。    

#### Case 3: About arrays
Kotlin中数组的几种方式：
1. IntArray / FloatArray / DoubleArray
2. Array < T >，如Array < Int >
3. Array < T? >，如Array < Int? >

Kotlin代码：

    fun test() {
        // case 1 IntArray
        val result0 = IntArray(2)
        result0[0] = 1
        result0[1] = 1

		// case 2 Array<Int>
        val result1 = Array(2) {
            0
        }
        result1[0] = 1
        result1[1] = 1

		// case 3 Array<Int?>
        val result2 = Array<Int?>(2) {
            null
        }
        result2[0] = 1
        result2[1] = 1
    }

字节码：
	
	   // IntArray
	   L0
	    LINENUMBER 6 L0
	    ICONST_2
	    NEWARRAY T_INT
	    ASTORE 1
	   L1
	    LINENUMBER 7 L1
	    ALOAD 1
	    ICONST_0
	    ICONST_1
	    IASTORE
	   L2
	    LINENUMBER 8 L2
	    ALOAD 1
	    ICONST_1
	    ICONST_1
	    IASTORE
	
	   // Array<Int>
	   L3
	    LINENUMBER 10 L3
	    ICONST_2
	    ISTORE 3
	   L4
	    LINENUMBER 23 L4
	    ILOAD 3
	    ANEWARRAY java/lang/Integer
	    ASTORE 4
	   L9
	    ICONST_0
	    ISTORE 8
	   L10
	    LINENUMBER 11 L10
	    ICONST_0
	   L11
	   L12
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    ASTORE 13
	    ALOAD 11
	    ILOAD 12
	    ALOAD 13
	    AASTORE
	   L16
	    LINENUMBER 13 L16
	    ALOAD 2
	    ICONST_0
	    ICONST_1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    AASTORE
	   L17
	    LINENUMBER 14 L17
	    ALOAD 2
	    ICONST_1
	    ICONST_1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    AASTORE
	
       // Array<Int?>
	   L18
	    LINENUMBER 16 L18
	    ICONST_2
	    ISTORE 4
	   L19
	    LINENUMBER 28 L19
	    ILOAD 4
	    ANEWARRAY java/lang/Integer
	    ASTORE 5
	   L31
	    LINENUMBER 19 L31
	    ALOAD 3
	    ICONST_0
	    ICONST_1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    AASTORE
	   L32
	    LINENUMBER 20 L32
	    ALOAD 3
	    ICONST_1
	    ICONST_1
	    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
	    AASTORE

结论：    
IntArray编译后相当于int[]，不存在装箱拆箱开销；    
Array < Int >和Array < Int? >编译后相当于Array < Integer >，存在装箱拆箱的开销，只不过一个可空一个不可空    

#### 使用建议
（1）如果方法的入参为primitive type基本数据类型，则建议使用非空类型，避免`装箱拆箱`的性能开销    
（2）如果方法的入参为reference type引用数据类型且非空且为public方法，编译器会在方法开头生成空校验代码来保证空安全    
（3）IntArray优于Array< Int >和Array< Int? >，没有装箱拆箱开销    

### Topic 5: Varargs-变长参数
[https://kotlinlang.org/docs/reference/functions.html#variable-number-of-arguments-varargs](https://kotlinlang.org/docs/reference/functions.html#variable-number-of-arguments-varargs)

Kotlin代码：

    fun printDouble(vararg values: Int) {
        values.forEach { println(it * 2) }
    }

    fun test() {
        // case 1
        printDouble(1, 2, 3)

		// case 2
        val values = intArrayOf(1, 2, 3)
        printDouble(*values)
		
		// case 3
        printDouble(0, *values, 4)
    }

字节码：

	  public final test()V
	   // case 1
	   L0
	    LINENUMBER 9 L0
	    ALOAD 0
	    ICONST_3
	    NEWARRAY T_INT
	    DUP
	    ICONST_0
	    ICONST_1
	    IASTORE
	    DUP
	    ICONST_1
	    ICONST_2
	    IASTORE
	    DUP
	    ICONST_2
	    ICONST_3
	    IASTORE
	    INVOKEVIRTUAL com/mouxuejie/test/Test11.printDouble ([I)V
	   
	   // case 2
	   L1
	    LINENUMBER 11 L1
	    ICONST_3
	    NEWARRAY T_INT
	    DUP
	    ICONST_0
	    ICONST_1
	    IASTORE
	    DUP
	    ICONST_1
	    ICONST_2
	    IASTORE
	    DUP
	    ICONST_2
	    ICONST_3
	    IASTORE
	    ASTORE 1
	   L2
	    LINENUMBER 12 L2
	    ALOAD 0
	    ALOAD 1
	    DUP
	    ARRAYLENGTH
	    INVOKESTATIC java/util/Arrays.copyOf ([II)[I
	    INVOKEVIRTUAL com/mouxuejie/test/Test11.printDouble ([I)V

	   // case 3
	   L3
	    LINENUMBER 14 L3
	    ALOAD 0
	    NEW kotlin/jvm/internal/IntSpreadBuilder
	    DUP
	    ICONST_3
	    INVOKESPECIAL kotlin/jvm/internal/IntSpreadBuilder.<init> (I)V
	    DUP
	    ICONST_0
	    INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.add (I)V
	    DUP
	    ALOAD 1
	    INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.addSpread (Ljava/lang/Object;)V
	    DUP
	    ICONST_4
	    INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.add (I)V
	    INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.toArray ()[I
	    INVOKEVIRTUAL com/mouxuejie/test/Test11.printDouble ([I)V


结论：    
case 1创建数组new int[] {1, 2, 3}
		
	printDouble(new int[]{1, 2, 3});

case 2创建数组new int[] {1, 2, 3}，且调用Arrays.copyOf()拷贝了一份数据，保证原始数组数据不被修改

	int[] values = new int[]{1, 2, 3};
	printDouble(Arrays.copyOf(values, values.length));

case 3等价于
	
	int[] values = new int[]{1, 2, 3};
	IntSpreadBuilder var10000 = new IntSpreadBuilder(3);
	var10000.add(0);
	var10000.addSpread(values);
	var10000.add(42);
	printDouble(var10000.toArray());

#### 使用建议
由于变长参数会`创建新数组`，额外占用内存，且可能存在`数据拷贝操作`，因此建议方法入参尽量使用定长数组代替变长参数。

### Topic 6: Delegated properties-委托属性
委托属性可以参考：    
[https://kotlinlang.org/docs/reference/delegated-properties.html](https://kotlinlang.org/docs/reference/delegated-properties.html)

lazy使用示例：

	private val dateFormat: DateFormat by lazy {
	    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
	}

lazy的懒加载特性，保证了只会在首次访问时才会初始化lambda中的代码

lazy的3种模式：    
- LazyThreadSafetyMode.SYNCHRONIZED
- LazyThreadSafetyMode.PUBLICATION 
- LazyThreadSafetyMode.NONE

LazyThreadSafetyMode.SYNCHRONIZED是默认模式，多线程时可以保证线程安全，但存在double-check lock开销；    
如果已知在单线程工作环境，则建议使用LazyThreadSafetyMode.NONE模式，减少同步锁开销。

### Topic 7: Ranges
#### Case 1: Inclusion tests
Kotlin代码：

    fun test0(i: Int) {
        if (i in 1..10) {
            println(i)
        }
    }

    fun test1(name: String) {
        if (name in "Alfred".."Alicia") {
            println(name)
        }
    }

反编译代码：

	public final void test0(int i) {
	    if (1 <= i && 10 >= i) {
            System.out.println(i);
	    }
	}
	
	public final void test1(@NotNull String name) {
        Intrinsics.checkParameterIsNotNull(name, "name");
        Comparable var10000 = (Comparable)"Alicia";
        Comparable var10001 = (Comparable)"Alfred";
        Comparable var2 = (Comparable)name;
        if (var2.compareTo(var10001) >= 0) {
           if (var2.compareTo(var10000) <= 0) {
              System.out.println(name);
           }
        }
	 }

Kotlin代码：

    private val myRange get() = 1..10
    fun test(i: Int) {
        if (i in myRange) {
            println(i)
        }
    }

字节码：

	 private final getMyRange()Lkotlin/ranges/IntRange;
	   L0
	    LINENUMBER 17 L0
	    ICONST_1
	    ISTORE 1
	    NEW kotlin/ranges/IntRange
	    DUP
	    ILOAD 1
	    BIPUSH 10
	    INVOKESPECIAL kotlin/ranges/IntRange.<init> (II)V
	    ARETURN
	
	  public final test(I)V
	   L0
	    LINENUMBER 19 L0
	    ALOAD 0
	    INVOKESPECIAL com/mouxuejie/test/Test12.getMyRange ()Lkotlin/ranges/IntRange;
	    ILOAD 1
	    INVOKEVIRTUAL kotlin/ranges/IntRange.contains (I)Z
	    IFEQ L1
	   L2
	    LINENUMBER 20 L2
	   L3
	    ICONST_0
	    ISTORE 2
	   L4
	    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
	    ILOAD 1
	    INVOKEVIRTUAL java/io/PrintStream.println (I)V
    
反编译代码：

    private final IntRange getMyRange() {
	    return new IntRange(1, 10);
	}
	
	public final void test(int i) {
	    if (this.getMyRange().contains(i)) {
	       System.out.println(i);
	    }
    }

结论：    
`直接引用1..n`相当于 i >= 1&&i <=n，不会创建额外的对象；通过`变量间接引用1..n`，则每次调用的时候都会创建IntRange对象。

#### Case 2: Iterations for loops
Kotlin代码：

    fun test0() {
        for (i in 1..10) {
            println(i)
        }
    }

    fun test1() {
        for (i in 1 until 10) {
            println(i)
        }
    }

    fun test2() {
        for (i in 10 downTo 1) {
            println(i)
        }
    }

    fun test3() {
        for (i in (1..10).reversed()) {
            println(i)
        }
    }

    fun test4() {
        for (i in 1..10 step 2) {
            println(i)
        }
    }

反编译代码：

    public final void test0() {
	    int i = 1;
	    for(byte var2 = 10; i <= var2; ++i) {
	       System.out.println(i);
	    }
	}

	public final void test1() {
	    int i = 1;
	    for(byte var2 = 10; i < var2; ++i) {
	        System.out.println(i);
	    }
	}
	
	public final void test2() {
	    int i = 10;
	    for(byte var2 = 1; i >= var2; --i) {
	       System.out.println(i);
	    }
	}
	
	public final void test3() {
	    int i = 10;
	    for(byte var2 = 1; i >= var2; --i) {
	       System.out.println(i);
	    }
	}
	
	public final void test4() {
	    byte var4 = 1;
	    IntProgression var10000 = RangesKt.step((IntProgression)(new IntRange(var4, 10)), 2);
	    int i = var10000.getFirst();
	    int var2 = var10000.getLast();
	    int var3 = var10000.getStep();
	    if (var3 > 0) {
	       if (i > var2) {
	          return;
	       }
	    } else if (i < var2) {
	       return;
	    }
	
	    while(true) {
	       System.out.println(i);
	       if (i == var2) {
	          return;
	       }
	
	       i += var3;
	    }
	}

结论：    
`.. / downTo / until`相当于[1, 10]/[10, 1]/[1, 10)，不会创建额外对象；    
但是`step步进操作`，会创建IntRange和IntProgression对象，并计算出first和last，额外占用内存。    

#### Case 3: Iterations forEach()
Kotlin代码：

    fun test() {
        (1..10).forEach {
            println(it)
        }
    }

反编译代码：

	public final void test() {
	    Iterable $this$forEach$iv = (Iterable)(new IntRange(1, 10));
	    Iterator var3 = $this$forEach$iv.iterator();
	
	    while(var3.hasNext()) {
	       int element$iv = ((IntIterator)var3).nextInt();
	       System.out.println(element$iv);
	    }
	}

#### Case 4: Iterations collection indices
Kotlin代码：

    fun test() {
        val list = listOf("A", "B", "C")
        for (i in list.indices) {
            println(list[i])
        }
    }

反编译代码：

	   public final void test() {
	      List list = CollectionsKt.listOf(new String[]{"A", "B", "C"});
	      int i = 0;
	
	      for(int var3 = ((Collection)list).size(); i < var3; ++i) {
	         Object var4 = list.get(i);
	         System.out.println(var4);
	      }
	   }

Kotlin代码：

    inline val SparseArray<*>.indices: IntRange
        get() = 0 until size()

    fun test7(map: SparseArray<String>) {
        for (i in map.indices) {
            println(map.valueAt(i))
        }
    }

    fun test8(map: SparseArray<String>) {
        for (i in 0 until map.size()) {
            println(map.valueAt(i))
        }
    }

反编译代码：

	   @NotNull
	   public final IntRange getIndices(@NotNull SparseArray $receiver) {
	      Intrinsics.checkParameterIsNotNull($receiver, "receiver$0");
	      return RangesKt.until(0, $receiver.size());
	   }
	
	   public final void test7(@NotNull SparseArray map) {
	      Intrinsics.checkParameterIsNotNull(map, "map");
	      IntRange var10000 = RangesKt.until(0, map.size());
	      int i = var10000.getFirst();
	      int var3 = var10000.getLast();
	      if (i <= var3) {
	         while(true) {
	            Object var4 = map.valueAt(i);
	            System.out.println(var4);
	            if (i == var3) {
	               break;
	            }
	
	            ++i;
	         }
	      }
	   }
	
	   public final void test8(@NotNull SparseArray map) {
	      Intrinsics.checkParameterIsNotNull(map, "map");
	      int i = 0;
	
		  for(int var3 = map.size(); i < var3; ++i) {
		      Object var4 = map.valueAt(i);
		      System.out.println(var4);
		   }
	   }

结论：    
对于继承Collection的集合类，until效率优于indices    

#### 使用建议
（1）`直接引用1..n`优于`变量间接引用1..n`    
（2）尽量避免使用step步进操作    
（3）.. / downTo / until的for循环优于forEach    
（4）集合Collection的until优于indices    

#### Topic 8: RecyclerView.ViewHolder

Kotlin代码：

    class RecyclerViewAdapter : RecyclerView.Adapter<ViewHolder>() {	
	    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
            holder.itemView.name_tv.text = "111"
            holder.itemView.distance_tv.text = "222"
       }
    }

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView)

反编译代码：

    public static final class RecyclerViewAdapter extends Adapter {
        public void onBindViewHolder(@NotNull Test14.ViewHolder holder, int position) {
            Intrinsics.checkParameterIsNotNull(holder, "holder");
            Intrinsics.checkExpressionValueIsNotNull(holder.itemView, "holder.itemView");
            TextView var3 = (TextView)var10000.findViewById(id.name_tv);
            Intrinsics.checkExpressionValueIsNotNull(var3, "holder.itemView.name_tv");
            var3.setText((CharSequence)"111");
            Intrinsics.checkExpressionValueIsNotNull(holder.itemView, "holder.itemView");
            var3 = (TextView)var10000.findViewById(id.distance_tv);
            Intrinsics.checkExpressionValueIsNotNull(var3, "holder.itemView.distance_tv");
            var3.setText((CharSequence)"222");
        }
     }

     public static final class ViewHolder extends androidx.recyclerview.widget.RecyclerView.ViewHolder {
          public ViewHolder(@NotNull View itemView) {
              Intrinsics.checkParameterIsNotNull(itemView, "itemView");
              super(itemView);
          }
      }

Kotlin代码：

    class RecyclerViewAdapter : RecyclerView.Adapter<ViewHolder>() {	
	    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
            holder.nameTv.text = "111"
            holder.distanceTv.text = "222"
       }
    }

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val nameTv: TextView = itemView.name_tv
        val distanceTv: TextView = itemView.distance_tv
    }

反编译代码：

	public static final class RecyclerViewAdapter extends Adapter {
        public void onBindViewHolder(@NotNull Test14.ViewHolder holder, int position) {
            Intrinsics.checkParameterIsNotNull(holder, "holder");
            holder.getNameTv().setText((CharSequence)"111");
            holder.getDistanceTv().setText((CharSequence)"222");
        }
     }

     public static final class ViewHolder extends androidx.recyclerview.widget.RecyclerView.ViewHolder {
          @NotNull
	      private final TextView nameTv;
	      @NotNull
	      private final TextView distanceTv;
	
	      @NotNull
	      public final TextView getNameTv() {
	         return this.nameTv;
	      }
	
	      @NotNull
	      public final TextView getDistanceTv() {
	         return this.distanceTv;
	      }
	
	      public ViewHolder(@NotNull View itemView) {
	         Intrinsics.checkParameterIsNotNull(itemView, "itemView");
	         super(itemView);
	         TextView var10001 = (TextView)itemView.findViewById(id.name_tv);
	         Intrinsics.checkExpressionValueIsNotNull(var10001, "itemView.name_tv");
	         this.nameTv = var10001;
	         var10001 = (TextView)itemView.findViewById(id.distance_tv);
	         Intrinsics.checkExpressionValueIsNotNull(var10001, "itemView.distance_tv");
	         this.distanceTv = var10001;
	      }
      }

结论：    
Kotlin使用RecyclerView时，每次通过id引用控件，相当于执行了一遍findViewById，因此为了减少性能开销，应该通过ViewHolder缓存控件，基本原则和Java一致。但是在Activity/Fragment中引用id时不会频繁执行findViewById，本身有缓存机制。

### 参考文档

[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62)    
[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-2-324a4a50b70](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-2-324a4a50b70)    
[https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4)    
[https://proandroiddev.com/the-costs-of-kotlin-android-extensions-6809e2b32b13](https://proandroiddev.com/the-costs-of-kotlin-android-extensions-6809e2b32b13)    



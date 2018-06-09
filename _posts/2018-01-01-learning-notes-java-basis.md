---
layout: post
comments: true
title: "Java学习笔记——查漏补缺"
description: "Java学习笔记——查漏补缺"
category: Java
tags: [Java]
---

**1、hashCode()/equal()**    
**2、深拷贝／浅拷贝**    
**（1）基本概念**    
**（2）clone()**    
**（3）序列化**    
**3、拆箱／装箱**    
**4、异常**    
**（1）Error／Exception**    
**（2）try...catch...finally**    
**（3）static{...}异常**    
**5、泛型**    
**6、参考文档**    

<!--more-->

### 1、hashCode()/equal()    

hashCode()/equal()实现

Object源码

### 2、深拷贝／浅拷贝    

#### （1）基本概念    

`深拷贝`：对基本数据类型进行值传递，对引用数据类型创建一个新对象，并复制其内容。    

![](/image/2018-01-01-learning-notes-java-basis/deep-copy.jpg)   

`浅拷贝`：对基本数据类型进行值传递，对引用数据类型进行引用传递。    

![](/image/2018-01-01-learning-notes-java-basis/shallow-copy.jpg)   

#### （2）clone()    

clone()是Object的方法。     
调用`clone()`方法的对象，需要implements Cloneable，整体来说`clone()是浅拷贝`。    
如果对象属性都是`基本数据类型`，直接调用对象的默认clone()则浅拷贝和深拷贝的效果一样。    
如果对象属性包含`引用数据类型`，直接调用对象的clone()是浅拷贝，需要在clone()方法里针对引用数据类型的属性再进行深拷贝。    

简单理解就是：类的每个属性都进行深拷贝，类才算深拷贝，否则就是浅拷贝。clone()是浅拷贝，但是如果属性只有基本数据类型，则浅拷贝和深拷贝效果一样；如果有对象属性，则需要对对象属性进行深拷贝，整个类才算深拷贝。
    
    //深拷贝
    public class SuperClass implements Cloneable {
        public String name;
        public int age;
        public SubClass sub;
        
        @Override
        public Object clone() {
            try {
                SuperClass cloneSuper = (A)super.clone();
                // 这一行去掉就成了浅拷贝
                cloneSuper.sub = (SubClass)sub.clone();
                return cloneSuper;
            } catch (Exception e) {
                e.printStackTrace();
            }
            return null;
        }
    }
    
    public class SubClass implements Cloneable {
        public String name;
        public int age;
        
        @Override
        public Object clone() {
            try {
                return super.clone();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return null; 
        }
    }

#### （3）序列化    
 
Serializable/Parcelable序列化再反序列化得到的对象是深拷贝对象。
    
[细说 Java 的深拷贝和浅拷贝](https://segmentfault.com/a/1190000010648514)    

### 3、拆箱／装箱    

100L == new Long(100L) ?

Long l1 = new Long(100);
Long l2 = new Long(100);
l1 == l2 ?

### 4、异常    

#### （1）Error／Exception    

![](/image/2018-01-01-learning-notes-java-basis/Error-Exception.svg)   

`Error／Exception`都是继承Throwable，`都是可以try...catch住的`。    

`Error`，用于标记严重错误，一般指与`虚拟机相关`的问题，如系统崩溃、虚拟机错误、内存空间不足、方法调用栈溢出等。对于这类错误导致的应用程序中断，仅靠程序本身无法恢复和预防，`一般不去try...catch这种错误`，建议让程序终止。    

`Exception`，可处理的异常，可以捕获且可能恢复，`应尽可能try...catch住`，使程序恢复运行。

**关于checked和unchecked：**    
`unchecked`异常：包括Error和RuntimeException。编译器不会去检查它，如果没有try...catch捕获，也没有throws抛出，还是会编译通过。

`checked`异常：除RuntimeException类型外的Exception。这类异常如果没有try……catch也没有throws抛出，编译是通不过的。

**关于throw和throws：**    
`throw`，出现在方法体，一般用于程序出现某种逻辑错误时主动抛出异常。    
`throws`，出现在方法头，表示该成员函数`可能`抛出的各种异常。    
throw/throws的异常，函数自身不处理，均由上层函数处理。    

[谈一谈Java中的Error和Exception](https://blog.csdn.net/goodlixueyong/article/details/47122487)    

#### （2）try...catch...finally...return    

原则：    
（1）不管有没有异常，finally都会被执行。    
（2）finally中如果有return，则执行到finally语句后会直接从finally的return跳出来；    
（3）finally中如果无return，则看进入finally之前的是try还是catch语句块，如果是try语句块的话，执行完finally语句后会从try语句块的return返回，反之从catch语句块的return返回。    
（4）finally是在try/catch语句块return后面的表达式运算后执行的，所以函数返回值是在finally执行前确定的。无论finally中的代码怎么样，返回的值都不会改变，仍然是之前return语句中保存的值（`对象除外`）。    

[Java中try catch finally语句中含有return语句的执行情况（总结版）](https://blog.csdn.net/ns_code/article/details/17485221)    
[java 异常捕捉 ( try catch finally ) 你真的掌握了吗？](http://www.blogjava.net/fancydeepin/archive/2012/07/08/java_try-catch-finally.html)    

#### （3）static{...}异常    

    public class A {
        
        static {
            if (true) {  
                throw new RuntimeException("static test");  
            } 
        }
    
        public void func() {  
            ...
        }
    }
    
    public class Test{
    
        pubic static void main(int args[]) {
            try {
                new A().func();
            } catch(Exception e) {
                e.printStackTrace();
            }

            try {
                new A().func();
            } catch(Exception e) {
                e.printStackTrace();
            }
        }
    }
    
第一次new A()时，会启动class A的类加载过程，到<clinit>类初始化的时候会执行static代码块，这时候会抛出RuntimeException，类初始化过程失败（erroneous state），这个异常被调用的地方catch住。    

第二次new A()时，由于class A的状态为失败（erroneous state），说明已经初始化过且类初始化失败，由于类加载过程只能加载一次，此时会直接抛出NoClassDefFoundError并在调用的地方catch住。    

[Java 静态块抛异常之后](http://shihlei.iteye.com/blog/2358390)    
[java类static初始化代码块中抛出未预期的异常，导致该类无法被正常加载](https://blog.csdn.net/mhxy199288/article/details/79422246)    

### 5、泛型    

### 6、参考文档    

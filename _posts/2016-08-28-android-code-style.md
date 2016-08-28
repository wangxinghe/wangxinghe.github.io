---
layout: post
comments: true
title: "Android编码规范"
description: "Android编码规范"
category: android
tags: [Android]
---


写在前面的话：

本文是我自己整理的一份Android编码规范，已经在团队内部分享，放到我用来知识积累的博客上。

## 0x00 命名规范
**基本原则**：遵循驼峰命名规则，名字能准确描述表达的含义，好的命名可以省去代码注释。

<!--more-->

### 1 常量命名

所有单词大写，单词间以"_"分隔

### 2 变量命名

驼峰命名。成员变量以`m`开头；静态变量以`s`开头

### 3 方法命名

驼峰命名。

### 4 接口

首字母大写，驼峰命名，使用名词。带`I`前缀，或`able`,`ible`,`er`等后缀，如`IManager`,`OnClickListener`

### 5 类

首字母大写，驼峰命名，使用名词。

### 6 包

所有单词小写，只能包含`a-z`字母，或有含义的阿拉伯数字如`4`代替`for`,`2`代替`to`

### 7 资源文件

（1）布局文件

| Type  | Format  |
| ----- |:-------|  
| Activity | activity_ |
| Fragment | fragment_ |
| Dialog   | dialog_   |
| PopupWindow | popup_ |
| Menu | menu_ |
| Adapter | layout_item_ |

（2）图片

| Suffix  | Meaning  |
| ----- |:-------|
| bg_xxx | 背景图片 |
| btn_xxx | 按钮 |
| ic_xxx | 单个图标 |
| bg _ 描述 _ 状态 | 控件上的不同状态 |
| btn _ 描述 _ 状态 | 按钮上的不同状态 |
| chx _ 描述 _ 状态 | 选择框，一般2态或4态 |

## 0x01 编码顺序

**基本原则**：

- 一个类的代码结构从上到下依次为，`package`，`import`，`常量／变量块`，`方法块`。
- 总体上来说，要按照先 `public`, 后 `protected`, 最后 `private`, 方法的排布也应该有一个逻辑的先后顺序，由重到轻。

### 1 import

类／包导入顺序为：`Android`，`Third Party`，`Java/Javax`。且这三组之间空一行，组内按a-z排列顺序

### 2 常量/变量

顺序为：（1）常量、静态变量、成员变量（2）`public`,`protected`,`private`

### 3 方法

对于普通类，方法顺序为：先`public`再`protected`后`private`    

对于组件类，方法顺序为：`构造方法`、`生命周期方法`、`public方法`、`protected方法`、`private方法`    

## 0x02 通用规则

### 1 基础

- 不要直接跨业务模块调用方法，一个模块提供一个对外类
- **禁止将整个类格式化**
- 每个类长度不超过1000行
- 一行最多只能写一条语句，不允许一行定义多个变量或执行多条语句
- 一个方法只做一件事情，方法体不能太长，且不能传入太多参数，一般5个以内，暂定不超过8个
- 每行代码不超过100个字符，超过的需要使用缩进换行
- 嵌套层数不应超过3层
- 代码中禁止使用硬编码，把一些数字或字符串定义成常量
- 用4个空格替代TAB符
- 恰当使用`TODO:`和`FIXME:`
- `if/else`,`switch`,`for`,`while`...

### 2 异常处理

**基本原则**：`Don't Ignore Exceptions. Don't Catch Generic Exception.`

- 异常可以抛给上层处理、或在catch块中抛给上层处理、或直接在try/catch中打印并做出合理处理，如果catch块不做处理需要注明原因
- 不要直接捕获通用Exception基类，对于已知的每种异常，catch中需要具体列出来，并对每种异常做出相应处理，形如`try/catch(JSONException e)/catch(Exception e)/finally`，先catch已知异常再catch通用Exception
- 在某些情况下，允许直接捕获Exception基类。如为了防止在UI或批处理任务中出现错误，可以在应用顶层加上`try/catch(Exception e)`，但是需要注明原因

## 0x03 代码注释

对于需要说明的代码，最好写注释说明。

## 0x04 代码示例

    public class CodingRuler {
        /** 公有的常量注释 */
        public static final String ACTION_MAIN = "android.intent.action.MAIN";
        /** 私有的常量注释（同类型的常量可以分块并紧凑定义） */
        private static final int MSG_AUTH_NONE    = 0;
        private static final int MSG_AUTH_SUCCESS = 1;
        private static final int MSG_AUTH_FAILED  = 2;
        /** 保护的成员变量注释 */
        protected Object mObject0;
        /** 私有的成员变量 mObject1 注释（同类型的成员变量可以分块并紧凑定义） */
        private Object mObject1;
        /** 私有的成员变量 mObject2 注释 */
        private Object mObject2;
        /** 私有的成员变量 mObject3 注释 */
        private Object mObject3;
        /**
         * 对于注释多于一行的，采用这种方式来
         * 定义该变量
         */
        private Object mObject4;
        
        /**
         * 公有方法描述...
         * 
         * @param param1  参数1描述...
         * @param param2  参数2描述...
         * @param paramXX 参数XX描述...
         */
        public void doSomething(int param1, float param2, String paramXX) {
            // TODO  使用TODO来标记代码，说明标识处有功能代码待编写
            // FIXME 使用FIXME来标记代码，说明标识处代码需要修正，甚至代码是
            //       错误的，不能工作，需要修复
        }
    
        /**
         * 保护方法描述...
         */
        @Deprecated
        protected void doSomething() {
            // ...implementation
        }
    
        /**
         * 私有方法描述...
         * 
         * @param param1  参数1描述...
         * @param param2  参数2描述...
         */
        private void doSomethingInternal(int param1, float param2) {
            // ...implementation        
        }
    
        /**
         * 条件表达式原则。
         */
        private void conditionFun() {
            boolean condition1 = true;
            boolean condition2 = false;
            boolean condition3 = false;
            boolean condition4 = false;
            boolean condition5 = false;
            boolean condition6 = false;
            // 原则： 1\. 所有 if 语句必须用 {} 包括起来，即便只有一句，禁止使用不带{}的语句
            //       2\. 在含有多种运算符的表达式中，使用圆括号来避免运算符优先级问题
            //       3\. 判断条件很多时，请将其它条件换行
            if (condition1) {
                // ...implementation
            }
            if (condition1) {
                // ...implementation
            } else {
                // ...implementation
            }
            if (condition1)          /* 禁止使用不带{}的语句 */
                condition3 = true;
            if ((condition1 == condition2) 
                || (condition3 == condition4)
                || (condition5 == condition6)) {
            }
        }
    
        /**
         * Switch语句原则。
         */
        private void switchFun() {
            // 原则： 1\. 请默认写上 default 语句，保持完整性
            int code = MSG_AUTH_SUCCESS;
            switch (code) {
            case MSG_AUTH_SUCCESS:
                break;
            case MSG_AUTH_FAILED:
                break;
            case MSG_AUTH_NONE:
                /* Falls through */
            default:
                break;
            }
        }

        /**
         * 循环表达式。
         */
        private void circulationFun() {
            // 原则： 1\. 循环中必须有终止循环的条件或语句，避免死循环
            //       2\. 循环要尽可能的短, 把长循环的内容抽取到方法中去
            //       3\. 嵌套层数不应超过3层, 要让循环清晰可读
            int array[] = { 1, 2, 3, 4, 5 };
            for (int data : array) {
                // ...implementation
            }
            int length = array.length;
            for (int ix = 0; ix  < length; ix++) {
                // ...implementation
            }
            boolean condition = true;
            while (condition) {
                // ...implementation
            }
            do {
                // ...implementation
            } while (condition);
        }

        /**
         * 异常捕获原则。
         */
        private void exceptionFun() {
            // 原则： 1\. 捕捉异常是为了处理它，通常在异常catch块中输出异常信息。
            //       2\. 资源释放的工作，可以放到 finally 块部分去做。如关闭 Cursor 等。
            try {
                // ...implementation
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
            }
        }

        /**
         * 其它原则（整理中...）。
         */
        private void otherFun() {
            // TODO
        }
    }

## 0x05 参考资料

- [http://source.android.com/source/code-style.html](http://source.android.com/source/code-style.html)
- [https://google.github.io/styleguide/javaguide.html](https://google.github.io/styleguide/javaguide.html)
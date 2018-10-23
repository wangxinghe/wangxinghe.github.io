---
layout: post
comments: true
title: "Android学习笔记——关于代码的组织"
description: "Android学习笔记——关于代码的组织"
category: Android
tags: [Android]
---

**1、App组件化**    
**2、组件基本结构**     
**（1）MVC**     
**（2）MVP**     
**（3）MVVM**     
**（4）VIPER**     
**3、设计模式**     
**（1）单例模式**     
**（2）Buildr模式**     
**（3）Proxy模式**     
**（4）......**        
**4、参考文档**    

<!--more-->

### 1、App组件化   

之前写的一篇文章：[App架构之组件化理解](http://mouxuejie.com/blog/2016-03-20/architecture-componentization/) 

组件间通过`Router`通信，这个Router是业务无关的一个基础模块。可以通过Annotation实现。

组件化、模块化属于App级别的架构。

### 2、组件基本结构     

下面的架构是属于组件或模块级别，而不是整个App的架构。

#### （1）MVC     

![](/image/2018-05-12-learning-notes-code-organize/MVC.png)   

Model：包括Bean数据结构、业务逻辑处理（如数据库操作、网络请求等）    
View：就是展现出来的用户界面。   
Controller：Model和View可以直接通信，没有完全起到隔离作用，不算一个纯粹的控制器。     

存在的问题是：Model和View是耦合的。

#### （2）MVP     

![](/image/2018-05-12-learning-notes-code-organize/MVP.png)    

Model：包括Bean数据结构、业务逻辑处理    
View：就是展现出来的用户界面。   
Presenter：Model和View之间通信的桥梁，是一个控制器。    

特点：Model和View解耦。

Demo参考：[应用MVP模式写出可维护的优美Android应用
](http://blog.zhaiyifan.cn/2015/06/01/use-mvp-to-write-nice-code/)

#### （3）MVVM     

![](/image/2018-05-12-learning-notes-code-organize/MVVM.png)    

Model：包括Bean数据结构、业务逻辑处理    
View：就是展现出来的用户界面。   
ViewModel：就是与界面(view)对应的Model。因为，数据库结构往往是不能直接跟界面控件一一对应上的，所以，需要再定义一个数据对象专门对应view上的控件。而ViewModel的职责就是把model对象封装成可以显示和接受输入的界面数据对象。PS：不一定是Bean，里面可能也有业务逻辑处理，只不过这个是和View一一对应。和Presenter还是有差别。

View和ViewModel之间通过DataBinding进行双向绑定。

特点：和MVP相比，View和ViewModel通过DataBinding实现一一对应关系。

Demo参考：[选择恐惧症的福音！教你认清MVC，MVP和MVVM](http://zjutkz.net/2016/04/13/%E9%80%89%E6%8B%A9%E6%81%90%E6%83%A7%E7%97%87%E7%9A%84%E7%A6%8F%E9%9F%B3%EF%BC%81%E6%95%99%E4%BD%A0%E8%AE%A4%E6%B8%85MVC%EF%BC%8CMVP%E5%92%8CMVVM/)    

关于DataBinding：[Android基础——框架模式MVVM之DataBinding的实践](https://blog.csdn.net/qq_30379689/article/details/53037430)

关于ViewModel的理解参考：[MVVM模式中ViewModel和View、Model有什么区别](https://blog.csdn.net/AlbenXie/article/details/72771141)

#### （4）VIPER     

![](/image/2018-05-12-learning-notes-code-organize/VIPER.png)    

View：展现出来的用户界面。   
Presenter：View和Interactor之间的桥梁，相当于是一个控制器，不做具体业务逻辑。    
Entity：Bean数据结构，不包括业务逻辑处理。    
Interactor：业务逻辑处理。    
Router：模块之间跳转，比如从当前A页面跳到B页面，就是由Router负责。（PS：如果有App级别的Router模块负责跳转，则组件内级别的Router跳转的话可以调用App级别的Router实现跳转）    

VIPER模式，将MVP架构中原本由Model单独负责的Bean数据结构和业务逻辑处理分拆成2个类处理，Entity处理Bean数据结构，Interactor处理业务逻辑。此外在MVP基础上还增加了Router，负责模块跳转。    

特点：和上面几个相比，减轻了Model的工作量，避免了`胖Model`的问题（备注：胖Model和瘦Model）

参考：
[在 Android 上使用 VIPER 架构](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0329/7754.html)    
[Architecting iOS Apps with VIPER](https://www.objc.io/issues/13-architecture/viper/)    
[https://gist.github.com/marciogranzotto/1c96e87484a17bd914e159b2604e6469](https://gist.github.com/marciogranzotto/1c96e87484a17bd914e159b2604e6469)    

延伸阅读：[新版Uber App移动架构设计](https://zhuanlan.zhihu.com/p/24489480)

uber/RIBs    
[https://github.com/uber/RIBs](https://github.com/uber/RIBs)    
[https://juejin.im/post/5a9f88466fb9a028e46e308a](https://juejin.im/post/5a9f88466fb9a028e46e308a)    


TODO：clean architecture

个人观点：    
MVP：适用于不太复杂的模块。    
MVVM：需要引入DataBinding框架，用起来比较复杂，感觉和MVP相比也没啥优势。    
VIPER：适用于逻辑比较复杂的模块，职责分明。    
上面那些架构都是在MVC基础上的优化，并不是死的，并不一定要去套上面的模式，遵循一些模块化的基本原则即可。    

### 3、设计模式     

#### （1）单例模式     

主要包括4种类型的写法：`饿汉式`，`双重检查锁`，`静态内部类`，`枚举`。且后面3种都属于懒汉式。

1.`饿汉式` static final field

    public class Singleton{
        //类加载时就初始化
        private static final Singleton instance = new Singleton();
    
        private Singleton(){}
        public static Singleton getInstance(){
            return instance;
        }
    }

**特点**：类加载的时候就会创建单例。    
**缺点**：不适用于实例创建依赖参数或者配置文件的场景；不适用于被使用的频率少的情况。    

2.`双重检查锁`

    public class Singleton{
        private volatile static Singleton instance;
        
        private Singleton (){}
        
        public static Singleton getSingleton() {
            if (instance == null) { //Single Checked                         
                synchronized (Singleton.class) {
                    if (instance == null) { //Double Checked
                        instance = new Singleton();
                    }
                }
            }
            return instance ;
        }
    ｝
    
    
    //懒汉式，线程安全（不建议）    
    public class Singleton{
        private static Singleton instance;

        private Singleton (){}
        
        public static synchronized Singleton getInstance() {
            if (instance == null) {
                instance = new Singleton();
            }
            return instance;
        }
    ｝

要点：    
（1）instance = new Singleton()`非原子操作`，可以拆分为如下4步：    

- 栈内存开辟空间给instance引用
- 堆内存开辟空间
- 堆内存中初始化Singleton对象    
- 栈中引用指向堆内存空间地址    

（2）`volatile` 保证内存可见性，防止`指令重排`。（`Java 5`之前无效）

如果不使用volatile，new操作可能会出现指令重排，即执行顺序是1-2-4-3时，当第1个线程执行instance = new Singleton()操作，执行到1-2-4步时，这时第2个线程执行到Single Check这个位置的if (instance == null)，发现非空就会返回单例，然而事实上这个单例对象还没有初始化完。

关于防止指令重排，实际上是会在volatile变量操作的前后插入内存屏障，使得位于屏障前的指令先于volatile指令执行，位于屏障后的指令后于volatile指令后执行。网上很多文章讲解要么不清楚要么有错误，这块还需要查书研究。

3.`静态内部类` static nested class

    public class Singleton {  
        private static class SingletonHolder {  
            private static final Singleton INSTANCE = new Singleton();  
        }  
    
        private Singleton (){}  
    
        public static final Singleton getInstance() {  
            return SingletonHolder.INSTANCE; 
        }  
    }
    
特点：懒加载，只有在SingletonHolder类被引用时才会完成INSTANCE对象的实例化，利用ClassLoader机制保证线程安全。    


4.`枚举` Enum    

    public enum Singleton{
        INSTANCE;
    }

特点：枚举类是在第一次访问时才被实例化，默认线程安全；在`反序列化`时可以自动防止重新创建新的对象。（枚举`Java 5`之后才引入）

5.`登记式单例`，使用 Map 容器来管理单例模式。

	private static Map<String Singleton> objMap = new HashMap<String Singleton>();
	
	public static void registerService(String key, Singleton instance) {
		if (!objMap.containsKey(key) ) {
			objMap.put(key, instance) ;
		}
	}
	
	public static Singleton getService(String key) {
		return objMap.get(key) ;
	}

Android源码里各种系统Service就是就是使用的这种方式。比如：

    private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }
    
    @Override
    public Object getSystemService(String name) {
        // 根据name来获取服务
        ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
        return fetcher == null ? null : fetcher.getService(this);
    }
    
    
    //注册ActivityManager
    registerService(ACTIVITY_SERVICE, new ServiceFetcher() {
        public Object createService(ContextImpl ctx) {
            return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
        }
    });

    //注册LayoutInflater
    registerService(LAYOUT_INFLATER_SERVICE, new ServiceFetcher() {
        public Object createService(ContextImpl ctx) {
            return PolicyManager.makeNewLayoutInflater(ctx.getOuterContext());
        }
    });
    
    //获取LayoutInflater
    LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

破坏单例模式的场景：    

- 序列化
- 反射
- 克隆（实现 Cloneable 接口）


参考：    
[那些年，我们一起写过的“单例模式”](https://mp.weixin.qq.com/s/wEK3UcHjaHz1x-iXoW4_VQ)    
[如何正确地写出单例模式](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)    


#### （2）Buildr模式     

![](/image/2018-05-12-learning-notes-code-organize/builder-uml.png)   

`定义`：将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。    

`使用场景`：    

- 需要生成的产品对象有`复杂的内部结构`，这些产品对象`具备共性`；    
- `隔离`复杂对象的创建和使用，并使得`相同的创建过程`可以创建不同的产品。     


AlertDialog就是Builder模式，源码如下：

    //Product
    public class AlertDialog extends Dialog implements DialogInterface {
        private AlertController mAlert;
    
        protected AlertDialog(Context context) {
            this(context, resolveDialogTheme(context, 0), true);
        }
        
        @Override
        public void setTitle(CharSequence title) {
            super.setTitle(title);
            mAlert.setTitle(title);
        }
        
        public void setMessage(CharSequence message) {
            mAlert.setMessage(message);
        }
        
        public void setView(View view) {
            mAlert.setView(view);
        }

        //Builder
        public static class Builder {
            private final AlertController.AlertParams P;
        
            public Builder(Context context) {
                this(context, resolveDialogTheme(context, 0));
            }
        
            public Builder(Context context, int theme) {
                P = new AlertController.AlertParams(new ContextThemeWrapper(
                        context, resolveDialogTheme(context, theme)));
                mTheme = theme;
            }
        
            public Builder setTitle(CharSequence title) {
                P.mTitle = title;
                return this;
            }
        
            public Builder setMessage(CharSequence message) {
                P.mMessage = message;
                return this;
            }
        
            public Builder setView(View view) {
                P.mView = view;
                P.mViewLayoutResId = 0;
                P.mViewSpacingSpecified = false;
                return this;
            }
        
            public AlertDialog create() {
                final AlertDialog dialog = new AlertDialog(P.mContext, mTheme, false);
                P.apply(dialog.mAlert);
                dialog.setCancelable(P.mCancelable);
                if (P.mCancelable) {
                    dialog.setCanceledOnTouchOutside(true);
                }
                dialog.setOnCancelListener(P.mOnCancelListener);
                dialog.setOnDismissListener(P.mOnDismissListener);
                if (P.mOnKeyListener != null) {
                    dialog.setOnKeyListener(P.mOnKeyListener);
                }
                return dialog;
            }
        }
    }
    
用户调用代码，即Director：
      
    private void showDialog(Context context) {  
        AlertDialog.Builder builder = new AlertDialog.Builder(context);  
        builder.setIcon(R.drawable.icon);  
        builder.setTitle("Title");  
        builder.setMessage("Message");  
        builder.setPositiveButton("Button1",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button1");  
                    }  
                });  
        builder.setNeutralButton("Button2",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button2");  
                    }  
                });  
        builder.setNegativeButton("Button3",  
                new DialogInterface.OnClickListener() {  
                    public void onClick(DialogInterface dialog, int whichButton) {  
                        setTitle("点击了对话框上的Button3");  
                    }  
                });  
        builder.create().show();  // 构建AlertDialog， 并且显示
    } 

#### （3）Proxy模式     

![](/image/2018-05-12-learning-notes-code-organize/proxy-uml.png)   

`定义`：代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。

`使用场景`：客户不想或者不能够直接引用一个目标对象时，代理对象可以在客户和目标对象之间起到中介的作用。

Android源码中的ActivityManagerService涉及到2个知识点。即`Proxy模式`和`Binder机制`。

下面的代码体现了Proxy设计模式：

    //RealObject（ActivityManagerService继承了ActivityManagerNative）
    public abstract class ActivityManagerNative extends Binder implements IActivityManager {
    
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
            switch (code) {
            case START_ACTIVITY_TRANSACTION:
            {
                data.enforceInterface(IActivityManager.descriptor);
                IBinder b = data.readStrongBinder();
                IApplicationThread app = ApplicationThreadNative.asInterface(b);
                String callingPackage = data.readString();
                Intent intent = Intent.CREATOR.createFromParcel(data);
                String resolvedType = data.readString();
                IBinder resultTo = data.readStrongBinder();
                String resultWho = data.readString();
                int requestCode = data.readInt();
                int startFlags = data.readInt();
                ProfilerInfo profilerInfo = data.readInt() != 0 ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
                Bundle options = data.readInt() != 0 ? Bundle.CREATOR.createFromParcel(data) : null;
                
                int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
                    
                reply.writeNoException();
                reply.writeInt(result);
                return true;
            }
            ...
        }
    }
       
    //ProxyObject    
    class ActivityManagerProxy implements IActivityManager {
        public ActivityManagerProxy(IBinder remote) {
            mRemote = remote;
        }

        public IBinder asBinder() {
            return mRemote;
        }

        public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
                String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
            //将数据装进data
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(caller != null ? caller.asBinder() : null);
            data.writeString(callingPackage);
            intent.writeToParcel(data, 0);
            data.writeString(resolvedType);
            data.writeStrongBinder(resultTo);
            data.writeString(resultWho);
            data.writeInt(requestCode);
            data.writeInt(startFlags);
            if (profilerInfo != null) {
                data.writeInt(1);
                profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
            } else {
                data.writeInt(0);
            }
            if (options != null) {
                data.writeInt(1);
                options.writeToParcel(data, 0);
            } else {
                data.writeInt(0);
            }
            //创建空的reply
            Parcel reply = Parcel.obtain();

            //请求服务端对象，执行任务            
            mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
            
            //任务执行完后，得到reply
            reply.readException();
            int result = reply.readInt();
            
            reply.recycle();
            data.recycle();
            return result;
        }
    }


#### （4）......        

### 4、参考文档    
（1）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)    
（2）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)    
（3）[https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)    
（4）[iOS应用架构谈](https://casatwy.com/iosying-yong-jia-gou-tan-kai-pian.html)    
（5）[iOS 架构模式–解密 MVC，MVP，MVVM以及VIPER架构](http://ios.jobbole.com/83727/)    
（6）[Android源码设计模式分析项目](https://github.com/simple-android-framework/android_design_patterns_analysis)    
（7）[https://blog.csdn.net/column/details/14783.html](https://blog.csdn.net/column/details/14783.html)    



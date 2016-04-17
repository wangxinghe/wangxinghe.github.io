---
layout: post
comments: true
title: "Android进程间通信机制Binder应用层分析"
description: "Android进程间通信机制Binder应用层分析"
category: android
tags: [Android]
---


##  1. 概述

今天学姐复习了下Android进程间通信方式，打算从`基本概念`、`使用原因`、`基本使用`、`原理分析`几个方面来讲讲。

想要对进程间通信方式有个相对全面的了解，就先从几个比较重要的概念`IPC`、`AIDL`、`Binder`说起吧。

<!--more-->

（1）`IPC`：Inter-Process Communication，即进程间通信

（2）`AIDL`：Android Interface Definition Language，即Android接口定义语言。Client和Service要实现        
跨进程通信，必须遵循的接口规范。需要创建.aidl文件，外在表现上和Java中的interface有点类似。

（3）`Binder`：Android进程间通信是通过`Binder`来实现的。远程Service在Client绑定服务时，会在onBind()的回调中返回一个Binder，当Client调用bindService()与远程Service建立连接成功时，会拿到远程Binder实例，从而使用远程Service提供的服务。

##  2. 为什么使用Binder?

下面内容学姐参考别人的博文，进行了总结。

Linux系统进程间通信方式：

（1）传统机制。如管道(Pipe)、信号(Signal)和跟踪(Trace)，适用于父子进程或兄弟进程，其中命名管道(Named Pipe)，支持多种类型进程

（2）System V IPC机制。如报文队列(Message)、共享内存(Share Memory)和信号量(Semaphore)

（3）Socket通信

然而，以上方式在性能和安全性方面存在不足：

（1）性能方面。管道和队列采用存储转发方式，数据从发送方缓存区->内核缓存区->接收方缓存区，会经历两次拷贝过程；共享内存无拷贝但控制复杂；Socket传输效率低下，且连接的建立与中断有一定开销。

（2）安全性方面。传统IPC方式无法获得对方进程可靠的UID和PID，无法鉴别对方身份，若采用在数据包里填入UID/PID的方式，容易被恶意程序利用；传统IPC方式的接入点是开放性的，无法建立私有通道，容易被恶意程序猜测出接收方地址，获得连接。

而`Binder`是基于C/S通信模式，传输过程只需要一次拷贝，且为Client添加UID/PID身份，性能和安全性更好，因此Android进程间通信使用了`Binder`。

##  3. 基本使用

假设本地Client需要使用远程Service的计算器功能。示例Demo: [binderDemo](https://github.com/wangxinghe/binderDemo)

### （1）新建Client和远程Service两个Android工程。

### （2）在远程Service工程中，创建ICalculator.aidl文件，并创建CalculatorService类。
    
    // ICalculator.aidl
    package me.wangxinghe.ipc;

    // Declare any non-default types here with import statements

    interface ICalculator {
        /**
         * Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString);

        int add(int a, int b);
        int minus(int a, int b);
    }

    //给Client提供服务的Service文件
    public class CalculatorService extends Service {
        public CalculatorService() {
        }

        @Override
        public void onCreate() {
            super.onCreate();
        }

        @Override
        public IBinder onBind(Intent intent) {
            //当Client调用bindService()时，远程Service通过onBind()回调返给Client
            return mBinder;
        }

        @Override
        public int onStartCommand(Intent intent, int flags, int startId) {
            return super.onStartCommand(intent, flags, startId);
        }

        @Override
        public void onDestroy() {
            super.onDestroy();
        }

        远程Service向Client提供的服务，接口方法add()和minus()实现由Service决定
        private ICalculator.Stub mBinder = new ICalculator.Stub() {
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,         
            double aDouble, String aString) throws RemoteException {

            }

            @Override
            public int add(int a, int b) throws RemoteException {
                return a + b;
            }

            @Override
            public int minus(int a, int b) throws RemoteException {
                return a - b;
            }
        };

    }

    //AndroidManifest中Service组件声明，由于是Client工程不能显示调用远程Service，只能采用隐式调用的方  
      式，因此需要填写category和action
    <service
        android:name=".CalculatorService"
        android:enabled="true"
        android:exported="true"
        android:process=":remote">
        <intent-filter>
            <category android:name="android.intent.category.DEFAULT"/>
            <action android:name="com.wangxinghe.ipc_server.CalculatorService"/>
        </intent-filter>
    </service>

### （3）在Client工程中，拷贝以上.aidl文件及目录，使用Service提供的服务。

    //远程Service连接回调
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //连接成功，获取远程Service的Binder
            mCalculator = ICalculator.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mCalculator = null;
        }
    };

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.bind_service:
                //绑定服务
                Intent intent = new Intent(ACTION_CALCULATOR_SERVICE);
                bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
                break;
            case R.id.unbind_service:
                //断开服务
                unbindService(mServiceConnection);
                break;
            case R.id.add:
                //调用远程服务的add()
                int result = 0;
                try {
                    if (mCalculator != null) {
                        result = mCalculator.add(2, 1);
                    }
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                mResultTv.setText(result + "");
                break;
            case R.id.minus:
                //调用远程服务的minus()
                int _result = 0;
                try {
                    if (mCalculator != null) {
                        _result = mCalculator.minus(2, 1);
                    }
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                mResultTv.setText(_result + "");
                break;
        }
    }
    
###（4）启动Client和Service工程，即可实现进程间通信。

##  4. 原理分析

由于`Binder`机制涉及的东西很多。学姐本文并不打算深入到内核源码，关于Client是怎样获取到远程Binder的会在后续文章再讲述。下面主要是从应用层角度分析。

我们先看看ICalculator.aidl编译生成的ICalculator.java文件。

    package me.wangxinghe.ipc;
    // Declare any non-default types here with import statements

    public interface ICalculator extends android.os.IInterface
    {
        
        /** Local-side IPC implementation stub class. */
        public static abstract class Stub extends android.os.Binder implements     
        me.wangxinghe.ipc.ICalculator
        {
            //接口描述符
            private static final java.lang.String DESCRIPTOR = "me.wangxinghe.ipc.ICalculator";
            
            /** Construct the stub at attach it to the interface. */
            public Stub()
            {
                //添加Binder和接口描述符到该Binder中，便于后续本地查找Binder
                this.attachInterface(this, DESCRIPTOR);
            }
            
            /**
             * Cast an IBinder object into an me.wangxinghe.ipc.ICalculator interface,
             * generating a proxy if needed.
             */
            public static me.wangxinghe.ipc.ICalculator asInterface(android.os.IBinder obj)
            {
                if ((obj==null)) {
                    return null;
                }
                
                //根据接口描述符，从Binder中查找对应的接口，若有则直接返回
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin!=null)&&(iin instanceof me.wangxinghe.ipc.ICalculator))) {
                    return ((me.wangxinghe.ipc.ICalculator)iin);
                }
                
                //否则返回接口的代理对象，并将远端Binder传入代理
                return new me.wangxinghe.ipc.ICalculator.Stub.Proxy(obj);
            }
            
            @Override 
            public android.os.IBinder asBinder()
            {
                //返回Binder
                return this;
            }
            
            @Override
            public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply,                                      
            int flags) throws android.os.RemoteException
            {
                switch (code) 
                {
                    case INTERFACE_TRANSACTION:
                    {
                        reply.writeString(DESCRIPTOR);
                        return true;
                    }
                    case TRANSACTION_basicTypes:
                    {
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        long _arg1;
                        _arg1 = data.readLong();
                        boolean _arg2;
                        _arg2 = (0!=data.readInt());
                        float _arg3;
                        _arg3 = data.readFloat();
                        double _arg4;
                        _arg4 = data.readDouble();
                        java.lang.String _arg5;
                        _arg5 = data.readString();
                        this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                        reply.writeNoException();
                        return true;
                    }
                    case TRANSACTION_add:
                    {
                        //解析data Parcel参数，执行具体的加法操作，并将计算结果写进reply Parcel
                        //add()由Service实现
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        int _arg1;
                        _arg1 = data.readInt();
                        int _result = this.add(_arg0, _arg1);
                        reply.writeNoException();
                        reply.writeInt(_result);
                        return true;
                    }
                    case TRANSACTION_minus:
                    {
                        //解析data Parcel参数，执行具体的减法操作，并将计算结果写进reply Parcel
                        //minus()由Service实现
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        int _arg1;
                        _arg1 = data.readInt();
                        int _result = this.minus(_arg0, _arg1);
                        reply.writeNoException();
                        reply.writeInt(_result);
                        return true;
                    }
                }
                return super.onTransact(code, data, reply, flags);
            }
            
            //代理类
            private static class Proxy implements me.wangxinghe.ipc.ICalculator
            {
                //远程Binder
                private android.os.IBinder mRemote;
                
                Proxy(android.os.IBinder remote)
                {
                    mRemote = remote;
                }
                
                @Override
                public android.os.IBinder asBinder()
                {
                    //返回远程Binder
                    return mRemote;
                }
                
                public java.lang.String getInterfaceDescriptor()
                {
                    //返回接口描述符
                    return DESCRIPTOR;
                }
                
                /**
                 * Demonstrates some basic types that you can use as parameters
                 * and return values in AIDL.
                 */
                @Override
                public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, 
                double aDouble, java.lang.String aString) throws android.os.RemoteException
                {
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        _data.writeInt(anInt);
                        _data.writeLong(aLong);
                        _data.writeInt(((aBoolean)?(1):(0)));
                        _data.writeFloat(aFloat);
                        _data.writeDouble(aDouble);
                        _data.writeString(aString);
                        mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                        _reply.readException();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                }
                
                @Override
                public int add(int a, int b) throws android.os.RemoteException
                {
                    //执行加法操作，将接口描述符和参数封装进data Parcel，调用远程Binder执行加法操作，
                             //并从reply Parcel中解析出结果，返回结果。该过程是同步操作
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    int _result;
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        _data.writeInt(a);
                        _data.writeInt(b);
                        mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                        _reply.readException();
                        _result = _reply.readInt();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                    return _result;
                }
                
                @Override 
                public int minus(int a, int b) throws android.os.RemoteException
                {
                    //执行减法操作，将接口描述符和参数封装进data Parcel，调用远程Binder执行减法操作，
                    //并从reply Parcel中解析出结果，返回结果。该过程是同步操作
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    int _result;
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        _data.writeInt(a);
                        _data.writeInt(b);
                        mRemote.transact(Stub.TRANSACTION_minus, _data, _reply, 0);
                        _reply.readException();
                        _result = _reply.readInt();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                    return _result;
                }
            }
        
            static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 
            0);
            static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
            static final int TRANSACTION_minus = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        }

        /**
         * Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double 
        aDouble, java.lang.String aString) throws android.os.RemoteException;
    
        //接口方法
        public int add(int a, int b) throws android.os.RemoteException;
    
        //接口方法
        public int minus(int a, int b) throws android.os.RemoteException;
    }

在`基本使用`部分，我们可以看到，Client在与Service建立连接成功后，会拿到远程Binder实例（此处不能直接使用远程Binder原因还不是很清楚），并调用Stub的asInterface方法将其转换成ICalculator接口。

    mCalculator = ICalculator.Stub.asInterface(service);
    
这一步执行的操作是：根据接口描述符，从Binder中本地查找对应的接口，若有则直接返回；否则，将Binder对象传给本地代理Stub.Proxy对象，并返回本地代理，由本地代理来接管相应的服务，Proxy也实现了ICalculator接口。

这一步之后，Client就可以使用远程Service提供的服务了。

再看看onClick()事件中的加法操作。

    result = mCalculator.add(2, 1);
    
mCalculator即为上面得到的本地代理对象，其add()的实现是

    @Override 
    public int add(int a, int b) throws android.os.RemoteException
    {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(a);
            _data.writeInt(b);
            mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
            _reply.readException();
            _result = _reply.readInt();
        }
        finally {
            _reply.recycle();
            _data.recycle();
        }
        return _result;
    }

可以看到实际上是调用连接建立成功后的远程Binder的transact(add)方法。再看看Binder.transact实现

    public final boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (false) Log.v("Binder", "Transact: " + code + " to " + this);
        if (data != null) {
            data.setDataPosition(0);
        }
        boolean r = onTransact(code, data, reply, flags);
        if (reply != null) {
            reply.setDataPosition(0);
        }
        return r;
    }
    
可以看到最后调用了远程Binder的onTransact()方法，也就是走到了远程Binder的Stub.onTransact()方法。这个方法实现是

    @Override 
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) 
    throws android.os.RemoteException
    {
        switch (code)
        {
            case TRANSACTION_add:
            {
                data.enforceInterface(DESCRIPTOR);
                int _arg0;
                _arg0 = data.readInt();
                int _arg1;
                _arg1 = data.readInt();
                int _result = this.add(_arg0, _arg1);
                reply.writeNoException();
                reply.writeInt(_result);
                return true;
            }
            ...
        }
    }
    
这里最终调用到了远程Service工程创建的ICalculator.Stub实例mBinder的add()方法即a+b，因此得到了最后的结果。

好了，我们再来梳理下，Client获得并使用Service服务的过程：

（1）Client与Service建立连接，得到远程Binder（这个Binder就是Service的onBind返回的Binder，但是客户端不能直接使用，具体原因还不是很明确）。
 
（2）将远程Binder传给本地代理，得到本地代理Stub.Proxy实例。

（3）通过本地代理Stub.Proxy实例间接调用远程Binder对象的add()方法。
    具体实现是：由于远程Binder已经传给了本地代理Stub.Proxy。那么通过本地代理Stub.Proxy实例间接调用远程Binder的transact(TRANSACTION_add)操作；调用远程Binder的onTransact()方法；调用远程Binder的实现类ICalculator.Stub的add()方法
    
（4）得到结果
    
下面再总结下`ICalculator`、`Stub`、`Proxy`这3个类的关系：

`ICalculator`：就是一个接口类。内部包括之前的接口方法和静态Stub类。

`Stub`：远程Binder实现类。继承类`Binder`和`ICalculator`接口。

`Proxy`：远程Binder代理类。实现`ICalculator`接口。

##  5. 总结

（1）Binder相当于不同进程间数据通信的通道

（2）核心是代理模式，使用本地代理Proxy操作远端Binder，使用相应服务

（3）如理解有误，欢迎指正

##  6. 参考链接

（1）[http://blog.csdn.net/singwhatiwanna/article/details/19756201](http://blog.csdn.net/singwhatiwanna/article/details/19756201)

（2）[http://blog.csdn.net/luoshengyang/article/details/6618363](http://blog.csdn.net/luoshengyang/article/details/6618363)


## 欢迎大家关注我的公众号：学姐的IT专栏

![学姐的IT专栏](/images/qrcode_for_gh_771805c73e44_430.jpg)

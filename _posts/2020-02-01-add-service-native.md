---
layout: post
comments: true
title: "Service注册和Binder线程池启动过程--Native层"
description: "Service注册和Binder线程池启动过程--Native层"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析**Native服务的注册**和**启动Binder线程池**过程. 以Media服务为例.

## 0.文件结构

	frameworks/av/media/mediaserver/mediaserver.rc
	frameworks/av/media/mediaserver/main_mediaserver.cpp
	frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp

	frameworks/native/libs/binder/IServiceManager.cpp
	frameworks/native/libs/binder/ProcessState.cpp
	frameworks/native/libs/binder/IPCThreadState.cpp

## 1.时序图

TODO

## 2.注册和启动入口

### 2.1 服务定义
[ -> frameworks/av/media/mediaserver/mediaserver.rc ]

    // 定义一个service, 名称为media, 路径为/system/bin/mediaserver
    service media /system/bin/mediaserver
	    class main
	    user media
	    group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm
	    ioprio rt 4
	    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks

手机Root后, 通过Android Studio的Device File Explorer可以看到/system/bin/mediaserver这个目录.

### 2.2 服务注册和线程启动入口
[ -> frameworks/av/media/mediaserver/main_mediaserver.cpp ]

	int main(int argc __unused, char **argv __unused)
	{
	    signal(SIGPIPE, SIG_IGN);
	 
	    // 创建ProcessState对象proc
	    sp<ProcessState> proc(ProcessState::self());
	    // 获取servicemanager的BpBinder对象
	    sp<IServiceManager> sm(defaultServiceManager());
	    // 初始化MediaPlayerService
	    MediaPlayerService::instantiate();
	    ...
	    // 创建并启动线程池(启动1个线程)
	    ProcessState::self()->startThreadPool();
	    // 将当前线程加入线程池(启动1个线程)
	    IPCThreadState::self()->joinThreadPool();
	}

## 3.注册服务

### 3.1 MediaPlayerService::instantiate
[ -> frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp ]

	void MediaPlayerService::instantiate() {
	    defaultServiceManager()->addService(
	            String16("media.player"), new MediaPlayerService());
	}

由文章 **[ServiceManager获取过程--Native层](http://mouxuejie.com/blog/2020-01-05/get-service-manager-native/)** 可知defaultServiceManager拿到BpServiceManager对象.

### 3.2 BpServiceManager.addService
[ -> frameworks/native/libs/binder/IServiceManager.cpp ]

	class BpServiceManager : public BpInterface<IServiceManager>
	{
	    virtual status_t addService(const String16& name,
			    const sp<IBinder>& service,
			    bool allowIsolated,
				int dumpsysPriority)
		{
	        Parcel data, reply;
			data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
	        data.writeString16(name);
	        data.writeStrongBinder(service);
	        data.writeInt32(allowIsolated ? 1 : 0);
	        data.writeInt32(dumpsysPriority);
	        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
	        return err == NO_ERROR ? reply.readExceptionCode() : err;
	    }
	};

这个过程和Java服务注册过程一模一样：**[Service注册过程--Java服务](http://mouxuejie.com/blog/2020-01-18/add-service/)** 

## 4.启动Binder线程池

### 4.1 ProcessState
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

#### 4.1.1 startThreadPool

	void ProcessState::startThreadPool()
	{
	    AutoMutex _l(mLock);
	    if (!mThreadPoolStarted) {
	        mThreadPoolStarted = true;
	        // 创建线程
	        spawnPooledThread(true);
	    }
	}

#### 4.1.2 spawnPooledThread

	void ProcessState::spawnPooledThread(bool isMain)
	{
	    if (mThreadPoolStarted) {
	        // Binder线程名称格式Binder:pid_seq, seq为序列号,每次累加1
	        String8 name = makeBinderThreadName();
	        // 启动线程
	        sp<Thread> t = new PoolThread(isMain);
	        t->run(name.string());
	    }
	}

#### 4.1.3 makeBinderThreadName

	// Binder线程名称格式Binder:pid_seq, seq为序列号,每次累加1
	String8 ProcessState::makeBinderThreadName() {
	    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
        pid_t pid = getpid();
        String8 name;
        name.appendFormat("Binder:%d_%X", pid, s);
        return name;
    }

### 4.2 PoolThread
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	class PoolThread : public Thread
	{
	public:
	    explicit PoolThread(bool isMain)
	        : mIsMain(isMain)
	    {
	    }
	     
	protected:
	    virtual bool threadLoop()
	    {
	        // 将当前线程加入线程池，mIsMain为true
	        IPCThreadState::self()->joinThreadPool(mIsMain);
	        return false;
	    }
	     
	    const bool mIsMain;
	};

### 4.3 IPCThreadState
[ -> frameworks/native/libs/binder/IPCThreadState.cpp ]
	
#### 4.3.1 joinThreadPool

	void IPCThreadState::joinThreadPool(bool isMain)
	{
	    // 线程进入循环标志
	    // isMain为true表示主线程(不会退出), false表示Binder驱动创建的线程
	    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
	 
	    status_t result;
	    do {
	        // 清除队列的引用
	        processPendingDerefs();
	        // 获取并执行指令
	        result = getAndExecuteCommand();
	 
	        // 非主线程且timeout,则退出线程
	        if(result == TIMED_OUT && !isMain) {
	            break;
	        }
	    } while (result != -ECONNREFUSED && result != -EBADF);
	 
	    // 线程退出循环标志
	    mOut.writeInt32(BC_EXIT_LOOPER);
        // false表示不读Binder驱动数据，只写
        talkWithDriver(false);
	}

#### 4.3.2 processPendingDerefs

	// When we've cleared the incoming command queue, process any pending derefs
	void IPCThreadState::processPendingDerefs()
	{
	    if (mIn.dataPosition() >= mIn.dataSize()) {
	        while (mPendingWeakDerefs.size() > 0 || mPendingStrongDerefs.size() > 0) {
	            while (mPendingWeakDerefs.size() > 0) {
	                RefBase::weakref_type* refs = mPendingWeakDerefs[0];
	                mPendingWeakDerefs.removeAt(0);
	                refs->decWeak(mProcess.get()); // 弱引用减1
	            }
	
	            if (mPendingStrongDerefs.size() > 0) {
	                BBinder* obj = mPendingStrongDerefs[0];
	                mPendingStrongDerefs.removeAt(0);
	                obj->decStrong(mProcess.get()); // 强引用减1
	            }
	        }
	    }
	}

#### 4.3.3 getAndExecuteCommand

	status_t IPCThreadState::getAndExecuteCommand()
	{
	    status_t result;
	    int32_t cmd;
	 
	    // 和Binder驱动交互，往Binder读写数据
	    result = talkWithDriver();
	    if (result >= NO_ERROR) {
	        size_t IN = mIn.dataAvail();
	        if (IN < sizeof(int32_t)) return result;
	        // 读取来自Binder驱动的指令
	        cmd = mIn.readInt32();
	        // 执行指令
	        result = executeCommand(cmd);
	    }
	 
	    return result;
	}

#### 4.3.4 executeCommand

	// 执行来自Binder驱动的指令，都是以BR_开头
	status_t IPCThreadState::executeCommand(int32_t cmd)
	{
	    status_t result = NO_ERROR;
	 
	    switch ((uint32_t)cmd) {
	    case BR_ERROR: ... 
	    case BR_OK: ...
	    case BR_ACQUIRE: ...
	    case BR_RELEASE: ...
	    case BR_INCREFS: ...
	    case BR_DECREFS: ... 
	    case BR_ATTEMPT_ACQUIRE: ...
	    case BR_TRANSACTION_SEC_CTX: ...
	    case BR_TRANSACTION: ...
	    case BR_DEAD_BINDER: ...
	    case BR_CLEAR_DEATH_NOTIFICATION_DONE: ...
	    case BR_FINISHED: ...
	    case BR_NOOP: ...
	    case BR_SPAWN_LOOPER: ...
	    default: ...
	    }

	    return result;
	}

## 5.总结

本文分析了**Native层服务注册**和**启动Binder线程池**2个过程.  

1.Native层服务注册  
调用链: `MediaPlayerService::instantiate -> defaultServiceManager.addService -> BpServiceManager.addService`  
这个过程和Java层服务注册过程一模一样.  

2.启动Binder线程池, 处理Client的通信请求    
(1) `ProcessState::self()->startThreadPool()`  创建并启动线程池  
调用链: `spawnPooledThread -> PoolThread.run -> IPCThreadState::joinThreadPool -> getAndExecuteCommand -> talkWithDriver -> executeCommand`  
创建并启动线程池ThreadPool, 启动一个Binder主线程(线程名Binder:pid_seq, seq为序列号,每次累加1)  
当前Binder主线程执行线程循环, 不断和Binder驱动交互进行读写数据，并处理来自Binder驱动的BR返回指令.  
(2) `IPCThreadState::self()->joinThreadPool()`  
调用链: `IPCThreadState::joinThreadPool -> getAndExecuteCommand -> talkWithDriver -> executeCommand`  
过程同上，又启动了一个Binder主线程.

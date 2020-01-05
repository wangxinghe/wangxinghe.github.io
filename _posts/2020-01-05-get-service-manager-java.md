---
layout: post
comments: true
title: "ServiceManager获取过程--Java层"
description: "ServiceManager获取过程--Java层"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

由于Android服务有Java服务和Native服务, 因此获取ServiceManager也区分Java层和Native层2种方式.
本文只分析Java层代码获取ServiceManager过程, Native层套路一样.

## 0.文件结构
	
	frameworks/base/core/java/android/os/ServiceManager.java
	frameworks/base/core/java/android/os/ServiceManagerNative.java
	frameworks/base/core/java/android/os/BinderProxy.java
	frameworks/base/core/java/com/android/internal/os/BinderInternal.java

	frameworks/base/core/jni/android_util_Binder.cpp
	 
	frameworks/native/libs/binder/ProcessState.cpp
	frameworks/native/libs/binder/IPCThreadState.cpp

## 1.时序图
![Alt text](/image/2020-01-05-get-service-manager-java/get-service-manager-java.png)

## 2.入口函数
[ -> frameworks/base/core/java/android/os/ServiceManager.java ]

	// ServiceManager.getIServiceManager()
	private static IServiceManager getIServiceManager() {
	    if (sServiceManager != null) {
	        return sServiceManager;
	    }
	 
	    sServiceManager = ServiceManagerNative
	            .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
	    return sServiceManager;
	}

## 3.BinderInternal.getContextObject
[ -> frameworks/base/core/java/com/android/internal/os/BinderInternal.java ]

    /**
     * Return the global "context object" of the system.  This is usually
	 * an implementation of IServiceManager, which you can use to find
	 * other services.
	 */
	public static final native IBinder getContextObject();

[ -> frameworks/base/core/jni/android_util_Binder.cpp ]

	static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
	{
	    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
	    return javaObjectForIBinder(env, b);
	}

### 3.1 获取ProcessState对象

#### 3.1.1 ProcessState::self()
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	// ProcessState单例
	sp<ProcessState> ProcessState::self()
	{
	    Mutex::Autolock _l(gProcessMutex);
	    if (gProcess != nullptr) {
	        return gProcess;
	    }
	    // 实例化ProcessState
	    gProcess = new ProcessState(kDefaultDriver);
	    return gProcess;
	}

	ProcessState::ProcessState(const char *driver)
	    : mDriverName(String8(driver))
	    , mDriverFD(open_driver(driver)) // 打开binder驱动,设置binder版本号和最大线程数
	    , mVMStart(MAP_FAILED)
	    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
	    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
		, mExecutingThreadsCount(0)
	    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
	    , mStarvationStartTimeMs(0)
	    , mManagesContexts(false)
	    , mBinderContextCheckFunc(nullptr)
	    , mBinderContextUserData(nullptr)
	    , mThreadPoolStarted(false)
	    , mThreadPoolSeq(1)
	    , mCallRestriction(CallRestriction::NONE)
	{
	    if (mDriverFD >= 0) {
	        // 执行binder mmap内存映射,分配一块虚拟内存,用来接收事务
	        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ,
			        MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
	    }
	}

#### 3.1.2 open_driver
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	static int open_driver(const char *driver)
	{
	    // 通过系统调用, 打开binder设备驱动
	    int fd = open(driver, O_RDWR | O_CLOEXEC);
	    if (fd >= 0) {
	        int vers = 0;
	        // 通过系统调用, 设置binder版本信息, 使用户空间和内核空间版本一致
	        status_t result = ioctl(fd, BINDER_VERSION, &vers);
	         
	        // 若用户空间和内核空间版本不一致, 则关闭binder驱动
	        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
	            close(fd);
	            fd = -1;
	        }
	 
	        // 设置binder最大线程数15
	        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
	        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
	    }
	    return fd;
	}

ioctl在讲解talkWithDriver时再详细分析.

### 3.2 获取BpBinder对象

#### 3.2.1 getContextObject
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
	{
	    // handle传0
	    return getStrongProxyForHandle(0);
	}

#### 3.2.2 getStrongProxyForHandle
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
	{
	    sp<IBinder> result;
	 
	    AutoMutex _l(mLock);
		// 查询是否包含handle=0的BpBinder对象
	    handle_entry* e = lookupHandleLocked(handle);
	 
	    if (e != nullptr) {
	        IBinder* b = e->binder;
	        // 若查询结果不存在handle=0的BpBinder对象, 则创建
	        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
	            if (handle == 0) {
	                Parcel data;
	                // 通过向binder驱动发送PING_TRANSACTION指令, 确认上下文管理器是否准备就绪.
	                // 具体参考【第3.4节】
	                status_t status = IPCThreadState::self()->transact(
	                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
	                if (status == DEAD_OBJECT)
	                    return NULL;
	            }
	            // 创建BpBinder对象, 并设置到handle_entry数据结构中
	            b = BpBinder::create(handle);
	            e->binder = b;
	            if (b) e->refs = b->getWeakRefs();
	            result = b;
	        } else {
		        // 否则直接返回查询结果
	            result.force_set(b);
	            e->refs->decWeak(this);
	        }
	    }
	 
	    return result;
	}

#### 3.2.3 lookupHandleLocked
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
	{
		// 获取顺序列表mHandleToObject的size
	    const size_t N=mHandleToObject.size();
		// 若mHandleToObject中数据 <= handle大小
	    if (N <= (size_t)handle) {
	        handle_entry e;
	        e.binder = nullptr;
	        e.refs = nullptr;
		    // 在索引N~handle范围依次插入空handle_entry对象,共handle+1-N个占位
		    status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
	        if (err < NO_ERROR) return nullptr;
	    }
	    // 返回索引为handle的对象
	    return &mHandleToObject.editItemAt(handle);
	}

#### 3.2.4 BpBinder::create
[ -> frameworks/native/libs/binder/BpBinder.cpp ]

	BpBinder* BpBinder::create(int32_t handle) {
	    return new BpBinder(handle, trackedUid);
	}

### 3.3 获取BinderProxy对象
#### 3.3.1 javaObjectForIBinder
[ -> frameworks/base/core/jni/android_util_Binder.cpp ]

	jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
	{
	    // 如果是Binder子类, 类型转换为JavaBBinder并返回
	    if (val->checkSubclass(&gBinderOffsets)) {
	        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
	        jobject object = static_cast<JavaBBinder*>(val.get())->object();
	        return object;
	    }
	 
	    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
	    nativeData->mOrgue = new DeathRecipientList;
	    nativeData->mObject = val;
	 
	    // 通过JNI调用BinderProxy.getInstance()获得BinderProxy对象
	    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
	            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());

	    // 获得BinderProxy.mNativeData
	    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
	    if (actualNativeData == nativeData) {
	        ...
	    } else {
	        delete nativeData;
	    }

		// 返回BinderProxy对象
	    return object;
	}

	BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
		// 通过JNI获得BinderProxy对象的属性mNativeData
	    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
	}
	
### 3.4 IPCThreadState
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

    // Special case for context manager...
    // The context manager is the only object for which we create
    // a BpBinder proxy without already holding a reference.
    // Perform a dummy transaction to ensure the context manager
    // is registered before we create the first local reference
    // to it (which will occur when creating the BpBinder).
    // If a local reference is created for the BpBinder when the
    // context manager is not present, the driver will fail to
    // provide a reference to the context manager, but the
    // driver API does not return status.
    //
    // Note that this is not race-free if the context manager
    // dies while this code runs.
    //
    // TODO: add a driver API to wait for context manager, or
    // stop special casing handle 0 for context manager and add
    // a driver API to get a handle to the context manager with
    // proper reference counting.
	status_t status = IPCThreadState::self()->transact(
	        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
根据注释可知, 在BpBinder对象创建之前, 需要先保证binder上下文管理器context manager准备就绪.
那么是怎么保证的呢?就是向binder驱动发送一条IBinder::PING_TRANSACTION指令, 看是否返回正常. 和长连接心跳检测机制有点类似.

调用链路:
IPCThreadState::transact ---> writeTransactionData -> waitForResponse -> talkWithDriver -> binder_ioctl -> binder_ioctl_write_read -> binder_thread_write -> binder_transaction

IPCThreadState::transact详细过程, 将会在后面binder驱动交互过程详细分析.

## 4.ServiceManagerNative.asInterface
[ -> frameworks/base/core/java/android/os/ServiceManagerNative.java ]
	
	static public IServiceManager asInterface(IBinder obj)
	{
	    if (obj == null) {
	        return null;
	    }
	    // 查询本地接口，返回结果null
	    IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
	    if (in != null) {
	        return in;
	    }
	 
		// 创建ServiceManagerProxy对象
	    return new ServiceManagerProxy(obj);
	}

### 4.1 queryLocalInterface
[ -> frameworks/base/core/java/android/os/BinderProxy.java ]

	public IInterface queryLocalInterface(String descriptor) {
	    return null;
	}

## 5.总结

ServiceManager获取过程分为2步:  
1.先判断是否有已创建的实例sServiceManager, 若有则直接返回, 否则新建一个  
2.创建ServiceManager实例过程:  
(1) BinderInternal.getContextObject  获取BinderProxy对象  

- `ProcessState::self()`  创建ProcessState对象
- `open_driver`  打开binder驱动,设置binder版本号和最大线程数15
- `getContextObject`  获取BpBinder对象.调用getStrongProxyForHandle(0). 首先通过lookupHandleLocked方法查询mHandleToObject顺序列表中是否存在handle=0的BpBinder, 若不存在则通过BpBinder::create创建. 当然在创建BpBinder之前需要保证上下文管理器context manager准备就绪, 具体方法就是向binder驱动发送IBinder::PING_TRANSACTION指令, 看返回结果是否正常.
- javaObjectForIBinder 获取BinderProxy对象. 通过JNI调用Java层的BinderProxy.getInstance()方法, 同时将上述得到的BpBinder对象赋值给BinderProxy.mNativeData.mObject属性.

(2) ServiceManagerNative.asInterface 获取ServiceManagerProxy对象  
先通过BinderProxy.queryLocalInterface查询本地是否包含ServiceManagerProxy对象, 结果返回null.  
new ServiceManagerProxy创建一个新对象.  
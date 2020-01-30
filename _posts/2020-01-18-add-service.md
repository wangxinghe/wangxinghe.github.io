---
layout: post
comments: true
title: "Service注册过程--Java服务"
description: "Service注册过程--Java服务"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析Java服务的注册流程.

## 0.文件结构

	frameworks/base/services/java/com/android/server/SystemServer.java
	 
	frameworks/base/core/java/android/os/ServiceManager.java
	frameworks/base/core/java/android/os/ServiceManagerNative.java
	frameworks/base/core/java/android/os/BinderProxy.java
	frameworks/base/core/java/android/os/Parcel.java
	 
	frameworks/base/core/jni/android_util_Binder.cpp
	frameworks/base/core/jni/android_os_Parcel.cpp
	 
	frameworks/native/libs/binder/IPCThreadState.cpp
	frameworks/native/libs/binder/BpBinder.cpp
	frameworks/native/libs/binder/Binder.cpp
	frameworks/native/libs/binder/Parcel.cpp
	frameworks/native/cmds/servicemanager/binder.c
	 
	kernel/msm-3.18/drivers/staging/android/binder.c
	kernel/msm-3.18/drivers/staging/android/binder_alloc.c

## 1.注册入口
[ -> frameworks/base/services/java/com/android/server/SystemServer.java]

	/**
	 * The main entry point from zygote.
	 */
	public static void main(String[] args) {
	    new SystemServer().run();
	}
	 
	private void run() {
	    startBootstrapServices();
	    startCoreServices();
	    startOtherServices();
	}

Java服务分为2种注册方式:  
(1) SystemServiceManager.startService().  对于继承于SystemService的服务如PWS  
其注册代码: `mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);`
(2) ServiceManager.addService().  其他服务如AMS

两种方式最终都是调用ServiceManager.addService.

以AMS服务注册为例进行分析.  
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java]

	public void setSystemProcess() {
	    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
	            DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
	}

[ -> frameworks/base/core/java/android/os/ServiceManager.java]

	/**
	 * Place a new @a service called @a name into the service
	 * manager.
	 *
	 * @param name the name of the new service
	 * @param service the service object
	 * @param allowIsolated set to true to allow isolated sandboxed processes
	 * @param dumpPriority supported dump priority levels as a bitmask
	 * to access this service
	 */
	public static void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority) {
	    getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
	}

AMS服务注册的入参为:  
服务名称 name=Context.ACTIVITY_SERVICE  
服务对象 service=AMS单例  
是否允许沙箱进程 allowIsolated=true  
dumpPriority=DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO  

前面文章 [**ServiceManager获取过程--Java层**](http://mouxuejie.com/blog/2020-01-05/get-service-manager-java/) 已经分析过getIServiceManager()拿到SMP对象. 最终SMP.addService入口如下:

[ -> frameworks/base/core/java/android/os/ServiceManagerNative.java]

	class ServiceManagerProxy implements IServiceManager {
	    public void addService(String name, IBinder service, boolean allowIsolated,
			    int dumpPriority) throws RemoteException {
	        Parcel data = Parcel.obtain();
	        Parcel reply = Parcel.obtain();
	        data.writeInterfaceToken(IServiceManager.descriptor);
	        data.writeString(name);
	        data.writeStrongBinder(service);
	        data.writeInt(allowIsolated ? 1 : 0);
	        data.writeInt(dumpPriority);
	        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
	        reply.recycle();
	        data.recycle();
	    }
	}

## 2.注册流程概览

### 2.1 Parcel封装Service信息  

**1. writeInterfaceToken**  
依次写入strictModePolicy / workSourceUid / IServiceManager.descriptor

**2. writeStrongBinder**  
(1) 将服务对象封装在`flat_binder_object`实体对象中(类型BINDER_TYPE_BINDER)  
(2) Parcel.writeObject(flat_binder_object)  

### 2.2 通信过程

分别从 **通信协议** 和 **通信流程** 角度图解 [ -> from gityuan.com]

![binder-protocol-flow](/image/2020-01-18-add-service/binder-protocol-flow.png)  

![binder-func-flow](/image/2020-01-18-add-service/binder-func-flow.png)  

`源线程`指发起注册请求的线程. `目标线程`指servicemanager进程的线程.

#### 2.2.1 BC_TRANSACTION过程 (源线程 -> Binder驱动)  

**1. IPCThreadState::writeTransactionData**  
(1) Parcel中数据写入`binder_transaction_data`对象buffer & offsets中  
(2) mOut数据: BC_TRANSACTION + binder_transaction_data对象  

**2. IPCThreadState::waitForResponse**  

**2.1 IPCThreadState::talkWithDriver**  
(1) 在用户空间封装`binder_write_read`对象bwr   
`bwr.write_buffer = (uintptr_t)mOut.data();`  
`bwr.read_buffer = (uintptr_t)mIn.data();`  
(2) 向Binder驱动写数据  
`ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)`  
ioctl -> binder_ioctl -> binder_ioctl_write_read  
(3) 拷贝binder_write_read对象bwr, 由用户空间->内核空间  
`copy_from_user(&bwr, ubuf, sizeof(bwr))`  

**2.2 binder_thread_write**  
(1) 拷贝BC_TRANSACTION, 由用户空间->内核空间  
`get_user(cmd, (uint32_t __user *)ptr)`  
(2) 拷贝binder_transaction_data对象到tr, 由用户空间->内核空间  
`copy_from_user(&tr, ptr, sizeof(tr))`  

**2.3 binder_transaction**  
(1) 创建`binder_transaction`事务t, 创建`binder_work`对象tcomplete  
(2) 将binder_transaction_data对象tr中的buffer/offsets拷贝到事务t的内核空间  
`copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size));`  
`copy_from_user(offp, (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size))`  
(3) 根据事务t的offsets依次读取buffer, 得到`flat_binder_object`对象(binder_object_header是其一属性), 读取`binder_object_header`对象的类型BINDER_TYPE_BINDER, 将类型改为BINDER_TYPE_HANDLE  
(4) 将t.work添加到目标线程或目标进程的同步或异步等待队列, 类型为BINDER_WORK_TRANSACTION  
(5) 将tcomplete添加到源线程的todo链表, type为BINDER_WORK_TRANSACTION_COMPLETE   
(6) 唤醒目标进程/目标线程的等待队列  

#### 2.2.2 BR_TRANSACTION_COMPLETE过程 (Binder驱动 -> 源线程)  

**1. binder_thread_read**  
(1) 从源线程的todo队列读取tcomplete, type为BINDER_WORK_TRANSACTION_COMPLETE  
(2) BR_TRANSACTION_COMPLETE返回用户空间  
`put_user(BR_TRANSACTION_COMPLETE, (uint32_t __user *)ptr)`  

#### 2.2.3 BR_TRANSACTION过程 (Binder驱动 -> ServiceManager进程)  

**1. binder_thread_read -> binder_transaction**  
(1) 从目标线程的todo队列读取t.work, type为BINDER_WORK_TRANSACTION  
(2) 根据t.work属性获得binder_transaction事务t  
(3) BR_TRANSACTION返回用户空间  
`put_user(BR_TRANSACTION, (uint32_t __user *)ptr)`  
(4) 创建`binder_transaction_data`对象tr, 其buffer/offsets指向事务t的内核缓冲区的buffer/offsets  
(5) 拷贝binder_transaction_data对象tr, 由内核空间->用户空间  
    `copy_to_user(ptr, &tr, sizeof(tr))`  
(6) 将事务t添加到目标线程的事务栈transaction_stack中  
 
**2. binder_parse**  
(1) 读取BR_TRANSACTION  
`uint32_t cmd = *(uint32_t *) ptr;`  
(2) 读取`binder_transaction_data`对象  
`memcpy(&txn.transaction_data, (void*) ptr, sizeof(struct binder_transaction_data));`  
(3) 将binder_transaction_data数据填充到`binder_io`对象msg中  
`bio_init_from_txn(&msg, &txn.transaction_data);`  
(4) 解析msg和txn, 将服务`svcinfo` si注册到svclist链表头部  
`svcmgr_handler -> do_add_service`  
(5) 发送BC_ACQUIRE命令, binder_ref强引用+1. 发送BC_REQUEST_DEATH_NOTIFICATION协议, 清理内存.  

#### 2.2.4 BC_FREE_BUFFER & BC_REPLY过程 (ServiceManager进程 -> Binder驱动)  

**1. binder_send_reply**  
(1) BC_FREE_BUFFER & BC_REPLY填充到data  

    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;

**2. binder_thread_write BC_FREE_BUFFER**  
释放buffer内核缓冲区内存, 及与buffer相关的binder_transaction事务

**3. binder_thread_write -> binder_transaction BC_REPLY**  
(1) 从目标线程的事务栈栈顶取出事务in_reply_to, 从中获得源进程/源线程(作为本次通信的目标进程/目标线程)  
(2) 创建`binder_transaction`事务t, 创建`binder_work`对象tcomplete, 根据in_reply_to设置事务t的目标进程/目标线程  
(3) 将`binder_transaction_data`对象中buffer/offsets拷贝到事务t的内核空间  
`copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size))`  
`copy_from_user(offp, (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size))`  
(4) 根据事务t的offsets依次读取buffer, 得到`flat_binder_object`对象(binder_object_header是其一属性), 读取`binder_object_header`对象的类型为BINDER_TYPE_HANDLE, binder_ref引用减1  
(5) 源线程(本次通信的目标线程)当前事务in_reply_to出栈  
(6) 将t.work添加到源线程(本次通信的目标线程)的同步或异步等待队列, 类型为BINDER_WORK_TRANSACTION  
(7) 将tcomplete添加到目标线程(本次通信的源线程)的todo链表, type为BINDER_WORK_TRANSACTION_COMPLETE   
(8) 唤醒源线程(本次通信的目标线程)的等待队列  
(9) 恢复servicemanager线程优先级  
(10) 释放事务in_reply_to所占用的内核缓冲区   

#### 2.2.5 BR_TRANSACTION_COMPLETE过程 (Binder驱动 -> ServiceManager进程)  

**1. binder_parse**  
什么都不做, 直接跳出循环.

#### 2.2.6 BR_REPLY过程 (Binder驱动 -> 源线程)  

**1. binder_thread_read**  
(1) 从源线程(本次通信的目标线程)的todo队列取出`binder_work`对象w(BC_REPLY过程放入)  
(2) 根据w属性获得`binder_transaction`事务t  
(3) 向源线程(本次通信的目标线程)的用户空间发送BR_REPLY返回协议  
(4) 将`binder_transaction_data`对象tr拷贝到源线程(本次通信的目标线程)提供的用户空间   
(5) 设置事务t的内核缓冲区标志位allow_user_free, 表示内存可释放, 释放事务t内存空间(此时内核缓冲区buffer还没有释放)  

## 3.Parcel封装Service信息
[ -> frameworks/base/core/java/android/os/ServiceManagerNative.java]

	Parcel data = Parcel.obtain();
	// 请求头 "android.os.IServiceManager"
	data.writeInterfaceToken(IServiceManager.descriptor);
	// 服务名称
	data.writeString(name);
	// 服务对象
	data.writeStrongBinder(service);
	// 是否允许沙箱进程
	data.writeInt(allowIsolated ? 1 : 0);
	// 支持转储优先级作为位掩码
	data.writeInt(dumpPriority);

我们重点分析:  
`data.writeInterfaceToken(IServiceManager.descriptor)`和`data.writeStrongBinder(service);`

### 3.1 Parcel.writeInterfaceToken  
[ -> frameworks/native/libs/binder/Parcel.cpp ]

	// Write RPC headers.  (previously just the interface token)
	status_t Parcel::writeInterfaceToken(const String16& interface)
	{
	    const IPCThreadState* threadState = IPCThreadState::self();
	    writeInt32(threadState->getStrictModePolicy() | STRICT_MODE_PENALTY_GATHER);
	    updateWorkSourceRequestHeaderPosition();
	    writeInt32(threadState->shouldPropagateWorkSource() ?
	            threadState->getCallingWorkSourceUid() : IPCThreadState::kUnsetWorkSource);
	    // currently the interface identification token is just its name as a string
	    return writeString16(interface);
	}

### 3.2 Parcel.writeStrongBinder
[ -> frameworks/base/core/java/android/os/Parcel.java]

	public final void writeStrongBinder(IBinder val) {
	    nativeWriteStrongBinder(mNativePtr, val);
	}
 
	private static native void nativeWriteStrongBinder(long nativePtr, IBinder val);

[ -> frameworks/base/core/jni/android_os_Parcel.cpp]

	static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz,
			jlong nativePtr, jobject object)
	{
	    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
	    if (parcel != NULL) {
	        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
	        if (err != NO_ERROR) {
	            signalExceptionForError(env, clazz, err);
	        }
	    }
	}

#### 3.2.1 ibinderForJavaObject
[ -> frameworks/base/core/jni/android_util_Binder.cpp]

	sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
	{
	    if (obj == NULL) return NULL;

	    // 若obj为Binder对象
	    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
	        // 得到JavaBBinderHolder对象
	        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
	            env->GetLongField(obj, gBinderOffsets.mObject);
			// 得到JavaBBinder对象
	        return jbh->get(env, obj);
	    }
	 
	    // 若obj为BinderProxy对象
	    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
	        // 先得到BinderProxyNativeData对象, 再取mObject得到BpBinder对象
	        return getBPNativeData(env, obj)->mObject;
	    }
	 
	    return NULL;
	}

	// 得到BinderProxyNativeData对象
	BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
	    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
	}

#### 3.2.2 writeStrongBinder
[ -> frameworks/native/libs/binder/Parcel.cpp]

	status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
	{
	    return flatten_binder(ProcessState::self(), val, this);
	}

#### 3.2.3 flatten_binder
[ -> frameworks/native/libs/binder/Parcel.cpp]

	status_t flatten_binder(const sp<ProcessState>& /*proc*/,
		    const sp<IBinder>& binder, Parcel* out)
	{
	    flat_binder_object obj;
	  
	    if (IPCThreadState::self()->backgroundSchedulingDisabled()) {
	        /* minimum priority for all nodes is nice 0 */
	        obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
	    } else {
	        /* minimum priority for all nodes is MAX_NICE(19) */
	        // 服务最小线程优先级19, FLAT_BINDER_FLAG_ACCEPTS_FDS表示可以传输文件描述符信息
	        obj.flags = 0x13 | FLAT_BINDER_FLAG_ACCEPTS_FDS;
	    }

	    if (binder != nullptr) {
	        // 得到JavaBBinder对象, 具体参考Binder.cpp中BBinder::localBinder()的实现
	        BBinder *local = binder->localBinder();
	        if (!local) {
	            BpBinder *proxy = binder->remoteBinder();
	            const int32_t handle = proxy ? proxy->handle() : 0;
	            obj.hdr.type = BINDER_TYPE_HANDLE;
	            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
	            obj.handle = handle;
	            obj.cookie = 0;
	        } else {
	            if (local->isRequestingSid()) {
	                obj.flags |= FLAT_BINDER_FLAG_TXN_SECURITY_CTX;
	            }
	            // 设置type为Binder实体类型BINDER_TYPE_BINDER
	            obj.hdr.type = BINDER_TYPE_BINDER;
	            // 设置JavaBBinder本地对象的内部弱引用对象的地址值
	            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
	            // 设置JavaBBinder本地对象的地址值
	            obj.cookie = reinterpret_cast<uintptr_t>(local);
	        }
	    } else {
	        obj.hdr.type = BINDER_TYPE_BINDER;
	        obj.binder = 0;
	        obj.cookie = 0;
	    }
	  
	    return finish_flatten_binder(binder, obj, out);
	}

	inline static status_t finish_flatten_binder(
	    const sp<IBinder>& /*binder*/, const flat_binder_object& flat, Parcel* out)
	{
	    return out->writeObject(flat, false);
	}

#### 3.2.4 Parcel.writeObject
分析这个方法之前, 我们先了解下Parcel类的几个属性的含义.  
**mData**: 数据缓冲区, 可以是int/String/flat_binder_object类型的数据  
**mDataPos**: 数据缓冲区mData中当前可以写入数据的位置  
**mDataCapacity**: 数据缓冲区mData数组的容量  
**mObjects**: 偏移数组. 值为mData中所有flat_binder_object类型数据的位置  
**mObjectsSize**: 偏移数组mObjects中当前可以写入数据的位置. 即偏移数组mObjects当前大小  
**mObjectsCapacity**: 偏移数组mObjects的容量  

[ -> frameworks/native/libs/binder/Parcel.cpp]

	status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData)
	{
		// 数据缓冲区是否有足够空间
	    const bool enoughData = (mDataPos+sizeof(val)) <= mDataCapacity;
	    // 偏移数组是否有足够空间
	    const bool enoughObjects = mObjectsSize < mObjectsCapacity;
	    if (enoughData && enoughObjects) { // 若数据缓冲区和偏移数组都有足够空间
	restart_write:
	        // 将flat_binder_object对象写入数据缓冲区mData中以mDataPos开始的一段内存
	        *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val;

	        // remember if it's a file descriptor
	        if (val.hdr.type == BINDER_TYPE_FD) {
	            if (!mAllowFds) {
	                // fail before modifying our object index
	                return FDS_NOT_ALLOWED;
	            }
	            mHasFds = mFdsKnown = true;
	        }
	 
	        // Need to write meta-data?
	        if (nullMetaData || val.binder != 0) {
		        // mObjects值记录flat_binder_object类型的数据在数据缓冲区mData中的起始位置
	            mObjects[mObjectsSize] = mDataPos;
	            acquire_object(ProcessState::self(), val, this, &mOpenAshmemSize);
	            // 位置+1
	            mObjectsSize++;
	        }
	 
	        // 调整mDataPos的值
	        return finishWrite(sizeof(flat_binder_object));
	    }

		// 若数据缓冲区mData空间不够, 则扩容	 
	    if (!enoughData) {
	        const status_t err = growData(sizeof(val));
	        if (err != NO_ERROR) return err;
	    }

		// 若偏移数组mObjects空间不够, 则扩容	 
	    if (!enoughObjects) {
	        size_t newSize = ((mObjectsSize+2)*3)/2;
	        if (newSize*sizeof(binder_size_t) < mObjectsSize) return NO_MEMORY;   // overflow
	        binder_size_t* objects = (binder_size_t*)realloc(mObjects, 
			        newSize*sizeof(binder_size_t));
	        if (objects == nullptr) return NO_MEMORY;
	        mObjects = objects;
	        mObjectsCapacity = newSize;
	    }

	    goto restart_write;
	}

	status_t Parcel::finishWrite(size_t len)
	{
		// 调整mDataPos的值
	    mDataPos += len;
	    if (mDataPos > mDataSize) {
	        mDataSize = mDataPos;
	    }
	    return NO_ERROR;
	}

因此, 将Parcel中保存struct flat_binder_object类型的数据的方式就是:  
将struct flat_binder_object类型的数据写入数据缓冲区mData  
将struct flat_binder_object类型的数据的存储起始位置保存到偏移数组mObjects  

## 4.Service注册过程  

mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);

### 4.1 BinderProxy.transact
各参数含义如下:  
**code**: 表示要干啥事, 此处为ADD_SERVICE_TRANSACTION表示注册服务  
**data**: 发送给Binder驱动的通信数据, 此处为要注册的Service信息  
**reply**: 进程间通信结果  
**flags**: 通信过程是同步/异步, 此处为0表示同步    

关于同步异步:  
TF_ONE_WAY	= 0x01,	/* this is a one-way call: async, no return */ 表示异步, 无返回值

[ -> frameworks/base/core/java/android/os/BinderProxy.java]

	public boolean transact(int code, Parcel data, Parcel reply, int flags)
				throws RemoteException {
	    return transactNative(code, data, reply, flags);
	}

	public native boolean transactNative(int code, Parcel data, Parcel reply, int flags)
			throws RemoteException;

[ -> frameworks/base/core/jni/android_util_Binder.cpp]

	static const JNINativeMethod gBinderProxyMethods[] = {
	     /* name, signature, funcPtr */
	    {"transactNative", "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z",
			    (void*)android_os_BinderProxy_transact},
	};

	static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
			jint code, jobject dataObj, jobject replyObj, jint flags)
	{
	    // 根据Java层的Parcel对象, 拿到Native层的Parcel对象
	    Parcel* data = parcelForJavaObject(env, dataObj);
	    Parcel* reply = parcelForJavaObject(env, replyObj);
	    // 拿到BpBinder对象
	    IBinder* target = getBPNativeData(env, obj)->mObject.get();
	    status_t err = target->transact(code, *data, reply, flags);
	  
	    if (err == NO_ERROR) {
	        return JNI_TRUE;
	    } else if (err == UNKNOWN_TRANSACTION) {
	        return JNI_FALSE;
	    }
	  
	    return JNI_FALSE;
	}

	BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
	    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
	}

### 4.2 BpBinder.transact
[ -> frameworks/native/libs/binder/BpBinder.cpp]

	status_t BpBinder::transact(uint32_t code, const Parcel& data,
			Parcel* reply, uint32_t flags)
	{
	    // mHandle=0, 即目标进程是servicemanager
	    status_t status = IPCThreadState::self()->transact(
	            mHandle, code, data, reply, flags);
	    return status;
	}

### 4.3 IPCThreadState.transact
[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	IPCThreadState* IPCThreadState::self()
	{
	    if (gHaveTLS) {
	restart:
	        const pthread_key_t k = gTLS;
	        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
	        if (st) return st;
	        return new IPCThreadState;
	    }
	  
	    pthread_mutex_lock(&gTLSMutex);
	    if (!gHaveTLS) {
	        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
	        gHaveTLS = true;
	    }
	    pthread_mutex_unlock(&gTLSMutex);
	    goto restart;
	}

	status_t IPCThreadState::transact(int32_t handle, uint32_t code, const Parcel& data,
			Parcel* reply, uint32_t flags)
	{
	    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
	    err = waitForResponse(reply);
	    return err;
	}

## 5.BC_TRANSACTION过程 (源线程 -> Binder驱动)

本小节分析**源线程向Binder驱动发起BC_TRANSACTION事务过程**.

### 5.1 IPCThreadState.writeTransactionData  
此处cmd为BC_TRANSACTION, 表示发起事务过程.

[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	// 封装binder_transaction_data数据结构, 并将相关数据写入mOut
	status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
		    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
	{
	    // 构造并初始化struct binder_transaction_data类型的对象
	    binder_transaction_data tr;
	    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
	    // 设置目标进程的句柄
	    tr.target.handle = handle;
	    // 设置code, 表示要干啥, 此处为ADD_SERVICE_TRANSACTION
	    tr.code = code;
	    // 设置通信过程同步/异步
	    tr.flags = binderFlags;
	    tr.cookie = 0;
	    tr.sender_pid = 0;
	    tr.sender_euid = 0;
	 
	    const status_t err = data.errorCheck();
	    if (err == NO_ERROR) {
	        // 设置要传输的数据缓冲区mData的大小
	        tr.data_size = data.ipcDataSize();
	        // 设置要传输的数据缓冲区mData的数据
	        tr.data.ptr.buffer = data.ipcData();
	        // 设置偏移数组的大小
	        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
	        // 设置偏移数组的数据
	        tr.data.ptr.offsets = data.ipcObjects();
	    } else if (statusBuffer) {
	        ...
	    } else {
	        ...
	    }

	    // 向输出通道mOut中写入BC码, 此处为BC_TRANSACTION
	    mOut.writeInt32(cmd);
	    // 向输出通道mOut中写入struct binder_transaction_data类型的数据
	    mOut.write(&tr, sizeof(tr));

	    return NO_ERROR;
	}

因此输出通道mOut中数据格式是:  
`命令协议 + binder_transaction_data结构体数据`

### 5.2 IPCThreadState.waitForResponse
[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
	{
	    while(1) {
		    // 驱动交互过程.见【第5.3节】
	        if ((err=talkWithDriver()) < NO_ERROR) break;
	        
	        cmd = (uint32_t)mIn.readInt32();
	         
	        switch (cmd) {
	        case BR_TRANSACTION_COMPLETE:
	            // acquireResult默认为true， 因此跳出switch，执行while循环
	            if (!reply && !acquireResult) goto finish;
	            break;
	        case BR_DEAD_REPLY: ...
	        case BR_FAILED_REPLY: ...
	        case BR_ACQUIRE_RESULT: ...
	        case BR_REPLY:
	            {
	                binder_transaction_data tr;
	                err = mIn.read(&tr, sizeof(tr));
	 
	                if (reply) {
	                    if ((tr.flags & TF_STATUS_CODE) == 0) {
	                        reply->ipcSetDataReference(
	                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
	                            tr.data_size,
	                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
	                            tr.offsets_size/sizeof(binder_size_t),
	                            freeBuffer, this);
	                    } else {
	                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
	                        freeBuffer(nullptr,
	                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
	                            tr.data_size,
	                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
	                            tr.offsets_size/sizeof(binder_size_t), this);
	                    }
	                } else {
	                    freeBuffer(nullptr,
	                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
	                        tr.data_size,
	                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
	                        tr.offsets_size/sizeof(binder_size_t), this);
	                    continue;
	                }
	            }
	            goto finish;
	        default: // 其他BR码
	            err = executeCommand(cmd);
	            if (err != NO_ERROR) goto finish;
	            break;
	        }
	    }
	 
	finish:
	    if (err != NO_ERROR) {
	        if (acquireResult) *acquireResult = err;
	        if (reply) reply->setError(err);
	        mLastError = err;
	    }
	 
	    return err;
	}

### 5.3 IPCThreadState.talkWithDriver
此处由于doReceive没有传值, C++中bool默认值为true, 因此doReceive默认为true.

[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	status_t IPCThreadState::talkWithDriver(bool doReceive)
	{
	    binder_write_read bwr;
	    // Is the read buffer empty?
	    // 输入缓冲区mIn中数据已经全部处理完时, 才需要读取, 即needRead为true
	    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

	    // We don't want to write anything if we are still reading
	    // from data left in the input buffer and the caller
	    // has requested to read the next data.
	    // 此处doReceive为true.
	    // 若needRead为true, 即mIn数据已全部处理完时, 此时才可以写数据, 设置outAvail为mOut.dataSize()
	    // 若needRead为false, 即mIn中数据还没处理完, 则优先处理读数据, 暂时不写数据, 设置outAvail为0
	    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

		// 设置输出缓冲区大小
	    bwr.write_size = outAvail;
	    // 设置输出缓冲区的数据
	    bwr.write_buffer = (uintptr_t)mOut.data();

	    // This is what we'll read.
	    if (doReceive && needRead) { // 此处doReceive为true, 若needRead为true, 即mIn数据已全部处理完
		    // 设置输入缓冲区大小为mIn的容量
	        bwr.read_size = mIn.dataCapacity();
	        // 设置输入缓冲区的数据
	        bwr.read_buffer = (uintptr_t)mIn.data();
	    } else { // 否则若doReceive或needRead为false, 则不读数据
	        bwr.read_size = 0;
	        bwr.read_buffer = 0;
	    }
	 
	    // Return immediately if there is nothing to do.
	    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
	 
	    bwr.write_consumed = 0;
	    bwr.read_consumed = 0;
	    status_t err;
	    do {
	        // 通过系统调用, 执行Binder驱动数据的读写操作. 参考【第5.4节】
	        // 此处的struct binder_write_read数据属于用户空间
	        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
	            err = NO_ERROR;
	        else
	            err = -errno;
	    } while (err == -EINTR);
	      
	    if (err >= NO_ERROR) {
	        // 清空mOut中已经写入到Binder驱动的数据
	        if (bwr.write_consumed > 0) {
	            if (bwr.write_consumed < mOut.dataSize())
	                mOut.remove(0, bwr.write_consumed);
	            else {
	                mOut.setDataSize(0);
	                processPostWriteDerefs();
	            }
	        }
	        // 从Binder驱动已经读取的数据设置到mIn中
	        if (bwr.read_consumed > 0) {
	            mIn.setDataSize(bwr.read_consumed);
	            mIn.setDataPosition(0);
	        }
	        return NO_ERROR;
	    }
	    return err;
	}

上面的过程做的事情是:  
(1) 封装struct binder_write_read类型数据, 并设置用于读/写的数据缓冲区.  

	bwr.write_buffer = (uintptr_t)mOut.data();
	bwr.read_buffer = (uintptr_t)mIn.data();

(2) 通过ioctl方法和Binder驱动交互执行数据读/写操作.  

	ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)

(3) 读/写过程完成后, 修改mOut和mIn中发生改变的属性.    

对于注册服务的BC_TRANSACTION过程, 只有写数据.

### 5.4 binder_ioctl
这里分析ioctl系统调用:  ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  
最终会调用到Binder驱动的binder_ioctl方法.  

[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
	{
		struct binder_proc *proc = filp->private_data;
		struct binder_thread *thread;

		// 获取线程
		thread = binder_get_thread(proc);

	    switch (cmd) {
	    case BINDER_WRITE_READ:// binder读写操作
	        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
	        break;
	    case BINDER_SET_MAX_THREADS: ...
	    case BINDER_SET_CONTEXT_MGR: ...
	    case BINDER_THREAD_EXIT: ...
	    case BINDER_VERSION: ...
	    case BINDER_GET_NODE_DEBUG_INFO: ...
	    }
	      
	    return ret;
	}

### 5.5 binder_ioctl_write_read
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int binder_ioctl_write_read(struct file *filp,
			unsigned int cmd, unsigned long arg, struct binder_thread *thread)
	{
	    int ret = 0;
	    // 当前进程
	    struct binder_proc *proc = filp->private_data;
	    unsigned int size = _IOC_SIZE(cmd);
	    void __user *ubuf = (void __user *)arg;
	    // 此处的struct binder_write_read数据属于内核空间
	    struct binder_write_read bwr;
	  
	    // 将传输过来的用户空间struct binder_write_read数据拷贝到内核空间bwr
	    copy_from_user(&bwr, ubuf, sizeof(bwr));
	   
	    // 当写缓存中有数据时, 执行binder写操作
	    if (bwr.write_size > 0) {
	        ret = binder_thread_write(proc, thread,
	                      bwr.write_buffer,
	                      bwr.write_size,
	                      &bwr.write_consumed);
	    }
	    // 当读缓存中有数据时, 执行binder读操作
	    if (bwr.read_size > 0) {
	        ret = binder_thread_read(proc, thread, bwr.read_buffer,
	                     bwr.read_size,
	                     &bwr.read_consumed,
	                     filp->f_flags & O_NONBLOCK);
	        binder_inner_proc_lock(proc);
	        // 如果当前进程的todo队列非空, 则唤醒当前进程中等待状态的线程来处理todo任务
	        if (!binder_worklist_empty_ilocked(&proc->todo))
	            binder_wakeup_proc_ilocked(proc);
	        binder_inner_proc_unlock(proc);
	    }

	    // Binder读/写完后, 将内核空间数据bwr拷贝到传输进来的用户空间struct binder_write_read
	    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
	        ret = -EFAULT;
	        goto out;
	    }
	out:
	    return ret;
	}

从上面代码可知, struct binder_write_read是用于空间和内核空间的数据传输格式. 可同时用于用户空间和内核空间.

对于BC_TRANSACTION过程, 只有写操作, 因此分析binder_thread_write方法.

### 5.6 binder_thread_write
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int binder_thread_write(struct binder_proc *proc,
	            struct binder_thread *thread,
	            binder_uintptr_t binder_buffer, size_t size,
	            binder_size_t *consumed)
	{
	    uint32_t cmd;
	    void __user *ptr = buffer + *consumed;
	    void __user *end = buffer + size;
	  
	    while (ptr < end && thread->return_error.cmd == BR_OK) {
	        int ret;
	          
	        // 从用户空间读取数据并保存到cmd中. 根据mOut数据格式可知, 此处获取的是BC_TRANSACTION
	        if (get_user(cmd, (uint32_t __user *)ptr))
	            return -EFAULT;
	        ptr += sizeof(uint32_t);
	        switch (cmd) {
	        case BC_INCREFS: ...
	        case BC_ACQUIRE: ...
	        case BC_RELEASE: ...
	        case BC_DECREFS: ...
	        case BC_INCREFS_DONE: ...
	        case BC_ACQUIRE_DONE: ...
	        case BC_ATTEMPT_ACQUIRE: ...
	        case BC_ACQUIRE_RESULT: ...
	        case BC_FREE_BUFFER: ...
	        case BC_TRANSACTION_SG: ...
	        case BC_REPLY_SG: ...
	        case BC_TRANSACTION:
	        case BC_REPLY:{
	            // 从用户空间拷贝数据到内核空间struct binder_transaction_data类型的数据tr中
	            // 根据mOut数据格式可知, 此处获取的是struct binder_transaction_data类型的数据
	            struct binder_transaction_data tr;
	            if (copy_from_user(&tr, ptr, sizeof(tr)))
	                return -EFAULT;
	            // 移动指针位置
	            ptr += sizeof(tr);

	            // binder事务处理
	            // 对于BC_TRANSACTION,在目标进程的内核空间分配一块缓冲区创建事务t,并设置目标进程/目标线程等
	            // 在内核空间创建binder_work类型对象tcomplete, 添加到源线程的todo队列
	            // 将事务t的binder_work类型对象work, 添加到目标线程/目标进程的todo队列.
	            // 唤醒目的进程/目的线程的等待队列
	            binder_transaction(proc, thread, &tr, cmd == BC_REPLY, 0);
	            break;
	        }
	        case BC_REGISTER_LOOPER: ...
	        case BC_ENTER_LOOPER: ...
	        case BC_EXIT_LOOPER: ...
	        case BC_REQUEST_DEATH_NOTIFICATION: ...
	        case BC_CLEAR_DEATH_NOTIFICATION: ...
	        case BC_DEAD_BINDER_DONE: ...
	        }

	        *consumed = ptr - buffer;
	    }
	    return 0;
	}

### 5.7 binder_transaction
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static void binder_transaction(struct binder_proc *proc,
	                   struct binder_thread *thread,
	                   struct binder_transaction_data *tr, int reply,
	                   binder_size_t extra_buffers_size)
	{
	    struct binder_transaction *t; // binder事务
	    struct binder_work *tcomplete;
	    struct binder_proc *target_proc = NULL; // 目标进程
	    struct binder_thread *target_thread = NULL; // 目标线程
	    struct binder_node *target_node = NULL; // 目标binder节点
	    struct binder_transaction *in_reply_to = NULL; // binder事务
	  
	    if (reply) {
			...
	    } else { // 此处传输指令为BC_TRANSACTION, reply为false
	        if (tr->target.handle) { // 若目标进程句柄非0, 即目标进程为普通服务进程
	            struct binder_ref *ref;
	            // 根据句柄handle查找红黑树中binder引用binder_ref.见【第5.7.1节】
	            ref = binder_get_ref_olocked(proc, tr->target.handle, true);
	            if (ref)
	                // 设置目标节点和目标进程, 且目标节点本地强引用和临时强引用+1，目标进程临时引用+1.
		            target_node = binder_get_node_refs_for_txn(ref->node, &target_proc,
				            &return_error);
	        } else { // 若目标进程句柄为0, 即目标进程为servicemanager进程
	            // 设置目标节点, 即binder_context_mgr_node
	            target_node = context->binder_context_mgr_node;
	            if (target_node)
		            // 设置目标进程, 且目标节点本地强引用和临时强引用+1，目标进程临时引用+1.见【第5.7.2节】
		            target_node = binder_get_node_refs_for_txn(target_node, &target_proc,
				            &return_error);
	        }
	  
	        // 对于同步通信方式即非oneway, 若当前线程事务栈非空, 则需要找到最优目标线程target_thread
	        // 对于异步通信方式即oneway, 不存在一个线程等待另一个线程的情况, 因此不存在寻找最优目标线程
			// 此处涉及到提高线程利用率, 具体原理见【第5.7.3节】
	        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
	            struct binder_transaction *tmp = thread->transaction_stack;
	            while (tmp) {
	                struct binder_thread *from = tmp->from;
	                if (from && from->proc == target_proc) {
	                    atomic_inc(&from->tmp_ref);
	                    target_thread = from;
	                    break;
	                }
	                tmp = tmp->from_parent;
	            }
	        }
	    }
	    // 分配struct binder_transaction类型的数据t
	    t = kzalloc(sizeof(*t), GFP_KERNEL);
	    // 分配struct binder_work类型的数据tcomplete
	    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

	    // 如果当前处理的是BC_TRANSACTION协议,且是同步通信方式, 则设置事务t的源线程为当前进程
	    // 便于目标进程或线程在处理完进程间通信事务后, 找到源线程, 并将结果返回给源线程
	    if (!reply && !(tr->flags & TF_ONE_WAY))
	        t->from = thread;
	    else
	        t->from = NULL;
	    // 设置uid
	    t->sender_euid = task_euid(proc->tsk);
	    // 设置目标进程
	    t->to_proc = target_proc;
	    // 设置目标线程
	    t->to_thread = target_thread;
	    // 设置code
	    t->code = tr->code;
	    // 设置通信方式
	    t->flags = tr->flags;
	    // 设置优先级
	    if (!(t->flags & TF_ONE_WAY) && binder_supported_policy(current->policy)) {
	        /* Inherit supported policies for synchronous transactions */
	        t->priority.sched_policy = current->policy;
	        t->priority.prio = current->normal_prio;
	    } else {
	        /* Otherwise, fall back to the default priority */
	        t->priority = target_proc->default_priority;
	    }
	    // 从目标进程分配一个内核缓冲区给事务t
	    t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size, tr->offsets_size, extra_buffers_size, !reply && (t->flags & TF_ONE_WAY));
	    t->buffer->debug_id = t->debug_id;
	    t->buffer->transaction = t;
	    t->buffer->target_node = target_node;

	    // 设置事务t中偏移数组的开始位置off_start, 即当前位置+binder_transaction_data数据大小
	    off_start = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));
	    // offp指向偏移数组的起始位置
	    offp = off_start;
	    // 将用户空间中binder_transaction_data类型数据tr的数据缓冲区, 拷贝到内核空间中事务t的内核缓冲区
	    copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)tr->data.ptr.buffer,
			    tr->data_size))
	    // 将用户空间中binder_transaction_data类型数据tr的偏移数组, 拷贝到内核空间中事务t的偏移数组中
	    // offp为事务t中偏移数组的起始位置
	    copy_from_user(offp, (const void __user *)(uintptr_t)tr->data.ptr.offsets,
			    tr->offsets_size))
	    // 设置事务t中偏移数组的结束位置off_end
	    off_end = (void *)off_start + tr->offsets_size;

	    for (; offp < off_end; offp++) { // 遍历偏移数组区间
	        // 从事务t的数据缓冲区中, 获取offp位置的binder_object_header对象
	        struct binder_object_header *hdr = (struct binder_object_header *)
			        (t->buffer->data + *offp);
	        switch (hdr->type) {
	        case BINDER_TYPE_BINDER: // 根据writeStrongBinder可知, type为BINDER_TYPE_BINDER
	        case BINDER_TYPE_WEAK_BINDER: {
	            struct flat_binder_object *fp;
	            // 根据hdr属性获取flat_binder_object对象的值.见【第5.7.4节】
	            fp = to_flat_binder_object(hdr);
	            // 从源进程的节点红黑树中, 根据flat_binder_object对象的binder属性查找Binder节点
	            // 若没有则创建一个Binder节点, 同时目标进程中创建binder引用且binder引用+1
	            // 同时修改flat_binder_object类型为引用类型BINDER_TYPE_HANDLE. 见【第5.7.5节】
	            ret = binder_translate_binder(fp, t, thread);
	        } break;
	        case BINDER_TYPE_HANDLE: ...
	        case BINDER_TYPE_WEAK_HANDLE: ...
	        case BINDER_TYPE_FD: ...
	        case BINDER_TYPE_FDA: ...
	        case BINDER_TYPE_PTR: ...
	        default: ...
	        }
	    }
	    // 设置事务tcomplete的类型
	    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	    // 设置事务t的类型
	    t->work.type = BINDER_WORK_TRANSACTION;

	    if (reply) {
	        ...
	    } else if (!(t->flags & TF_ONE_WAY)) { // 此处为同步通信，即非oneway
	        binder_inner_proc_lock(proc);
	        /*
	         * Defer the TRANSACTION_COMPLETE, so we don't return to
	         * userspace immediately; this allows the target process to
	         * immediately start processing this transaction, reducing
	         * latency. We will then return the TRANSACTION_COMPLETE when
	         * the target replies (or there is an error).
	         */
	        // 将事务tcomplete添加到源线程的todo链表
	        binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
	        // 设置事务t需要等待回复
	        t->need_reply = 1;
	        // 将事务t压入源线程thread的事务堆栈中，即将事务t放到事务栈栈顶
	        t->from_parent = thread->transaction_stack;
	        thread->transaction_stack = t;
	        binder_inner_proc_unlock(proc);

	        // 将事务t的work添加到目标线程或目标线程的同步或异步等待队列, 并唤醒目标线程的等待队列.
	        // 见【第5.7.6节】
	        binder_proc_transaction(t, target_proc, target_thread);
	    } else { // 若为异步通信，即oneway
	        // 将事务tcomplete添加到源线程的todo链表
	        binder_enqueue_thread_work(thread, tcomplete);
	        // 将事务t的work添加到目标线程或目标线程的同步或异步等待队列, 并唤醒目标线程的等待队列
	        binder_proc_transaction(t, target_proc, NULL);
	    }
	}

总结：  
(1) 对于注册服务的BC_TRANSACTION过程, 先确定目标进程和目标线程, 目标进程为servicemanager进程.  
(2) 创建事务t, 设置事务t的源线程/目标进程/目标线程/通信方式/code等等属性.   
(3) 从目标进程的内核空间分配一块缓冲区给事务t, 将用户空间中binder_transaction_data的数据缓冲区和偏移数组拷贝到内核空间中事务t的数据缓冲区和偏移数组  
(4) 遍历事务t的偏移数组, 从数据缓冲区依次取出binder_object_header对象hdr, 其type为BINDER_TYPE_BINDER.  
(5) 根据hdr属性获得flat_binder_object对象.  
(6) 在源进程的节点红黑树中, 根据flat_binder_object对象的binder属性查找Binder节点, 若没有则创建一个Binder节点. 同时目标进程中创建Binder引用, 引用+1.  
(7) 修改flat_binder_object类型为引用类型BINDER_TYPE_HANDLE  
(8) 在内核空间创建struct binder_work类型的对象`tcomplete`, type为BINDER_WORK_TRANSACTION_COMPLETE. 将tcomplete添加到`源线程的todo链表`  
(9) 设置事务t的binder_work类型对象work的类型为BINDER_WORK_TRANSACTION  
(10) 若事务t为同步通信, 则设置事务t的need_reply为1, 并将事务t压入源线程的事务堆栈的栈顶  
(11) 将事务t的binder_work类型对象`work`添加到`目标线程或目标进程的同步或异步等待队列`, 并唤醒目标线程的等待队列  

之后源线程/目标进程/目标线程就会并发地处理各自的todo队列中的工作项.

#### 5.7.1 binder_get_ref_olocked
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static struct binder_ref *binder_get_ref_olocked(struct binder_proc *proc,
							 u32 desc, bool need_strong_ref)
	{
		struct rb_node *n = proc->refs_by_desc.rb_node;
		struct binder_ref *ref;
	
		while (n) {
			ref = rb_entry(n, struct binder_ref, rb_node_desc);
	
			if (desc < ref->data.desc) {
				n = n->rb_left;
			} else if (desc > ref->data.desc) {
				n = n->rb_right;
			} else if (need_strong_ref && !ref->data.strong) {
				binder_user_error("tried to use weak ref as strong ref\n");
				return NULL;
			} else {
				return ref;
			}
		}
		return NULL;
	}

上面代码做的工作是根据desc(即handle)查找binder_ref，具体过程：  
根据 binder_proc 获取 refs_by_desc 红黑树  
遍历 refs_by_desc 红黑树，拿到每个节点对应的binder_ref  
比较要查找的desc和binder_ref中的desc的大小，从而匹配到目标binder_ref  

#### 5.7.2 binder_get_node_refs_for_txn
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

    // 设置目标节点和目标进程, 且引用+1
	static struct binder_node *binder_get_node_refs_for_txn(
			struct binder_node *node,
			struct binder_proc **procp,
			uint32_t *error)
	{
		struct binder_node *target_node = NULL;
	
		binder_node_inner_lock(node);
		if (node->proc) {
			target_node = node;
			// 目标节点本地强引用+1
			binder_inc_node_nilocked(node, 1, 0, NULL);
			// 目标节点临时引用+1
			binder_inc_node_tmpref_ilocked(node);
			// 目标进程临时引用+1
			node->proc->tmp_ref++;
			*procp = node->proc;
		} else
			*error = BR_DEAD_REPLY;
		binder_node_inner_unlock(node);
	
		return target_node;
	}
	
	static int binder_inc_node_nilocked(struct binder_node *node, int strong,
					    int internal, struct list_head *target_list)
	{
		struct binder_proc *proc = node->proc;
	
		assert_spin_locked(&node->lock);
		if (proc)
			assert_spin_locked(&proc->inner_lock);
		if (strong) {
			if (internal) {
				if (target_list == NULL && node->internal_strong_refs == 0 &&
				    !(node->proc &&
				      node == node->proc->context-> binder_context_mgr_node &&
				      node->has_strong_ref)) {
					pr_err("invalid inc strong node for %d\n", node->debug_id);
					return -EINVAL;
				}
				// 节点内部强引用+1
				node->internal_strong_refs++;
			} else
				// 节点本地强引用+1
				node->local_strong_refs++;
			if (!node->has_strong_ref && target_list) {
				binder_dequeue_work_ilocked(&node->work);
				/*
				 * Note: this function is the only place where we queue
				 * directly to a thread->todo without using the
				 * corresponding binder_enqueue_thread_work() helper
				 * functions; in this case it's ok to not set the
				 * process_todo flag, since we know this node work will
				 * always be followed by other work that starts queue
				 * processing: in case of synchronous transactions, a
				 * BR_REPLY or BR_ERROR; in case of oneway
				 * transactions, a BR_TRANSACTION_COMPLETE.
				 */
				 // 将Binder节点的事务work添加到目标进程todo链表
				binder_enqueue_work_ilocked(&node->work, target_list);
			}
		} else {
			if (!internal)
				// 节点本地弱引用+1
				node->local_weak_refs++;
			if (!node->has_weak_ref && list_empty(&node->work.entry)) {
				// 将Binder节点的事务work添加到目标进程todo链表 
				binder_enqueue_work_ilocked(&node->work, target_list);
			}
		}
		return 0;
	}

	static void binder_inc_node_tmpref_ilocked(struct binder_node *node)
	{
		/*
		 * No call to binder_inc_node() is needed since we
		 * don't need to inform userspace of any changes to
		 * tmp_refs
		 */
		 // 节点临时引用+1
		node->tmp_refs++;
	}

#### 5.7.3 target_thread的确定
目标线程target_thread的确定过程, 即寻找最优线程策略的过程, 其目的是提高线程利用率.  
只有同步通信才需要才有这种策略, 异步通信不存在线程等待的情况.

[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

    if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
	    // 获取当前线程事务栈
        struct binder_transaction *tmp = thread->transaction_stack;
        while (tmp) {
	        // 获取发起事务的线程
            struct binder_thread *from = tmp->from;
            // 若发起事务的线程所在的进程就是目标进程, 则说明该线程就是目标线程
            if (from && from->proc == target_proc) {
                atomic_inc(&from->tmp_ref);
                target_thread = from;
                break;
            }
            // 获取当前当前事务的上一个事务
            tmp = tmp->from_parent;
        }
    }

在分析这个过程之前，需要先搞清楚struct binder_transaction各个属性的含义：

	struct binder_transaction {
	    int debug_id; // 调试id
	    struct binder_work work;
	    struct binder_thread *from; // 源线程
	    struct binder_transaction *from_parent; // 上一个事务, 即该事务依赖的事务
	    struct binder_proc *to_proc; // 目标进程
	    struct binder_thread *to_thread; // 目标线程
	    struct binder_transaction *to_parent; // 下一个事务
	    unsigned need_reply:1; // 是否需要回复. 1-同步需要回复, 0-异步不需要回复

	    struct binder_buffer *buffer; // 指向一块内核缓冲区,用来保存进程间通信数据
	    unsigned int    code; // code
	    unsigned int    flags; // 通信方式同步异步

	    struct binder_priority  priority; // 源线程的优先级
	    struct binder_priority  saved_priority; // 暂存的源线程优先级
	    bool    set_priority_called;
	    kuid_t  sender_euid; // 用户id

	    spinlock_t lock;
	};

`(进程1 线程A) --- 事务T1 ---> (进程2 线程B) --- 事务T2 ---> (进程3 线程C)`  
假设进程1的线程A发起事务T1, 由进程2的线程B处理. 线程B在处理事务T1过程中, 需要进程3的线程C处理事务T2. 线程C在处理事务T2过程中, 又需要进程1处理事务T3.  

**这里存在一个问题, 线程C发起事务T3给进程1后, 具体交给哪个线程处理呢?**  

假设进程1新建一个线程B, 由线程B处理事务T3, 那是不是每来一个事务都新建一个线程呢? 这种策略明显是不太合理的. 最好的策略是交给空闲线程处理, 而此时线程A刚好是空闲的, 因此由线程A处理是最合理的.  

**那么线程C怎么找到线程A的呢?**  

根据struct binder_transaction的含义:  
T2 -> from_parent = T1  
T3 -> from_parent = T2  
T3 -> to_parent = T1  

我们再回过头来看上面代码的逻辑和注释就很好理解了.   
线程B的事务栈是: T1, 线程C的事务栈从上往下是T2 T1. 线程C先检查事务栈顶T2的源线程B, 发现进程B不是目标进程; 继续往下检查事务T1的源线程A, 发现进程A刚好是目标进程, 于是判定线程A即为目标线程.  

#### 5.7.4 to_flat_binder_object
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// 根据hdr属性获取flat_binder_object对象
	#define to_flat_binder_object(hdr) \
	    container_of(hdr, struct flat_binder_object, hdr)

#### 5.7.5 binder_translate_binder
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// 从源进程的节点红黑树中, 根据flat_binder_object对象的binder属性查找Binder节点
	// 若没有则创建一个Binder节点, 同时在目标进程中创建binder引用且binder引用+1
	// 同时修改flat_binder_object类型为引用类型BINDER_TYPE_HANDLE
	static int binder_translate_binder(struct flat_binder_object *fp,
	                   struct binder_transaction *t,
	                   struct binder_thread *thread)
	{
	    struct binder_node *node;
	    struct binder_proc *proc = thread->proc; // 源进程
	    struct binder_proc *target_proc = t->to_proc; // 目标进程
	    struct binder_ref_data rdata;
	    int ret = 0;

	    // 从源进程的节点红黑树中, 根据flat_binder_object对象的binder属性查找Binder节点
	    // 对于服务注册过程来说来说, 由于服务还未注册, 因此首次访问时获得的Binder节点为空
	    node = binder_get_node(proc, fp->binder);
	    if (!node) {
	        // 创建Binder节点
	        node = binder_new_node(proc, fp);
	        if (!node)
	            return -ENOMEM;
	    }

	    if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
	        ret = -EPERM;
	        goto done;
	    }
	 
	    // 在目标进程中, 节点对应的binder引用binder_ref加1
	    // 若type为BINDER_TYPE_BINDER, 则强引用加1, ref->data.strong++
	    // 若type为BINDER_TYPE_WEAK_BINDER, 则弱引用加1, ref->data.weak++
	    ret = binder_inc_ref_for_node(target_proc, node,
	            fp->hdr.type == BINDER_TYPE_BINDER,
	            &thread->todo, &rdata);
	    if (ret)
	        goto done;
	 
	    // 设置type为BINDER_TYPE_XXX_HANDLE
	    // 当Binder驱动程序将进程间通信数据传递给目标进程时, Binder实体对象变成了引用对象
	    // 参考上面binder_inc_ref_for_node方法
	    if (fp->hdr.type == BINDER_TYPE_BINDER)
	        fp->hdr.type = BINDER_TYPE_HANDLE;
	    else
	        fp->hdr.type = BINDER_TYPE_WEAK_HANDLE;
	    fp->binder = 0;
	    // 设置句柄
	    fp->handle = rdata.desc;
	    fp->cookie = 0;
	 
	done:
	    // 移除临时引用, node->tmp_refs--
	    binder_put_node(node);
	    return ret;
	}

**(1) binder_new_node**

	// 在进程节点红黑树中, 根据flat_binder_object对象的binder属性查找目标节点, 若不存在则新建节点并初始化
	static struct binder_node *binder_new_node(struct binder_proc *proc,
						   struct flat_binder_object *fp)
	{
		struct binder_node *node;
		struct binder_node *new_node = kzalloc(sizeof(*node), GFP_KERNEL);
	
		if (!new_node)
			return NULL;
		binder_inner_proc_lock(proc);
		// 初始化Binder节点
		node = binder_init_node_ilocked(proc, new_node, fp);
		binder_inner_proc_unlock(proc);
		if (node != new_node)
			/*
			 * The node was already added by another thread
			 */
			kfree(new_node);
	
		return node;
	}

**(2) binder_init_node_ilocked**

	// 在进程节点红黑树中, 根据flat_binder_object对象的binder属性查找目标节点, 若不存在则新建节点并初始化
	static struct binder_node *binder_init_node_ilocked(
							struct binder_proc *proc,
							struct binder_node *new_node,
							struct flat_binder_object *fp)
	{
		struct rb_node **p = &proc->nodes.rb_node; // 进程的节点红黑树
		struct rb_node *parent = NULL;
		struct binder_node *node;
		binder_uintptr_t ptr = fp ? fp->binder : 0;
		binder_uintptr_t cookie = fp ? fp->cookie : 0;
		__u32 flags = fp ? fp->flags : 0;
		s8 priority;

		assert_spin_locked(&proc->inner_lock);
	
		while (*p) { // 遍历节点红黑树
			parent = *p;
			// 获取Binder节点
			node = rb_entry(parent, struct binder_node, rb_node);

			if (ptr < node->ptr)
				p = &(*p)->rb_left;
			else if (ptr > node->ptr)
				p = &(*p)->rb_right;
			else {
				/*
				 * A matching node is already in
				 * the rb tree. Abandon the init
				 * and return it.
				 */
				// 找到目标节点, 节点临时引用+1
				binder_inc_node_tmpref_ilocked(node);
				return node;
			}
		}
        
        // 没找到目标节点, 则新建节点
		node = new_node;
		// 节点临时引用+1
		node->tmp_refs++;
		rb_link_node(&node->rb_node, parent, p);
		rb_insert_color(&node->rb_node, &proc->nodes);
		// debug_id
		node->debug_id = atomic_inc_return(&binder_last_id);
		// 进程
		node->proc = proc;
		node->ptr = ptr;
		node->cookie = cookie;
		// 设置节点type为BINDER_WORK_NODE
		node->work.type = BINDER_WORK_NODE;
		priority = flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
		node->sched_policy = (flags & FLAT_BINDER_FLAG_SCHED_POLICY_MASK) >>
			FLAT_BINDER_FLAG_SCHED_POLICY_SHIFT;
		node->min_priority = to_kernel_prio(node->sched_policy, priority);
		node->accept_fds = !!(flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
		node->inherit_rt = !!(flags & FLAT_BINDER_FLAG_INHERIT_RT);
		spin_lock_init(&node->lock);
		INIT_LIST_HEAD(&node->work.entry);
		INIT_LIST_HEAD(&node->async_todo);

		return node;
	}

**(3) binder_get_node_ilocked**

	// 从目标进程的节点红黑树中查找指定Binder节点
	static struct binder_node *binder_get_node_ilocked(struct binder_proc *proc,
							   binder_uintptr_t ptr)
	{
		// 进程的节点红黑树
		struct rb_node *n = proc->nodes.rb_node;
		struct binder_node *node;
	
		assert_spin_locked(&proc->inner_lock);
	
		while (n) {
			// 获取节点红黑树上的Binder节点
			node = rb_entry(n, struct binder_node, rb_node);
	
			if (ptr < node->ptr)
				n = n->rb_left;
			else if (ptr > node->ptr)
				n = n->rb_right;
			else {
				/*
				 * take an implicit weak reference
				 * to ensure node stays alive until
				 * call to binder_put_node()
				 */
				 // 找到目标节点, 且目标节点临时引用+1
				binder_inc_node_tmpref_ilocked(node);
				return node;
			}
		}
		return NULL;
	}

**(4) binder_inc_ref_for_node**

	static int binder_inc_ref_for_node(struct binder_proc *proc,
				struct binder_node *node, bool strong,
				struct list_head *target_list, struct binder_ref_data *rdata)
	{
		struct binder_ref *ref;
		struct binder_ref *new_ref = NULL;
		int ret = 0;
	
		ref = binder_get_ref_for_node_olocked(proc, node, NULL);
		if (!ref) {
			new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
			ref = binder_get_ref_for_node_olocked(proc, node, new_ref);
		}
		ret = binder_inc_ref_olocked(ref, strong, target_list);
		*rdata = ref->data;
		if (new_ref && ref != new_ref)
			/*
			 * Another thread created the ref first so
			 * free the one we allocated
			 */
			kfree(new_ref);
		return ret;
	}

**(5) binder_get_ref_for_node_olocked**

	// 从进程的refs_by_node节点红黑树中查找目标节点的binder_ref
	// 若没找到, 则创建新Binder引用new_ref并初始化, 同时将new_ref插入refs_by_node红黑树
	static struct binder_ref *binder_get_ref_for_node_olocked(
						struct binder_proc *proc,
						struct binder_node *node,
						struct binder_ref *new_ref)
	{
		struct binder_context *context = proc->context;
		struct rb_node **p = &proc->refs_by_node.rb_node; // 进程的refs_by_node节点红黑树
		struct rb_node *parent = NULL;
		struct binder_ref *ref;
		struct rb_node *n;

		while (*p) { // 遍历进程的refs_by_node红黑树
			parent = *p;
			// 得到binder_ref
			ref = rb_entry(parent, struct binder_ref, rb_node_node);
	
			if (node < ref->node)
				p = &(*p)->rb_left;
			else if (node > ref->node)
				p = &(*p)->rb_right;
			else
			    // 找到目标Binder引用
				return ref;
		}

        // 若未找到目标Binder引用, 且传入的new_ref为空, 则返回null
		if (!new_ref)
			return NULL;
	
		binder_stats_created(BINDER_STAT_REF);
		// 初始化new_ref各属性
		new_ref->data.debug_id = atomic_inc_return(&binder_last_id);
		new_ref->proc = proc;
		new_ref->node = node;
		rb_link_node(&new_ref->rb_node_node, parent, p);
		rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);
        
        // 遍历refs_by_desc红黑树节点, 计算new_ref的desc大小, 服务进程desc是累加的, servicemanager恒为0
		new_ref->data.desc = (node == context->binder_context_mgr_node) ? 0 : 1;
		for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
			ref = rb_entry(n, struct binder_ref, rb_node_desc);
			if (ref->data.desc > new_ref->data.desc)
				break;
			new_ref->data.desc = ref->data.desc + 1;
		}
	
	    // new_ref插入refs_by_desc红黑树中
		p = &proc->refs_by_desc.rb_node;
		while (*p) {
			parent = *p;
			ref = rb_entry(parent, struct binder_ref, rb_node_desc);
	
			if (new_ref->data.desc < ref->data.desc)
				p = &(*p)->rb_left;
			else if (new_ref->data.desc > ref->data.desc)
				p = &(*p)->rb_right;
			else
				BUG();
		}
		rb_link_node(&new_ref->rb_node_desc, parent, p);
		rb_insert_color(&new_ref->rb_node_desc, &proc->refs_by_desc);
	
		hlist_add_head(&new_ref->node_entry, &node->refs);

		return new_ref;
	}

#### 5.7.6 binder_proc_transaction
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// 此处proc为目标进程, thread为目标线程
	// 含义: 将事务t的work添加到目标线程或目标进程的同步或异步等待队列, 并唤醒目标线程的等待队列
	static bool binder_proc_transaction(struct binder_transaction *t,
	                    struct binder_proc *proc,
	                    struct binder_thread *thread)
	{
	    struct binder_node *node = t->buffer->target_node; // 目标节点
	    struct binder_priority node_prio; // Binder节点优先级
	    bool oneway = !!(t->flags & TF_ONE_WAY); // oneway
	    bool pending_async = false;
	  
	    // node_prio中设置目标节点的优先级信息
	    node_prio.prio = node->min_priority;
	    node_prio.sched_policy = node->sched_policy;
	  
	    if (oneway) { // 通信方式异步
	        if (node->has_async_transaction) { // 目标节点有异步事务
	            // 设置标识位pending_async
	            pending_async = true;
	        } else { // 目标节点无异步事务
	            node->has_async_transaction = 1;
	        }
	    }

	    // 若目标线程传入为空且目标节点无异步事务
	    if (!thread && !pending_async)
	        // 从进程的等待线程队列waiting_threads中取出第一个线程作为目标线程, 并删除该线程的等待线程节点.
	        thread = binder_select_thread_ilocked(proc);
	  
	    if (thread) { // 目标线程存在
	        // 事务t的saved_priority中保存目标线程的优先级, 并调整目标线程的优先级
	        // 目标线程优先级取源线程和目标进程所要求的线程优先级之间的较高优先级(priority值越小优先级越高)
	        binder_transaction_priority(thread->task, t, node_prio, node->inherit_rt);
	        // 将事务t的work添加到目标线程的todo链表
	        binder_enqueue_thread_work_ilocked(thread, &t->work);
	    } else if (!pending_async) { // 目标线程不存在, 且目标节点无异步事务
	        // 将事务t的work添加到目标进程的todo链表
	        binder_enqueue_work_ilocked(&t->work, &proc->todo);
	    } else { // 目标线程不存在, 且目标节点有异步事务
	        // 将事务t的work添加到目标进程的async_todo链表
	        binder_enqueue_work_ilocked(&t->work, &node->async_todo);
	    }

	    // 如果目标节点无异步事务, 则同步唤醒线程
	    if (!pending_async)
	        binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

	    return true;
	}

**(1) binder_select_thread_ilocked**

	// 从进程的等待线程队列waiting_threads中取出第一个线程, 并删除该线程的等待线程节点.
	static struct binder_thread *binder_select_thread_ilocked(struct binder_proc *proc)
	{
		struct binder_thread *thread;
	
		assert_spin_locked(&proc->inner_lock);
		// 从进程的线程等待队列waiting_threads中取出第一个线程
		thread = list_first_entry_or_null(&proc->waiting_threads,
						  struct binder_thread,
						  waiting_thread_node);
	
		if (thread)
		    // 删除该线程的等待线程节点waiting_thread_node
			list_del_init(&thread->waiting_thread_node);
	
		return thread;
	}

**(2) binder_transaction_priority**  

基本思路:  
当源线程发送事务t给目标线程执行时, Binder驱动需要修改目标线程优先级, 使得事务t和在源线程中执行一样.

	static void binder_transaction_priority(struct task_struct *task,
			struct binder_transaction *t, struct binder_priority node_prio, bool inherit_rt)
	{
		struct binder_priority desired_prio = t->priority;
	
		if (t->set_priority_called)
			return;
	
		// 将源线程的优先级信息task保存到事务t的saved_priority中
		t->set_priority_called = true;
		t->saved_priority.sched_policy = task->policy;
		t->saved_priority.prio = task->normal_prio;
	
		if (!inherit_rt && is_rt_policy(desired_prio.sched_policy)) {
			desired_prio.prio = NICE_TO_PRIO(0);
			desired_prio.sched_policy = SCHED_NORMAL;
		}
	
	    // 若目标进程要求的最小优先级node_prio小于事务t的优先级(值越小优先级越高)
        // 或者优先级相等, sched_policy=SCHED_FIFO, 则设置desired_prio信息
		if (node_prio.prio < t->priority.prio ||
		    (node_prio.prio == t->priority.prio &&
		     node_prio.sched_policy == SCHED_FIFO)) {
			/*
			 * In case the minimum priority on the node is
			 * higher (lower value), use that priority. If
			 * the priority is the same, but the node uses
			 * SCHED_FIFO, prefer SCHED_FIFO, since it can
			 * run unbounded, unlike SCHED_RR.
			 */
			desired_prio = node_prio;
		}

        // 修改源线程的优先级为desired_prio
		binder_set_priority(task, desired_prio);
	}

这段代码的含义是:  
源线程将事务t发送给目标线程执行时, Binder驱动需要先调整目标线程的优先级.  
调整策略为, 在源线程和目标进程所要求的线程优先级之间取较高的优先级(值越小优先级越高)  
调整前先保存目标线程原始的优先级到saved_priority  

**(3) binder_enqueue_thread_work_ilocked**

	// 将work添加到target_list链表, 线程的process_todo标识置true
	static void binder_enqueue_thread_work_ilocked(struct binder_thread *thread,
					   struct binder_work *work)
	{
		binder_enqueue_work_ilocked(work, &thread->todo);
		thread->process_todo = true;
	}

**(4) binder_enqueue_work_ilocked**

	// 将work添加到target_list链表
	binder_enqueue_work_ilocked(struct binder_work *work, struct list_head *target_list)
	{
		list_add_tail(&work->entry, target_list);
	}

**(5) binder_wakeup_thread_ilocked**

	static void binder_wakeup_thread_ilocked(struct binder_proc *proc,
						 struct binder_thread *thread, bool sync)
	{
		assert_spin_locked(&proc->inner_lock);
	
		if (thread) { // 若目标线程非空
			if (sync)
			    // 同步唤醒目标线程的等待队列
				wake_up_interruptible_sync(&thread->wait);
			else
				// 异步唤醒目标线程的等待队列
				wake_up_interruptible(&thread->wait);
			return;
		}
	
		/* Didn't find a thread waiting for proc work; this can happen
		 * in two scenarios:
		 * 1. All threads are busy handling transactions
		 *    In that case, one of those threads should call back into
		 *    the kernel driver soon and pick up this work.
		 * 2. Threads are using the (e)poll interface, in which case
		 *    they may be blocked on the waitqueue without having been
		 *    added to waiting_threads. For this case, we just iterate
		 *    over all threads not handling transaction work, and
		 *    wake them all up. We wake all because we don't know whether
		 *    a thread that called into (e)poll is handling non-binder
		 *    work currently.
		 */
		// 若目标线程为空, 则轮询目标进程中的线程, 并唤醒线程中的等待队列
		binder_wakeup_poll_threads_ilocked(proc, sync);
	}
	
	static void binder_wakeup_poll_threads_ilocked(struct binder_proc *proc, bool sync)
	{
		struct rb_node *n;
		struct binder_thread *thread;
	
		for (n = rb_first(&proc->threads); n != NULL; n = rb_next(n)) {
			thread = rb_entry(n, struct binder_thread, rb_node);
			if (thread->looper & BINDER_LOOPER_STATE_POLL &&
			    binder_available_for_proc_work_ilocked(thread)) {
				if (sync)
					wake_up_interruptible_sync(&thread->wait);
				else
					wake_up_interruptible(&thread->wait);
			}
		}
	}

## 6.BR_TRANSACTION_COMPLETE过程 (Binder驱动 -> 源线程)

上面已经分析了:  
(1) struct binder_work类型为BINDER_WORK_TRANSACTION_COMPLETE的对象添加到源线程的todo队列.  
(2) struct binder_work类型为BINDER_WORK_TRANSACTION的对象添加到目标进程/目标线程的todo队列.  
(3) 目标进程/目标线程的等待队列被唤醒.  

本小节分析 **源线程被唤醒并执行todo队列任务** 的情况.

### 6.1 binder_thread_read

	// binder驱动读过程
	// 以源线程执行binder_thread_read方法为例, 执行BINDER_WORK_TRANSACTION_COMPLETE
	// 目的进程或线程执行binder_thread_read方法为例, 执行BINDER_WORK_TRANSACTION
	static int binder_thread_read(struct binder_proc *proc,
	                  struct binder_thread *thread,
	                  binder_uintptr_t binder_buffer, size_t size,
	                  binder_size_t *consumed, int non_block)
	{
	    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	    void __user *ptr = buffer + *consumed;
	    void __user *end = buffer + size;
	   
	    int ret = 0;
	    int wait_for_proc_work;
	   
	    if (*consumed == 0) {
	        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
	            return -EFAULT;
	        ptr += sizeof(uint32_t);
	    }
	   
	retry:
	    // 判断当前线程是否正在等待进程任务(当满足下面2个条件时), 见【第6.1.1节】
	    // (1)当前线程事务栈和todo队列均为空
	    // (2)且当前线程looper标志为BINDER_LOOPER_STATE_ENTERED或BINDER_LOOPER_STATE_REGISTERED
	    wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
	   
	    // 设置线程looper等待标志
	    thread->looper |= BINDER_LOOPER_STATE_WAITING;
	   
	    if (wait_for_proc_work) {
	        // 恢复当前线程优先级
	        binder_restore_priority(current, proc->default_priority);
	    }
	  
	    if (non_block) {
	        ...
	    } else {
	        // 当wait_for_proc_work为true, 则进入休眠等待状态
	        ret = binder_wait_for_work(thread, wait_for_proc_work);
	    }
	  
	    // 设置线程looper退出等待标志
	    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
	  
	    if (ret)
	        return ret;
	  
	    while (1) {
	        uint32_t cmd;
	        struct binder_transaction_data tr;
	        struct binder_work *w = NULL;
	        struct list_head *list = NULL;
	        struct binder_transaction *t = NULL;
	  
	        // 当前线程todo队列非空
	        if (!binder_worklist_empty_ilocked(&thread->todo))
	            list = &thread->todo;
	        // 当前进程todo队列非空, 且wait_for_proc_work标识为true, 则取proc->todo
	        else if (!binder_worklist_empty_ilocked(&proc->todo) && wait_for_proc_work)
	            list = &proc->todo;
	        else {
	            // 若无数据且当前线程looper_need_return为false, 则重试
	            if (ptr - buffer == 4 && !thread->looper_need_return)
	                goto retry;
	            break;
	        }
	  
	        // binder读操作到buffer末尾, 结束循环
	        if (end - ptr < sizeof(tr) + 4) {
	            binder_inner_proc_unlock(proc);
	            break;
	        }
	  
	        // 从list队列中取出第一项任务
	        w = binder_dequeue_work_head_ilocked(list);
	        if (binder_worklist_empty_ilocked(&thread->todo))
	            thread->process_todo = false;
	  
	        switch (w->type) {
	        case BINDER_WORK_TRANSACTION: {
	            // 根据w实例(类型为binder_work)指针获得t实例(类型为binder_transaction)
	            t = container_of(w, struct binder_transaction, work);
	        } break;
	        case BINDER_WORK_RETURN_ERROR: ...
	        case BINDER_WORK_TRANSACTION_COMPLETE: {
	            // 将BR_TRANSACTION_COMPLETE返回协议写入用户空间的缓冲区
	            cmd = BR_TRANSACTION_COMPLETE;
	            if (put_user(cmd, (uint32_t __user *)ptr))
	                return -EFAULT;
	            ptr += sizeof(uint32_t);
	            // 释放已完成的任务对象
	            kfree(w);
	        } break;
	        case BINDER_WORK_NODE: ...
	        case BINDER_WORK_DEAD_BINDER:
	        case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
	        case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
	        }
  
	        // 若任务为空, 则继续循环
	        if (!t)
	            continue;
	  
	        if (t->buffer->target_node) { // 目标Binder节点非空
	            struct binder_node *target_node = t->buffer->target_node;
	            struct binder_priority node_prio;
				// 设置目标进程的Binder本地对象
	            tr.target.ptr = target_node->ptr;
	            tr.cookie =  target_node->cookie;
	            node_prio.sched_policy = target_node->sched_policy;
	            node_prio.prio = target_node->min_priority;
	            binder_transaction_priority(current, t, node_prio, target_node->inherit_rt);
	            // 发送BR_TRANSACTION返回协议
	            cmd = BR_TRANSACTION;
	        } else { // 目标Binder节点为空
	            tr.target.ptr = 0;
	            tr.cookie = 0;
	            // 发送BR_REPLY返回协议
	            cmd = BR_REPLY;
	        }
	        tr.code = t->code;
	        tr.flags = t->flags;
	        tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);
	  
	        t_from = binder_get_txn_from(t);
	        if (t_from) {
	            struct task_struct *sender = t_from->proc->tsk;
	            tr.sender_pid = task_tgid_nr_ns(sender, task_active_pid_ns(current));
	        } else {
	            tr.sender_pid = 0;
	        }
	  
	        tr.data_size = t->buffer->data_size;
	        tr.offsets_size = t->buffer->offsets_size;
	        // 修改binder_transaction_data结构体tr中的数据缓冲区和偏移数组的地址值
	        // 使其指向binder_transaction结构体t中的数据缓冲区和偏移数组的地址值
	        // 而前面分析往Binder驱动写数据, 即调用binder_thread_write方法时 
	        // Binder驱动分配给binder_transaction结构体t的内核缓冲区同时映射到进程的内核空间和用户空间.
	        // 通过这种方式来减少一次数据拷贝
	        tr.data.ptr.buffer = (binder_uintptr_t)((uintptr_t)t->buffer->data + 
			        binder_alloc_get_user_buffer_offset(&proc->alloc));
	        tr.data.ptr.offsets = tr.data.ptr.buffer +
			        ALIGN(t->buffer->data_size, sizeof(void *));
	  
	        // 将返回协议BR_TRANSACTION拷贝到目标线程thread提供的用户空间
	        if (put_user(cmd, (uint32_t __user *)ptr)) {
	            if (t_from)
	                binder_thread_dec_tmpref(t_from);
	            binder_cleanup_transaction(t, "put_user failed", BR_FAILED_REPLY);
	            return -EFAULT;
	        }
	        ptr += sizeof(uint32_t);
	        // 将struct binder_transaction_data结构体tr拷贝到目标线程thread提供的用户空间
	        if (copy_to_user(ptr, &tr, sizeof(tr))) {
	            if (t_from)
	                binder_thread_dec_tmpref(t_from);
	            binder_cleanup_transaction(t, "copy_to_user failed", BR_FAILED_REPLY);
	            return -EFAULT;
	        }
	        ptr += sizeof(tr);
	  
	        if (t_from)
	            binder_thread_dec_tmpref(t_from);
	        // 表示Binder驱动程序为它所分配的内核缓冲区允许目标线程thread在用户空间发出BC_FREE_BUFFER命令协议来释放
	        // 这里说的目标线程thread, 即servicemanager进程的主线程
	        t->buffer->allow_user_free = 1;
	        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) { // 同步通信
	            // 将事务t压入目标线程thread的事务堆栈中
	            binder_inner_proc_lock(thread->proc);
	            t->to_parent = thread->transaction_stack;
	            t->to_thread = thread;
	            thread->transaction_stack = t;
	            binder_inner_proc_unlock(thread->proc);
	            // 对于同步通信, 要等事务t处理完, 才释放内核空间
	        } else {
				// 对于异步通信, 事务t已经不需要了, 释放内核空间	        
	            binder_free_transaction(t);
	        }
	        break;
	    }
	  
	done:
	    // 异常处理: 当w->type为BINDER_WORK_DEAD_BINDER或BINDER_WORK_DEAD_BINDER_AND_CLEAR或BINDER_WORK_CLEAR_DEATH_NOTIFICATION
	    *consumed = ptr - buffer;
	    binder_inner_proc_lock(proc);
	    //当请求线程数和等待线程数均为0，已启动线程数小于最大线程数(15)，且looper状态为已注册或已进入时创建新的线程。
	    if (proc->requested_threads == 0 &&
	        list_empty(&thread->proc->waiting_threads) &&
	        proc->requested_threads_started < proc->max_threads &&
	        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)))
	    {
	        proc->requested_threads++;
	        binder_inner_proc_unlock(proc);
	        // 向用户空间发送BR码指令BR_SPAWN_LOOPER, 创建新线程
	        if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
	            return -EFAULT;
	    } else
	        binder_inner_proc_unlock(proc);
	    return 0;
	}

前面分析了BC_TRANSACTION过程, 会将tcomplete对象添加到源线程的todo队列.  
总结:  
从源线程的todo队列中取出tcomplete, type为BINDER_WORK_TRANSACTION_COMPLETE.  
将BR_TRANSACTION_COMPLETE写入用户空间缓冲区mIn中.

binder_thread_write和binder_thread_read方法执行完后， 会依次返回talkWithDriver -> waitForResponse方法.  

#### 6.1.1 binder_available_for_proc_work_ilocked

	static bool binder_available_for_proc_work_ilocked(struct binder_thread *thread)
	{
		return !thread->transaction_stack &&
			binder_worklist_empty_ilocked(&thread->todo) &&
			(thread->looper & (BINDER_LOOPER_STATE_ENTERED | BINDER_LOOPER_STATE_REGISTERED));
	}

## 7.BR_TRANSACTION过程 (Binder驱动 -> ServiceManager进程)
前面分析了BC_TRANSACTION过程, 会将t.work对象添加到目标进程/目标线程(servicemanager)的todo队列.  
而servicemanager进程启动后, 会通过binder_loop循环不断从Binder驱动读取数据并解析数据.  
调用链为:  binder_loop -> ioctl -> binder_thread_read -> binder_parse  

本小节分析:  
(1) **目标进程servicemanager被唤醒并执行todo队列任务**.  见第【6.1节】binder_thread_read方法.  
(2) **数据解析过程**  

### 7.1 binder_thread_read

 **目标进程servicemanager被唤醒并执行todo队列任务**:  
(1) 从目标线程的todo队列读取t.work, type为BINDER_WORK_TRANSACTION  
(2) 根据t.work属性获得binder_transaction事务t  
(3) BR_TRANSACTION返回用户空间  
`put_user(BR_TRANSACTION, (uint32_t __user *)ptr)`  
(4) 创建`binder_transaction_data`对象tr, 其buffer/offsets指向事务t的内核缓冲区的buffer/offsets  
(5) 拷贝binder_transaction_data对象tr, 由内核空间->用户空间  
    `copy_to_user(ptr, &tr, sizeof(tr))`  
(6) 将事务t添加到目标线程的事务栈transaction_stack中  

### 7.2 binder_parse
[ -> frameworks/native/cmds/servicemanager/binder.c]

	int binder_parse(struct binder_state *bs, struct binder_io *bio,
	             uintptr_t ptr, size_t size, binder_handler func)
	{
	    int r = 1;
	    uintptr_t end = ptr + (uintptr_t) size;
	 
	    while (ptr < end) {
	        // 从Binder驱动读取cmd. 此处为BR_TRANSACTION
	        uint32_t cmd = *(uint32_t *) ptr;
	        ptr += sizeof(uint32_t);
	 
	        switch(cmd) {
	        case BR_NOOP: ...
	        case BR_TRANSACTION_COMPLETE: ...
	        case BR_INCREFS: ...
	        case BR_ACQUIRE: ...
	        case BR_RELEASE: ...
	        case BR_DECREFS: ...
	        case BR_TRANSACTION_SEC_CTX:
	        case BR_TRANSACTION: {
		        struct binder_transaction_data_secctx txn;
	            if (cmd == BR_TRANSACTION_SEC_CTX) {
	                memcpy(&txn, (void*) ptr, sizeof(struct binder_transaction_data_secctx));
	                ptr += sizeof(struct binder_transaction_data_secctx);
	            } else /* BR_TRANSACTION */ {
	                // 从Binder驱动读取binder_transaction_data
	                memcpy(&txn.transaction_data, (void*) ptr,
			                sizeof(struct binder_transaction_data));
	                ptr += sizeof(struct binder_transaction_data);
 
	                txn.secctx = 0;
	            }
 
	            if (func) {
	                unsigned rdata[256/4];
	                struct binder_io msg;
	                struct binder_io reply;
	                int res;
	                // 初始化binder_io类型对象reply
	                bio_init(&reply, rdata, sizeof(rdata), 4);
	                // 初始化binder_io类型对象msg, 解析binder_transaction_data类型对象
	                bio_init_from_txn(&msg, &txn.transaction_data);
	                // 执行svcmgr_handler方法
	                res = func(bs, &txn, &msg, &reply);
	                if (txn.transaction_data.flags & TF_ONE_WAY) { // 异步通信
	                    // 释放内核缓冲区内存
	                    binder_free_buffer(bs, txn.transaction_data.data.ptr.buffer);
	                } else { // 同步通信
	                    // 向binder驱动发送reply结果
	                    binder_send_reply(bs, &reply, txn.transaction_data.data.ptr.buffer, res);
	                }
	            }
	            break;
	        }
	        case BR_REPLY: ...
	        case BR_DEAD_BINDER: ...
	        case BR_FAILED_REPLY: ...
	        case BR_DEAD_REPLY: ...
	        default: ...
	        }
	    }
	 
	    return r;
	}

#### 7.2.1 数据结构binder_io  
[ -> frameworks/native/cmds/servicemanager/binder.h ]

	// 用来解析进程间通信数据, 作用类似于Binder中的Parcel
	struct binder_io
	{
	    // 数据缓冲区当前地址
	    char *data;            /* pointer to read/write from */
	    // 偏移数组当前地址
	    binder_size_t *offs;   /* array of offsets */
	    // 数组缓冲区data0中未被解析的内容
	    size_t data_avail;     /* bytes available in data buffer */
	    // 偏移数组offs0中未被解析的内容
	    size_t offs_avail;     /* entries available in offsets array */
	 
	    // 数据缓冲区起始地址
	    char *data0;           /* start of data buffer */
	    // 偏移数组起始地址
	    binder_size_t *offs0;  /* start of offsets buffer */
	    // 描述数据缓冲区的属性, BIO_F_SHARED/BIO_F_OVERFLOW/BIO_F_IOERROR/BIO_F_MALLOCED
	    uint32_t flags;
	    uint32_t unused;
	};

#### 7.2.2 bio_init  

	// data: bio中缓冲区data
	// maxdata: 缓冲区data的大小
	// maxoffs: 偏移数组的大小
	void bio_init(struct binder_io *bio, void *data, size_t maxdata, size_t maxoffs)
	{
	    // 偏移数组的大小
	    size_t n = maxoffs * sizeof(size_t);
	
	    if (n > maxdata) { // 偏移数组的大小大于缓冲区data的大小, 则溢出
	        bio->flags = BIO_F_OVERFLOW;
	        bio->data_avail = 0;
	        bio->offs_avail = 0;
	        return;
	    }
	
		// 指向数据缓冲区
	    bio->data = bio->data0 = (char *) data + n;
	    // 指向偏移数组
	    bio->offs = bio->offs0 = data;
	    // 数据缓冲区可用大小
	    bio->data_avail = maxdata - n;
	    // 偏移数组可用大小
	    bio->offs_avail = maxoffs;
	    bio->flags = 0;
	}

#### 7.2.3 bio_init_from_txn  

	void bio_init_from_txn(struct binder_io *bio, struct binder_transaction_data *txn)
	{
	    bio->data = bio->data0 = (char *)(intptr_t)txn->data.ptr.buffer;
	    bio->offs = bio->offs0 = (binder_size_t *)(intptr_t)txn->data.ptr.offsets;
	    bio->data_avail = txn->data_size;
	    bio->offs_avail = txn->offsets_size / sizeof(size_t);
	    bio->flags = BIO_F_SHARED;
	}

### 7.3 svcmgr_handler
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	int svcmgr_handler(struct binder_state *bs,
	               struct binder_transaction_data_secctx *txn_secctx,
	               struct binder_io *msg,
	               struct binder_io *reply)
	{
	    uint16_t *s;
	    size_t len;
	    uint32_t handle;
	  
	    struct binder_transaction_data *txn = &txn_secctx->transaction_data;
	  
	    if (txn->target.ptr != BINDER_SERVICE_MANAGER)
	        return -1;
	  
	    if (txn->code == PING_TRANSACTION)
	        return 0;

	    // 解析msg数据. 对应输入数据:Parcel.writeInterfaceToken(IServiceManager.descriptor)
	    strict_policy = bio_get_uint32(msg);
	    bio_get_uint32(msg);  // Ignore worksource header.
	    s = bio_get_string16(msg, &len);
	    if (s == NULL) {
	        return -1;
	    }
	
	    if ((len != (sizeof(svcmgr_id) / 2)) || memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
	        fprintf(stderr,"invalid id %s\n", str8(s, len));
	        return -1;
	    }

	    switch(txn->code) {
	    case SVC_MGR_GET_SERVICE: ...
	    case SVC_MGR_CHECK_SERVICE: ...
	    case SVC_MGR_ADD_SERVICE: // 等于ADD_SERVICE_TRANSACTION
	        // 解析Parcel.writeString(name)
	        s = bio_get_string16(msg, &len);
	        // 解析Parcel.writeStrongBinder(service)
	        handle = bio_get_ref(msg);
	        // 解析Parcel.writeInt(allowIsolated ? 1 : 0)
	        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
	        // 解析Parcel.writeInt(dumpPriority)
	        dumpsys_priority = bio_get_uint32(msg);
	        if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated,
			    dumpsys_priority, txn->sender_pid, (const char*) txn_secctx->secctx))
	            return -1;
	        break;
	    case SVC_MGR_LIST_SERVICES: ...
	    }
	  
	    bio_put_uint32(reply, 0);
	    return 0;
	}

### 7.4 do_add_service
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
	     uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid, const char* sid)
	{
	    struct svcinfo *si;
	
		// 数据有效性校验, writeInterfaceToken中长度校验
	    if (!handle || (len == 0) || (len > 127))
	        return -1;
	
		// 权限校验
	    if (!svc_can_register(s, len, spid, sid, uid)) {
	        ...
	        return -1;
	    }
	
		// 根据name查找释放已存在相应服务
	    si = find_svc(s, len);
	    if (si) { // 若服务已存在
	        if (si->handle) {
	            ...
	            // 向Binder驱动发送BC_RELEASE命令协议, 释放服务对象
	            svcinfo_death(bs, si);
	        }
	        si->handle = handle;
	    } else {
		    // 为服务si分配内存, 并将服务添加到svclist链表头部
	        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
	        if (!si) {
	            ...
	            return -1;
	        }
	        si->handle = handle;
	        si->len = len;
	        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
	        si->name[len] = '\0';
	        si->death.func = (void*) svcinfo_death;
	        si->death.ptr = si;
	        si->allow_isolated = allow_isolated;
	        si->dumpsys_priority = dumpsys_priority;
	        si->next = svclist;
	        svclist = si;
	    }
	
	    // 向Binder驱动发送BC_ACQUIRE命令协议, 强引用+1
	    binder_acquire(bs, handle);
	    // 向Binder驱动发送BC_REQUEST_DEATH_NOTIFICATION命令协议, 清理内存
	    binder_link_to_death(bs, handle, &si->death);
	    return 0;
	}

#### 7.4.1 find_svc

	struct svcinfo *svclist = NULL;
	struct svcinfo *find_svc(const uint16_t *s16, size_t len)
	{
	    struct svcinfo *si;
	
	    for (si = svclist; si; si = si->next) {
	        if ((len == si->len) && !memcmp(s16, si->name, len * sizeof(uint16_t))) {
	            return si;
	        }
	    }
	    return NULL;
	}

#### 7.4.2 svcinfo_death

	void svcinfo_death(struct binder_state *bs, void *ptr)
	{
	    struct svcinfo *si = (struct svcinfo* ) ptr;
	
	    if (si->handle) {
	        binder_release(bs, si->handle);
	        si->handle = 0;
	    }
	}

	void binder_release(struct binder_state *bs, uint32_t target)
	{
	    uint32_t cmd[2];
	    cmd[0] = BC_RELEASE;
	    cmd[1] = target;
	    binder_write(bs, cmd, sizeof(cmd));
	}

#### 7.4.3 binder_acquire

	void binder_acquire(struct binder_state *bs, uint32_t target)
	{
	    uint32_t cmd[2];
	    cmd[0] = BC_ACQUIRE;
	    cmd[1] = target;
	    binder_write(bs, cmd, sizeof(cmd));
	}

#### 7.4.4 binder_link_to_death

	void binder_link_to_death(struct binder_state *bs, uint32_t target,
			struct binder_death *death)
	{
	    struct {
	        uint32_t cmd;
	        struct binder_handle_cookie payload;
	    } __attribute__((packed)) data;
	
	    data.cmd = BC_REQUEST_DEATH_NOTIFICATION;
	    data.payload.handle = target;
	    data.payload.cookie = (uintptr_t) death;
	    binder_write(bs, &data, sizeof(data));
	}

## 8.BC_FREE_BUFFER & BC_REPLY过程 (ServiceManager进程 -> Binder驱动)

本小节分析:  **目标进程/目标线程(servicemanager) 向 源进程/源线程 返回结果**.

本次Reply通信中, 源进程/源线程 和 目标进程/目标线程 刚好是反过来的.  
proc/thread指的servicemanager进程及线程,  target_proc/target_thread指源进程/源线程

### 8.1 binder_send_reply
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	void binder_send_reply(struct binder_state *bs,
	                       struct binder_io *reply,
	                       binder_uintptr_t buffer_to_free,
	                       int status)
	{
	    struct {
	        uint32_t cmd_free;
	        binder_uintptr_t buffer;
	        uint32_t cmd_reply;
	        struct binder_transaction_data txn;
	    } __attribute__((packed)) data;
	
	    // 设置cmd_free=BC_FREE_BUFFER
	    data.cmd_free = BC_FREE_BUFFER;
	    // 设置要释放的内存地址
	    data.buffer = buffer_to_free;
	    // 设置cmd_reply=BC_REPLY
	    data.cmd_reply = BC_REPLY;
	    // 设置txn
	    data.txn.target.ptr = 0;
	    data.txn.cookie = 0;
	    data.txn.code = 0;
	    if (status) {
	        data.txn.flags = TF_STATUS_CODE;
	        data.txn.data_size = sizeof(int);
	        data.txn.offsets_size = 0;
	        data.txn.data.ptr.buffer = (uintptr_t)&status;
	        data.txn.data.ptr.offsets = 0;
	    } else { // status为0, 表示解析成功
	        data.txn.flags = 0;
	        data.txn.data_size = reply->data - reply->data0;
	        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
	        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
	        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
	    }
	    // 往Binder驱动写data
	    binder_write(bs, &data, sizeof(data));
	}

### 8.2 binder_thread_write BC_FREE_BUFFER
目标进程/目标线程(servicemanager) 向 源进程/源线程 发送BC_FREE_BUFFER命令协议, 释放内核缓冲区内存.

	static int binder_thread_write(struct binder_proc *proc,
	            struct binder_thread *thread,
	            binder_uintptr_t binder_buffer, size_t size,
	            binder_size_t *consumed)
	{
	    uint32_t cmd;
	    void __user *ptr = buffer + *consumed;
	    void __user *end = buffer + size;
	  
	    while (ptr < end && thread->return_error.cmd == BR_OK) {
	        int ret;
	          
	        // 获取BC_FREE_BUFFER
	        if (get_user(cmd, (uint32_t __user *)ptr))
	            return -EFAULT;
	        ptr += sizeof(uint32_t);
	        switch (cmd) {
	        ...
	        case BC_FREE_BUFFER: {
	        	binder_uintptr_t data_ptr;
				struct binder_buffer *buffer;
	
				// 获取要释放的内核缓冲区的用户空间地址data_ptr
				if (get_user(data_ptr, (binder_uintptr_t __user *)ptr))
					return -EFAULT;
				ptr += sizeof(binder_uintptr_t);
	
				// 在进程中找到与用户空间地址data_ptr对应的内核缓冲区buffer
				buffer = binder_alloc_prepare_to_free(&proc->alloc, data_ptr);
				if (buffer == NULL) { // 空判断
					...
					break;
				}
				if (!buffer->allow_user_free) { // 不允许释放
					...
					break;
				}
	
    	        // 检查buffer是否已分配给binder_transaction事务使用
				if (buffer->transaction) {
					buffer->transaction->buffer = NULL;
					buffer->transaction = NULL;
				}
				
				// 检查buffer是否已分配给Binder实体对象用来处理异步事务
				if (buffer->async_transaction && buffer->target_node) {
					struct binder_node *buf_node;
					struct binder_work *w;
	
					buf_node = buffer->target_node;
					binder_node_inner_lock(buf_node);
					// 获取异步事务队列async_todo的第一项, 出队列
					w = binder_dequeue_work_head_ilocked(&buf_node->async_todo);
					if (!w) { // 异步事务队列为空, 说明没有正在处理的异步事务
						buf_node->has_async_transaction = 0;
					} else { // 异步事务队列非空
						// 将w添加到servicemanager进程的todo队列尾
						binder_enqueue_work_ilocked(w, &proc->todo);
						// 唤醒servicemanager进程,处理todo队列事务
						binder_wakeup_proc_ilocked(proc);
					}
					binder_node_inner_unlock(buf_node);
				}
				// 释放内核缓冲区buffer的Binder实体对象或引用对象
				binder_transaction_buffer_release(proc, buffer, NULL);
				// 释放内核缓冲区buffer
				binder_alloc_free_buf(&proc->alloc, buffer);
				break;
			}
	        ...
	        }

	        *consumed = ptr - buffer;
	    }
	    return 0;
	}

#### 8.2.1 binder_transaction_buffer_release

	static void binder_transaction_buffer_release(struct binder_proc *proc,
						      struct binder_buffer *buffer,
						      binder_size_t *failed_at)
	{
		binder_size_t *offp, *off_start, *off_end;
		int debug_id = buffer->debug_id;
	
		// 检查buffer是否已分配给Binder实体对象使用
		if (buffer->target_node)
			binder_dec_node(buffer->target_node, 1, 0);
	
		off_start = (binder_size_t *)(buffer->data + ALIGN(buffer->data_size, sizeof(void *)));
		if (failed_at)
			off_end = failed_at;
		else
			off_end = (void *)off_start + buffer->offsets_size;
		// 遍历buffer中的所有Binder实体对象或引用对象
		for (offp = off_start; offp < off_end; offp++) {
			struct binder_object_header *hdr;
			size_t object_size = binder_validate_object(buffer, *offp);
	
			hdr = (struct binder_object_header *)(buffer->data + *offp);
			switch (hdr->type) {
			case BINDER_TYPE_BINDER:
			case BINDER_TYPE_WEAK_BINDER: { // 释放Binder实体对象
				struct flat_binder_object *fp;
				struct binder_node *node;
	
				fp = to_flat_binder_object(hdr);
				node = binder_get_node(proc, fp->binder);
				binder_dec_node(node, hdr->type == BINDER_TYPE_BINDER, 0);
				binder_put_node(node);
			} break;
			case BINDER_TYPE_HANDLE:
			case BINDER_TYPE_WEAK_HANDLE: { // 释放Binder引用对象
				struct flat_binder_object *fp;
				struct binder_ref_data rdata;
				int ret;
	
				fp = to_flat_binder_object(hdr);
				ret = binder_dec_ref_for_handle(proc, fp->handle,
					hdr->type == BINDER_TYPE_HANDLE, &rdata);
			} break;
			case BINDER_TYPE_FD: {
				struct binder_fd_object *fp = to_binder_fd_object(hdr);
				if (failed_at)
					task_close_fd(proc, fp->fd);
			} break;
			case BINDER_TYPE_PTR:
				break;
			case BINDER_TYPE_FDA: ...
		}
	}

#### 8.2.2 binder_alloc_free_buf

	void binder_alloc_free_buf(struct binder_alloc *alloc, struct binder_buffer *buffer)
	{
		mutex_lock(&alloc->mutex);
		binder_free_buf_locked(alloc, buffer);
		mutex_unlock(&alloc->mutex);
	}

	static void binder_free_buf_locked(struct binder_alloc *alloc,struct binder_buffer *buffer)
	{
		size_t size, buffer_size;
	
		buffer_size = binder_alloc_buffer_size(alloc, buffer);
		size = ALIGN(buffer->data_size, sizeof(void *)) +
			ALIGN(buffer->offsets_size, sizeof(void *)) +
			ALIGN(buffer->extra_buffers_size, sizeof(void *));

		if (buffer->async_transaction) {
			alloc->free_async_space += size + sizeof(struct binder_buffer);
		}
	
		binder_update_page_range(alloc, 0,
			(void *)PAGE_ALIGN((uintptr_t)buffer->data),
			(void *)(((uintptr_t)buffer->data + buffer_size) & PAGE_MASK));
	
		rb_erase(&buffer->rb_node, &alloc->allocated_buffers);
		buffer->free = 1;
		if (!list_is_last(&buffer->entry, &alloc->buffers)) {
			struct binder_buffer *next = binder_buffer_next(buffer);
			if (next->free) {
				rb_erase(&next->rb_node, &alloc->free_buffers);
				binder_delete_free_buffer(alloc, next);
			}
		}
		if (alloc->buffers.next != &buffer->entry) {
			struct binder_buffer *prev = binder_buffer_prev(buffer);
			if (prev->free) {
				binder_delete_free_buffer(alloc, buffer);
				rb_erase(&prev->rb_node, &alloc->free_buffers);
				buffer = prev;
			}
		}
		binder_insert_free_buffer(alloc, buffer);
	}

### 8.3 binder_thread_write BC_REPLY
目标进程/目标线程(servicemanager) 向 源进程/源线程 发送BC_REPLY命令协议.

这个过程和BC_TRANSACTION过程一样, 都会调用binder_thread_write -> binder_transaction.  
不同点是binder_transaction事务传输的方向由servicemanager进程发送到源进程.  

	static void binder_transaction(struct binder_proc *proc,
	                   struct binder_thread *thread,
	                   struct binder_transaction_data *tr, int reply,
	                   binder_size_t extra_buffers_size)
	{
	    struct binder_transaction *t; // binder事务
	    struct binder_work *tcomplete;
	    struct binder_proc *target_proc = NULL; // 目标进程
	    struct binder_thread *target_thread = NULL; // 目标线程
	    struct binder_node *target_node = NULL; // 目标binder节点
	    struct binder_transaction *in_reply_to = NULL; // binder事务
	  
	    if (reply) { // BC_REPLY
			// 取出sercicemanager进程中主线程的事务栈
			in_reply_to = thread->transaction_stack;
			if (in_reply_to == NULL) {
				return_error = BR_FAILED_REPLY;
				goto err_empty_call_stack;
			}
			if (in_reply_to->to_thread != thread) {
				return_error = BR_FAILED_REPLY;
				goto err_bad_call_stack;
			}
			// 指向事务in_reply_to的下一个事务
			thread->transaction_stack = in_reply_to->to_parent;
			// 获取事务in_reply_to的源线程为目标线程
			target_thread = binder_get_txn_from_and_acq_inner(in_reply_to);
			if (target_thread == NULL) {
				return_error = BR_DEAD_REPLY;
				goto err_dead_binder;
			}
			if (target_thread->transaction_stack != in_reply_to) {
				return_error = BR_FAILED_REPLY;
				goto err_dead_binder;
			}
			// 目标进程
			target_proc = target_thread->proc;
			// 目标进程临时引用+1
			target_proc->tmp_ref++;
	    } else {
	        ...
	    }
	    // 分配struct binder_transaction类型的数据t
	    t = kzalloc(sizeof(*t), GFP_KERNEL);
	    // 分配struct binder_work类型的数据tcomplete
	    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

	    if (!reply && !(tr->flags & TF_ONE_WAY))
	        t->from = thread;
	    else // 如果当前处理的是BC_REPLY, 源线程设置为空
	        t->from = NULL;
	    // 设置uid
	    t->sender_euid = task_euid(proc->tsk);
	    // 设置目标进程
	    t->to_proc = target_proc;
	    // 设置目标线程
	    t->to_thread = target_thread;
	    // 设置code
	    t->code = tr->code;
	    // 设置通信方式
	    t->flags = tr->flags;
	    // 设置优先级
	    if (!(t->flags & TF_ONE_WAY) && binder_supported_policy(current->policy)) {
	        /* Inherit supported policies for synchronous transactions */
	        t->priority.sched_policy = current->policy;
	        t->priority.prio = current->normal_prio;
	    } else {
	        /* Otherwise, fall back to the default priority */
	        t->priority = target_proc->default_priority;
	    }
	    // 从目标进程分配一个内核缓冲区给事务t
	    t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size, tr->offsets_size, extra_buffers_size, !reply && (t->flags & TF_ONE_WAY));
	    t->buffer->debug_id = t->debug_id;
	    t->buffer->transaction = t;
	    // BC_REPLY时, target_node为空
	    t->buffer->target_node = target_node;

	    // 设置事务t中偏移数组的开始位置off_start, 即当前位置+binder_transaction_data数据大小
	    off_start = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));
	    // offp指向偏移数组的起始位置
	    offp = off_start;
	    // 将用户空间中binder_transaction_data类型数据tr的数据缓冲区, 拷贝到内核空间中事务t的内核缓冲区
	    copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)tr->data.ptr.buffer,
			    tr->data_size))
	    // 将用户空间中binder_transaction_data类型数据tr的偏移数组, 拷贝到内核空间中事务t的偏移数组中
	    // offp为事务t中偏移数组的起始位置
	    copy_from_user(offp, (const void __user *)(uintptr_t)tr->data.ptr.offsets,
			    tr->offsets_size))
	    // 设置事务t中偏移数组的结束位置off_end
	    off_end = (void *)off_start + tr->offsets_size;

	    for (; offp < off_end; offp++) { // 遍历偏移数组区间
	        // 从事务t的数据缓冲区中, 获取offp位置的binder_object_header对象
	        struct binder_object_header *hdr = (struct binder_object_header *)
			        (t->buffer->data + *offp);
	        switch (hdr->type) {
	        case BINDER_TYPE_BINDER: ...
	        case BINDER_TYPE_WEAK_BINDER: ...
	        case BINDER_TYPE_HANDLE:
	        case BINDER_TYPE_WEAK_HANDLE: {
				struct flat_binder_object *fp;
				struct binder_ref_data rdata;
				int ret;
			    // 根据hdr属性获取flat_binder_object对象的值.见【第5.7.4节】
				fp = to_flat_binder_object(hdr);
				// flat_binder_object中服务引用binder_ref减1
				ret = binder_dec_ref_for_handle(proc, fp->handle,
					hdr->type == BINDER_TYPE_HANDLE, &rdata);
			}
	        case BINDER_TYPE_FD: ...
	        case BINDER_TYPE_FDA: ...
	        case BINDER_TYPE_PTR: ...
	        default: ...
	        }
	    }
	    // 设置事务tcomplete的类型
	    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	    // 设置事务t的类型
	    t->work.type = BINDER_WORK_TRANSACTION;

	    if (reply) { // BC_REPLY
	        // binder_work类型对象tcomplete添加到servicemanager进程的主线程的todo队列
	        binder_enqueue_thread_work(thread, tcomplete);
			// 由于在BC_TRANSACTION过程, 已经将事务t压入源线程(本次通信的目标线程)的事务堆栈的栈顶
			// 指向源线程(本次通信的目标线程)的当前事务(in_reply_to)的上一个事务
			// 相当于当前事务in_reply_to出栈
			// 将事务in_reply_to的源线程置空
			binder_pop_transaction_ilocked(target_thread, in_reply_to);
			// 事务t的work属性添加到目标线程的todo队列
			binder_enqueue_thread_work_ilocked(target_thread, &t->work);
			// 同步唤醒目标线程的等待队列
			wake_up_interruptible_sync(&target_thread->wait);
			// 恢复当前线程的优先级
			binder_restore_priority(current, in_reply_to->saved_priority);
			// 释放事务in_reply_to占用的内核空间
			binder_free_transaction(in_reply_to);
	    } else if (!(t->flags & TF_ONE_WAY)) {
			...
	    } else {
			...
	    }
	}

对于BC_REPLY过程, 数据传输方向是目标线程(servicemanager) -> 源线程

总结：  
(1) 从目标线程的事务栈栈顶取出事务in_reply_to, 从中获得源进程/源线程(作为本次通信的目标进程/目标线程)  
(2) 创建`binder_transaction`事务t, 创建`binder_work`对象tcomplete, 根据in_reply_to设置事务t的目标进程/目标线程  
(3) 将`binder_transaction_data`对象中buffer/offsets拷贝到事务t的内核空间  
`copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size))`  
`copy_from_user(offp, (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size))`  
(4) 根据事务t的offsets依次读取buffer, 得到`flat_binder_object`对象(binder_object_header是其一属性), 读取`binder_object_header`对象的类型为BINDER_TYPE_HANDLE, binder_ref引用减1  
(5) 源线程(本次通信的目标线程)当前事务in_reply_to出栈  
(6) 将t.work添加到源线程(本次通信的目标线程)的同步或异步等待队列, 类型为BINDER_WORK_TRANSACTION  
(7) 将tcomplete添加到目标线程(本次通信的源线程)的todo链表, type为BINDER_WORK_TRANSACTION_COMPLETE   
(8) 唤醒源线程(本次通信的目标线程)的等待队列  
(9) 恢复servicemanager线程优先级  
(10) 释放事务in_reply_to所占用的内核缓冲区   

#### 8.3.1 binder_get_txn_from_and_acq_inner

	// 获取事务t的源线程
	static struct binder_thread *binder_get_txn_from_and_acq_inner(
			struct binder_transaction *t)
	{
		struct binder_thread from = binder_get_txn_from(t);

		if (t->from) {
			return from;
		}
		// 源线程临时引用-1
		binder_thread_dec_tmpref(from);
		return NULL;
	}

	static struct binder_thread *binder_get_txn_from(struct binder_transaction *t)
	{
		struct binder_thread from = t->from;
		if (from)
			// 源线程临时引用+1
			atomic_inc(&from->tmp_ref);
		return from;
	}

#### 8.3.2 binder_pop_transaction_ilocked

	static void binder_pop_transaction_ilocked(struct binder_thread *target_thread,
						   struct binder_transaction *t)
	{
		target_thread->transaction_stack = target_thread->transaction_stack->from_parent;
		t->from = NULL;
	}

## 9.BR_TRANSACTION_COMPLETE过程 (Binder驱动 -> ServiceManager进程)

本小节分析 **servicemanager线程被唤醒并执行todo队列任务** 的情况.

根据代码可知, 什么都不做, 直接跳出循环.

### 9.1 binder_parse
[ -> frameworks/native/cmds/servicemanager/binder.c]

	int binder_parse(struct binder_state *bs, struct binder_io *bio,
	             uintptr_t ptr, size_t size, binder_handler func)
	{
	    int r = 1;
	    uintptr_t end = ptr + (uintptr_t) size;
	 
	    while (ptr < end) {
	        // 从Binder驱动读取cmd. 此处为BR_TRANSACTION_COMPLETE
	        uint32_t cmd = *(uint32_t *) ptr;
	        ptr += sizeof(uint32_t);
	 
	        switch(cmd) {
	        ...
	        case BR_TRANSACTION_COMPLETE:
		        break;
	        }
	        ...
	    }
	 
	    return r;
	}

## 10.BR_REPLY过程 (Binder驱动 -> 源线程)

本小节分析 **源线程(本次通信的目标线程)被唤醒并执行todo队列任务** 的情况.

### 10.1 binder_thread_read

	static int binder_thread_read(struct binder_proc *proc,
	                  struct binder_thread *thread,
	                  binder_uintptr_t binder_buffer, size_t size,
	                  binder_size_t *consumed, int non_block)
	{
	    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	    void __user *ptr = buffer + *consumed;
	    void __user *end = buffer + size;
	   
	    int ret = 0;
	    int wait_for_proc_work;
	   
	    if (*consumed == 0) {
	        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
	            return -EFAULT;
	        ptr += sizeof(uint32_t);
	    }
	   
	retry:
	    binder_inner_proc_lock(proc);
	    // 获取当前线程是否在looper模式BINDER_LOOPER_STATE_ENTERED或BINDER_LOOPER_STATE_REGISTERED
	    wait_for_proc_work = binder_available_for_proc_work_ilocked(thread);
	    binder_inner_proc_unlock(proc);
	   
	    thread->looper |= BINDER_LOOPER_STATE_WAITING;
	   
	    if (wait_for_proc_work) {
	        binder_restore_priority(current, proc->default_priority);
	    }
	  
	    if (non_block) {
	        ...
	    } else {
	        // 阻塞
	        ret = binder_wait_for_work(thread, wait_for_proc_work);
	    }
	  
	    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;
	  
	    if (ret)
	        return ret;
	  
	    while (1) {
	        uint32_t cmd;
	        struct binder_transaction_data tr;
	        struct binder_work *w = NULL;
	        struct list_head *list = NULL;
	        struct binder_transaction *t = NULL;
	  
	        // 当前线程todo队列非空
	        if (!binder_worklist_empty_ilocked(&thread->todo))
	            list = &thread->todo;
	        // 当前进程todo队列非空, 且wait_for_proc_work标识为true, 则取proc->todo
	        else if (!binder_worklist_empty_ilocked(&proc->todo) && wait_for_proc_work)
	            list = &proc->todo;
	        else {
	            ...
	        }
	  
	        // binder读操作到buffer末尾, 结束循环
	        if (end - ptr < sizeof(tr) + 4) {
	            binder_inner_proc_unlock(proc);
	            break;
	        }
	  
	        // 从list队列中取出第一项任务
	        w = binder_dequeue_work_head_ilocked(list);
	        if (binder_worklist_empty_ilocked(&thread->todo))
	            thread->process_todo = false;
	  
	        switch (w->type) {
	        case BINDER_WORK_TRANSACTION: {
	            // 根据w实例(类型为binder_work)指针获得t实例(类型为binder_transaction)
	            t = container_of(w, struct binder_transaction, work);
	        } break;
	        ...
	        }
  
	        // 若任务为空, 则继续循环
	        if (!t)
	            continue;
	  
	        if (t->buffer->target_node) { // 目标Binder节点非空
	            ...
	        } else { // 目标Binder节点为空(由于BC_REPLY过程, target_node为空)
	            tr.target.ptr = 0;
	            tr.cookie = 0;
	            // 发送BR_REPLY返回协议
	            cmd = BR_REPLY;
	        }
	        tr.code = t->code;
	        tr.flags = t->flags;
	        tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);
	  
	        t_from = binder_get_txn_from(t);
	        if (t_from) {
	            struct task_struct *sender = t_from->proc->tsk;
	            tr.sender_pid = task_tgid_nr_ns(sender, task_active_pid_ns(current));
	        } else {
	            tr.sender_pid = 0;
	        }
	  
	        tr.data_size = t->buffer->data_size;
	        tr.offsets_size = t->buffer->offsets_size;
	        // 修改binder_transaction_data结构体tr中的数据缓冲区和偏移数组的地址值
	        // 使其指向binder_transaction结构体t中的数据缓冲区和偏移数组的地址值
	        // 而前面分析往Binder驱动写数据, 即调用binder_thread_write方法时 
	        // Binder驱动分配给binder_transaction结构体t的内核缓冲区同时映射到进程的内核空间和用户空间.
	        // 通过这种方式来减少一次数据拷贝
	        tr.data.ptr.buffer = (binder_uintptr_t)((uintptr_t)t->buffer->data + 
			        binder_alloc_get_user_buffer_offset(&proc->alloc));
	        tr.data.ptr.offsets = tr.data.ptr.buffer +
			        ALIGN(t->buffer->data_size, sizeof(void *));
	  
	        // 将返回协议BR_TRANSACTION拷贝到目标线程thread提供的用户空间
	        if (put_user(cmd, (uint32_t __user *)ptr)) {
	            if (t_from)
	                binder_thread_dec_tmpref(t_from);
	            binder_cleanup_transaction(t, "put_user failed", BR_FAILED_REPLY);
	            return -EFAULT;
	        }
	        ptr += sizeof(uint32_t);
	        // 将struct binder_transaction_data结构体tr拷贝到目标线程thread提供的用户空间
	        if (copy_to_user(ptr, &tr, sizeof(tr))) {
	            if (t_from)
	                binder_thread_dec_tmpref(t_from);
	            binder_cleanup_transaction(t, "copy_to_user failed", BR_FAILED_REPLY);
	            return -EFAULT;
	        }
	        ptr += sizeof(tr);
	  
	        if (t_from)
	            binder_thread_dec_tmpref(t_from);

	        // 内存可以释放
	        t->buffer->allow_user_free = 1;
	        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) { // 同步通信
	            ...
	        } else {
				// 对于异步通信, 事务t已经不需要了, 释放内核空间	        
	            binder_free_transaction(t);
	        }
	        break;
	    }
	  
	done: // 异常处理
	    *consumed = ptr - buffer;
	    ...
	    return 0;
	}

总结:  
(1) 从源线程(本次通信的目标线程)的todo队列取出`binder_work`对象w(BC_REPLY过程放入)  
(2) 根据w属性获得`binder_transaction`事务t  
(3) 向源线程(本次通信的目标线程)的用户空间发送BR_REPLY返回协议  
(4) 将`binder_transaction_data`对象tr拷贝到源线程(本次通信的目标线程)提供的用户空间   
(5) 设置事务t的内核缓冲区标志位allow_user_free, 表示内存可释放, 释放事务t内存空间(此时内核缓冲区buffer还没有释放)  

上面过程执行完binder_thread_read后, 会跳出talkWithDriver, 走到IPCThreadState::waitForResponse的BR_REPLY

### 10.2 IPCThreadState.waitForResponse
[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
	{
	    while(1) {
		    // 驱动交互过程.见【第5.3节】
	        if ((err=talkWithDriver()) < NO_ERROR) break;
	        
	        cmd = (uint32_t)mIn.readInt32();
	         
	        switch (cmd) {
	        ...
	        case BR_REPLY:
	            {
	                binder_transaction_data tr;
	                err = mIn.read(&tr, sizeof(tr));
	 
	                if (reply) {
	                    // 前面binder_send_reply设置flags为0
	                    if ((tr.flags & TF_STATUS_CODE) == 0) {
	                        // 设置reply
	                        reply->ipcSetDataReference(
	                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
	                            tr.data_size,
	                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
	                            tr.offsets_size/sizeof(binder_size_t),
	                            freeBuffer, this);
	                    } else {
	                        ...
	                    }
	                } else {
	                    ...
	                }
	            }
	            goto finish;
	        ...
	        }
	    }
	 
	finish:
	    ...
	    return err;
	}

	// 向Binder驱动发送BC_FREE_BUFFER指令, 释放内存(由~Parcel析构函数触发,会调用freeDataNoInit)
	void IPCThreadState::freeBuffer(Parcel* parcel, const uint8_t* data,
	                                size_t /*dataSize*/,
	                                const binder_size_t* /*objects*/,
	                                size_t /*objectsSize*/, void* /*cookie*/)
	{
	    if (parcel != nullptr) parcel->closeFileDescriptors();
	    IPCThreadState* state = self();
	    state->mOut.writeInt32(BC_FREE_BUFFER);
	    state->mOut.writePointer((uintptr_t)data);
	}

### 10.3 Parcel.ipcSetDataReference
[ -> frameworks/native/libs/binder/Parcel.cpp ]

	void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
		    const binder_size_t* objects, size_t objectsCount,
		    release_func relFunc, void* relCookie)
	{
	    binder_size_t minOffset = 0;
	    // 释放当前数据缓冲区
	    freeDataNoInit();

        // 重新初始化数据缓冲区
	    mError = NO_ERROR;
	    mData = const_cast<uint8_t*>(data);
	    mDataSize = mDataCapacity = dataSize;
	    mDataPos = 0;
	    mObjects = const_cast<binder_size_t*>(objects);
	    mObjectsSize = mObjectsCapacity = objectsCount;
	    mNextObjectHint = 0;
	    mObjectsSorted = false;
	    mOwner = relFunc;
	    mOwnerCookie = relCookie;
	    for (size_t i = 0; i < mObjectsSize; i++) {
	        binder_size_t offset = mObjects[i];
	        if (offset < minOffset) {
				...
	            mObjectsSize = 0;
	            break;
	        }
	        minOffset = offset + sizeof(flat_binder_object);
	    }
	    scanForFds();
	}

	// 析构函数, 释放内存
	Parcel::~Parcel()
	{
	    freeDataNoInit();
	}





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

2.1 Parcel封装Service信息  
2.2 Service注册过程  
(1) BC_TRANSACTION  
(2) BR_TRANSACTION  
(3) BC_REPLY  
(4) BR_REPLY  
2.3 返回结果  

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

我们重点分析`data.writeStrongBinder(service);`

### 3.1 Parcel.writeStrongBinder
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

#### 3.1.1 ibinderForJavaObject
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

#### 3.1.2 writeStrongBinder
[ -> frameworks/native/libs/binder/Parcel.cpp]

	status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
	{
	    return flatten_binder(ProcessState::self(), val, this);
	}

#### 3.1.3 flatten_binder
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

#### 3.1.4 Parcel.writeObject
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
	    // mHandle=0, 即目的进程是servicemanager
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

## 5.发送BC_TRANSACTION命令协议

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
	    // 设置目的进程的句柄
	    tr.target.handle = handle;
	    // 设置code, 表示要干啥
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
命令协议 + binder_transaction_data结构体数据

### 5.2 IPCThreadState.waitForResponse
[ -> frameworks/native/libs/binder/IPCThreadState.cpp]

	status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
	{
	    while(1) {
	        if ((err=talkWithDriver()) < NO_ERROR) break;
	         
	        cmd = (uint32_t)mIn.readInt32();
	         
	        switch (cmd) {
	        case BR_TRANSACTION_COMPLETE: ...
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
(1) 封装struct binder_write_read类型数据, 并设置用于读/写的数据缓冲区  
(2) 通过ioctl方法和Binder驱动交互执行数据读/写操作.  
(3) 读/写过程完成后, 修改mOut和mIn中发生改变的属性    

对于注册服务的BC_TRANSACTION过程, 只有写数据.

### 5.4 binder_ioctl
这里分析ioctl系统调用:  ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)  
最终会调用到Binder驱动的binder_ioctl方法.  

[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
	{
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
	    // 目的进程, 此处指的servicemanager进程
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
	        // 如果目的进程的todo链表非空, 则唤醒目的进程中等待状态的线程来处理todo任务
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

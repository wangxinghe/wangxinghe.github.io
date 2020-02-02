---
layout: post
comments: true
title: "Service获取过程"
description: "Service获取过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析Service获取过程, 包括从Java层获取和从Native层获取.

## 0.文件结构

	frameworks/base/core/java/android/os/ServiceManager.java
	frameworks/base/core/java/android/os/ServiceManagerNative
	frameworks/base/core/java/android/os/IServiceManager.java
	frameworks/base/core/java/android/os/IBinder.java
	 
	frameworks/native/libs/binder/IServiceManager.cpp
	frameworks/native/cmds/servicemanager/binder.h
	frameworks/native/cmds/servicemanager/service_manager.c

## 1.时序图
TODO

## 2.获取服务入口

### 2.1 Java层

#### 2.1.1 ServiceManager.getService  
[ -> frameworks/base/core/java/android/os/ServiceManager.java ]

	public static IBinder getService(String name) {
	    IBinder service = sCache.get(name);
	    if (service != null) {
	        return service;
	   } else {
	        return Binder.allowBlocking(rawGetService(name));
	    }
	}

	private static IBinder rawGetService(String name) throws RemoteException {
	    final IBinder binder = getIServiceManager().getService(name);
	    return binder;
	}

#### 2.1.2 ServiceManagerProxy.getService  
[ -> frameworks/base/core/java/android/os/ServiceManagerNative ]

	class ServiceManagerProxy implements IServiceManager {
		public IBinder getService(String name) throws RemoteException {
		    Parcel data = Parcel.obtain();
		    Parcel reply = Parcel.obtain();
		    data.writeInterfaceToken(IServiceManager.descriptor);
		    data.writeString(name);
		    // 获取服务通信过程
		    mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
		    // 读取IBinder实例
		    IBinder binder = reply.readStrongBinder();
		    reply.recycle();
		    data.recycle();
		    return binder;
		}
	}

### 2.2 Native层
以AMS服务为例.

[ -> frameworks/av/media/libmediaplayerservice/ActivityManager.cpp ]

	int openContentProviderFile(const String16& uri)
	{
	    int fd = -1;
	 
	    // 获取BpServiceManager对象
	    sp<IServiceManager> sm = defaultServiceManager();
	    // 获取服务
	    sp<IBinder> binder = sm->getService(String16("activity"));
	    // 获取BpActivityManager, 等价于IActivityManager::asInterface
	    sp<IActivityManager> am = interface_cast<IActivityManager>(binder);
	    if (am != NULL) {
	        fd = am->openContentUri(uri);
	    }
	 
	    return fd;
	}

#### 2.2.1 BpServiceManager.getService  
[ -> frameworks/native/libs/binder/IServiceManager.cpp ]

	class BpServiceManager : public BpInterface<IServiceManager>
	{ 
	    virtual sp<IBinder> getService(const String16& name) const
	    {
	        // 获取服务
	        sp<IBinder> svc = checkService(name);
	        if (svc != nullptr) return svc;
	 
	        // 超时时间5s
	        const long timeout = uptimeMillis() + 5000;
	        // 重试时间间隔
	        // retry interval in millisecond; note that vendor services stay at 100ms
	        const long sleepTime = gSystemBootCompleted ? 1000 : 100;
	 
	        int n = 0;
	        while (uptimeMillis() < timeout) {
	            n++;
	            // 休眠sleepTime毫秒
	            usleep(1000*sleepTime);
	 
	            // 获取服务
	            sp<IBinder> svc = checkService(name);
	            if (svc != nullptr) return svc;
	        }
	        return nullptr;
	    }
	}

#### 2.2.2 BpServiceManager.checkService  
[ -> frameworks/native/libs/binder/IServiceManager.cpp ]

	virtual sp<IBinder> checkService( const String16& name) const
	{
	    Parcel data, reply;
	    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
	    data.writeString16(name);
	    remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
	    return reply.readStrongBinder();
	}

**获取服务**和**注册服务**整理流程差不多. 可直接参考 **[Service注册过程--Java服务](http://mouxuejie.com/blog/2020-01-18/add-service/)**

**差别在于入参code.**   
(1) 注册服务的code是:  `ADD_SERVICE_TRANSACTION`  
(2) 获取服务的code是:  
`GET_SERVICE_TRANSACTION`:  Java层获取服务  
`CHECK_SERVICE_TRANSACTION`:  Native层获取服务  

code对应关系如下:    
Java层

	// frameworks/base/core/java/android/os/IBinder.java
	int FIRST_CALL_TRANSACTION  = 0x00000001; 

	// frameworks/base/core/java/android/os/IServiceManager.java
	int GET_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION;
	int CHECK_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+1;
	int ADD_SERVICE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+2;
	int LIST_SERVICES_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+3;
	int CHECK_SERVICES_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+4;
	int SET_PERMISSION_CONTROLLER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+5;

Native层

	// frameworks/native/include/binder/IBinder.h
	FIRST_CALL_TRANSACTION  = 0x00000001,

	// frameworks/native/include/binder/IServiceManager.h
    enum {
	    GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
	    CHECK_SERVICE_TRANSACTION,
	    ADD_SERVICE_TRANSACTION,
	    LIST_SERVICES_TRANSACTION,
	};

servicemanager层

	// frameworks/native/cmds/servicemanager/binder.h
	enum {
	   /* Must match definitions in IBinder.h and IServiceManager.h */
	   PING_TRANSACTION  = B_PACK_CHARS('_','P','N','G'),
	   SVC_MGR_GET_SERVICE = 1,
	   SVC_MGR_CHECK_SERVICE,
	   SVC_MGR_ADD_SERVICE,
	   SVC_MGR_LIST_SERVICES,
	};

## 3.ServiceManager进程处理

ServiceManager进程处理过程略有差别.

### 3.1 svcmgr_handler  
[ -> frameworks/native/cmds/servicemanager/service_manager.c ]

	int svcmgr_handler(struct binder_state *bs,
	               struct binder_transaction_data_secctx *txn_secctx,
	               struct binder_io *msg,
	               struct binder_io *reply)
	{
	    uint16_t *s;
	    size_t len;
	    uint32_t handle;
	   
	    struct binder_transaction_data *txn =
			&txn_secctx->transaction_data;
	 
	    switch(txn->code) {
	    case SVC_MGR_GET_SERVICE: // 等于GET_SERVICE_TRANSACTION
	    case SVC_MGR_CHECK_SERVICE: // 等于CHECK_SERVICE_TRANSACTION
	        s = bio_get_string16(msg, &len);
	        if (s == NULL) {
	            return -1;
	        }
	        handle = do_find_service(s, len, txn->sender_euid,
			    txn->sender_pid, (const char*) txn_secctx->secctx);
	        if (!handle)
	            break;
	        // 将handle保存到reply
	        bio_put_ref(reply, handle);
	        return 0;
	    case SVC_MGR_ADD_SERVICE: ...
	    case SVC_MGR_LIST_SERVICES: ...
	    }
	   
	    bio_put_uint32(reply, 0);
	    return 0;
	}

### 3.2 do_find_service

	uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid, const char* sid)
	{
	    // 查找服务
	    struct svcinfo *si = find_svc(s, len);
	 
	    // 有效性判断
	    if (!si || !si->handle) {
	        return 0;
	    }
	 
	    // 检查服务是否允许从isolated进程访问
	    if (!si->allow_isolated) {
	        // If this service doesn't allow access from isolated processes,
	        // then check the uid to see if it is isolated.
	        uid_t appid = uid % AID_USER;
	        if (appid >= AID_ISOLATED_START &&
			        appid <= AID_ISOLATED_END) {
	            return 0;
	        }
	    }
	 
	    // 权限检查
	    if (!svc_can_find(s, len, spid, sid, uid)) {
	        return 0;
	    }
	 
	    return si->handle;
	}

### 3.3 find_svc

	// 遍历svclist链表, 根据name查找服务
	struct svcinfo *find_svc(const uint16_t *s16, size_t len)
	{
	    struct svcinfo *si;
	 
	    for (si = svclist; si; si = si->next) {
	        if ((len == si->len) &&
	            !memcmp(s16, si->name, len * sizeof(uint16_t))) {
	            return si;
	        }
	    }
	    return NULL;
	}

## 4.返回结果
本小节分析Binder通信过程结束后，读取结果.

### 4.1 Parcel::readStrongBinder  
[ -> frameworks/native/libs/binder/Parcel.cpp ]

	sp<IBinder> Parcel::readStrongBinder() const
	{
	    sp<IBinder> val;
	    readNullableStrongBinder(&val);
	    return val;
	}

	status_t Parcel::readNullableStrongBinder(sp<IBinder>* val) const
	{
	    return unflatten_binder(ProcessState::self(), *this, val);
	}

### 4.2 unflatten_binder  
[ -> frameworks/native/libs/binder/Parcel.cpp ]

	status_t unflatten_binder(const sp<ProcessState>& proc,
	    const Parcel& in, sp<IBinder>* out)
	{
	    // 读取flat_binder_object对象
	    const flat_binder_object* flat = in.readObject(false);
	 
	    if (flat) {
	        switch (flat->hdr.type) {
	            case BINDER_TYPE_BINDER:
	                *out = reinterpret_cast<IBinder*>(flat->cookie);
	                return finish_unflatten_binder(nullptr, *flat, in);
	            case BINDER_TYPE_HANDLE: // 此处为引用类型
	                *out = proc->getStrongProxyForHandle(flat->handle);
	                return finish_unflatten_binder(
	                    static_cast<BpBinder*>(out->get()), *flat, in);
	        }
	    }
	    return BAD_TYPE;
	}

### 4.3 ProcessState::getStrongProxyForHandle  
[ -> frameworks/native/libs/binder/ProcessState.cpp ]

	// 先从mHandleToObject查询是否包含句柄为handle的BpBinder对象
	// 若没有则创建BpBinder对象
	sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
	{
	    sp<IBinder> result;
	 
	    // 查询是否包含句柄为handle的BpBinder对象
	    handle_entry* e = lookupHandleLocked(handle);
	 
	    if (e != nullptr) {
	        // Binder实体对象
	        IBinder* b = e->binder;
	        // Binder实体对象为空
	        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
	            // 如果获取ServiceManager守护进程，需要先确认守护进程是否启动
	            if (handle == 0) {
	                Parcel data;
	                status_t status = IPCThreadState::self()->transact(
	                    0, IBinder::PING_TRANSACTION, data, nullptr, 0);
	                if (status == DEAD_OBJECT)
	                   return nullptr;
	            }
	 
	            // 创建BpBinder实例
	            b = BpBinder::create(handle);
	            // 设置Binder实体
	            e->binder = b;
	            if (b) e->refs = b->getWeakRefs();
	            result = b;
	        } else {
	            result.force_set(b);
	            e->refs->decWeak(this);
	        }
	    }
	 
	    return result;
	}

### 4.4 finish_unflatten_binder  
[ -> frameworks/native/libs/binder/Parcel.cpp ]

	// 返回状态码
	inline static status_t finish_unflatten_binder(
	    BpBinder* /*proxy*/, const flat_binder_object& /*flat*/,
	    const Parcel& /*in*/)
	{
	    return NO_ERROR;
	}

### 4.5 interface_cast (Native层)  
接下来看`sp<IActivityManager> am = interface_cast<IActivityManager>(binder);`

根据文章 **[ServiceManager获取过程--Native层](http://mouxuejie.com/blog/2020-01-05/get-service-manager-native/)** 的章节**4.获取BpServiceManager对象** 可类比得出, 下面三者等价.  

	// 以AMS为例
	interface_cast<IActivityManager>(binder)
	IActivityManager::asInterface(binder)
	new BpActivityManager(new BpBinder(handle)) 

## 5.总结  
**获取服务**和**注册服务**Binder通信过程差不多.   
差别是code不同, Client要SM做的事情不同, 从而存在一些业务逻辑差别.   
获取服务code:   
GET_SERVICE_TRANSACTION-Native层  
CHECK_SERVICE_TRANSACTION－Java层  
注册服务code:   
ADD_SERVICE_TRANSACTION

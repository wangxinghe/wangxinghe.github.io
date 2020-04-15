---
layout: post
comments: true
title: "ContentProvider过程"
description: "ContentProvider过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文以查询联系人demo为切入点, 分析ContentProvider原理.

## 0.文件结构

	frameworks/base/core/java/android/app/ContextImpl.java
	frameworks/base/core/java/android/app/ActivityThread.java
	frameworks/base/core/java/android/content/ContentProvider.java
	frameworks/base/core/java/android/content/ContentProviderNative.java
	frameworks/base/core/java/android/app/ContentProviderHolder.java
	frameworks/base/core/java/android/content/ContentResolver.java
	frameworks/base/core/java/android/app/IActivityManager.aidl
	frameworks/base/core/java/android/os/ICancellationSignal.aidl

	frameworks/base/core/java/android/database/BulkCursorToCursorAdaptor.java
	frameworks/base/core/java/android/database/AbstractWindowedCursor.java
	frameworks/base/core/java/android/database/AbstractCursor.java
	frameworks/base/core/java/android/database/CursorWindow.java
	frameworks/base/core/jni/android_database_CursorWindow.cpp

	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
	frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java

	packages/providers/ContactsProvider/src/com/android/providers/contacts/ContactsProvider2.java

## 1.时序图

TODO

## 2.分析入口

通过ContentProvider操作手机通信录的demo示例:

	// 查询联系人信息
	public void getContactInfo() {
	    ContentResolver contentResolver = getContext().getContentResolver();
	    Uri uri = Uri.parse("content://com.android.contacts/contacts");
	    Cursor cursor = contentResolver.query(uri, null, null, null, null);
	    while(cursor.moveToNext()){
	        // 获取联系人姓名
	        String name = cursor.getString(cursor.getColumnIndex(ContactsContract.Contacts.DISPLAY_NAME));
	        String contactId = cursor.getString(cursor.getColumnIndex(ContactsContract.Contacts._ID)); 
	        
	        // 获取联系人手机号码
	        Cursor phones = contentResolver.query(
			        ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
			        null, 
			        ContactsContract.CommonDataKinds.Phone.CONTACT_ID +" = "+ contactId, 
	                null, null); 
	        while (phones.moveToNext()) {
	            String phone = phones.getString(phones.getColumnIndex("data1"));
	        }
         
	        // 获取联系人Email
	        Cursor emails = contentResolver.query(
			        ContactsContract.CommonDataKinds.Email.CONTENT_URI, 
                    null, 
                    ContactsContract.CommonDataKinds.Email.CONTACT_ID + " = " + contactId, 
                    null, null);
	        while (emails.moveToNext()) {
	            String email = emails.getString(emails.getColumnIndex("data1"));
	        }
	    }
	}

	// 添加联系人信息
	public void addContactInfo() {
	    ContentValues values = new ContentValues();
	    // 首先向RawContacts.CONTENT_URI执行一个空值插入，目的是获取系统返回的rawContactId
	    Uri rawContactUri = getContext().getContentResolver().insert(RawContacts.CONTENT_URI, values);
	    long rawContactId = ContentUris.parseId(rawContactUri);
     
	    // 往data表中插入联系人信息
	    values.clear();
	    values.put(Data.RAW_CONTACT_ID, rawContactId);
	    values.put(Data.MIMETYPE, StructuredName.CONTENT_ITEM_TYPE);
	    // 名字
	    values.put(StructuredName.GIVEN_NAME, "zhangsan");
	    // 手机号码
	    values.put(Phone.NUMBER, "5554");
	    values.put(Phone.TYPE, Phone.TYPE_MOBILE);
 	    // Email
 	    values.put(Email.DATA, "ljq218@126.com");
	    values.put(Email.TYPE, Email.TYPE_WORK);
		getContext().getContentResolver().insert(ContactsContract.Data.CONTENT_URI, values);
	}

本文以获取联系人姓名(即query操作) 为例. 

## 3.ContextImpl.getContentResolver
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

	private final ApplicationContentResolver mContentResolver;

	private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
	        @NonNull LoadedApk packageInfo, @Nullable String splitName,
	        @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
	        @Nullable ClassLoader classLoader, @Nullable String overrideOpPackageName) {
	    ...
	    mContentResolver = new ApplicationContentResolver(this, mainThread);
	}
 
	@Override
	public ContentResolver getContentResolver() {
	    return mContentResolver;
	}

根据上面代码可知, ContextImpl.getContentResolver获取到ApplicationContentResolver对象.

## 4.ApplicationContentResolver.query
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

	private static final class ApplicationContentResolver extends ContentResolver {
	    private final ActivityThread mMainThread;
 
	    public ApplicationContentResolver(Context context, ActivityThread mainThread) {
	        super(context);
	        mMainThread = Preconditions.checkNotNull(mainThread);
	    }
 
	    @Override
	    protected IContentProvider acquireProvider(Context context, String auth) {
	        return mMainThread.acquireProvider(context,
	                ContentProvider.getAuthorityWithoutUserId(auth),
	                resolveUserIdFromAuthority(auth), true);
	    }
 
	    @Override
	    protected IContentProvider acquireExistingProvider(Context context, String auth) {
	        return mMainThread.acquireExistingProvider(context,
	                ContentProvider.getAuthorityWithoutUserId(auth),
	                resolveUserIdFromAuthority(auth), true);
	    }
 
	    @Override
	    protected IContentProvider acquireUnstableProvider(Context c, String auth) {
	        return mMainThread.acquireProvider(c,
	                ContentProvider.getAuthorityWithoutUserId(auth),
	                resolveUserIdFromAuthority(auth), false);
	    }
 
	    protected int resolveUserIdFromAuthority(String auth) {
	        return ContentProvider.getUserIdFromAuthority(auth, getUserId());
	    }
		...
	}

而ApplicationContentResolver继承ContentResolver, 我们看下父类的实现:

	public abstract class ContentResolver implements ContentInterface {
		...
		public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
		        @Nullable String[] projection, @Nullable String selection,
		        @Nullable String[] selectionArgs, @Nullable String sortOrder) {
		    return query(uri, projection, selection, selectionArgs, sortOrder, null);
		}

		public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
		        @Nullable String[] projection, @Nullable String selection,
		        @Nullable String[] selectionArgs, @Nullable String sortOrder,
		        @Nullable CancellationSignal cancellationSignal) {
		    // 将查询条件封装到Bundle. 参考【第4.1节】
		    Bundle queryArgs = createSqlQueryBundle(selection, selectionArgs, sortOrder);
		    // 查询操作. 参考【第4.2节】
		    return query(uri, projection, queryArgs, cancellationSignal);
		}
		...
	}

其中 ContentResolver.query(...) 各参数含义:  
- uri: URI类型, 以content://开头, 如content://com.android.contacts/contacts
- projection: String[]类型, 表示返回哪些列, null表示返回所有列
- selection: String类型, SQL WHERE后面语句(可以用?占位), 表示返回哪行, null表示返回所有行
- selectionArgs: String[]类型, 表示selection中?占位符的值
- sortOrder: 返回行的排序方式, ORDER BY后面语句, null表示默认方式(即无序)
- cancellationSignal: CancellationSignal类型, 取消某操作(ContentResolver增删改查操作)的信号, 当操作取消时会抛出OperationCanceledException异常, null表示不取消
- 返回值: Cursor类型, 指向第一项的前面, 可能返回null(如果没有返回内容)

### 4.1 createSqlQueryBundle

	public static @Nullable Bundle createSqlQueryBundle(
	        @Nullable String selection,
	        @Nullable String[] selectionArgs,
	        @Nullable String sortOrder) {
	    if (selection == null && selectionArgs == null && sortOrder == null) {
	        return null;
	    }
 
	    Bundle queryArgs = new Bundle();
	    if (selection != null) {
	        queryArgs.putString(QUERY_ARG_SQL_SELECTION, selection);
	    }
	    if (selectionArgs != null) {
	        queryArgs.putStringArray(QUERY_ARG_SQL_SELECTION_ARGS, selectionArgs);
	    }
	    if (sortOrder != null) {
	        queryArgs.putString(QUERY_ARG_SQL_SORT_ORDER, sortOrder);
	    }
	    return queryArgs;
	}

### 4.2 query

	@Override
	public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
	        @Nullable String[] projection, @Nullable Bundle queryArgs,
	        @Nullable CancellationSignal cancellationSignal) {
	    // 此处mWrapped为空, 调不到
	    if (mWrapped != null) {
	        return mWrapped.query(uri, projection, queryArgs, cancellationSignal);
	    }
 
	    // 获取ContentProviderProxy对象(stable为false). 参考【第5节】
	    IContentProvider unstableProvider = acquireUnstableProvider(uri);
	    if (unstableProvider == null) {
	        return null;
	    }
	    IContentProvider stableProvider = null;
	    Cursor qCursor = null;
	    try {
	        long startTime = SystemClock.uptimeMillis();
 
	        ICancellationSignal remoteCancellationSignal = null;
	        if (cancellationSignal != null) {
	            cancellationSignal.throwIfCanceled();
	            // 拿到ICancellationSignal.Stub.Proxy
	            remoteCancellationSignal = unstableProvider.createCancellationSignal();
	            // 设置远程代理
	            cancellationSignal.setRemote(remoteCancellationSignal);
	        }
	        try {
	            // ContentProviderProxy#query. 参考【第7节】
	            qCursor = unstableProvider.query(mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
	        } catch (DeadObjectException e) {
	            unstableProviderDied(unstableProvider);
	            // 获取ContentProviderProxy对象(stable为true)
	            stableProvider = acquireProvider(uri);
	            if (stableProvider == null) {
	                return null;
	            }
	            // ContentProviderProxy#query
	            qCursor = stableProvider.query(mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
	        }
	        if (qCursor == null) {
	            return null;
	        }
 
	        // 获取查到的行数, 当qCursor被关闭时会抛出异常
	        qCursor.getCount();
 
	        // CursorWrapperInner封装qCursor和provder并返回
	        final IContentProvider provider = (stableProvider != null) ? stableProvider : acquireProvider(uri);
	        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
	        stableProvider = null;
	        qCursor = null;
	        return wrapper;
	    } catch (RemoteException e) {
	        return null;
	    } finally {
		    // 资源释放
	        if (qCursor != null) {
	            qCursor.close();
	        }
	        if (cancellationSignal != null) {
	            cancellationSignal.setRemote(null);
	        }
	        if (unstableProvider != null) {
	            releaseUnstableProvider(unstableProvider);
	        }
	        if (stableProvider != null) {
	            releaseProvider(stableProvider);
	        }
	    }
	}

ApplicationContentResolver#query分为2个主要步骤：`获取Provider`和`query查询`.  

## 5.获取ContentProvider

### 5.1 ApplicationContentResolver.acquireUnstableProvider  
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

	// 根据Uri查询ContentProvider
	public final IContentProvider acquireUnstableProvider(Uri uri) {
	    // uri必须以content开头
	    if (!SCHEME_CONTENT.equals(uri.getScheme())) {
	        return null;
	    }
 
	    // 获取authority(全格式为host:port)
	    String auth = uri.getAuthority();
	    if (auth != null) {
	        // 走到实现类ApplicationContentResolver#acquireUnstableProvider
	        // 最终走到ActivityThread.acquireProvider
	        return acquireUnstableProvider(mContext, uri.getAuthority());
	    }
	    return null;
	}

### 5.2 ActivityThread
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

#### 5.2.1 acquireProvider

	public final IContentProvider acquireProvider(Context c, String auth, int userId, boolean stable) {
	    // 先从map缓存中查询本地ContentProvider. 参考【第5.2.2节】
	    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
	    if (provider != null) {
	        return provider;
	    }
	    // IActivity.Stub.Proxy#getContentProvider获取ContentProviderHolder. 参考【第5.3节】
	    ContentProviderHolder holder = null;
	    synchronized (getGetProviderLock(auth, userId)) {
	        holder = ActivityManager.getService().getContentProvider(getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
	    }
 
	    if (holder == null) {
	        return null;
	    }
	    // 增加引用计数, 同时删除老的provider
	    holder = installProvider(c, holder, holder.info, true /*noisy*/, holder.noReleaseNeeded, stable);
 
	    // ContentProviderProxy
	    return holder.provider;
	}

获取ContentProvider过程：  
1.先根据ProviderKey从本地缓存中查询, 若查询到则直接返回  
2.若本地缓存没有, 则通过Binder通信找AMS获取    

其中ProviderKey由authority和userId组成. 它们的获取逻辑为：  
[ -> frameworks/base/core/java/android/content/ContentProvider.java ]

	// 例如userId@some.authority, 返回值为some.authority 
	public static String getAuthorityWithoutUserId(String auth) {
	    if (auth == null) return null;
	    int end = auth.lastIndexOf('@');
	    return auth.substring(end+1);
	}
 
	// 例如userId@some.authority, 返回值为userId
	public static int getUserIdFromAuthority(String auth, int defaultUserId) {
	    if (auth == null) return defaultUserId;
	    int end = auth.lastIndexOf('@');
	    if (end == -1) return defaultUserId;
	    String userIdString = auth.substring(0, end);
	    try {
	        return Integer.parseInt(userIdString);
	    } catch (NumberFormatException e) {
	        return UserHandle.USER_NULL;
	    }
	}

#### 5.2.2 acquireExistingProvider

	// 从缓存查询ContentProvider
	public final IContentProvider acquireExistingProvider(Context c, String auth, int userId, boolean stable) {
	    synchronized (mProviderMap) {
		    // 根据authority和userId封装ProviderKey对象
	        final ProviderKey key = new ProviderKey(auth, userId);
	        // 根据ProviderKey查询ProviderClientRecord
	        final ProviderClientRecord pr = mProviderMap.get(key);
	        if (pr == null) {
	            return null;
	        }
 
			// 拿到ContentProviderProxy
	        IContentProvider provider = pr.mProvider;
	        // 拿到mRemote-Transport(即ContentProviderNative实现类)
	        IBinder jBinder = provider.asBinder();

	        if (!jBinder.isBinderAlive()) {
	            // 若mRemote所在进程已经死掉, 则从mProviderMap和mProviderRefCountMap中移除该provider
	            // 及引用计数, 并告诉AMS该provider不可用
	            handleUnstableProviderDiedLocked(jBinder, true);
	            return null;
	        }
 
	        // 获取provider引用计数, 引用计数+1
	        ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
	        if (prc != null) {
		        // 参考【第5.2.3节】
	            incProviderRefLocked(prc, stable);
	        }
	        return provider;
	    }
	}

#### 5.2.3 incProviderRefLocked

	private final void incProviderRefLocked(ProviderRefCount prc, boolean stable) {
	    if (stable) {
	        prc.stableCount += 1;
	        if (prc.stableCount == 1) {
	            int unstableDelta;
	            if (prc.removePending) {
	                unstableDelta = -1;
	                prc.removePending = false;
	                mH.removeMessages(H.REMOVE_PROVIDER, prc);
	            } else {
	                unstableDelta = 0;
	            }
	            ActivityManager.getService().refContentProvider(prc.holder.connection, 1, unstableDelta);
	        }
	    } else {
	        prc.unstableCount += 1;
	        if (prc.unstableCount == 1) {
	            if (prc.removePending) {
	                prc.removePending = false;
	                mH.removeMessages(H.REMOVE_PROVIDER, prc);
	            } else {
	                ActivityManager.getService().refContentProvider(prc.holder.connection, 0, 1);
	            }
	        }
	    }
	}

若本地缓存没有查找到, 则通过Binder通信找AMS获取.  
下面主要分析`ActivityManager.getService().getContentProvider(...)` 过程.  

之前文章 **[AIDL过程](http://mouxuejie.com/blog/2020-02-08/aidl/)** 已经分析过ActivityManager.getService()拿到IActivityManager.Stub.Proxy对象.   
在那篇文章是以startService为例, AMP向AMS发送启动服务的请求, 不关注返回值. 本文是getContentProvider, 是AMP向AMS发送获取ContentProvider对象的请求, AMS会向AMP返回ContentProvider.   
那么通过AMP.getContentProvider(...)获取到的provider究竟是什么呢? 我们通过下面的分析会知道是ContentProviderProxy.   

### 5.3 AMP.getContentProvider

#### 5.3.1 IActivityManager.aidl

	interface IActivityManager {
	    ...
	    ContentProviderHolder getContentProvider(in IApplicationThread caller,
			    in String callingPackage, in String name, int userId, boolean stable);
	    ...
	}

IActivityManager.aidl文件编译后的Java产物为:  

	public interface IActivityManager extends android.os.IInterface
	{
	    /** Local-side IPC implementation stub class. */
	    public static abstract class Stub extends android.os.Binder
	            implements android.app.IActivityManager
	    {
	        private static final java.lang.String DESCRIPTOR = "android.app.IActivityManager";
	        /** Construct the stub at attach it to the interface. */
	        public Stub()
	        {
	            this.attachInterface(this, DESCRIPTOR);
	        }
 
	         // Cast an IBinder object into an android.app.IActivityManager interface,
	         // generating a proxy if needed.
	        public static android.app.IActivityManager asInterface(android.os.IBinder obj)
	        {
	            if ((obj==null)) {
	                return null;
	            }
	            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	            if (((iin!=null)&&(iin instanceof android.app.IActivityManager))) {
	                return ((android.app.IActivityManager)iin);
	            }
	            return new android.app.IActivityManager.Stub.Proxy(obj);
	        }
  
	        @Override 
	        public android.os.IBinder asBinder()
	        {
	            return this;
	        }
  
	        @Override 
	        public boolean onTransact(int code, android.os.Parcel data,
			        android.os.Parcel reply, int flags) throws android.os.RemoteException
	        {
	            java.lang.String descriptor = DESCRIPTOR;
	            switch (code)
	            {
	                case INTERFACE_TRANSACTION:
	                {
	                    reply.writeString(descriptor);
	                    return true;
	                }
	                case TRANSACTION_getContentProvider:
	                {
	                    data.enforceInterface(descriptor);
	                    IApplicationThread _arg0 = (IApplicationThread) data.readStrongBinder();
	                    java.lang.String _arg1 = (java.lang.String) data.readString();
	                    java.lang.String _arg2 = data.readString();
	                    int _arg3 = data.readInt();
	                    boolean _arg4 = data.readBoolean();
	                    android.app.ContentProviderHolder _result = this.getContentProvider(_arg0, _arg1, _arg2, _arg3, _arg4);
	                    reply.writeNoException();
	                    if ((_result!=null)) {
	                        reply.writeInt(1);
	                        // 会走到ContentProviderHolder.writeToParcel, reply中写进的provider是Transport
	                        _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
	                    } else {
	                        reply.writeInt(0);
	                    }
	                    return true;
	                }
	                default:
	                {
	                    return super.onTransact(code, data, reply, flags);
	                }
	            }
	        }
  
	        private static class Proxy implements android.app.IActivityManager
	        {
	            private android.os.IBinder mRemote;
	            Proxy(android.os.IBinder remote)
	            {
	                mRemote = remote;
	            }
	            @Override
	            public android.os.IBinder asBinder()
	            {
	                return mRemote;
	            }
	            public java.lang.String getInterfaceDescriptor()
	            {
	                return DESCRIPTOR;
	            }
  
	            @Override
	            public android.app.ContentProviderHolder getContentProvider(
	                    in IApplicationThread caller, java.lang.String callingPackage,
	                    java.lang.String name, int userId,
	                    boolean stable) throws android.os.RemoteException
	            {
	                android.os.Parcel _data = android.os.Parcel.obtain();
	                android.os.Parcel _reply = android.os.Parcel.obtain();
	                android.app.ContentProviderHolder _result;
	                try {
	                    _data.writeInterfaceToken(DESCRIPTOR);
	                    _data.writeStrongBinder(caller);
	                    _data.writeString(callingPackage);
	                    _data.writeString(name);
	                    _data.writeInt(userId);
	                    _data.writeBoolean(stable);
	                    mRemote.transact(Stub.TRANSACTION_getContentProvider, _data, _reply, 0);
	                    _reply.readException();
	                    if ((0!=_reply.readInt())) {
	                        // 最终ContentProviderHolder中读取出来的provider是ContentProviderProxy
	                        _result = android.app.ContentProviderHolder.CREATOR.createFromParcel(_reply);
	                    } else {
	                        _result = null;
	                    }
	                } finally {
	                    _reply.recycle();
	                    _data.recycle();
	                }
	                return _result;
	            }
	        }
  
	        static final int TRANSACTION_getContentProvider = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
	    }
  
	    public android.app.ContentProviderHolder getContentProvider(
	            in IApplicationThread caller, java.lang.String callingPackage,
	            java.lang.String name, int userId,
	            boolean stable) throws android.os.RemoteException;
	}

在文章 **[AIDL过程](http://mouxuejie.com/blog/2020-02-08/aidl/)** 已经分析过IActivityManager.aidl的调用逻辑：    
	
	IActivityManager.Stub.Proxy#getContentProvider -> mRemote.transact ->
	IActivityManager.Stub#onTransact -> IActivityManager#getContentProvider(AMS端) ->
	ContentViewHolder.writeToParcel(AMS端) -> ContentProviderHolder.CREATOR.createFromParcel(AMP端)

我们再看AMS.getContentProvider过程, 那么它返回的provider是什么呢?

#### 5.3.2 AMS.getContentProvider  
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	@Override
	public final ContentProviderHolder getContentProvider(
	        IApplicationThread caller, String callingPackage, String name, int userId,
	        boolean stable) {
	    // 不允许isolate进程调用
	    enforceNotIsolatedCaller("getContentProvider");
	    ...
	    final int callingUid = Binder.getCallingUid();
	    return getContentProviderImpl(caller, name, null, callingUid, callingPackage, null, stable, userId);
	}

	private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
	        String name, IBinder token, int callingUid, String callingPackage, 
	        String callingTag, boolean stable, int userId) {
	    ContentProviderRecord cpr;
	    ContentProviderConnection conn = null;
	    ProviderInfo cpi = null;
	    boolean providerRunning = false;
 
	    synchronized(this) {
	        long startTime = SystemClock.uptimeMillis();
 
	        ProcessRecord r = null;
	        if (caller != null) {
	            r = getRecordForAppLocked(caller);
	        }
 
	        boolean checkCrossUser = true;
 
	        // 根据name获取ContentProviderRecord对象, 若普通用户未获取到, 则从系统用户获取
	        cpr = mProviderMap.getProviderByName(name, userId);
	        if (cpr == null && userId != UserHandle.USER_SYSTEM) {
	            cpr = mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
	            if (cpr != null) {
	                cpi = cpr.info;
	                if (isSingleton(cpi.processName, cpi.applicationInfo, cpi.name, cpi.flags)
	                        && isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
	                    userId = UserHandle.USER_SYSTEM;
	                    checkCrossUser = false;
	                } else {
	                    cpr = null;
	                    cpi = null;
	                }
	            }
	        }
 
	        // 检查ContentProviderRecord所在进程是否被杀, 若进程被杀, 执行数据清理操作
	        if (cpr != null && cpr.proc != null) {
		        // 检查provider是否正在运行
	            providerRunning = !cpr.proc.killed;
 
	            if (cpr.proc.killed && cpr.proc.killedByAm) {
	                appDiedLocked(cpr.proc);
	            }
	        }
 
			// 若provider正在运行
	        if (providerRunning) {
	            cpi = cpr.info;
	            String msg;
 
	            if (r != null && cpr.canRunHere(r)) {
	                // 权限检查
	                ...
 
	                ContentProviderHolder holder = cpr.newHolder(null);
	                holder.provider = null;
	                return holder;
	            }
 
	            // IPackageManager.Stub.Proxy#resolveContentProvider. 
		        // Binder通信PMP请求PMS解析出ContentProvider, 即PMS.resolveContentProvider
		        // 而PMS又通过ComponentResolver组件解析器解析出ProviderInfo. 
		        // 对于联系人ProviderInfo中provider名称是ContactsProvider2, 参考【第5.3.3节】
	            if (AppGlobals.getPackageManager().resolveContentProvider(name, 0 /*flags*/, userId) == null) {
	                return null;
	            }
 
	            final long origId = Binder.clearCallingIdentity();
 
	            // ContentProviderRecord计数+1
	            conn = incProviderCountLocked(r, cpr, token, callingUid, callingPackage, callingTag, stable);
	            if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
	                if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_LOW_APP_ADJ) {
	                    // 更新lru进程信息
	                    mProcessList.updateLruProcessLocked(cpr.proc, false, null);
	                }
	            }
 
	            final int verifiedAdj = cpr.proc.verifiedAdj;
	            boolean success = updateOomAdjLocked(cpr.proc, true, OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
	            if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
	                success = false;
	            }
	            if (!success) {
	                boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
	                appDiedLocked(cpr.proc);
	                if (!lastRef) {
	                    return null;
	                }
	                providerRunning = false;
	                conn = null;
	            } else {
	                cpr.proc.verifiedAdj = cpr.proc.setAdj;
	            }
	            Binder.restoreCallingIdentity(origId);
	        }
 
			// 若provider不在运行
	        if (!providerRunning) {
	        	// IPackageManager.Stub.Proxy#resolveContentProvider. 
		        // Binder通信PMP请求PMS解析出ContentProvider, 即PMS.resolveContentProvider
		        // 而PMS又通过ComponentResolver组件解析器解析出ProviderInfo. 
		        // 对于联系人ProviderInfo中provider名称是ContactsProvider2, 参考【第5.3.3节】
	            cpi = AppGlobals.getPackageManager().resolveContentProvider(name, STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);

	            if (cpi == null) {
	                return null;
	            }
	            // If the provider is a singleton AND
	            // (it's a call within the same user || the provider is a
	            // privileged app)
	            // Then allow connecting to the singleton provider
	            boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo, cpi.name, cpi.flags)
                    && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
	            if (singleton) {
	                userId = UserHandle.USER_SYSTEM;
	            }
	            cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);
 
	            // 各种有效性检查, 包括:系统是否ready
	            ...
 
	            // 根据ComponentName从mProviderMap中查找, 若没找到则新建
	            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
	            cpr = mProviderMap.getProviderByClass(comp, userId);
	            final boolean firstClass = cpr == null;
	            if (firstClass) {
	                try {
	                    ApplicationInfo ai = AppGlobals.getPackageManager().getApplicationInfo(
			                    cpi.applicationInfo.packageName, STOCK_PM_FLAGS, userId);
	                    ai = getAppInfoForUser(ai, userId);
	                    cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
	                } catch (RemoteException ex) {
	                    // pm is in same process, this will never happen.
	                } finally {
	                    Binder.restoreCallingIdentity(ident);
	                }
	            }
 
	            // 如果是单进程, 或provider允许多进程多个实例
	            if (r != null && cpr.canRunHere(r)) {
	                return cpr.newHolder(null);
	            }
 
	            // 检查当前ContentProviderRecord是否已启动
	            final int N = mLaunchingProviders.size();
	            int i;
	            for (i = 0; i < N; i++) {
	                if (mLaunchingProviders.get(i) == cpr) {
	                    break;
	                }
	            }
 
	            // 若当前ContentProviderRecord未启动
	            if (i >= N) {
	                final long origId = Binder.clearCallingIdentity();
	                try {
	                    // Content provider is now in use, its package can't be stopped.
	                    AppGlobals.getPackageManager().setPackageStoppedState(cpr.appInfo.packageName, false, userId);
 
	                    // 获取当前ContentProvider所在进程
	                    ProcessRecord proc = getProcessRecordLocked(cpi.processName, cpr.appInfo.uid, false);
	                    if (proc != null && proc.thread != null && !proc.killed) {
	                        // 若进程正在运行中, 且cpi还未发布, 则将其加到map中并发布Provider
	                        if (!proc.pubProviders.containsKey(cpi.name)) {
	                            proc.pubProviders.put(cpi.name, cpr);
	                            // 安装和发布provider
	                            proc.thread.scheduleInstallProvider(cpi);
	                        }
	                    } else {
	                        // 否则启动新进程
	                        proc = startProcessLocked(cpi.processName,
	                                cpr.appInfo, false, 0,
	                                new HostingRecord("content provider",
	                                new ComponentName(cpi.applicationInfo.packageName, cpi.name)), false, false, false);
	                    }
	                    // cpr添加到mLaunchingProviders列表
	                    cpr.launchingApp = proc;
	                    mLaunchingProviders.add(cpr);
	                } finally {
	                    Binder.restoreCallingIdentity(origId);
	                }
	            }
 
	            // ContentProviderRecord对象存到mProviderMap
	            if (firstClass) {
	                mProviderMap.putProviderByClass(comp, cpr);
	            }
	            mProviderMap.putProviderByName(name, cpr);
 
	            // provider引用+1
	            conn = incProviderCountLocked(r, cpr, token, callingUid, callingPackage, callingTag, stable);
	            if (conn != null) {
	                conn.waiting = true;
	            }
	        }
	    }
	    // 等待provider被发布, 超时时间20秒
	    final long timeout = SystemClock.uptimeMillis() + CONTENT_PROVIDER_WAIT_TIMEOUT;
	    boolean timedOut = false;
	    synchronized (cpr) {
	        while (cpr.provider == null) {
	            try {
	                final long wait = Math.max(0L, timeout - SystemClock.uptimeMillis());
	                if (conn != null) {
	                    conn.waiting = true;
	                }
	                cpr.wait(wait);
	                if (cpr.provider == null) {
	                    timedOut = true;
	                    break;
	                }
	            } catch (InterruptedException ex) {
	            } finally {
	                if (conn != null) {
	                    conn.waiting = false;
	                }
	            }
	        }
	    }
 
	    // 超时返空
	    if (timedOut) {
	        return null;
	    }
 
	    // 返回ContentProviderHolder. 持有的provider是ContactsProvider2(可理解为Transport)
	    return cpr.newHolder(conn);
	}

AMS.getContentProvider(String name, int userId)总结:  
1.根据name从mProviderMap中查找ContentProviderRecord对象cpr, 若普通用户未获取到, 则从系统用户获取  
2.如果cpr非空, 则检查cpr所在进程是否被杀, 若进程被杀, 执行数据清理操作. 检查provider是否正在运行       
3.若provider正在运行:   
(1) 通过IPackageManger.Stub.Proxy#resolveContentProvider解析出ProviderInfo, 且cpr引用计数+1, 并更新lru进程信息  
4.若provider不在运行:  
(1) 通过IPackageManger.Stub.Proxy#resolveContentProvider解析出ProviderInfo  
(2) 根据ProviderInfo封装ComponentName, 并从mProviderMap中查找cpr, 若没找到则新建一个  
(3) 如果当前是单进程, 或provider允许多进程多个实例, 则直接返回  
(4) 检查cpr是否在mLaunchingProviders列表, 若不在, 则检查cpr所在进程是否正在运行.  
(5) 若cpr所在进程正在运行, 但ProviderInfo对应的provider还未发布, 则将ProviderInfo加到map中, 执行ApplicationThread#scheduleInstallProvider发布Provider  
(6) 若cpr所在进程不在运行, 则启动新进程. cpr添加到mLaunchingProviders和mProviderMap, cpr引用计数+1, 等待provider被发布, 超时时间20秒, 若超时返空   
5.返回ContentProviderHolder. 此时持有的provider是ContactsProvider2(可理解为Transport)     

#### 5.3.3 PMS.resolveContentProvider
[ -> frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java ]

	@Override
	public ProviderInfo resolveContentProvider(String name, int flags, int userId) {
	    return resolveContentProviderInternal(name, flags, userId);
	}
 
	private ProviderInfo resolveContentProviderInternal(String name, int flags, int userId) {
	    if (!sUserManager.exists(userId)) return null;
	    flags = updateFlagsForComponent(flags, userId, name);
	    final int callingUid = Binder.getCallingUid();
		// 组件解析器解析ComponentResolver.queryProvider. 参考【第5.3.4节】
	    final ProviderInfo providerInfo = mComponentResolver.queryProvider(name, flags, userId);
	    if (providerInfo == null) {
	        return null;
	    }
	    if (!mSettings.isEnabledAndMatchLPr(providerInfo, flags, userId)) {
	        return null;
	    }
	    synchronized (mPackages) {
	        final PackageSetting ps = mSettings.mPackages.get(providerInfo.packageName);
	        final ComponentName component = new ComponentName(providerInfo.packageName, providerInfo.name);
	        if (filterAppAccessLPr(ps, callingUid, component, TYPE_PROVIDER, userId)) {
	            return null;
	        }
	        return providerInfo;
	    }
	}

我们知道PMS.resolveContentProvider最终会拿到ContactsProvider2.  那么它是怎么拿到的呢? 我们继续往下看这个过程.

#### 5.3.4 ComponentResolver.queryProvider
[ -> frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java ]

	ProviderInfo queryProvider(String authority, int flags, int userId) {
	    synchronized (mLock) {
	        final PackageParser.Provider p = mProvidersByAuthority.get(authority);
	        if (p == null) {
	            return null;
	        }
	        final PackageSetting ps = (PackageSetting) p.owner.mExtras;
	        if (ps == null) {
	            return null;
	        }
	        return PackageParser.generateProviderInfo(p, flags, ps.readUserState(userId), userId);
	    }
	}

现在分析下mProvidersByAuthority的取值.   
根据赋值的地方反查可得出如下调用链:  

	PMS: PackageHandler.INIT_COPY -> startCopy -> handleReturnCode -> processPendingInstall ->
		 processInstallRequestsAsync -> installPackagesLI -> commitPackagesLocked ->
		 commitReconciledScanResultLocked -> commitPackageSettings
	ComponentResolver: addAllComponents -> addProvidersLocked

我们再看下ComponentResolver#addProvidersLocked实现.

	private void addProvidersLocked(PackageParser.Package pkg, boolean chatty) {
	    final int providersSize = pkg.providers.size();
	    StringBuilder r = null;
	    for (int i = 0; i < providersSize; i++) {
	        PackageParser.Provider p = pkg.providers.get(i);
	        p.info.processName = fixProcessName(pkg.applicationInfo.processName, p.info.processName);
	        mProviders.addProvider(p);
	        p.syncable = p.info.isSyncable;
	        if (p.info.authority != null) {
	            String[] names = p.info.authority.split(";");
	            p.info.authority = null;
	            for (int j = 0; j < names.length; j++) {
	                if (j == 1 && p.syncable) {
	                    p = new PackageParser.Provider(p);
	                    p.syncable = false;
	                }
	                if (!mProvidersByAuthority.containsKey(names[j])) {
	                    mProvidersByAuthority.put(names[j], p);
	                    if (p.info.authority == null) {
	                        p.info.authority = names[j];
	                    } else {
	                        p.info.authority = p.info.authority + ";" + names[j];
	                    }
	                } else {
	                    final PackageParser.Provider other = mProvidersByAuthority.get(names[j]);
	                    final ComponentName component =
	                            (other != null && other.getComponentName() != null)
	                                    ? other.getComponentName() : null;
	                    final String packageName = component != null ? component.getPackageName() : "?";
	                }
	            }
	        }
	    }
	}

其中有个很重要的角色包解析器`PackageParser`, 它会在App安装过程中解析各种文件包括AndroidManifest.xml  
解析出来的信息会封装在`PackageParser.Package`对象中.

我们再看下Contacts应用程序的providers包下的AndroidManifest.xml文件.  
[ -> packages/providers/ContactsProvider/AndroidManifest.xml ]  

	...
	<application android:process="android.process.acore"
	    android:label="@string/app_label"
	    android:icon="@drawable/app_icon"
	    android:allowBackup="false"
	    android:usesCleartextTraffic="false">
 
	    <provider android:name="ContactsProvider2"
	        android:authorities="contacts;com.android.contacts"
	        android:label="@string/provider_label"
	        android:multiprocess="false"
	        android:exported="true"
	        android:grantUriPermissions="true"
	        android:readPermission="android.permission.READ_CONTACTS"
	        android:writePermission="android.permission.WRITE_CONTACTS"
	        android:visibleToInstantApps="true">
	        <path-permission
                android:pathPrefix="/search_suggest_query"
                android:readPermission="android.permission.GLOBAL_SEARCH" />
	        <path-permission
                android:pathPrefix="/search_suggest_shortcut"
                android:readPermission="android.permission.GLOBAL_SEARCH" />
	        <path-permission
                android:pathPattern="/contacts/.*/photo"
                android:readPermission="android.permission.GLOBAL_SEARCH" />
	        <grant-uri-permission android:pathPattern=".*" />
	    </provider>
	    ...
	</application>
	...

mProvidersByAuthority最终会存储key为contacts和com.android.contacts, value为ContactsProvider2的信息.
因此我们得出结论, PMS.resolveContentProvider拿到的是ContactsProvider2对应的ProviderInfo.  

那么经过AIDL Binder通信, AMP.getContentProvider拿到的究竟是什么呢?  
我们再看下ContentProviderHolder的实现:  

#### 5.3.5 ContentProviderHolder  
[ -> frameworks/base/core/java/android/app/ContentProviderHolder.java ]

	public class ContentProviderHolder implements Parcelable {
	    public final ProviderInfo info;
	    public IContentProvider provider;
	    public IBinder connection;
	    public boolean noReleaseNeeded;
 
	    public ContentProviderHolder(ProviderInfo _info) {
	        info = _info;
	    }
 
	    @Override
	    public int describeContents() {
	        return 0;
	    }
 
	    @Override
	    public void writeToParcel(Parcel dest, int flags) {
	        info.writeToParcel(dest, 0);
	        if (provider != null) {
	            // 写入的是Transport
	            dest.writeStrongBinder(provider.asBinder());
	        } else {
	            dest.writeStrongBinder(null);
	        }
	        dest.writeStrongBinder(connection);
	        dest.writeInt(noReleaseNeeded ? 1 : 0);
	    }
 
	    public static final Parcelable.Creator<ContentProviderHolder> CREATOR
	            = new Parcelable.Creator<ContentProviderHolder>() {
	        @Override
	        public ContentProviderHolder createFromParcel(Parcel source) {
	            return new ContentProviderHolder(source);
	        }
 
	        @Override
	        public ContentProviderHolder[] newArray(int size) {
	            return new ContentProviderHolder[size];
	        }
	    };
 
	    private ContentProviderHolder(Parcel source) {
	        info = ProviderInfo.CREATOR.createFromParcel(source);
	        // 拿到的provider是ContentProviderProxy
	        provider = ContentProviderNative.asInterface(source.readStrongBinder());
	        connection = source.readStrongBinder();
	        noReleaseNeeded = source.readInt() != 0;
	    }
	}

结合上面的IActivityManager.aidl分析过程中的调用链:  

	IActivityManager.Stub.Proxy#getContentProvider -> mRemote.transact ->
	IActivityManager.Stub#onTransact -> IActivityManager#getContentProvider(AMS端) ->
	ContentViewHolder.writeToParcel(AMS端) -> ContentProviderHolder.CREATOR.createFromParcel(AMP端)

我们已经知道了AMS.getContentProvider拿到的是ContactsProvider2(可理解为Transport和ContentProviderNative)  
结合ContentProviderHolder源码可知, ContentViewHolder.writeToParcel(AMS端)写入的是Transport,   ContentProviderHolder.CREATOR.createFromParcel(AMP端)拿到的是ContentProviderProxy.  

**最终ActivityManager.getService().getContentProvider(...) 拿到的是ContentProviderProxy**

同时只有provider先发布, 才能获取到provider. 发布逻辑主要在ApplicationThread.scheduleInstallProvider

## 6.安装&发布Provider

### 6.1 ApplicationThread  
[ -> frameworks/base/core/java/android/app/ActivityThread.java ]

#### 6.1.1 scheduleInstallProvider

	private class ApplicationThread extends IApplicationThread.Stub {
	    @Override
	    public void scheduleInstallProvider(ProviderInfo provider) {
	        sendMessage(H.INSTALL_PROVIDER, provider);
	    }
	}

#### 6.1.2 H.INSTALL_PROVIDER

	class H extends Handler {
     
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            ...
	            case INSTALL_PROVIDER:
	                handleInstallProvider((ProviderInfo) msg.obj);
	                break;
	            ...
	        }
	    }
	}

#### 6.1.3 handleInstallProvider

	public void handleInstallProvider(ProviderInfo info) {
	    installContentProviders(mInitialApplication, Arrays.asList(info));
	}

#### 6.1.4 installContentProviders

	private void installContentProviders(Context context, List<ProviderInfo> providers) {
	    final ArrayList<ContentProviderHolder> results = new ArrayList<>();
 
	    for (ProviderInfo cpi : providers) {
	        // 安装ContentProvider
	        // 新建ContentProvider并设置上下文Context
	        // 将Provider本地对象封装到ContentProviderRecord中并添加到各map中
	        ContentProviderHolder cph = installProvider(context, null, cpi, false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
	        if (cph != null) {
	            cph.noReleaseNeeded = true;
	            results.add(cph);
	        }
	    }
 
		// 发布ContentProvider. AMP#publishContentProviders(向AMS发送Binder请求)
	    // 主要就是将进程中pubProviders的provider添加到相关map, 并设置provider和进程等信息
	    // 然后通过notifyAll通知provider发布完成.
	    ActivityManager.getService().publishContentProviders(getApplicationThread(), results);
	}

ApplicationThread.installContentProviders()过程分为2个过程:  
1.installProvider:  安装Provider  
2.publishContentProviders: 发布Provider  

### 6.2 安装Provider

#### 6.2.1 ApplicationThread.installProvider

	// 安装Provider.
	// 进程本地或SystemServer进程的provider是永久安装不释放的(noReleaseNeeded为true), 没有引用计数. 其他远程provider是引用计数, 初始计数为1
	// 当provider安装时, 需要增加引用计数(若支持引用计数)并返回provider
	private ContentProviderHolder installProvider(Context context,
	        ContentProviderHolder holder, ProviderInfo info,
	        boolean noisy, boolean noReleaseNeeded, boolean stable) {
	    // localProvider指ContentProvider(联系人为ContactsProvider2), provider指Transport-ContentProviderNative实现类
	    ContentProvider localProvider = null;
	    IContentProvider provider;
	    if (holder == null || holder.provider == null) {
	        Context c = null;
	        ApplicationInfo ai = info.applicationInfo;
	        if (context.getPackageName().equals(ai.packageName)) {
	            c = context;
	        } else if (mInitialApplication != null && mInitialApplication.getPackageName().equals(ai.packageName)) {
	            c = mInitialApplication;
	        } else {
	            c = context.createPackageContext(ai.packageName, Context.CONTEXT_INCLUDE_CODE);
	        }
  
	        if (info.splitName != null) {
	            c = c.createContextForSplit(info.splitName);
	        }
  
	        final java.lang.ClassLoader cl = c.getClassLoader();
	        LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
	        if (packageInfo == null) {
	            // System startup case.
	            packageInfo = getSystemContext().mPackageInfo;
	        }
	        // 新建ContentProvider(联系人为ContactsProvider2), 并初始化
	        // 参考【第5.3.3节】PMS.resolveContentProvider可知, ProviderInfo的name是ContactsProvider2
	        localProvider = packageInfo.getAppFactory().instantiateProvider(cl, info.name);
	        // 返回Transport(ContentProviderNative的实现类)
	        provider = localProvider.getIContentProvider();
	        localProvider.attachInfo(c, info);
	    } else {
	        // 安装外部Provider
	        provider = holder.provider;
	    }
  
	    ContentProviderHolder retHolder;
  
	    synchronized (mProviderMap) {
	        // 返回ContentProvider的本地对象Transport(也是一个Binder)
	        IBinder jBinder = provider.asBinder();
	        if (localProvider != null) {
	            ComponentName cname = new ComponentName(info.packageName, info.name);
	            ProviderClientRecord pr = mLocalProvidersByName.get(cname);
	            if (pr != null) {
	                // 使用本地provider
	                provider = pr.mProvider;
	            } else {
	                // ContentProviderHolder是Parcelable
	                // 写入ContentProviderHolder的provider是Transport, 经过Binder通信传输后, 拿到的provider是ContentProviderProxy对象
	                holder = new ContentProviderHolder(info);
	                holder.provider = provider;
	                holder.noReleaseNeeded = true;
	                // 新建ProviderClientRecord, 并将其添加到各个map中
	                pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
	                mLocalProviders.put(jBinder, pr);
	                mLocalProvidersByName.put(cname, pr);
	            }
	            retHolder = pr.mHolder;
	        } else {
	            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
	            if (prc != null) {
	                // provider引用+1, 同时释放老的provider
	                // We need to transfer our new reference to the existing
	                // ref count, releasing the old one...  but only if
	                // release is needed (that is, it is not running in the
	                // system process).
	                if (!noReleaseNeeded) {
	                    incProviderRefLocked(prc, stable);
	                    ActivityManager.getService().removeContentProvider(holder.connection, stable);
	                }
	            } else {
	                ProviderClientRecord client = installProviderAuthoritiesLocked(provider, localProvider, holder);
	                if (noReleaseNeeded) {
	                    prc = new ProviderRefCount(holder, client, 1000, 1000);
	                } else {
	                    prc = stable ? new ProviderRefCount(holder, client, 1, 0) : new ProviderRefCount(holder, client, 0, 1);
	                }
	                mProviderRefCountMap.put(jBinder, prc);
	            }
	            retHolder = prc.holder;
	        }
	    }
	    return retHolder;
	}

#### 6.2.2 ApplicationThread.installProviderAuthoritiesLocked

	// 添加到mProviderMap中(key为ProviderKey, value为ProviderClientRecord)
	// ProviderKey包含元素authrity和userId, ProviderClientRecord包含元素auths/provider/localProvider/holder
	private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
	        ContentProvider localProvider, ContentProviderHolder holder) {
	    final String auths[] = holder.info.authority.split(";");
	    final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid);
 
	    if (provider != null) {
	        // If this provider is hosted by the core OS and cannot be upgraded,
	        // then I guess we're okay doing blocking calls to it.
	        for (String auth : auths) {
	            switch (auth) {
	                case ContactsContract.AUTHORITY:
	                case CallLog.AUTHORITY:
	                case CallLog.SHADOW_AUTHORITY:
	                case BlockedNumberContract.AUTHORITY:
	                case CalendarContract.AUTHORITY:
	                case Downloads.Impl.AUTHORITY:
	                case "telephony":
	                    Binder.allowBlocking(provider.asBinder());
	            }
	        }
	    }
 
	    final ProviderClientRecord pcr = new ProviderClientRecord(auths, provider, localProvider, holder);
	    for (String auth : auths) {
	        final ProviderKey key = new ProviderKey(auth, userId);
	        final ProviderClientRecord existing = mProviderMap.get(key);
	        if (existing == null) {
	            mProviderMap.put(key, pcr);
	        }
	    }
	    return pcr;
	}

### 6.3 发布Provider 
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

#### 6.3.1 AMS.publishContentProviders

	// 遍历ContentProviderHolder列表, 从调用进程的pubProviders map中查找是否包含指定名称的ContentProviderRecord
	// 若存在则将其存到相关map中, 并检查是否在launching过程中, 若是则将其从mLaunchingProviders列表移除, 同时杀死provider所在进程
	// 否则设置provider和进程信息后, notifyAll通知provider发布完成.
	public final void publishContentProviders(IApplicationThread caller, List<ContentProviderHolder> providers) {
	    if (providers == null) {
	        return;
	    }
 
	    enforceNotIsolatedCaller("publishContentProviders");
	    synchronized (this) {
	        // 获取调用进程
	        final ProcessRecord r = getRecordForAppLocked(caller);
 
	        final int N = providers.size();
	        for (int i = 0; i < N; i++) {
	            ContentProviderHolder src = providers.get(i);
	            if (src == null || src.info == null || src.provider == null) {
	                continue;
	            }
	            // 以name为key, 从当前进程发布的provider map中查找ContentProviderRecord对象
	            ContentProviderRecord dst = r.pubProviders.get(src.info.name);
	            if (dst != null) {
	                // 存到map中
	                ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
	                mProviderMap.putProviderByClass(comp, dst);
	                String names[] = dst.info.authority.split(";");
	                for (int j = 0; j < names.length; j++) {
	                    mProviderMap.putProviderByName(names[j], dst);
	                }
 
	                // 检查当前ContentProviderRecord是否在mLaunchingProviders列表, 若在则从列表移除
	                int launchingCount = mLaunchingProviders.size();
	                int j;
	                boolean wasInLaunchingProviders = false;
	                for (j = 0; j < launchingCount; j++) {
	                    if (mLaunchingProviders.get(j) == dst) {
	                        mLaunchingProviders.remove(j);
	                        wasInLaunchingProviders = true;
	                        j--;
	                        launchingCount--;
	                    }
	                }
	                // 若在mLaunchingProviders列表, 则发消息杀死该provider所在的进程
	                if (wasInLaunchingProviders) {
	                    mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
	                }
 
	                r.addPackage(dst.info.applicationInfo.packageName, dst.info.applicationInfo.longVersionCode, mProcessStats);
	                // 设置provider和进程, 并通知provider发布完成
	                synchronized (dst) {
	                    dst.provider = src.provider;
	                    dst.setProcess(r);
	                    dst.notifyAll();
	                }
	                // 更新oomAdj
	                updateOomAdjLocked(r, true, OomAdjuster.OOM_ADJ_REASON_GET_PROVIDER);
	            }
	        }
	    }
	}

#### 6.3.2 MainHandler.CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG

	final class MainHandler extends Handler {
	    public MainHandler(Looper looper) {
	        super(looper, null, true);
	    }
 
	    @Override
	    public void handleMessage(Message msg) {
	        switch (msg.what) {
	            ...
	            case CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG: {
	                ProcessRecord app = (ProcessRecord)msg.obj;
	                synchronized (ActivityManagerService.this) {
	                    // 如果当前provider在launching过程中, 则杀死provider所在的进程
	                    processContentProviderPublishTimedOutLocked(app);
	                }
	            } break;
	            ...
	        }
	    }
	}
 
	private final void processContentProviderPublishTimedOutLocked(ProcessRecord app) {
	    cleanupAppInLaunchingProvidersLocked(app, true);
	    mProcessList.removeProcessLocked(app, false, true, "timeout publishing content providers");
	}

## 7.Query过程

### 7.1 ContentProviderProxy.query  
[ -> frameworks/base/core/java/android/content/ContentProviderNative.java ]

	final class ContentProviderProxy implements IContentProvider
	{
	    // Transport
	    private IBinder mRemote;

	    public ContentProviderProxy(IBinder remote)
	    {
	        mRemote = remote;
	    }
 
	    @Override
	    public IBinder asBinder()
	    {
	        return mRemote;
	    }
 
	    @Override
	    public Cursor query(String callingPkg, Uri url, @Nullable String[] projection,
	            @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal)
	            throws RemoteException {
	        BulkCursorToCursorAdaptor adaptor = new BulkCursorToCursorAdaptor();
	        Parcel data = Parcel.obtain();
	        Parcel reply = Parcel.obtain();
	        try {
	            // 封装data
	            data.writeInterfaceToken(IContentProvider.descriptor);
	            data.writeString(callingPkg);
	            url.writeToParcel(data, 0);
	            int length = 0;
	            if (projection != null) {
	                length = projection.length;
	            }
	            data.writeInt(length);
	            for (int i = 0; i < length; i++) {
	                data.writeString(projection[i]);
	            }
	            data.writeBundle(queryArgs);
	            data.writeStrongBinder(adaptor.getObserver().asBinder());
	            data.writeStrongBinder(cancellationSignal != null ? cancellationSignal.asBinder() : null);
 
	            // mRemote为Transport, Binder通信最终会走到Transport.onTransact方法
	            mRemote.transact(IContentProvider.QUERY_TRANSACTION, data, reply, 0);
 
	            DatabaseUtils.readExceptionFromParcel(reply);
 
	            if (reply.readInt() != 0) {
	                // 从reply中读取BulkCursorDescriptor
	                BulkCursorDescriptor d = BulkCursorDescriptor.CREATOR.createFromParcel(reply);
	                Binder.copyAllowBlocking(mRemote, (d.cursor != null) ? d.cursor.asBinder() : null);
	                // 初始化BulkCursorToCursorAdaptor
	                adaptor.initialize(d);
	            } else {
	                adaptor.close();
	                adaptor = null;
	            }
	            return adaptor;
	        } catch (RemoteException ex) {
	            adaptor.close();
	            throw ex;
	        } catch (RuntimeException ex) {
	            adaptor.close();
	            throw ex;
	        } finally {
	            data.recycle();
	            reply.recycle();
	        }
	    }
		...
	}

### 7.2 Transport.onTransact
[ -> frameworks/base/core/java/android/content/ContentProviderNative.java ]

由于Transport继承ContentProviderNative, 因此等价于调用ContentProviderNative.onTransact

	abstract public class ContentProviderNative extends Binder implements IContentProvider {
	    public ContentProviderNative()
	    {
	        attachInterface(this, descriptor);
	    }
  
	    /**
	     * Cast a Binder object into a content resolver interface, generating
	     * a proxy if needed.
	     */
	    static public IContentProvider asInterface(IBinder obj)
	    {
	        if (obj == null) {
	            return null;
	        }
	        IContentProvider in = (IContentProvider)obj.queryLocalInterface(descriptor);
	        if (in != null) {
	            return in;
	        }
  
	        return new ContentProviderProxy(obj);
	    }
  
	    /**
	     * Gets the name of the content provider.
	     * Should probably be part of the {@link IContentProvider} interface.
	     * @return The content provider name.
	     */
	    public abstract String getProviderName();
  
	    @Override
	    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
	            throws RemoteException {
	        switch (code) {
	            case QUERY_TRANSACTION:
	            {
	                data.enforceInterface(IContentProvider.descriptor);
 
	                String callingPkg = data.readString();
	                Uri url = Uri.CREATOR.createFromParcel(data);
 
	                String[] projection
	                int num = data.readInt();
	                String[] projection = null;
	                if (num > 0) {
	                    projection = new String[num];
	                    for (int i = 0; i < num; i++) {
	                        projection[i] = data.readString();
	                    }
	                }
 
	                Bundle queryArgs = data.readBundle();
	                IContentObserver observer = IContentObserver.Stub.asInterface(data.readStrongBinder());
	                ICancellationSignal cancellationSignal = ICancellationSignal.Stub.asInterface(data.readStrongBinder());
 
	                // 真正执行数据查询工作
	                Cursor cursor = query(callingPkg, url, projection, queryArgs, cancellationSignal);
	                if (cursor != null) {
	                    CursorToBulkCursorAdaptor adaptor = null;
 
	                    try {
	                        adaptor = new CursorToBulkCursorAdaptor(cursor, observer, getProviderName());
	                        cursor = null;
 
	                        BulkCursorDescriptor d = adaptor.getBulkCursorDescriptor();
	                        adaptor = null;
 
	                        // 将数据写入reply并返回
	                        reply.writeNoException();
	                        reply.writeInt(1);
	                        d.writeToParcel(reply, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
	                    } finally {
	                        // Close cursor if an exception was thrown while constructing the adaptor.
	                        if (adaptor != null) {
	                            adaptor.close();
	                        }
	                        if (cursor != null) {
	                            cursor.close();
	                        }
	                    }
	                } else {
	                    reply.writeNoException();
	                    reply.writeInt(0);
	                }
 
	                return true;
	            }
	            case GET_TYPE_TRANSACTION: ...
	            case INSERT_TRANSACTION: ...
	            case BULK_INSERT_TRANSACTION: ...
	            case APPLY_BATCH_TRANSACTION: ...
	            case DELETE_TRANSACTION: ...
	            case UPDATE_TRANSACTION: ...
	            case OPEN_FILE_TRANSACTION: ...
	            case OPEN_ASSET_FILE_TRANSACTION: ...
	            case CALL_TRANSACTION: ...
	            case GET_STREAM_TYPES_TRANSACTION: ...
	            case OPEN_TYPED_ASSET_FILE_TRANSACTION: ...
	            case CREATE_CANCELATION_SIGNAL_TRANSACTION: ...
	            case CANONICALIZE_TRANSACTION: ...
	            case UNCANONICALIZE_TRANSACTION: ...
	            case REFRESH_TRANSACTION: ...
	        }
  
	        return super.onTransact(code, data, reply, flags);
	    }
  
	    @Override
	    public IBinder asBinder()
	    {
	        return this;
	    }
	}

### 7.3 Transport.query
[ -> frameworks/base/core/java/android/content/ContentProvider.java ]

	// Binder object that deals with remoting.
	class Transport extends ContentProviderNative {
	    volatile AppOpsManager mAppOpsManager = null;
	    volatile int mReadOp = AppOpsManager.OP_NONE;
	    volatile int mWriteOp = AppOpsManager.OP_NONE;
	    volatile ContentInterface mInterface = ContentProvider.this;
   
	    ContentProvider getContentProvider() {
	        return ContentProvider.this;
	    }
   
	    @Override
	    public String getProviderName() {
	        return getContentProvider().getClass().getName();
	    }
   
	    @Override
	    public Cursor query(String callingPkg, Uri uri, @Nullable String[] projection,
	            @Nullable Bundle queryArgs, @Nullable ICancellationSignal cancellationSignal) {
	        uri = validateIncomingUri(uri);
	        uri = maybeGetUriWithoutUserId(uri);
	        // 若没有读Uri权限FLAG_GRANT_READ_URI_PERMISSION
	        if (enforceReadPermission(callingPkg, uri, null) != AppOpsManager.MODE_ALLOWED) {
	            // 返回指定列的空Cursor
	            if (projection != null) {
	                return new MatrixCursor(projection, 0);
	            }
   
	            // ContentProvider.query查询所有列(projection为null). 参考【第7.4节】
	            cursor = mInterface.query(uri, projection, queryArgs, CancellationSignal.fromTransport(cancellationSignal));
	            if (cursor == null) {
	                return null;
	            }
   
	            // 返回包含所有列的空Cursor
	            return new MatrixCursor(cursor.getColumnNames(), 0);
	        }
 
	        // ContentProvider.query查询所有列(projection为null)
	        return mInterface.query(uri, projection, queryArgs, CancellationSignal.fromTransport(cancellationSignal));
	    }
   
	    ...
	}

前面已经分析过了:  AMS.getContentProvider拿到的是ContactsProvider2, AMP.getContentProvider拿到的是ContentProviderProxy.

而mInterface=ContentProvider.this, 对于联系人查询可知, mInterface对应ContactsProvider2.

### 7.4 ContactsProvider2.query

[ -> frameworks/base/core/java/android/content/ContentProvider.java ]

	@Override
	public @Nullable Cursor query(@NonNull Uri uri, @Nullable String[] projection,
	        @Nullable Bundle queryArgs, @Nullable CancellationSignal cancellationSignal) {
	    queryArgs = queryArgs != null ? queryArgs : Bundle.EMPTY;
 
	    String sortClause = queryArgs.getString(ContentResolver.QUERY_ARG_SQL_SORT_ORDER);
	    // 一般走不到这里
	    if (sortClause == null && queryArgs.containsKey(ContentResolver.QUERY_ARG_SORT_COLUMNS)) {
	        sortClause = ContentResolver.createSqlSortClause(queryArgs);
	    }
 
	    return query(
	            uri,
	            projection,
	            queryArgs.getString(ContentResolver.QUERY_ARG_SQL_SELECTION),
	            queryArgs.getStringArray(ContentResolver.QUERY_ARG_SQL_SELECTION_ARGS),
	            sortClause,
	            cancellationSignal);
	}

[ -> packages/providers/ContactsProvider/src/com/android/providers/contacts/ContactsProvider2.java ]

	@Override
	public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
	    return query(uri, projection, selection, selectionArgs, sortOrder, null);
	}
 
	@Override
	public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs,
	        String sortOrder, CancellationSignal cancellationSignal) {
        ... 
        return queryDirectoryIfNecessary(uri, projection, selection, selectionArgs, sortOrder, cancellationSignal);
	}

	protected Cursor queryLocal(final Uri uri, final String[] projection, String selection,
	        String[] selectionArgs, String sortOrder, final long directoryId,
	        final CancellationSignal cancellationSignal) {
	    ...
	    final int match = sUriMatcher.match(uri);
	    switch (match) {
	        ...
	        case CONTACTS: {
	            setTablesAndProjectionMapForContacts(qb, projection);
	            appendLocalDirectoryAndAccountSelectionIfNeeded(qb, directoryId, uri);
	            break;
	        }
	        case CONTACTS_ID: ...
	        ...
	    }
		... 
	    Cursor cursor = doQuery(db, qb, projection, selection, selectionArgs, localizedSortOrder, groupBy, having, limit, cancellationSignal);
	    return cursor;
	}

	private Cursor doQuery(final SQLiteDatabase db, SQLiteQueryBuilder qb, String[] projection,
	        String selection, String[] selectionArgs, String sortOrder, String groupBy,
	        String having, String limit, CancellationSignal cancellationSignal) {
	    if (projection != null && projection.length == 1
	            && BaseColumns._COUNT.equals(projection[0])) {
	        qb.setProjectionMap(sCountProjectionMap);
	    }
	    final Cursor c = qb.query(db, projection, selection, selectionArgs, groupBy, having,
	            sortOrder, limit, cancellationSignal);
	    if (c != null) {
	        c.setNotificationUri(getContext().getContentResolver(), ContactsContract.AUTHORITY_URI);
	    }
	    return c;
	}

剩下的操作就是SQLite数据库操作了. 并且Cursor操作直接操作到Native层了.  本节暂不分析SQLite原理.

通过ContentProviderProxy.query查询操作最终拿到的Cursor是BulkCursorToCursorAdaptor类型.  
通过ApplicationContentResolver.query拿到的是CursorWrapperInner(new BulkCursorToCursorAdaptor())

## 8.总结

ContentProvider整个查询过程分为如下步骤:  
1.`ContextImpl.getContentResolver()`: 拿到ApplicationContentResolver  

2.`ApplicationContentResolver.query`:  执行查询操作, 该过程又分为3个过程:    
2.1 获取ContentProviderProxy.  
其调用链为:  

	ApplicationContentResolver.acquireProvider -> ActivityManager.getService().getContentProvider(...) ->
	IActivityManager.Stub.Proxy#getContentProvider -> mRemote.transact ->
	IActivityManager.Stub#onTransact -> ActivityManagerService#getContentProvider(AMS端) ->
	ContentViewHolder.writeToParcel(AMS端) -> ContentProviderHolder.CREATOR.createFromParcel(AMP端)

2.2 发布ContentProvider(仅当provider未发布时)  
ApplicationThread.scheduleInstallProvider, AMS发布的provider是ContactsProvider2, 而AMP端拿到的provider是ContentProviderProxy. 发布超时时间是20秒  

2.3 query操作  
ContentProviderProxy.query -> ContactsProvider2.query -> SQLite操作, 同时拿到new CursorWrapperInner(new BulkCursorToCursorAdaptor()).   

后续操作都是通过CursorWrapperInner来进行读取.  


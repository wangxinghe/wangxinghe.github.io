---
layout: post
comments: true
title: "sendBroadcast过程"
description: "sendBroadcast过程"
category: AOSP
tags: [AOSP]
---

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

<!--more-->

本文分析广播Broadcast发送过程.

## 0.文件结构

	frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
	frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
	frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java
	frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
	frameworks/base/services/core/java/com/android/server/am/BroadcastRecord.java

	frameworks/base/core/java/android/app/ContextImpl.java

## 1.时序图
TODO

## 2.ContextImpl.sendBroadcast
[ -> frameworks/base/core/java/android/app/ContextImpl.java ]

	@Override
	public void sendBroadcast(Intent intent) {
	    warnIfCallingFromSystemProcess();
	    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
	 
	    intent.prepareToLeaveProcess(this);
	    ActivityManager.getService().broadcastIntent(
	            mMainThread.getApplicationThread(), intent, resolvedType, null,
	            Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false, getUserId());
	}

## 3.ActivityManagerService
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

### 3.1 broadcastIntent

	public final int broadcastIntent(IApplicationThread caller,
	        Intent intent, String resolvedType, IIntentReceiver resultTo,
	        int resultCode, String resultData, Bundle resultExtras,
	        String[] requiredPermissions, int appOp, Bundle bOptions,
	        boolean serialized, boolean sticky, int userId) {
	    enforceNotIsolatedCaller("broadcastIntent");
	    synchronized(this) {
	        intent = verifyBroadcastLocked(intent);
	 
	        final ProcessRecord callerApp = getRecordForAppLocked(caller);
	        final int callingPid = Binder.getCallingPid();
	        final int callingUid = Binder.getCallingUid();
	 
	        final long origId = Binder.clearCallingIdentity();
	        try {
	            return broadcastIntentLocked(callerApp,
	                    callerApp != null ? callerApp.info.packageName : null,
	                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
	                    requiredPermissions, appOp, bOptions, serialized, sticky,
	                    callingPid, callingUid, callingUid, callingPid, userId);
	        } finally {
	            Binder.restoreCallingIdentity(origId);
	        }
	    }
	}

### 3.2 broadcastIntentLocked

	final int broadcastIntentLocked(ProcessRecord callerApp,
	        String callerPackage, Intent intent, String resolvedType,
	        IIntentReceiver resultTo, int resultCode, String resultData,
	        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
	        boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
	        int realCallingPid, int userId) {
	    return broadcastIntentLocked(callerApp, callerPackage, intent, resolvedType, resultTo,
	        resultCode, resultData, resultExtras, requiredPermissions, appOp, bOptions, ordered,
	        sticky, callingPid, callingUid, realCallingUid, realCallingPid, userId,
	        false /* allowBackgroundActivityStarts */);
	}

具体实现细节看下面代码：

	final int broadcastIntentLocked(ProcessRecord callerApp,
	        String callerPackage, Intent intent, String resolvedType,
	        IIntentReceiver resultTo, int resultCode, String resultData,
	        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
	        boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
	        int realCallingPid, int userId, boolean allowBackgroundActivityStarts) {
	    intent = new Intent(intent);
	 
	    // 1.根据各种限制设置intent的flags: 比如Instant App不能使用FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS, 广播不会发送到杀死的进程, 系统还没启动不允许启动新进程
	    // 2.检查接收广播的userId是否正在运行
	    // 3.权限和限制检查: 如CHANGE_DEVICE_IDLE_TEMP_WHITELIST权限, START_ACTIVITIES_FROM_BACKGROUND权限, OP_RUN_ANY_IN_BACKGROUND操作权限
	    // 4.非系统用户不允许发送ProtectedBroadcast, 系统用户指(ROOT_UID/SYSTEM_UID/PHONE_UID/BLUETOOTH_UID/NFC_UID/SE_UID/NETWORK_STACK_UID)
	    ...
	 
	    userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true, ALLOW_NON_FULL, "broadcast", callerPackage);
	    boolean timeoutExempt = false;
	    final String action = intent.getAction();
	    if (action != null) {
	        // 如果action所属Intent需要发送给未启动的或后台的进程, 则Intent添加flags为FLAG_RECEIVER_INCLUDE_BACKGROUND
	        if (getBackgroundLaunchBroadcasts().contains(action)) {
	            intent.addFlags(Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
	        }
	 
	        // 包管理器发出的和包移除相关的action的处理，如ACTION_UID_REMOVED/ACTION_PACKAGE_REMOVED/ACTION_PACKAGE_CHANGED/...
	        ...
	    }
	    // sticky广播处理, 常规的sendBroadcast发送的都是非sticky广播, 走不到这里
	    if (sticky) {
	        // 权限和条件检查包括: Manifest.permission.BROADCAST_STICKY权限检查, requiredPermissions和Component为空检查, mStickyBroadcasts重复广播检查
	        ...
	 
	        // 将发送的广播Intent替换或插入到mStickyBroadcasts队列中相应位置
	        ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
	        if (stickies == null) {
	            stickies = new ArrayMap<>();
	            mStickyBroadcasts.put(userId, stickies);
	        }
	        ArrayList<Intent> list = stickies.get(intent.getAction());
	        if (list == null) {
	            list = new ArrayList<>();
	            stickies.put(intent.getAction(), list);
	        }
	        final int stickiesCount = list.size();
	        int i;
	        for (i = 0; i < stickiesCount; i++) {
	            if (intent.filterEquals(list.get(i))) {
	                // This sticky already exists, replace it.
	                list.set(i, new Intent(intent));
	                break;
	            }
	        }
	        if (i >= stickiesCount) {
	            list.add(new Intent(intent));
	        }
	    }
	 
	    // 设置广播的目标用户数组users
	    int[] users;
	    if (userId == UserHandle.USER_ALL) {
	        // 所有启动用户
	        users = mUserController.getStartedUserArray();
	    } else {
	        // 特定用户
	        users = new int[] {userId};
	    }
	 
	    // Figure out who all will receive this broadcast.
	 
	    // 收集receivers列表组件信息. 参考【第3.3节】
	    List receivers = null;
	    if ((intent.getFlags() & Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
	        receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
	    }
	 
	    // 通过IntentResolver收集已注册的Receiver列表registeredReceivers
	    List<BroadcastFilter> registeredReceivers = null;
	    if (intent.getComponent() == null) {
	        if (userId == UserHandle.USER_ALL && callingUid == SHELL_UID) {
	            for (int i = 0; i < users.length; i++) {
	                // 过滤Shell限制的用户
	                ...
	 
	                List<BroadcastFilter> registeredReceiversForUser =
	                        mReceiverResolver.queryIntent(intent, resolvedType, false /*defaultOnly*/, users[i]);
	                if (registeredReceivers == null) {
	                    registeredReceivers = registeredReceiversForUser;
	                } else if (registeredReceiversForUser != null) {
	                    registeredReceivers.addAll(registeredReceiversForUser);
	                }
	            }
	        } else {
	            registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false /*defaultOnly*/, userId);
	        }
	    }
	 
	    final boolean replacePending = (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
	 
	    // 整体逻辑: 将新发送的无序广播入队列
	    int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
	    if (!ordered && NR > 0) {
	        // If we are not serializing this broadcast, then send the
	        // registered receivers separately so they don't wait for the
	        // components to be launched.
	        if (isCallerSystem) {
	            checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid, isProtectedBroadcast, registeredReceivers);
	        }
	        final BroadcastQueue queue = broadcastQueueForIntent(intent);
	        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
	                callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
	                requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
	                resultCode, resultData, resultExtras, ordered, sticky, false, userId,
	                allowBackgroundActivityStarts, timeoutExempt);
	        final boolean replaced = replacePending && (queue.replaceParallelBroadcastLocked(r) != null);
	        if (!replaced) {
	            queue.enqueueParallelBroadcastLocked(r);
	            queue.scheduleBroadcastsLocked();
	        }
	        registeredReceivers = null;
	        NR = 0;
	    }
	 
	    // Merge into one list receivers.
	    // 1.receivers中移除如下action对应的广播: ACTION_PACKAGE_ADDED/ACTION_PACKAGE_RESTARTED/ACTION_PACKAGE_DATA_CLEARED/ACTION_EXTERNAL_APPLICATIONS_AVAILABLE
	    // 2.将registeredReceivers列表合并到receivers列表
	    ...
	    if (isCallerSystem) {
	        checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid, isProtectedBroadcast, receivers);
	    }
	 
	    // 将新发送的有序广播入队列
	    if ((receivers != null && receivers.size() > 0) || resultTo != null) {
	        BroadcastQueue queue = broadcastQueueForIntent(intent);
	        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
	                callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
	                requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
	                resultData, resultExtras, ordered, sticky, false, userId,
	                allowBackgroundActivityStarts, timeoutExempt);
	 
	        final BroadcastRecord oldRecord = replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
	        if (oldRecord != null) {
	            // Replaced, fire the result-to receiver.
	            if (oldRecord.resultTo != null) {
	                final BroadcastQueue oldQueue = broadcastQueueForIntent(oldRecord.intent);
	                oldQueue.performReceiveLocked(oldRecord.callerApp, oldRecord.resultTo,
	                        oldRecord.intent, Activity.RESULT_CANCELED, null, null, false, false, oldRecord.userId);
	            }
	        } else {
	            queue.enqueueOrderedBroadcastLocked(r);
	            queue.scheduleBroadcastsLocked();
	        }
	    } else {
	        // 该广播没有广播接收器, 说明是隐式广播, 记录一下即可
	        if (intent.getComponent() == null && intent.getPackage() == null
	                && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
	            addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
	        }
	    }
	 
	    return ActivityManager.BROADCAST_SUCCESS;
	}

### 3.3 collectReceiverComponents

	private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType, int callingUid, int[] users) {
	    List<ResolveInfo> receivers = null;
	    try {
	        for (int user : users) {
	            // 过滤掉有Shell限制的用户
	            ...
	 
	            List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
	                    .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
	 
	            // 如果目标不是系统用户, 过滤掉newReceivers列表中所有仅系统用户可见的广播
	            ...
	 
	            if (receivers == null) {
	                receivers = newReceivers;
	            } else if (newReceivers != null) {
	                // 遍历receivers和newReceivers列表, 将flags为ActivityInfo.FLAG_SINGLE_USER的广播添加到singleUserReceivers和receivers列表
	                ...
	            }
	        }
	    } catch (RemoteException ex) {
	        // pm is in same process, this will never happen.
	    }
	    return receivers;
	}

**AMS.collectReceiverComponents过程：**  
1.该过程目的是收集BroadcastReceiver目标组件列表. 具体过程如下  
2.遍历目标用户数组users，过滤掉有Shell限制的用户  
3.执行PMS.queryIntentReceivers, 利用组件解析器ComponentResolver解析出BroadcastReceiver组件列表  
4.过滤掉上述BroadcastReceiver组件列表中, 目标用户是非系统用户但是仅系统用户可见的广播  
5.将上述组件列表中flags为ActivityInfo.FLAG_SINGLE_USER的广播添加到singleUserReceivers列表和receivers列表  

## 4.PMS.queryIntentReceivers
[ -> frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java ]

调用链：queryIntentReceivers -> queryIntentReceiversInternal

	@Override
	public @NonNull ParceledListSlice<ResolveInfo> queryIntentReceivers(Intent intent,
	        String resolvedType, int flags, int userId) {
	    return new ParceledListSlice<>(
	            queryIntentReceiversInternal(intent, resolvedType, flags, userId, false /*allowDynamicSplits*/));
	}

queryIntentReceiversInternal实现：

	private @NonNull List<ResolveInfo> queryIntentReceiversInternal(Intent intent,
	        String resolvedType, int flags, int userId, boolean allowDynamicSplits) {
	    // 检查userId在用户管理服务中是否能找到
	    if (!sUserManager.exists(userId)) return Collections.emptyList();
	 
	    final int callingUid = Binder.getCallingUid();
	    // 请求只能来自系统用户或具备INTERACT_ACROSS_USERS/INTERACT_ACROSS_USERS_FULL权限的用户, 否则抛异常
	    mPermissionManager.enforceCrossUserPermission(callingUid, userId,
	            false /*requireFullPermission*/, false /*checkShell*/, "query intent receivers");
	 
	    // 更新flags 
	    flags = updateFlagsForResolve(flags, userId, intent, callingUid, false /*includeInstantApps*/);
	    // 获取Component
	    ComponentName comp = intent.getComponent();
	    if (comp == null) {
	        if (intent.getSelector() != null) {
	            intent = intent.getSelector();
	            comp = intent.getComponent();
	        }
	    }
	 
	    final String instantAppPkgName = getInstantAppPackageName(callingUid);
	    if (comp != null) {
	        final List<ResolveInfo> list = new ArrayList<>(1);
	        final ActivityInfo ai = getReceiverInfo(comp, flags, userId);
	        if (ai != null) {
	            // 如下两种情况Activity不能使用. blockResolution为true
	            // 1) the calling package is normal and the activity is within an instant application or
	            // 2) the calling package is ephemeral and the activity is not visible to instant applications.
	            ...
	 
	            if (!blockResolution) {
	                ResolveInfo ri = new ResolveInfo();
	                ri.activityInfo = ai;
	                list.add(ri);
	            }
	        }
	        // 过滤掉list中的临时的BroadcastReceiver组件
	        return applyPostResolutionFilter(list, instantAppPkgName, allowDynamicSplits, callingUid, false, userId, intent);
	    }
	 
	    synchronized (mPackages) {
	        String pkgName = intent.getPackage();
	        if (pkgName == null) {
	            // 通过组件解析器ComponentResolver解析出符合条件的组件信息列表
	            final List<ResolveInfo> result = mComponentResolver.queryReceivers(intent, resolvedType, flags, userId);
	            // 过滤掉临时的Activity
	            return applyPostResolutionFilter(result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId, intent);
	        }
	        // 获取包解析信息
	        final PackageParser.Package pkg = mPackages.get(pkgName);
	        if (pkg != null) {
	            // 通过组件解析器ComponentResolver解析出符合条件的组件信息列表
	            final List<ResolveInfo> result = mComponentResolver.queryReceivers(intent, resolvedType, flags, pkg.receivers, userId);
	            // 过滤掉临时的Activity
	            return applyPostResolutionFilter(result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId, intent);
	        }
	        return Collections.emptyList();
	    }
	}

**PMS.queryIntentReceivers过程：**  
1.首先是边界条件检查：检查userId是否存在, 检查请求只能来自系统用户或具备INTERACT_ACROSS_USERS/INTERACT_ACROSS_USERS_FULL权限的用户.  
2.利用组件解析器ComponentResolver解析出和BroadcastReceiver对应的组件列表List< ActivityInfo>,  过滤掉其中的临时组件并返回  
(1)如果Intent.getComponent非空, 则根据Component解析  
(2)否则根据MIME Type/scheme/action等信息解析(执行ComponentResolver.queryReceivers)  

注意：  
`ComponentResolver`：组件解析器. Resolves all Android component types [activities, services, providers and receivers].  
`PackageParser`：包解析器  

## 5.ComponentResolver.queryReceivers
[ -> frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java ]

	List<ResolveInfo> queryReceivers(Intent intent, String resolvedType, int flags, int userId) {
	    synchronized (mLock) {
	        return mReceivers.queryIntent(intent, resolvedType, flags, userId);
	    }
	}

## 6.ActivityIntentResolver
[ -> frameworks/base/services/core/java/com/android/server/pm/ComponentResolver.java ]

### 6.1 queryIntent

	private static final class ActivityIntentResolver extends IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo> {
	    @Override
	    public List<ResolveInfo> queryIntent(Intent intent, String resolvedType, boolean defaultOnly, int userId) {
	        if (!sUserManager.exists(userId)) return null;
	        mFlags = (defaultOnly ? PackageManager.MATCH_DEFAULT_ONLY : 0);
	        return super.queryIntent(intent, resolvedType, defaultOnly, userId);
	    }
	 
	    List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags, int userId) {
	        if (!sUserManager.exists(userId)) {
	            return null;
	        }
	        mFlags = flags;
	        return super.queryIntent(intent, resolvedType, (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0, userId);
	    }
	 
	    public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly, int userId) {
	        String scheme = intent.getScheme();
	 
	        ArrayList<R> finalList = new ArrayList<R>();
	 
	        F[] firstTypeCut = null;
	        F[] secondTypeCut = null;
	        F[] thirdTypeCut = null;
	        F[] schemeCut = null;
	 
	 
	        // 找出和MIME Type匹配的所有filter
	        if (resolvedType != null) {
	            int slashpos = resolvedType.indexOf('/');
	            if (slashpos > 0) {
	                final String baseType = resolvedType.substring(0, slashpos);
	                if (!baseType.equals("*")) {
	                    if (resolvedType.length() != slashpos+2 || resolvedType.charAt(slashpos+1) != '*') {
	                        firstTypeCut = mTypeToFilter.get(resolvedType);
	                        secondTypeCut = mWildTypeToFilter.get(baseType);
	                    } else {
	                        firstTypeCut = mBaseTypeToFilter.get(baseType);
	                        secondTypeCut = mWildTypeToFilter.get(baseType);
	                    }
	                    thirdTypeCut = mWildTypeToFilter.get("*");
	                } else if (intent.getAction() != null) {
	                    firstTypeCut = mTypedActionToFilter.get(intent.getAction());
	                }
	            }
	        }
	 
	        // 找出和scheme匹配的所有filter
	        if (scheme != null) {
	            schemeCut = mSchemeToFilter.get(scheme);
	        }
	 
	        // 找出和action匹配的所有filter
	        if (resolvedType == null && scheme == null && intent.getAction() != null) {
	            firstTypeCut = mActionToFilter.get(intent.getAction());
	        }
	 
	        // 过滤掉以上各filter组中不符合条件的元素, 将其封装为ResolveInfo对象添加到finalList
	        FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
	        if (firstTypeCut != null) {
	            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
	                    scheme, firstTypeCut, finalList, userId);
	        }
	        if (secondTypeCut != null) {
	            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
	                    scheme, secondTypeCut, finalList, userId);
	        }
	        if (thirdTypeCut != null) {
	            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
	                    scheme, thirdTypeCut, finalList, userId);
	        }
	        if (schemeCut != null) {
	            buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
	                    scheme, schemeCut, finalList, userId);
	        }
	        filterResults(finalList);
	        // 按优先级降序排列
	        sortResults(finalList);
	 
	        return finalList;
	    }
	}

**ComponentResolver.queryReceivers -> ActivityIntentResolver.queryIntent过程：**  
1.找出符合条件的IntentFilter数组：  
(1)找出MIME Type匹配的IntentFilter数组  
(2)找出scheme匹配的IntentFilter数组  
(3)找出action匹配的IntentFilter数组  
2.以Intent和上述IntentFilter数组作为入参,调用ComponentResolver.buildResolveList获取符合条件的BroadcastReceiver组件信息列表List< ActivityResolveInfo>  
3.将List< ActivityResolveInfo>按优先级降序排序并返回  

### 6.2 queryIntentForPackage

调用链：queryIntentForPackage -> queryIntentFromList  

    List<ResolveInfo> queryIntentForPackage(Intent intent, String resolvedType,
            int flags, List<PackageParser.Activity> packageActivities, int userId) {
        if (!sUserManager.exists(userId)) {
            return null;
        }
        if (packageActivities == null) {
            return null;
        }
        mFlags = flags;
        final boolean defaultOnly = (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0;
        final int activitiesSize = packageActivities.size();
        ArrayList<PackageParser.ActivityIntentInfo[]> listCut = new ArrayList<>(activitiesSize);
 
        ArrayList<PackageParser.ActivityIntentInfo> intentFilters;
        for (int i = 0; i < activitiesSize; ++i) {
            intentFilters = packageActivities.get(i).intents;
            if (intentFilters != null && intentFilters.size() > 0) {
                PackageParser.ActivityIntentInfo[] array = new PackageParser.ActivityIntentInfo[intentFilters.size()];
                intentFilters.toArray(array);
                listCut.add(array);
            }
        }
        return super.queryIntentFromList(intent, resolvedType, defaultOnly, listCut, userId);
    }

queryIntentFromList实现：

    public List<R> queryIntentFromList(Intent intent, String resolvedType, boolean defaultOnly,
            ArrayList<F[]> listCut, int userId) {
        ArrayList<R> resultList = new ArrayList<R>();
 
        final boolean debug = localLOGV || ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
 
        FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
        final String scheme = intent.getScheme();
        int N = listCut.size();
        for (int i = 0; i < N; ++i) {
            buildResolveList(intent, categories, debug, defaultOnly, resolvedType, scheme, listCut.get(i), resultList, userId);
        }
        filterResults(resultList);
        sortResults(resultList);
        return resultList;
    }

### 6.3 buildResolveList

    private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly, String resolvedType, String scheme,
            F[] src, List<R> dest, int userId) {
        final String action = intent.getAction();
        final Uri data = intent.getData();
        final String packageName = intent.getPackage();
 
        final boolean excludingStopped = intent.isExcludingStopped();
 
        final int N = src != null ? src.length : 0;
        boolean hasNonDefaults = false;
        F filter;
        for (int i=0; i<N && (filter=src[i]) != null; i++) {
            int match;
 
            // 过滤掉target为stopped状态的filter
            if (excludingStopped && isFilterStopped(filter, userId)) {
                continue;
            }
 
            // 过滤掉packageName不匹配的filter
            if (packageName != null && !isPackageForFilter(packageName, filter)) {
                continue;
            }
 
            // 过滤掉重复filter
            if (!allowFilterResult(filter, dest)) {
                continue;
            }
 
            // filter各属性匹配
            match = filter.match(action, resolvedType, scheme, data, categories, TAG);
            if (match >= 0) {
                if (!defaultOnly || filter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                    // 匹配成功, 将filter信息封装成ResolveInfo并添加到dest列表
                    final R oneResult = newResult(filter, match, userId);
                    if (oneResult != null) {
                        dest.add(oneResult);
                    }
                } else {
                    hasNonDefaults = true;
                }
            }
        }
    }
 
 
    @Override
    protected ResolveInfo newResult(PackageParser.ActivityIntentInfo info, int match, int userId) {
        // 各种边界条件检查包括: userId是否存在, Component和flag是否可用和匹配, PackageSetting是否为空, instant app和ephemeral app相关检查
        ...
         
        // 创建ResolveInfo对象并设置各属性
        final ResolveInfo res = new ResolveInfo();
        res.activityInfo = ai;
        if ((mFlags & PackageManager.GET_RESOLVED_FILTER) != 0) {
            res.filter = info;
        }
        res.handleAllWebDataURI = info.handleAllWebDataURI();
        res.priority = info.getPriority();
        res.preferredOrder = activity.owner.mPreferredOrder;
        res.match = match;
        res.isDefault = info.hasDefault;
        res.labelRes = info.labelRes;
        res.nonLocalizedLabel = info.nonLocalizedLabel;
        if (sPackageManagerInternal.userNeedsBadging(userId)) {
            res.noResourceId = true;
        } else {
            res.icon = info.icon;
        }
        res.iconResourceId = info.icon;
        res.system = res.activityInfo.applicationInfo.isSystemApp();
        res.isInstantAppAvailable = userState.instantApp;
        return res;
    }


**ComponentResolver.buildResolveList过程：**  
1.该方法主要做的事情：根据IntentFilter[]条件, 利用组件解析器ComponentResolver查找符合条件的BroadcastReceiver组件信息列表. 入参为Intent和数组IntentFilter[], 出参为List< ActivityResolveInfo>.   
2.遍历IntentFilter数组, 过滤掉如下IntentFilter: target为stopped状态、packageName不匹配或者重复的filter  
3.IntentFilter和Intent进行匹配, 如匹配成功, 则进行边界条件检查后将IntentFilter信息封装成ResolveInfo对象, 添加到dest列表返回  
4.IntentFilter边界条件检查包括: userId是否存在, Component和flag是否可用和匹配, PackageSetting是否为空, instant app和ephemeral app相关检查  

## 7.AMS.broadcastQueueForIntent
[ -> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java ]

	BroadcastQueue broadcastQueueForIntent(Intent intent) {
	    if (isOnOffloadQueue(intent.getFlags())) {
	        return mOffloadBroadcastQueue;
	    }
	 
	    final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
	    return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
	}

## 8.BroadcastQueue
[ -> frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java ]

### 8.1 enqueueParallelBroadcastLocked
### 8.2 scheduleBroadcastsLocked

这小节参考文章 **[registerReceiver过程](http://mouxuejie.com/blog/2020-04-05/register-receiver/)** 第6节.

## 9.总结

**ContextImpl.sendBroadcast过程：**  
1.边界和限制条件检查：  
(1)根据各种限制设置intent的flags: 比如Instant App不能使用FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS, 广播不会发送到杀死的进程, 系统还没启动不允许启动新进程  
(2)检查接收广播的userId是否正在运行  
(3)权限和限制检查: 如CHANGE_DEVICE_IDLE_TEMP_WHITELIST权限, START_ACTIVITIES_FROM_BACKGROUND权限, OP_RUN_ANY_IN_BACKGROUND操作权限  
(4)非系统用户不允许发送ProtectedBroadcast, 系统用户指(ROOT_UID/SYSTEM_UID/PHONE_UID/BLUETOOTH_UID/NFC_UID/SE_UID/NETWORK_STACK_UID)  

2.特殊action处理：  
(1)若action目标为发送给未启动的或后台的进程, 则Intent添加flags为FLAG_RECEIVER_INCLUDE_BACKGROUND  
(2)若action为包管理器发出的和包移除相关的action, 则做相应逻辑处理. 如ACTION_UID_REMOVED/ACTION_PACKAGE_REMOVED/ACTION_PACKAGE_CHANGED/...  

3.Sticky广播处理(仅针对sendStickyBroadcast)：  
(1)权限和条件检查包括: Manifest.permission.BROADCAST_STICKY权限检查, requiredPermissions和Component为空检查, mStickyBroadcasts重复广播检查  
(2)将发送的广播Intent替换或插入到mStickyBroadcasts队列中相应位置  

4.设置广播的目标用户数组users, 如果userId为USER_ALL则users为所有启动用户, 否则users为userId  
5.执行AMS.collectReceiverComponents, 利用组件解析器ComponentResolver收集符合条件的BroadcastReceiver组件列表信息  
6.若Intent.getComponent为空, 则执行ActivityIntentResolver.queryIntent, 解析出所有已注册的BroadcastReceiver组件列表  
7.无序广播处理：将步骤6的广播入队到无序广播队列, 并执行广播接收器处理逻辑BroadcastQueue.scheduleBroadcastsLocked  

8.有序广播处理(仅针对sendOrderedBroadcast)：  
(1)集合步骤5和6的BroadcastReceiver列表, 移除Package操作相关的广播ACTION_PACKAGE_ADDED/ACTION_PACKAGE_RESTARTED/ACTION_PACKAGE_DATA_CLEARED/ACTION_EXTERNAL_APPLICATIONS_AVAILABLE  
(2)若BroadcastReceiver列表非空. 如果原来广播队列就有相同Intent的广播且Intent.flags为FLAG_RECEIVER_REPLACE_PENDING, 则先把老的广播替换掉，并调用BroadcastQueue.performReceiveLocked对老广播执行广播接收操作;   
否则将广播插入有序广播队列, 并调用BroadQueue.scheduleBroadcastsLocked处理下一个广播接收操作  


**广播发送过程总结：**  
1.先将广播添加到无序广播列表  
2.找出该广播对应的所有接收器, 并执行广播接收器操作  


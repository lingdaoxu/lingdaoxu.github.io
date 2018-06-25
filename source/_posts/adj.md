title: adj
author: 道墟
date: 2018-06-25 18:06:55
tags:
---
# 进程的生命周期


# lowmemorykiller
六个档次
- CACHED_APP_MAX_ADJ
- CACHED_APP_MIN_ADJ
- BACKUP_APP_ADJ
- PERCEPTIBLE_APP_ADJ
- VISIBLE_APP_ADJ
- FOREGROUND_APP_ADJ

# adj级别
> ProcessList.java 中定义的


# state级别
> ActivityManager.java 中定义的

# adj算法核心

- updateOomAdjLocked：更新adj，当目标进程为空，或者被杀则返回false；否则返回true;
- computeOomAdjLocked：计算adj，返回计算后RawAdj值;
- applyOomAdjLocked：应用adj，当需要杀掉目标进程则返回false；否则返回true。

> updateOomAdjLocked实现过程中依次会 **computeOomAdjLocked** 和 **applyOomAdjLocked**

# 触发时机

#activity
> ass：/services/core/java/com/android/server/am/ActivityStackSupervisor 
  as: /services/core/java/com/android/server/am/ActivityStack.java

- ASS.realStartActivityLocked: 启动Activity
- AS.resumeTopActivityInnerLocked: 恢复栈顶Activity
- AS.finishCurrentActivityLocked: 结束当前Activity
- AS.destroyActivityLocked: 摧毁当前Activity

# Service

> /services/core/java/com/android/server/am/ActiveServices.java

- realStartServiceLocked: 启动服务
- bindServiceLocked: 绑定服务(只更新当前app)
- unbindServiceLocked: 解绑服务 (只更新当前app)
- bringDownServiceLocked: 结束服务 (只更新当前app)
- sendServiceArgsLocked: 在bringup或则cleanup服务过程调用 (只更新当前app)

# broadcast

> /services/core/java/com/android/server/am/BroadcastQueue.java

- processNextBroadcast: 处理下一个广播
- processCurBroadcastLocked: 处理当前广播
- deliverToRegisteredReceiverLocked: 分发已注册的广播 (只更新当前app)


# ContentProvider

> /services/core/java/com/android/server/am/ActivityManagerService.java

- AMS.removeContentProvider: 移除provider
- AMS.publishContentProviders: 发布provider (只更新当前app)
- AMS.getContentProviderImpl: 获取provider (只更新当前app)

# Process

> /services/core/java/com/android/server/am/ActivityManagerService.java

- setSystemProcess: 创建并设置系统进程
- addAppLocked: 创建persistent进程
- attachApplicationLocked: 进程创建后attach到system_server的过程;
- trimApplications: 清除没有使用app
- appDiedLocked: 进程死亡
- killAllBackgroundProcesses: 杀死所有后台进程.即(ADJ>9或removed=true的普通进程)
- killPackageProcessesLocked: 以包名的形式 杀掉相关进程;


# 重要参数

 # ProcessList.java
- 空进程存活时长： MAX_EMPTY_TIME = 30min

- (缓存+空)进程个数上限：MAX_CACHED_APPS = SystemProperties.getInt(“sys.fw.bg_apps_limit”,32) = 32(默认)；

- 空进程个数上限：MAX_EMPTY_APPS = computeEmptyProcessLimit(MAX_CACHED_APPS) = MAX_CACHED_APPS/2 = 16；

- trim空进程个数上限：TRIM_EMPTY_APPS = computeTrimEmptyApps() = MAX_EMPTY_APPS/2 = 8；

- trim缓存进程个数上限：TRIM_CACHED_APPS = computeTrimCachedApps() = MAX_CACHED_APPS-MAX_EMPTY_APPS)/3 = 5;

- TRIM_CRITICAL_THRESHOLD = 3;

# ActivityManagerService.java

- mBServiceAppThreshold = SystemProperties.getInt(“ro.sys.fw.bservice_limit”, 5);

- mMinBServiceAgingTime =SystemProperties.getInt(“ro.sys.fw.bservice_age”, 5000);

- mProcessLimit = ProcessList.MAX_CACHED_APPS

- mProcessLimit = emptyProcessLimit(空进程上限) + cachedProcessLimit(缓存进程上限)

- oldTime = now - ProcessList.MAX_EMPTY_TIME;

- LRU进程队列长度 = numEmptyProcs(空进程数) + mNumCachedHiddenProcs(cached进程) + mNumNonCachedProcs（非cached进程）

- emptyFactor = numEmptyProcs/3, 且大于等于1

- cachedFactor = mNumCachedHiddenProcs/3, 且大于等于1


# 方法
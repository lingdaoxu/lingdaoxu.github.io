title: '日常问题:动态广播无法唤起singtask模式下的activity'
author: 道墟
tags: []
categories:
  - 工作问题
  - rk3228&rk3399
date: 2018-06-07 01:12:00
---
之前在AMS服务中我们有梳理过activity的启动过程，刚好今天同事反馈了一个app开发中遇到的问题跟activity的启动相关，这边记录下问题的梳理过程。
问题简单描述：
> 假设有如下两个app，分别以A和B指代，目前在A应用程序里面有一个activity名字叫做a1(a1的launchmode是singleTask),在a1里面注册了一个动态广播接收器用来接收自定义广播，当接收到广播后会继续启动a1本身 ，按照我们的设想这时候不管a1是否存在activity，这时候都应该弹掉上面的activity显示a1。 现在问题来了：
- 这时候如果A应用程序是在后台的，当前显示的应用程序是B的话，这时候如果发出广播，a1不能正常启动
- 如果这时候显示的是launch桌面，但是应用程序还是在后台，这时候如果发出广播，会看到a1正常启动并成为当前的焦点activity
- 如果我们把动态注册改为静态注册接收广播，这时候我们不管是出于第一种情况还是第二种情况都可以正常启动a1

上面就是我们遇到的问题，虽然我们可以通过最后一种用静态注册的形式来实现产品的需求，但是为什么静态注册就可以而动态注册就不可以呢？难道是静态注册可以赋予app更高的权限吗，如果是这样的话在framewrok层是怎么实现的呢？为何之前的activity启动流程中并未看到这部分的代码？相信作为有追求的你这时候肯定也是满脑子问号，那么下面就让我们追踪源码去看下到底为什么要有上面的现象出现吧。

> 在代码分析的过程中，肯定没有像下面讲解的那么顺利，因为经验原因我也是进行了各种的猜测然后去求证再反手打脸再继续猜测的循环过程，不断的在失败中找到正确的逻辑思路，之所以会出现这种情况还是经验太少的原因，但是随着我们遇到问题的增加，经验这种东西肯定是会累积的，这也是成长所需要经历的痛苦。

面对这样一个问题的时候我们需要先找一个切入点，按上面我们设想从生命周期来看，这时候至少是需要调用到a1 的 onResume 方法的，我们观察到的现象则是，在第一种情况下，a1所在的task中，在a1上面的activity确实是被弹出了task，但是通过log我们发现 onResume 并没有被调用。这就证明了至少正常的singtask launchmode的流程是有执行的，只是在执行生命周期上出现了问题，那么我们就从生命周期作为切入点触发。
在之前的activity流程分析中我们知道一个activity的 onResume 在 ActivityStackSupervisor 类中是通过  ActivityStack类的 resumeTopActivityLocked 方法调起的，我们往这个方法里一探究竟：

```java
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
			//这边我们可以看到是通过这个方法继续往下走的
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```

resumeTopActivityInnerLocked 方法是在是太长了 所以就不完整截图，直接截取关键点
```java
		//这边判断如果当前的activity 跟要启动的activity next是同一个的话就不继续往下走了，而根据我们的根据错误的流程就是走到这里来了
        // If the top activity is the resumed one, nothing to do.
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed " + next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
```

根据流程我们发现错误的原因就是 mResumedActivity == next，意味着当我们调到这边的时候 要启动的activity和当前的activity是同一个，这就奇怪了我们明明要启动的activity跟当前的不是同一个，那么问题肯定是出在next的赋值上了，我们往上看看next是在哪里赋值的。

```java
        // Find the first activity that is not finishing.
        final ActivityRecord next = topRunningActivityLocked(null);
```
看来我们需要继续跟踪进 topRunningActivityLocked 方法进行分析了

```java
    final ActivityRecord topRunningActivityLocked(ActivityRecord notTop) {
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            ActivityRecord r = mTaskHistory.get(taskNdx).topRunningActivityLocked(notTop);
            if (r != null) {
                return r;
            }
        }
        return null;
    }
```
topRunningActivityLocked 方法很简单 就是是当前的stack 的 task 列表里面从后向前遍历。
这时候我们上面的三个疑问可以先解答一个了：

在情况2 因为当前是在launch界面的话，因为launch进程是属于homeStack，并没有在进程的Stack中，所以这时候Stack的task列表其实只有进程A的，这样自然 获取到的next 和当前的 mResumedActivity（也就是launch的activity）是不一样的，所以不会进入到终止流程。

但是这样还是无法完全解释 情况3的为什么静态注册可以跳转，所以我们需要继续追踪


根据我们的追踪发现错误的情况就是因为 在task列表中，排在后面的是 程序B的task，这样获取到的自然是 程序B的当前activity了，那这就不对了按理到这步的时候 进程A的task应该已经移动到 task列表的最后面去了（也就是所谓的movefornt操作）。看来我们的疑问还是需要到 ActivityStackSupervisor 的 startActivityUncheckedLocked 方法中寻找答案了，我们需要找到在该方法中将要启动的后台task移动到前台的方法，下面是代码段

```java
                        if (sourceRecord == null || (sourceStack.topActivity() != null &&
                                sourceStack.topActivity().task == sourceRecord.task)) {
                            // We really do want to push this one into the user's face, right now.
                            if (launchTaskBehind && sourceRecord != null) {
                                intentActivity.setTaskToAffiliateWith(sourceRecord.task);
                            }
                            movedHome = true;
                            targetStack.moveTaskToFrontLocked(intentActivity.task, noAnimation,
                                    options, r.appTimeTracker, "bringingFoundTaskToFront");
                            movedToFront = true;
                            if ((launchFlags &
                                    (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                                    == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                                // Caller wants to appear on home activity.
                                intentActivity.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                            }
                            options = null;
                        }
                    }
```
我们可以看到上面是执行移动操作的代码，要想进入这个代码段需要通过千前置条件的判断，这时候我们注意这个条件
sourceRecord == null
不知道大家发现重点没。在情况1和情况3有个重要的区别就是 sourceRecord 是不同的，在情况1中是动态注册的，并且是用的a1的Context进行启动的，所以这时候 sourceRecord 是不为null的，而在静态注册中我们使用的是application的Context来启动的，所以这时候 sourceRecord 是null，那么现在我们原因就找到了，sourceRecord 是否为null 导致的。所以如果是在情况1中我们可以尝试使用Applicaton的Context来启动就可以达到用动态注册也能实现跳转的问题（当然这时候需要给intent添加new_task flag）。









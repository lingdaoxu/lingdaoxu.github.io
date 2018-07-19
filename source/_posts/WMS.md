title: WMS
author: 道墟
date: 2018-07-13 11:04:38
tags:
---
# 一. relayoutWindow 
> 修改指定窗口的布局参数

# 1.0 参数列表分析
|参数| 释义|
|------------ | ------------ |
|Session session| 调用者所在进程的session示例|
|IWindow client|    需要进行relayout的窗口|
|int seq|       状态栏和导航栏可见性相关的 符号|
|WindowManager.LayoutParams attrs|     窗口的新布局属性  根据attrs 重新布局一个窗口|
|int requestedWidth| 客户端需求的宽度|
|int requestedHeight| 客户端需求的高度|
|int viewVisibility| 窗口的可见性|
|int flags| 定义一些布局行为|
|Rect outFrame| 返回给调用者的实例，保存了窗口重新布局后的大小和位置|
|Rect outOverscanInsets| 可以绘制内容的矩形边界|
|Rect outContentInsets| 可视矩形边界到 mframe的像素差|
|Rect outVisibleInsets||
|Rect outStableInsets||
|Rect outOutsets||
|Configuration outConfig| 重新布局后，wms为窗口计算出来的Configuration|
|Surface outSurface| 用来接收分配的 Surface|

# 1.1 权限相关检查
```java
        ////Log.i(TAG, "relayoutWindow");
        boolean toBeDisplayed = false;
        boolean inTouchMode;
        boolean configChanged;
        boolean surfaceChanged = false;
        boolean animating;

        //权限检查
        boolean hasStatusBarPermission =
                mContext.checkCallingOrSelfPermission(android.Manifest.permission.STATUS_BAR)
                        == PackageManager.PERMISSION_GRANTED;

        long origId = Binder.clearCallingIdentity();
```
# 1.2 根据用户传入的参数更新windowState相关属性

```java
        //下面操作（一直到方法结束）是在锁住mWindowMap情况下完成的。在wms中，几乎所有的操作都是在锁住mWindowMap下完成的。
        synchronized(mWindowMap) {
            //获取需要更新的 WindowState
            WindowState win = windowForClientLocked(session, client, false);
            if (win == null) {
                return 0;
            }


            WindowStateAnimator winAnimator = win.mWinAnimator;
            
            //更新win的宽度和高度为客户端请求的大小
            if (viewVisibility != View.GONE && (win.mRequestedWidth != requestedWidth
                    || win.mRequestedHeight != requestedHeight)) {
                win.mLayoutNeeded = true;
                win.mRequestedWidth = requestedWidth;
                win.mRequestedHeight = requestedHeight;
            }

            if (attrs != null) {
                // attrs.type：TYPE_SYSTEM_OVERLAY 和 TYPE_SECURE_SYSTEM_OVERLAY 类型，attrs.flags 设置为不接受输入事件
                // attrs.type：TYPE_STATUS_BAR，mKeyguardHidden 为true 则设置 attrs.flags 清除 FLAG_SHOW_WALLPAPER
                mPolicy.adjustWindowParamsLw(attrs);
            }

            //检查权限，并设置 标示位 mSystemUiVisibility
            // if they don't have the permission, mask out the status bar bits
            int systemUiVisibility = 0;
            if (attrs != null) {
                systemUiVisibility = (attrs.systemUiVisibility|attrs.subtreeSystemUiVisibility);
                if ((systemUiVisibility & StatusBarManager.DISABLE_MASK) != 0) {
                    if (!hasStatusBarPermission) {
                        systemUiVisibility &= ~StatusBarManager.DISABLE_MASK;
                    }
                }
            }

            if (attrs != null && seq == win.mSeq) {
                win.mSystemUiVisibility = systemUiVisibility;
            }
                
            //判断是否需要延迟销毁
            winAnimator.mSurfaceDestroyDeferred =
                    (flags&WindowManagerGlobal.RELAYOUT_DEFER_SURFACE_DESTROY) != 0;

            int attrChanges = 0;
            int flagChanges = 0;
            if (attrs != null) {
                if (win.mAttrs.type != attrs.type) {
                    throw new IllegalArgumentException(
                            "Window type can not be changed after the window is added.");
                }
                flagChanges = win.mAttrs.flags ^= attrs.flags;
                attrChanges = win.mAttrs.copyFrom(attrs);
                if ((attrChanges & (WindowManager.LayoutParams.LAYOUT_CHANGED
                        | WindowManager.LayoutParams.SYSTEM_UI_VISIBILITY_CHANGED)) != 0) {
                    win.mLayoutNeeded = true;
                }
            }

            if (DEBUG_LAYOUT) Slog.v(TAG, "Relayout " + win + ": viewVisibility=" + viewVisibility
                    + " req=" + requestedWidth + "x" + requestedHeight + " " + win.mAttrs);

            win.mEnforceSizeCompat =
                    (win.mAttrs.privateFlags & PRIVATE_FLAG_COMPATIBLE_WINDOW) != 0;
            //更新透明度信息
            if ((attrChanges & WindowManager.LayoutParams.ALPHA_CHANGED) != 0) {
                winAnimator.mAlpha = attrs.alpha;
            }

            final boolean scaledWindow =
                ((win.mAttrs.flags & WindowManager.LayoutParams.FLAG_SCALED) != 0);

            if (scaledWindow) {
                // requested{Width|Height} Surface's physical size
                // attrs.{width|height} Size on screen
                win.mHScale = (attrs.width  != requestedWidth)  ?
                        (attrs.width  / (float)requestedWidth) : 1.0f;
                win.mVScale = (attrs.height != requestedHeight) ?
                        (attrs.height / (float)requestedHeight) : 1.0f;
            } else {
                win.mHScale = win.mVScale = 1;
            }

            boolean imMayMove = (flagChanges & (FLAG_ALT_FOCUSABLE_IM | FLAG_NOT_FOCUSABLE)) != 0;

            final boolean isDefaultDisplay = win.isDefaultDisplay();
            boolean focusMayChange = isDefaultDisplay && (win.mViewVisibility != viewVisibility
                    || ((flagChanges & FLAG_NOT_FOCUSABLE) != 0)
                    || (!win.mRelayoutCalled));

            boolean wallpaperMayMove = win.mViewVisibility != viewVisibility
                    && (win.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0;
            wallpaperMayMove |= (flagChanges & FLAG_SHOW_WALLPAPER) != 0;
            if ((flagChanges & FLAG_SECURE) != 0 && winAnimator.mSurfaceControl != null) {
                winAnimator.mSurfaceControl.setSecure(isSecureLocked(win));
            }

            win.mRelayoutCalled = true;
            final int oldVisibility = win.mViewVisibility;
            win.mViewVisibility = viewVisibility;
            if (DEBUG_SCREEN_ON) {
                RuntimeException stack = new RuntimeException();
                stack.fillInStackTrace();
                Slog.i(TAG, "Relayout " + win + ": oldVis=" + oldVisibility
                        + " newVis=" + viewVisibility, stack);
            }
```
# 1.3 根据window可见性更新或创建Surface及启动动画效果
> 动画系统在下一篇里面具体介绍

```java
            
            if (viewVisibility == View.VISIBLE &&
                    (win.mAppToken == null || !win.mAppToken.clientHidden)) {
                toBeDisplayed = !win.isVisibleLw();
                if (win.mExiting) {
                    winAnimator.cancelExitAnimationForNextAnimationLocked();
                    win.mExiting = false;
                }
                if (win.mDestroying) {
                    win.mDestroying = false;
                    mDestroySurface.remove(win);
                }
                if (oldVisibility == View.GONE) {
                    winAnimator.mEnterAnimationPending = true;
                }
                winAnimator.mEnteringAnimation = true;
                if (toBeDisplayed) {
                    if ((win.mAttrs.softInputMode & SOFT_INPUT_MASK_ADJUST)
                            == SOFT_INPUT_ADJUST_RESIZE) {
                        win.mLayoutNeeded = true;
                    }
                    if (win.isDrawnLw() && okToDisplay()) {
                        winAnimator.applyEnterAnimationLocked();
                    }
                    if ((win.mAttrs.flags
                            & WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON) != 0) {
                        if (DEBUG_VISIBILITY) Slog.v(TAG,
                                "Relayout window turning screen on: " + win);
                        win.mTurnOnScreen = true;
                    }
                    if (win.isConfigChanged()) {
                        if (DEBUG_CONFIGURATION) Slog.i(TAG, "Window " + win
                                + " visible with new config: " + mCurConfiguration);
                        outConfig.setTo(mCurConfiguration);
                    }
                }
                if ((attrChanges&WindowManager.LayoutParams.FORMAT_CHANGED) != 0) {
                    // If the format can be changed in place yaay!
                    // If not, fall back to a surface re-build
                    if (!winAnimator.tryChangeFormatInPlaceLocked()) {
                        winAnimator.destroySurfaceLocked();
                        toBeDisplayed = true;
                        surfaceChanged = true;
                    }
                }
                try {
                    if (!win.mHasSurface) {
                        surfaceChanged = true;
                    }
                    SurfaceControl surfaceControl = winAnimator.createSurfaceLocked();
                    if (surfaceControl != null) {
                        outSurface.copyFrom(surfaceControl);
                        if (SHOW_TRANSACTIONS) Slog.i(TAG,
                                "  OUT SURFACE " + outSurface + ": copied");
                    } else {
                        // For some reason there isn't a surface.  Clear the
                        // caller's object so they see the same state.
                        outSurface.release();
                    }
                } catch (Exception e) {
                    mInputMonitor.updateInputWindowsLw(true /*force*/);

                    Slog.w(TAG, "Exception thrown when creating surface for client "
                             + client + " (" + win.mAttrs.getTitle() + ")",
                             e);
                    Binder.restoreCallingIdentity(origId);
                    return 0;
                }
                if (toBeDisplayed) {
                    focusMayChange = isDefaultDisplay;
                }
                if (win.mAttrs.type == TYPE_INPUT_METHOD
                        && mInputMethodWindow == null) {
                    mInputMethodWindow = win;
                    imMayMove = true;
                }
                if (win.mAttrs.type == TYPE_BASE_APPLICATION
                        && win.mAppToken != null
                        && win.mAppToken.startingWindow != null) {
                    // Special handling of starting window over the base
                    // window of the app: propagate lock screen flags to it,
                    // to provide the correct semantics while starting.
                    final int mask =
                        WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                        | WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD
                        | WindowManager.LayoutParams.FLAG_ALLOW_LOCK_WHILE_SCREEN_ON;
                    WindowManager.LayoutParams sa = win.mAppToken.startingWindow.mAttrs;
                    sa.flags = (sa.flags&~mask) | (win.mAttrs.flags&mask);
                }
            } else {
            
            
                winAnimator.mEnterAnimationPending = false;
                winAnimator.mEnteringAnimation = false;
                if (winAnimator.mSurfaceControl != null) {
                    if (DEBUG_VISIBILITY) Slog.i(TAG, "Relayout invis " + win
                            + ": mExiting=" + win.mExiting);
                    // If we are not currently running the exit animation, we
                    // need to see about starting one.
                    if (!win.mExiting) {
                        surfaceChanged = true;
                        // Try starting an animation; if there isn't one, we
                        // can destroy the surface right away.
                        int transit = WindowManagerPolicy.TRANSIT_EXIT;
                        if (win.mAttrs.type == TYPE_APPLICATION_STARTING) {
                            transit = WindowManagerPolicy.TRANSIT_PREVIEW_DONE;
                        }
                        if (win.isWinVisibleLw() &&
                                winAnimator.applyAnimationLocked(transit, false)) {
                            focusMayChange = isDefaultDisplay;
                            win.mExiting = true;
                        } else if (win.mWinAnimator.isAnimating()) {
                            // Currently in a hide animation... turn this into
                            // an exit.
                            win.mExiting = true;
                        } else if (win == mWallpaperTarget) {
                            // If the wallpaper is currently behind this
                            // window, we need to change both of them inside
                            // of a transaction to avoid artifacts.
                            win.mExiting = true;
                            win.mWinAnimator.mAnimating = true;
                        } else {
                            if (mInputMethodWindow == win) {
                                mInputMethodWindow = null;
                            }
                            winAnimator.destroySurfaceLocked();
                        }
                        //TODO (multidisplay): Magnification is supported only for the default
                        if (mAccessibilityController != null
                                && win.getDisplayId() == Display.DEFAULT_DISPLAY) {
                            mAccessibilityController.onWindowTransitionLocked(win, transit);
                        }
                    }
                }

                outSurface.release();
                if (DEBUG_VISIBILITY) Slog.i(TAG, "Releasing surface in: " + win);
            }
```
# 1.4 更新窗口焦点、壁纸可见性以及屏幕旋转
```java
            if (focusMayChange) {
                //System.out.println("Focus may change: " + win.mAttrs.getTitle());
                if (updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES,
                        false /*updateInputWindows*/)) {
                    imMayMove = false;
                }
                //System.out.println("Relayout " + win + ": focus=" + mCurrentFocus);
            }

            //对输入法进行更新
            // updateFocusedWindowLocked() already assigned layers so we only need to
            // reassign them at this point if the IM window state gets shuffled
            if (imMayMove && (moveInputMethodWindowsIfNeededLocked(false) || toBeDisplayed)) {
                // Little hack here -- we -should- be able to rely on the
                // function to return true if the IME has moved and needs
                // its layer recomputed.  However, if the IME was hidden
                // and isn't actually moved in the list, its layer may be
                // out of data so we make sure to recompute it.
                assignLayersLocked(win.getWindowList());
            }

            //wallpaperMayMove 表示窗口以 wallpaper作为背景 典型的例子 launcher和home
            if (wallpaperMayMove) {
                getDefaultDisplayContentLocked().pendingLayoutChanges |=
                        WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
            }

            //将窗口所在的 DisplayContent 标记为需要重新布局
            final DisplayContent displayContent = win.getDisplayContent();
            if (displayContent != null) {
                displayContent.layoutNeeded = true;
            }
            win.mGivenInsetsPending = (flags&WindowManagerGlobal.RELAYOUT_INSETS_PENDING) != 0;
            configChanged = updateOrientationFromAppTokensLocked(false);
```
# 1.5 遍历所有的 DisplayContent 的所有窗口，为它们计算布局尺寸，并将布局尺寸设置给它们的surface（详细见第二节）
```java
            performLayoutAndPlaceSurfacesLocked();

            if (toBeDisplayed && win.mIsWallpaper) {
                DisplayInfo displayInfo = getDefaultDisplayInfoLocked();
                updateWallpaperOffsetLocked(win,
                        displayInfo.logicalWidth, displayInfo.logicalHeight, false);
            }
            if (win.mAppToken != null) {
                win.mAppToken.updateReportedVisibilityLocked();
            }
```

# 1.6 返回布局结果
```java
            outFrame.set(win.mCompatFrame);
            outOverscanInsets.set(win.mOverscanInsets);
            outContentInsets.set(win.mContentInsets);
            outVisibleInsets.set(win.mVisibleInsets);
            outStableInsets.set(win.mStableInsets);
            outOutsets.set(win.mOutsets);
            if (localLOGV) Slog.v(
                TAG, "Relayout given client " + client.asBinder()
                + ", requestedWidth=" + requestedWidth
                + ", requestedHeight=" + requestedHeight
                + ", viewVisibility=" + viewVisibility
                + "\nRelayout returning frame=" + outFrame
                + ", surface=" + outSurface);

            if (localLOGV || DEBUG_FOCUS) Slog.v(
                TAG, "Relayout of " + win + ": focusMayChange=" + focusMayChange);

            inTouchMode = mInTouchMode;

            mInputMonitor.updateInputWindowsLw(true /*force*/);

            if (DEBUG_LAYOUT) {
                Slog.v(TAG, "Relayout complete " + win + ": outFrame=" + outFrame.toShortString());
            }
        }
```

# 1.7 向ams更新Configuration 因为屏幕可能旋转
```java
        if (configChanged) {
            sendNewConfiguration();
        }
```

# 二.布局子系统
# 2.1 performLayoutAndPlaceSurfacesLocked

```java
    private final void performLayoutAndPlaceSurfacesLocked() {
        int loopCount = 6;
        do {
            mTraversalScheduled = false;
            performLayoutAndPlaceSurfacesLockedLoop();
            mH.removeMessages(H.DO_TRAVERSAL);
            loopCount--;
        } while (mTraversalScheduled && loopCount > 0);
        mInnerFields.mWallpaperActionPending = false;
    }
```
performLayoutAndPlaceSurfacesLocked方法代码很短只有几行，但是在这之前我们需要搞清楚代码里面这个循环条件的含义，首先我们来关注下 **mTraversalScheduled** 变量，看看这个变量是做什么的：

> mTraversalScheduled 在循环开始的会将值置成false，当值为true的时候才能继续循环，所以在 performLayoutAndPlaceSurfacesLockedLoop 肯定对值进行了重新赋值，下面我们跟着 performLayoutAndPlaceSurfacesLockedLoop来继续跟踪。

# 2.2 performLayoutAndPlaceSurfacesLockedLoop 
```java
private final void performLayoutAndPlaceSurfacesLockedLoop() {
        if (mInLayout) {
            if (DEBUG) {
                throw new RuntimeException("Recursive call!");
            }
            Slog.w(TAG, "performLayoutAndPlaceSurfacesLocked called while in layout. Callers="
                    + Debug.getCallers(3));
            return;
        }

        if (mWaitingForConfig) {
            // Our configuration has changed (most likely rotation), but we
            // don't yet have the complete configuration to report to
            // applications.  Don't do any window layout until we have it.
            return;
        }

        if (!mDisplayReady) {
            // Not yet initialized, nothing to do.
            return;
        }

        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "wmLayout");
        mInLayout = true;
        
        //清理僵尸窗口
        boolean recoveringMemory = false;
        if (!mForceRemoves.isEmpty()) {
            recoveringMemory = true;
            // Wait a little bit for things to settle down, and off we go.
            while (!mForceRemoves.isEmpty()) {
                WindowState ws = mForceRemoves.remove(0);
                Slog.i(TAG, "Force removing: " + ws);
                removeWindowInnerLocked(ws);
            }
            Slog.w(TAG, "Due to memory failure, waiting a bit for next layout");
            Object tmp = new Object();
            synchronized (tmp) {
                try {
                    tmp.wait(250);
                } catch (InterruptedException e) {
                }
            }
        }

        try {
            //主要的布局逻辑 下面会对这个方法进行追踪
            performLayoutAndPlaceSurfacesLockedInner(recoveringMemory);

            mInLayout = false;
            //needsLayout 前面 relayoutWindow 方法中 对 DispplayContent的mLayoutNeeded字段赋值
            if (needsLayout()) {
                //这边对 mTraversalScheduled 进行了赋值为true，同时也对次数进行了控制
                if (++mLayoutRepeatCount < 6) {
                    requestTraversalLocked();
                } else {
                    Slog.e(TAG, "Performed 6 layouts in a row. Skipping");
                    mLayoutRepeatCount = 0;
                }
            } else {
                mLayoutRepeatCount = 0;
            }
            //通知监听者，窗口布局发生了变化
            if (mWindowsChanged && !mWindowChangeListeners.isEmpty()) {
                mH.removeMessages(H.REPORT_WINDOWS_CHANGE);
                mH.sendEmptyMessage(H.REPORT_WINDOWS_CHANGE);
            }
        } catch (RuntimeException e) {
            mInLayout = false;
            Slog.wtf(TAG, "Unhandled exception while laying out windows", e);
        }

        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
    }
```

# 2.3 performLayoutAndPlaceSurfacesLockedInner(布局系统主要逻辑)

# 2.3.0 主要逻辑
```
布局前的预处理
遍历所有 DisplayContent {
    遍历 DisplaContent 下所有的窗口{
        对窗口进行布局
    }
    对布局结果进行检查，是否有必要重新对 DisplayContent 执行布局；
    对 DisplayContent 的布局后处理；
}
完成布局后的策略处理；
```


# 2.3.1 布局前的处理
```java
    // "Something has changed!  Let's make it correct now."
    private final void performLayoutAndPlaceSurfacesLockedInner(boolean recoveringMemory) {
        if (DEBUG_WINDOW_TRACE) {
            Slog.v(TAG, "performLayoutAndPlaceSurfacesLockedInner: entry. Called by "
                    + Debug.getCallers(3));
        }

        //更新处于焦点状态的窗口
        int i;
        boolean updateInputWindowsNeeded = false;

        if (mFocusMayChange) {
            mFocusMayChange = false;
            updateInputWindowsNeeded = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES,
                    false /*updateInputWindows*/);
        }

        //将所有处于退出状态的tokens标记为不可见
        // Initialize state of exiting tokens.
        final int numDisplays = mDisplayContents.size();
        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
            final DisplayContent displayContent = mDisplayContents.valueAt(displayNdx);
            for (i=displayContent.mExitingTokens.size()-1; i>=0; i--) {
                displayContent.mExitingTokens.get(i).hasVisible = false;
            }
        }

        for (int stackNdx = mStackIdToStack.size() - 1; stackNdx >= 0; --stackNdx) {
            // Initialize state of exiting applications.
            final AppTokenList exitingAppTokens =
                    mStackIdToStack.valueAt(stackNdx).mExitingAppTokens;
            for (int tokenNdx = exitingAppTokens.size() - 1; tokenNdx >= 0; --tokenNdx) {
                exitingAppTokens.get(tokenNdx).hasVisible = false;
            }
        }

        //初始化 mInnerFields 中的状态变量 这些在后面的布局后处理中会使用到
        mInnerFields.mHoldScreen = null;
        mInnerFields.mScreenBrightness = -1;
        mInnerFields.mButtonBrightness = -1;
        mInnerFields.mUserActivityTimeout = -1;
        mInnerFields.mObscureApplicationContentOnSecondaryDisplays = false;

        //递增布局序号，wns每进行一次布局都会导致序号递增 AppWindowToken 中也保存了一个相应的布局序号，在布局过程中，wms通过对比这两个序号的值以确定 AppWindowToken 的布局状态是否最新。
        mTransactionSequence++;

        //获取手机屏幕的宽高尺寸，这个尺寸将用来布局水印和StrictMode的红色警告框
        final DisplayContent defaultDisplay = getDefaultDisplayContentLocked();
        final DisplayInfo defaultInfo = defaultDisplay.getDisplayInfo();
        final int defaultDw = defaultInfo.logicalWidth;
        final int defaultDh = defaultInfo.logicalHeight;

        if (SHOW_LIGHT_TRANSACTIONS) Slog.i(TAG,
                ">>> OPEN TRANSACTION performLayoutAndPlaceSurfaces");
        SurfaceControl.openTransaction();
        try {

            //布局水印
            if (mWatermark != null) {
                mWatermark.positionSurface(defaultDw, defaultDh);
            }

            //布局StrictMode的红色警告框
            if (mStrictModeFlash != null) {
                mStrictModeFlash.positionSurface(defaultDw, defaultDh);
            }
            if (mCircularDisplayMask != null) {
                mCircularDisplayMask.positionSurface(defaultDw, defaultDh, mRotation);
            }
            if (mEmulatorDisplayOverlay != null) {
                mEmulatorDisplayOverlay.positionSurface(defaultDw, defaultDh, mRotation);
            }
```
# 2.3.2 布局 DisplayContent
```java
            boolean focusDisplayed = false;

            //遍历 DisplayContent 列表
            for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
                final DisplayContent displayContent = mDisplayContents.valueAt(displayNdx);
                boolean updateAllDrawn = false;

                //获取 displayContent 下的 窗口列表
                WindowList windows = displayContent.getWindowList();
                DisplayInfo displayInfo = displayContent.getDisplayInfo();
                final int displayId = displayContent.getDisplayId();
                //当前 displayContent 所描述显示屏的逻辑尺寸
                final int dw = displayInfo.logicalWidth;
                final int dh = displayInfo.logicalHeight;

                //当前显示屏用于显示应用程序的区域尺寸。逻辑尺寸-系统装饰尺寸（状态栏 导航栏）
                final int innerDw = displayInfo.appWidth;
                final int innerDh = displayInfo.appHeight;

                //用来标识是否是默认的 displayContent（也就是手机屏幕），手机屏幕拥有状态栏、导航栏、并应输入事件，而其他屏幕没有这些特性。
                final boolean isDefaultDisplay = (displayId == Display.DEFAULT_DISPLAY);
				
                //重置 display 的参数
                // Reset for each display.
                mInnerFields.mDisplayHasContent = false;
                mInnerFields.mPreferredRefreshRate = 0;
                mInnerFields.mPreferredModeId = 0;

                int repeats = 0;
                do {
                    repeats++;
                    //限制最多尝试次数
                    if (repeats > 6) {
                        Slog.w(TAG, "Animation repeat aborted after too many iterations");
                        displayContent.layoutNeeded = false;
                        break;
                    }

                    if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("On entry to LockedInner",
                        displayContent.pendingLayoutChanges);

                    // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_WALLPAPER 的处理
                    if ((displayContent.pendingLayoutChanges &
                            WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER) != 0 &&
                            adjustWallpaperWindowsLocked()) {
                        assignLayersLocked(windows);
                        displayContent.layoutNeeded = true;
                    }

                    // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_CONFIG 的处理
                    if (isDefaultDisplay && (displayContent.pendingLayoutChanges
                            & WindowManagerPolicy.FINISH_LAYOUT_REDO_CONFIG) != 0) {
                        if (DEBUG_LAYOUT) Slog.v(TAG, "Computing new config from layout");
                        if (updateOrientationFromAppTokensLocked(true)) {
                            displayContent.layoutNeeded = true;
                            mH.sendEmptyMessage(H.SEND_NEW_CONFIGURATION);
                        }
                    }

                    // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_LAYOUT 的处理
                    if ((displayContent.pendingLayoutChanges
                            & WindowManagerPolicy.FINISH_LAYOUT_REDO_LAYOUT) != 0) {
                        displayContent.layoutNeeded = true;
                    }

                    // FIRST LOOP: Perform a layout, if needed.
                    if (repeats < 4) {
                        //完成对 displayContent 的所有窗口的布局工作 下一个章节分析的重点
                        performLayoutLockedInner(displayContent, repeats == 1,
                                false /*updateInputWindows*/);
                    } else {
                        Slog.w(TAG, "Layout repeat skipped after too many iterations");
                    }

```
# 2.3.2.1 performLayoutLockedInner

```java
    public final void performLayoutLockedInner(final DisplayContent displayContent,
                                    boolean initial, boolean updateInputWindows) {
		Log.i(TAG, "performLayoutLockedInner->displayContent:" + displayContent);
		//if(displayContent.getDisplayId() != 0 && isShowDualScreen()){
		//	return;
		//}

        //layoutNeeded为true才执行布局操作
        if (!displayContent.layoutNeeded) {
            return;
        }
        displayContent.layoutNeeded = false;
        WindowList windows = displayContent.getWindowList();

        //获取是否是手机屏幕参数
        boolean isDefaultDisplay = displayContent.isDefaultDisplay;

        //获取屏幕的信息
        DisplayInfo displayInfo = displayContent.getDisplayInfo();
        final int dw = displayInfo.logicalWidth;
        final int dh = displayInfo.logicalHeight;

        if (mInputConsumer != null) {
            mInputConsumer.layout(dw, dh);
        }

        final int N = windows.size();
        int i;

        if (DEBUG_LAYOUT) {
            Slog.v(TAG, "-------------------------------------");
            Slog.v(TAG, "performLayout: needed="
                    + displayContent.layoutNeeded + " dw=" + dw + " dh=" + dh);
        }

        //通知 PhoneWindowManager 开始布局准备
        mPolicy.beginLayoutLw(isDefaultDisplay, dw, dh, mRotation);
        if (isDefaultDisplay) {
            mSystemDecorLayer = mPolicy.getSystemDecorLayerLw();
            mScreenRect.set(0, 0, dw, dh);
        }

        mPolicy.getContentRectLw(mTmpContentRect);
        displayContent.resize(mTmpContentRect);

        int seq = mLayoutSeq+1;
        if (seq < 0) seq = 0;
        mLayoutSeq = seq;

        boolean behindDream = false;

        //只是第一个非顶级窗口所在的index,用于节省查找时间
        // First perform layout of any root windows (not attached
        // to another window).
        int topAttached = -1;

        //对顶级窗口进行布局
        for (i = N-1; i >= 0; i--) {
            final WindowState win = windows.get(i);

            //获取当前窗口是否可见
            // Don't do layout of a window if it is not visible, or
            // soon won't be visible, to avoid wasting time and funky
            // changes while a window is animating away.
            final boolean gone = (behindDream && mPolicy.canBeForceHidden(win, win.mAttrs))
                    || win.isGoneForLayoutLw();

            if (DEBUG_LAYOUT && !win.mLayoutAttached) {
                Slog.v(TAG, "1ST PASS " + win
                        + ": gone=" + gone + " mHaveFrame=" + win.mHaveFrame
                        + " mLayoutAttached=" + win.mLayoutAttached
                        + " screen changed=" + win.isConfigChanged());
                final AppWindowToken atoken = win.mAppToken;
                if (gone) Slog.v(TAG, "  GONE: mViewVisibility="
                        + win.mViewVisibility + " mRelayoutCalled="
                        + win.mRelayoutCalled + " hidden="
                        + win.mRootToken.hidden + " hiddenRequested="
                        + (atoken != null && atoken.hiddenRequested)
                        + " mAttachedHidden=" + win.mAttachedHidden);
                else Slog.v(TAG, "  VIS: mViewVisibility="
                        + win.mViewVisibility + " mRelayoutCalled="
                        + win.mRelayoutCalled + " hidden="
                        + win.mRootToken.hidden + " hiddenRequested="
                        + (atoken != null && atoken.hiddenRequested)
                        + " mAttachedHidden=" + win.mAttachedHidden);
            }

            //不可见则跳过
            // If this view is GONE, then skip it -- keep the current
            // frame, and let the caller know so they can ignore it
            // if they want.  (We do the normal layout for INVISIBLE
            // windows, since that means "perform layout as normal,
            // just don't display").
            if (!gone || !win.mHaveFrame || win.mLayoutNeeded
                    || ((win.isConfigChanged() || win.setInsetsChanged()) &&
                            ((win.mAttrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0 ||
                            (win.mHasSurface && win.mAppToken != null &&
                            win.mAppToken.layoutConfigChanges)))) {
                if (!win.mLayoutAttached) {
                    //这边只对顶级窗口进行布局处理
                    if (initial) {
                        //Slog.i(TAG, "Window " + this + " clearing mContentChanged - initial");
                        win.mContentChanged = false;
                    }
                    if (win.mAttrs.type == TYPE_DREAM) {
                        // Don't layout windows behind a dream, so that if it
                        // does stuff like hide the status bar we won't get a
                        // bad transition when it goes away.
                        behindDream = true;
                    }
                    win.mLayoutNeeded = false;
                    //通知窗体准备布局
                    win.prelayout();
                    //mPolicy.layoutWindowLw(win, null);
					DisplayInfo defaultDisplayInfo = getDefaultDisplayInfoLocked();
					//调用 PhoneWindowManager 的 layoutWindowLw 开始布局
					mPolicy.layoutWindowLw(win ,null ,defaultDisplayInfo.logicalWidth,defaultDisplayInfo.logicalHeight);
					//更新窗口的布局版本号
                    win.mLayoutSeq = seq;
                    if (DEBUG_LAYOUT) Slog.v(TAG, "  LAYOUT: mFrame="
                            + win.mFrame + " mContainingFrame="
                            + win.mContainingFrame + " mDisplayFrame="
                            + win.mDisplayFrame);
                } else {
                    //当前窗口不是顶级窗口，则记录下这个位置，以便于后面快速定位到这里开始（这边只会记录下 第一个遇到的非顶级窗口）
                    if (topAttached < 0) topAttached = i;
                }
            }
        }

        boolean attachedBehindDream = false;

        //处理非顶级窗口（这边的处理历程和上面的顶级窗口的类似）
        // Now perform layout of attached windows, which usually
        // depend on the position of the window they are attached to.
        // XXX does not deal with windows that are attached to windows
        // that are themselves attached.
        for (i = topAttached; i >= 0; i--) {
            final WindowState win = windows.get(i);

            if (win.mLayoutAttached) {
                if (DEBUG_LAYOUT) Slog.v(TAG, "2ND PASS " + win
                        + " mHaveFrame=" + win.mHaveFrame
                        + " mViewVisibility=" + win.mViewVisibility
                        + " mRelayoutCalled=" + win.mRelayoutCalled);
                // If this view is GONE, then skip it -- keep the current
                // frame, and let the caller know so they can ignore it
                // if they want.  (We do the normal layout for INVISIBLE
                // windows, since that means "perform layout as normal,
                // just don't display").
                if (attachedBehindDream && mPolicy.canBeForceHidden(win, win.mAttrs)) {
                    continue;
                }
                if ((win.mViewVisibility != View.GONE && win.mRelayoutCalled)
                        || !win.mHaveFrame || win.mLayoutNeeded) {
                    if (initial) {
                        //Slog.i(TAG, "Window " + this + " clearing mContentChanged - initial");
                        win.mContentChanged = false;
                    }
                    win.mLayoutNeeded = false;
                    win.prelayout();
                    //mPolicy.layoutWindowLw(win, win.mAttachedWindow);
					DisplayInfo defaultDisplayInfo = getDefaultDisplayInfoLocked();
					//更新布局
		    		mPolicy.layoutWindowLw(win, win.mAttachedWindow,defaultDisplayInfo.logicalWidth,defaultDisplayInfo.logicalHeight);
				//	mPolicy.layoutWindowLw(win, null, defaultDisplayInfo.logicalWidth,defaultDisplayInfo.logicalHeight);
                    win.mLayoutSeq = seq;
                    if (DEBUG_LAYOUT) Slog.v(TAG, "  LAYOUT: mFrame="
                            + win.mFrame + " mContainingFrame="
                            + win.mContainingFrame + " mDisplayFrame="
                            + win.mDisplayFrame);
                }
            } else if (win.mAttrs.type == TYPE_DREAM) {
                // Don't layout windows behind a dream, so that if it
                // does stuff like hide the status bar we won't get a
                // bad transition when it goes away.
                attachedBehindDream = behindDream;
            }
        }


        //布局后窗口的大小和位置可能改变了，所以需要设置需要更新标识
        // Window frames may have changed.  Tell the input dispatcher about it.
        mInputMonitor.setUpdateInputWindowsNeededLw();
        if (updateInputWindows) {
            mInputMonitor.updateInputWindowsLw(false /*force*/);
        }
        //通知布局完成
        mPolicy.finishLayoutLw();
    }
```

# 2.3.2.1.1 PhoneWindowManager 的 beginLayoutLw 方法
这个方法为布局准备了一些用到的参数，这些参数描述了屏幕上的8个矩形区域，这8个区域构成了 PhoneWindowManager 布局窗口的准绳。
- 描述整个屏幕的逻辑显示区域
	- mUnrestrictedScreenLeft
	- mUnrestrictedScreenTop
	- mUnrestrictedScreenWidth
	- mUnrestrictedScreenHeight
- 描述了屏幕除去导航栏之后的区域，导航栏不可见的时候，跟上面的值是相等的
	- mRestrictedScreenLeft
	- mRestrictedScreenTop
	- mRestrictedScreenWidth
	- mRestrictedScreenHeight

- 描述整个屏幕的逻辑显示区域
	- mStableFullscreenLeft
	- mStableFullscreenTop
	- mStableFullscreenRight
	- mStableFullscreenBottom

- 描述排除状态栏和导航栏之外的区域。并且不受可见性影响
	- mStableLeft
	- mStableTop
	- mStableRight
	- mStableBottom

StableFull和Stable不直接参与窗口的布局过程。只是为了提供一个不受可见性影响的显示区域大小和位置

- Dock区域用来描述可用来放置停靠窗口的区域。停靠指的是显示在屏幕某一侧的半屏窗口。Dock区域主要用途是用来作为输入法窗口的布局容器。
	- mDockLeft
	- mDockTop
	- mDockRight
	- mDockBottom

- 描述屏幕中排除状态栏、导航栏、输入法后的显示区域。
	- mContentLeft
	- mContentTop
	- mContentRight
	- mContentBottom

- 与 Content 区域一样，大部分情况下，大部分情况下，cur区域与Content区域是相同的。
	- mCurLeft
	- mCurTop
	- mCurRight
	- mCurBottom

cur区域与Content区域和别的区域计算的方法不一样，这两个的区域不是在 beginLayoutLw 中被确定下来。因为这两个区域受输入法窗口尺寸的影响，所以需要在输入法窗口完成布局后，在 offsetInputMethodWindowLw 方法中得到最终的位置。在此之前，它们与Dock区域是一样的。

- 与Dock区域绝大多数情况是相同的。状态栏和导航栏的可见性发生变化时会有一个淡入淡出的动画效果，这两个区域在这个过程中是不一致的。Dock认为这时候，导航栏和状态栏是可见的，system则认为是不可见的。
	- mSystemLeft
	- mSystemTop
	- mSystemRight
	- mSystemBottom
	

上面这些值可以使用dumpsys window来查看 

```
WINDOW MANAGER POLICY STATE (dumpsys window policy)
    mSafeMode=false mSystemReady=true mSystemBooted=true
    mLidState=-1 mLidOpenRotation=-1 mCameraLensCoverState=-1 mHdmiPlugged=false
    mLastSystemUiFlags=0x8600 mResettingSystemUiFlags=0x0 mForceClearedSystemUiFlags=0x0
    mWakeGestureEnabledSetting=true
    mSupportAutoRotation=true
    mUiMode=1 mDockMode=0 mCarDockRotation=-1 mDeskDockRotation=-1
    mUserRotationMode=0 mUserRotation=0 mAllowAllRotations=0
    mCurrentAppOrientation=-1
    mCarDockEnablesAccelerometer=true mDeskDockEnablesAccelerometer=true
    mLidKeyboardAccessibility=0 mLidNavigationAccessibility=0 mLidControlsSleep=false
    mShortPressOnPowerBehavior=1 mLongPressOnPowerBehavior=1
    mDoublePressOnPowerBehavior=0 mTriplePressOnPowerBehavior=0
    mHasSoftInput=true
    mAwake=true
    mScreenOnEarly=true mScreenOnFully=true
    mKeyguardDrawComplete=true mWindowManagerDrawComplete=true
    mOrientationSensorEnabled=true
    mOverscanScreen=(0,0) 1080x1920
    mRestrictedOverscanScreen=(0,0) 1080x1794
    mUnrestrictedScreen=(0,0) 1080x1920
    mRestrictedScreen=(0,0) 1080x1794
    mStableFullscreen=(0,0)-(1080,1794)
    mStable=(0,63)-(1080,1794)
    mSystem=(0,0)-(1080,1920)
    mCur=(0,63)-(1080,1794)
    mContent=(0,63)-(1080,1794)
    mVoiceContent=(0,63)-(1080,1794)
    mDock=(0,63)-(1080,1794)
    mDockLayer=268435456 mStatusBarLayer=161000
    mShowingLockscreen=false mShowingDream=false mDreamingLockscreen=false mDreamingSleepToken=null
    mStatusBar=Window{285e842 u0 StatusBar} isStatusBarKeyguard=false
    mNavigationBar=Window{7e294a7 u0 NavigationBar}
    mFocusedWindow=Window{f3be5be u0 com.newland.kotlindagger2/com.newland.kotlindagger2.MainActivity}
    mFocusedApp=Token{2ecfac5 ActivityRecord{c866d3c u0 com.newland.kotlindagger2/.MainActivity t204}}
    mTopFullscreenOpaqueWindowState=Window{f3be5be u0 com.newland.kotlindagger2/com.newland.kotlindagger2.MainActivity}
    mTopFullscreenOpaqueOrDimmingWindowState=Window{f3be5be u0 com.newland.kotlindagger2/com.newland.kotlindagger2.MainActivity}
    mTopIsFullscreen=false mHideLockScreen=false
    mForceStatusBar=false mForceStatusBarFromKeyguard=false
    mDismissKeyguard=0 mWinDismissingKeyguard=null mHomePressed=false
    mAllowLockscreenWhenOn=false mLockScreenTimeout=60000 mLockScreenTimerActive=false
    mEndcallBehavior=2 mIncallPowerBehavior=1 mLongPressOnHomeBehavior=0
    mLandscapeRotation=1 mSeascapeRotation=3
    mPortraitRotation=0 mUpsideDownRotation=2
    mDemoHdmiRotation=1 mDemoHdmiRotationLock=false
    mUndockedHdmiRotation=-1
    mKeyMapping.size=0
    BarController.StatusBar
      mState=WINDOW_STATE_SHOWING
      mTransientBar=TRANSIENT_BAR_NONE
    BarController.NavigationBar
      mState=WINDOW_STATE_SHOWING
      mTransientBar=TRANSIENT_BAR_NONE
    PolicyControl.sImmersiveStatusFilter=null
    PolicyControl.sImmersiveNavigationFilter=null
    PolicyControl.sImmersivePreconfirmationsFilter=null
    WakeGestureListener
      mTriggerRequested=false
      mSensor=null
    WindowOrientationListener
      mEnabled=true
      mCurrentRotation=0
      mSensor={Sensor name="Goldfish 3-axis Accelerometer", vendor="The Android Open Source Project", version=1, type=1, maxRange=2.8, resolution=2.480159E-4, power=3.0, minDelay=10000}
      mRate=2
      mProposedRotation=0
      mPredictedRotation=0
      mLastFilteredX=0.0
      mLastFilteredY=9.77631
      mLastFilteredZ=0.812348
      mLastFilteredTimestampNanos=72661454232816 (4.412ms ago)
      mTiltHistory={last: 5.0}
      mFlat=false
      mSwinging=false
      mAccelerating=false
      mOverhead=false
      mTouched=false
      mTiltToleranceConfig=[[-25, 70], [-25, 65], [-25, 60], [-25, 65]]
```


```java
    public void beginLayoutLw(boolean isDefaultDisplay, int displayWidth, int displayHeight,
                              int displayRotation) {
        final int overscanLeft, overscanTop, overscanRight, overscanBottom;
        if (isDefaultDisplay) {
            switch (displayRotation) {
                case Surface.ROTATION_90:
                    overscanLeft = mOverscanTop;
                    overscanTop = mOverscanRight;
                    overscanRight = mOverscanBottom;
                    overscanBottom = mOverscanLeft;
                    break;
                case Surface.ROTATION_180:
                    overscanLeft = mOverscanRight;
                    overscanTop = mOverscanBottom;
                    overscanRight = mOverscanLeft;
                    overscanBottom = mOverscanTop;
                    break;
                case Surface.ROTATION_270:
                    overscanLeft = mOverscanBottom;
                    overscanTop = mOverscanLeft;
                    overscanRight = mOverscanTop;
                    overscanBottom = mOverscanRight;
                    break;
                default:
                    overscanLeft = mOverscanLeft;
                    overscanTop = mOverscanTop;
                    overscanRight = mOverscanRight;
                    overscanBottom = mOverscanBottom;
                    break;
            }
        } else {
            overscanLeft = 0;
            overscanTop = 0;
            overscanRight = 0;
            overscanBottom = 0;
        }
        mOverscanScreenLeft = mRestrictedOverscanScreenLeft = 0;
        mOverscanScreenTop = mRestrictedOverscanScreenTop = 0;
        mOverscanScreenWidth = mRestrictedOverscanScreenWidth = displayWidth;
        mOverscanScreenHeight = mRestrictedOverscanScreenHeight = displayHeight;
        mSystemLeft = 0;
        mSystemTop = 0;
        mSystemRight = displayWidth;
        mSystemBottom = displayHeight;
        mUnrestrictedScreenLeft = overscanLeft;
        mUnrestrictedScreenTop = overscanTop;
        mUnrestrictedScreenWidth = displayWidth - overscanLeft - overscanRight;
        mUnrestrictedScreenHeight = displayHeight - overscanTop - overscanBottom;
        mRestrictedScreenLeft = mUnrestrictedScreenLeft;
        mRestrictedScreenTop = mUnrestrictedScreenTop;
        mRestrictedScreenWidth = mSystemGestures.screenWidth = mUnrestrictedScreenWidth;
        mRestrictedScreenHeight = mSystemGestures.screenHeight = mUnrestrictedScreenHeight;
        mDockLeft = mContentLeft = mVoiceContentLeft = mStableLeft = mStableFullscreenLeft
                = mCurLeft = mUnrestrictedScreenLeft;
        mDockTop = mContentTop = mVoiceContentTop = mStableTop = mStableFullscreenTop
                = mCurTop = mUnrestrictedScreenTop;
        mDockRight = mContentRight = mVoiceContentRight = mStableRight = mStableFullscreenRight
                = mCurRight = displayWidth - overscanRight;
        mDockBottom = mContentBottom = mVoiceContentBottom = mStableBottom = mStableFullscreenBottom
                = mCurBottom = displayHeight - overscanBottom;
        mDockLayer = 0x10000000;
        mStatusBarLayer = -1;

        // start with the current dock rect, which will be (0,0,displayWidth,displayHeight)
        final Rect pf = mTmpParentFrame;
        final Rect df = mTmpDisplayFrame;
        final Rect of = mTmpOverscanFrame;
        final Rect vf = mTmpVisibleFrame;
        final Rect dcf = mTmpDecorFrame;
        pf.left = df.left = of.left = vf.left = mDockLeft;
        pf.top = df.top = of.top = vf.top = mDockTop;
        pf.right = df.right = of.right = vf.right = mDockRight;
        pf.bottom = df.bottom = of.bottom = vf.bottom = mDockBottom;
        dcf.setEmpty();  // Decor frame N/A for system bars.

        if (isDefaultDisplay) {
            // For purposes of putting out fake window up to steal focus, we will
            // drive nav being hidden only by whether it is requested.
            final int sysui = mLastSystemUiFlags;
            boolean navVisible = (sysui & View.SYSTEM_UI_FLAG_HIDE_NAVIGATION) == 0;
            boolean navTranslucent = (sysui
                    & (View.NAVIGATION_BAR_TRANSLUCENT | View.SYSTEM_UI_TRANSPARENT)) != 0;
            boolean immersive = (sysui & View.SYSTEM_UI_FLAG_IMMERSIVE) != 0;
            boolean immersiveSticky = (sysui & View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY) != 0;
            boolean navAllowedHidden = immersive || immersiveSticky;
            navTranslucent &= !immersiveSticky;  // transient trumps translucent
            boolean isKeyguardShowing = isStatusBarKeyguard() && !mHideLockScreen;
            if (!isKeyguardShowing) {
                navTranslucent &= areTranslucentBarsAllowed();
            }

            // When the navigation bar isn't visible, we put up a fake
            // input window to catch all touch events.  This way we can
            // detect when the user presses anywhere to bring back the nav
            // bar and ensure the application doesn't see the event.
            if (navVisible || navAllowedHidden) {
                if (mHideNavFakeWindow != null) {
                    mHideNavFakeWindow.dismiss();
                    mHideNavFakeWindow = null;
                }
            } else if (mHideNavFakeWindow == null) {
                mHideNavFakeWindow = mWindowManagerFuncs.addFakeWindow(
                        mHandler.getLooper(), mHideNavInputEventReceiverFactory,
                        "hidden nav", WindowManager.LayoutParams.TYPE_HIDDEN_NAV_CONSUMER, 0,
                        0, false, false, true);
            }

            // For purposes of positioning and showing the nav bar, if we have
            // decided that it can't be hidden (because of the screen aspect ratio),
            // then take that into account.
            navVisible |= !canHideNavigationBar();
            if((mLastSystemUiFlags & View.SYSTEM_UI_FLAG_SHOW_FULLSCREEN) != 0){
                navVisible = false;
            }

            boolean updateSysUiVisibility = false;
            if (mNavigationBar != null) {
                boolean transientNavBarShowing = mNavigationBarController.isTransientShowing();
                // Force the navigation bar to its appropriate place and
                // size.  We need to do this directly, instead of relying on
                // it to bubble up from the nav bar, because this needs to
                // change atomically with screen rotations.
                mNavigationBarOnBottom = (!mNavigationBarCanMove || displayWidth < displayHeight);
                if (mNavigationBarOnBottom) {
                    // It's a system nav bar or a portrait screen; nav bar goes on bottom.
                    int top = displayHeight - overscanBottom
                            - mNavigationBarHeightForRotation[displayRotation];
                    mTmpNavigationFrame.set(0, top, displayWidth, displayHeight - overscanBottom);
                    mStableBottom = mStableFullscreenBottom = mTmpNavigationFrame.top;
                    if (transientNavBarShowing) {
                        mNavigationBarController.setBarShowingLw(true);
                    } else if (navVisible) {
                        mNavigationBarController.setBarShowingLw(true);
                        mDockBottom = mTmpNavigationFrame.top;
                        mRestrictedScreenHeight = mDockBottom - mRestrictedScreenTop;
                        mRestrictedOverscanScreenHeight = mDockBottom - mRestrictedOverscanScreenTop;
                    } else {
                        // We currently want to hide the navigation UI.
                        mNavigationBarController.setBarShowingLw(false);
                    }
                    if (navVisible && !navTranslucent && !navAllowedHidden
                            && !mNavigationBar.isAnimatingLw()
                            && !mNavigationBarController.wasRecentlyTranslucent()) {
                        // If the opaque nav bar is currently requested to be visible,
                        // and not in the process of animating on or off, then
                        // we can tell the app that it is covered by it.
                        mSystemBottom = mTmpNavigationFrame.top;
                    }
                } else {
                    // Landscape screen; nav bar goes to the right.
                    int left = displayWidth - overscanRight
                            - mNavigationBarWidthForRotation[displayRotation];
                    mTmpNavigationFrame.set(left, 0, displayWidth - overscanRight, displayHeight);
                    mStableRight = mStableFullscreenRight = mTmpNavigationFrame.left;
                    if (transientNavBarShowing) {
                        mNavigationBarController.setBarShowingLw(true);
                    } else if (navVisible) {
                        mNavigationBarController.setBarShowingLw(true);
                        mDockRight = mTmpNavigationFrame.left;
                        mRestrictedScreenWidth = mDockRight - mRestrictedScreenLeft;
                        mRestrictedOverscanScreenWidth = mDockRight - mRestrictedOverscanScreenLeft;
                    } else {
                        // We currently want to hide the navigation UI.
                        mNavigationBarController.setBarShowingLw(false);
                    }
                    if (navVisible && !navTranslucent && !mNavigationBar.isAnimatingLw()
                            && !mNavigationBarController.wasRecentlyTranslucent()) {
                        // If the nav bar is currently requested to be visible,
                        // and not in the process of animating on or off, then
                        // we can tell the app that it is covered by it.
                        mSystemRight = mTmpNavigationFrame.left;
                    }
                }
                // Make sure the content and current rectangles are updated to
                // account for the restrictions from the navigation bar.
                mContentTop = mVoiceContentTop = mCurTop = mDockTop;
                mContentBottom = mVoiceContentBottom = mCurBottom = mDockBottom;
                mContentLeft = mVoiceContentLeft = mCurLeft = mDockLeft;
                mContentRight = mVoiceContentRight = mCurRight = mDockRight;
                mStatusBarLayer = mNavigationBar.getSurfaceLayer();
                // And compute the final frame.
                mNavigationBar.computeFrameLw(mTmpNavigationFrame, mTmpNavigationFrame,
                        mTmpNavigationFrame, mTmpNavigationFrame, mTmpNavigationFrame, dcf,
                        mTmpNavigationFrame);
                if (DEBUG_LAYOUT) Slog.i(TAG, "mNavigationBar frame: " + mTmpNavigationFrame);
                if (mNavigationBarController.checkHiddenLw()) {
                    updateSysUiVisibility = true;
                }
            }
            if (DEBUG_LAYOUT) Slog.i(TAG, String.format("mDock rect: (%d,%d - %d,%d)",
                    mDockLeft, mDockTop, mDockRight, mDockBottom));

            // decide where the status bar goes ahead of time
            if (mStatusBar != null) {
                // apply any navigation bar insets
                pf.left = df.left = of.left = mUnrestrictedScreenLeft;
                pf.top = df.top = of.top = mUnrestrictedScreenTop;
                pf.right = df.right = of.right = mUnrestrictedScreenWidth + mUnrestrictedScreenLeft;
                pf.bottom = df.bottom = of.bottom = mUnrestrictedScreenHeight
                        + mUnrestrictedScreenTop;
                vf.left = mStableLeft;
                vf.top = mStableTop;
                vf.right = mStableRight;
                vf.bottom = mStableBottom;

                mStatusBarLayer = mStatusBar.getSurfaceLayer();

                // Let the status bar determine its size.
                mStatusBar.computeFrameLw(pf, df, vf, vf, vf, dcf, vf);

                // For layout, the status bar is always at the top with our fixed height.
                mStableTop = mUnrestrictedScreenTop + mStatusBarHeight;

                boolean statusBarTransient = (sysui & View.STATUS_BAR_TRANSIENT) != 0;
                boolean statusBarTranslucent = (sysui
                        & (View.STATUS_BAR_TRANSLUCENT | View.SYSTEM_UI_TRANSPARENT)) != 0;
                if (!isKeyguardShowing) {
                    statusBarTranslucent &= areTranslucentBarsAllowed();
                }

                // If the status bar is hidden, we don't want to cause
                // windows behind it to scroll.
                if (mStatusBar.isVisibleLw() && !statusBarTransient) {
                    // Status bar may go away, so the screen area it occupies
                    // is available to apps but just covering them when the
                    // status bar is visible.
                    mDockTop = mUnrestrictedScreenTop + mStatusBarHeight;

                    mContentTop = mVoiceContentTop = mCurTop = mDockTop;
                    mContentBottom = mVoiceContentBottom = mCurBottom = mDockBottom;
                    mContentLeft = mVoiceContentLeft = mCurLeft = mDockLeft;
                    mContentRight = mVoiceContentRight = mCurRight = mDockRight;

                    if (DEBUG_LAYOUT) Slog.v(TAG, "Status bar: " +
                        String.format(
                            "dock=[%d,%d][%d,%d] content=[%d,%d][%d,%d] cur=[%d,%d][%d,%d]",
                            mDockLeft, mDockTop, mDockRight, mDockBottom,
                            mContentLeft, mContentTop, mContentRight, mContentBottom,
                            mCurLeft, mCurTop, mCurRight, mCurBottom));
                }
                if (mStatusBar.isVisibleLw() && !mStatusBar.isAnimatingLw()
                        && !statusBarTransient && !statusBarTranslucent
                        && !mStatusBarController.wasRecentlyTranslucent()) {
                    // If the opaque status bar is currently requested to be visible,
                    // and not in the process of animating on or off, then
                    // we can tell the app that it is covered by it.
                    mSystemTop = mUnrestrictedScreenTop + mStatusBarHeight;
                }
                if (mStatusBarController.checkHiddenLw()) {
                    updateSysUiVisibility = true;
                }
            }
            if (updateSysUiVisibility) {
                updateSystemUiVisibilityLw();
            }
        }
    }
```
# 2.3.2.1.2 PhoneWindowManager 的 layoutWindowLw 方法
layoutWindowLw 使用上面那些布局参数来确定一个窗口的位置和尺寸。

```java
    /** {@inheritDoc} */
    @Override
    public void layoutWindowLw(WindowState win, WindowState attached) {

        //状态栏和导航栏不再被布局。因为这两个在 beginLayoutLw 方法中计算布局参数时完成
        // We've already done the navigation bar and status bar. If the status bar can receive
        // input, we need to layout it again to accomodate for the IME window.
        if ((win == mStatusBar && !canReceiveInput(win)) || win == mNavigationBar) {
            return;
        }
        final WindowManager.LayoutParams attrs = win.getAttrs();
        final boolean isDefaultDisplay = win.isDefaultDisplay();
        final boolean needsToOffsetInputMethodTarget = isDefaultDisplay &&
                (win == mLastInputMethodTargetWindow && mLastInputMethodWindow != null);
        if (needsToOffsetInputMethodTarget) {
            if (DEBUG_LAYOUT) Slog.i(TAG, "Offset ime target window by the last ime window state");
            offsetInputMethodWindowLw(mLastInputMethodWindow);
        }

        final int fl = PolicyControl.getWindowFlags(win, attrs);
        final int sim = attrs.softInputMode;
        final int sysUiFl = PolicyControl.getSystemUiVisibility(win, null);

        //提供这四个矩形变量作为缓存，后续的计算中会更新这四个矩形内容
        final Rect pf = mTmpParentFrame;
        final Rect df = mTmpDisplayFrame;
        final Rect of = mTmpOverscanFrame;
        final Rect cf = mTmpContentFrame;
        final Rect vf = mTmpVisibleFrame;
        final Rect dcf = mTmpDecorFrame;
        final Rect sf = mTmpStableFrame;
        //下面是做这些内容的更新
		
		......
		
        //将计算好的矩形交给windowstate,有窗口自己计算自己的布局结果
        win.computeFrameLw(pf, df, of, cf, vf, dcf, sf, osf);

        //完成输入法窗口布局后需要更新 content 和 cur 区域就是在这里完成的
        // Dock windows carve out the bottom of the screen, so normal windows
        // can't appear underneath them.
        if (attrs.type == TYPE_INPUT_METHOD && win.isVisibleOrBehindKeyguardLw()
                && !win.getGivenInsetsPendingLw()) {
            setLastInputMethodWindowLw(null, null);
            offsetInputMethodWindowLw(win);
        }
        if (attrs.type == TYPE_VOICE_INTERACTION && win.isVisibleOrBehindKeyguardLw()
                && !win.getGivenInsetsPendingLw()) {
            offsetVoiceInputWindowLw(win);
        }
    }
```

# 2.3.2.1.2.1 WindowState 的 computeFrameLw 方法

该方法的产出
- mFrame:描述窗口的位置和尺寸
- mContainingFrame 和 mParentFrame 保存pf参数的值
- mDisplayFrame 保存df参数的值
- mContentFrame 表示 可现实内容区域 由cf和mFrame相交得出
- mContentInsets 表示 mContentFrame 和 mFrame 的四条边界之间的距离
- mVisibleFrame 表示当前窗口不被系统窗口所遮挡的区域，由vf和mFrame相交得出
- mVisibleInsets 表示 mContentFrame 和 mFrame 的四条边界之间的距离


```java
    /**
     *
     * @param pf 描述放置窗口的位置和尺寸  LayoutParam 中的x,y,width,height以及gravity等都是相对pf进行计算的。根据窗口类型和flag不同，pf可能是 Restricted Unrestricted Dock Content。
     * @param df 用来限定窗口的最终位置。当窗口通过pf完成位置和尺寸的计算后，需要通过df再进行一次校正，必须完成位于df之内。 绝大部分情况和pf保持一致。
     * @param of
     * @param cf cf不会直接影响窗口布局位置和尺寸，但是影响到窗口内容的绘制。cf表示当前屏幕排除所有系统窗口（状态 导航 输入法）后所留下的矩形区域。根据 LayoutParams.softInputMode以及flag取值不同可能是 Content Dock Restricted
     * @param vf 跟cf一样，表示不被任何系统串口所遮挡的一块矩形区域。根据 LayoutParams.softInputMode 的取值选择与cf保持一致，或者选择cur区域。
     * @param dcf
     * @param sf
     * @param osf
     */
    @Override
    public void computeFrameLw(Rect pf, Rect df, Rect of, Rect cf, Rect vf, Rect dcf, Rect sf,
            Rect osf) {
	
	}
```


# 2.3.3 检查布局结果

这一步主要是对一些flag的上的处理
- FLAG_FORCE_NOT_FULLSCREEN:必须同时显示状态栏等系统窗口
- FLAG_FULLSCREEN：拥有这个flag的满屏窗口被显示的时候，必须隐藏状态栏等系统窗口。这个flag和 FLAG_FORCE_NOT_FULLSCREEN 是冲突的，FLAG_FORCE_NOT_FULLSCREEN的优先级更高
- FLAG_SHOW_WHEN_LOCKED：锁屏状态下，拥有这个flag的窗口被显示时，隐藏锁屏界面，当窗口关闭时显示锁屏界面。这个flag仅能对满屏窗口（不包括系统窗口）起作用。
- FLAG_DISMISS_KEYGUARD:窗口显示时关闭锁屏状态，并且退出窗口也不会进入锁屏状态。不过如果用户设置了密码保护，则必须在输入密码后才能显示出来。这个flag仅能对满屏窗口（不包括系统窗口）起作用。

为什么需要这个步骤：因为这些flag影响了系统窗口、锁屏界面的可见性，也就会影响到窗口的布局过程。然后这些flag生效的条件又是要求窗口能够覆盖屏幕，因此窗口的布局也影响到这些flag的有效性。这两个方面是互相依赖又互相影响的。

解决方案：引入 pendingLayoutChanges 机制

机制原理：首先根据系统窗口、锁屏界面的有效性布局，然后根据这个布局来生效flag,这时候观察系统窗口、锁屏界面是否发生改变。如果可见性发生改变，则通过 pendingLayoutChanges 变量标记下需要额外完成何种工作。完成这些工作后，重新进行布局，再进行检查，如此反复，直到系统窗口和锁屏界面可见性不再发生变化。

```java

			  // 处理 pendingLayoutChanges 字段
              // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_WALLPAPER 的处理
              if ((displayContent.pendingLayoutChanges &
                      WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER) != 0 &&
                      adjustWallpaperWindowsLocked()) {
                  assignLayersLocked(windows);
                  displayContent.layoutNeeded = true;
              }

              // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_CONFIG 的处理
              if (isDefaultDisplay && (displayContent.pendingLayoutChanges
                      & WindowManagerPolicy.FINISH_LAYOUT_REDO_CONFIG) != 0) {
                  if (DEBUG_LAYOUT) Slog.v(TAG, "Computing new config from layout");
                  if (updateOrientationFromAppTokensLocked(true)) {
                      displayContent.layoutNeeded = true;
                      mH.sendEmptyMessage(H.SEND_NEW_CONFIGURATION);
                  }
              }

              // pendingLayoutChanges 含有 FINISH_LAYOUT_REDO_LAYOUT 的处理
              if ((displayContent.pendingLayoutChanges
                      & WindowManagerPolicy.FINISH_LAYOUT_REDO_LAYOUT) != 0) {
                  displayContent.layoutNeeded = true;
              }
			  // performLayoutLockedInner 方法进行布局
			  ...
              //清空 pendingLayoutChanges 字段
              // FIRST AND ONE HALF LOOP: Make WindowManagerPolicy think
              // it is animating.
              displayContent.pendingLayoutChanges = 0;

				
              //PhoneWindowManager 来检查布局（结果检查）
			  //检查内容：状态栏 导航栏可见性是否与顶层窗口的属性冲突，是否需要解除锁屏状态等
              //布局检查只会发生在 手机屏幕上 因为只有手机屏幕才会有 系统窗口和 锁屏界面
			  if (isDefaultDisplay) {
			  
			  	 //初始化检查所需的状态变量
                  mPolicy.beginPostLayoutPolicyLw(dw, dh);
                  for (i = windows.size() - 1; i >= 0; i--) {
                      WindowState w = windows.get(i);
                      if (w.mHasSurface) {
					  		//记录下可能影响到系统窗口或者锁屏界面可见性的窗口
                          mPolicy.applyPostLayoutPolicyLw(w, w.mAttrs, w.mAttachedWindow);
                      }
                  }
                  //finishPostLayoutPolicyLw修改系统窗口或者锁屏界面的可见性，并将改变的结果保存到 pendingLayoutChanges 字段
                  displayContent.pendingLayoutChanges |= mPolicy.finishPostLayoutPolicyLw();
                  if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats(
                      "after finishPostLayoutPolicyLw", displayContent.pendingLayoutChanges);
              }
              //通过 pendingLayoutChanges 来判断是否需要重新检测
          } while (displayContent.pendingLayoutChanges != 0);
```

# 2.3.3.1 beginPostLayoutPolicyLw：初始化参数
```java
    /** {@inheritDoc} */
    @Override
    public void beginPostLayoutPolicyLw(int displayWidth, int displayHeight) {
        //保存显示在最上方的满屏窗口
        mTopFullscreenOpaqueWindowState = null;
        mAppsToBeHidden.clear();
        mAppsThatDismissKeyguard.clear();
        //是否强制显示状态栏等系统窗口
        mForceStatusBar = false;
        //是否在锁屏状态下显示状态看等系统窗口
        mForceStatusBarFromKeyguard = false;
        mForcingShowNavBar = false;
        mForcingShowNavBarLayer = -1;
        //是否需要隐藏锁屏界面
        mHideLockScreen = false;
        //是否允许在点亮状态下自动进行锁屏
        mAllowLockscreenWhenOn = false;
        //是否需要解除锁屏
        mDismissKeyguard = DISMISS_KEYGUARD_NONE;
        //当前是否正在显示锁屏界面
        mShowingLockscreen = false;
        //是否正在显示屏保
        mShowingDream = false;
        mWinShowWhenLocked = null;
        mKeyguardSecure = isKeyguardSecure();
        mKeyguardSecureIncludingHidden = mKeyguardSecure
                && (mKeyguardDelegate != null && mKeyguardDelegate.isShowing());
    }
```

# 2.3.3.2 applyPostLayoutPolicyLw：设置参数

方法输出
- mTopFullscreenOpaqueWindowState:首个满屏窗口
- mForceStatusBar 和 mForceStatusBarFromKeyguard:强制显示系统窗口，对应flag FLAG_FORCE_NOT_FULLSCREEN。这两个变量的区别是 mForceStatusBarFromKeyguard 还要求 .privateFlags & PRIVATE_FLAG_KEYGUARD
- mHideLockScreen：隐藏锁屏界面，只有当首个满屏界面flag有 FLAG_SHOW_WHEN_LOCKED才为true
- mDismissKeyguard:是否执行解锁操作
	- DISMISS_KEYGUARD_NONE：不会解锁
	- DISMISS_KEYGUARD_CONTINUE：维持当前状态
	- DISMISS_KEYGUARD_START：执行解锁操作


```java
    public void applyPostLayoutPolicyLw(WindowState win, WindowManager.LayoutParams attrs,
            WindowState attached) {

        if (DEBUG_LAYOUT) Slog.i(TAG, "Win " + win + ": isVisibleOrBehindKeyguardLw="
                + win.isVisibleOrBehindKeyguardLw());
        final int fl = PolicyControl.getWindowFlags(win, attrs);
        if (mTopFullscreenOpaqueWindowState == null
                && win.isVisibleLw() && attrs.type == TYPE_INPUT_METHOD) {
            mForcingShowNavBar = true;
            mForcingShowNavBarLayer = win.getSurfaceLayer();
        }
        if (attrs.type == TYPE_STATUS_BAR && (attrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0) {
            mForceStatusBarFromKeyguard = true;
        }

        //注意调用本方法的顺序是z轴自上向下的，下面这个判断覆盖了方法的剩下部分，
        // 所以这边可以推断出 只有 mTopFullscreenOpaqueWindowState（第一个可见的满屏窗口）之上的窗口才能影响到系统窗口和锁屏界面的可见性
        if (mTopFullscreenOpaqueWindowState == null &&
                win.isVisibleOrBehindKeyguardLw() && !win.isGoneForLayoutLw()) {

            // FLAG_FORCE_NOT_FULLSCREEN 有这个属性则必须显示系统窗口
            if ((fl & FLAG_FORCE_NOT_FULLSCREEN) != 0) {
                if ((attrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0) {
                    mForceStatusBarFromKeyguard = true;
                } else {
                    mForceStatusBar = true;
                }
            }

            //窗口类型为 PRIVATE_FLAG_KEYGUARD 则表示正在显示锁屏界面
            if ((attrs.privateFlags & PRIVATE_FLAG_KEYGUARD) != 0) {
                mShowingLockscreen = true;
            }

            //是否是应用窗口
            boolean appWindow = attrs.type >= FIRST_APPLICATION_WINDOW
                    && attrs.type < FIRST_SYSTEM_WINDOW;

            if (attrs.type == TYPE_DREAM) {
                // If the lockscreen was showing when the dream started then wait
                // for the dream to draw before hiding the lockscreen.
                if (!mDreamingLockscreen
                        || (win.isVisibleLw() && win.hasDrawnLw())) {
                    mShowingDream = true;
                    appWindow = true;
                }
            }

            final boolean showWhenLocked = (fl & FLAG_SHOW_WHEN_LOCKED) != 0;
            final boolean dismissKeyguard = (fl & FLAG_DISMISS_KEYGUARD) != 0;
            final IApplicationToken appToken = win.getAppToken();

            // For app windows that are not attached, we decide if all windows in the app they
            // represent should be hidden or if we should hide the lockscreen. For attached app
            // windows we defer the decision to the window it is attached to.
            if (appWindow && attached == null) {
                if (showWhenLocked) {
                    // Remove any previous windows with the same appToken.
                    mAppsToBeHidden.remove(appToken);
                    mAppsThatDismissKeyguard.remove(appToken);
                    if (mAppsToBeHidden.isEmpty()) {
                        if (dismissKeyguard && !mKeyguardSecure) {
                            mAppsThatDismissKeyguard.add(appToken);
                        } else {
                            mWinShowWhenLocked = win;
                            mHideLockScreen = true;
                            mForceStatusBarFromKeyguard = false;
                        }
                    }
                } else if (dismissKeyguard) {
                    if (mKeyguardSecure) {
                        mAppsToBeHidden.add(appToken);
                    } else {
                        mAppsToBeHidden.remove(appToken);
                    }
                    mAppsThatDismissKeyguard.add(appToken);
                } else {
                    mAppsToBeHidden.add(appToken);
                }
                //判断是否是满屏的应用窗口
                if (attrs.x == 0 && attrs.y == 0
                        && attrs.width == WindowManager.LayoutParams.MATCH_PARENT
                        && attrs.height == WindowManager.LayoutParams.MATCH_PARENT) {
                    if (DEBUG_LAYOUT) Slog.v(TAG, "Fullscreen window: " + win);
                    //找到首个满屏的应用窗口
                    mTopFullscreenOpaqueWindowState = win;
                    if (!mAppsThatDismissKeyguard.isEmpty() &&
                            mDismissKeyguard == DISMISS_KEYGUARD_NONE) {
                        if (DEBUG_LAYOUT) Slog.v(TAG,
                                "Setting mDismissKeyguard true by win " + win);
                        mDismissKeyguard = mWinDismissingKeyguard == win ?
                                DISMISS_KEYGUARD_CONTINUE : DISMISS_KEYGUARD_START;
                        mWinDismissingKeyguard = win;
                        mForceStatusBarFromKeyguard = mShowingLockscreen && mKeyguardSecure;
                    } else if (mAppsToBeHidden.isEmpty() && showWhenLocked) {
                        //满屏窗口 flag 有 FLAG_SHOW_WHEN_LOCKED
                        if (DEBUG_LAYOUT) Slog.v(TAG,
                                "Setting mHideLockScreen to true by win " + win);
                        mHideLockScreen = true;
                        mForceStatusBarFromKeyguard = false;
                    }
                    if ((fl & FLAG_ALLOW_LOCK_WHILE_SCREEN_ON) != 0) {
                        mAllowLockscreenWhenOn = true;
                    }
                }

                if (mWinShowWhenLocked != null &&
                        mWinShowWhenLocked.getAppToken() != win.getAppToken()) {
                    win.hideLw(false);
                }
            }
        }
    }
```

# 2.3.3.3 finishPostLayoutPolicyLw：调教系统窗口与锁屏界面的可见性

```java
    public int finishPostLayoutPolicyLw() {
        if (mWinShowWhenLocked != null &&
                mWinShowWhenLocked != mTopFullscreenOpaqueWindowState) {
            // A dialog is dismissing the keyguard. Put the wallpaper behind it and hide the
            // fullscreen window.
            // TODO: Make sure FLAG_SHOW_WALLPAPER is restored when dialog is dismissed. Or not.
            mWinShowWhenLocked.getAttrs().flags |= FLAG_SHOW_WALLPAPER;
            mTopFullscreenOpaqueWindowState.hideLw(false);
            mTopFullscreenOpaqueWindowState = mWinShowWhenLocked;
        }

        int changes = 0;
        boolean topIsFullscreen = false;

        final WindowManager.LayoutParams lp = (mTopFullscreenOpaqueWindowState != null)
                ? mTopFullscreenOpaqueWindowState.getAttrs()
                : null;

        // If we are not currently showing a dream then remember the current
        // lockscreen state.  We will use this to determine whether the dream
        // started while the lockscreen was showing and remember this state
        // while the dream is showing.
        if (!mShowingDream) {
            mDreamingLockscreen = mShowingLockscreen;
        }

        if (mStatusBar != null) {
            if (DEBUG_LAYOUT) Slog.i(TAG, "force=" + mForceStatusBar
                    + " forcefkg=" + mForceStatusBarFromKeyguard
                    + " top=" + mTopFullscreenOpaqueWindowState);
            if (mForceStatusBar || mForceStatusBarFromKeyguard) {
                if (DEBUG_LAYOUT) Slog.v(TAG, "Showing status bar: forced");
                if (mStatusBarController.setBarShowingLw(true)) {
                    changes |= FINISH_LAYOUT_REDO_LAYOUT;
                }
                // Maintain fullscreen layout until incoming animation is complete.
                topIsFullscreen = mTopIsFullscreen && mStatusBar.isAnimatingLw();
                // Transient status bar on the lockscreen is not allowed
                if (mForceStatusBarFromKeyguard && mStatusBarController.isTransientShowing()) {
                    mStatusBarController.updateVisibilityLw(false /*transientAllowed*/,
                            mLastSystemUiFlags, mLastSystemUiFlags);
                }
            } else if (mTopFullscreenOpaqueWindowState != null) {
                final int fl = PolicyControl.getWindowFlags(null, lp);
                if (localLOGV) {
                    Slog.d(TAG, "frame: " + mTopFullscreenOpaqueWindowState.getFrameLw()
                            + " shown frame: " + mTopFullscreenOpaqueWindowState.getShownFrameLw());
                    Slog.d(TAG, "attr: " + mTopFullscreenOpaqueWindowState.getAttrs()
                            + " lp.flags=0x" + Integer.toHexString(fl));
                }
                topIsFullscreen = (fl & WindowManager.LayoutParams.FLAG_FULLSCREEN) != 0
                        || (mLastSystemUiFlags & View.SYSTEM_UI_FLAG_FULLSCREEN) != 0;
                // The subtle difference between the window for mTopFullscreenOpaqueWindowState
                // and mTopIsFullscreen is that that mTopIsFullscreen is set only if the window
                // has the FLAG_FULLSCREEN set.  Not sure if there is another way that to be the
                // case though.
                if (mStatusBarController.isTransientShowing()) {
                    if (mStatusBarController.setBarShowingLw(true)) {
                        changes |= FINISH_LAYOUT_REDO_LAYOUT;
                    }
                } else if (topIsFullscreen) {
                    if (DEBUG_LAYOUT) Slog.v(TAG, "** HIDING status bar");
                    if (mStatusBarController.setBarShowingLw(false)) {
                        changes |= FINISH_LAYOUT_REDO_LAYOUT;
                    } else {
                        if (DEBUG_LAYOUT) Slog.v(TAG, "Status bar already hiding");
                    }
                } else {
                    if (DEBUG_LAYOUT) Slog.v(TAG, "** SHOWING status bar: top is not fullscreen");
                    if (mStatusBarController.setBarShowingLw(true)) {
                        changes |= FINISH_LAYOUT_REDO_LAYOUT;
                    }
                }
            }
        }

        if (mTopIsFullscreen != topIsFullscreen) {
            if (!topIsFullscreen) {
                // Force another layout when status bar becomes fully shown.
                changes |= FINISH_LAYOUT_REDO_LAYOUT;
            }
            mTopIsFullscreen = topIsFullscreen;
        }

        // Hide the key guard if a visible window explicitly specifies that it wants to be
        // displayed when the screen is locked.
        if (mKeyguardDelegate != null && mStatusBar != null) {
            if (localLOGV) Slog.v(TAG, "finishPostLayoutPolicyLw: mHideKeyguard="
                    + mHideLockScreen);
            if (mDismissKeyguard != DISMISS_KEYGUARD_NONE && !mKeyguardSecure) {
                mKeyguardHidden = true;
                if (setKeyguardOccludedLw(true)) {
                    changes |= FINISH_LAYOUT_REDO_LAYOUT
                            | FINISH_LAYOUT_REDO_CONFIG
                            | FINISH_LAYOUT_REDO_WALLPAPER;
                }
                if (mKeyguardDelegate.isShowing()) {
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            mKeyguardDelegate.keyguardDone(false, false);
                        }
                    });
                }
            } else if (mHideLockScreen) {
                mKeyguardHidden = true;
                if (setKeyguardOccludedLw(true)) {
                    changes |= FINISH_LAYOUT_REDO_LAYOUT
                            | FINISH_LAYOUT_REDO_CONFIG
                            | FINISH_LAYOUT_REDO_WALLPAPER;
                }
            } else if (mDismissKeyguard != DISMISS_KEYGUARD_NONE) {
                // This is the case of keyguard isSecure() and not mHideLockScreen.
                if (mDismissKeyguard == DISMISS_KEYGUARD_START) {
                    // Only launch the next keyguard unlock window once per window.
                    mKeyguardHidden = false;
                    if (setKeyguardOccludedLw(false)) {
                        changes |= FINISH_LAYOUT_REDO_LAYOUT
                                | FINISH_LAYOUT_REDO_CONFIG
                                | FINISH_LAYOUT_REDO_WALLPAPER;
                    }
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            mKeyguardDelegate.dismiss();
                        }
                    });
                }
            } else {
                mWinDismissingKeyguard = null;
                mKeyguardHidden = false;
                if (setKeyguardOccludedLw(false)) {
                    changes |= FINISH_LAYOUT_REDO_LAYOUT
                            | FINISH_LAYOUT_REDO_CONFIG
                            | FINISH_LAYOUT_REDO_WALLPAPER;
                }
            }
        }

        if ((updateSystemUiVisibilityLw()&SYSTEM_UI_CHANGING_LAYOUT) != 0) {
            // If the navigation bar has been hidden or shown, we need to do another
            // layout pass to update that window.
            changes |= FINISH_LAYOUT_REDO_LAYOUT;
        }

        // update since mAllowLockscreenWhenOn might have changed
        updateLockScreenTimeout();
        return changes;
    }
```
# 2.3.4 布局后处理
为布局好的窗口设置Surface大小和位置，并附加动画效果等。

主要工作内容:
- 设置窗口的遮挡状态
- 从窗口的LayoutParams中提取屏幕亮度、键盘亮度、输入超时等
- 发起和取消 Dimming效果
- 设置窗口 surface的位置和尺寸，位置变化是由动画效果的
- 窗口的客户端已经完成surface的绘制则显示这个窗口

```java
				mInnerFields.mObscured = false;
                mInnerFields.mSyswin = false;
                displayContent.resetDimming();

                // Only used if default window
                final boolean someoneLosingFocus = !mLosingFocus.isEmpty();

                final int N = windows.size();
                for (i=N-1; i>=0; i--) {
                    WindowState w = windows.get(i);
                    final TaskStack stack = w.getStack();
                    if (stack == null && w.getAttrs().type != TYPE_PRIVATE_PRESENTATION) {
                        continue;
                    }

                    final boolean obscuredChanged = w.mObscured != mInnerFields.mObscured;

                    // Update effect.
                    w.mObscured = mInnerFields.mObscured;
                    if (!mInnerFields.mObscured) {
                        handleNotObscuredLocked(w, innerDw, innerDh);
                    }

                    if (stack != null && !stack.testDimmingTag()) {
                        handleFlagDimBehind(w);
                    }

                    if (isDefaultDisplay && obscuredChanged && (mWallpaperTarget == w)
                            && w.isVisibleLw()) {
                        // This is the wallpaper target and its obscured state
                        // changed... make sure the current wallaper's visibility
                        // has been updated accordingly.
                        updateWallpaperVisibilityLocked();
                    }

                    final WindowStateAnimator winAnimator = w.mWinAnimator;

                    // If the window has moved due to its containing content frame changing, then
                    // notify the listeners and optionally animate it.
                    if (w.hasMoved()) {
                        // Frame has moved, containing content frame has also moved, and we're not
                        // currently animating... let's do something.
                        final int left = w.mFrame.left;
                        final int top = w.mFrame.top;
                        if ((w.mAttrs.privateFlags & PRIVATE_FLAG_NO_MOVE_ANIMATION) == 0) {
                            Animation a = AnimationUtils.loadAnimation(mContext,
                                    com.android.internal.R.anim.window_move_from_decor);
                            winAnimator.setAnimation(a);
                            winAnimator.mAnimDw = w.mLastFrame.left - left;
                            winAnimator.mAnimDh = w.mLastFrame.top - top;
                            winAnimator.mAnimateMove = true;
                            winAnimator.mAnimatingMove = true;
                        }

                        //TODO (multidisplay): Accessibility supported only for the default display.
                        if (mAccessibilityController != null
                                && displayId == Display.DEFAULT_DISPLAY) {
                            mAccessibilityController.onSomeWindowResizedOrMovedLocked();
                        }

                        try {
                            w.mClient.moved(left, top);
                        } catch (RemoteException e) {
                        }
                    }

                    //Slog.i(TAG, "Window " + this + " clearing mContentChanged - done placing");
                    w.mContentChanged = false;

                    // Moved from updateWindowsAndWallpaperLocked().
                    if (w.mHasSurface) {
                        // Take care of the window being ready to display.
                        final boolean committed =
                                winAnimator.commitFinishDrawingLocked();
                        if (isDefaultDisplay && committed) {
                            if (w.mAttrs.type == TYPE_DREAM) {
                                // HACK: When a dream is shown, it may at that
                                // point hide the lock screen.  So we need to
                                // redo the layout to let the phone window manager
                                // make this happen.
                                displayContent.pendingLayoutChanges |=
                                        WindowManagerPolicy.FINISH_LAYOUT_REDO_LAYOUT;
                                if (DEBUG_LAYOUT_REPEATS) {
                                    debugLayoutRepeats(
                                            "dream and commitFinishDrawingLocked true",
                                            displayContent.pendingLayoutChanges);
                                }
                            }
                            if ((w.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0) {
                                if (DEBUG_WALLPAPER_LIGHT) Slog.v(TAG,
                                            "First draw done in potential wallpaper target " + w);
                                mInnerFields.mWallpaperMayChange = true;
                                displayContent.pendingLayoutChanges |=
                                        WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
                                if (DEBUG_LAYOUT_REPEATS) {
                                    debugLayoutRepeats(
                                            "wallpaper and commitFinishDrawingLocked true",
                                            displayContent.pendingLayoutChanges);
                                }
                            }
                        }

                        winAnimator.setSurfaceBoundariesLocked(recoveringMemory);
                    }

                    final AppWindowToken atoken = w.mAppToken;
                    if (DEBUG_STARTING_WINDOW && atoken != null
                            && w == atoken.startingWindow) {
                        Slog.d(TAG, "updateWindows: starting " + w + " isOnScreen="
                            + w.isOnScreen() + " allDrawn=" + atoken.allDrawn
                            + " freezingScreen=" + atoken.mAppAnimator.freezingScreen);
                    }
                    if (atoken != null
                            && (!atoken.allDrawn || atoken.mAppAnimator.freezingScreen)) {
                        if (atoken.lastTransactionSequence != mTransactionSequence) {
                            atoken.lastTransactionSequence = mTransactionSequence;
                            atoken.numInterestingWindows = atoken.numDrawnWindows = 0;
                            atoken.startingDisplayed = false;
                        }
                        if ((w.isOnScreenIgnoringKeyguard()
                                || winAnimator.mAttrType == TYPE_BASE_APPLICATION)
                                && !w.mExiting && !w.mDestroying) {
                            if (DEBUG_VISIBILITY || DEBUG_ORIENTATION) {
                                Slog.v(TAG, "Eval win " + w + ": isDrawn=" + w.isDrawnLw()
                                        + ", isAnimating=" + winAnimator.isAnimating());
                                if (!w.isDrawnLw()) {
                                    Slog.v(TAG, "Not displayed: s=" + winAnimator.mSurfaceControl
                                            + " pv=" + w.mPolicyVisibility
                                            + " mDrawState=" + winAnimator.drawStateToString()
                                            + " ah=" + w.mAttachedHidden
                                            + " th=" + atoken.hiddenRequested
                                            + " a=" + winAnimator.mAnimating);
                                }
                            }
                            if (w != atoken.startingWindow) {
                                if (!atoken.mAppAnimator.freezingScreen || !w.mAppFreezing) {
                                    atoken.numInterestingWindows++;
                                    if (w.isDrawnLw()) {
                                        atoken.numDrawnWindows++;
                                        if (DEBUG_VISIBILITY || DEBUG_ORIENTATION) Slog.v(TAG,
                                                "tokenMayBeDrawn: " + atoken
                                                + " freezingScreen=" + atoken.mAppAnimator.freezingScreen
                                                + " mAppFreezing=" + w.mAppFreezing);
                                        updateAllDrawn = true;
                                    }
                                }
                            } else if (w.isDrawnLw()) {
                                atoken.startingDisplayed = true;
                            }
                        }
                    }

                    if (isDefaultDisplay && someoneLosingFocus && (w == mCurrentFocus)
                            && w.isDisplayedLw()) {
                        focusDisplayed = true;
                    }

                    updateResizingWindows(w);
                }
				//if(numDisplays > 1 && displayNdx != numDisplays -1){
				
				//}

				//Log.i(TAG, "displayNdx = " + displayNdx);
				//Log.i(TAG, "mDisplayManagerInternal.setDisplayProperties");
				//boolean isShowDualScreen = false;
				//try{
				//	isShowDualScreen = Settings.System.getInt(mContext.getContentResolver(), Settings.DUAL_SCREEN_ICON_USED) == 1;
				//}catch (Exception e){
				//	isShowDualScreen = false;
				//}
				//if(displayId == 0 || !isShowDualScreen){
					mDisplayManagerInternal.setDisplayProperties(displayId,mInnerFields.mDisplayHasContent, mInnerFields.mPreferredRefreshRate,mInnerFields.mPreferredModeId,true /* inTraversal, must call performTraversalInTrans... below */);
					getDisplayContentLocked(displayId).stopDimmingIfNeeded();
					if (updateAllDrawn) {
						updateAllDrawnLocked(displayContent);
					}
				//}else{
					//not refresh second display
				//}
				
            }

            if (focusDisplayed) {
                mH.sendEmptyMessage(H.REPORT_LOSING_FOCUS);
            }

            // Give the display manager a chance to adjust properties
            // like display rotation if it needs to.
            mDisplayManagerInternal.performTraversalInTransactionFromWindowManager();

        } catch (RuntimeException e) {
            Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
        } finally {
            SurfaceControl.closeTransaction();
            if (SHOW_LIGHT_TRANSACTIONS) Slog.i(TAG,
                    "<<< CLOSE TRANSACTION performLayoutAndPlaceSurfaces");
        }

        final WindowList defaultWindows = defaultDisplay.getWindowList();

        // If we are ready to perform an app transition, check through
        // all of the app tokens to be shown and see if they are ready
        // to go.
        if (mAppTransition.isReady()) {
            defaultDisplay.pendingLayoutChanges |= handleAppTransitionReadyLocked(defaultWindows);
            if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("after handleAppTransitionReadyLocked",
                    defaultDisplay.pendingLayoutChanges);
        }

        if (!mAnimator.mAppWindowAnimating && mAppTransition.isRunning()) {
            // We have finished the animation of an app transition.  To do
            // this, we have delayed a lot of operations like showing and
            // hiding apps, moving apps in Z-order, etc.  The app token list
            // reflects the correct Z-order, but the window list may now
            // be out of sync with it.  So here we will just rebuild the
            // entire app window list.  Fun!
            defaultDisplay.pendingLayoutChanges |= handleAnimatingStoppedAndTransitionLocked();
            if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("after handleAnimStopAndXitionLock",
                defaultDisplay.pendingLayoutChanges);
        }

        if (mInnerFields.mWallpaperForceHidingChanged && defaultDisplay.pendingLayoutChanges == 0
                && !mAppTransition.isReady()) {
            // At this point, there was a window with a wallpaper that
            // was force hiding other windows behind it, but now it
            // is going away.  This may be simple -- just animate
            // away the wallpaper and its window -- or it may be
            // hard -- the wallpaper now needs to be shown behind
            // something that was hidden.
            defaultDisplay.pendingLayoutChanges |= WindowManagerPolicy.FINISH_LAYOUT_REDO_LAYOUT;
            if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("after animateAwayWallpaperLocked",
                defaultDisplay.pendingLayoutChanges);
        }
        mInnerFields.mWallpaperForceHidingChanged = false;

        if (mInnerFields.mWallpaperMayChange) {
            if (DEBUG_WALLPAPER_LIGHT) Slog.v(TAG, "Wallpaper may change!  Adjusting");
            defaultDisplay.pendingLayoutChanges |=
                    WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
            if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("WallpaperMayChange",
                    defaultDisplay.pendingLayoutChanges);
        }

        if (mFocusMayChange) {
            mFocusMayChange = false;
            if (updateFocusedWindowLocked(UPDATE_FOCUS_PLACING_SURFACES,
                    false /*updateInputWindows*/)) {
                updateInputWindowsNeeded = true;
                defaultDisplay.pendingLayoutChanges |= WindowManagerPolicy.FINISH_LAYOUT_REDO_ANIM;
            }
        }

        if (needsLayout()) {
            defaultDisplay.pendingLayoutChanges |= WindowManagerPolicy.FINISH_LAYOUT_REDO_LAYOUT;
            if (DEBUG_LAYOUT_REPEATS) debugLayoutRepeats("mLayoutNeeded",
                    defaultDisplay.pendingLayoutChanges);
        }

        for (i = mResizingWindows.size() - 1; i >= 0; i--) {
            WindowState win = mResizingWindows.get(i);
            if (win.mAppFreezing) {
                // Don't remove this window until rotation has completed.
                continue;
            }
            win.reportResized();
            mResizingWindows.remove(i);
        }

        if (DEBUG_ORIENTATION && mDisplayFrozen) Slog.v(TAG,
                "With display frozen, orientationChangeComplete="
                + mInnerFields.mOrientationChangeComplete);
        if (mInnerFields.mOrientationChangeComplete) {
            if (mWindowsFreezingScreen != WINDOWS_FREEZING_SCREENS_NONE) {
                mWindowsFreezingScreen = WINDOWS_FREEZING_SCREENS_NONE;
                mLastFinishedFreezeSource = mInnerFields.mLastWindowFreezeSource;
                mH.removeMessages(H.WINDOW_FREEZE_TIMEOUT);
            }
            stopFreezingDisplayLocked();
        }

        // Destroy the surface of any windows that are no longer visible.
        boolean wallpaperDestroyed = false;
        i = mDestroySurface.size();
        if (i > 0) {
            do {
                i--;
                WindowState win = mDestroySurface.get(i);
                win.mDestroying = false;
                if (mInputMethodWindow == win) {
                    mInputMethodWindow = null;
                }
                if (win == mWallpaperTarget) {
                    wallpaperDestroyed = true;
                }
                win.mWinAnimator.destroySurfaceLocked();
            } while (i > 0);
            mDestroySurface.clear();
        }

        // Time to remove any exiting tokens?
        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
            final DisplayContent displayContent = mDisplayContents.valueAt(displayNdx);
            //if(displayContent.getDisplayId() != 0 && isShowDualScreen())
			//	continue;
			ArrayList<WindowToken> exitingTokens = displayContent.mExitingTokens;
            for (i = exitingTokens.size() - 1; i >= 0; i--) {
                WindowToken token = exitingTokens.get(i);
                if (!token.hasVisible) {
                    exitingTokens.remove(i);
                    if (token.windowType == TYPE_WALLPAPER) {
                        mWallpaperTokens.remove(token);
                    }
                }
            }
        }

        // Time to remove any exiting applications?
        for (int stackNdx = mStackIdToStack.size() - 1; stackNdx >= 0; --stackNdx) {
            // Initialize state of exiting applications.
            final AppTokenList exitingAppTokens =
                    mStackIdToStack.valueAt(stackNdx).mExitingAppTokens;
            for (i = exitingAppTokens.size() - 1; i >= 0; i--) {
                AppWindowToken token = exitingAppTokens.get(i);
                if (!token.hasVisible && !mClosingApps.contains(token) &&
                        (!token.mIsExiting || token.allAppWindows.isEmpty())) {
                    // Make sure there is no animation running on this token,
                    // so any windows associated with it will be removed as
                    // soon as their animations are complete
                    token.mAppAnimator.clearAnimation();
                    token.mAppAnimator.animating = false;
                    if (DEBUG_ADD_REMOVE || DEBUG_TOKEN_MOVEMENT) Slog.v(TAG,
                            "performLayout: App token exiting now removed" + token);
                    token.removeAppFromTaskLocked();
                }
            }
        }

        if (wallpaperDestroyed) {
            defaultDisplay.pendingLayoutChanges |=
                    WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
            defaultDisplay.layoutNeeded = true;
        }

        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
            final DisplayContent displayContent = mDisplayContents.valueAt(displayNdx);
            if (displayContent.pendingLayoutChanges != 0) {
                displayContent.layoutNeeded = true;
            }
        }

        // Finally update all input windows now that the window changes have stabilized.
        mInputMonitor.updateInputWindowsLw(true /*force*/);

        setHoldScreenLocked(mInnerFields.mHoldScreen);
        if (!mDisplayFrozen) {
            if (mInnerFields.mScreenBrightness < 0 || mInnerFields.mScreenBrightness > 1.0f) {
                mPowerManagerInternal.setScreenBrightnessOverrideFromWindowManager(-1);
            } else {
                mPowerManagerInternal.setScreenBrightnessOverrideFromWindowManager(
                        toBrightnessOverride(mInnerFields.mScreenBrightness));
            }
            if (mInnerFields.mButtonBrightness < 0 || mInnerFields.mButtonBrightness > 1.0f) {
                mPowerManagerInternal.setButtonBrightnessOverrideFromWindowManager(-1);
            } else {
                mPowerManagerInternal.setButtonBrightnessOverrideFromWindowManager(
                        toBrightnessOverride(mInnerFields.mButtonBrightness));
            }
            mPowerManagerInternal.setUserActivityTimeoutOverrideFromWindowManager(
                    mInnerFields.mUserActivityTimeout);
        }

        if (mTurnOnScreen) {
            if (mAllowTheaterModeWakeFromLayout
                    || Settings.Global.getInt(mContext.getContentResolver(),
                        Settings.Global.THEATER_MODE_ON, 0) == 0) {
                if (DEBUG_VISIBILITY || DEBUG_POWER) {
                    Slog.v(TAG, "Turning screen on after layout!");
                }
                mPowerManager.wakeUp(SystemClock.uptimeMillis(), "android.server.wm:TURN_ON");
            }
            mTurnOnScreen = false;
        }

        if (mInnerFields.mUpdateRotation) {
            if (DEBUG_ORIENTATION) Slog.d(TAG, "Performing post-rotate rotation");
            if (updateRotationUncheckedLocked(false)) {
                mH.sendEmptyMessage(H.SEND_NEW_CONFIGURATION);
            } else {
                mInnerFields.mUpdateRotation = false;
            }
        }

        if (mWaitingForDrawnCallback != null ||
                (mInnerFields.mOrientationChangeComplete && !defaultDisplay.layoutNeeded &&
                        !mInnerFields.mUpdateRotation)) {
            checkDrawnWindowsLocked();
        }

        final int N = mPendingRemove.size();
        if (N > 0) {
            if (mPendingRemoveTmp.length < N) {
                mPendingRemoveTmp = new WindowState[N+10];
            }
            mPendingRemove.toArray(mPendingRemoveTmp);
            mPendingRemove.clear();
            DisplayContentList displayList = new DisplayContentList();
            for (i = 0; i < N; i++) {
                WindowState w = mPendingRemoveTmp[i];
                removeWindowInnerLocked(w);
                final DisplayContent displayContent = w.getDisplayContent();
                if (displayContent != null && !displayList.contains(displayContent)) {
                    displayList.add(displayContent);
                }
            }

            for (DisplayContent displayContent : displayList) {
                assignLayersLocked(displayContent.getWindowList());
                displayContent.layoutNeeded = true;
            }
        }

        // Remove all deferred displays stacks, tasks, and activities.
        for (int displayNdx = mDisplayContents.size() - 1; displayNdx >= 0; --displayNdx) {
            mDisplayContents.valueAt(displayNdx).checkForDeferredActions();
        }

        if (updateInputWindowsNeeded) {
            mInputMonitor.updateInputWindowsLw(false /*force*/);
        }
        setFocusedStackFrame();

        // Check to see if we are now in a state where the screen should
        // be enabled, because the window obscured flags have changed.
        enableScreenIfNeededLocked();

        scheduleAnimationLocked();

        if (DEBUG_WINDOW_TRACE) {
            Slog.e(TAG, "performLayoutAndPlaceSurfacesLockedInner exit: animating="
                    + mAnimator.mAnimating);
        }
    }
```
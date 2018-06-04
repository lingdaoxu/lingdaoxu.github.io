title: SystemServer服务
author: 道墟
date: 2018-06-04 09:53:03
tags:
---
# 一.概述

SystemServer是Android系统的核心服务，大部分服务都运行在这个进程中，它通过Zygote进程启动,在ZygoteInit类中的main方法通过startSystemServer方法（{% post_link Zygote进程 %} 2.3节）。
在SystemServer中运行的服务有60多种，为了防止应用程序对系统造成破坏，android的应用程序没有权限直接访问设备的底层资源，只能通过SystemServer中的服务代理访问。

# 二.SystemServer 的创建过程
如上节所说的，SystemServer服务是在Zygote进程中fork并初始化的，然后再调用SystemServer的main方法来启动系统的服务。所以下面我们分成两个部分分别介绍走一遍流程。

# 2.1 创建SystemServer进程
首先我们来看 startSystemServer 方法。


```java
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
		
		//1.准备启动参数
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",    //进程ID
            "--setgid=1000",   //组ID
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",   //设定了SystemServer的执行类
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
			
			//2.调用forkSystemServer fork出SystemServer进程
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
			
			//3.初始化SystemServer进程
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

# 2.1.1 fork SystemServer进程
第一步是准备参数在代码中已经把重要的参数都注释出来了，我们重点关注第二步 fork 出 SystemServer 进程，首先我们跟踪进 Zygote 类的 forkSystemServer 方法。

```java
    public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        // Enable tracing as soon as we enter the system_server.
        if (pid == 0) {
            Trace.setTracingEnabled(true);
        }
        VM_HOOKS.postForkCommon();
        return pid;
    }
```
可以看到其实 forkSystemServer 方法相当的简单 只是调用了 native 方法 nativeForkSystemServer,跟踪到 jni 层的方法如下：
路径：frameworks/base/core/jni/com_android_internal_os_Zygote.cpp

```cpp
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
		
  // 调用 ForkAndSpecializeCommon 方法来 frok 出子进程
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      // The zygote process checks whether the child process has died or not.
      ALOGI("System server process %d has been created", pid);
      gSystemServerPid = pid;
      // There is a slight window that the system server process has crashed
      // but it went unnoticed because we haven't published its pid yet. So
      // we recheck here just to make sure that all is well.
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          ALOGE("System server process %d has died. Restarting Zygote!", pid);
          RuntimeAbort(env);   //启动SystemServer失败，重启系统
      }
  }
  return pid;
}
```

从上面的函数中我们看到，主要还是通过 ForkAndSpecializeCommon 方法来fork出子进程，这跟前面我们介绍 应用 通过Zygote类的forkAndSpecialize 方法（该方法最终也是调动了ForkAndSpecializeCommon来fork子进程）fork出子进程是一样的。只不过这边比较特殊的是，在fork后多了一步判断，如果fork失败则 Zygote进程会直接终止自己（注意这时候我们是在Zygote进程来准备fork出SystemServer进程）,Zygote进程终止后会自动重启，然后再走一遍这个流程。

在上一篇的Zygote进程 fork 应用程序的时候并没有深入的介绍 Zygote.forkAndSpecialize 的流程，这边我们结合 SystmServer进程的 fork 一起介绍，如上所说，这两个进程的fork最终都会调用到 ForkAndSpecializeCommon 方法，我们来看下这个方法的详细代码

```cpp
// Utility routine to fork zygote and specialize the child process.
static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {
									 
 //设置处理sigchld 信号的函数 SigChldHandler
  SetSigChldHandler();

#ifdef ENABLE_SCHED_BOOST
  SetForkLoad(true);
#endif

  sigset_t sigchld;
  sigemptyset(&sigchld);
  sigaddset(&sigchld, SIGCHLD);

  // Temporarily block SIGCHLD during forks. The SIGCHLD handler might
  // log, which would result in the logging FDs we close being reopened.
  // This would cause failures because the FDs are not whitelisted.
  //
  // Note that the zygote process is single threaded at this point.
  if (sigprocmask(SIG_BLOCK, &sigchld, NULL) == -1) {
    ALOGE("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno));
    RuntimeAbort(env, __LINE__, "Call to sigprocmask(SIG_BLOCK, { SIGCHLD }) failed.");
  }

  // Close any logging related FDs before we start evaluating the list of
  // file descriptors.
  __android_log_close();

  // If this is the first fork for this zygote, create the open FD table.
  // If it isn't, we just need to check whether the list of open files has
  // changed (and it shouldn't in the normal case).
  if (gOpenFdTable == NULL) {
    gOpenFdTable = FileDescriptorTable::Create();
    if (gOpenFdTable == NULL) {
      RuntimeAbort(env, __LINE__, "Unable to construct file descriptor table.");
    }
  } else if (!gOpenFdTable->Restat()) {
    RuntimeAbort(env, __LINE__, "Unable to restat file descriptor table.");
  }

  pid_t pid = fork();

  if (pid == 0) {
    // The child process.
    gMallocLeakZygoteChild = 1;

    // Clean up any descriptors which must be closed immediately
    DetachDescriptors(env, fdsToClose);

    // Re-open all remaining open file descriptors so that they aren't shared
    // with the zygote across a fork.
    if (!gOpenFdTable->ReopenOrDetach()) {
      RuntimeAbort(env, __LINE__, "Unable to reopen whitelisted descriptors.");
    }

    if (sigprocmask(SIG_UNBLOCK, &sigchld, NULL) == -1) {
      ALOGE("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno));
      RuntimeAbort(env, __LINE__, "Call to sigprocmask(SIG_UNBLOCK, { SIGCHLD }) failed.");
    }

    // Keep capabilities across UID change, unless we're staying root.
    if (uid != 0) {
      EnableKeepCapabilities(env);
    }

    DropCapabilitiesBoundingSet(env);

    bool use_native_bridge = !is_system_server && (instructionSet != NULL)
        && android::NativeBridgeAvailable();
    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      use_native_bridge = android::NeedsNativeBridge(isa_string.c_str());
    }
    if (use_native_bridge && dataDir == NULL) {
      // dataDir should never be null if we need to use a native bridge.
      // In general, dataDir will never be null for normal applications. It can only happen in
      // special cases (for isolated processes which are not associated with any app). These are
      // launched by the framework and should not be emulated anyway.
      use_native_bridge = false;
      ALOGW("Native bridge will not be used because dataDir == NULL.");
    }

    if (!MountEmulatedStorage(uid, mount_external, use_native_bridge)) {
      ALOGW("Failed to mount emulated storage: %s", strerror(errno));
      if (errno == ENOTCONN || errno == EROFS) {
        // When device is actively encrypting, we get ENOTCONN here
        // since FUSE was mounted before the framework restarted.
        // When encrypted device is booting, we get EROFS since
        // FUSE hasn't been created yet by init.
        // In either case, continue without external storage.
      } else {
        ALOGE("Cannot continue without emulated storage");
        RuntimeAbort(env);
      }
    }

    if (!is_system_server) {
        int rc = createProcessGroup(uid, getpid());
        if (rc != 0) {
            if (rc == -EROFS) {
                ALOGW("createProcessGroup failed, kernel missing CONFIG_CGROUP_CPUACCT?");
            } else {
                ALOGE("createProcessGroup(%d, %d) failed: %s", uid, pid, strerror(-rc));
            }
        }
    }

    SetGids(env, javaGids);

    SetRLimits(env, javaRlimits);

    if (use_native_bridge) {
      ScopedUtfChars isa_string(env, instructionSet);
      ScopedUtfChars data_dir(env, dataDir);
      android::PreInitializeNativeBridge(data_dir.c_str(), isa_string.c_str());
    }

    int rc = setresgid(gid, gid, gid);
    if (rc == -1) {
      ALOGE("setresgid(%d) failed: %s", gid, strerror(errno));
      RuntimeAbort(env);
    }

    rc = setresuid(uid, uid, uid);
    if (rc == -1) {
      ALOGE("setresuid(%d) failed: %s", uid, strerror(errno));
      RuntimeAbort(env);
    }

    if (NeedsNoRandomizeWorkaround()) {
        // Work around ARM kernel ASLR lossage (http://b/5817320).
        int old_personality = personality(0xffffffff);
        int new_personality = personality(old_personality | ADDR_NO_RANDOMIZE);
        if (new_personality == -1) {
            ALOGW("personality(%d) failed: %s", new_personality, strerror(errno));
        }
    }

    SetCapabilities(env, permittedCapabilities, effectiveCapabilities);

    SetSchedulerPolicy(env);

    const char* se_info_c_str = NULL;
    ScopedUtfChars* se_info = NULL;
    if (java_se_info != NULL) {
        se_info = new ScopedUtfChars(env, java_se_info);
        se_info_c_str = se_info->c_str();
        if (se_info_c_str == NULL) {
          ALOGE("se_info_c_str == NULL");
          RuntimeAbort(env);
        }
    }
    const char* se_name_c_str = NULL;
    ScopedUtfChars* se_name = NULL;
    if (java_se_name != NULL) {
        se_name = new ScopedUtfChars(env, java_se_name);
        se_name_c_str = se_name->c_str();
        if (se_name_c_str == NULL) {
          ALOGE("se_name_c_str == NULL");
          RuntimeAbort(env);
        }
    }
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);
    if (rc == -1) {
      ALOGE("selinux_android_setcontext(%d, %d, \"%s\", \"%s\") failed", uid,
            is_system_server, se_info_c_str, se_name_c_str);
      RuntimeAbort(env);
    }

    // Make it easier to debug audit logs by setting the main thread's name to the
    // nice name rather than "app_process".
    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_info_c_str != NULL) {
      SetThreadName(se_name_c_str);
    }

    delete se_info;
    delete se_name;

    UnsetSigChldHandler();

    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server ? NULL : instructionSet);
    if (env->ExceptionCheck()) {
      ALOGE("Error calling post fork hooks.");
      RuntimeAbort(env);
    }
  } else if (pid > 0) {
    // the parent process

    // We blocked SIGCHLD prior to a fork, we unblock it here.
    if (sigprocmask(SIG_UNBLOCK, &sigchld, NULL) == -1) {
      ALOGE("sigprocmask(SIG_SETMASK, { SIGCHLD }) failed: %s", strerror(errno));
      RuntimeAbort(env, __LINE__, "Call to sigprocmask(SIG_UNBLOCK, { SIGCHLD }) failed.");
    }
  }
  return pid;
}
}  // anonymous namespace

```

这边我们重点关注 SetSigChldHandler 函数的调用，这边设置了处理 sigchld 信号的函数 SigChldHandler：

```cpp
// This signal handler is for zygote mode, since the zygote must reap its children
static void SigChldHandler(int /*signal_number*/) {
  pid_t pid;
  int status;

  // It's necessary to save and restore the errno during this function.
  // Since errno is stored per thread, changing it here modifies the errno
  // on the thread on which this signal handler executes. If a signal occurs
  // between a call and an errno check, it's possible to get the errno set
  // here.
  // See b/23572286 for extra information.
  int saved_errno = errno;

//调用 waitpid 来防止子进程变成僵尸
  while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
     // Log process-death status that we care about.  In general it is
     // not safe to call LOG(...) from a signal handler because of
     // possible reentrancy.  However, we know a priori that the
     // current implementation of LOG() is safe to call from a SIGCHLD
     // handler in the zygote process.  If the LOG() implementation
     // changes its locking strategy or its use of syscalls within the
     // lazy-init critical section, its use here may become unsafe.
    if (WIFEXITED(status)) {
      if (WEXITSTATUS(status)) {
        ALOGI("Process %d exited cleanly (%d)", pid, WEXITSTATUS(status));
      }
    } else if (WIFSIGNALED(status)) {
      if (WTERMSIG(status) != SIGKILL) {
        ALOGI("Process %d exited due to signal (%d)", pid, WTERMSIG(status));
      }
      if (WCOREDUMP(status)) {
        ALOGI("Process %d dumped core.", pid);
      }
    }
	
	//如果死亡的是 SystemServer 进程，那么杀死父进程（也就是Zygote）
    // If the just-crashed process is the system_server, bring down zygote
    // so that it is restarted by init and system server will be restarted
    // from there.
    if (pid == gSystemServerPid) {
      ALOGE("Exit zygote because system server (%d) has terminated", pid);
      kill(getpid(), SIGKILL);
    }
  }

  // Note that we shouldn't consider ECHILD an error because
  // the secondary zygote might have no children left to wait for.
  if (pid < 0 && errno != ECHILD) {
    ALOGW("Zygote SIGCHLD error in waitpid: %s", strerror(errno));
  }

  errno = saved_errno;
}
```

这边我们重点关注 上面代码注释的那段，如果死亡的是 SystemServer 进程，那么杀死父进程（也就是Zygote），如果Zygote死亡那么导致的是INit进程会杀死所有的用户进程并重启Zygote。
我们在调试系统代码的时候经常会杀死system_process进程来重启系统的原理就是这个。为什么是system_process进程而不是参数system_server ，这个我们在后面 2.2.1 会介绍到。

# 2.1.2 调用 handleSystemServerProcess 初始化 SystemServer 进程


```java
    /**
     * Finish remaining work for the newly forked system server process.
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {

        closeServerSocket();
		
		//设置 umask = 0077
        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);

        if (parsedArgs.niceName != null) {
			//修改进程名 上面我们传进来的参数为 system_server
            Process.setArgV0(parsedArgs.niceName);
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            performSystemServerDexOpt(systemServerClasspath);
        }
		
		// invokeWith 参数基本为null
        if (parsedArgs.invokeWith != null) {
            String[] args = parsedArgs.remainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2, parsedArgs.remainingArgs.length);
            }

            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
                Thread.currentThread().setContextClassLoader(cl);
            }
			
			
			//通过 RuntimeInit 类 zygoteInit 来调用 SystemServer 的 main 方法
            /*
             * Pass the remaining arguments to SystemServer.
             */
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }

        /* should never reach here */
    }
```
上面方法主要做了以下几件事
- 关闭从zygote继承的socket
- 设置SystemServer进程的umask为0077，这样SystemServer进程创建的文件的属性为0700，只有进程本身可以访问
- 修改进程的名称为参数 “system_server”
- invokeWith 参数基本为null,所以一般是通过 RuntimeInit 类 zygoteInit 来调用 SystemServer 的 main 方法


# 2.2 SystemServer 类main方法来执行初始化

首先我们来查看 main 方法的源码

```java
    public static void main(String[] args) {
        new SystemServer().run();
    }
```
相当的简单,直接 new 一个 SystemServer 类的 实例同事执行了 run 方法，这里我们要重点解释下 SystemServer 并未继承自 Thread 类，这边调用 run 方法，只是刚好名字叫做 run 。

```java
    private void run() {
	
		//如果时间不正确则调整系统时间
        // If a device's clock is before 1970 (before 0), a lot of
        // APIs crash dealing with negative numbers, notably
        // java.io.File#setLastModified, so instead we fake it and
        // hope that time from cell towers or NTP fixes it shortly.
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }

        // If the system has "persist.sys.language" and friends set, replace them with
        // "persist.sys.locale". Note that the default locale at this point is calculated
        // using the "-Duser.locale" command line flag. That flag is usually populated by
        // AndroidRuntime using the same set of system properties, but only the system_server
        // and system apps are allowed to set them.
        //
        // NOTE: Most changes made here will need an equivalent change to
        // core/jni/AndroidRuntime.cpp
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();

            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }

        // Here we go!
        Slog.i(TAG, "Entered the Android system server!");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

		//设置当前的虚拟机 运行库 路径
        // In case the runtime switched since last boot (such as when
        // the old runtime was removed in an OTA), set the system
        // property so that it is in sync. We can't do this in
        // libnativehelper's JniInvocation::Init code where we already
        // had to fallback to a different runtime because it is
        // running as root and we need to be the system user to set
        // the property. http://b/11463182
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

        // Enable the sampling profiler.
        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            mProfilerSnapshotTimer = new Timer();
            mProfilerSnapshotTimer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
                }
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }

        // Mmmmmm... more memory!
        VMRuntime.getRuntime().clearGrowthLimit();
		
		//调整虚拟机堆的内存，设置堆利用率为0.8，当实际利用率偏离这个值的时候，虚拟机在立即回收的时候会重新调整堆大小，使得利用率接近这个设置值
        // The system server has to run all of the time, so it needs to be
        // as efficient as possible with its memory usage.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

        // Some devices rely on runtime fingerprint generation, so make sure
        // we've defined it before booting further.
        Build.ensureFingerprintProperty();

        // Within the system server, it is an error to access Environment paths without
        // explicitly specifying a user.
        Environment.setUserRequired(true);

        // Ensure binder calls into the system always run at foreground priority.
        BinderInternal.disableBackgroundScheduling(true);

        // Prepare the main looper thread (this thread).
        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();
		
		//装载 libandroid_servers.so
        // Initialize native services.
        System.loadLibrary("android_servers");

        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();

        // Initialize the system context.
        createSystemContext();
		
		//创建 SystemServiceManager 的实例 mSystemServiceManager ，它将负责系统Service的启动
        // Create the system service manager.
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
		
		//启动全部的 java services
        // Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        // For debug builds, log event loop stalls to dropbox for analysis.
        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```
run 方法主要做这些事情

- 如果时间不正确则调整系统时间
- 设置当前的虚拟机 运行库 路径:值为 属性 persist.sys.dalvik.vm.lib.2 的值
- 调整虚拟机堆的内存，设置堆利用率为0.8，当实际利用率偏离这个值的时候，虚拟机在立即回收的时候会重新调整堆大小，使得利用率接近这个设置值
- 装载 libandroid_servers.so,路径在 framework/base/services/jni 下
- 调用 createSystemContext 方法获取 Context
- 创建 SystemServiceManager 的实例 mSystemServiceManager ，它将负责系统Service的启动
- 启动全部的 java services
- 进入消息循环：Looper.loop()

# 2.2.1 createSystemContext 方法获取 Context

```java
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
		//设置主题
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }
```

```java
    public static ActivityThread systemMain() {
        // The system process on low-memory devices do not get to use hardware
        // accelerated drawing, since this can add too much overhead to the
        // process.
        if (!ActivityManager.isHighEndGfx()) {
            HardwareRenderer.disable(true);
        } else {
            HardwareRenderer.enableForegroundTrimming();
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(true);
        return thread;
    }
```
在 {% post_link Zygote进程 %} 篇的 3.5.3节中，当 zygote 进程fork应用进程，在最后启动应用的时候会调用 ActivityThread 类的 main 方法，跟这边的 systemMain方法相似的是这两个方法都 创建了 ActivityThread 对象（区别在于 普通应用 thread.attach(false); SystemServer则是 true）。ActivityThread 是应用程序的主线程类，那么为什么 SystemServer 需要创建这个对象呢？这是因为 SystemServer 不仅仅是一个后台进程，同时也是一个运行着组件的Service的进程，像一些系统对话框也是从这个进程弹出的，所以它也需要一个上下文环境（activityThread.getSystemContext()）。

上文我们说到SystemServer 调用 hread.attach 方法的参数为 true ，那么我们来看看 true分支下的流程

```java
    private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                        }
                    }
                }
            });
        } else {			//这边为 true 流程 也就是 SystemServer 进程执行的 
		
			//设置ddms中看到的应用名称 这也是为什么 我们上面说要重启系统 杀死system_process进程的原因
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
				//创建 Instrumentation 对象
                mInstrumentation = new Instrumentation();
				//创建 Context 对象
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
				//创建 应用的Application 对象
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
				//调用 onCreate 生命周期
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        // add dropbox logging to libcore
        DropBox.setReporter(new DropBoxReporter());

        ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                synchronized (mResourcesManager) {
                    // We need to apply this change to the resources
                    // immediately, because upon returning the view
                    // hierarchy will be informed about it.
                    if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                        // This actually changed the resources!  Tell
                        // everyone about it.
                        if (mPendingConfiguration == null ||
                                mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                            mPendingConfiguration = newConfig;

                            sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                        }
                    }
                }
            }
            @Override
            public void onLowMemory() {
            }
            @Override
            public void onTrimMemory(int level) {
            }
        });
    }
```
从上面代码可以看出这是完全在模拟一个类似应用的环境。但是每一个上下文环境都需要一个 apk文件，那么这个文件从哪里来呢？这个就需要我们跟踪进 getSystemContext 方法了。

```java
    public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }
```

```java
    static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ContextImpl context = new ContextImpl(null, mainThread,
                packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                context.mResourcesManager.getDisplayMetricsLocked());
        return context;
    }
```

```java
    LoadedApk(ActivityThread activityThread) {
        mActivityThread = activityThread;
        mApplicationInfo = new ApplicationInfo();
        mApplicationInfo.packageName = "android";
        mPackageName = "android";
        mAppDir = null;
        mResDir = null;
        mSplitAppDirs = null;
        mSplitResDirs = null;
        mOverlayDirs = null;
        mSharedLibraries = null;
        mDataDir = null;
        mDataDirFile = null;
        mLibDir = null;
        mBaseClassLoader = null;
        mSecurityViolation = false;
        mIncludeCode = true;
        mRegisterPackage = false;
        mClassLoader = ClassLoader.getSystemClassLoader();
        mResources = Resources.getSystem();
    }
```
因为 framework-res.apk的包名就是 “android”，所以这边创建的Application对象代表的就是 framework-res.apk。









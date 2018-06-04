title: Zygote进程
author: 道墟
tags: []
categories: []
date: 2018-06-01 16:21:00
---
# 一.概述

# 1.1 什么是Zygote

Linux的进程是通过系统调用fork产生的，fork出的子进程除了内核中的一些核心数据结构和父进程不同外，其余的内存映像都是和父进程共享的。当子进程需要去改写这些共享数据的内存时，操作系统才会为子进程分配一个新的内存页，并将老的页面数据复制到新的页面上后再进行修改操作（写时拷贝）。

fork出来的进程会继续执行系统调用exec。exec将用一个新的可执行文件的内容替换当前进程的代码段、数据段、堆和栈段。

fork+exec是Linux启动进程的标准做法，Init进程中启动的服务也是这样做的，Zygote进程就是这么被启动起来的。（{% post_link Init进程 %} 2.2.3处）

那么Zygote是怎么做的呢？Zygote在创建应用程序的只执行了fork操作，没有去调用exec。

Zygote初始化时会创建虚拟机，同时把需要的系统类库和资源文件加载到内存中。Zygote fork出子进程后（系统会根据apk的类型来选择fork 32位还是64位的 Zygote），这个子进程也继承了能正常工作的虚拟机和各种系统资源。接下来子进程只需要加载APK中的字节码就可以运行了。这样就达到了减少启动时间的作用。

# 1.2 启动的时机

上篇{% post_link Init进程 %}中我们知道Zygote进程是由Init进程启动的，那么到底是在什么时候启动的呢？我们现在追踪下源码。
在Init进程的初始化过程中会解析init.rc文件，从文件中我们可以看出init.rc会根据ro.zygote属性来加载哪个文件，下面是我获取的公司的6.0系统的板子属性。

>root@rk3399_stbvr:/ # getprop ro.zygote
zygote64_32

```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc
import /init.trace.rc

```

现在我们来看看init.zygote64_32.rc文件的内容：
```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class core
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class core
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks
```

# 1.3 参数分析
app_process 启动的参数格式是:

app_process [虚拟机参数] 运行目录 参数[java类]


以上文rc文件32位启动为例：
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary

- 虚拟机参数：以符号‘-’开头。例如：-Xzygote
- 运行目录：一般是在 /system/bin
- 参数：以符号“--”开头。参数“--zygote”表示要启动zygote进程。“--application”表示要以普通进程的方式执行java代码。
- java类：将要执行的java类。必须要有一个静态方法main。使用参数“zygote”时不会执行这个类，而是固定执行ZygoteInit类。


# 二.Zygote的初始化
# 2.1 app_process的main方法
```cpp
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
        // EINVAL. Don't die on such kernels.
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.

    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;
	
	//解析参数
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
		//执行非zygote模式
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
		//执行zygote模式
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
	
	//将本进程的名字改成 “--nice-name”参数的指定的值，如果没指定则默认为ZYGOTE_NICE_NAME宏指定的值（可能是zygote或者是zygote64）
    if (!niceName.isEmpty()) {
		//替换启动参数窜中的“app_process”为参数--nice-name指定的名称
        runtime.setArgv0(niceName.string());
		//用来改变进程名
        set_process_name(niceName.string());
    }

    if (zygote) {
		//参数带有--zygote 所以执行ZygoteInit类
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
		//否则执行通过参数传进来的java类
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

# 2.2 启动虚拟机 AndroidRuntime类
负责启动虚拟机以及Java线程.
AndroidRuntime类在一个进程中只有一个实例对象，保存在全局变量gCurRuntime中。

```cpp
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
    SkGraphics::Init();
    // There is also a global font cache, but its budget is specified by
    // SK_DEFAULT_FONT_CACHE_COUNT_LIMIT and SK_DEFAULT_FONT_CACHE_LIMIT.

    // Pre-allocate enough space to hold a fair number of options.
    mOptions.setCapacity(20);

    assert(gCurRuntime == NULL);        // one per process
	
	//保存到全局变量中
    gCurRuntime = this;
}
```

# 2.2.1 启动虚拟机
在2.1的main函数的末尾我们执行了runtime的start函数来执行java类。
在zygote进程执行java代码前，还需要初始化整个java运行环境。
下面我们来追踪下start函数的执行过程

# 2.2.1.1 输出启动log

```cpp
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());
```
这个日志标志android系统的启动，因为以后的进程都是从zygote进程fork出来的，所以这个不会再执行start函数。
所以如果日志中反复出现这段打印的话，并且进程ID为zygote，则说明系统正在不断的重启。

# 2.2.1.2 获取系统目录

```cpp
    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }
```
首先从环境变量ANDROID_ROOT中获取，如果环境变量没有设置则把系统目录设置为/system，然后检查/system目录是否存在，如果不存在zygote退出。系统目录是在Init进程中创建的。

# 2.2.1.3 启动虚拟机

```cpp
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
```

# 2.2.1.4 调用虚函数onVmCreated
```cpp
    onVmCreated(env);
```
这边onVmCreated只是一个虚函数，在android系统中调用的其实是其子类 AppRuntime中重载的。代码如下

```cpp
    virtual void onVmCreated(JNIEnv* env)
    {
		//如果是zygote进程的话，这时候mClassName是null的，所以执行到这边就结束了。
        if (mClassName.isEmpty()) {
            return; // Zygote. Nothing to do here.
        }

        /*
         * This is a little awkward because the JNI FindClass call uses the
         * class loader associated with the native method we're executing in.
         * If called in onStarted (from RuntimeInit.finishInit because we're
         * launching "am", for example), FindClass would see that we're calling
         * from a boot class' native method, and so wouldn't look for the class
         * we're trying to look up in CLASSPATH. Unfortunately it needs to,
         * because the "am" classes are not boot classes.
         *
         * The easiest fix is to call FindClass here, early on before we start
         * executing boot class Java code and thereby deny ourselves access to
         * non-boot classes.
         */
        char* slashClassName = toSlashClassName(mClassName.string());
        mClass = env->FindClass(slashClassName);
        if (mClass == NULL) {
            ALOGE("ERROR: could not find class '%s'\n", mClassName.string());
        }
        free(slashClassName);

        mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass));
    }
```

# 2.2.1.5 注册系统的JNI函数

```cpp
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
```
```cpp
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);
	
	//通过register_jni_procs函数将全局数组gRegJNI中的jni本地函数在虚拟机中注册
    
	if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}
```

# 2.2.1.6 准备参数: 调用java类的main函数使用的
```cpp
    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
```
# 2.2.1.7 调用java类的main方法

如果启动的是Zygote进程调用的java类是ZygoteInit类，否则的话是RuntimeInit。

```cpp
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
		//获取main方法的ID
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
			//通过id调用java类的main方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
```

# 2.3 初始化工作—— ZygoteInit 类

从这边起我们就从C++进入到了java

```java
    public static void main(String argv[]) {
        try {
            RuntimeInit.enableDdms();
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }
			// 注册Zyogte的socket监听端口，用来接收启动应用程序的消息  3.1会详细分析
            registerZygoteSocket(socketName);
			
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
				
			//装在系统资源。包括系统预加载类、Framework资源和openGL的资源。
			//这样当应用程序被fork处理后，应用的进程内已经很包含了这些系统资源，大大节省应用的启动时间。
			//在 第四节 预加载系统类和资源中会详细介绍
            preload();
			
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());

            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            gcAndFinalize();

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            if (startSystemServer) {
						//启动 SystemServer进程 在中会详细介绍
                startSystemServer(abiList, socketName);
            }

            Log.i(TAG, "Accepting command socket connections");
			//进入监听和接收消息的循环 在3.3节会详细分析
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```
{% post_link SystemServer服务 %} 的启动也是在这边的startSystemServer()方法中调用的。


# 三.Zygote启动应用程序

# 3.1 注册zygote的socket
在上面2.3 ZygoteInit的main方法中我们调用了 registerZygoteSocket 方法来创建一个本地socket，然后调用了runSelectLoop来进入等待监听socket连接的循环。

下面我们先来查看 registerZygoteSocket 方法

```java
    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
			//ANDROID_SOCKET_PREFIX的值为“ANDROID_SOCKET_”
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
				//通过环境变量获取socket句柄
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
				//生成本地服务的socket并保存在全局变量中。
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
registerZygoteSocket通过环境变量获取到socket句柄，但是这个socket是在哪里创建的呢？
回到 1.2 节，我们是否还记得 zygote的rc文件的zygote服务里面定义了这么一条命令

```
   socket zygote stream 660 root system
```
Init进程会根据这个选项创建一个AF_UNIX的socket，并把这个句柄放在环境变量ANDROID_SOCKET_[zygote]（zygote这个是指选项的名称）中。
根据句柄创建FileDescriptor对象，然后通过LocalServerSocket方法生成一个本地服务的socket,并保存在sServerSocket中。

# 3.2 请求启动应用
android启动一个新的进程都是在AMS中完成的，不管是因为什么原因启动一个新的进程，最终都是AMS的startProcessLocked来实现的。代码如下：

```java

            if (entryPoint == null) entryPoint = "android.app.ActivityThread";

            //启动进程
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
```
"android.app.ActivityThread" 参数就是应用启动后执行的java类。

```java
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
```

最终start方法调用了 startViaZygote 方法来启动应用。
首先将应用程序的启动参数保存到argsForZygote列表中，然后调用 zygoteSendArgsAndGetResult 将应用程序进程的启动参数发送到zygote进程中。

```java
    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
		
		
			//启动参数保存到argsForZygote列表中
            ArrayList<String> argsForZygote = new ArrayList<String>();

            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
                argsForZygote.add("--enable-jni-logging");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
                argsForZygote.add("--enable-safemode");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
                argsForZygote.add("--enable-debugger");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
                argsForZygote.add("--enable-checkjni");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_JIT) != 0) {
                argsForZygote.add("--enable-jit");
            }
            if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
                argsForZygote.add("--generate-debug-info");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
                argsForZygote.add("--enable-assert");
            }
            if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
                argsForZygote.add("--mount-external-default");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
                argsForZygote.add("--mount-external-read");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
                argsForZygote.add("--mount-external-write");
            }
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

            //TODO optionally enable debuger
            //argsForZygote.add("--enable-debugger");

            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");

                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }

                argsForZygote.add(sb.toString());
            }

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }

            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }

            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }

            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }

            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }
			
			// openZygoteSocketIfNeeded创建通信的socket，然后zygoteSendArgsAndGetResult将参数列表发送到zygote进程中
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```

创建通信的socket
```java
    private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }
```
发送到zygote进程

```java
private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            // Throw early if any of the arguments are malformed. This means we can
            // avoid writing a partial response to the zygote.
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                if (args.get(i).indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx("embedded newlines not allowed");
                }
            }

            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();

            // Always read the entire result from the input stream to avoid leaving
            // bytes in the stream for future process starts to accidentally stumble
            // upon.
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```

# 3.3 处理启动应用的请求
在2.3节中 我们知道在ZygoteInit的main方法中 会通过runSelectLoop开启无限循环来接收启动应用的请求

```java
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```
这边通过i=0判断请求连接事件是否到来，如果到来了调用 acceptCommandPeer 来和客户端建立一个socket链接，然后把socket加入到监听数组中，来等待socket上命令的到来。

如果i>0这时候表示socket上有数据到了。这时候直接调用 ZygoteConnection 类的 runOnce 方法，处理完断开连接，并清除scoket。

# 3.4 fork应用进程



```java
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
			//通过 readArgumentList 方法从socket连接中读取参数行
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            Log.w(TAG, "IOException on command socket " + ex.getMessage());
            closeSocket();
            return true;
        }

        if (args == null) {
            // EOF reached.
            closeSocket();
            return true;
        }

        /** the stderr of the most recent request, if avail */
        PrintStream newStderr = null;

        if (descriptors != null && descriptors.length >= 3) {
            newStderr = new PrintStream(
                    new FileOutputStream(descriptors[2]));
        }

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        try {
            parsedArgs = new Arguments(args);

            if (parsedArgs.abiListQuery) {
                return handleAbiListQuery();
            }

            if (parsedArgs.permittedCapabilities != 0 || parsedArgs.effectiveCapabilities != 0) {
                throw new ZygoteSecurityException("Client may not specify capabilities: " +
                        "permitted=0x" + Long.toHexString(parsedArgs.permittedCapabilities) +
                        ", effective=0x" + Long.toHexString(parsedArgs.effectiveCapabilities));
            }
			//检查客户端进程是否有权利指定进程的用户id 组id 和所属的组
			//如果 客户端是root进程则可以任意指定
			//如果客户端是system进程，只有在系统属性 ro.fatorytest值为1 或者2的时候可以指定
			//没有指定则继承客户端的值
            applyUidSecurityPolicy(parsedArgs, peer);
            applyInvokeWithSecurityPolicy(parsedArgs, peer);

            applyDebuggerSystemProperty(parsedArgs);
            applyInvokeWithSystemProperty(parsedArgs);

            int[][] rlimits = null;

            if (parsedArgs.rlimits != null) {
                rlimits = parsedArgs.rlimits.toArray(intArray2d);
            }

            if (parsedArgs.invokeWith != null) {
                FileDescriptor[] pipeFds = Os.pipe2(O_CLOEXEC);
                childPipeFd = pipeFds[1];
                serverPipeFd = pipeFds[0];
                Os.fcntlInt(childPipeFd, F_SETFD, 0);
            }

            /**
             * In order to avoid leaking descriptors to the Zygote child,
             * the native code must close the two Zygote socket descriptors
             * in the child process before it switches from Zygote-root to
             * the UID and privileges of the application being launched.
             *
             * In order to avoid "bad file descriptor" errors when the
             * two LocalSocket objects are closed, the Posix file
             * descriptors are released via a dup2() call which closes
             * the socket and substitutes an open descriptor to /dev/null.
             */

            int [] fdsToClose = { -1, -1 };

            FileDescriptor fd = mSocket.getFileDescriptor();

            if (fd != null) {
                fdsToClose[0] = fd.getInt$();
            }

            fd = ZygoteInit.getServerSocketFileDescriptor();

            if (fd != null) {
                fdsToClose[1] = fd.getInt$();
            }

            fd = null;
			
			//fork子进程 最后的fork还在native层完成的  在SystemServer进程 2.1 节会详细介绍
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        } catch (ErrnoException ex) {
            logAndPrintError(newStderr, "Exception creating pipe", ex);
        } catch (IllegalArgumentException ex) {
            logAndPrintError(newStderr, "Invalid zygote arguments", ex);
        } catch (ZygoteSecurityException ex) {
            logAndPrintError(newStderr,
                    "Zygote security policy prevents request: ", ex);
        }

        try {
            if (pid == 0) {
                // in child
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
				//pid=0 表示在子进程中，所以执行 handleChildProc 在 3.5 节详细展开
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                // should never get here, the child is expected to either
                // throw ZygoteInit.MethodAndArgsCaller or exec().
                return true;
            } else {
                // in parent...pid of < 0 means failure
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
				//这时候还在zygote进程中，所以调用 handleParentProc 继续处理
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }
```




# 3.5 子进程的初始化
我们接着上面的流程继续分析 handleChildProc 方法

```java
    private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
        /**
         * By the time we get here, the native code has closed the two actual Zygote
         * socket connections, and substituted /dev/null in their place.  The LocalSocket
         * objects still need to be closed properly.
         */

        closeSocket();
        ZygoteInit.closeServerSocket();

        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);
                Os.dup2(descriptors[1], STDOUT_FILENO);
                Os.dup2(descriptors[2], STDERR_FILENO);

                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);
                }
                newStderr = System.err;
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        // End of the postFork event.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }
```
在方法的最后会根据参数的不同选择初始化的方式
invokeWith不为null的时候，将会通过调用exec的方式来启动app_process进程来执行java类。
正常情况下还是调用RuntimeInit.zygoteInit来启动的

```java
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();

        commonInit();
        nativeZygoteInit();
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```

# 3.5.1 commonInit方法


```java
    private static final void commonInit() {
        if (DEBUG) Slog.d(TAG, "Entered RuntimeInit!");
		
		//设置进程的UncaughtException的处理方法
		//这边默认的实现是打印出错误堆栈然后退出应用
		//应用可以调用 Thread.setDefaultUncaughtExceptionHandler 来设置自定义的处理方式
		//想腾讯的bug平台的实现就是基于这点
        /* set default handler; this applies to all threads in the VM */
        Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

		//设置时区
        /*
         * Install a TimezoneGetter subclass for ZoneInfo.db
         */
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });
        TimeZone.setDefault(null);
		
		//重置android系统的log系统
        /*
         * Sets handler for java.util.logging to use Android log facilities.
         * The odd "new instance-and-then-throw-away" is a mirror of how
         * the "java.util.logging.config.class" system property works. We
         * can't use the system property here since the logger has almost
         * certainly already been initialized.
         */
        LogManager.getLogManager().reset();
        new AndroidConfig();
		
		//设置 http.agent 属性
        /*
         * Sets the default HTTP User-Agent used by HttpURLConnection.
         */
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        /*
         * Wire socket tagging to traffic stats.
         */
        NetworkManagementSocketTagger.install();

        /*
         * If we're running in an emulator launched with "-trace", put the
         * VM into emulator trace profiling mode so that the user can hit
         * F9/F10 at any time to capture traces.  This has performance
         * consequences, so it's not something you want to do always.
         */
        String trace = SystemProperties.get("ro.kernel.android.tracing");
        if (trace.equals("1")) {
            Slog.i(TAG, "NOTE: emulator trace profiling enabled");
            Debug.enableEmulatorTraceOutput();
        }

        initialized = true;
    }
```

# 3.5.2 nativeZygoteInit 方法

# 3.5.3 applicationInit 方法

```java
    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);
		
		//设置虚拟机参数
        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		
		//这边会真正调用到 ActivityThread.main 方法
        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```

```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

# 四.预加载系统类和资源
在 2.3 节 ZygoteInit类的main方法中会调用 preload 方法来可进行系统类和资源的预加载。


```java
    static void preload() {
        Log.d(TAG, "begin preload");
        preloadClasses();
        preloadResources();
        preloadOpenGL();
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        Log.d(TAG, "end preload");
    }
```
title: Init进程
author: 道墟
date: 2018-05-31 14:32:19
tags:
---
本文源码所在文件:\system\core\init\init.cpp


# 一.概述
Init进程是内核启动后创建的第一个用户进程，地位非常的重要。在Init初始化的过程中会启动很多重要的守护进程，当然Init本身也是**一个守护进程**。

在介绍Init进程前先简单的介绍下Android的启动过程
# 1.1 bootloader引导
# 1.1.1 什么是bootloader引导
当我们按下手机的电源键时，最先运行的就是bootloader.bootloader主要作用就是初始化基本的硬件设备（CPU,内存，Flash等）并且通过建立内存空间的映射，为装载linux内核准备好适合的环境。内核装载完毕，bootloader将会从内存中清除。

# 1.1.2 Fastboot模式
Fastboot是Android设计的一套通过usb来更新手机分区映像的协议。

# 1.1.3 Recovery模式
Android特有的升级系统。利用该模式可以进行恢复出厂设置，执行ota、补丁和固件升级。进入到Recovery模式实际上就是进入了一个文字模式的Linux

# 1.2 装载和启动Linux内核
Android的boot.img存放的就是Linux的内核和一个根文件系统。上面的bootloader会把boot.img映像装载进内存。然后Linux内核会执行整个系统的初始化，完成整个后装载整个根文件系统，最后启动Init进程

> 什么是根文件系统？
根文件系统是Linux系统一种特殊的文件系统，Android是基于Linux的，当然也有根文件系统。android的根文件系统是基于busybox实现的。


# 1.3 启动android系统

android系统的启动可以更加详细的分为以下几个阶段

# 1.3.1 启动Init进程
Init进程是系统启动的**第一个进程**。在Init的启动过程中会解析Linux的配置脚本init.rc文件。解析init.rc文件，init进程会加载android的**文件系统**、**创建系统目录**、**初始化属性系统**、**启动android系统重要的守护进程**（USB守护进程、adb守护进程、vold守护进程、rild守护进程）。

Init作为守护进程的作用：修改属性请求、重启崩溃的进程等操作。

# 1.3.2 启动ServiceManager
由init启动。管理binder服务，负责binder的注册和查找。

# 1.3.3 启动Zygote进程
Init进程初始化结束时会启动Zygote进程。Zygote进程负责fork应用进程。是所有应用进程的父进程。

Zygote进程初始化时会创建Dalivik虚拟机、预装载系统的资源文件和Java类。从Zygote中fork的进程都会继承和共享这些预加载的资源。

启动结束后，转为守护进程来响应应用建立的请求。

# 1.3.4 启动SystemService
SystemService是Zygote进程fork的第一个进程，也是android系统的核心进程。在SystemService中运行这android大部分的binder服务。

首先会启动SensorService,接着是ActivityManagerService（AMS）,WindowsManagerService（WMS）,PackagerManagerService(PKMS)等服务。

# 1.3.5 启动Launcher
SystemServer加载完所有的服务后，最后会调用AMS中的systemReady()方法。在这个方法的执行过程中会发出Intent(android.intent.category.Home).凡是响应这个Intent的应用都会启动起来（这个流程我们到时候在AMS的分析部分会重点追踪整个过程），这边要跟开机广播区别开来。

# 二.Init进程的初始化过程
Init进程的源码位于\system\core\init\下。程序的入口函数main()位于init.cpp中。

# 2.1 main函数的流程
# 2.1.1 选择启动程序

因为Init和ueventd和watchdogd守护进程的代码存在大量的代码重合，所以直接合并到一个文件中，通过参数来判断执行那个守护进程。

```cpp
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
```
# 2.1.2 设置进程创建的文件的属性

**缺省情况下一个进程创建的文件属性是022**，使用umask可以设置文件属性的掩码。参数为0表示掩码为0777。

```cpp
    // Clear the umask.
    umask(0);
```

# 2.1.3 创建目录和挂载文件系统

```cpp
    //确保只执行一次
    bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);

    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    if (is_first_stage) {
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        mount("proc", "/proc", "proc", 0, NULL);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }
```

tmpfs是一种基于内存的文件系统，mount后就可以使用。tmpfs文件系统下的所有文件都存放在内存中，访问速度快，但是关机后会消失，适合用来存放临时性的文件。而且tmpfs文件系统的大小是动态变化的，一开始很小，随着文件的增多会随之变大。从上面的代码我们可以看到Android系统将tmpfs文件系统挂载到/dev目录下，而这个目录是用来保存系统创造的设备节点，正好符合tmpfs的特点。

devpts是虚拟终端文件系统，通常mount在目录/dev/pts下。

# 2.1.4 初始化log系统


```cpp
	//将标准输入、输出、错误重定向到空设备文件/dev/null下，这是守护进程常用的手段
    open_devnull_stdio();
	
	//创建设备节点/dev/kmsg,这样init进程就可以使用kernel的log系统，之所以是用使用kernel的log系统是因为这时候android层的log系统还没有启动起来
    klog_init();
	
	//设置log级别
    klog_set_level(KLOG_NOTICE_LEVEL);
	
	//打印init进程开始启动的日志
    NOTICE("init%s started!\n", is_first_stage ? "" : " second stage");
```

> 日志输出级别的宏定义如下
 KLOG_ERROR_LEVEL 3
 KLOG_WARNING_LEVEL 4
 KLOG_NOTICE_LEVEL 5
 KLOG_INFO_LEVEL 6
 KLOG_DEBUG_LEVEL 7
 KLOG_DEFAULT_LEVEL 3 //默认为3
当我们设置的级别低于5的时候会输出到 kernel log 中


# 2.1.5 初始化属性系统

```cpp
    if (!is_first_stage) {
		//在/dev下创建一个设备表示正在初始化中，当初始化完成则移除
		// Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
		
		//创建一个共享区域来存储属性值
        property_init();

        // If arguments are passed both on the command line and in DT,
        // properties set in DT always have priority over the command-line ones.
        process_kernel_dt();
		//解析kernel的启动参数
        process_kernel_cmdline();

        // Propogate the kernel variables to internal variables
        // used by init as well as the current required properties.
        export_kernel_boot_props();
    }
```
# 2.1.6 初始化SELinux内核
SELinux内核是android4.3开始引入的。在后面的解析系列中会重点解析这块。
```cpp
    // Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
    selinux_initialize(is_first_stage);

    // If we're in the kernel domain, re-exec init to transition to the init domain now
    // that the SELinux policy has been loaded.
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }

    // These directories were necessarily created before initial policy load
    // and therefore need their security context restored to the proper value.
    // This must happen before /dev is populated by ueventd.
    INFO("Running restorecon...\n");
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon_recursive("/sys");

    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        ERROR("epoll_create1 failed: %s\n", strerror(errno));
        exit(1);
    }
```
# 2.1.7 初始化子进程退出的信号处理过程

```cpp
    signal_handler_init();
```

# 2.1.8 设置系统属性的默认值
```cpp
    property_load_boot_defaults();
```
上面的函数会从设备的根目录下的default.prop文件(路径:/default.prop)的属性值读取设置到属性系统中。

下面是截取的nexu5x 6.0系统模拟器下的文件值
```
#
# ADDITIONAL_DEFAULT_PROPERTIES
#
ro.secure=1
ro.allow.mock.location=0
ro.debuggable=1
ro.zygote=zygote32
dalvik.vm.image-dex2oat-Xms=64m
dalvik.vm.image-dex2oat-Xmx=64m
dalvik.vm.dex2oat-Xms=64m
dalvik.vm.dex2oat-Xmx=512m
ro.dalvik.vm.native.bridge=0
debug.atrace.tags.enableflags=0
#
# BOOTIMAGE_BUILD_PROPERTIES
#
ro.bootimage.build.date=Wed Dec 13 00:51:01 UTC 2017
ro.bootimage.build.date.utc=1513126261
ro.bootimage.build.fingerprint=Android/sdk_google_phone_x86/generic_x86:6.0/MASTER/4499259:userdebug/test-keys
persist.sys.usb.config=adb

```
# 2.1.9 启动属性服务（sockect）
```cpp
    start_property_service();
```

# 2.1.10 解析init.rc文件
解析完成后的结果是将init文件中的Server项和Action项分别加入到内部Service列表:service_list 和Action列表:action_list
解析的过程会在下面的详细的分析

```cpp
    init_parse_config_file("/init.rc");
```

# 2.1.11 初始化执行列表action_queue

```cpp
    //将rc文件中触发器为early-init的action添加到执行列表
	action_for_each_trigger("early-init", action_add_queue_tail);
	
	//queue_builtin_action 作用就是将一个函数和一个名称生成action并插入到执行列表
    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
	//等待冷插拔设备初始化完成
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    //从硬件RNG的设备文件/dev/hw_randow中读取512字节并写到Linux RNGd的设备文件/dev/urandom中（这块因为水平有限，还未知道具体作用）
	queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
	//初始化组合键监听模块
    queue_builtin_action(keychord_init_action, "keychord_init");
	//在屏幕上显示Android字样的Logo
    queue_builtin_action(console_init_action, "console_init");
		
	//将rc文件中触发器为init的action添加到执行列表
    // Trigger all the boot actions to get us started.
    action_for_each_trigger("init", action_add_queue_tail);

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

	//当处于充电模式，则charger加入执行队列；否则late-init加入队列。
    // Don't mount filesystems or start core system services in charger mode.
    char bootmode[PROP_VALUE_MAX];
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }
	
	//检查Action列表中通过修改属性来触发的Action，查看相关的属性是否已经设置了，如果已经设置则将该action加入到执行列表
    // Run all property triggers based on current state of the properties.
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");
```

# 2.1.12 无限循环执行
```cpp
   while (true) {
        if (!waiting_for_exec) {
            execute_one_command();
            restart_processes();
        }

        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (!action_queue_empty() || cur_action) {
            timeout = 0;
        }

        bootchart_sample(&timeout);

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

```
# 2.2 启动Service进程

上面最后无限循环的过程中 restart_processes 函数会检查service_list中所有服务，对于带有SVC_RESTARTING标志的服务，将服务作为参数调用 restart_service_if_needed

```cpp
static void restart_service_if_needed(struct service *svc) {
    time_t next_start_time = svc->time_started + 5;

    if (next_start_time <= gettime()) {
        svc->flags &= (~SVC_RESTARTING);
        service_start(svc, NULL); 
        return;
    }

    if ((next_start_time < process_needs_restart) ||
        (process_needs_restart == 0)) {
        process_needs_restart = next_start_time;
    }
}
```

由上面代码可以看出最后调用了service_start方法，下面我们来分析这个方法。

# 2.2.1 重置Service结构中的标志和可执行判断
```cpp
    // Starting a service removes it from the disabled or reset state and
    // immediately takes it out of the restarting state if it was in there.
	//这四个表示都是有启动相关的所以直接去掉
    svc->flags &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));
    svc->time_started = 0;
	
	//如果已经在启动了则不需要再执行
    // Running processes require no additional work --- if they're in the
    // process of exiting, we've ensured that they will immediately restart
    // on exit, unless they are ONESHOT.
    if (svc->flags & SVC_RUNNING) {
        return;
    }
	
	//如果服务需要控制台，但是控制台没有启动则退出
    bool needs_console = (svc->flags & SVC_CONSOLE);
    if (needs_console && !have_console) {
        ERROR("service '%s' requires console\n", svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }

	//检查服务的二进制文件是否存在
    struct stat s;
    if (stat(svc->args[0], &s) != 0) {
        ERROR("cannot find '%s', disabling '%s'\n", svc->args[0], svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }

	//检查服务是否有SVC_ONESHOT参数
    if ((!(svc->flags & SVC_ONESHOT)) && dynamic_args) {
        ERROR("service '%s' must be one-shot to use dynamic args, disabling\n",
               svc->args[0]);
        svc->flags |= SVC_DISABLED;
        return;
    }
	
```


# 2.2.2 设置安全上下文

关于安全机制在后续的文章中分析

```cpp
    char* scon = NULL;
    if (is_selinux_enabled() > 0) {
        if (svc->seclabel) {
            scon = strdup(svc->seclabel);
            if (!scon) {
                ERROR("Out of memory while starting '%s'\n", svc->name);
                return;
            }
        } else {
            char *mycon = NULL, *fcon = NULL;

            INFO("computing context for service '%s'\n", svc->args[0]);
            int rc = getcon(&mycon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }

            rc = getfilecon(svc->args[0], &fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                freecon(mycon);
                return;
            }

            rc = security_compute_create(mycon, fcon, string_to_security_class("process"), &scon);
            if (rc == 0 && !strcmp(scon, mycon)) {
                ERROR("Warning!  Service %s needs a SELinux domain defined; please fix!\n", svc->name);
            }
            freecon(mycon);
            freecon(fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }
        }
    }
```

# 2.2.3 fork子进程,exec

```cpp
	
    //首先打印 服务启动日志
    NOTICE("Starting service '%s'...\n", svc->name);
	
    //fork进程
    pid_t pid = fork();
    if (pid == 0) {
        struct socketinfo *si;
        struct svcenvinfo *ei;
        char tmp[32];
        int fd, sz;

        umask(077);
		
        //准备环境变量
        if (properties_initialized()) {
            get_property_workspace(&fd, &sz);
            snprintf(tmp, sizeof(tmp), "%d,%d", dup(fd), sz);
            add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
        }

        for (ei = svc->envvars; ei; ei = ei->next)
            add_environment(ei->name, ei->value);

        //如果服务选项中有socket选项。这时候就开始创建参数中定义的socket。
       for (si = svc->sockets; si; si = si->next) {
            int socket_type = (
                    !strcmp(si->type, "stream") ? SOCK_STREAM :
                        (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
            int s = create_socket(si->name, socket_type,
                                  si->perm, si->uid, si->gid, si->socketcon ?: scon);
            if (s >= 0) {
                publish_socket(si->name, s);
            }
        }
        freecon(scon);
        scon = NULL;

        if (svc->writepid_files_) {
            std::string pid_str = android::base::StringPrintf("%d", pid);
            for (auto& file : *svc->writepid_files_) {
                if (!android::base::WriteStringToFile(pid_str, file)) {
                    ERROR("couldn't write %s to %s: %s\n",
                          pid_str.c_str(), file.c_str(), strerror(errno));
                }
            }
        }

        if (svc->ioprio_class != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), svc->ioprio_class, svc->ioprio_pri)) {
                ERROR("Failed to set pid %d ioprio = %d,%d: %s\n",
                      getpid(), svc->ioprio_class, svc->ioprio_pri, strerror(errno));
            }
        }
		
       //处理标准输入 标准输出 标准错误3个文件描述符
        if (needs_console) {
            setsid();
            open_console();
        } else {
            zap_stdio();
        }
		
        //这边明显不执行
        if (false) {
            for (size_t n = 0; svc->args[n]; n++) {
                INFO("args[%zu] = '%s'\n", n, svc->args[n]);
            }
            for (size_t n = 0; ENV[n]; n++) {
                INFO("env[%zu] = '%s'\n", n, ENV[n]);
            }
        }

        setpgid(0, getpid());

        // As requested, set our gid, supplemental gids, and uid.
        if (svc->gid) {
            if (setgid(svc->gid) != 0) {
                ERROR("setgid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->nr_supp_gids) {
            if (setgroups(svc->nr_supp_gids, svc->supp_gids) != 0) {
                ERROR("setgroups failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->uid) {
            if (setuid(svc->uid) != 0) {
                ERROR("setuid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->seclabel) {
            if (is_selinux_enabled() > 0 && setexeccon(svc->seclabel) < 0) {
                ERROR("cannot setexeccon('%s'): %s\n", svc->seclabel, strerror(errno));
                _exit(127);
            }
        }
		
        //执行exec
        if (!dynamic_args) {
            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
            }
        } else {
            char *arg_ptrs[INIT_PARSER_MAXARGS+1];
            int arg_idx = svc->nargs;
            char *tmp = strdup(dynamic_args);
            char *next = tmp;
            char *bword;

            /* Copy the static arguments */
            memcpy(arg_ptrs, svc->args, (svc->nargs * sizeof(char *)));

            while((bword = strsep(&next, " "))) {
                arg_ptrs[arg_idx++] = bword;
                if (arg_idx == INIT_PARSER_MAXARGS)
                    break;
            }
            arg_ptrs[arg_idx] = NULL;
            execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
        }
        _exit(127);
    }
```
# 三. 解析启动脚本init.rc

# 3.1 init.rc文件格式

# 3.2 init脚本的关键字定义

# 3.3 脚本文件解析过程

# 3.4 init中启动的守护进程











title: Init进程
author: 道墟
tags: []
categories:
  - android6.0
date: 2018-05-31 14:32:00
---
# 一.概述
Init进程是Linux系统中用户空间的第一个进程，Android是基于Linux内核的，所以init也是Android系统中用户空间的第一个进程，进程号是1。在Init初始化的过程中会启动很多重要的守护进程，当然Init本身也是**一个守护进程**。



# 二.Init进程的初始化过程

Init进程的源码位于\system\core\init\下。程序的入口函数main()位于init.cpp中。

# 2.1 main函数的流程

```cpp
int main(int argc, char** argv) {
    //2.1.1 选择启动程序
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
	
    //2.1.2 设置进程创建的文件的属性
    // Clear the umask.
    umask(0);

    add_environment("PATH", _PATH_DEFPATH);

    //2.1.3 创建目录和挂载文件系统
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

    //2.1.4 初始化log系统
    // We must have some place other than / to create the device nodes for
    // kmsg and null, otherwise we won't be able to remount / read-only
    // later on. Now that tmpfs is mounted on /dev, we can actually talk
    // to the outside world.
    open_devnull_stdio();
    klog_init();
    klog_set_level(KLOG_NOTICE_LEVEL);

    NOTICE("init%s started!\n", is_first_stage ? "" : " second stage");
	
    //2.1.5 初始化属性系统
    if (!is_first_stage) {
        // Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

        property_init();

        // If arguments are passed both on the command line and in DT,
        // properties set in DT always have priority over the command-line ones.
        process_kernel_dt();
        process_kernel_cmdline();

        // Propogate the kernel variables to internal variables
        // used by init as well as the current required properties.
        export_kernel_boot_props();
    }

    //2.1.6 初始化SELinux内核
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

    // 2.1.7 初始化子进程退出的信号处理过程
    signal_handler_init();
	
    //2.1.8 设置系统属性的默认值
    property_load_boot_defaults();
	
	//2.1.9 启动属性服务（sockect）
    start_property_service();

    //2.1.10 解析init.rc文件
    init_parse_config_file("/init.rc");
	
    //2.1.11 初始化执行列表action_queue
    action_for_each_trigger("early-init", action_add_queue_tail);

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    queue_builtin_action(keychord_init_action, "keychord_init");
    queue_builtin_action(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    action_for_each_trigger("init", action_add_queue_tail);

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    char bootmode[PROP_VALUE_MAX];
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }

    // Run all property triggers based on current state of the properties.
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");
    //2.1.12 无限循环执行
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

    return 0;
}
```


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
读取相应文件的属性，下文5.3中有介绍。
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
	//这四个都是跟启动相关的所以直接去掉
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

init.rc文件是以块为单位，主要分为两类：
- 动作:action,以on开头;
- 服务:service，以service开头。
- 选项:Options
- 命令:Commands 
注释以#开头。无论是action还是service的执行顺序都不是以文件中的编排顺序执行的，**执行与否和是否执行都是Init进程中决定的**。

# 3.1.1 action

格式：
on [trigger] 触发条件
   [Command1]
   [Command2]
   [Command3]

在on后面紧跟的字符串是action的触发器，触发器后面的是命令列表，每一行都是一个命令。可以通过 trigger 触发器字符串（trigger late-init） 来触发。

当相应的事件发生后，系统会对rc文件中的各个 trigger 进行匹配，匹配到的action会被加入到 命令执行列表的尾部，已经在队伍中的action不会被重复添加。

触发器几种类别
- on early-init; 在初始化早期阶段触发；
- on init; 在初始化阶段触发；
- on late-init; 在初始化晚期阶段触发；
- on boot/charger： 当系统启动/充电时触发，还包含其他情况，此处不一一列举；
- on property:<key>=<value>: 当属性值满足条件时触发；


```
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init

# Load properties from /system/ + /factory after fs mount.
on load_system_props_action
    load_system_props

on load_persist_props_action
    load_persist_props
    start logd
    start logd-reinit
```


# 3.1.2 service

格式：
service [name:名称] [pathname：路径] [argument：参数]
	[option]
	[option]

在service后面是服务的名称，我们可以使用“start”命令加服务名称来启动一个服务（start logd）。名称后面的则是执行文件路径和执行参数。然后下面的行称为选项，每一行都是一个选项。例如class表示服务的类别，我们可以通过class_start来一次性启动一组服务。
```
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0

service logd /system/bin/logd
    class core
    socket logd stream 0666 logd logd
    socket logdr seqpacket 0666 logd logd
    socket logdw dgram 0222 logd logd
    group root system
     writepid /dev/cpuset/system-background/tasks

service logd-reinit /system/bin/logd --reinit
    oneshot
    writepid /dev/cpuset/system-background/tasks
    disabled

service healthd /sbin/healthd
    class core
    critical
    seclabel u:r:healthd:s0
    group root system

service console /system/bin/sh
    class core
    console
    disabled
    user shell
    group shell log
    seclabel u:r:shell:s0
```
# 3.1.3 Options 选项
选项是service的修订项，它决定了服务何时运行以及怎么运行。


- disabled: 表示不能通过触发器来触发，只能根据start service名开启动；

- oneshot: service退出后不再重启；

- user/group： 设置执行服务的用户/用户组，默认都是root；

- class：设置所属的类名，当所属类启动/退出时，服务也启动/停止，默认为default；

- onrestart:当服务重启时执行相应命令；

- socket: 创建名为/dev/socket/<name>的socket，并把**文件描述符**传递给要启动的进程。

- critical:这是一个关键服务，如果在4分钟内重启启动4次，则系统会重启并进入Recovery模式。

# 3.1.4 Commands 命令

- class_start <service_class_name>： 启动属于同一个class的所有服务；

- start <service_name>： 启动指定的服务，若已启动则跳过；

- stop <service_name>： 停止正在运行的服务

- setprop <name> <value>：设置属性值

- mkdir <path>：创建指定目录

- symlink <target> <sym_link>： 创建连接到<target>的<sym_link>符号链接；

- write <path> <string>： 向文件path中写入字符串；

- exec： fork并执行，会阻塞init进程直到程序完毕；

- exprot <name> <name>：设定环境变量；

- loglevel <level>：设置log级别

commands的命令远不止上面这些，这里只是列出一些常用的命令。

# 3.2 脚本文件解析过程
下面我们来追踪下脚本文件的解析过程

> 源码所在文件路径如下
\system\core\init\init_parser.cpp


首先从入口函数开始

```cpp
int init_parse_config_file(const char* path) {
    INFO("Parsing %s...\n", path);
    Timer t;
    std::string data;
    if (!read_file(path, &data)) {
        return -1;
    }

    data.push_back('\n'); // TODO: fix parse_config.
    parse_config(path, data);
    dump_parser_state();

    NOTICE("(Parsing %s took %.2fs.)\n", path, t.duration());
    return 0;
}
```
从上面的函数可以看出来，首先通过read_file将文件的内容读到内存中，然后再通过parse_config函数进行解析。下面我们继续来追踪parse_config的方法：

```cpp
static void parse_config(const char *fn, const std::string& data)
{
    struct listnode import_list;
    struct listnode *node;
    char *args[INIT_PARSER_MAXARGS];

    int nargs = 0;

    parse_state state;
    state.filename = fn;
    state.line = 0;
    state.ptr = strdup(data.c_str());  // TODO: fix this code!
    state.nexttoken = 0;
    state.parse_line = parse_line_no_op;

    list_init(&import_list);
    state.priv = &import_list;

    for (;;) {
        switch (next_token(&state)) {
		//如果是结束标识符，则跳转到下面的parser_done位置
        case T_EOF:
            state.parse_line(&state, 0, 0);
            goto parser_done;
		
		//行结束符
        case T_NEWLINE:
            state.line++;
            if (nargs) {
                int kw = lookup_keyword(args[0]);
				//判断是否是section：关键字 on service import
                if (kw_is(kw, SECTION)) {
                    state.parse_line(&state, 0, 0);
					//具体处理见 3.2.1 
                    parse_new_section(&state, kw, nargs, args);
                } else {
					//当作当前section所属行处理
                    state.parse_line(&state, nargs, args);
                }
                nargs = 0;
            }
            break;
		//单词结束符 则先放入到数组中
        case T_TEXT:
            if (nargs < INIT_PARSER_MAXARGS) {
                args[nargs++] = state.text;
            }
            break;
        }
    }

parser_done:
    list_for_each(node, &import_list) {
         struct import *import = node_to_item(node, struct import, list);
         int ret;

         ret = init_parse_config_file(import->filename);
         if (ret)
             ERROR("could not import file '%s' from '%s'\n",
                   import->filename, fn);
    }
}
```


parse_new_section方法追踪

```cpp
static void parse_new_section(struct parse_state *state, int kw,
                       int nargs, char **args)
{
    printf("[ %s %s ]\n", args[0],
           nargs > 1 ? args[1] : "");
    switch(kw) {
    case K_service:
        state->context = parse_service(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_service;
            return;
        }
        break;
    case K_on:
        state->context = parse_action(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_action;
            return;
        }
        break;
    case K_import:
        parse_import(state, nargs, args);
        break;
    }
    state->parse_line = parse_line_no_op;
}
```
由上面的函数可以看出，该方法通过传入的参数kw来决定用什么方法来处理：

- on关键字：parse_action
- service关键字：parse_service
- import关键字：parse_action

然后我们可以看出上面对结构体state的parse_line字段分别进行赋值了方法地址,该方法用来解析命令行。

- on关键字：parse_line_action
- service关键字：parse_line_service


# 3.3 执行action

3.2的解析过程只是将init.rc中的action和service添加到各自的列表中。真正将它们添加到执行列表的还是在init进程中处理的。上文2.1.11 中在init的初始化过程通过action_for_each_trigger将action添加到action_queue中。

```cpp
void action_for_each_trigger(const char *trigger,
                             void (*func)(struct action *act))
{
    struct listnode *node, *node2;
    struct action *act;
    struct trigger *cur_trigger;

    list_for_each(node, &action_list) {
        act = node_to_item(node, struct action, alist);
        list_for_each(node2, &act->triggers) {
            cur_trigger = node_to_item(node2, struct trigger, nlist);
            if (!strcmp(cur_trigger->name, trigger)) {
                func(act);
            }
        }
    }
}
```

# 四.Init进程对信号的处理

# 4.1 僵尸进程

当一个进程退出exit()时，会向它的父进程发送一个SIGCHLD信号。父进程收到该信号后，会释放分配给该子进程的系统资源；并且父进程需要调用wait()或waitpid()等待子进程结束。

如果父进程没有做这种处理，且父进程初始化时也没有调用signal(SIGCHLD, SIG_IGN)来显示忽略对SIGCHLD的处理，这时子进程将一直保持当前的退出状态，不会完全退出。这样的子进程不能被调度，所做的只是在进程列表中占据一个位置，保存了该进程的PID、终止状态、CPU使用时间等信息；我们将这种进程称为“Zombie”进程，即僵尸进程。

android中查看僵尸进程的方法是通过 adb shell ps来查看的，僵尸进程的的进程状态为“Z”。


由于僵尸进程仍会在进程列表中占据一个位置，而Linux所支持的最大进程数量是有限的；超过这个界限值后，我们就无法创建进程。所以，我们有必要清理那些僵尸进程，以保证系统的正常运作。

# 4.2 处理过程

待补充

# 五. 属性系统

# 5.1 属性系统介绍
android的属性用来保存系统设置和进程间传递一些信息。每个属性由属性名称和属性值组成，名称通常以“."分割,这些名称的前缀有特殊的含义，不能随便改动。属性值只能是字符串。

对于进程来说，读取属性值是没有限制的，任何进程都可以读取属性值。但是写属性值只能通过init进程进行，而且init进程还会检查请求的进程是否具有该权限（5.0以后只在）。当属性值修改成功后，init进程会init.rc中是否有跟该属性修改值相匹配的“触发器”。如果有则执行触发器下的命令。

前缀分类

- “ro.”：表示只读属性，一旦设置则不能改变。

- “persist”:表示属性值会被写入目录/data/property下与属性同名的文件中。下次开机init进程会从中读取值。所以这边设置的值是永久生效的。

- “ctl”:表示控制信息，用来执行一些命令：ctl.start、ctl.stop、ctl.restart
	- setprop ctl.start bootanim：查看开机动画
	- setprop ctl.stop bootanim：关闭开机动画
	- setprop ctl.start pre-recovery：进入recovery模式

# 5.2 创建属性系统的共享空间
在上面Init进程的初始化函数中我们调用了property_init函数为属性系统创建了一块共享内存区域。这块区域只能由init进程进行写入，读则不限制。

```cpp
void property_init() {
	//防止多次初始化
    if (property_area_initialized) {
        return;
    }

    property_area_initialized = true;
	
	//创建和初始化属性的共享内存空间，这块内存空间由/dev/__properties__ 设备创建,这块空间的文件描述符保存在
	//static workspace pa_workspace; 中，最后在service_start的时候保存到环境变量中
    if (__system_property_area_init()) {
        return;
    }

    pa_workspace.size = 0;
    pa_workspace.fd = open(PROP_FILENAME, O_RDONLY | O_NOFOLLOW | O_CLOEXEC);
    if (pa_workspace.fd == -1) {
        ERROR("Failed to open %s: %s\n", PROP_FILENAME, strerror(errno));
        return;
    }
}
```

# 5.3 初始化属性系统的值
通过从以下文件中读取值进行初始化

/system/build.prop：定了系统初始和永久的一些属性
/data/local.prop：这个则需要和ro.debuggable配合使用，如果这个值为1，那么从该文件中读取值覆盖系统缺省的属性值，之所有有这个设计是为了方便开发人员调试，因为/system/build.prop 是在根目录下的。
/data/property：该目录下的文件读取出来写入到属性值中。
/default.prop: Init进程初始化过程中调用property_load_boot_defaults函数读取的





# 源码地址
> 本文源码基于android6.0 
system\core\init\init.cpp
system\core\init\init_parser.cpp
system\core\init\signal_handler.cpp
system\core\init\property_service.cpp
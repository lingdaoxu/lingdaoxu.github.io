title: Init进程
author: 道墟
date: 2018-05-31 14:32:19
tags:
---
> 本文源码所在文件
\system\core\init\init.cpp


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



# 2.2 启动Service进程















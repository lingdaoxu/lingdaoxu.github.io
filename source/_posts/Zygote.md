title: Zygote进程
author: 道墟
date: 2018-06-01 16:21:17
tags:
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
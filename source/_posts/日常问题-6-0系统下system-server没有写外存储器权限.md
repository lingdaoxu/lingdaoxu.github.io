title: '日常问题:6.0系统下system_server没有写外存储器权限'
author: 道墟
date: 2018-06-14 17:42:20
tags:
---

# 问题描述
上篇blog我们讲了怎么添加系统服务来保存日志的解决方案，然后领导很开心的给加了个需求：正常情况下不抓取日志，只有当U盘插入的时候检测到有特定的文件夹的时候才开始写日志到U盘里面去，方便测试人员捕获日志。刚拿到需求的时候想了下很简单，接收下U盘挂载广播就要可以了，确实在4.4上一切顺利，很快就完成任务了，然后今天移植到6.0系统发现无法再U盘上创建和写入文件。下面来追踪下解决的思路。



# 解决方案

先从我们自定义的服务SystemLogService入手，SystemLogService是在SystemServer中初始化的，所以是属于system_server进程，所以初步怀疑是system_server进程没有权限导致的。下面我们来看看system_server进程的相关信息

# 通过  ps |grep system_server 命令来获取进程号

```
system    496   212   2023356 131972          0 0000000000 S system_server
```

# 进入到 proc 中查看虚拟文件系统查看进程目录

```
## 496 是进程号
root@rk3399_m_stbvr:/ # cd proc/496/
root@rk3399_m_stbvr:/proc/496 # ls -la
dr-xr-xr-x system   system            2013-01-18 16:50 attr
-r-------- system   system          0 2018-06-14 17:59 auxv
-r--r--r-- system   system          0 2018-06-14 17:59 cgroup
--w------- system   system          0 2018-06-14 17:59 clear_refs
-r--r--r-- system   system          0 2013-01-18 16:50 cmdline
-rw-r--r-- system   system          0 2013-01-18 16:50 comm
-r--r--r-- system   system          0 2018-06-14 17:59 cpuset
lrwxrwxrwx system   system            2018-06-14 17:59 cwd -> /
-r-------- system   system          0 2018-06-14 17:59 environ
lrwxrwxrwx system   system            2018-06-14 17:59 exe -> /system/bin/app_process64
dr-x------ system   system            2013-01-18 16:50 fd
dr-x------ system   system            2018-06-14 17:59 fdinfo
-r-------- system   system          0 2018-06-14 17:59 io
-r--r--r-- system   system          0 2018-06-14 17:59 limits
-rw-r--r-- system   system          0 2018-06-14 17:59 loginuid
dr-x------ system   system            2018-06-14 17:59 map_files
-r--r--r-- system   system          0 2018-06-14 17:59 maps
-rw------- system   system          0 2018-06-14 17:59 mem
-r--r--r-- system   system          0 2018-06-14 17:59 mountinfo
-r--r--r-- system   system          0 2013-01-18 16:50 mounts
-r-------- system   system          0 2018-06-14 17:59 mountstats
dr-xr-xr-x system   system            2013-01-18 16:50 net
dr-x--x--x system   system            2018-06-14 17:59 ns
-r-------- system   system          0 2018-06-14 17:59 oom_adj
-r--r--r-- system   system          0 2018-06-14 17:59 oom_score
-r-------- system   system          0 2013-01-18 16:50 oom_score_adj
-r-------- system   system          0 2018-06-14 17:59 pagemap
-r-------- system   system          0 2018-06-14 17:59 personality
lrwxrwxrwx system   system            2018-06-14 17:59 root -> /
-rw-r--r-- system   system          0 2018-06-14 17:59 sched
-r--r--r-- system   system          0 2018-06-14 17:59 schedstat
-r--r--r-- system   system          0 2018-06-14 17:59 sessionid
-r--r--r-- system   system          0 2018-06-14 17:42 smaps
-r-------- system   system          0 2018-06-14 17:59 stack
-r--r--r-- system   system          0 2013-01-18 16:50 stat
-r--r--r-- system   system          0 2018-06-14 17:59 statm
-r--r--r-- system   system          0 2018-06-14 17:59 status
-r-------- system   system          0 2018-06-14 17:59 syscall
dr-xr-xr-x system   system            2013-01-18 16:50 task
-rw-rw-rw- system   system          0 2018-06-14 17:59 timerslack_ns
-r--r--r-- system   system          0 2013-01-18 16:50 wchan
```

# 查看status文件
```
root@rk3399_m_stbvr:/proc/496 # cat status
Name:   system_server
State:  S (sleeping)
Tgid:   496
Ngid:   0
Pid:    496
PPid:   212
TracerPid:      0
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
FDSize: 512
# 1015 和 1023 原来是没有的，后面修改后才有的
Groups: 1001 1002 1003 1004 1005 1006 1007 1008 1009 1010 1015 1018 1021 1023 1032 3001 3002 3003 3006 3007
VmPeak:  2155712 kB
VmSize:  2023356 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:    133496 kB
VmRSS:    131172 kB
VmData:    93112 kB
VmStk:      8196 kB
VmExe:        16 kB
VmLib:    101224 kB
VmPTE:       676 kB
VmPMD:        28 kB
VmSwap:        0 kB
Threads:        76
SigQ:   2/7258
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000001204
SigIgn: 0000000000000000
SigCgt: 00000002000094f8
CapInh: 0000000000000000
CapPrm: 0000001007813c20
CapEff: 0000001007813c20
CapBnd: 0000000000000000
CapAmb: 0000000000000000
Seccomp:        0
Cpus_allowed:   3f
Cpus_allowed_list:      0-5
Mems_allowed:   1
Mems_allowed_list:      0
voluntary_ctxt_switches:        3557
nonvoluntary_ctxt_switches:     1410
```

# 查看U盘的所属用户组

因为这边挂载了两个文件系统，所以都需要查看

/mnt/media_rw/E844-FF71
```
root@rk3399_m_stbvr:/mnt/media_rw/E844-FF71 # ls -la
-rwxrwx--- media_rw media_rw 17127363 2013-05-03 16:27 8ch_voices_id_21_ddp.h264
-rwxrwx--- media_rw media_rw 34050184 2013-05-03 16:19 8ch_voices_id_21_ddp_DVB_mpeg2_25fps.trp
drwxrwx--- media_rw media_rw          2018-06-11 18:11 Android
drwxrwx--- media_rw media_rw          2018-06-11 18:06 LOST.DIR
drwxrwx--- media_rw media_rw          2018-06-14 15:55 System Volume Information
drwxrwx--- media_rw media_rw          2018-06-14 18:10 logcat
drwxrwx--- media_rw media_rw          2018-06-12 18:29 鏈€鏂版祴璇曚娇鐢ㄧ殑鐮佹祦
```


/storage/E844-FF71
```
root@rk3399_m_stbvr:/storage/E844-FF71 # ls -la
-rwxrwx--x root     sdcard_rw 17127363 2013-05-03 16:27 8ch_voices_id_21_ddp.h264
-rwxrwx--x root     sdcard_rw 34050184 2013-05-03 16:19 8ch_voices_id_21_ddp_DVB_mpeg2_25fps.trp
drwxrwx--x root     sdcard_rw          2018-06-11 18:11 Android
drwxrwx--x root     sdcard_rw          2018-06-11 18:06 LOST.DIR
drwxrwx--x root     sdcard_rw          2018-06-14 15:55 System Volume Information
drwxrwx--x root     sdcard_rw          2018-06-14 18:10 logcat
drwxrwx--x root     sdcard_rw          2018-06-12 18:29 鏈€鏂版祴璇曚娇鐢ㄧ殑鐮佹祦
```

# 查看所属用户组对应的数值

\system\core\include\private\android_filesystem_config.h
```
#define AID_ROOT             0  /* traditional unix root user */

#define AID_SYSTEM        1000  /* system server */

#define AID_RADIO         1001  /* telephony subsystem, RIL */
#define AID_BLUETOOTH     1002  /* bluetooth subsystem */
#define AID_GRAPHICS      1003  /* graphics devices */
#define AID_INPUT         1004  /* input devices */
#define AID_AUDIO         1005  /* audio devices */
#define AID_CAMERA        1006  /* camera devices */
#define AID_LOG           1007  /* log devices */
#define AID_COMPASS       1008  /* compass device */
#define AID_MOUNT         1009  /* mountd socket */
#define AID_WIFI          1010  /* wifi subsystem */
#define AID_ADB           1011  /* android debug bridge (adbd) */
#define AID_INSTALL       1012  /* group for installing packages */
#define AID_MEDIA         1013  /* mediaserver process */
#define AID_DHCP          1014  /* dhcp client */
#define AID_SDCARD_RW     1015  /* external storage write access */
#define AID_VPN           1016  /* vpn system */
#define AID_KEYSTORE      1017  /* keystore subsystem */
#define AID_USB           1018  /* USB devices */
#define AID_DRM           1019  /* DRM server */
#define AID_MDNSR         1020  /* MulticastDNSResponder (service discovery) */
#define AID_GPS           1021  /* GPS daemon */
#define AID_UNUSED1       1022  /* deprecated, DO NOT USE */
#define AID_MEDIA_RW      1023  /* internal media storage write access */
#define AID_MTP           1024  /* MTP USB driver access */
#define AID_UNUSED2       1025  /* deprecated, DO NOT USE */
#define AID_DRMRPC        1026  /* group for drm rpc */
#define AID_NFC           1027  /* nfc subsystem */
#define AID_SDCARD_R      1028  /* external storage read access */
#define AID_CLAT          1029  /* clat part of nat464 */
#define AID_LOOP_RADIO    1030  /* loop radio devices */
#define AID_MEDIA_DRM     1031  /* MediaDrm plugins */
#define AID_PACKAGE_INFO  1032  /* access to installed package details */
#define AID_SDCARD_PICS   1033  /* external storage photos access */
#define AID_SDCARD_AV     1034  /* external storage audio/video access */
#define AID_SDCARD_ALL    1035  /* access all users external storage */
#define AID_LOGD          1036  /* log daemon */
#define AID_SHARED_RELRO  1037  /* creator of shared GNU RELRO files */

#define AID_SHELL         2000  /* adb and debug shell user */
#define AID_CACHE         2001  /* cache access */
#define AID_DIAG          2002  /* access to diagnostic resources */

/* The range 2900-2999 is reserved for OEM, and must never be
 * used here */
#define AID_OEM_RESERVED_START 2900
#define AID_OEM_RESERVED_END   2999

/* The 3000 series are intended for use as supplemental group id's only.
 * They indicate special Android capabilities that the kernel is aware of. */
#define AID_NET_BT_ADMIN  3001  /* bluetooth: create any socket */
#define AID_NET_BT        3002  /* bluetooth: create sco, rfcomm or l2cap sockets */
#define AID_INET          3003  /* can create AF_INET and AF_INET6 sockets */
#define AID_NET_RAW       3004  /* can create raw INET sockets */
#define AID_NET_ADMIN     3005  /* can configure interfaces and routing tables. */
#define AID_NET_BW_STATS  3006  /* read bandwidth statistics */
#define AID_NET_BW_ACCT   3007  /* change bandwidth statistics accounting */
#define AID_NET_BT_STACK  3008  /* bluetooth: access config files */

#define AID_EVERYBODY     9997  /* shared between all apps in the same profile */
#define AID_MISC          9998  /* access to misc storage */
#define AID_NOBODY        9999

#define AID_APP          10000  /* first app user */

#define AID_ISOLATED_START 99000 /* start of uids for fully isolated sandboxed processes */
#define AID_ISOLATED_END   99999 /* end of uids for fully isolated sandboxed processes */

#define AID_USER        100000  /* offset for uid ranges for each user */

#define AID_SHARED_GID_START 50000 /* start of gids for apps in each user to share */
#define AID_SHARED_GID_END   59999 /* start of gids for apps in each user to share */
```

这边我们可以查找 sdcard_rw 对应的是 AID_SDCARD_RW 1015,media_rw对应的是 AID_MEDIA_RW 1023

# 修改 system_server 的所属用户组

system_server是由ZygoteInit类负责初始化和启动的，frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中。在startSystemServer方法中启动sysetm_server时通过 --setgroups 为其设置了所属用户组，代码如下

```java
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1015,1023,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
```
在这里添加对应的用户组


# 总结
解决的步骤如下
- 查看进程当前所属的组
- 查看需要的用户组
- 查看对应用户组的值
- 修改启动参数添加组


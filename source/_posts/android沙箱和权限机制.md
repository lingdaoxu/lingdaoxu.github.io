title: android沙箱和权限机制
author: 道墟
date: 2018-07-01 18:24:52
tags:
---

# 进程隔离
android中进程的地址空间是独立的，一个进程不能直接访问另一个进程的地址空间。

#应用程序沙箱
在每个应用程序的安装阶段，Android自动为每个应用赋予一个独一无二的UID，通常称作appid，应用执行时就在特定进程内以该UID运行。

系统守护进程和应用程序都是在明确的、恒定的UID运行的，很少有守护进程以root用户（UID=0）运行。这些UID是静态定义在 \system\core\include\private\android_filesystem_config.h 文件中。

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

ps 查看 com.daoxu.myapplication 的信息

```
USER      PID   PPID  VSIZE  RSS   WCHAN            PC  NAME
u0_a62    6399  1362  1274908 36840 SyS_epoll_ b739f2b5 S com.daoxu.myapplication
```

/data/system/package.list 查看 com.daoxu.myapplication 进程信息
```
com.daoxu.myapplication 10062 1 /data/data/com.daoxu.myapplication default none

```



> 系统服务的UID从1000开始。1000是system用户（AID_SYSTEM），具有特殊的权限。
  应用程序是从10000开始（AID_APP），在ps中查看用户到的USER信息为 u0_a62 （支持多用户的androifd版本是uY_aXXX的格式，如果不支持多用户的则是app_XXX），XXX表示从AID_APP开始的偏移量（myapplication的为 10062-10000=62），Y则是Android的user ID（主要这边跟uid是不同的，是不同用户的的编号），所以上面ps看到的信息是u0_a62。

# 权限
#权限的本质
如前面所说的，Android应用程序是沙箱隔离的，默认只能访问程序自己创建的文件和非常有限的系统服务，为了与系统和其他应用交互，Android引入了权限机制。

#权限查看

可以使用pm list permissons查看当前系统已知的权限列表。通常权限名称以定义它的程序的包名作为前缀，再接上permission字符串。内置的权限都是在android包里面定义的（这个在pkms中用详细的介绍），所以一般都是 android.permission 作为前缀。

```
1|root@generic_x86:/ # pm list permissions                                     
All Permissions:

permission:android.permission.REAL_GET_TASKS
permission:android.permission.ACCESS_CACHE_FILESYSTEM
permission:android.permission.REMOTE_AUDIO_PLAYBACK
permission:com.google.android.apps.photos.permission.C2D_MESSAGE
permission:android.permission.INTENT_FILTER_VERIFICATION_AGENT
permission:android.permission.BIND_INCALL_SERVICE
permission:com.google.android.gms.trustagent.framework.model.DATA_CHANGE_NOTIFICATION
permission:android.permission.WRITE_SETTINGS
permission:android.permission.CONTROL_KEYGUARD
permission:com.google.android.calendar.permission.C2D_MESSAGE
permission:android.permission.CONFIGURE_WIFI_DISPLAY
permission:android.permission.ACCESS_WIMAX_STATE
permission:android.permission.SET_INPUT_CALIBRATION
permission:android.permission.RECOVERY
permission:android.permission.TEMPORARY_ENABLE_ACCESSIBILITY

```

#权限申请
在应用程序的AndroidManifest.xml文件中添加 <uses-permission></uses-permission> 标签。

```
    <uses-permission android:name="android.permission.INTERNET"></uses-permission>
```

#权限管理
在每个应用程序安装时，系统使用 **包管理器服务** ，将权限赋予它们。包管理器维护一个已安装程序包的核心数据库，包括预安装（系统应用）和用户安装的程序包。其中包括如下信息：
- 安装路径
- 版本
- 签名证书
- 每个包的权限
- 所有已定义权限的列表。
这个包数据库以xml的形式存放在/data/system/packages.xml中，文件会随着应用的安装、升级和卸载进行更新。下面我们跟踪一个包来查看下。

首先我们在程序包里申请了如下三个权限

```
    <uses-permission android:name="android.permission.INTERNET"></uses-permission>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"></uses-permission>
```
在 targetSdkVersion 27 查看文件信息

```
    <package name="com.daoxu.securityclientdemo" codePath="/data/app/com.daoxu.securityclientdemo-1" nativeLibraryPath="/data/app/com.daoxu.securityclientdemo-1/lib" publicFlags="944291654" privateFlags="0" ft="16456479468" it="1645642beba" ut="16456479c3e" version="1" userId="10065">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.INTERNET" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="7" />
    </package>
```



在 targetSdkVersion 22 查看文件信息

```
    <package name="com.daoxu.securityclientdemo" codePath="/data/app/com.daoxu.securityclientdemo-1" nativeLibraryPath="/data/app/com.daoxu.securityclientdemo-1/lib" publicFlags="944291654" privateFlags="0" ft="164564de598" it="164564dec1b" ut="164564dec1b" version="1" userId="10066">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.READ_EXTERNAL_STORAGE" granted="true" flags="0" />
            <item name="android.permission.WRITE_EXTERNAL_STORAGE" granted="true" flags="0" />
        </perms>
        <proper-signing-keyset identifier="7" />
    </package>
```

从上面的例子我们可以看出在不同的targetSdkVersion版本，给的授权列表是不同的，之所以会这样是因为6.0以后的动态权限机制导致的，这就涉及到了下面我们要讲的权限的保护分级。

#权限级别
- normal 级别：默认值，低风险权限，无需用户确认。

- dangerous 级别：危险级别，需要用户确认才能授权。

- signature 级别：只会赋予与申明权限的应用程序使用相同签名的应用权限。一般我们所说的系统权限（这边所谓的系统权限要和 sharedUserId 的system 区别开来不是一个概念）其实就是指的 system 应用所声明的权限，所以应用程序在使用这些权限的时候需要使用系统签名（不需要声明 sharedUserId 也是可以的）。

- signatureOrSystem 级别：这个级别的权限可以授权与 “系统应用” 和 与声明权限具有相同签名的应用程序，所以这个算是一个折中的方案。需要注意的是 在4.3 以前 这个  “系统应用”  是泛指在system分区下的应用，而4.3以后是特指 /system/priv-app目录下的应用。

# 权限的赋予
权限的执行会在Android的各个层次上实施。
- 高层的组件:应用和系统服务，通过包管理器查询应用程序被赋予的是哪些权限
- 底层的组件：本地的守护进程。通常不通过包管理器，而是依赖进程的UID、GID和补充GID来决定赋予的权限。
- 系统资源：设备文件、UNIX域套接字和网络套接字，通过内核根据所有者、目标资源的访问权限和访问进程的UID和GID来进行控制。

#权限和进程属性

每个应用程序在安装时都会被分配一个独一无二的UID，在一个专有的进程中执行（有待商榷，一个应用程序可以是多个进程）。应用启动的时候，进程的UID和GID由包管理器服务设置为应用程序的UID。如果应用已被赋予额外的权限，就把这些权限映射成一组GID,作为补充GID分配给进程。

内置权限到GID的映射定义在 源码：frameworks\base\data\etc\platform.xml (机子：/system/etc/permissions/platform.xml）里。

```
    <permission name="android.permission.NET_TUNNELING" >
        <group gid="vpn" />
    </permission>

    <permission name="android.permission.INTERNET" >
        <group gid="inet" />
    </permission>

    <permission name="android.permission.READ_LOGS" >
        <group gid="log" />
    </permission>

    <permission name="android.permission.WRITE_MEDIA_STORAGE" >
        <group gid="media_rw" />
        <group gid="sdcard_rw" />
    </permission>

    <permission name="android.permission.ACCESS_MTP" >
        <group gid="mtp" />
    </permission>


    <!-- ================================================================== -->
    <!-- ================================================================== -->
    <!-- ================================================================== -->

    <!-- The following tags are assigning high-level permissions to specific
         user IDs.  These are used to allow specific core system users to
         perform the given operations with the higher-level framework.  For
         example, we give a wide variety of permissions to the shell user
         since that is the user the adb shell runs under and developers and
         others should have a fairly open environment in which to
         interact with the system. -->

    <assign-permission name="android.permission.MODIFY_AUDIO_SETTINGS" uid="media" />
    <assign-permission name="android.permission.ACCESS_SURFACE_FLINGER" uid="media" />
    <assign-permission name="android.permission.WAKE_LOCK" uid="media" />
    <assign-permission name="android.permission.UPDATE_DEVICE_STATS" uid="media" />
    <assign-permission name="android.permission.UPDATE_APP_OPS_STATS" uid="media" />

    <assign-permission name="android.permission.ACCESS_SURFACE_FLINGER" uid="graphics" />

    <!-- This is a list of all the libraries available for application
         code to link against. -->

    <library name="android.test.runner"
            file="/system/framework/android.test.runner.jar" />
    <library name="javax.obex"
            file="/system/framework/javax.obex.jar" />
    <library name="org.apache.http.legacy"
            file="/system/framework/org.apache.http.legacy.jar" />

    <!-- These are the standard packages that are white-listed to always have internet
         access while in power save mode, even if they aren't in the foreground. -->
    <allow-in-power-save-except-idle package="com.android.providers.downloads" />
```
assign-permission 用于相反的目的。它用于给运行在指定的UID下，没有对应包文件的系统进程赋予更高层次的权限（比如给守护进程，一般是bin文件，是不能像应用程序那样申请权限，就可以在这边申请）。

这边的 group gid 就定义在上面我们介绍的 \system\core\include\private\android_filesystem_config.h 文件中。

包管理器在启动的时候会维护一个权限到GID的列表，当包管理器给一个包授权的时候，会检查权限是否有对应的GID。如果有则加入到补充GID列表中。这个补充GID列表会写在上面我们说的 /data/system/package.list 中，在最后一个字段就是这个列表。







# 使用命令
查看服务列表:service list
查看权限列表:pm list permissions,-f的话可以显示更加显示的信息
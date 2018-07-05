title: SELinux
author: 道墟
date: 2018-07-05 17:55:37
tags:
---
# SELinux模式
三种模式
- disabled：关闭状态，不加载任何策略，仅执行默认的DAC
- permissive：宽容模式，策略会被加载，对象的访问也会被检查，但是对访问的拒绝仅会做记录而不会实际执行
- enforcing：强制模式，安全策略会被加载并执行，违反策略的行为也会被记录

查看和修改模式的命令
getenforce:查看模式
setenforce 0:设置模式，但是设置的模式在重启后会被还原为默认的模式

# 安全上下文
安全上下文是由分号分割的四个域组成的字符串：
- 用户名：一般与一组或者一类用户相关，比如user_u代表特权用户，admin_u代表管理员
- 角色
- 类型
- 可选的MLS安全范围

下面我们来看下进程的安全上下文信息，执行命令 ps -Z

```
u:r:init:s0                    root      1     0     /init
u:r:kernel:s0                  root      2     0     kthreadd
u:r:kernel:s0                  root      3     2     ksoftirqd/0
u:r:kernel:s0                  root      5     2     kworker/0:0H
u:r:fingerprintd:s0            system    1369  1     /system/bin/fingerprintd
u:r:system_server:s0           system    1633  1362  system_server
u:r:sdcardd:s0                 media_rw  1724  1251  /system/bin/sdcard
u:r:platform_app:s0:c512,c768  u0_a15    1762  1362  com.android.systemui
u:r:kernel:s0                  root      1792  2     kworker/1:1H
u:r:sdcardd:s0                 media_rw  1799  1251  /system/bin/sdcard
u:r:untrusted_app:s0:c512,c768 u0_a16    1953  1362  com.google.android.googlequicksearchbox:interactor
u:r:untrusted_app:s0:c512,c768 u0_a36    1966  1362  com.android.inputmethod.latin
u:r:untrusted_app:s0:c512,c768 u0_a8     1986  1362  com.google.android.gms.persistent
u:r:radio:s0                   radio     2005  1362  com.android.phone
u:r:untrusted_app:s0:c512,c768 u0_a9     2019  1362  com.android.launcher3
u:r:untrusted_app:s0:c512,c768 u0_a8     2403  1362  com.google.android.gms
u:r:platform_app:s0:c512,c768  u0_a4     3056  1362  com.android.defcontainer
u:r:untrusted_app:s0:c512,c768 u0_a26    3168  1362  com.android.deskclock
u:r:untrusted_app:s0:c512,c768 u0_a16    3660  1362  com.google.android.googlequicksearchbox:search
u:r:platform_app:s0:c512,c768  u0_a52    4015  1362  com.android.messaging
u:r:kernel:s0                  root      6498  2     kworker/2:1
u:r:kernel:s0                  root      11069 2     kworker/1:0
u:r:untrusted_app:s0:c512,c768 u0_a46    11659 1362  com.svox.pico
u:r:untrusted_app:s0:c512,c768 u0_a8     13011 1362  com.google.process.gapps
u:r:untrusted_app:s0:c512,c768 u0_a8     13063 1362  com.google.android.gms.unstable
u:r:kernel:s0                  root      13770 2     kworker/0:2
u:r:kernel:s0                  root      13783 2     kworker/1:1
u:r:su:s0                      root      13791 1307  /system/bin/sh
u:r:untrusted_app:s0:c512,c768 u0_a22    13800 1362  com.google.android.calendar
u:r:untrusted_app:s0:c512,c768 u0_a1     13816 1362  com.android.providers.calendar
u:r:su:s0                      root      14046 1307  /system/bin/sh
u:r:untrusted_app:s0:c512,c768 u0_a58    14057 1362  com.newland.kotlindagger2
```
查看文件的上下文，执行命令 ls -Z
```
drwxr-xr-x root     root              u:object_r:system_file:s0 app
drwxr-xr-x root     shell             u:object_r:system_file:s0 bin
-rw-r--r-- root     root              u:object_r:system_file:s0 build.prop
drwxr-xr-x root     root              u:object_r:system_file:s0 etc
drwxr-xr-x root     root              u:object_r:system_file:s0 fonts
drwxr-xr-x root     root              u:object_r:system_file:s0 framework
drwxr-xr-x root     root              u:object_r:system_file:s0 lib
drwx------ root     root              u:object_r:system_file:s0 lost+found
drwxr-xr-x root     root              u:object_r:system_file:s0 media
drwxr-xr-x root     root              u:object_r:system_file:s0 priv-app
drwxr-xr-x root     root              u:object_r:system_file:s0 tts
drwxr-xr-x root     root              u:object_r:system_file:s0 usr
drwxr-xr-x root     shell             u:object_r:system_file:s0 vendor
drwxr-xr-x root     shell             u:object_r:system_file:s0 xbin
```

# 安全规则
内核中的安全服务器使用SELinux安全策略，实时拒绝或者允许某些进程对内核对象的访问。
策略文件是由一些策略源文件编译得到的二进制文件。
策略源文件是由专用的策略语言编写的，包含了：
- 声明:定义策略实体，比如类型、用户、角色
- 规则:允许或拒绝对于对象的访问；指定允许的类型转换、指明如何设定默认用户、角色和类型。

#声明
- 类型声明
- 属性声明
- 权限声明


#文件位置
- \external\sepolicy\file_contexts   		文件安全上下文
- \external\sepolicy\property_contexts 		系统属性安全上下文
- \external\sepolicy\seapp_contexts			用于生成应用进程和文件的安全上下文
- /system/etc/security/mac_permissions.xml	将应用签名证书映射成setinfo值


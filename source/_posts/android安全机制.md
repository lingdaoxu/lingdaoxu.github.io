title: android安全机制
author: 道墟
date: 2018-06-25 23:15:43
tags:
---
http://tbfungeek.github.io/2016/04/06/Android-%E5%88%9D%E6%AD%A5%E4%B9%8BAndroid-%E5%AE%89%E5%85%A8%E6%9C%BA%E5%88%B6/

https://blog.csdn.net/ekeuy/article/details/44598145


https://yq.aliyun.com/articles/3008?spm=5176.100240.searchblog.19.yMfiLd


涉及到的文件信息

机子里面的
/data/system/packages.list 可以查看应用的uid 和groups信息


frameworks\base\data\etc\platform.xml （机子上/system/etc/permissions/platform.xml）权限对应的groups信息

\system\core\include\private\android_filesystem_config.h groups 键值对信息


建个app 然后申请几个权限，然后去 刚才platform.xml 查询到 申请对应的gid 信息 然后去 android_filesystem_config.h对比下值 看看是否跟 packages.list 或者 proc/进程号/status的一致。
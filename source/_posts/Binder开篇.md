title: Binder 开篇
author: 道墟
tags: []
categories:
  - android6.0
date: 2018-06-05 23:59:00
---
# 一.概述
Binder是Android系统提供的一种IPC（进程间通信）机制。由于Android是基于Linux内核的，因此，除了Binder外，还存在其他的IPC机制，例如管道和socket等。Binder相对于其他IPC机制来说，就更加灵活和方便了。对于初学Android的朋友而言，最难却又最想掌握的恐怕就是Binder机制了，因为Android系统基本上可以看作是一个基于Binder通信的C/S架构。Binder就像网络一样，把系统各个部分连接在了一起，因此它是非常重要的。

# 二.Binder对象定义
Binder用到的对象很多，而且名字还挺像的，如果不先预先约定理清楚的话，在后续的文章中会容易看晕，这边我们先对文章内提到的各个对象先约定好称呼（非官方定义，仅在本系列文章中使用）。

- Binder实体对象：Binder实体对象就是Binder**服务的提供者**。它必须继承自BBinder类，所以也叫 BBinder 对象。
- Binder引用对象：Binder引用对象是Binder实体对象在**客户端的代表**，它的类型是BpBinder类，所以也叫 BpBinder 对象。
- Binder代理对象：代理对象也叫**接口对象**，它主要是为客户端上层应用提供接口服务，从IInterface类派生。它实现了Binder服务的函数接口，当然这只是一个空壳，里面其实是直接转调**Binder引用对象**的方法。一般应用程序是使用Binder代理对象的。代理对象和引用的对象的关系是多对1的。
- IBinder对象：BBinder 和 BpBinder 都是从 IBinder 类中继承的。所以可以用IBinder对象来统称它们两个。

# 三.Binder架构
Binder通信架构主要由四部分组成
# 3.1 Binder驱动

Binder驱动位于内核空间，是Binder架构的核心，通过文件系统的标准接口（open ioctl mmap等）向用户空间提供服务。应用层和Binder驱动的数据交换是通过ioctl进行（ioctl可以达到一次系统调用就完成用户系统和Binder驱动的双向数据交互）。

作用：提供Binder通信的通道、维护Binder对象的引用计数、转换传输中的Binder实体对象和引用对象、管理数据缓存区。
	
# 3.2 ServiceManager
ServiceManager是一个守护进程，它是在init进程中通过init.rc文件启动的（{% post_link Init进程 %} 2.1.10节）。
```
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
```
作用：提供Binder服务的查询，返回被查询服务的引用。

ServiceManager进程也是通过Binder框架来提供服务的。这样就会才生鸡生蛋，蛋孵鸡的悖论了，作为Binder服务的提供者，它也需要使用某种方式让使用者获取到它的引用对象，然后再其他情况下，这个操作是由ServiceManagaer来提供查询服务的，这就互相冲突了。这边Binder架构使用一个非常暴力简单的方式解决这个问题。因为Binder引用对象的核心数据是一个实体对象的引用号，它是在驱动内部分配的一个值。所以Binder框架强制规定0代表了ServiceManager。

为什么需要ServiceManager?

> 理论上我们可以把查询的步骤放在Binder架构的核心--Binder驱动中来直接处理，那为什么还需要通过ServiceManager进程来作为中转呢?
这就要提到android的安全机制了，在android系统中是不允许任意进程都能注册Binder服务的，虽然任意的进程都可以创建Binder服务，但是只有root和system用户可以不受限制的在ServiceManager中注册服务。当然普通用户ID的进程也是可以注册Binder服务的，但是在ServiceManager有张表会记录着可以注册Binder服务的进程的用户ID，以及进程能够注册的服务名称，ServiceManager是就是过这张表来控制普通进程的注册请求（5.0以后不再通过表来控制，而是使用SELinux来检查权限）。

# 3.3 服务端

Binder服务分为：实名服务和匿名服务。它们从开发到使用并没有什么区别，唯一的区别就是实名服务可以通过ServiceManager查询到。Android的实名服务都是系统提供，比如我们后续要学习的：AMS,PKMS，WMS等。而我们平时我们在普通应用里面创建Binder服务都是匿名服务。

如果按上面说的匿名服务无法通过ServiceManager查询到，那么使用者是通过什么方式获取到它的引用对象呢？其实还是通过Binder。

匿名服务经常使用的场景：服务进程回调客户进程中的函数。下面是整个使用过程：
> 客户端和服务端通过binder连接上后，客户端将本进程创建的**匿名服务的实体对象**作为参数传递到服务端，binder驱动会在中间**将实体对象转换成引用对象**，这样服务进程就得到了客户进程创建的Binder服务的引用对象，然后就可以回调进程中的binder服务的函数了。

# 3.3.1 组件Service和匿名Binder服务
# 3.4 客户端

# 四.Binder的层次

- 服务类和接口类：位于framework层，为应用程序提供各种各样的服务，例如AMS,PKMS,WMS等
- Binder核心类：Ibinder、BBinder、BpBinder,这是上层 服务类和接口类的基础
- IPCThreadState：和驱动交换
- 驱动层

二三层可以可以合称为libbinder层







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
- Binder代理对象：代理对象也叫接口对象，它主要是为客户端上层应用提供接口服务，从IInterface类派生。它实现了Binder服务的函数接口，当然这只是一个空壳，里面其实是直接转调**Binder引用对象**的方法。一般应用程序是使用Binder代理对象的。代理对象和引用的对象的关系是多对1的。
- IBinder对象：BBinder 和 BpBinder 都是从 IBinder 类中继承的。所以可以用IBinder对象来统称它们两个。

# 三.Binder
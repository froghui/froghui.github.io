---
layout: post
title:  "docker背后的存储之device mapper(一)：模块和框架"
date:   2016-01-21 20:33:25
categories: linux
tags: linux docker
---

我们知道，docker在centOS上背后使用的存储技术是device mapper。本文将从device mapper实现的细节来讨论device mapper在docker中是如何使用的。
#   1.模块
device mapper的实现是按照linux模块来做的，主要模块及其依赖如下。
![](/assets/2016-01-21-device-mapper/dm_modules.svg)

主要模块的功能：

*   dm_mod: 这是最底层的一个模块，包括dm_target的接口，dm, dm_table, dm_io以及dm_ioctl。
*   dm_bufio:   操作metadata 底层IO的接口。
*   dm_pesistent_data:  device mapper提供的一个metadata和data的持久层操作，包括dm_space_map完成block data的元数据管理，dm_btree完成btree的维护操作。
*   dm_thin_pool:   包括dm_thin和dm_thin_metadata，支持docker使用的thin pool和snapshot的功能。


device mapper对外暴露的API是通过暴露ioctl的达到的。具体来说,当使用dmsetup的command或者API时，由dm_mod模块的dm_ioctl对相应的API指令做出响应，dm_target汇聚了所有可能的target_type，其中docker使用的是dm_thin_pool module里面注册
的pool_target和thin_target。最重要的一个指令是dmsetup create创建mapped_device及其gendisk对象，这会在kernel中注册几个磁盘
如dm-0(pool), dm-1(docker 1), dm-2..。


# 2.框架及实现
linux block device主要通过暴露一个gendisk给系统，这个gendisk对象关联到一堆operations 函数，分别用来对该设备进行一些处理；另外通过产生bio request，最终bio request进入到和gendisk绑定的make_request_fn以及队列request_queue进行调度处理。其结构可以看下图：
![](/assets/2016-01-21-device-mapper/dm_framework.svg)


![](/assets/2016-01-21-device-mapper/dm_table.svg)

# 3.debug环境搭建
在src目录建立一个run.sh如下，然后可以对drirvers/md下的文件打一些log，通过dmesg或者tail -f /var/log/messages来观察输出。


```bash
#!/bin/bash

#compile the module
make M=drivers/md

#copy the ko file
sudo cp drivers/md/dm-mod.ko /lib/modules/2.6.32-431.29.2.el6.x86_64/kernel/drivers/md/dm-mod.ko
sudo cp drivers/md/persistent-data/dm-persistent-data.ko /lib/modules/2.6.32-431.29.2.el6.x86_64/kernel/drivers/md/persistent-data/dm-persistent-data.ko
sudo cp drivers/md/dm-thin-pool.ko /lib/modules/2.6.32-431.29.2.el6.x86_64/kernel/drivers/md/dm-thin-pool.ko

#generate dependency
sudo depmod -a

#need to stop docker first so that we can remove modules
sudo service docker stop

#reload into the mormory, we need to remove the leaf modules from memory first, then install them
#modprobe will handle the module dependencies for us
sudo modprobe -r dm_thin_pool
sudo modprobe -r dm_mirror
sudo modprobe libcrc32c
sudo modprobe dm_mirror
sudo modprobe dm_thin_pool

sleep 2
#sudo service docker start
```



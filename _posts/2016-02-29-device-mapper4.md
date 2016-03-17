---
layout: post
title:  "docker背后的存储之device mapper(四)：dm-thin讨论"
date:   2016-02-29 20:33:25
categories: linux
tags: linux docker
---

我们知道，docker在centOS上背后使用的存储技术是device mapper。本文将从device mapper实现的细节来讨论device mapper在docker中是如何使用的。

# 1.关于blkdiscard
device mapper实际上是通过虚拟出一个硬盘设备来完成所谓的data mapping的。在虚拟磁盘设备dm-X上做磁盘操作，需要同时考虑到申请新的block和清除对block的占用。

对于申请新的block，需要向data space bitmap index申请新的可用的数据块，然后加入到 two level data mapping和detail tree中，同时会在data space bitmap index 中记录使用的ref count。

对于清除对block的占用，是通过产生block discard request来删除某块物理磁盘空间，具体到device mapper thin的实现，这时候一方面需要从 two level data mapping的那个map tree中对virtual block的映射删除，另一方面将virtual block实际对应的data block的磁盘 ref count计数减1。这样改data block在ref count == 0的时候就可用放入池中重新使用了。

blkdiscard对用户空间有两种暴露方式：

* mount -o discard   
* ioctl discard (blkdiscard utils) 产生blockdiscard request

第一种方式由ext4文件系统自动的产生blkdiscard request。

```bash
mount -t ext4 -o nodiscard /dev/mapper/docker-xxx /tmp/xxx

man mount
discard/nodiscard
      Controls  whether  ext4 should issue discard/TRIM commands to  
      the underlying block device when blocks are freed.  This is     
      useful for SSD devices    
      and sparse/thinly-provisioned LUNs, but it is off by default     
      until sufficient testing has been done.  
```bash

第二种方式显示指定blkdiscard request。

```bash
NAME
       blkdiscard - discard sectors on a device

SYNOPSIS
       blkdiscard [-o offset] [-l length] [-s] [-v] device

DESCRIPTION
       blkdiscard  is  used  to  discard device sectors.  This is useful for solid-state drivers (SSDs) and thinly-provisioned storage.  Unlike fstrim(8) this
       command is used directly on the block device.

       By default, blkdiscard will discard all blocks on the device.  Options may be used to modify this behavior based on range or size, as explained  below.

       The device argument is the pathname of the block device.

       WARNING: All data in the discarded region on the device will be lost!
```

上述两种方式最终都会执行到process_prepared_discard。

```C
static void process_prepared_discard(struct dm_thin_new_mapping *m)
{
    int r;
    struct thin_c *tc = m->tc;

    r = dm_thin_remove_block(tc->td, m->virt_block);
    if (r)
        DMERR_LIMIT("dm_thin_remove_block() failed");

    process_prepared_discard_passdown(m);
}

//删除该virtual_block对应的physical data block，以及<virtual_block, physical_block>映射数据
static int __remove(struct dm_thin_device *td, dm_block_t block)
{
    int r;
    struct dm_pool_metadata *pmd = td->pmd;
    dm_block_t keys[2] = { td->id, block };

    r = dm_btree_remove(&pmd->info, pmd->root, keys, &pmd->root);
    if (r)
        return r;

    td->mapped_blocks--;
    td->changed = 1;

    return 0;
}
```

所以在container的虚拟磁盘设备未被删除时，对磁盘空间的申请和撤销是通过make_requst这条线完成的(gendisk的request_queue)。

docker提供了参数可以对上述两种情形进行配置：

* dm.mountopt是在container存活时，在container的文件系统中删除文件是否需要自动的blkdiscard，
如果设置为dm.mountopt=nodiscard，那么一个container的文件删除了，并不会通知device mapper有block需要回收。

* dm.blkdiscard是在销毁容器时通知device mapper将该容器占有的data space返回到pool中以便其他container可以使用。
设置成dm.blkdiscard=ture，会通过ioctl发起blkdiscard操作。否则不发起。


当最后删除虚拟磁盘涉笔dm-X时，通过btree的删除会做两件事情；
一是删除detail_tree的映射数据；
二是删除two level data mapping 这个btree的第二层，对dev的映射，也会把对应的dm-X映射到的data block的ref count计数减1，达到清除data mapping的目的。和前面gendisk的blkdiscard request相比，这里是对该dev的所有<virtual_block, physical_block>进行处理。而不是对单个<virtual_block, phycical_block>进行处理。

```C
    r = dm_btree_remove(&pmd->details_info, pmd->details_root,
                &key, &pmd->details_root);
    if (r)
        return r;

    r = dm_btree_remove(&pmd->tl_info, pmd->root, &key, &pmd->root);
    if (r)
        return r;
```

for each pointing data block:

```C
    dm_sm_dec_block(sm, b);
    sm->dec(sm, block)
```


几个主要操作的总结

*   container里申请空间(dd)   由文件系统代表从data space bitmap index中申请新的数据块，将<virtual_block, physical_block>映射关系保存
*   container里删除磁盘空间(rm)   仅在配置dm.mountopt=discard的情况下才创建bio request， 通知data space bitmap index数据块phycical_block删除(refcount--)，并删除data mapping<virtual_block, phycical_block>映射关系。
*   删除container(dmsetup message delete)   将该container映射的所有phycical_block从data space bitmap index删除(refcount--)，并且删除所有<virtual_block, phycical_block>映射关系。
*   如果配置了dm.blkdiscard=true在dmsetup message delete之前会先调用ioctl interface(FITRIM)产生bio request，通知删除该命令所指定的virtual_block对应的phycical_block的data space bitmap index(refcount--)，同时摘除<virtual_block,physical>映射关系。

#2.关于COPY-ON-WRITE
![](/assets/2016-01-21-device-mapper/thin_copy_on_write.svg)
这里的copy-on-write主要是指metadata数据，基于base device创建出来的snapshot device在刚刚建出来的时候，其mapping数据完全复用base device的那份。当有write请求落到任意一个设备上时(base deive或者snap child device)会引起copy-on-write，即分配新的metadata block数据，将data mapping数据在新的metadata block数据复制一份，同时在data bitmap index中对data block的计数ref count进行增加处理。这样相当于恰好有两个设备指向了同一个物理设备block，但是注意他们的metadata数据已经完全分开了。

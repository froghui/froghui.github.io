---
layout: post
title:  "docker背后的存储之device mapper(二)：dm-thin概述"
date:   2016-01-25 20:33:25
categories: linux
tags: linux docker
---

我们知道，docker在centOS上背后使用的存储技术是device mapper。本文将从device mapper实现的细节来讨论device mapper在docker中是如何使用的。
#   1.metadata disk block format 
使用device mapper时，需要指定一个metadata disk和一个data disk。前者主要用来存储元数据，对metadata disk和data disk做管理。具体的格式如下
![](/assets/2016-01-21-device-mapper/metadata_encoding.svg)

*   super block:超级块保存了整个metadata disk format的重要数据，其中data_space_map_root保存了对数据分区映射的重要数据(nr_blocks描述了整个数据分区的大小，nr_allocated描述了已分配的数据分区的大小，bitmap_root记录了对数据分区进行映射的bitmap root在元数据分区上的块号，ref_count_root记录了对数据分区进行映射的ref count root在元数据分区上的块号)；metadata_space_map_root保存了对元数据分区映射的重要数据(nr_blocks描述了整个元数据分区的大小，nr_allocated描述了已分配的元数据分区的大小，bitmap_root记录了对元数据分区进行映射的bitmap root在元数据分区上的块号，ref_count_root记录了对元数据分区进行映射的ref count root在元数据分区上的块号)；data mapping root记录了数据分区映射的btree的起点；

*   metadata space: 整个元数据分区按照4K大小分成block，每个block是否已经分配使用，使用的个数是多少等信息对于元数据分区的管理是非常重要的。metadata space这几个block数据就是对应这部分工作的。首先需要一个metadata space map root，这个root block对于需要管理的index bitmap block做索引管理。比如可以通过disk_index_entry[15]知道第16个index bitmap block位于metadata分区中的位置(blocknr)，该index还可以分配出多少个空闲的entry(这个entry其实对应的就是metadata block nr)。每个index bitmap 数据块中用两个bits来映射一个entry，这两个bits表示映射到的block对应的ref count。注意到ref count >2以后该entry表示的ref count进入ref count btree。ref count btree的key为该entry的block nr, value为实际的ref count。

*   data space: 和metadata space类似，不过由于data space没有大小限制，对index的查找是通过btree完成的，而不是通过metadata space里面的静态数组完成。data bitmap btree的key为后面的data index bitmap的index (1...)，而value为对应的落在metadata数据分区上的block nr。通过这一层的btree管理，可以快速的得到第N个data index bitmap的信息，进而可以读出整个data index bitmap。同metadata space一样，。每个index bitmap 数据块中用两个bits来映射一个entry，这两个bits表示映射到的block对应的ref count。注意到ref count >2以后该entry表示的ref count进入第二个btree，即ref count btree。ref count btree的key为该entry的block nr, value为实际的ref count。

*   data mapping: 这是一个两层的btree，第一层的key为device id, value为device root信息(block_nr)，这一层可以通过查询device id定位到device root这个block。第二层的key为virtual block nr，value为一个64bits值，其中40bits表示physical block nr, 24bits表示该physical block使用的时间。这一层可以通过查询bio request的virtual block，定位到具体的物理磁盘号。注意这里的物理磁盘块大小是API显示指定的，例如128表示的是64K。

*   data details mapping:   这个btree的key为virtual block nr，value为detail值，可以知道该virtual block的使用状况。


#   2.主要交互流程:

##  2.1 create pool
```bash
dmsetup create 
docker-8:1-1055230-pool 
--table 
"0 209715200   
thin-pool   
/dev/loop0 /dev/sda3
128 32768 1 "
```
初始化metadata device(/dev/loop0)以及data device(/dev/sda3)。其中最主要的是初始化pool，对metadata device做初始化分割。按照block_size=4K，依次分割成super_block，metadata space map, data space map等。做完划分以后，将数据写入到metadata device磁盘中。
注意到这里对metadata device的切割大小是device mapper根据inode(/dev/loop0)读出的磁盘大小来完成的；而对data device的分割大小是根据指定的len(209715200)来完成的。

来看代码实现，实际上上述调用会连续的往ioctl发三条task run，create table, load table以及resume。
![](/assets/2016-01-21-device-mapper/1_thin_pool_create.svg)

##  2.2 create base thin and activate
```bash
dmsetup message 
/dev/mapper/docker-8:1-1055230-pool
0
"create_thin 0”
```
![](/assets/2016-01-21-device-mapper/2_create_thin.svg)
在这个步骤会首先向metadata分区申请一个block，获得的block号为dev_root，
然后将<key,value>=<dev_id,dev_root>加入到data mapping btree的第一层中。
最后在transaction结束时将<key, disk_entry>=<dev_id, 对应的disk细节信息>加入到data device detail btree中。


```bash
dmsetup create 
docker-8:1-1055230-base
—addnodeoncreate 
--table "20971520 
thin 
/dev/mapper/docker-8:2-1055230-pool
0”
```
![](/assets/2016-01-21-device-mapper/3_activate_thin.svg)
dm在这一步会产生/dev/mapper/docker-8:1-1055230-base这个gendisk，该gendisk表示了pool大磁盘的一个分区
可以被mkfs, mount等。可以往该gendisk分区写文件。

随后，docker 在createBaseImage这一步中对base disk进行格式化
mkfs.ext4 -E  nodiscard,lazy_itable_init=0,lazy_journal_init=0 /dev/mapper/docker-8:1-1055230-base

注意2.1&2.2是docker在初始化的时候会完成的事情。后续的步骤在device mapper初始化以后不会再进行，而是从metadata分区读出元数据。

##  2.3 create snapshot and activate
```bash
dmsetup message 
/dev/mapper/docker-8:1-1055230-pool
0
"create_snap <childDeviceId> <baseDeviceId>"
```

![](/assets/2016-01-21-device-mapper/4_create_snapshot.svg)
将<key,value>=<childDevId,dev_root>加入到data mapping btree的第一层中。
dev_root为指向原baseDeviceId的所指向的dev_root。这样可以看出snapshot完成是指向同样的二层map tree，所以所有的映射数据是和basedeviceId完全一样的。

最后在transaction结束时将<key, disk_entry>=<dev_id, 对应的disk细节信息>加入到data device detail btree中

```bash
dmsetup create 
docker-8:1-1055230-<uuid>
—addnodeoncreate 
--table "20971520 
thin 
/dev/mapper/docker-8:2-1055230-pool
<childDeviceId>”
```

![](/assets/2016-01-21-device-mapper/5_activate_snapshot.svg)

dm在这一步会产生/dev/mapper/docker-8:1-1055230-#{uuid} 这个gendisk，该gendisk表示了pool大磁盘的一个分区
可以被mount，mount以后可以往该gendisk分区写文件。一旦有些文件就会引起Copy-on-Write过程，新的childDeviceId的第二层mapping tree会和原来的baseDeviceId的那个mapping tree完全独立开来，只不过对数据块的引用计数ref count需要+1。


##  2.4 deactivate device of the container

```bash
dmsetup remove  docker-8:1-1055230-<uuid>
```
![](/assets/2016-01-21-device-mapper/6_deactivate_snapshot.svg)

这个命令相当于告诉device mapper将/dev/mapper/docker-8:1-1055230-<uuid>这个gendisk从系统中删除，但是对data block等的占用并没有杰出，也就是说随时可以从<devId>和tablesize等数据(docker将这份数据保存在/devicemapper/metadata文件夹下)重新产生一个gendisk并且可以被mount。

这个动作对应的device mapper driver的Put操作.umount文件系统并且调用该命令删除系统gendisk。当然，如果是删除一个container，第一步就是需要Put操作，然后再调用下一个命令，彻底的删除数据。

```go
// Put unmounts a device and removes it.
func (d *Driver) Put(id string) error {
    err := d.DeviceSet.UnmountDevice(id)
    if err != nil {
        logrus.Errorf("Error unmounting device %s: %s", id, err)
    }
    return err
}
```

##  2.5 delete device of the container
对应在删除一个container时，需要两步,第一步就是前面的deactive device。

第二步，是将该<childDeviceId>对应的space map所引用的每一个data block的ref count--，这样后续的新的docker container可以使用这些data space。另外删除整个childDeviceId在two level data mapping中的数据，以及device detail mapping数据。做完这一步，data block的数据会真正的删除。

```bash
dmsetup message 
/dev/mapper/docker-8:1-1055230-pool
0
delete <childDeviceId>”
```

![](/assets/2016-01-21-device-mapper/7_delete_thin.svg)

docker在删除device之前，会根据配置参数发起一个blkDiscard操作，目的是通知device mapper回收所有的data block。但是这一步并不是必须的，原因是在后面发起的message delete <childDeviceId>，device mapper的执行逻辑会将所有该device占有的data block做回收。

```go
// Should be called with devices.Lock() held.
func (devices *DeviceSet) deleteDevice(info *devInfo, syncDelete bool) error {
    if devices.doBlkDiscard {
        devices.issueDiscard(info)
    }

    // Try to deactivate device in case it is active.
    if err := devices.deactivateDevice(info); err != nil {
        logrus.Debugf("Error deactivating device: %s", err)
        return err
    }

    //在这里发起dmsetup message delete <childDeviceId>命令
    if err := devices.deleteTransaction(info, syncDelete); err != nil {
        return err
    }

    devices.markDeviceIDFree(info.DeviceID)

    return nil
}
```










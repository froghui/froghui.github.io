---
layout: post
title:  "Linux的硬盘分区 MBR和GPT"
date:   2015-07-07 20:33:25
categories: linux 
tags: linux
---

#	ms-dos格式和MBR
首先，我们知道硬盘分区是针对单块硬盘，不存在对N块硬盘一起分区的概念(类似对N块硬盘分区的概念在linux里面交给lvm等软件去做)
而其如果使用硬件Raid将几块硬盘联合起来在操作系统看来其实就是一块硬盘。

MBR是单块硬盘上的第一个扇区的数据，
按照MBR的格式，一个硬盘最多可以P+P+P+P四个主分区，或者P+P+P+E三个primary+一个extend分区，其中extend分区可以再分成很多的logic分区。老的linux对于logic分区的数目是有限制的(IDE 63, SCSI 15)，而新的linux系统已经对logic分区的数目没有限制了。

一个系统的启动是这样的(BIOS启动指定到某块硬盘)，
*	BIOS根据指定的启动顺序(如/dev/sda，/dev/sdb)，加载指定硬盘的MBR，从而读取MBR
*	加载MBR中的最基本的启动管理程序 此时BIOS就功成圆满，而接下来就是MBR内的启动管理程序的工作了。

	HDD 上的位置	代码的用意
	001-440 bytes	由 BIOS 启动的 MBR 启动代码
	441-446 bytes	MBR 硬盘签名
	447-510 bytes	分区表 (主分区和扩展分区，而非逻辑分区)
	511-512 bytes	MBR 启动签名 0xAA55.


MBR分区的一个例子

	Model: LSI LSI (scsi)
	Disk /dev/sda: 1797GB
	Sector size (logical/physical): 512B/4096B
	Partition Table: msdos

	Number  Start   End     Size    Type      File system     Flags
	 1      1049kB  538MB   537MB   primary   ext4            boot
	 2      538MB   108GB   107GB   primary   ext4
	 3      108GB   1788GB  1680GB  primary   ext4
	 4      1788GB  1797GB  8590MB  extended
	 5      1788GB  1797GB  8589MB  logical   linux-swap(v1)

#	GPT
由于旧的ms-dos格式的MBR分区最多只能支持2T的硬盘大小，为了克服这一点GPT格式被发明出来。和ms-dos格式相比，新的GPT格式不再区分primary和extend, 可以支持超过2T的硬盘。使用命令parted -l可以查看当前磁盘分区状况，其中比较常见的Partition Table有msdos(MBR), gpt, loop等。

	Model: HP LOGICAL VOLUME (scsi)
	Disk /dev/sda: 2400GB
	Sector size (logical/physical): 512B/512B
	Partition Table: gpt

	Number  Start   End     Size    File system     Name  Flags
	 1      1049kB  538MB   537MB   ext4                  boot
	 2      538MB   698GB   698GB   ext4
	 3      698GB   2392GB  1694GB  ext4
	 4      2392GB  2400GB  8589MB  linux-swap(v1)


	Model: Linux device-mapper (thin-pool) (dm)
	Disk /dev/mapper/docker-252:5-262892-pool: 107GB
	Sector size (logical/physical): 512B/512B
	Partition Table: loop

	Number  Start  End    Size   File system  Flags
	 1      0.00B  107GB  107GB  ext4


	Model: Virtio Block Device (virtblk)
	Disk /dev/vda: 64.4GB
	Sector size (logical/physical): 512B/512B
	Partition Table: msdos

	Number  Start   End     Size    Type      File system     Flags
	 1      1049kB  269MB   268MB   primary   ext4            boot
	 2      269MB   11.0GB  10.7GB  primary   ext4
	 3      11.0GB  21.7GB  10.7GB  primary   ext4
	 4      21.7GB  64.4GB  42.7GB  extended
	 5      21.7GB  32.5GB  10.7GB  logical   ext4
	 6      32.5GB  36.8GB  4295MB  logical   linux-swap(v1)
	 7      36.8GB  64.4GB  27.6GB  logical   ext4
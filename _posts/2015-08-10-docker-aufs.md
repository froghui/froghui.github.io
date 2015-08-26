---
layout: post
title:  "docker 存储之aufs"
date:   2015-08-10 11:33:25
categories: docker linux
---
概述
========
docker将网络，存储，监控以及cgroup/namespace和docker本身抽象隔离开来，主要的driver有这么几种:

1)execdriver  — lxc libcontianer 实现隔离(cgroup + namespace)

基于cgroup, namespace将image运行起来，隔离多个进程，这方面包括uts, process, memory，network等等。docker支持lib container和lxc两种方式实现。
2)graphdriver — 实现image和container的存储 (rootfs)   graph和aufs(device mapper)目录

3)metricdriver — docker的状态监控

4)networkdriver — docker vm的网络设置  （bridge+iptables) (bridge + mac switch) (ipvlan)



Docker image是一个抽象的概念，表示的是一个容器运行的rootfs环境。非常类型Cloud foundry里面的buildpack概念。
Dockerfile是docker的一种语法，用来build生成镜像。
Docker registry是image的管理。
docker image最为主要的概念是分层管理，Copy on Write，可以做到最小化image，而且可复用。
docker image分成两块，一块是metadata，另一块是image tar diff。
/var/lib/docker/graph/{imageId} : 每个imageID代表一个layer, 是每次dockerfile里面动作的一个结果，例如ADD, RUN等。主要有两个文件, json存储了该image的metadata，例如创建时间， parent image， cmd等等。layersize是一个描述当前image和parent image相比的diff size。

aufs的目录信息
具体的数据室存储在aufs目录的。
/var/lib/docker/aufs/diff/{imageId}:
/var/lib/docker/aufs/diff/{containerId}:
/var/lib/docker/aufs/diff/{containerId-init}:
存储每次该imageId指定的image相比其parent image的diff rootfs。例如ADD的一个文件, COPY的一个文件, RUN会改变的目录及文件等等。

/var/lib/docker/aufs/layers/{imageId}:
/var/lib/docker/aufs/layers/{containerId}:
/var/lib/docker/aufs/layers/{containerId-init}:
存储了当前image的parent image chain，便于快速构建image tree。
root@vagrant-ubuntu-trusty-64:/var/lib/docker/aufs/layers# cat 60c018693d4c850166810f7f8104fa1b310bbad355eb1d43cbca02a31a0db9a8
18392d3874fbb2fcbfa861a81d5b6ea32c81c3293e9c0cda62143b599f186c4a
05a707a5c88538f5695ce679e1b61331b43a24faf3a755b12ff7873fd4e62003
8a09e63f6f5cca6b2b88f2a50f6a5c5399002f8394bf6dd44243ebec7277e632
...

/var/lib/docker/aufs/mnt/{containerId}:
/var/lib/docker/aufs/mnt/{containerId-init}:
存储了aufs将layed image fs union以后形成的rootfs


下面结合数据结构研究一下几个docker命令的实现
docker history [IMAGE]
========
根据image的repository:tag得到其image id;
一种是可以根据image id找到其/var/lib/docker/graph/{imageId}目录，找到其json数据，从而知道其parent ID;
循环 parentID until last 
另一种是通过/var/lib/docker/aufs/layers/{imageId}找到其image tree.

这里tagStore 存储在/var/lib/docker/repositories-{graphDriver}
Repository map[string][]string
"ubuntu":
{"14.04":"d2a0ecffe6fa4ef3de9646a75cc629bbd9da7eead7f767cb810f9808d6b3ecb6",
"14.04.2":"d2a0ecffe6fa4ef3de9646a75cc629bbd9da7eead7f767cb810f9808d6b3ecb6",
"latest":"d2a0ecffe6fa4ef3de9646a75cc629bbd9da7eead7f767cb810f9808d6b3ecb6",
"trusty":"d2a0ecffe6fa4ef3de9646a75cc629bbd9da7eead7f767cb810f9808d6b3ecb6",
"trusty-20150630":"d2a0ecffe6fa4ef3de9646a75cc629bbd9da7eead7f767cb810f9808d6b3ecb6"}

docker create and run
======

首先，创建containerId-init的layer, diff。并mount到/mnt/{containerId}-init
/var/lib/docker/aufs/diff/{containerId}-init  （RW)
/var/lib/docker/aufs/diff/{baseImageId}   (RO)
...
————>
/var/lib/docker/aufs/mnt/{containerId}-init

等价于下面的mount命令
mount -t aufs
br:/var/lib/docker/aufs/diff/{containerId}-init=rw
:/var/lib/docker/aufs/diff/{baseImageId}=ro+wh,xino=/dev/shm/aufs.xino
:/var/lib/docker/aufs/diff/{parentofBaseImageId}=ro+wh
...
/var/lib/docker/aufs/mnt/{conatainer}-init

其次准备container top-level layed file
SetupInitLayer对/var/lib/docker/aufs/mnt/{conatainer}-init写入一些top-level文件，也就是写到/var/lib/docker/aufs/diff/{containerId}-init中, 这是为了起到保护作用。


最后创建containerId的layer, diff，并mount到/mnt/{containerId}
/var/lib/docker/aufs/diff/{containerId}   （RW)
/var/lib/docker/aufs/diff/{containerId}-init  （RO)
/var/lib/docker/aufs/diff/{baseImageId}   (RO)
...
————>
/var/lib/docker/aufs/mnt/{containerId}

例如, 一个container

{% highlight bash %}
7767c05d8bbf        docker.dp/tomcat/ngx4a-2:latest   "/bin/sh -c /bin/sta “
root@vagrant-ubuntu-trusty-64:/var/lib/docker/aufs/layers# cat 7767c05d8bbf2214ad94ccb887b723342d7b3f3e63f811fa53f5320b83fcf2b8-init (init image)
60c018693d4c850166810f7f8104fa1b310bbad355eb1d43cbca02a31a0db9a8 (base image)
18392d3874fbb2fcbfa861a81d5b6ea32c81c3293e9c0cda62143b599f186c4a

root@vagrant-ubuntu-trusty-64:/var/lib/docker/aufs/layers# cat 7767c05d8bbf2214ad94ccb887b723342d7b3f3e63f811fa53f5320b83fcf2b8
7767c05d8bbf2214ad94ccb887b723342d7b3f3e63f811fa53f5320b83fcf2b8-init
60c018693d4c850166810f7f8104fa1b310bbad355eb1d43cbca02a31a0db9a8
{% endhighlight %}

docker build
======
首先分析出dockerfile每步需要做的动作，对每一步都会创建临时的一个container
该container的rootfs是来自两部分
1)/var/lib/docker/aufs/diff/{containerId}-init(此为当前container运行的init image，ro: read only)，
注意/var/lib/docker/aufs/diff/{containerId}-init的parent image即为当前指定的base image(container从哪个image创建出来)。
2)/var/lib/docker/aufs/diff/{containerId}(rw: readwrite), 而mount的target为/var/lib/docker/aufs/mnt/{containerId}，这样对于该container而言，对roots的写全部落入了/var/lib/docker/aufs/diff/{containerId}中。这样就可以得到当前步骤相对于parent image的diff。

/var/lib/docker/aufs/diff/{containerId} (RW)
/var/lib/docker/aufs/diff/{containerId}-init}  （RO)
/var/lib/docker/aufs/diff/{baseImageId}   (RO)
...
————>
/var/lib/docker/aufs/mnt/{containerId}

在该container RUN完之后，立即创建一个新的image对象保存metadata，并写到/var/lib/docker/graph/{imageId}中,
接下来调用driver.Diff(container.ID, initID)获取该container相对于init image的diff数据(tar包)，并接着调用
driver.ApplyDiff(imageId, parentId, layerData archive.ArchiveReader)将数据写到 文件系统/var/lib/docker/aufs/diff/{imageId}中


aufs的作用
aufs 可以将每次对layed image所做的更改commit 成一个image（tar)，然后可以merge在一起。像这样变成一个rootfs
/var/lib/docker/aufs/diff/{containerId} (RW)
/var/lib/docker/aufs/diff/{containerId}-init}  （RO)
/var/lib/docker/aufs/diff/{baseImageId}   (RO)
...
————>
/var/lib/docker/aufs/mnt/{containerId}


---
layout: post
title:  "step by step build a mini linux container"
date:   2015-09-10 11:33:25
categories: docker 
tags: docker linux
---
# 概述
Linux container技术现在可谓是如火如荼，尤其以docker为代表(更早一点的有lxc, warden等)。虽然linux container是一种进程级别的技术，但由于其使用了linux kernel提供的资源隔离和限制的功能，可以“伪装”出一台小的"虚拟机"，并且这台小虚拟机占用资源少，“虚拟”速度快，因而被一致的认为是很有前途的超越虚拟机的技术。

要理解这门技术，最快的方法就是从头自己build 一个mini container。本文的目的就是教你如何一步一步的创造出自己的linux container。(
全部code在[source code of mini container](https://github.com/froghui/unicorn "unicorn"))

总体上，我们需要一个父进程负载创建linux container子进程，子进程内执行必要的程序。父进程负载管理linux container的生命周期。

# 父进程
这里的父进程负责

*   使用unicon fs技术准备rootfs，这样可以在子进程内使用pivot_root或者chroot方法将rootfs直接切换到准备好的rootfs中。
*   调用clone方法，准备namespace NEWIPC NEWUTS NEWPID NEWNS NEWNET.
*   在子进程创建之后设置cgroup
*   在子进程创建之后设置主机的network，同时通知子进程去配置自己namespace中的network.
*   等待子进程退出

````c
    //使用一对pipe和子进程通信   
    pipe(pipes);
    printf(" parent pid:: [%5d] \n",getpid());
    //unicorn id for the container
    char * unicorn_id = malloc(11);
    random_string(unicorn_id, 10);
    
    //net configuration
    net_t *n = calloc(1, sizeof(net_t));;
    assert(n != NULL);
    n->unicorn_id = unicorn_id;
    char * hostname = malloc(20);
    sprintf(hostname,"unicorn-%s",unicorn_id);
    n->hostname = hostname;
    n->mtu=1500;
    n->ip=ip;
    n->netmask=mask;
    n->gateway=gw;
    //prepare auts and then clone
    //这里使用了aufs准备rootfs目录，同时调用clone方法准备命名空间
    prepare_rootfs(mount_base, unicorn_id, rootfs_base);
    int child_pid = clone(child_main, child_stack+STACK_SIZE,  CLONE_NEWIPC | CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | SIGCHLD, n);
    
    //prepare unicorn info
    unicorn_t *u = calloc(1, sizeof(unicorn_t));;
    assert(u != NULL);
    u->child_pid = child_pid;
    u->parent_pid = getpid();
    u->unicorn_id = unicorn_id;
    //配置cgroup
    cgroup_add(u);
    //配置主机上的network
    net_add(u);
    printf(" child pid from parent:: [%5d] \n",child_pid);
    sleep(1);
    //call child to prepare the inner ethenet
    close(pipes[1]);
    //wait to die
    waitpid(child_pid, NULL, 0);
    return 0;
````

````c
    //prepare_rootfs主要使用aufs准备rootfs目录
    int prepare_rootfs(char * mount_base, char * unicorn_id, char * rootfs_base){  
    //use aufs to create union target rootfs
    char rootfs_path[100];
    check_result(sprintf(rootfs_path,"%s/%s",mount_base,unicorn_id), rootfs_path);
    check_mkdir(mkdir(rootfs_path, 0755), "mkdir root");
    check_result(sprintf(rootfs_path,"%s/rootfs",rootfs_path), rootfs_path);
    check_mkdir(mkdir(rootfs_path,0755), "mkdir rootfs");
    char mount_copy_on_write_dst[100];
    check_result(sprintf(mount_copy_on_write_dst,"%s/%s-init",mount_base,unicorn_id),mount_copy_on_write_dst);
    check_mkdir(mkdir(mount_copy_on_write_dst,0755),"mkdir copy_on_write dir");

    // mount -n -t aufs -o br:$mount_copy_on_write_dst=rw:$rootfs_base=ro+wh none rootfs_path
    char mount_data[100];
    check_result(sprintf(mount_data,"br:%s=rw:%s=ro+wh",mount_copy_on_write_dst, rootfs_base), mount_data);
    printf("rootfs_path:: %s mount_opt:: %s \n", rootfs_path, mount_data);
    check_result(mount("none",rootfs_path,"aufs",0, mount_data),"mount rootfs_base to rootfs dir");

    return 0;
````

````c
//一旦子进程创建，根据子进程的pid,创建对应的cgroup，并将其加入。这样该进程以及该进程产生的新进程都会受到资源控制
char * const cgroup_base_dir="/tmp/cgroup";
char* cgroup_subsystems[] = {"cpu", "cpuset", "cpuacct", "memory", "blkio"};
int cgroup_subsystems_lengh = 5;

int cgroup_add(unicorn_t * u){
    int i;
    for (i=0; i< cgroup_subsystems_lengh; i++){
        char system_path[80];
        check_result(sprintf(system_path, "%s/%s", cgroup_base_dir, cgroup_subsystems[i]),system_path);

        char cmd[100];
        check_result(sprintf(cmd, "mkdir -p %s/%s", system_path, u->unicorn_id),cmd);
        check_result(system(cmd), "system call mkdir ");

        if( strcmp(cgroup_subsystems[i], "cpuset") == 0){
            check_result(sprintf(cmd, "cat %s/cpuset.mems > %s/%s/cpuset.mems", system_path, system_path, u->unicorn_id),cmd);
            check_result(system(cmd), "system call add cpuset.mems ");

            check_result(sprintf(cmd, "cat %s/cpuset.cpus > %s/%s/cpuset.cpus", system_path, system_path, u->unicorn_id),cmd);
            check_result(system(cmd), "system call add cpuset.cpus ");

        }
        check_result(sprintf(cmd, "echo %d > %s/%s/tasks", u->child_pid, system_path, u->unicorn_id),cmd);
        check_result(system(cmd), "system call add pid ");
    }

    return 0;
}
````

````c
//网络方面的配置，使用了虚拟网卡对和bridge
int net_add(unicorn_t * u){
    //ip link add name u-${unicorn-id}-0 type veth peer name u-${unicorn-id}-1
    //ip link set u-${unicorn-id}-0 netns 1
    //ip link set u-${unicorn-id}-1 netns $PID
    //brctl addif ${bridge} u-${unicorn-id}-0   
    char cmd[100];
    check_result(sprintf(cmd, "ip link add name u-%s-0 type veth peer name u-%s-1", u->unicorn_id, u->unicorn_id),cmd);
    check_result(system(cmd), "system call ip link add ");

    check_result(sprintf(cmd, "ip link set u-%s-0 netns 1", u->unicorn_id),cmd);
    check_result(system(cmd), "system call ip link set ");


    check_result(sprintf(cmd, "ip link set u-%s-1 netns %d", u->unicorn_id, u->child_pid),cmd);
    check_result(system(cmd), "system call ip link set ");

    check_result(sprintf(cmd, "brctl addif br0 u-%s-0", u->unicorn_id),cmd);
    check_result(system(cmd), "system call addif br0 ");

    check_result(sprintf(cmd, "ifconfig u-%s-0 up", u->unicorn_id),cmd);
    check_result(system(cmd), "system call inconfig up ");


    return 0;
    
}
````


#   子进程
子进程所做的事情要简单一些，主要有：

*   使用pivot_root或者choot切换到预先准备好的rootfs
*   mount其他需要的文件到rootfs，例如volums, /etc/hostname, /etc/hosts, /etc/reslov.conf，伪文件系统/sys,/proc, /dev/pts, /dev/shm等。
*   配置网络，包括hostname以及interfaces(虚拟网卡对的另一端)。
*   调用exec系统调用运行对应的入口程序如bash，start.sh等。(对应docker中的CMD或者ENTRYPOINT)

````C
char* const child_args[] = {
    "/bin/bash",
    NULL
};

int child_main(void* arg)
{
    net_t * n = (net_t *)arg;
    //printf("in child main\n");
    char c;
    close(pipes[1]);   
    printf(" child pid from cloned child process:: [%5d] unicorn_id:: [%s] \n", getpid(), n->unicorn_id);

    pivot_move(mount_base, n->unicorn_id);
        
    //wait for the net peer to be setup
    read(pipes[0],&c,1);
    sethostname(n->hostname, 20);
    net_conf(n);

    check_result(execv(child_args[0], child_args), "execv");
    printf("return from chinld_main \n");
    return 1;
}

//这是子进程内最重要的一步，使用pivot_root切换rootfs目录，然后mount其他需要的目录，
//例如volums, /etc/hostname, /etc/hosts, /etc/reslov.conf，伪文件系统/sys,/proc, /dev/pts, /dev/shm等。
int pivot_move(char * mount_base, char * unicorn_id){
    char rootfs_path[100];
    check_result(sprintf(rootfs_path,"%s/%s/rootfs",mount_base,unicorn_id), rootfs_path);
    check_mkdir(mkdir(rootfs_path, 0755), "mkdir root");


    char pivot_old_dir[120];
    check_result(sprintf(pivot_old_dir,"%s/%s",rootfs_path,".pivot_old"),pivot_old_dir);
    check_mkdir(mkdir(pivot_old_dir,0755),"mkdir .pivot_old");
    check_result(chdir(rootfs_path),"chdir to rootfs_path");
    //printf("rootfs::%s pivot_old::%s \n", rootfs_path, pivot_old_dir);
    check_result(pivot_root(".", ".pivot_old"),"pivot_root");  
    check_result(chdir("/"),"chdir to /");
    check_result(umount2("/.pivot_old",MNT_DETACH), "umount /.pivot_old");
    check_result(rmdir(".pivot_old"),"remove .pivot_old");

    // mkdir -p /dev/pts
    //mount -t devpts -o newinstance,ptmxmode=0666 devpts /dev/pts
    //ln -sf pts/ptmx /dev/ptmx 
    check_mkdir(mkdir("/dev",0755),"mkdir /dev");
    check_mkdir(mkdir("/dev/pts",0755),"mkdir /dev/pts");
    check_result(mount("devpts","/dev/pts","devpts",0, "newinstance,ptmxmode=0666"),"mount /dev/pts");
    check_result(symlink("pts/ptmx","/dev/ptmx"),"ln -sf pts/ptmx /dev/ptmx ");

    //mkdir -p /dev/shm
    //mount -t tmpfs tmpfs /dev/shm
    //system("mkdir -p /dev/shm");
    check_mkdir(mkdir("/dev/shm",0755),"mkdir /dev/shm");
    check_result(mount("tmpfs","/dev/shm","tmpfs",0, NULL),"mount /dev/shm");

    //for top tool: mount -t proc proc /proc
    check_mkdir(mkdir("/proc",0755),"mkdir /proc");
    check_result(mount("proc","/proc","proc",0, NULL),"mount /proc");

    //for device data: mount -t proc proc /proc
    check_mkdir(mkdir("/sys",0755),"mkdir /sys");
    check_result(mount("sysfs","/sys","sysfs",0, NULL),"mount /sys");

    //export PS1="root@@\h:\w\$"

    return 0;
}

//子进程网络命名空间需要配置网卡
int net_conf(net_t * n){
    //ifconfig u-${unicorn-id}-1 $ip netmask $netmask mtu $mtu
    char cmd[100];
    check_result(sprintf(cmd, "ifconfig u-%s-1 %s netmask %s mtu %d", n->unicorn_id, n->ip, n->netmask, n->mtu),cmd);
    check_result(system(cmd), "system call ifconfig  ");
    
    return 0;
}
````


# 主要技术
总结一下，linux container主要用到了以下主要技术:

*   control group: 用来做资源限制，比如限制最大内存为4G，cpu使用为25%等(注意这里cpu的限制和完全公平的调度算法有关，换句话说，这里的cpu限制是控制整个contaier里面的进程们所占的所有cpu使用时间比率)
*   namespace: 用来做资源隔离，让container所在的所有进程只能感受到自己使用的资源，例如pid, network, uts, ipc, fs等等。
*   union fs: union fs的作用是将多层次的文件系统整合成一个单一的文件系统。
*   pivot_root或者chroot方法: 我们知道linux的rootfs是可以灵活挂载的，对于linux container而言，让其在自己独立的一个rootfs内，用到的技术正是pivot_root或者chroot。

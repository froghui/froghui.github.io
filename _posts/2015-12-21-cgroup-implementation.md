---
layout: post
title:  "Linux内核的cgroup实现(一)：Overview"
date:   2015-12-21 20:33:25
categories: linux
tags: linux docker
---

我们知道，docker背后实现的技术主要是linux namespace和linux cgroup(control group)，本文就从内核的角度来分析一下linux cgroup究竟是如何实现的。本文研读的代码是linux kernel 2.6.32。

#	1.数据结构:
![](/assets/2015-12-21-cgroup-implementation/cgroup-implement.svg)

##	1.1 task_struct
每个进程的数据结构，通过css_set可以描述当前的进程和哪些cgroup关联，以及关联的子系统有哪些。对每个进程而言，每种类型的子系统最多可以关联一个cgroup。

##	1.2 css_set
css_set和一个或几个task相关联，描述一个任务属于哪些cgroup。通过该group内的task fork出来的child task也必然在该css_set内。如果某个task被放到/sys/cgroup/cpuset /sys/cgroup/mememry两个cgroup中，就使用一个css_set将这两个cgroup和css_set关联。
从kernel角度出发，一个task一定有一个css_set与之相关联。css_set是task和cgroup之间的桥梁。css_set可以有一个或多个子系统注册。

##	1.3 cg_cgroup_link
用来将css_set和cgroup做关联。

##	1.4 cgroup: 
一个cgroup可以有一个或多个子系统注册，例如一个子系统cpu注册的cgroup:
/cgroup/cpu/docker/{uuid}/    
同时cpu, mememory注册的cgroup   
/cgroup/cpu_and_mem/{uuid}/    

每个cgroup对应一个mount到的cgroup文件系统，一个cgroup可以有多个子系统注册，多个子系统注册只需要在指定mount -t的时候带不同的子系统配置。

/cgroup/cpu (cpu)    
/cgroup2/cpu (cpu)    
/cgroup2/cpu_and_memory (cpu,memory)    
/cgroup/cpu/docker (cpu)    

在cgroup下通过mkdir新建的child group 如/cgroup/cpu/docker/{uid}也会创建新的cgroup。

##	1.5 cgroup_subsys_state
每个子系统的每次mount 都有一个cgroup_subsys_state与之对应，例如cpu和memory的cgroup_subsys_state

/cgroup/cpu (cpu state)    
/cgroup/memory (memory state)    
/cgroup2/cpu (cpu state)    
/cgroup2/cpu_and_memery (cpu state)    
/cgroup2/cpu_and_memery (memory state)     

##	1.6 cgroup_subsys
内核可以支持的子系统。每个子系统必须附属于一个cgroupfs_root对象，一个唯一的subsys_id可以标示子系统。子系统提供相应的函数实现，必要的时候需要在内核中做适当的处理(例如cpuset需要在调度算法里按指定的cpu进行调度)

##	1.7 cgroupfs_root
cgroup的根对象，该对象在系统中mount完产生，其数目是有限的。比如 /cgroup/cpu, /cgroup/memory等2个cgroupfs_root。当然如果/cgroup/cpu_and_memery只有一个cgroupfs_root。通过cgroupfs_root可以关联到cgroup_subsys子系统，例如/cgroup/cpu关联到cpu的cgroup_subsys，而/cgroup/cpu_and_memery关联到cpu和memory两个cgroup_subsys。

```C
struct cpuset {
    struct cgroup_subsys_state css;

    unsigned long flags;        /* "unsigned long" so bitops work */
    cpumask_var_t cpus_allowed;    /* CPUs allowed to tasks in cpuset */
    nodemask_t mems_allowed;    /* Memory Nodes allowed to tasks */

    struct cpuset *parent;        /* my parent */

    struct fmeter fmeter;        /* memory_pressure filter */

    /* partition number for rebuild_sched_domains() */
    int pn;

    /* for custom sched domain */
    int relax_domain_level;

    /* used for walking a cpuset heirarchy */
    struct list_head stack_list;
};
```


# 	2.用户case
##	2.1初始化
向系统注册 cgroup类型的文件系统

````C
//该函数初始化cgroup的树状结构，主要是init_css_set, rootnode, dummpytop等。
int __init cgroup_init_early(void)
{
	int i;
	atomic_set(&init_css_set.refcount, 1);
	INIT_LIST_HEAD(&init_css_set.cg_links);
	INIT_LIST_HEAD(&init_css_set.tasks);
	INIT_HLIST_NODE(&init_css_set.hlist);
	css_set_count = 1;
	init_cgroup_root(&rootnode);
	root_count = 1;
	init_task.cgroups = &init_css_set;

	init_css_set_link.cg = &init_css_set;
	init_css_set_link.cgrp = dummytop;
	list_add(&init_css_set_link.cgrp_link_list,
		 &rootnode.top_cgroup.css_sets);
	list_add(&init_css_set_link.cg_link_list,
		 &init_css_set.cg_links);

	for (i = 0; i < CSS_SET_TABLE_SIZE; i++)
		INIT_HLIST_HEAD(&css_set_table[i]);

	for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
		struct cgroup_subsys *ss = subsys[i];

		BUG_ON(!ss->name);
		BUG_ON(strlen(ss->name) > MAX_CGROUP_TYPE_NAMELEN);
		BUG_ON(!ss->create);
		BUG_ON(!ss->destroy);
		if (ss->subsys_id != i) {
			printk(KERN_ERR "cgroup: Subsys %s id == %d\n",
			       ss->name, ss->subsys_id);
			BUG();
		}

		if (ss->early_init)
			cgroup_init_subsys(ss);
	}
	return 0;
}
```


```C
/**
 * cgroup_init - cgroup initialization
 *
 * Register cgroup filesystem and /proc file, and initialize
 * any subsystems that didn't request early init.
 */
int __init cgroup_init(void)
{
	int err;
	int i;
	struct hlist_head *hhead;

	err = bdi_init(&cgroup_backing_dev_info);
	if (err)
		return err;

	for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
		struct cgroup_subsys *ss = subsys[i];
		if (!ss->early_init)
			cgroup_init_subsys(ss);
		if (ss->use_id)
			cgroup_subsys_init_idr(ss);
	}

	/* Add init_css_set to the hash table */
	hhead = css_set_hash(init_css_set.subsys);
	hlist_add_head(&init_css_set.hlist, hhead);
	BUG_ON(!init_root_id(&rootnode));
	err = register_filesystem(&cgroup_fs_type);
	if (err < 0)
		goto out;

	proc_create("cgroups", 0, NULL, &proc_cgroupstats_operations);

out:
	if (err)
		bdi_destroy(&cgroup_backing_dev_info);

	return err;
}
````

##	2.2 mount cgroup
如果在用户空间执行 mount -t cgroup -o cpu,cpuset,memory /sysgroup/cpu_and_mem
那么下面的函数将会被调用

````C
static int cgroup_get_sb(struct file_system_type *fs_type,
             int flags, const char *unused_dev_name,
             void *data, struct vfsmount *mnt)
````


该函数首先初始化一个cgroupfs_root对象，cgroupfs_root结构里面包括了一个cgroup对象，初始化该cgroup对象使其成为一个top cgroup。所谓top cgroup就是最顶层的cgroup对象，该cgroup的top_cgroup指向自己。

````C
static void init_cgroup_root(struct cgroupfs_root *root)
{
	struct cgroup *cgrp = &root->top_cgroup;
	INIT_LIST_HEAD(&root->subsys_list);
	INIT_LIST_HEAD(&root->root_list);
	root->number_of_cgroups = 1;
	cgrp->root = root;
	cgrp->top_cgroup = cgrp;
	init_cgroup_housekeeping(cgrp);
}
````


接下来根据指定的option对每个需要加载的subsys进行处理，将top cgroup里面的subsys指针初始化，并将对应的cgroup_subsys链接到cgroupfs_root中

````C
            cgrp->subsys[i] = dummytop->subsys[i];
            cgrp->subsys[i]->cgroup = cgrp;
            list_move(&ss->sibling, &root->subsys_list);
            ss->root = root;
````

之后，将该top group 加入到root列表中；
紧接着，因为每个任务对应一个css_set，将每个任务对应的css_set和当前的root_cgroup做关联，也就算将每个任务默认的加入到root cgroup中
最后，在root dir下对每个cgroup_subsys按照该subsystem的规则生成控制文件

在2.6 kernel中cpu, cpuset, memory等subsystem仅可以加入一个group，不允许某个subsystem加入到多个group中。
例如下面试图将cpu加到两个cgroup就会失败

mount -t cgroup -o cpu,memory cpu_and_mem /tmp/cpu_and_mem  
mount -t cgroup -o cpu cgroup /cgroup/cpu  

##	2.3.mkdir group1 under mounted dentry
对应的是在root cgroup下创建子cgroup，函数

````C
static int cgroup_mkdir(struct inode *dir, struct dentry *dentry, int mode)
````
负责处理该请求。

该函数首先对该cgroup下注册的的每个subsys创建cgroup_subsys_state对象，用来跟踪状态。cgroup_subsys_state对象是通过调用cgroup_subsys提供的函数create生成，
例如

````C
struct cgroup_subsys_state *css = ss->create(ss, cgrp);
````

接着会创建新的cgroup对象，并建立新的cgroup和parent cgroup, cgroupfs_root等直接的层级关系。
最后调用new indoor在当前目录下创建cgroup.event_control cgroup.procs notify_on_release tasks release_agent等控制文件，同时调用cgroup_subsys的populate方法创建子系统额外的控制文件

##	2.4. echo $PID > tasks
这个操作会往当前cgroup中加入对应的task，具体的做法是通过调用函数

````C
int cgroup_attach_task(struct cgroup *, struct task_struct *);
````

该函数首先对每个注册在当前cgroup下的cgroup_subsys，依次调用其提供的函数：

````C
ss->can_attach
ss->can_attach_task(cgrp)
ss->pre_attach(cgrp);
ss->attach_task(cgrp,
ss->attach(ss, cgrp, oldcgrp, tsk, false);
````

来完成cgroup_subsys的加入，完成这一步之后标志着对该task的cgroup控制在内核开始生效。
接着对task_struct中的css_set数据进行处理，将该task_struct和对应的cgroup和cgroup_subsys_state做关联。
注意默认该task在top cgroup(cgroup_root)中，所以必定需要做一次cgroup migration, 将该task从一个css_set移到新的css_set，并和当前新的cgroup关联。

##	2.5. fork process from cgroup
和2.4类似，也是两个主要目的，一个是完成task_struct的关联工作；另一个是完成cgroup_subsys的回调将该forked task加入到cgroup中完成控制。

这几个目的是在fork系统调用中通过调用下面的三个函数来完成的。
先看第一步，将该task加入到对应的css_set中。task_struct的css_set指针指向父进程的css_set并增加引用计数。

````C
void cgroup_fork(struct task_struct *child)
{
    task_lock(current);
    child->cgroups = current->cgroups;
    get_css_set(child->cgroups);
    task_unlock(current);
    INIT_LIST_HEAD(&child->cg_list);
}
````

第二步对每个cgroup_subsys调用fork操作，完成cgroup的控制。

````C
void cgroup_fork_callbacks(struct task_struct *child)
{
    if (need_forkexit_callback) {
        int i;
        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
            struct cgroup_subsys *ss = subsys[i];
            if (ss->fork)
                ss->fork(ss, child);
        }
    }
}
````

第三步将task_struct加入到css_set的链表中，以便从css_set可以遍历所管理的tasks。

````C
void cgroup_post_fork(struct task_struct *child)
{
    if (use_task_css_set_links) {
        write_lock(&css_set_lock);
        task_lock(child);
        if (list_empty(&child->cg_list))
            list_add(&child->cg_list, &child->cgroups->tasks);
        task_unlock(child);
        write_unlock(&css_set_lock);
    }
}
````

#	2.6. 具体实现
各个子系统在内核根据具体的代码逻辑有不同的实现，比如cpu和cpuset主要是在schedule过程中做相应的调度。这部分内容会在后续的系列文章中展开。

*	cpu: 按share调度cpu
*	cpuset: 限制某些cpu node可以使用
*	cpuacct: cpu使用统计
*	memory: 内存使用限制到bytes
*	blkio: 限制block device的读写速度。


##	2.7. umount /sys/cgroup/cpu_and_mem
static void cgroup_kill_sb(struct super_block *sb) {
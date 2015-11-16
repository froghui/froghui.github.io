---
layout: post
title:  "centos sshpass+ssh无法执行remote shell"
date:   2015-11-16 12:33:25
categories: bash 
tags: bash ssh
---

#	问题
今天在一台物理机上执行一段远程bash，发现没有任何响应和输出。代码形如

	sshpass -p password -p58422 root@10.101.10.19 "ifconfig"

该台机器的open-ssh和open-ssl版本如下

	ssh -v
	OpenSSH_5.3p1, OpenSSL 1.0.1e-fips 11 Feb 2013

但是很奇怪的是在另一台物理机上执行同样的bash能成功。而且最为吊诡的是换成另一个目标ip在出问题的物理机上又成功了。

	sshpass -p password -p58422 root@10.101.10.20 "ifconfig"

这是神马情况？考虑可能是open-ssh的bug，于是升级了open-ssh，然并卵！

最后发现，这是一个sshpass结合StrictHostKeyChecking出现的问题。sshpass就是一个简单的密码补齐工具，非常类似expect。
如果没有在/etc/ssh/ssh_config和~/.ssh/config等下面配置过StrictHostKeyChecking，然后使用ssh登陆，默认行为是这样的:

	ssh -p58422 root@10.101.20.19
	The authenticity of host '[10.101.20.19]:58422 ([10.101.20.19]:58422)' can't be established.
	RSA key fingerprint is 4c:60:31:94:0e:50:39:af:2b:7f:b2:1f:08:9b:73:31.
	Are you sure you want to continue connecting (yes/no)?

在这种情况下，捆绑sshpass和ssh使用，就会像石沉大海一样，没有任何反应。其实很好理解，sshpass希望是看到如下字符:

	root@10.101.20.19's password:
以便很高兴的把密码填上，登陆进入。然而，对于这样yes/no的回答，它就傻眼了。

#	解决方法
解决的方法很简单，不要让StrictHostKeyChecking出现交互式回答。有两种方式：

*	配置一下 /etc/ssh/ssh_config或者~/.ssh/config

```bash
	StrictHostKeyChecking no
```
*	在ssh 调用时显示指定选项

```bash
	sshpass -p password -p12345 -o StrictHostKeyChecking=no root@10.101.10.19 "ifconfig"
```

现在明白为什么换一台物理机执行没问题了把，答案是第二台机器在~/.ssh/config配置了StrictHostKeyChecking no

而诡异的换一个IP可以的原因是10.101.20.20已经把RSA 加到了~/.ssh/knownhosts中。

#	关于StrictHostKeyChecking
如果试着在~/.ssh/config配置了StrictHostKeyChecking yes，再使用sshpass+ssh，这个时候ssh就会先报错，然后我们就知道发生了什么。

	sshpass -p password ssh -p58422 root@10.101.20.19
	No RSA host key is known for [10.101.20.19]:58422 and you have requested strict checking.
	Host key verification failed.

所以，关于StrictHostKeyChecking 有至少三种模式

*	no 这种模式下显示warning，然后自动的加入到~/.ssh/knownhosts

```bash
Warning: Permanently added '[10.101.20.19]:58422' (RSA) to the list of known hosts.
```
*	yes 严格验证RSA
*	checking 需要用户交互式回答,这个是默认模式

显然，只有模式no能和sshpass一起配合使用。
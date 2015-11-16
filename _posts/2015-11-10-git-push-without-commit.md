---
layout: post
title:  "git中无commit的push"
date:   2015-11-10 12:33:25
categories: git 
tags: git
---

#	缘起
今天在看代码时，发现某位同事的某次代码commit，在主页面activities视图是可见的，但是到了Commits视图却怎么也看不到这个commit, git log也不能显示这个commit。后来猜测是这位同事发现该commit有问题，通过push -f的方式直接将这次commit从remote branch中取消了。这样看上去从历史活动页面有这个commit，但是从代码的Files/Commits视图都看不到这次commit。

# 	验证
为了证实我的想法，我试着按以下步骤做了一下

	git checkout -b testdev dev
	git push origin testdev    （@0197802400）

	echo "this is just a text" > test.txt
	git commit -m "this is just a test"
	git push origin testdev

	git reset --hard  0197802400
	git push -f origin testdev

证明确实如此，而且最后一次push -f的更新不会显示commit Id。
![](/assets/2015-11-10-git-push-without-commit/verify.png)
---
layout: post
category: git
date: 2016-11-10 03:38:55 UTC
title: git基础之回调(hooks)
tags: [git-receive-pack，回调，传输协议]
permalink: /git/basic/hooks
key: 3f12c79935cd5191995734c82742620a
description: git中的回调使用以及背后的实现
keywords: [git-receive-pack，回调，传输协议]
---

这个周一直在研究如何整合Jenkins和git，也就是通过git项目的提交来触发Jenkins的部署。Jenkins支持github项目的分支监控，但如果是git Server的话，分支的提交就不能被监控了。

不过在Jenkins的**部署触发模块(trigger build)**中有一种是通过远程来触发(Trigger builds remotely)的，也就是当远程仓库收到更新之后访问某个URL来触发Jenkins的部署，在Jenkins的控制后台也提到了这一点。

> One typical example for this feature would be to trigger new build from the source control system's hook script, when somebody has just committed a change into the repository, or from a script that parses your source control email notifications.

基于此简单的看了一下git回调的使用，现总结如下:

(1) 无论是裸库还是非裸库，都有一个hooks文件夹，里面提供各种不同阶段的回调。有本地的，有远程的。以Mac下的git为例(version 2.2.1):

```bash
-rwxr-xr-x  1 allen  staff   452 Mar  4  2016 applypatch-msg.sample
-rwxr-xr-x  1 allen  staff   896 Mar  4  2016 commit-msg.sample
-rwxr-xr-x  1 allen  staff   189 Mar  4  2016 post-update.sample
-rwxr-xr-x  1 allen  staff   398 Mar  4  2016 pre-applypatch.sample
-rwxr-xr-x  1 allen  staff  1642 Mar  4  2016 pre-commit.sample
-rwxr-xr-x  1 allen  staff  1356 Mar  4  2016 pre-push.sample
-rwxr-xr-x  1 allen  staff  4951 Mar  4  2016 pre-rebase.sample
-rwxr-xr-x  1 allen  staff  1239 Mar  4  2016 prepare-commit-msg.sample
-rwxr-xr-x  1 allen  staff  3611 Mar  4  2016 update.sample
```

默认都带了后缀**sample**, 使用的时候只要将后缀去掉即可。

(2) 以**update.sample**为例，里面提供关于tags，本地分支，远端分支改动的回调:

```bash
case "$refname", "$newrev_type" in
	refs/tags/*,commit)
        ...
		;;
	refs/tags/*,delete)
        ...
		;;
	refs/tags/*,tag)
        ...
		;;
	refs/heads/*,commit)
        ...
		;;
	refs/heads/*,delete)
        ...
		;;
	refs/remotes/*,commit)
		# tracking branch
		;;
	refs/remotes/*,delete)
        ...
		;;
	*)
		# Anything else (is there anything else?)
		echo "*** Update hook: unknown type of update to ref $refname of type $newrev_type" >&2
		exit 1
		;;
esac
```

上面**refname**指向的就是我们进行改动的分支，如**refs/heads/master**,  **refs/heads/feature**等。

结合上个例子提到的，我们可以在回调中访问Jenkins的某个地址来触发部署，因为作为共享代码库的一般是裸库，所以当开发人员将改动提交之后，对应的应该是裸库的<b style="color:red">本地分支的改动</b>

```bash
case "$refname","$newrev_type" in
	 refs/heads/*,commit)
		# branch
	    # 获取分支的简称，即去掉refs/heads/前缀 --> master
		BRANCH="$(git rev-parse --symbolic --abbrev-ref $refname)"
		
		# Jenkins build trigger url
		URL="http://xxxx/buildByToken/build?job=JOB_NAME/"${BRANCH}"&token=TOKEN"
		
		echo "${refname} receive some updates"
		curl -sS $URL
		;;
esac
```

由于hooks文件夹下的内容本质只是一堆脚本，所以我们可以通过修改<b style="color:red">脚本内容来添加自己所需要的回调</b>。

另外要注意虽然叫回调，<b class="highlight">但它们的执行时机却是发生在真正的改动执行之前</b>，它们会检查相应的更新如果符合要求，就进行反之则阻止。 就拿上面的update来说(如果是向远端分支提交新的改动)，实际上是以新的对象(ref)去替换老的对象，如果引进回调， 那么就要在回调执行成功之后才会触发真正的更新。

实际上每种回调都是由特定的服务触发的，比如说update就是由**git-receive-pack**来触发的，关于它的底层实现可以参考[git的 pack协议](https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt)，[git-receive-pack的运行机制](http://stackoverflow.com/questions/10662056/how-does-git-receive-pack-work)。


##参考

\> [git中的回调是什么](http://githooks.com/)

\> [关于git回调的介绍](https://git-scm.com/docs/githooks)


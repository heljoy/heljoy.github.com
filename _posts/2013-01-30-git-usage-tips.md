---
layout: post
title: "Git使用与学习记录"
description: "some tips about git in learning"
category: git
tags: [git, beginner, how-to]
---
{% include heljoy/setup %}

<p class="paragraph">
Git是一个很强大的版本管理系统，使用很复杂，但初学者只用一些简单的功能也能达到自己的要求，本身使用Git就是一个学习的过程，下面记录自己在这个学习过程中了解到很受用的小功能以备忘。
</p>

<!-- more --> 

*1. 分支管理相关*

{% highlight bash %}

# git branch					#查看本地分支
# git branch -r					#查看远程分支
# git branch <new_branch>			#基于当前分支创建一个新分支
# git checkout <new_branch>			#切换到新创建的分支上
# git checkout -b <branch> origin/<branch>	#创建一个远程分支的跟踪分支
# git branch -m <old_branch> <new_branch>	#重命名一个分支

{% endhighlight %}

<p class="paragraph">
分支中`master`是默认的主分支，可以不要这个分支，但最好不要这么做，因为大家都认为你有这个分支。Git中分支管理是最常用的操作之一，也很容易创建新分支，合并也很容易，不过这个我用得少一点。
</p>

*2. 内容缓存(git stash)*

<p class="paragraph">
缓存当前工作目录更改内容，执行这个命令后工作目录回到最后一次提交时的状态，所有未提交的更改得被隐藏起来了。
</p>

{% highlight bash %}

# git stash save              #缓存更改
# git stash apply             #将缓存的更改还原回来

{% endhighlight %}

<p class="paragraph">
我平时用得最多的就这两条，显然这个命令还有更强大的功能，我也是从help中了解到的。
</p>

{% highlight bash %}

# git stash save <message>           #缓存的时候加上备注，这个跟提交时相似
# git stash pop stash@{<revision>}   #有多次stash缓存时选择应用哪一个
# git stash list                     #查看当前所有缓存
# git stash clear                    #清除所有缓存内容
# git stash drop stash@{<revision>}  #清除某一次的缓存

{% endhighlight %}

<p class="paragraph">
不过这个命令还是少用的好，因为每个缓存都是针对某一环境的补丁，用多了关系就复杂了不好记，我经常用一次，然后切换到另一分支修改或查看内容，再切换回来apply上去。
</p>

*3. 远程仓库相关*

<p class="paragraph">
远程仓库是相对SVN一类工具而言，Git中也可以实现一个中心式的版本管理环境。
</p>

{% highlight bash %}

# git clone git@github.com:heljoy/heljoy.github.com.git	#检出本博客的源代码
# git remote add <new_repo> <repo_url>
# git pull origin <branch>				#拉取远程origin仓库，分支为跟踪分支
# git pull origin <remote_branch>:<local_branch>	#拉取远程分支到本地分支
# git remote set-url origin  <new_repo_url>		#更改远程仓库origin的服务器地址
# git push origin <local_branch>:<remote_branch>	#提交本地分支的更新到远程仓库分支
# git push origin :<remote_branch>			#删除远程仓库分支，这个没用过

{% endhighlight %}

<p class="paragraph">
我目前常用的就这几个，组合起来基本能完成我的要求。与`master`分支相似，`origin`是默认的远程仓库名，可以删除也可以更改这个仓库的服务器地址。同样本地的一个仓库可以有很多远程仓库，这样可以很方便将多个仓库的代码放到一个仓库中，之前一直没想到可以这样使用。
</p>

*4. GIT仓库页面浏览*

<p class="paragraph">
如果觉得命令行交互模式的方式不够直观，还可以通过网页的方式查看仓库的各种记录，类似kernel项目的网站，不过下面只是用于自己临时浏览，需要长期的网页服务还是daemon方式好些。
</p>

{% highlight bash %}
# git instaweb --httpd=webrick
{% endhighlight %}

<p class="paragraph">
使用浏览器访问提示的端口就能看到仓库页面了，使用Ctrl-C停止web服务。
</p>

*5. 相关教程链接 *

+  [写给Git初学者的7个建议] (http://linux.cn/thread/11840/1/1/)

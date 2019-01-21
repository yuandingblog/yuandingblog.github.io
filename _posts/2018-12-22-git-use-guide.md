---
layout:     post
title:      企业级Git使用及快速上手01
subtitle:   Git基础知识和核心概念介绍
date:       2018-12-22
author:     老麦
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Git
    - devops
    - GitHub
    - 分支
---
# 写在前面
对于习惯于通过shell、sqlplus来操控数据库的DBA来说，对GitHub的印象也仅限于它被称为`世界上最大的同性交友网站`，大概知道git是个版本管理工具，而GitHub作为全球最大的代码托管平台，程序员的最爱，上面有大量的开源程序和优秀资源。但版本管理工具多了，从vcs到svn，甚至直接重命名文件（文件夹）副本的方式来管理。。。一切的改变，也许就是从某次不经意注册了GitHub账号开始，然后search，fork，clone，还挺好玩...至此一发不可收拾。

> 版本控制是一种记录一个或若干文件内容变化，以便将来查阅特定版本修订情况的系统。在开发过程中，为了跟踪代码、文档、项目中的信息变化，版本控制变得前所未有的重要。

![img2019-01-16-pm3.23.13](http://img.yuandingit.com/img2019-01-16-pm3.23.13.png)


# Git诞生
> Git是一个分布式版本控制软件，最初由林纳斯·托瓦兹创作，于2005年以GPL发布。最初目的是为更好地管理Linux内核开发而设计。

上面这个来自wikipedia的关于git的介绍，看上去丝毫没有半分传奇性可言。但实际上，Git诞生的历史就像科比的81分，或者麦迪的35秒13分一样，完全就是软件史上的`另一个`传奇（上一个传奇是Linux，也是Linus干的）。关于Linus在2005年花了两个星期用C完成Git开发，并用它来代替商业软件BitKeeper管理Linux代码的故事，请移步[这里](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137402760310626208b4f695940a49e5348b689d095fc000)观看。

作为团队协作，版本控制，代码托管(gitlab, github等)的主流工具，git的使用已经逐渐融入我们的开发和日常工作，相比较SVN等其他工具，在存储性能、分支管理等方面有巨大的优势：
![img2019-01-16-pm4.15.43](http://img.yuandingit.com/img2019-01-16-pm4.15.43.png)


# Git两大阵营GitHub和GitLab
两者都提供基于web界面的Git repositories(仓库)，拥有用各自名称命名的开发协作流程，如github flow和gitlab flow等分支管理模型，给开发团队提供了存储、分享、发布、测试和合作开发项目中心化云存储场所。在很大程度上GitLab是仿造GitHub来做的，GitLab比较适合企业使用，可以让开发团队拥有更多的安全性和灵活性的选择，比如免费的私有仓库，内置CI/CD服务，支持私有部署等。以及，GitHub在2018年被微软收购后，已经和Google领投的GitLab展开了用户争夺战，相信这也是未来微软和Google两大生态阵营交锋的战场。
![img2019-01-16-pm3.25.53](http://img.yuandingit.com/img2019-01-16-pm3.25.53.png)

# Git常用命令
git的常用命令，大概分为以下几类（下图来自git cheat sheet）：
* repository创建相关
* 本地版本操作相关
* 分支管理相关
* 远程分支操作相关
* 合并（变基）
* 回滚（撤销操作）
![](http://img.yuandingit.com/15476249067929.jpg)

以下是git使用的一些经验和最佳实践，包括commit和branch等
![](http://img.yuandingit.com/15476249245581.jpg)


# Git工作流程
Git里面分为工作区域、暂存区和版本库，工作区（Working Directory）
就是你在电脑里能看到的目录，比如我的learngit文件夹就是一个工作区；
工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库；Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。Git的工作流程大致如下：
1. 在工作目录中修改某些文件
2. 使用 git add 文件名保存到暂存区域(staging area)
3. 使用git commit提交更新，将保存在暂存区域的文件快照永久转储到Git 仓库中(repository)
![](http://img.yuandingit.com/15476250763384.jpg)
# Git文件生命周期
![](http://img.yuandingit.com/15476246091363.jpg)

使用Git进行版本控制下的文件无非就两中状态：已被跟踪的（untracked），未被跟跟踪的(tracked)：

* 未被跟踪的：还未纳入版本控制,简单来说就是对文件还未使用过git commit命令的文件 
* 已被跟踪的：已经被纳入版本库控制,就是对文件使用过git commit文件名 命令的文件
* modified： 文件已经被修改, 仅仅是修改, 并没有进行其他的操作
* staged： 使用git add 文件名 命令可使文件进入暂存staged状
* unmodified：使用git commit 把staged状态(暂存区里面的文件)的文件提交到本地版本库中，此时文件状态为unmodified

一图胜千言，可以用git status -s来当前分支下的文件状态：
![](http://img.yuandingit.com/15476257253525.jpg)
上图Rakefile文件状态出现了两个M，左边的M表示文件被修改并放入暂存区，右边的M表示文件被修改还未放入暂存区。

# 远程仓库
远程仓库是指托管在internet或者其他网络的项目版本库，主要是为了团队协作，团队成员可以根据需要推送或者拉取数据。

```
$ git remote -v 查看远程仓库
upstream	git@michaellqu:michaellqu/bktest.git (fetch)
upstream	git@michaellqu:michaellqu/bktest.git (push)
$ git remote show origin 查看远程仓库origin的详细
* remote origin
  Fetch URL: https://github.com/michaellqq/java-demo.git
  Push  URL: https://github.com/michaellqq/java-demo.git
  HEAD branch: master
  Remote branch:
    master tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (fast-forwardable)
$ git remote add origin git@github.com:michaellqq/git-learn.git 添加远程仓库
$ git clone git@github.com:michaellqq/git-test.git  自动添加clone的地址为远程仓库，默认origin。默认情况下，git clone会自动设置本地master分支跟踪克隆的远程仓库的master分支
```
# 分支
几乎所有版本控制系统都以某种形式支持分支：使用分支意味着你可以把你的工作从开发主线上分离出来，以免影响开发主线。Git无论是创建还是切换分支，处理方式都是难以置信的轻量，Git鼓励工作流中频繁使用分支及合并。Git分支本质上就是指向提交对象的可变指针，会在每次commit后自动向前移动：

```
michaeldeMacBook-Pro:test-gitlab michael$ git branch -av
  iss01                  b46d797 Add test
* master                 6e110ad Add y
  remotes/origin/develop b46d797 Add test
  remotes/origin/gitlab  7803dc8 Add 2
  remotes/origin/master  6e110ad Add y
```

# Git对象
在Git系统中有四种类型的对象，几乎所有Git操作都是在这四种Git对象上进行的。这四种对象是：

* blob：blob对象通常用来存储文件的内容。一个blob对象就是一块二进制数据，它没有指向任何东西或有任何其它属性，甚至没有文件名。因为blob对象内容全部都是数据，所以如若两个文件在一个目录树或是一个版本仓库中有同样的数据内容，那么它们将会被指向同一blob对象。
* tree：可以简单理解为一个目录，管理一些tree对象或是blob对象。它有一串指向blob对象或是其它tree对象的指针，一般用来表示内容之间的目录层次关系(就像文件和子目录)。
* commit：commit对象指向一个tree对象，并且带有相关的描述信息，标记项目某一个特定时间点的状态。它包括一些关于时间点的元数据，如时间戳、最近一次提交的作者、指向上次提交的指针等等。
* tag：tag对象包括一个对象名(SHA1签名)、对象类型、标签名、标签创建人的名字, 还有一条可能包含有签名(signature)的消息。

![](http://img.yuandingit.com/15476273150050.jpg)

可以简单通过下图，理解一下tree、blob和commit对象之间的关系：
![](http://img.yuandingit.com/15476290356976.jpg) 


# 总结
作为团队协作，版本控制，代码托管(gitlab, github等)的主流工具，git的使用已经逐渐融入我们的开发和日常工作。随着云计算的发展，越来越多的公有云平台也都提供了自己的代码托管平台或服务，并且集成了CI/CD流水线，容器（甚至K8S）服务，让用户在公有云平台上实现完整的软件研发和发布。

本文对Git的基础知识和几个核心概念进行了简单的介绍，后面将逐渐通过系列文章来进一步介绍基于Git的协作，以及主流的代码分支管理策略。



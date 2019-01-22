---
layout:     post
title:      企业级Git使用及快速上手03
subtitle:   merge和rebase区别和应用场景
date:       2019-01-22
author:     老麦
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Git
    - devops
    - GitHub
    - rebase
    - merge
    - 分支
---
> 上篇文章中，我们简单了解了github的集成管理者工作流

# 前言
上一篇的场景中，小明通过分别添加小猛和小阳在github上的仓库为远程仓库，并用merge的方式合并了小猛和小阳完成的代码成果，最终把三个人的劳动成果推送到主仓库，整个协作过程如下图：
![img-2019-01-17-pm2.57.51](http://img.yuandingit.com/img-2019-01-17-pm2.57.51.png)
处于演示目的，我们假设几个人各自仅提交了一次合并请求，最终小明的master分支commit log history如下：

```
michaeldeMacBook-Pro:beat-taobao michael$ git log --oneline --all --graph
*   f770c78 (HEAD -> master, origin/master) Merge branch 'xybranch'
|\
| * 6e9dfdb (upstream-xiaoyang/master, xybranch) Order version 1.0
* |   3decafb Merge branch 'xmbranch'
|\ \
| * | fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
| |/
* | de413f1 Add member register.html
* | 84ed127 Add member index.html
|/
* 3825d0a Init project
```
可以看到经过多次merge，整个项目提交历史有多处分叉，如果在现实的协作过程中，master更加活跃（经过多次合并）的话，可能会有更多的分叉。那么，有没有什么操作可以代替merge呢？答案就是rebase。在比较两者之前，我们先深入了解下合并操作。

# 合并（merge）
首先假设你当前在项目master分支（c2），决定修复测试人员提出的53#bug，于是创建了一个分支iss53（c2），并向前推进到c3；此时接到一个紧急故障修复的请求，于是你切回master（c2），新建hotfix分支（c2），然后在hotfix分支上工作并解决了问题，即提交了c4。
* 快进（fast-forward）

假设你hotfix上的工作经过测试，解决了紧急问题，可以被合并回master分支，可以使用：

```
michaeldeMacBook-Pro:bktest michael$ git checkout master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 4 commits.
  (use "git push" to publish your local commits)
michaeldeMacBook-Pro:bktest michael$ git merge hotfix
Updating 6580a09..326e5aa
Fast-forward
 ss | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 ss
```
注意上面的“Fast-forward”，前提条件是其中一个分支没有新的commit发生。这种最简单的合并后，两个分支就拥有了完全相同的历史。
![](http://img.yuandingit.com/15481404564116.jpg)

* 三方合并

hotfix分支完成使命并合并回master后，即可被删除。假设你回到iss53上继续工作，往前推进到c5完成了53# bug的修复，完成测试后合并回master：

```
michaeldeMacBook-Pro:bktest michael$ git checkout master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 5 commits.
  (use "git push" to publish your local commits)
michaeldeMacBook-Pro:bktest michael$ git merge iss53
Merge made by the 'recursive' strategy.
 22r3    | 0
 dfdfdfd | 0
 2 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 22r3
 create mode 100644 dfdfdfd
```
注意上面的'recursive' ，表示为了完成合并，git创建了一个新的提交（c6）来包括两个分支所有劳动成果。
![](http://img.yuandingit.com/15481401608805.jpg)

总结：通常情况下，git commit都是有规范的，保证一次提交只涉及一个关联改动，比如实现某个功能或者修改了配置文件，并且能更好地注释这个提交。在上面的合并提交c6，它不是由开发人员手动创建的，而是由 Git 自动生成的。它也不涉及一个关联改动，这样的merge多少会污染你的分支历史，增加理解项目历史的难度。

# 变基（rebase）
假设我们在另一个项目中，遇到同上面情况一样的，开发任务分叉到两个不同分支，master分支为2adc861（假设对应图中c3），experiment为fd09ac5（假设对应图中c4）：

```
michaeldeMacBook-Pro:bktest michael$ git log --all --graph --oneline
* 2adc861 (HEAD -> master) Add 5.txt
| * fd09ac5 (experiment) Add 4.txt
|/
* db71035 Add 3.txt
* 681f3b0 Add 2.txt
* cac6b02 Initial commit
```
![](http://img.yuandingit.com/15481458870761.jpg)
一种方法是，使用上面讲的merge完成三方合并，合并结果新生成一个快照并提交ba15ce5（假设对应图中c5）。如下

```
michaeldeMacBook-Pro:bktest michael$ git merge experiment
Merge made by the 'recursive' strategy.
 4.txt | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 4.txt
michaeldeMacBook-Pro:bktest michael$ git log --all --graph --oneline
*   ba15ce5 (HEAD -> master) Merge branch 'experiment'
|\
| * fd09ac5 (experiment) Add 4.txt
* | 2adc861 Add 5.txt
|/
* db71035 Add 3.txt
* 681f3b0 Add 2.txt
* cac6b02 Initial commit
```
![](http://img.yuandingit.com/15481460158034.jpg)

除此之外，还有一种方法就是rebase，比如在experiment分支上，通过rebase master分支实现下图的合并效果。它的原理是先找到两个分支的共同祖先c2，然后把当前分支（experiment）上的提交c4，逐一应用到目标基底分支（master）c3之后，如下图：
![](http://img.yuandingit.com/15481461302044.jpg)
代码如下，我们可以看到和以上merge相比，主要区别是
“Add 4.txt”这次提交的的commit id变为由fd09ac5变为4c0daff，并加入到目标基底分支（master）2adc861之后，如下：
```
michaeldeMacBook-Pro:bktest michael$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: Add 4.txt
michaeldeMacBook-Pro:bktest michael$ git log --all --graph --oneline
* 4c0daff (HEAD -> experiment) Add 4.txt
* 2adc861 (master) Add 5.txt
* db71035 Add 3.txt
* 681f3b0 Add 2.txt
* cac6b02 Initial commit
```

在experiment上完成rebase后，可以回到master分支，进行一次fast-forward合并：
![](http://img.yuandingit.com/15481465079075.jpg)
代码如下：

```
michaeldeMacBook-Pro:bktest michael$ git checkout master
Switched to branch 'master'
michaeldeMacBook-Pro:bktest michael$ git merge experiment
Updating 2adc861..4c0daff
Fast-forward
 4.txt | 0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 4.txt
michaeldeMacBook-Pro:bktest michael$ git log --all --graph --oneline
* 4c0daff (HEAD -> master, experiment) Add 4.txt
* 2adc861 Add 5.txt
* db71035 Add 3.txt
* 681f3b0 Add 2.txt
* cac6b02 Initial commit
```
总结：经过rebase，它会把整个当前分支移动到目标基底分支的后面。rebase为当前分支上每一个提交创建一个新的提交，即重写了项目历史，并且不会带来合并提交。rebase最大的好处是让整个项目历史会非常整洁。首先，它不像 git merge 那样引入不必要的合并提交。其次，如上图所示，rebase导致最后的项目历史呈现出完美的一条线——你可以从项目终点到起点浏览而不需要任何的fork。

# rebase再现上篇场景
在上篇文章中，小明通过merge的方式合并了小猛和小阳的代码成果。现在我们让小明用rebase的方式来合并，并push到主仓库。然后再让小猛和小阳分别从主仓库拉取，rebase到自己的本地分支，为后面的进一步迭代做好准备。
1. 首先，小明回退本地master分支，回退到merge其余两人代码之前的一次提交（de413f1），这里使用git reset --hard的方式；
2. 然后，小明rebase方式合并小阳的代码，即xmbranch；
3. 小明rebase方式合并小猛的代码，即xybranch；
4. 小明推送rebase之后的代码到主仓库，这里使用git push -f，用带force的参数覆盖远程仓库；
5. 小猛用git pull --rebase方式合并主仓库代码；
6. 小阳用git pull --rebase方式合并主仓库代码；

小明回退到merge之前，自己本地master分支的快照de413f1
```
michaeldeMacBook-Pro:beat-taobao michael$ git reset --hard de413f1
HEAD is now at de413f1 Add member register.html
michaeldeMacBook-Pro:beat-taobao michael$ git branch -av
* master                           de413f1 [behind 4] Add member register.html
  xmbranch                         fd9e9c8 Shopping cart version 1.0
  xybranch                         6e9dfdb Order version 1.0
  remotes/origin/master            f770c78 Merge branch 'xybranch'
  remotes/upstream-xiaomeng/master fd9e9c8 Shopping cart version 1.0
  remotes/upstream-xiaoyang/master 6e9dfdb Order version 1.0
michaeldeMacBook-Pro:beat-taobao michael$ git log --oneline --all --graph
* de413f1 (HEAD -> master, origin/master) Add member register.html
* 84ed127 Add member index.html
| * 6e9dfdb (upstream-xiaoyang/master, xybranch) Order version 1.0
|/
| * fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
|/
* 3825d0a Init project
```

rebase小阳的代码，完成后可以发现“Add member index.html”对应的commit id已经变为8736191，“Add member register.html”对应的变为80f5b85 

> 正如以上对rebase的总结所说： 经过rebase，它会把整个当前分支移动到目标基底分支的后面。rebase为当前分支上每一个提交创建一个新的提交，即重写了项目历史，并且不会带来合并提交。

```
michaeldeMacBook-Pro:beat-taobao michael$ git rebase xybranch
First, rewinding head to replay your work on top of it...
Applying: Add member index.html
Applying: Add member register.html
michaeldeMacBook-Pro:beat-taobao michael$ git log --oneline --all --graph
* 80f5b85 (HEAD -> master) Add member register.html
* 8736191 Add member index.html
* 6e9dfdb (upstream-xiaoyang/master, xybranch) Order version 1.0
| * de413f1 (origin/master) Add member register.html
| * 84ed127 Add member index.html
|/
| * fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
|/
* 3825d0a Init project
```

rebase小猛的代码，可以看到“Order version 1.0”的commit id变为6c338f1，“Add member index.html”对应的commit id再次变成ba7ef56，“Add member register.html”对应的变为c46c7d1：
```
michaeldeMacBook-Pro:beat-taobao michael$ git rebase xmbranch
First, rewinding head to replay your work on top of it...
Applying: Order version 1.0
Applying: Add member index.html
Applying: Add member register.html
michaeldeMacBook-Pro:beat-taobao michael$ git log --oneline --all --graph
* c46c7d1 (HEAD -> master) Add member register.html
* ba7ef56 Add member index.html
* 6c338f1 Order version 1.0
* fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
| * de413f1 (origin/master) Add member register.html
| * 84ed127 Add member index.html
|/
| * 6e9dfdb (upstream-xiaoyang/master, xybranch) Order version 1.0
|/
* 3825d0a Init project
```
强制推送到主仓库：

```
michaeldeMacBook-Pro:beat-taobao michael$ git push -f
Enumerating objects: 25, done.
Counting objects: 100% (22/22), done.
Delta compression using up to 4 threads
Compressing objects: 100% (12/12), done.
Writing objects: 100% (19/19), 1.58 KiB | 1.58 MiB/s, done.
Total 19 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3), completed with 1 local object.
To michaellqu:michaellqu/beat-taobao.git
 + de413f1...c46c7d1 master -> master (forced update)
 # 提交历史变成一条直线
 michaeldeMacBook-Pro:beat-taobao michael$ git log --oneline --graph
* c46c7d1 (HEAD -> master, origin/master) Add member register.html
* ba7ef56 Add member index.html
* 6c338f1 Order version 1.0
* fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
* 3825d0a Init project
```
小猛同步主仓库代码到本地：

```
michaeldeMacBook-Pro:xm-beat-tb michael$ git remote add upstream-xiaoming git@michaellqu:michaellqu/beat-taobao.git

michaeldeMacBook-Pro:xm-beat-tb michael$ git pull --rebase upstream-xiaoming master
From michaellqu:michaellqu/beat-taobao
 * branch            master     -> FETCH_HEAD
Updating fd9e9c8..c46c7d1
Fast-forward
 member/index.html    | 2 ++
 member/register.html | 1 +
 order/README.md      | 1 +
 order/index.html     | 1 +
 4 files changed, 5 insertions(+)
 create mode 100644 member/index.html
 create mode 100644 member/register.html
 create mode 100644 order/index.html
Current branch master is up to date.

michaeldeMacBook-Pro:xm-beat-tb michael$ git log --oneline --graph
* c46c7d1 (HEAD -> master, upstream-xiaoming/master) Add member register.html
* ba7ef56 Add member index.html
* 6c338f1 Order version 1.0
* fd9e9c8 (origin/master, origin/HEAD) Shopping cart version 1.0
* 3825d0a Init project
```

小阳在同步主仓库代码到本地之前，又做了两次提交，看看有什么不同：

```
michaeldeMacBook-Pro:xy-beat-tb michael$ touch 1.txt
michaeldeMacBook-Pro:xy-beat-tb michael$ git add .
michaeldeMacBook-Pro:xy-beat-tb michael$ git commit -m"Add 1.txt"
[master 550e5e0] Add 1.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 1.txt
 
michaeldeMacBook-Pro:xy-beat-tb michael$ touch 2.txt
michaeldeMacBook-Pro:xy-beat-tb michael$ git add .
michaeldeMacBook-Pro:xy-beat-tb michael$ git commit -m"Add 2.txt"
[master cf5086f] Add 2.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 2.txt
 
michaeldeMacBook-Pro:xy-beat-tb michael$ git log --oneline --graph
* cf5086f (HEAD -> master) Add 2.txt
* 550e5e0 Add 1.txt
* 6e9dfdb (origin/master, origin/HEAD) Order version 1.0
* 3825d0a Init project

michaeldeMacBook-Pro:xy-beat-tb michael$ git pull --rebase upstream-xiaoming master
remote: Enumerating objects: 17, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 13 (delta 2), reused 13 (delta 2), pack-reused 0
Unpacking objects: 100% (13/13), done.
From michaellqu:michaellqu/beat-taobao
 * branch            master     -> FETCH_HEAD
 + f770c78...c46c7d1 master     -> upstream-xiaoming/master  (forced update)
First, rewinding head to replay your work on top of it...
Applying: Add 1.txt
Applying: Add 2.txt

michaeldeMacBook-Pro:xy-beat-tb michael$ git log --oneline --graph
* 000dfe8 (HEAD -> master) Add 2.txt
* 4bd7f57 Add 1.txt
* c46c7d1 (upstream-xiaoming/master) Add member register.html
* ba7ef56 Add member index.html
* 6c338f1 Order version 1.0
* fd9e9c8 Shopping cart version 1.0
* 3825d0a Init project
```
# 总结
本篇，我们详细对比了merge和rebase的区别，并用rebase方式代替上一篇的merge，重现了上一篇的代码集成场景。rebase还有很多应用场景，比如使用交互式rebase -i合并历史提交等，相对都比较简单，不再赘述。

最后，重点强调一条黄金准则：绝不要在公共分支上使用rebase ！！！
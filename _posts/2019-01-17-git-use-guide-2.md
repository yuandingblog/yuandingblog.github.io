---
layout:     post
title:      企业级Git使用及快速上手02
subtitle:   通过实际场景体验Git的灵活及强大
date:       2019-01-17
author:     老麦
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Git
    - devops
    - GitHub
    - 分支
---
# 场景描述
上篇文章中，简单介绍了Git的基础知识和核心概念，本篇会通过一个github协作场景，一步一步带大家熟悉Git的命令和使用，来体验Git的灵活及强大。首先看下需求：
小明所在的团队最近需要开发一个购物网站，网站大概分为会员、购物车和订单三个模块，小明负责项目整体维护，及会员模块，其他两个模块由位于不同办公地点的小猛和小阳分别负责，小明决定通过github组织和实现协作（注：实际项目可能使用企业内部的私有仓库）。整个协作过程主要包括：
1. 小明决定采用集成管理者工作流方式，在github上创建一个名为beat-taobao的repository作为主仓库；
2. 小猛和小阳分别克隆此仓库，并在本地建立开发分支develop，进行自己的模块开发；
3. 小阳首先完成积分模块，合并入自己本地master后，将代码推送到自己在github上的仓库；
4. 小阳发送邮件给小明，请求拉取自己的更新；
5. 小明添加小阳在github上的仓库为远程仓库，在本地合并其代码；
6. 小明将合并后的修改推送到主仓库；
7. 其他人提交的代码合并请求，小明重复执行5和6；

![img-2019-01-17-pm2.57.51](http://img.yuandingit.com/img-2019-01-17-pm2.57.51.png)

# 配置SSH keys
由于后面将会在一台电脑上模拟小明，小猛和小阳三个独立的github账号，所以现注册三个GitHub账号，并在本地为不同用户生成各自SSH密钥。一般来说，Git服务器为了安全，都会需要我们提供一个安全的SSH密钥，默认情况下，生成密钥的文件名都是一样的，但是不同的用户，必须设置不同文件名的密钥文件，否则会发生覆盖。

```
$ ssh-keygen -t rsa -C “xiaoming@beat-tb.com” 
使用默认名称即id_rsa.pub
$ ssh-keygen -t rsa -C “xiaomeng@beat-tb.com” 
使用qq.pub
$ ssh-keygen -t rsa -C “xiaoyang@beat-tb.com”
使用gmail.pub
```
```
michaeldeMacBook-Pro:.ssh michael$ ls
config  gmail  gmail.pub  id_rsa  id_rsa.pub  known_hosts  qq  qq.pub
```
密钥生成后，将不同用户的SSH keys添加到各自对应的Github账号（.pub内容）;
![img-2019-01-17-pm3.57.23](http://img.yuandingit.com/img-2019-01-17-pm3.57.23.png)

在生成密钥的.ssh目录下，新建一个config文件，配置多账户规范:

```
michaeldeMacBook-Pro:.ssh michael$ cat config
Host michaellqu
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa

Host michaellqq
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/qq
Host michaellgg
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gmail
```
# 小明创建公共仓库

1、用小明的github账号登陆，并建立一个空的repository；
![img-2019-01-17-am11.32.45](http://img.yuandingit.com/img-2019-01-17-am11.32.45.png)
该仓库将作为主仓库，地址为 

Github也会自动跳到Quick setup，并给仓库地址git@github.com:michaellqu/beat-taobao.git ，并给出几种情况：
```
…or create a new repository on the command line 初始化本地仓库，增加远程仓库并推送
echo "# beat-taobao" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin git@github.com:michaellqu/beat-taobao.git
git push -u origin master
```

```
…or push an existing repository from the command line 推送已存在的本地仓库
git remote add origin git@github.com:michaellqu/beat-taobao.git
git push -u origin master
```

2、小明作为项目管理者，进行了以下操作：
初始化本地repository，配置当前repository用户信息
```
michaeldeMacBook-Pro:projectdemo michael$ git init beat-taobao
Initialized empty Git repository in /Users/michael/projectdemo/beat-taobao/.git/
michaeldeMacBook-Pro:projectdemo michael$ cd beat-taobao
michaeldeMacBook-Pro:beat-taobao michael$ git config --local user.name "xiaoming"
michaeldeMacBook-Pro:beat-taobao michael$ git config --local user.email "xiaoming@beat-tb.com"
```

创建项目文件
```
michaeldeMacBook-Pro:beat-taobao michael$ mkdir shopping-cart member order
michaeldeMacBook-Pro:beat-taobao michael$ echo "# beat-taobao" >> README.md
michaeldeMacBook-Pro:beat-taobao michael$ echo "# This is index" >> index.html
michaeldeMacBook-Pro:beat-taobao michael$ touch member/README.md order/README.md shopping-cart/README.md
michaeldeMacBook-Pro:beat-taobao michael$ tree
.
├── README.md
├── index.html
├── member
│   └── README.md
├── order
│   └── README.md
└── shopping-cart
    └── README.md

3 directories, 5 files
```

此时新增的文件和目录结构还没加入暂存区，都是未跟踪状态，通过git status查看
```
michaeldeMacBook-Pro:beat-taobao michael$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	README.md
	index.html
	member/
	order/
	shopping-cart/

nothing added to commit but untracked files present (use "git add" to track)
```
加入暂存区，完成首次commit，通过git log查看commit历史，当前为3825d0a3
```
michaeldeMacBook-Pro:beat-taobao michael$ git add .
michaeldeMacBook-Pro:beat-taobao michael$ git commit -m"Init project"
[master (root-commit) 3825d0a] Init project
 5 files changed, 2 insertions(+)
 create mode 100644 README.md
 create mode 100644 index.html
 create mode 100644 member/README.md
 create mode 100644 order/README.md
 create mode 100644 shopping-cart/README.md
michaeldeMacBook-Pro:beat-taobao michael$ git log
commit 3825d0a38589c57f3b2ea5c6faa78ec250209f1b (HEAD -> master)
Author: xiaoming <xiaoming@beat-tb.com>
Date:   Thu Jan 17 12:03:29 2019 +0800

    Init project
```

添加远程仓库，并修改remote仓库的url为上面配置ssh key时在config里设置的别名。即修改.git/config中的remote仓库对应的，url = git@github.com:michaellqu/beat-taobao.git为url = git@michaellqu:michaellqu/beat-taobao.git 
```
michaeldeMacBook-Pro:beat-taobao michael$ git remote add origin git@github.com:michaellqu/beat-taobao.git
michaeldeMacBook-Pro:beat-taobao michael$ git remote -v
origin	git@github.com:michaellqu/beat-taobao.git (fetch)
origin	git@github.com:michaellqu/beat-taobao.git (push)

michaeldeMacBook-Pro:beat-taobao michael$ cat ~/.ssh/config
Host michaellqu
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa

Host michaellqq
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/qq
Host michaellgg
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gmail
    
michaeldeMacBook-Pro:beat-taobao michael$ vi .git/config
michaeldeMacBook-Pro:beat-taobao michael$ git remote -v
origin	git@michaellqu:michaellqu/beat-taobao.git (fetch)
origin	git@michaellqu:michaellqu/beat-taobao.git (push)

```

git push -u的方式推送master，并建立跟踪分支
```
michaeldeMacBook-Pro:beat-taobao michael$ git push -u origin master
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 4 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (6/6), 395 bytes | 395.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To michaellqu/beat-taobao.git
 * [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
michaeldeMacBook-Pro:beat-taobao michael$ git branch -av
* master                3825d0a Init project
  remotes/origin/master 3825d0a Init project
michaeldeMacBook-Pro:beat-taobao michael$ git branch -vv
* master 3825d0a [origin/master] Init project
```
push到远程仓库后，可以看到github上当前仓库的项目文件和结构
![img-2019-01-17-pm2.16.59](http://img.yuandingit.com/img-2019-01-17-pm2.16.59.png)

# 小猛协作过程
1. 小猛作为协作者，使用自己的github账号登陆，并fork主仓库 https://github.com/michaellqu/beat-taobao，并更改仓库名为xm-beat-tb
![img-2019-01-17-pm3.00.49](http://img.yuandingit.com/img-2019-01-17-pm3.00.49.png)

2. 小猛clone远程仓库到本地，在本地完成开发，完成后推送到自己在github的公开仓库，并通知小明：

克隆自己仓库，配置用户信息
```
michaeldeMacBook-Pro:projectdemo michael$ git clone git@github.com:michaellqq/xm-beat-tb.git
Cloning into 'xm-beat-tb'...
remote: Enumerating objects: 6, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 6 (delta 0), reused 6 (delta 0), pack-reused 0
Receiving objects: 100% (6/6), done.

michaeldeMacBook-Pro:projectdemo michael$ cd xm-beat-tb/
michaeldeMacBook-Pro:xm-beat-tb michael$ git remote -v
origin	git@github.com:michaellqq/xm-beat-tb.git (fetch)
origin	git@github.com:michaellqq/xm-beat-tb.git (push)

michaeldeMacBook-Pro:xm-beat-tb michael$ git config --local user.name "xiaomeng"
michaeldeMacBook-Pro:xm-beat-tb michael$ git config --local user.email "xiaomeng@beat-tb.com"
michaeldeMacBook-Pro:xm-beat-tb michael$ git branch -v
* master 3825d0a Init project
michaeldeMacBook-Pro:xm-beat-tb michael$ git log --oneline
3825d0a (HEAD -> master, origin/master, origin/HEAD) Init project
```
修改remote仓库的url为上面配置ssh key时在config里设置的别名。即修改.git/config中的remote仓库对应的，url = git@github.com:michaellqq/xm-beat-tb.git为url = git@michaellqq:michaellqq/xm-beat-tb.git


```
michaeldeMacBook-Pro:xm-beat-tb michael$ vi .git/config
michaeldeMacBook-Pro:xm-beat-tb michael$ git remote -v
origin	git@michaellqq:michaellqq/xm-beat-tb.git (fetch)
origin	git@michaellqq:michaellqq/xm-beat-tb.git (push)
```


开发购物车模块,完成后commit，并推送到自己的远程仓库

```
michaeldeMacBook-Pro:xm-beat-tb michael$ git log --oneline
3825d0a (HEAD -> master, origin/master, origin/HEAD) Init project
michaeldeMacBook-Pro:xm-beat-tb michael$ cd shopping-cart/
michaeldeMacBook-Pro:shopping-cart michael$ echo "This is shopping-cart page" >> README.md
michaeldeMacBook-Pro:shopping-cart michael$ echo "I'm done" > Index.html
michaeldeMacBook-Pro:shopping-cart michael$ echo "I'm done" >  shopping-car.html

michaeldeMacBook-Pro:shopping-cart michael$ git add .
michaeldeMacBook-Pro:shopping-cart michael$ git commit -m"Shopping cart version 1.0"
[master fd9e9c8] Shopping cart version 1.0
 3 files changed, 4 insertions(+)
 create mode 100644 shopping-cart/Index.html
 create mode 100644 shopping-cart/shopping-cart.html
michaeldeMacBook-Pro:shopping-cart michael$ git log --oneline
fd9e9c8 (HEAD -> master) Shopping cart version 1.0
3825d0a (origin/master, origin/HEAD) Init project

michaeldeMacBook-Pro:shopping-cart michael$ git push
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 4 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (6/6), 442 bytes | 442.00 KiB/s, done.
Total 6 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To michaellqq:michaellqq/xm-beat-tb.git
   3825d0a..fd9e9c8  master -> master

michaeldeMacBook-Pro:xm-beat-tb michael$ git log --oneline
fd9e9c8 (HEAD -> master, origin/master, origin/HEAD) Shopping cart version 1.0
3825d0a Init project
```

# 小阳的协作过程
1. 小猛作为协作者，使用自己的github账号登陆，并fork主仓库 https://github.com/michaellqu/beat-taobao，并更改仓库名为xy-beat-tb
2. clone远程仓库到本地，在本地完成开发，完成后推送到自己在github的公开仓库，并通知小明：
* 克隆自己仓库，配置用户信息
* 修改remote仓库的url为上面配置ssh key时在config里设置的别名。即修改.git/config中的remote仓库对应的，url = git@github.com:michaellgg/xy-beat-tb.git为url = git@michaellgg:michaellgg/xy-beat-tb.git
* 开发订单模块,完成后commit，并推送到自己的远程仓库

过程如下：

```
michaeldeMacBook-Pro:projectdemo michael$ git clone git@github.com:michaellgg/xy-beat-tb.git
Cloning into 'xy-beat-tb'...
remote: Enumerating objects: 6, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (2/2), done.
Receiving objects: 100% (6/6), done.
remote: Total 6 (delta 0), reused 6 (delta 0), pack-reused 0

michaeldeMacBook-Pro:projectdemo michael$ cd xy-beat-tb/
michaeldeMacBook-Pro:xy-beat-tb michael$ git config --local user.name "xiaoyang"
michaeldeMacBook-Pro:xy-beat-tb michael$ git config --local user.email "xiaoyang@beat-tb.com"

michaeldeMacBook-Pro:xy-beat-tb michael$ git remote -v
origin	git@github.com:michaellgg/xy-beat-tb.git (fetch)
origin	git@github.com:michaellgg/xy-beat-tb.git (push)

michaeldeMacBook-Pro:xy-beat-tb michael$ vi .git/config
michaeldeMacBook-Pro:xy-beat-tb michael$ git remote -v
origin	git@michaellgg:michaellgg/xy-beat-tb.git (fetch)
origin	git@michaellgg:michaellgg/xy-beat-tb.git (push)

michaeldeMacBook-Pro:xy-beat-tb michael$ cd order
michaeldeMacBook-Pro:order michael$ echo "This is order page" > README.md
michaeldeMacBook-Pro:order michael$ echo "I'm done" > index.html
michaeldeMacBook-Pro:order michael$ git add .
michaeldeMacBook-Pro:order michael$ git commit -m"Order version 1.0"
[master 6e9dfdb] Order version 1.0
 2 files changed, 2 insertions(+)
 create mode 100644 order/index.html
 
 michaeldeMacBook-Pro:xy-beat-tb michael$ git push
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 4 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (5/5), 375 bytes | 375.00 KiB/s, done.
Total 5 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To michaellgg:michaellgg/xy-beat-tb.git
   3825d0a..6e9dfdb  master -> master

michaeldeMacBook-Pro:xy-beat-tb michael$ git log --oneline --graph
* 6e9dfdb (HEAD -> master, origin/master, origin/HEAD) Order version 1.0
* 3825d0a Init project
```
# 小明完成代码集成
小明在完成公共仓库创建后，也按分工开始自己负责的会员模块的开发，并在本地做了两次commit，分别为84ed127，de413f1：

```
michaeldeMacBook-Pro:beat-taobao michael$ cd member/
michaeldeMacBook-Pro:member michael$ ls
README.md
michaeldeMacBook-Pro:member michael$ echo "Do something first" >> index.html
michaeldeMacBook-Pro:member michael$ git add .
michaeldeMacBook-Pro:member michael$ git commit -m"Add member index.html"
[master 84ed127] Add member index.html
 1 file changed, 1 insertion(+)
 create mode 100644 member/index.html
michaeldeMacBook-Pro:member michael$ echo "I'm done" >> index.html
michaeldeMacBook-Pro:member michael$ echo "This is a register page" >> register.html
michaeldeMacBook-Pro:member michael$ git add .
michaeldeMacBook-Pro:member michael$ git commit "Add member register.html"
error: pathspec 'Add member register.html' did not match any file(s) known to git
michaeldeMacBook-Pro:member michael$ git commit -m"Add member register.html"
[master de413f1] Add member register.html
 2 files changed, 2 insertions(+)
 create mode 100644 member/register.html
michaeldeMacBook-Pro:member michael$ git log --oneline
de413f1 (HEAD -> master) Add member register.html
84ed127 Add member index.html
3825d0a (origin/master) Init project
```
小明收到小猛邮件，得知小猛已完成购物车模块的开发及测试，请求代码合并。小明添加小猛github上的仓库为远程仓库，拉取其代码到本地，新建一个分支xmbranch ，review其代码后合并入master：

```
michaeldeMacBook-Pro:member michael$ git remote add upstream-xiaomeng git@michaellqq:michaellqq/xm-beat-tb.git
michaeldeMacBook-Pro:member michael$ git remote -v
origin	git@michaellqu:michaellqu/beat-taobao.git (fetch)
origin	git@michaellqu:michaellqu/beat-taobao.git (push)

michaeldeMacBook-Pro:member michael$ git fetch upstream-xiaomeng
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 6 (delta 1), reused 6 (delta 1), pack-reused 0
Unpacking objects: 100% (6/6), done.
From michaellqq:michaellqq/xm-beat-tb
 * [new branch]      master     -> upstream-xiaomeng/master
michaeldeMacBook-Pro:member michael$ git branch -av
* master                           de413f1 [ahead 2] Add member register.html
  remotes/origin/master            3825d0a Init project
  remotes/upstream-xiaomeng/master fd9e9c8 Shopping cart version 1.0
  
michaeldeMacBook-Pro:member michael$ git checkout -b xmbranch upstream-xiaomeng/master
Branch 'xmbranch' set up to track remote branch 'master' from 'upstream-xiaomeng'.
Switched to a new branch 'xmbranch'
michaeldeMacBook-Pro:member michael$ git branch -av
  master                           de413f1 [ahead 2] Add member register.html
* xmbranch                         fd9e9c8 Shopping cart version 1.0
  remotes/origin/master            3825d0a Init project
  remotes/upstream-xiaomeng/master fd9e9c8 Shopping cart version 1.0

#可以看到xmbranch分支和小明的master分支已经分叉
michaeldeMacBook-Pro:member michael$ git log --oneline --all --graph
* de413f1 (master) Add member register.html
* 84ed127 Add member index.html
| * fd9e9c8 (HEAD -> xmbranch, upstream-xiaomeng/master) Shopping cart version 1.0
|/
* 3825d0a (origin/master) Init project

#经过review，小明决定合并小猛的代码
michaeldeMacBook-Pro:member michael$ git checkout master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 2 commits.
  (use "git push" to publish your local commits)
michaeldeMacBook-Pro:member michael$ git merge xmbranch
Merge made by the 'recursive' strategy.
 shopping-cart/Index.html         | 1 +
 shopping-cart/README.md          | 1 +
 shopping-cart/shopping-cart.html | 2 ++
 3 files changed, 4 insertions(+)
 create mode 100644 shopping-cart/Index.html
 create mode 100644 shopping-cart/shopping-cart.html
 
 #可以看到经过合并后新生成一个commit3decafb
 michaeldeMacBook-Pro:member michael$ git log --oneline --all --graph
*   3decafb (HEAD -> master) Merge branch 'xmbranch'
|\
| * fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
* | de413f1 Add member register.html
* | 84ed127 Add member index.html
|/
* 3825d0a (origin/master) Init project
```
同样，小明也可以再次添加小阳的仓库，并在本地新建分支review其代码后，决定是否合并。完成所有合并后，推送到主仓库。

```
michaeldeMacBook-Pro:member michael$ git remote add upstream-xiaoyang git@michaellgg:michaellgg/xy-beat-tb.git
michaeldeMacBook-Pro:member michael$ git fetch upstream-xiaoyang
michaeldeMacBook-Pro:member michael$ git checkout -b xybranch upstream-xiaoyang/master
michaeldeMacBook-Pro:member michael$ git checkout master
michaeldeMacBook-Pro:member michael$ git merge xybranch

michaeldeMacBook-Pro:member michael$ git log --oneline --all --graph
*   f770c78 (HEAD -> master) Merge branch 'xybranch'
|\
| * 6e9dfdb (upstream-xiaoyang/master, xybranch) Order version 1.0
* |   3decafb Merge branch 'xmbranch'
|\ \
| * | fd9e9c8 (upstream-xiaomeng/master, xmbranch) Shopping cart version 1.0
| |/
* | de413f1 Add member register.html
* | 84ed127 Add member index.html
|/
* 3825d0a (origin/master) Init project

michaeldeMacBook-Pro:member michael$ git push
Enumerating objects: 29, done.
Counting objects: 100% (26/26), done.
Delta compression using up to 4 threads
Compressing objects: 100% (16/16), done.
Writing objects: 100% (23/23), 2.00 KiB | 293.00 KiB/s, done.
Total 23 (delta 5), reused 0 (delta 0)
remote: Resolving deltas: 100% (5/5), completed with 1 local object.
To michaellqu:michaellqu/beat-taobao.git
   3825d0a..f770c78  master -> master
michaeldeMacBook-Pro:member michael$ git branch -av
* master                           f770c78 Merge branch 'xybranch'
  xmbranch                         fd9e9c8 Shopping cart version 1.0
  xybranch                         6e9dfdb Order version 1.0
  remotes/origin/master            f770c78 Merge branch 'xybranch'
  remotes/upstream-xiaomeng/master fd9e9c8 Shopping cart version 1.0
  remotes/upstream-xiaoyang/master 6e9dfdb Order version 1.0
```
# 总结
本篇主要通过一个集成管理者工作流的协作场景，详细演示了github创建repository，以及如何fork一个repository，然后clone到本地；在管理本地仓库的过程中，使用暂存区、工作区，添加远程仓库，建立跟踪分支，并进行fetch，新建分支，merge等操作。虽然是一个简化的流程，但熟悉之后，可以进一步根据实际工作场景来灵活运用适合自己团队的协作模式。
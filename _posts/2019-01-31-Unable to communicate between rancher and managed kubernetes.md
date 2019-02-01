---  
layout: post  
title: "Rancher与托管Kubernetes集群之间无法通讯"  
subtitle: TLS认证失败故障处理  
date: 2019-01-31  
author: "张志龙"  
header-img: "img/post-bg-rwd.jpg"  
header-mask: "0.1"  
catalog: True  
tags:  
- Rancher  
- Kubernetes  
- 故障处理  
---  

>　　最近在使用rancher管理k8s集群的过程中，遇到了rancher server无法和rancher agent因TLS认证错误无法通讯的问题，经查询后，可以通过重新部署agent的方式来解决，在此记录问题的处理过程，希望对各位读者也能有所帮助。  

# 环境说明  
　　四台Linux虚拟机，其中一台以docker形式运行rancher-server，其他三台运行通过rancher-server创建的k8s集群，集群名称为“myk8s”。  
　　以下为环境中相关软件的版本说明。  
* 操作系统: CentOS 7.5 64位  
* rancher 主机的docker: 升级前docker-ce17.03，升级后docker-ce18.09  
* k8s主机的docker: docker-ce17.03  
* rancher: rancher2.1.5  

# 故障现象  

　　针对Rancher-server所在的服务器的docker-ce版本进行了升级，从17.03升级到了18.06，升级之后Rancher-server的容器启动也正常，但是在Rancher-server的管理台页面看到所管理的myk8s集群无法连接了，一直处于“Provisioning”状态，下方提示等待agent连接，无法通过Rancher server的web页面管理myk8s集群。  
![2019-02-01-09-45-36](http://img.zzl.yuandingsoft.com/blog/2019-02-01-09-45-36.png)  
　　查询Rancher-server容器的日志，发现有大量的提示TLS握手错误，原因是认证证书有问题，导致Rancher-server和Rancher-agent进行通讯，因此Rancher-server无法通过安装在k8s集群中的Rancher-agent对k8s集群进行管理了。  
``` javascript  
2019/01/23 10:00:49 [INFO] 2019/01/23 10:00:49 http: TLS handshake error from 192.168.51.201:56744: remote error: tls: bad certificate  
2019/01/23 10:00:59 [INFO] 2019/01/23 10:00:59 http: TLS handshake error from 192.168.51.200:52786: remote error: tls: bad certificate  
2019/01/23 10:00:59 [INFO] 2019/01/23 10:00:59 http: TLS handshake error from 192.168.51.202:43754: remote error: tls: bad certificate  
```  
# 故障原因  
　　在故障发生之前，仅对Rancher-server所在的服务器的docker-ce的版本进行了升级，升级之前也没有关闭正在运行的容器，升级过程中docker服务重启了，所有的容器也都被重启了。  
　　因此导致故障的原因可能有两个，一个是docker版本的升级，一个是没有正常启停容器和docker服务。  
　　通过直接重启Rancher-server所在服务器上的docker服务进行验证，发现Rancher-server容器重启后运行正常，可以正常管理k8s集群。因此可以排除没有正常启停容器和docker服务这个原因。  
　　在Rancher的官方网站上查找到运行rancher的环境要求，发现官方推荐在centos上运行rancher的docker版本是docker-ce 17.03.2。而我将docker的版本升级到了18.09，可能不是最佳的运行版本，对rancher的支持有问题，才导致这个问题的出现。  
![2019-02-01-10-25-20](http://img.zzl.yuandingsoft.com/blog/2019-02-01-10-25-20.png)  
　　同样在kubernetes官方，其推荐的docker版本也是docker-ce17.03.2，因此我们在安装使用rancher及kubernetes的时候，一定要到官方查看推荐的最佳运行环境，以避免出现各种问题。  
# 故障处理  
　　要想恢复Rancher-server管理对应k8s集群的功能，必须让Rancher-server和Rancher-agent能够正常通讯才行，只要按照最初的配置重新部署一次Rancher-agent即可恢复正常的管理。  
　　那么该如何获取Rancher-agent最初的配置是首要的问题。  
　　在此之前我们需要了解Rancher-agent在k8s集群中具体都运行了哪些资源。  
　　Rancher-agent在k8s集群中是以两种资源形式运行的，一个是以deployment资源运行的cluster-agent，一个是以daemonset资源运行的node-agent，因此需要重新部署这两种资源，让其与rancher-server能够正常通讯。  

## 1.准备工作  
　　Rancher官方提供了生成agent资源以及获取k8s集群的kubeconfig文件的脚本，脚本中用到了jq（命令行json处理工具）等命令。因此需要事先准备相关的脚本并上传到rancher-server所在服务器，并在服务器上安装配套的命令。  
　　相关的脚本和命令可以依照下方给出方法下载，或者从我的网盘中下载，链接: [https://pan.baidu.com/s/11IsNCxzmFbHqI-5gpJP17w](https://pan.baidu.com/s/11IsNCxzmFbHqI-5gpJP17w) 提取码: qxe6 
### 1.1获取生成agent资源的脚本  
　　官方提供了相关的脚本，访问相关的网页，复制其中的脚本，单节点运行rancher-server的使用get_agents_yaml_single.sh脚本，以集群形式运行rancher-server的 使用get_agents_yaml_ha.sh脚本。  
　　以下为官方提供的网页：  
　　[https://gist.github.com/superseb/d59f26102f0a8671672f8035811b2184](https://gist.github.com/superseb/d59f26102f0a8671672f8035811b2184)  
### 1.2获取生成kubeconfig的脚本  
使用如下命令下载，下载得到get_kubeconfig_custom_cluster_rancher2.sh脚本。  
``` javascript  
wget https://gist.githubusercontent.com/superseb/f6cd637a7ad556124132ca39961789a4/raw/c2bb60a51fa4dafef2964589ce0bc9d923653aa1/get_kubeconfig_custom_cluster_rancher2.sh  
```  
### 1.3安装jq命令  
　　如果系统中没有jq命令，则可通过如下方式安装。  
　　通常系统自带的repo仓库中没有jq相关的rpm包，直接从jq官方下载的源码包也无法正常编译通过，因此可以添加包含jq软件的repo仓库，之后通过yum方式直接安装。安装方式如下：  
``` javascript  
# 下载包含jq的repo仓库源，其实就是epel的yum源  
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm  
# 安装repo仓库  
rpm -ivh epel-release-latest-7.noarch.rpm  
# 查看已经安装的repo仓库  
yum repolist  
# 安装jq  
yum install jq  
```  

## 2.获取agent资源的配置文件  
　　登录运行rancher-server的主机，上传相关的脚本，我的环境是单节点运行rancher-server的，因此我使用的脚本是get_agents_yaml_single.sh。我的k8s集群的名称是“myk8s”。  
　　脚本的使用语法是：./get_agents_yaml_single.sh _k8s-cluster-name_  
　　执行脚本，将会自动生成myk8s集群对应的agent资源配置文件，我这里生成的文件是“**rancher-agents-myk8s.yml**”。  
``` javascript  
./get_agents_yaml_single.sh myk8s  
```  
## 3.获取原k8s集群的kubeconfig  
　　登录运行rancher-server的主机，上传相关的脚本get_kubeconfig_custom_cluster_rancher2.sh。  
　　脚本的使用语法是：./get_kubeconfig_custom_cluster_rancher2.sh _k8s-cluster-name_  
　　执行脚本，将会自动生成myk8s集群对应的kubeconfig配置文件，我这里生成的文件是“**kubeconfig**”。  
``` javascript  
./get_kubeconfig_custom_cluster_rancher2.sh myk8s  
```  
## 4.重新部署agent  
　　执行kubectl命令部署相关的agent资源，之后在rancher-server的管理台中，可以看到myk8s集群已经可以被管理了，且之前运行的所有服务也都在。  
``` javascript  
kubectl --kubeconfig=kubeconfig apply -f rancher-agents-myk8s.yml  
```  
　　故障处理流程的参考链接：[https://gist.github.com/superseb/d59f26102f0a8671672f8035811b2184](https://gist.github.com/superseb/d59f26102f0a8671672f8035811b2184)  
　　本文中涉及到的脚本及命令，如果无法从原地址下载，可以从我的网盘中下载，
链接: [https://pan.baidu.com/s/11IsNCxzmFbHqI-5gpJP17w](https://pan.baidu.com/s/11IsNCxzmFbHqI-5gpJP17w) 提取码: qxe6 
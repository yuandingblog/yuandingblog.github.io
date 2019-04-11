---  
layout: post  
title: "业务容器化改造实践（3）"  
subtitle: 业务容器化改造的方案——搭建容器平台和私有镜像仓库  
date: 2019-03-26  
author: "张志龙"  
header-img: "img/post-bg-discovery-k8s.jpg"  
header-mask: "0.1"  
catalog: True  
tags:  
  - 容器  
  - Container  
  - Docker  
  - Kubernetes  
  - Rancher  
  - Harbor  
  - Gitlab  
  - Jenkins  
  - CI/CD  
  - DevOps  
---  

>　　前面的文章讲到了业务容器化改造的思路和方案，主要是从三个方面进行：业务的微服务化拆分、构建自动的CI/CD发布流水线、搭建容器平台。上篇文章介绍了业务微服务化拆分相关的方案，本篇文章介绍容器平台的搭建方案。  


# 一、容器平台的方案架构及规划  
## 1.1 容器平台的方案架构  
　　我们选择Kubernetes作为容器云管理平台，除了官方开源的kubernetes平台之外，各大公有云厂商也都推出了公有云版本的kubernetes平台，比如：阿里云的ACK（Aliyun Container Service for Kubernetes）、腾讯云的TKE（Tencent Kubernetes Engine）、华为云的CCE（Cloud Container Engine）、谷歌的GKE（Google Kubernetes Engine）、亚马逊的EKS（Amazon Elastic Container Service for Kubernetes）、微软的AKS（Azure Kubernetes Service）等。还有一批厂商推出了可私有部署的kubernetes平台或管理平台，比如：IBM的ICP（IBM Cloud Private）和Rancher公司的RKE（Rancher Kubernetes Engine）都是可以私有部署的kubernetes平台。另外Rancher公司的Rancher产品是一款kubernetes的管理平台，通过该平台我们可以在私有的物理机、虚拟机或者公有云的云主机上快速安装部署kubernetes平台，也可以直接接管公有云厂商提供的公有云版本的kubernetes集群。  
　　无论是使用公有云的Kubernetes平台还是自建私有的Kubernetes平台，其技术架构基本上一致，都是以服务器层为基础，在其上构建Kubernetes平台，利用命令行或者web界面调用其中封装好的API接口来操纵和使用Kubernetes集群。二者的区别主要是公有云厂商提供了所有的IaaS层资源，用户在其上购买云主机，一键部署kubernetes集群，然后使用，不用关心底层的实现和维护问题，同时公有云厂商提供了完善的配套工具和解决方案：如监控、日志、镜像仓库、应用商店等；而自建私有的Kubernetes平台，需要用户自己准备相关的服务器，然后在其上安装kubernetes集群，再根据需要在其上搭建配套的监控、日志、镜像仓库、应用商店等，同时用户需要根据实际情况去维护自下而上所有的系统和服务。  
　　本次我们是利用Rancher自建私有的Kubernetes平台，同时搭建Harbor私有镜像仓库，最终建好的Kubernetes平台主要由四层组成：第一层为基础设施层，包含了服务器、存储、网络等基础资源，其中服务器可以是物理机，基于VMware、OpenStack、CloudStack的虚拟机，甚至是来自公有云的云主机。第二层是容器基础设施层，即docker层，是在服务器主机的操作系统上安装Docker，将底层的环境和资源容器化，形成可以供容器使用的计算资源、存储资源、网络资源等。第三层应用编排和资源调度层，即Kubernetes层，利用Rancher构建Kubernetes平台，将Docker层的资源调度起来，同时可以编排容器，决定容器可以运行几个副本，在哪台主机上运行，遇到压力峰谷时自动扩缩容。最上面一层是应用管理层，即用户可以通过界面对整个平台进行管理，管理台中同时可以提供配套的管理工具，如：CI/CD工具、镜像仓库、应用商店、监控、日志、多租户管理等。  
　　容器云管理平台的架构图如下：  
![2019-02-28-09-47-29](http://img.zzl.yuandingsoft.com/blog/2019-02-28-09-47-29.png)  

## 1.2 容器平台的规划  
　　首先明确一下我们需要搭建的内容：一套Kubernetes平台、一套私有镜像仓库、一套CI/CD工具、一套代码仓库。其中CI/CD工具和代码仓库可以直接在kubernetes平台中运行，因此不必单独搭建。为了方便管理镜像，同时搭建kubernetes平台的时候就已经用到很多镜像，所以私有的镜像仓库可以独立部署。利用Rancher搭建Kubernetes平台的时候，Rancher管理台自身可以作为一个单独的容器来运行，也可以先用Rancher的RKE搭建一套Kunernetes集群，在集群中运行高可用的Rancher管理台，为了方便，我们采用单独容器的形式来运行Rancher管理台。  
　　在kubernetes集群中，部署容器应用时，如果涉及到应用数据的持久化，一般可以采用给容器挂载外部目录的方式来实现，即将外部的目录（来自宿主机的目录，或者是NFS等存储）映射到容器中的目录上，这样容器中的数据就存储到外部的目录中了，从而实现了数据的持久化。考虑到容器可能会在任意的一个宿主机上运行，只要容器启动就能够访问到对应的目录，因此不能直接使用宿主机操作系统上的目录，而要使用一个所有宿主机都能够访问到的共享的目录或存储，解决方案有多种，可以使用基于宿主机本地硬盘的分布式存储（如Ceph），可以使用外部的存储设备（如NFS等），或者使用云厂商提供的云硬盘。我们这里采用NFS的方式来提供共享的存储。  
　　因此，我们需要一台服务器来运行私有镜像仓库和Rancher管理台，同时提供NFS的共享存储，其余服务器运行Kunernetes集群，最终容器平台的规划如下：  

序号 | 用途 | 配置要求 | 数量  
---------|----------|---------|------  
 1 | Rancher + Harbor | Linux主机（3.10以上内核，推荐CentOS/Redhat7.4或Ubuntu16.04以上），2C，4G内存，100G硬盘 | 1台  
 2 | Kubernetes | Linux主机（3.10以上内核，推荐CentOS/Redhat7.4或Ubuntu16.04以上），4C，16G内存，60G硬盘 | 3台  
 3 | CI/CD工具 + 代码仓库 | 运行在Kubernetes中 | -  



>  注意：本次安装过程均为在线安装，即所有服务器均可以访问互联网。离线安装也可以，需要提前准备对应的软件包和镜像等，本文中不做讨论。  



# 二、准备工作  
　　按照要求准备四台服务器，物理机、虚拟机、云主机均可，各个主机的规划如下：  

序号 | 主机名 | IP地址 | 配置 | 用途  
---------|---------|---------|---------|---------  
 1 | k8s-node1 | 192.168.51.200 | 4核，16G内存，50G磁盘 | K8s集群的Node1节点  
 2 | k8s-node2 | 192.168.51.201 | 4核，16G内存，50G磁盘 | K8s集群的Node2节点  
 3 | k8s-node3 | 192.168.51.202 | 4核，16G内存，50G磁盘 | K8s集群的Node3节点  
 4 | harbor | 192.168.51.203 | 2核，4G内存，100G磁盘 | Harbor和rancher节点  

　　其中Rancher和Harbor需要使用域名进行访问，因此各个主机上需要配置hosts表，用于解析。需要配置的域名如下，IP地址为Rancher和Harbor所在主机的IP地址：  
　　Rancher server的登录域名：rancher.yuandingit.com  
　　Harbor私有镜像仓库的名称：harbor.yuandingit.com  
　　因此所有主机上的hosts文件配置为：  
```  
192.168.51.200 	k8s-node1  
192.168.51.201 	k8s-node2  
192.168.51.202 	k8s-node3  
192.168.51.203 	harbor	harbor.yuandingit.com  
192.168.51.203 	harbor	rancher.yuandingit.com  
```  

　　为了保证各个节点之间服务的正常访问，可以关闭操作系统的防火墙，如必须开启，请根据需要开放对应的端口，具体参考Docker、Kubernetes、Rancher、Harbor官方的要求。如下为关闭防火墙的方法：  
```bash  
# 禁用SElinux  
# 临时将selinux修改为宽松模式，重启后会恢复到/etc/selinux/config中配置的模式。  
setenforce 0  

# 或者可以永久关闭selinux（需要重启才能生效）：  
# 修改/etc/selinux/config文件配置，将SELINUX=enforcing修改为SELINUX=disabled  

# 禁用防火墙  
systemctl disable firewalld && systemctl stop firewalld  
```  


# 三、安装Docker  
　　四台服务器上均安装docker，目前Rancher支持的docker版本是操作系统自带的docker 1.12.6、1.13.1，或者docker-ce 17.03.2。kubernetes支持的docker版本是操作系统自带的docker 1.11、1.12、1.13，或者docker-ce 17.03、18.09。我们推荐安装docker-ce 17.03.2，其它版本不支持或者可能有bug。  
　　以上支持的版本为笔者编写本篇文章时查询的结果，后续可能会有变化，具体可以查询官方文档进行确认：[Rancher官方文档](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/basic-environment-configuration/)，[Kubernetes官方文档](https://kubernetes.io/docs/setup/cri/)。  
　　下面是安装docker-ce 17.03.2版本的方法，其它版本的安装方法可能与此不同，具体可查询 [Docker官方文档](https://docs.docker.com/install/linux/docker-ce/centos/) 进行确认。   

## 3.1 添加yum源  
　　由于安装docker-ce时，需要用到一些操作系统的依赖包，因此需要添加可用的yum源，如操作系统的ISO文件，或者国内的阿里云等提供的开放YUM源。另外docker-ce的yum源在国外，可能根本无法下载到，可以替换为国内的yum源。下面给出可能用到的yum源，可下载对应的repo文件，放置到 /etc/yum.repo.d/ 目录下即可。  
　　Docker官方的yum源：[https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)  
　　清华大学的Docker yum源配置方法：[https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)  
　　阿里云的CentOS操作系统yum源：[http://mirrors.aliyun.com/repo/Centos-7.repo](http://mirrors.aliyun.com/repo/Centos-7.repo)  


```bash  
# 下载docker官方的yum文件  
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo  

# 替换为清华大学的源  
sudo sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo  

# 更新yum源  
yum makecache fast  
```  
## 3.2 安装docker-ce  
　　Docker-ce的安装方式如下：  
```bash  
# 如果之前安装过其他版本的docker，请先卸载掉  
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine docker-selinux  

# 安装依赖包，通常情况下如下三个软件包默认是已经安装的  
yum install -y yum-utils device-mapper-persistent-data lvm2  

# Docker-ce的yum源中有更新的版本，因此需要查看17.03.2的版本的具体版本号，为后续安装做准备。  
yum list docker-ce --showduplicates | sort -r  

# 上述命令的结果通常如下，从中选择对应的docker-ce 17.03.2版本即可  
docker-ce.x86_64            3:18.09.3-3.el7                    docker-ce-stable  
docker-ce.x86_64            3:18.09.2-3.el7                    docker-ce-stable  
……  
docker-ce.x86_64            17.03.3.ce-1.el7                   docker-ce-stable  
docker-ce.x86_64            17.03.2.ce-1.el7.centos            docker-ce-stable  
docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable  
docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable  

# 安装版本号为17.03.2的docker-ce软件，同时需要安装对应版本的docker-ce-selinux，否则安装不上：其他版本依赖的软件包可能是其他的，具体参考官方文档。  
yum install -y --setopt=obsoletes=0 docker-ce-17.03.2.ce docker-ce-selinux-17.03.2.ce  

# 设置开机启动docker  
systemctl start docker && systemctl enable docker  

```  

　　此时使用docker时，可能会存在下载镜像慢的问题，可以注册一个阿里云的帐号，免费开通“容器镜像服务”，并根据其中的说明将镜像加速器配置到本地的docker中。  
![2019-03-19-18-01-55](http://img.zzl.yuandingsoft.com/blog/2019-03-19-18-01-55.png)  
　　例如，我这里用到的镜像加速器地址是：https://supz5x7p.mirror.aliyuncs.com，可以用如下的方法添加到本地docker中：  
```bash  
# 新建docker的daemon.json文件,并将加速器配置到文件中  
mkdir -p /etc/docker  
echo '{  
"registry-mirrors": ["https://supz5x7p.mirror.aliyuncs.com"]  
}' > /etc/docker/daemon.json  

# 因为daemon.json文件内容改变了，需要重启docker服务  
systemctl daemon-reload && systemctl restart docker  

# 验证docker可以正常运行，没有报错，且输出 “Hello from Docker!” 即说明可用。  
docker run hello-world  

```  

# 四、构建Kubernetes集群  
## 4.1 安装Rancher管理台  
　　本次规划中，Rancher和Harbor在一台服务器上，由于二者均需要使用80和443端口，因此可以将rancher的端口调整为8080和8443，为了保证rancher的数据持久化，可以挂在本地的目录给rancher的容器，因此单容器运行Rancher管理台的命令如下：  
```bash  
docker run -d \  
--restart=unless-stopped \  
-n rancher-managment  \  
-p 8080:80 \  
-p 8443:443 \  
-v /data/rancher/data:/var/lib/rancher \  
rancher/rancher:v2.1.6  
```  
　　成功运行之后，可以通过如下方式访问，可以确认Rancher的管理地址，以及设置管理员用户名和密码。  
　　https://192.168.51.203:8443  
　　https://rancher.yuandingit.com:8443   (需要客户端配置hosts映射)  

## 4.2 创建Kubernetes集群  
　　进入“global”→选择“add cluster”→选择“from my own existing nodes ： custom”→填写集群名称“myk8s”→“下一步”。  
![2019-03-19-19-03-13](http://img.zzl.yuandingsoft.com/blog/2019-03-19-19-03-13.png)  
 
　　定制各个节点的角色，各个节点的角色规划如下：  

序号 | 主机名 | IP地址 | 角色  
---------|----------|---------|---------  
 1 | k8s-node1 | 192.168.51.200 | Etcd、 controlplane、worker  
 2 | k8s-node2 | 192.168.51.201 | Worker  
 3 | k8s-node3 | 192.168.51.202 | Worker  


　　根据规划，选择角色，然后复制下方的命令到对应的节点上去执行，将会自动开始安装。注意节点上需要已经安装好对应的docker。  
　　开始执行命令之后，即可点击“done”返回，观察集群的安装进度。  

## 4.3 Kubectl配置  
　　通过rancher server构建的k8s集群默认是没有安装kubectl命令行的，可以通过页面上的kubectl命令行进行管理，如果需要在服务器上执行kubectl命令管理集群的话，可以自行安装kubectl命令和配置kubeconfig文件。  
　　下载kubectl命令:  
　　可以从rancher官网下载 kubectl_linux-amd64：[https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/download/](https://www.cnrancher.com/docs/rancher/v2.x/cn/installation/download/)  
　　
　　安装kubectl命令：  
```bash  
# 复制kubectl命令到PATH中  
cp kubectl_linux-amd64 /usr/local/bin/kubectl  

# 授予执行权限  
chmod +x /usr/local/bin/kubectl  

# 增加自动补全功能  
echo "source <(kubectl completion bash)" >> ~/.bash_profile  
```  
　　配置kubeconfig文件：  
　　登录rancher server，进入对应的集群，在右上角点击“kubeconfig file”，复制其中的内容，然后粘贴到服务器上的~/.kube/config（文件和目录不存在就创建一个）文件中，在重开一个终端即可正常使用kubectl命令管理集群。  

## 4.4 配置kubernetes集群的共享存储  
　　我们选择使用NFS的方式来提供共享存储，其中192.168.51.203服务器为NFS服务器，其他三台服务器为NFS客户端，可以将NFS服务器上的目录直接挂在到NFS客户端服务器上，作为本地目录来使用。这样在启动容器的时候，配置持久卷，可以配置为主机路径，也可以直接配置为NFS，容器都可以使用来自NFS的共享目录。不过对于容器中挂在外部目录时有子路径（subPath）的，则不能使用主机路径，因此推荐直接使用NFS的方式，尽量避免使用本地路径。  
　　下面介绍NFS服务的配置方法：  
```bash  
# 所有的服务器上安装nfs软件包 nfs-utils，自动会安装好依赖的软件包 rpcbind。  
yum install nfs-utils  

# NFS服务器端启动NFS服务和rpcbind服务，NFS客户端只用启动rpcbind服务即可。  
systemctl start nfs &&  systemctl enable nfs  
systemctl start rpcbind &&  systemctl enable rpcbind  

# NFS服务器端配置共享的NFS目录，假设共享的目录为 “/nfs”，挂载到客户端也是“/nfs”  

# NFS服务器端执行  
# 创建共享目录  
mkdir /nfs  
# 编辑/etc/exports文件，添加NFS共享目录，并配置权限，如果有IP地址段限制则写IP地址段，如果没有限制，可以写为“*” ，比如 /nfs *(rw,no_root_squash,no_all_squash,sync)  
vi /etc/exports  
/nfs	192.168.51.0/24(rw,no_root_squash,no_all_squash,sync)  
# 导出共享目录，将会看到已经添加的NFS目录，加上参数 -rv 时，则可以不用重启nfs服务即可显示新增的共享目录，且客户端可以直接mount。  
exportfs -rv  

# 客户端执行挂载  
mount 192.168.51.203:/nfs /nfs  
# 客户端如果需要实现开机自动挂载，则可以将相关的记录添加到/etc/fstab分区表中  
vi /etc/fstab  
192.168.51.203:/nfs	/nfs	nfs	defaults	0	0  
```  



# 五、构建Harbor私有镜像仓库  
## 5.1 下载所需软件  

　　因为运行harbor需要docker-compose，因此需要下载harbor和docker-compose软件。  
　　下载：Harbor online installer包（最新版本，本次是1.7.1）Harbor：[https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)  
　　下载运行harbor的命令docker-compose（1.7.1+）: [https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)  


　　安装docker-compose:  
```bash  
# 复制命令到PATH中：  
cp docker-compose-Linux-x86_64 /usr/local/bin/docker-compose  
# 授予执行权限  
chmod +x /usr/local/bin/docker-compose  
```  

## 5.2 安装harbor  
　　解压缩软件包  
```bash  
tar -xvf harbor-online-installer-v1.7.1.tgz  
# 解压后得到harbor目录。  
```  
　　进入harbor目录，修改harbor.cfg文件，主要修改以下的参数。  

序号 | Key | Value | 说明  
--|--|--|--  
 1 | hostname | harbor.yuandingit.com | 镜像仓库的域名  
 2 | Ui_url_protocol | http | 访问镜像仓库的协议，默认http  
 3 | Customize_crt | off | 是否使用crt认证，默认是on  
 4 | harbor_admin_password | *** | admin用户的密码  

```bash  
# 进入harbor目录，并执行install.sh，同时安装chart仓库和镜像扫描功能。该脚本中使用到了docke-compose命令。  
cd harbor  
./install.sh --with-chartmuseum --with-clair  
```  
　　安装完成后启动的容器会占用操作系统的80、443、4443、1514端口。  
　　安装完成后的访问地址是：http://harbor.yuandingit.com（如果没有使用NDS进行域名解析，则需要在hosts文件中增加域名和IP的映射 ；或者直接使用IP地址登录访问：http://192.168.51.203　）。  
　　登录时，可以使用admin登录，也可以注册一个普通的用户。  

## 5.3 其他节点访问Harbor私有镜像仓库  
　　其他节点需要访问harbor镜像仓库时，需要在本机的/etc/hosts文件中加入harbor的解析：  
　　192.168.51.203	harbor.yuandingit.com  

　　由于本次实验中镜像仓库使用的是http协议，因此在其他服务器使用该私有镜像仓库时，需要调整docker的配置参数，将私有仓库的地址加入到不安全仓库列表中，之后可以直接使用该镜像仓库中的镜像，具体方法如下：  
```bash  
#修改 /etc/docker/daemon.json 文件，加入一行 "insecure-registries": ["harbor.yuandingit.com"]  
# 如果该文件中已经有其他的内容，在文件末尾的“}”之前加入这一行，同时上一行的行尾要加上逗号“,”。  
cat /etc/docker/daemon.json  
{  
"registry-mirrors": ["https://supz5x7p.mirror.aliyuncs.com"],  
"insecure-registries": ["harbor.yuandingit.com"]  
}  

# 重启docker进程  
systemctl daemon-reload && systemctl restart docker  
```  

## 5.4 使用Harbor私有镜像仓库  

**1.命令行上传/下载私有镜像仓库**  
　　Harbor私有镜像仓库采用的是项目管理，即一个镜像仓库中拥有多个项目，每个项目中拥有多个镜像，每个镜像拥有多个标签（版本），因此在使用harbor私有镜像仓库之前，需要在仓库中建立项目。登录Harbor之后，点击“新建项目”→填写项目名称，勾选访问级别。  
　　Harbor私有镜像仓库中镜像的命名格式为： 镜像仓库地址/项目名称/镜像名称:镜像标签  ，比如镜像 harbor.yuandingit.com/test/test-image:v1.0 的各部分组成为：  

项目 | 格式  
-- | --  
 镜像仓库地址 | harbor.yuandingit.com  
 项目名称 | test  
 镜像名称 | test-image  
 镜像标签 | v1.0  

```bash  
# 制作名称符合要求的镜像：假设为 harbor.yuandingit.com/test/test-image:v1.0  
docker build -t harbor.yuandingit.com/test/test-image:v1.0 .  
# 将已有镜像重命名为符合要求的镜像  
docker tag source-image:v1 harbor.yuandinit.com/test/test-image:v1.0  
# 登录harbor私有镜像仓库，输入用户名和密码  
docker login harbor.yuandingit.com  
# 上传docker镜像  
docker push harbor.yuandinit.com/test/test-image:v1.0  
# 下载使用镜像  
docker pull harbor.yuandinit.com/test/test-image:v1.0  
```  

**2.Rancher从Harbor私有镜像仓库下载镜像**  
　　在Rancher与Harbor配套使用时，由于Harbor具有用户名密码认证，Rancher不能直接从Harbor镜像仓库下载镜像。  
　　有两种方式解决此问题：一种是把Harbor中建立的项目设置为“公开”，避免进行访问验证，推荐在测试环境使用；另外一种是在rancher中建立对应的harbor镜像仓库的凭证secret，但是该配置只能针对某个命名空间或某个项目，每个命名空间都需要设置，推荐在生产环境使用。  
　　通过Rancher控制台设置镜像仓库secret的方法为，进入k8s集群的一个项目中，点击“资源”→“镜像库凭证”→“添加镜像库凭证”→填写名称、镜像仓库地址、用户名和密码。  
　　或者通过如下的命令创建:  

``` bash
# kubectl create secret docker-registry \  
# harbor-yuandingit-com \                      # secret的名称  
# --namespace=cicd \                           # secret所属的命名空间  
# --docker-server=harbor.yuandingit.com \      # harbor私有镜像仓库的地址  
# --docker-username=admin \                    # harbor私有镜像仓库的用户名  
# --docker-password=Harbor12345                # harbor私有镜像仓库的密码  
# --docker-email=xxx@xxx.com                   # 邮箱，必须填写的  

kubectl create secret docker-registry \  
harbor-yuandingit-com \  
--namespace=cicd \  
--docker-server=harbor.yuandingit.com \  
--docker-username=admin \  
--docker-password=Harbor12345 \  
--docker-email=xxx@xxx.com  
```
>　　注意: 如果在创建secret之前就部署了应用，且应用无法从harbor私有镜像仓库下载镜像，那么在创建secret之后，必须将该应用删除，再次部署才可以从harbor私有镜像仓库中下载镜像。  


>　　结语：至此，我们在所有节点上安装了docker，安装了Rancher并创建了Kubernetes集群，同时还安装好了私有的镜像仓库Harbor，最基础的容器平台已经搭建完成。后续我们将以此为基础开始安装Jenkins和Gitlab，为构建CI/CD自动化流水线做准备。  
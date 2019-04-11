---  
layout: post  
title: "业务容器化改造实践（4）"  
subtitle: 业务容器化改造的方案——在k8s中部署gitlab代码仓库  
date: 2019-04-02  
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

>　　前面的文章讲到了业务容器化改造的思路和方案，主要是从三个方面进行：业务的微服务化拆分、构建自动的CI/CD发布流水线、搭建容器平台。上两篇文章介绍了业务微服务化拆分和搭建容器平台相关的方案，本篇文章开始介绍如何构建自动的CI/CD流水线。  
>　　本篇是构建自动化CI/CD流水线方案的第一篇，主要介绍如何在kubernetes集群中部署gitlab代码仓库。  


# 一、CI/CD流水线的方案架构  
　　我们对业务进行了微服务化拆分，并以容器的形式在Kubernetes集群中运行，那么必须有一套合适的自动CI/CD发布流水线来实现业务的持续集成和持续发布，以满足快速的业务上线和迭代，同时降低管理的成本。下面是我们设计的CI/CD流水线的方案架构图。  
![2019-02-28-10-02-15](http://img.zzl.yuandingsoft.com/blog/2019-02-28-10-02-15.png)  
　　从图中可以看到，涉及的组件主要有四个：gitlab代码仓库、Jenkins集成工具、Harbor私有镜像仓库、Kubernetes集群。其中Kubernetes集群和Harbor私有镜像仓库等基础平台在“业务容器化改造的方案——搭建容器平台和私有镜像仓库”一文中进行了详细的介绍，从本篇文章开始将会介绍gitlab和Jenkins的搭建过程，以及CI/CD流水线的构建过程，本篇主要介绍如何在Kubernetes集群中部署gitlab代码仓库。  
　　本方案中的CI/CD流水线的原理流程如下：  
1. 开发人员编写好程序代码，通过git提交到本地的代码仓库，提交时，需要包含后续打包成镜像的Dockerfile，以及部署到Kubernetes集群中的YAML资源配置模板。  
2. 将本地所有的程序代码及相关的配置文件都推送到远端的gitlab代码仓库服务器中，进行统一的管理。  
3. gitlab与jenkins之间配置有webhook，可以触发jenkins进行后续的集成和交付步骤。  
4. Jenkins配置的流水线任务触发之后，会自动拉取gitlab中的项目代码，然后对源代码进行测试、编译，最终形成软件制品，如JAR包、WAR包等。  
5. 根据项目代码中的Dockerfile，开始构建Docker镜像，镜像中包含了软件的制品及运行所需的环境。  
6. 将构建好的Docker镜像推送到私有的镜像仓库Harbor中，进行统一的管理，方便后续下载使用。  
7. 根据项目代码中的YAML资源配置模板，通过脚本将其中的变量替换为本次构建实际的输入值，生成最终可以在Kubernetes集群中部署的YAML资源配置文件。  
8. 调用Kubernetes的API，执行部署。  
9. Kubernetes集群将依据YAML资源配置文件，从镜像仓库中拉取镜像，分配资源，并启动相关的容器和服务，开始对外提供服务。  


# 二、部署Gitlab代码仓库  
　　Gitlab可以独立部署，也可以以单docker容器的方式运行，也可以部署到K8S集群中。我们这里采用在K8S中部署Gitlab的方式，以NFS目录作为存储持久化数据目录，以服务的方式对外提供服务。  
## 2.1 Gitlab的资源配置文件  
　　在K8S中部署Gitlab，需要配置的资源为Deployment、Service、PV、PVC，下面列出各个资源的yaml配置文件。该配置文件为最基础的配置，可以直接运行使用，一些高级的配置，如容器的就绪检测、存活检测、调度策略等内容可根据实际情况添加对应的配置即可。  
* **Deployment**  

```Groovy  
apiVersion: apps/v1beta1  
kind: Deployment  
metadata:  
  name: gitlab  
  namespace: cicd  
spec:  
  replicas: 1  
  template:  
    metadata:  
      labels:  
        app: gitlab  
    spec:  
      hostAliases:  
      - ip: 192.168.51.203  
        hostnames:  
        - harbor.yuandingit.com  
        - rancher.yuandingit.com  
      hostname: gitlab  
      containers:  
      - name: gitlab  
        image: gitlab/gitlab-ce:latest  
        imagePullPolicy: IfNotPresent  
        volumeMounts:  
        - name: gitlab-config  
          mountPath: /etc/gitlab/  
        - name: gitlab-data  
          mountPath: /var/opt/gitlab/  
        - name: gitlab-logs  
          mountPath: /var/log/gitlab/  
      volumes:  
      - name: gitlab-config  
        persistentVolumeClaim:  
          claimName: pvc-gitlab-config  
      - name: gitlab-data  
        persistentVolumeClaim:  
          claimName: pvc-gitlab-data  
      - name: gitlab-logs  
        persistentVolumeClaim:  
          claimName: pvc-gitlab-logs  
```  

* **Service**  

```Groovy  
apiVersion: v1  
kind: Service  
metadata:  
  labels:  
    svc: gitlab  
  name: gitlab  
  namespace: cicd  
spec:  
  type: NodePort  
  ports:  
  - name: http  
    nodePort: 32082  
    port: 80  
    protocol: TCP  
    targetPort: 80  
  - name: https  
    nodePort: 32083  
    port: 443  
    protocol: TCP  
    targetPort: 443  
  - name: ssh  
    nodePort: 32084  
    port: 22  
    protocol: TCP  
    targetPort: 22  
  selector:  
    app: gitlab  
```  

* **PV**  

```Groovy  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: pv-gitlab-data  
spec:  
  accessModes:  
  - ReadWriteOnce  
  capacity:  
    storage: 3Gi  
  hostPath:  
    path: /nfs/gitlab/data/  
  persistentVolumeReclaimPolicy: Retain  
---  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: pv-gitlab-config  
spec:  
  accessModes:  
  - ReadWriteOnce  
  capacity:  
    storage: 1Gi  
  hostPath:  
    path: /nfs/gitlab/config/  
  persistentVolumeReclaimPolicy: Retain  
---  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: pv-gitlab-logs  
spec:  
  accessModes:  
  - ReadWriteOnce  
  capacity:  
    storage: 2Gi  
  hostPath:  
    path: /nfs/gitlab/logs  
  persistentVolumeReclaimPolicy: Retain  
```  
* **PVC**  

```Groovy  
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: pvc-gitlab-data  
  namespace: cicd  
spec:  
  accessModes:  
  - ReadWriteOnce  
  resources:  
    requests:  
      storage: 3Gi  
---  
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: pvc-gitlab-config  
  namespace: cicd  
spec:  
  accessModes:  
  - ReadWriteOnce  
  resources:  
    requests:  
      storage: 1Gi  
---  
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: pvc-gitlab-logs  
  namespace: cicd  
spec:  
  accessModes:  
  - ReadWriteOnce  
  resources:  
    requests:  
      storage: 2Gi  
```  

## 2.2 部署Gitlab  
　　上述的资源配置文件内容可以写在一个文件中，也可以分开写，假设最终的资源配置文件为gitlab.yaml。则可以执行如下的命令在k8s集群中进行部署。  
```bash  
# 创建CICD命名空间  
kubectl create namespace cicd  
# 执行部署gitlab  
kubectl apply -f gitlab.yaml  
```  
　　因为我们已经将gitlab的服务端口80映射到宿主机上的32082了，Gitlab启动之后，其外部访问地址为：http://192.168.51.200:32082。  
　　在Kubernetes集群内部，gitlab的服务名是gitlab，其所在的命名空间为cicd，则服务全名是gitlab.cicd.svc.cluster.local，进行跨命名空间访问时，服务名可以省略掉后缀，只保留 服务名+命名空间名，即“gitlab.cicd”，所以在集群内部gitlab的访问地址是：http://gitlab.cicd。  

## 2.3 初始化Gitlab  
### 2.3.1 用户及密码  
　　登录gitlab的访问地址之后，首先是要更改管理员的密码，如果没有更改，可以使用管理员密码登录并进行修改。  
　　默认的管理员用户和密码是：admin@example.com / 5iveL!fe  
　　可以在登录界面注册一个新的普通用户供后续与Jenkins之间交互使用。  

### 2.3.2 允许本地网络的webhook  
　　使用**管理员用户**登录系统，修改系统的配置，允许针对本地网络进行webhook，只需配置一次，后续所有的项目都可以使用。  
　　配置的路径是：“admin area”→“Settings”→“network”→“outbound Requests”，勾选允许针对本地网络使用webhook。  
![2019-02-28-11-32-07](http://img.zzl.yuandingsoft.com/blog/2019-02-28-11-32-07.png)  


### 2.3.3 配置Jenkins连接Gitlab的Token  
　　后续jenkins配置与gitlab的连接时，需要用到gitlab中的token，因此在这里建立token并记录下来【**一定要记录下来，因为创建之后关掉页面就再也看不到token的值了**】。  
　　使用与Jenkins交互用的普通用户登录系统，进行配置，配置的路径是：在页面右上角点击用户图标→“settings”→“Access Token”，之后填写token的名称，勾选“API”，然后创建token，最后记录生成的token。  
![2019-02-28-11-34-02](http://img.zzl.yuandingsoft.com/blog/2019-02-28-11-34-02.png)  

# 2.4 创建项目，并同步代码  
　　使用普通用户登录后，可根据需要先创建项目组，再在组内创建项目，并设置可用级别。同时将本地的项目与gitlab的项目关联起来，可以将本地项目代码推送到gitlab中即可。  
　　可以在创建好的项目中查看到项目的链接，有git和http两种，以http为例，假设项目的名称是test，项目组是cicd，gitlab的主机名为gitlab，那么默认的项目链接就是 ： http://gitlab/cicd/test.git 。因为没有外部的dns服务器，而且k8s集群内部的dns中也没有相关的记录，所以这个地址其实在集群外部和内部都是无法访问的。  
　　对于集群外部，如开发人员本地访问时，将本地代码推送到代码仓库，可以使用IP+端口的方式访问，则可以访问的地址需要设置为：http://192.168.51.200:32082/cicd/test.git 。  
　　对于集群内部，如Jenkins访问Gitlab时，可以通过集群内部的服务进行访问，则可以访问的地址需要设置为： http://gitlab.cicd/cicd/test.git 。  


>　　结语：依照上述步骤，我们已经在kubernetes集群中安装部署了gitlab，并实现了gitlab数据的持续化，gitlab已经具备了作为私有代码仓库的能力。后续我们将继续安装jenkins，并利用jenkins和gitlab构建CI/CD自动化流水线。  
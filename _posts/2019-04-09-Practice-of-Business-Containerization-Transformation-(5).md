---  
layout: post  
title: "业务容器化改造实践（5）"  
subtitle: 业务容器化改造的方案——在k8s中部署Jenkins持续集成交付平台  
date: 2019-04-09  
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

>　　前面的文章讲到了业务容器化改造的思路和方案，主要是从三个方面进行：业务的微服务化拆分、构建自动的CI/CD发布流水线、搭建容器平台。前面介绍了业务微服务化拆分和搭建容器平台相关的方案，本篇文章属于构建自动的CI/CD流水线内容。  
>　　本篇是构建自动化CI/CD流水线方案的第二篇，主要介绍如何在kubernetes集群中部署Jenkins。  


# 一、CI/CD流水线的方案架构回顾  
　　我们对业务进行了微服务化拆分，并以容器的形式在Kubernetes集群中运行，那么必须有一套合适的自动CI/CD发布流水线来实现业务的持续集成和持续发布，以满足快速的业务上线和迭代，同时降低管理的成本。下面是我们设计的CI/CD流水线的方案架构图。  
![2019-02-28-10-02-15](http://img.zzl.yuandingsoft.com/blog/2019-02-28-10-02-15.png)  
　　从图中可以看到，涉及的组件主要有四个：gitlab代码仓库、Jenkins集成工具、Harbor私有镜像仓库、Kubernetes集群。其中Kubernetes集群和Harbor私有镜像仓库等基础平台在“业务容器化改造的方案——搭建容器平台和私有镜像仓库”一文中进行了详细的介绍，gitlab私有代码仓库在“业务容器化改造的方案——在k8s中部署gitlab代码仓库”一文中进行了详细的介绍，本篇主要介绍如何在Kubernetes集群中部署Jenkins CI/CD工具。  
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


# 二、Jnekins持续集成工具架构设计  
　　Jenkins自身支持Master-Slave架构，即使用一个Jenkins作为Master服务器（节点），用于管理多个Slave服务器（节点），slave服务器的用途是执行具体的构建任务，这样的好处是将构建任务的压力分摊给多个salve节点来执行，避免master的压力过大造成的构建任务阻塞。  
　　Jenkins支持基于kubernetes的jnlp slave，即可以在kubernetes集群中启动jnlp容器，作为salve服务器来执行构建任务，这样将带来更多的好处：  
* 容器化  
　　Jenkins Slave节点是运行在kubernetes集群中的一个POD，该POD中可以有多个具体执行构建步骤的容器，容器本身的环境是标准化的，可以自己定制容器镜像，具备很强的版本控制和可移植性。同时每个构建任务具备自己的salve节点，与其他构建任务之间具备很好的隔离性。  
* 容器slave全生命周期管理  
　　当构建任务触发时，将会自动在kubernetes集群中创建用于本次构建的Slave节点，并自动在其中执行构建任务，当构建任务结束时（无论是成功或失败）将自动销毁该Slave节点，并释放资源。  
* 高并发  
　　当有多个构建任务同时触发时，将会在Kubernetes集群中创建多个用于执行构建任务的Slave节点。　　
* 资源共享  
　　所有用于执行构建任务的Slave节点共享一个Kuberenetes集群的资源。  

　　因此我们采用Master Slave的架构来部署Jenkins，在Kubernetes中部署持续运行的Jenkins Master，在Jenkins中配置用于执行构建任务的Jenkins Slave，Slave节点只在执行构建任务的时候才会启动运行。  

# 三、部署Jenkins持续集成工具  
　　Jenkins可以独立部署，也可以以单docker容器的方式运行，也可以部署到K8S集群中。我们这里采用在K8S中部署Jenkins的方式，以NFS目录作为存储持久化数据目录，以服务的方式对外提供服务。  
## 3.1 Jenkins的资源配置文件  
　　在K8S中部署Jitlab，需要配置的资源为Role、Deployment、Service、PV、PVC，下面列出各个资源的yaml配置文件。该配置文件为最基础的配置，可以直接运行使用，一些高级的配置，如容器的就绪检测、存活检测、调度策略等内容可根据实际情况添加对应的配置即可。  
* **Role**  

```Groovy  
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  labels:  
    k8s-app: jenkins  
  name: jenkins-admin  
  namespace: cicd  
---  
kind: ClusterRoleBinding  
apiVersion: rbac.authorization.k8s.io/v1beta1  
metadata:  
  name: jenkins-admin  
  labels:  
    k8s-app: jenkins  
subjects:  
  - kind: ServiceAccount  
    name: jenkins-admin  
    namespace: cicd  
roleRef:  
  kind: ClusterRole  
  name: cluster-admin  
  apiGroup: rbac.authorization.k8s.io  
```  
* **Deployment**  

```Groovy  
apiVersion: apps/v1beta1  
kind: Deployment  
metadata:  
  name: jenkins  
  namespace: cicd  
spec:  
  replicas: 1  
  template:  
    metadata:  
      labels:  
        app: jenkins  
    spec:  
      securityContext:  
        runAsUser: 0  
        fsGroup: 0  
      hostAliases:  
      - ip: 192.168.51.203  
        hostnames:  
        - harbor.yuandingit.com  
        - rancher.yuandingit.com  
      hostname: jenkins  
      containers:  
      - name: jenkins  
        image: jenkinsci/blueocean:latest  
        imagePullPolicy: IfNotPresent  
        ports:  
        - containerPort: 8080  
          name: web  
          protocol: TCP  
        - containerPort: 50000  
          name: agent  
          protocol: TCP  
        volumeMounts:  
        - name: jenkinshome  
          mountPath: /var/jenkins_home  
        - name: jenkinsdocker  
          mountPath: /var/run/docker.sock  
      volumes:  
      - name: jenkinshome  
        persistentVolumeClaim:  
          claimName: pvc-jenkins-data  
      - name: jenkinsdocker  
        hostPath:  
          path: /var/run/docker.sock  
          type: Socket  
      serviceAccount: jenkins-admin  
```  
* **Service**  

```Groovy  
apiVersion: v1  
kind: Service  
metadata:  
  labels:  
    svc: jenkins  
  name: jenkins  
  namespace: cicd  
spec:  
  type: NodePort  
  ports:  
  - name: http  
    nodePort: 32080  
    port: 8080  
    protocol: TCP  
    targetPort: 8080  
  - name: slavelistener  
    nodePort: 32081  
    port: 50000  
    protocol: TCP  
    targetPort: 50000  
  selector:  
    app: jenkins  
```  
* **PV**  

```Groovy  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: pv-jenkins-data  
spec:  
  accessModes:  
  - ReadWriteOnce  
  capacity:  
    storage: 1Gi  
  hostPath:  
    path: /nfs/jenkins/data  
  persistentVolumeReclaimPolicy: Retain  
```  
* **PVC**  

```Groovy  
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: pvc-jenkins-data  
  namespace: cicd  
spec:  
  accessModes:  
  - ReadWriteOnce  
  resources:  
    requests:  
      storage: 1Gi  
```  
## 3.2部署Jenkins  
　　上述的资源配置文件内容可以写在一个文件中，也可以分开写，假设最终的资源配置文件为jenkins.yaml。则可以执行如下的命令在k8s集群中进行部署。  
```bash  
# cicd命名空间在部署gitlab时已经创建，这里不用再创建  
# kubectl create namespace cicd  
# 执行部署jenkins  
kubectl apply -f jenkins.yaml  
```  

　　因为我们已经将 Jenkins 的服务端口8080映射到宿主机上的32080了，Jenkins启动之后，其外部访问地址为：http://192.168.51.200:32080。  
　　在Kubernetes集群内部，Jenkins的服务名是jenkins，其所在的命名空间为cicd，则服务全名是jenkins.cicd.svc.cluster.local，进行跨命名空间访问时，服务名可以省略掉后缀，只保留 服务名+“ jenkins.cicd ”，所以在集群内部 Jnekins 的访问地址是：http://jenkins.cicd。  

## 3.3 初始化Jenkins  
　　访问jenkins的地址，可以看到jenkins正在启动中的提示，等待启动完成，进入解锁jenkins页面，页面上会提示存有解锁密码的文件，然后进入运行jenkins的容器中查找对应的解锁密码输入即可。  
![2019-04-02-17-06-57](http://img.zzl.yuandingsoft.com/blog/2019-04-02-17-06-57.png)  

```bash  
# 进入jenkins容器  
kubectl exec -it --namespace=cicd jenkins-7644854cd4-mvpvx /bin/bash  
#在容器内查看对应文件的内容即可。  
cat /var/jenkins_home/secrets/initialAdminPassword  

# 或者查看容器的运行日志，也可以看到解密密码  
kubectl logs --namespace=cicd jenkins-7644854cd4-mvpvx  
#解锁密码是一串数字和字母混合的的字符串，例如：b7818539ba1642c382510214256bc87b  
```  
　　解锁后，下一步选择“安装推荐的插件”，安装完成之后到达设置管理员用户的界面，可以设置一个新的管理员用户，也可以直接使用admin用户。  
![2019-04-02-17-14-10](http://img.zzl.yuandingsoft.com/blog/2019-04-02-17-14-10.png)  
![2019-04-02-17-45-27](http://img.zzl.yuandingsoft.com/blog/2019-04-02-17-45-27.png)  


　　最后再确认Jenkins的登录URL（该URL后续可以在配置中修改），最终出现“Jenkins已就绪”的界面，点击“开始使用jenkins”即可正式进入jenkins。  

## 3.4 安装Jenkins插件  
　　本环境中使用gitlab作为代码仓库，使用docker镜像，使用k8s集群，因此需要安装这三者相关的插件。如下：  

序号 | 名称 | 用途  
-----|-----|-----  
1 | Gitlab | 配置gitlab的代码仓库。  
2 | Gitlab hook | Gitlab的web hook，当代码仓库中代码有变化的时候，会自动触发jenkins构建任务。  
3 | Docker plugin | 与docker集成的插件  
4 | Docker build step | 允许使用docker命令作为构建的步骤。  
5 | CloudBees Docker Build and Publish | 允许构建docker镜像，并发布到镜像仓库中。  
6 | Kubernetes | 配置Kubernetes集群信息，允许jenkinss通过在kubernetes运行slave pod来执行部署任务。  
7 | kubernetes continuous deploy | 配置k8s集群的信息，允许自动在集群上发布k8s资源。  
8 | Kubernetes Cli | 允许可在jenkins中使用kubectl命令  

　　插件的安装步骤如下：“系统管理”→“插件管理”→“可选插件”（搜索选择对应的插件）→“直接安装”。  
　　插件安装完毕之后，后续在jenkins的任务中，可以使用对应的插件定义相关的步骤。  

## 3.5 配置基于Kubernetes的Slave节点  
　　配置基于Kubernetes的Slave节点，需要在Jenkins的系统配置中添加Kubernetes集群（对应的插件是Kubernetes），并配置POD模板，具体的配置方式如下：  
　　“系统管理”→“系统设置”→“云”→“添加Kuberenetes”。  
　　首先是配置一个云，类型选择kubernetes，这里面主要配置项如下图，主要配置的内容有四项，其他内容可以不变：  
　　**名称**：配置的是kubernetes集群的名称，自己取一个名字即可，后续在编写Jenkinsfile的时候会用到，用于指定Slave节点在哪个Kubernetes集群中运行。  
　　**Kubernetes地址**：配置的是Kubernetes集群的地址，因为我们这里的Jenkins是部署在Kubernetes集群中的，且这里添加的Kubernetes集群也是同一个，因此我们可以直接配置集群的地址，而不用配置任何证书和凭据。地址可以配置为：https://kubernetes.default （集群内部地址） 或者  https://192.168.51.200:6443 （集群外部地址）。需要注意的是，如果要添加其他的集群，则需要配置为对应集群的地址，并配置一个PKCS格式的凭据。具体方法后面会讲到。  
　　**Jenkins地址**：配置的是Jenkins Master服务器的地址，即jenkins自身的访问地址，用于Jenkins Slave与Master进行通讯，因为我们这里的Jenkins是部署在Kubernetes集群中的，且这里添加的Kubernetes集群也是同一个，因此我们可以配置为： http://jenkins.cicd.svc.default.local:8080 （集群内部地址） 或者 http://192.168.51.200:32080 （集群外部地址）。  
　　**容器数量**：这个值表示允许Kubernetes运行的最大并发运行代理程序容器数。如果设置为空，则表示没有限制。设置为0表示不启动任何容器。  
![2019-03-22-17-20-49](http://img.zzl.yuandingsoft.com/blog/2019-03-22-17-20-49.png)  

　　同时我们需要在该kubernetes中添加POD模板，即定义Jenkins Slave节点的镜像、挂载的盘等参数。  
　　通常情况下，我们只需要定义POD模板的名称，命名空间，和标签，标签在定义执行任务的Jenkins-slave节点中会用到。下面的容器列表中我们不用配置任何的容器，采用在最后配置一个“POD的原始yaml”的形式来实现；当然在这里通过图形界面配置也是可以的，配置的内容和“POD的原始yaml”文件的内容是对应的。  
　　**注意**：在POD模板中配置容器模板时，容器的名称不要设置为jnlp，因为默认会启动一个名为jnlp的容器（使用的镜像是jenkins/jnlp-slave:alpine，启动时运行/usr/local/bin/jenkins-slave命令），用于和Jenkins Master进行通讯，除非你要自定义这个jnlp容器（如，使用具备jnlp功能的其他镜像）。  
![2019-03-22-17-49-31](http://img.zzl.yuandingsoft.com/blog/2019-03-22-17-49-31.png)  
![2019-03-22-17-52-39](http://img.zzl.yuandingsoft.com/blog/2019-03-22-17-52-39.png)  

　　经过上述的配置，基于Kubernetes的jenkins slave就配置好了，后续我们在创建CI/CD流水线的时候，通过配置即可让构建任务运行在jenkins salve节点上，而非jenkins master节点。下一篇文章中将会讲解如何在构建任务中使用Jenkins Slave来执行构建任务。  

>　　结语：我们已经在kubernetes集群中安装部署了jenkins，并实现了jenkins数据的持续化，同时配置jenkins slave，jenkins的基础平台已经搭建完毕。后续我们将继利用jenkins和gitlab开始构建CI/CD自动化流水线。  
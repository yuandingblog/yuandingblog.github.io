---  
layout: post  
title: "业务容器化改造实践（6）"  
subtitle: 业务容器化改造的方案——构建自动的CI/CD发布流水线  
date: 2019-04-17  
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
>　　本篇是构建自动化CI/CD流水线方案的第三篇，主要介绍如何利用Jenkins和Gitlab构建自动化的发布流水线。  

# 一、CI/CD流水线的方案架构回顾  
　　我们对业务进行了微服务化拆分，并以容器的形式在Kubernetes集群中运行，那么必须有一套合适的自动CI/CD发布流水线来实现业务的持续集成和持续发布，以满足快速的业务上线和迭代，同时降低管理的成本。下面是我们设计的CI/CD流水线的方案架构图。  
![2019-02-28-10-02-15](http://img.zzl.yuandingsoft.com/blog/2019-02-28-10-02-15.png)  
　　从图中可以看到，涉及的组件主要有四个：gitlab代码仓库、Jenkins集成工具、Harbor私有镜像仓库、Kubernetes集群。所有的组件的部署方式在前文均已进行了详细的描述，本篇主要介绍如何利用Jenkins和Gitlab构建自动化的发布流水线。  
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


# 二、CI/CD流水线的实现  
## 2.1 手工发布过程  
　　针对常见的项目，在容器中部署运行，但不借用CI/CD流水线，采用纯人工执行的方式进行发布部署，通常会涉及三类角色：开发人员、运维人员、测试人员。  
　　开发人员需要做的是编写代码、提交到代码仓库，代码完成之后，进行编译、测试形成软件制品（如jar包、可执行程序等），甚至是直接形成软件的docker镜像。  
　　运维人员需要做的是制作软件的docker镜像（也可能是开发人员制作的），编写部署的配置文件，首先执行部署测试系统。当测试通过之后，可以上线，再部署或升级生产系统。  
　　测试系统部署完毕之后，测试人员对软件功能进行测试，测试通过之后，将结果反馈给项目组进行上线决策。  
　　假设，针对一个java项目（helloworld），我们可以将发布过程所做的步骤归纳如下：  
**1.提交代码、制作镜像所需的Dockerfile，以及部署到k8s集群中的YAML配置模板。**  
　　可能是开发人员提交代码，运维人员提交Dockerfile和YAML模板。  
```bash  
# 添加到暂存区  
git add .  
# 提交  
git commit -m  
# 推送到服务器端  
git push  
```  
**2.在具备编译条件的环境中克隆/拉取项目代码。**  
　　开发人员拉取源代码，准备编译。  
```bash  
# 环境中没有任何项目代码  
# git语法：  git clone http://gitlab-server/group/project.git  
git clone http://gitlab.yuandingit.com/group-java/helloworld.git  

# 环境中之前已经克隆过项目代码  
git pull  
```  
**3.使用maven编译java项目**  
　　开发人员将源代码编译成软件制品。  
　　编译之后，将形成一个target目录，其中有打包好的jar包，假设为 helloworld-1.0.jar。  
　　通常情况下，该jar包可以通过命令  java -jar helloworld-1.0.jar  执行启动运行，并且可通过网页访问到： http://ipaddr:8080/helloWorld 。  
```bash  
# maven编译时根据生命周期的不同，可以使用如下三种方式进行编译，均可以生成java项目的制品 JAR包。在使用时根据自己的需要选择其中一种即可。  
# mvn package完成项目的编译、单元测试、打包功能，但没有把打包好的可执行jar包部署到本地maven仓库和远程manven私服仓库。  
mvn clean package  
# mvn install 完成项目的编译、单元测试、打包功能，同时把打包好的可执行jar包部署到本地maven仓库，但没有部署到远程manven私服仓库。  
mvn clean install  
# mvn deploy 完成项目的编译、单元测试、打包功能，同时把打包好的可执行jar包部署到本地maven仓库和远程manven私服仓库。  
mvn clean deploy  
```  

**4.制作docker镜像**  
　　开发人员或运维人员制作docker镜像。  
　　编译完成之后已经获得了可以执行的jar包，接下来可以将该jar包打包到docker镜像中，形成可以运行该java项目的docker镜像。  
　　以下为一个简单的　Dockerfile 的写法，Dockerfile可以放在项目的根目录中。  
```bash  
# 以java:8作为基础镜像  
FROM java:8  
# 将项目的制品jar包添加到镜像中的一个目录中，假设是 /project  
ADD target/helloworld-1.0.jar /project/helloworld-1.0.jar  
# 启动容器时执行的命令，运行test.jar  
ENTRYPOINT ["java","-jar","/project/helloworld-1.0.jar"]  
```  
　　以下为制作docker镜像，并上传到私有镜像仓库（harbor.yuandingit.com）中的命令。  
```bash  
# 制作docker镜像。假设镜像名称为 harbor.yuandingit.com/test/helloworld:v1.0  
docker build -t harbor.yuandingit.com/test/helloworld:v1.0 .  
# 登录harbor私有镜像仓库，输入用户名和密码  
docker login harbor.yuandingit.com  
# 上传docker镜像  
docker push harbor.yuandingit.com/test/helloworld:v1.0  
```  

**5.部署软件**  
　　运维人员修改YAML模板，将其中的需要修改的配置项修改为本次部署时实际的配置项，可修改的配置项可能是镜像的版本、环境变量、资源限制等等，形成最终可以执行部署的YAML配置文件，最后在系统中执行部署软件，当容器正常启动、软件运行正常即可。  
　　假设YAML模板的名称是Template.yaml，内容如下：  
```yaml  
apiVersion: extensions/v1beta1  
kind: Deployment  
metadata:  
  name: helloworld  
  labels:  
    app: helloworld  
spec:  
  selector:  
    matchLabels:  
      app: helloworld  
  replicas: 2  
  template:  
    metadata:  
      labels:  
        app: helloworld  
    spec:  
      containers:  
      - name: helloworld  
        image: harbor.yuandingit.com/test/helloworld:v1.0.TAG_ID  
        imagePullPolicy: IfNotPresent  
        ports:  
        - containerPort: 8080  
          name: web  

---  
  apiVersion: v1  
  kind: Service  
  metadata:  
    name: helloword  
  spec:  
    selector:  
      app: helloword  
    type: NodePort  
    ports:  
    - port: 8080  
      targetPort: 8080  
      nodePort: 32180  
```  
　　修改YAML配置模板，并执行部署，部署成功后，可以通过“http://k8s-node-ip:32180/helloWorld”访问到。  
```bash  
# 替换YAML模板中的变量，下例是使用sed命令将文件中的 "TAG_ID" 替换为环境变量中的值 "${BUILD_ID}"。  
sed -i s/"TAG_ID"/"${BUILD_ID}"/g Template.yaml  

# 在可访问k8s集群的命令行中执行部署  
kubectl apply -f Template.yaml  
```  

## 2.2 发布流程自动化  
　　上述的手工发布过程可以通过一定的技术手段，形成自动化的流程，由机器代替人工来执行之前发布过程中的每一个步骤，将各个步骤从头到尾串联起来，形成自动化的发布流水线。而Jenkins软件就提供这个功能，使用Jenkins时，与手工方式不同的是，所需要提交到代码仓库中进行版本管理的文件，除了源代码之外，还有制作镜像的Dockerfile，执行部署到k8s集群中的Template.yaml配置模板，定义Jenkins流水线的配置文件Jenkinfile。  
　　下面我们来看在Jenkins中如何配置自动化的发布流水线。Jenkins中配置的流水线被称为Job，支持类型丰富的流水线配置，最常用的是“自由风格的软件项目”和“流水线”。  

### 2.2.1 自由风格的软件项目（freestyle）  
　　“自由风格的软件项目”可以通过图形界面配置所有的步骤，包含配置代码仓库源、触发器、环境、构建（Build）。如下图所示，在Build部分可以添加需要执行的步骤，比如执行shell命令、将文件发送到远程机器或到远程机器上执行shell命令、执行docker命令、构建docker镜像、部署到kubernetes等等，可以添加的步骤类型与安装的插件有关，最常见的一些步骤都已经具备，添加之后按照要求填写补充完成步骤内容即可。如下图中是添加了一个“执行shell”的步骤，可以将所需执行的shell命令写入。  
![2019-04-01-10-33-12](http://img.zzl.yuandingsoft.com/blog/2019-04-01-10-33-12.png)  

　　对于上述的手工发布过程中所执行的所有的命令，可以按照手工执行的步骤，将各个阶段的操作在任务（Job）中配置完成。除了本地提交代码的步骤是手工执行的之外，其他几个步骤均可以在jenkins的任务中进行配置。下面介绍各个步骤的配置方法。  

*  **新建任务，选择Jenkins-slave**  
　　新建一个任务（Job），类型选择“自由风格的软件项目”，在“General”中可以配置执行任务的jenkins-slave节点的标签，例如上篇文章中配置的jenkins-slave-pod标签，这样将在kubernetes集群中启动一对应的pod来执行jenkins的构建任务。  
　　需要注意的是，这里选择的Jenkins-slave可以启动，并且执行构建任务，但是只会在jenkins-slave默认的容器（jnlp）中执行，可能会出现构建步骤中用到的命令找不到的问题，此时需要使用针对本任务专门定制的jnlp镜像，以保证所有的构建步骤均可以成功。  
![2019-04-02-14-07-43](http://img.zzl.yuandingsoft.com/blog/2019-04-02-14-07-43.png)  

*  **配置项目源代码**  
　　在“Source Code Management”中，可以配置代码仓库的源地址，在执行构建的时候首先从该地址拉取所有的源代码。其中“Repository URL”填写仓库的地址，“Credentials”选择登陆代码仓库的凭据。  
　　如果登录gitlab的凭据还没有设置，可以点“Add”，在跳出的“添加凭据”页面添加，类型选择“用户名和密码”，然后填写用户名和密码，并设置ID和描述方便后续识别使用。  
![2019-04-02-14-08-12](http://img.zzl.yuandingsoft.com/blog/2019-04-02-14-08-12.png)  
![2019-04-03-09-35-25](http://img.zzl.yuandingsoft.com/blog/2019-04-03-09-35-25.png)  

*  **配置任务触发器**  
　　在“Build Triggers”中配置触发该构建任务的条件，可以配置为定时构建（轮询SCM也是定时构建），或者配置为自动触发构建。配置为Gitlab自动触发构建时，需要安装gitlab webhook插件，并需要将此处的 URL 配置到gitlab项目中的webhook中（gitlab中设置的路径为 进入项目→“设置”→“集成”，然后填写url，添加webhook，测试成功返回HTTP200即可）。  
![2019-04-03-09-20-58](http://img.zzl.yuandingsoft.com/blog/2019-04-03-09-20-58.png)  

*  **配置构建步骤--编译代码**  
　　在“Build”中添加任务执行所需的步骤，主要是三步：编译代码、制作镜像、部署到k8s集群。  
　　其中编译代码步骤，需要使用到的是maven编译，即执行mvn package命令进行编译，需要配置maven工具。配置路径为“系统管理”→“全局工具配置”→“Maven”→“Add Maven”，填写maven工具的名称，如maven-3.6.0，选择自动安装（不选的话，需要手工去安装），选择“从Apache安装”。后续使用到maven的时候会自动在执行任务的节点上安装maven。  
　　编译步骤中选择“调用顶层Maven目标”，其中Maven的版本选择全局工具配置中配置的maven，即 maven-3.6.0，目标（Goals）中填写要执行的命令，如果要执行“mvn package”就填写“package”。具体如下图所示：  
![2019-04-03-10-27-53](http://img.zzl.yuandingsoft.com/blog/2019-04-03-10-27-53.png)  
![2019-04-03-10-31-50](http://img.zzl.yuandingsoft.com/blog/2019-04-03-10-31-50.png)  

*  **配置构建步骤--构建镜像**  
　　其中制作镜像步骤，会根据源代码中带的Dockerfile制作docker镜像，并可以将制作好的镜像上传到私有的镜像仓库中，因此我们需要使用“Docker Build and Publish”（对应的插件是CloudBees Docker Build and Publish）。  
　　各项的填写方法为：  
　　**Repository Name**：镜像的名称，包含镜像仓库前缀，及镜像的名字。如：harbor.yuandingit.com/test/helloworld 。  
　　**Tag**：标签，镜像的标签。如：v1.0 。  
　　**Docker Host URI**：执行构建docker镜像的主机，如果是本机（执行构建任务的节点上有docker）执行，可以使用unix:///var/run/docker.sock ；如果是需要发送到远程的具备docker环境的主机上执行，则需要填写对方的IP地址和端口，通常是： tcp://ip:2376 ，需要对应的docker主机开放允许远程来执行镜像构建，这样具备一定的风险。  
　　**Docker Registry URL**：私有镜像仓库的地址，比如：http://harbor.yuandingit.com 。  
　　**Registry Credentials**：登录私有镜像仓库的凭据，即用户名和密码，点 ADD 可在jenkins中配置。  
![2019-04-03-11-30-03](http://img.zzl.yuandingsoft.com/blog/2019-04-03-11-30-03.png)  

*  **配置构建步骤--部署到k8s集群**  
　　其中部署到k8s集群步骤，会根据代码仓库中带的Template.yaml配置模板，首先替换变量，然后再执行部署。  
　　替换变量执行的是shell命令，使用sed命令替换YAML模板中的变量，本模板中只有一个变量（TAG_ID）需要替换为实际的值（${BUILD_ID}，该值为Jenkins的内置变量，是本次构建任务的次数），形成可最终部署的配置文件。  
![2019-04-03-11-32-41](http://img.zzl.yuandingsoft.com/blog/2019-04-03-11-32-41.png)  
　　要执行部署应用到k8s集群中，有多中方式，比如：远程到对应的k8s集群节点上执行kubectl命令部署（对应插件是 Publish Over SSH），本地配置kubeconfig文件作为客户端执行kubectl命令来部署，或者使用kubernetes continuous deploy插件来部署。  
　　下图是使用kubernetes continuous deploy插件来部署的配置，首先需要配置kubeconfig凭据，点Add在弹出的添加凭据页面可以添加，类型选择“kubernetes Configuration”，kubeconfig配置有三种方法：1.直接将kubeconfig文件（k8s集群的kubecofnig文件默认是k8s集群主节点操作系统的~/.kube/config 文件）的内容填写进来；2.选择Jenkins Master上的文件，需要事先把kubeconfig文件拷贝到Jenkins Master节点上；3.选择Kubernetes集群主节点上的kubeconfig文件。  
　　然后是配置需要执行部署的yaml配置文件，这里填写的是源代码中的相对路径，如：Template.yaml是放在源代码的根目录下，所以直接填写Template.yaml 。  
![2019-04-03-11-32-27](http://img.zzl.yuandingsoft.com/blog/2019-04-03-11-32-27.png)  
![2019-04-03-11-42-12](http://img.zzl.yuandingsoft.com/blog/2019-04-03-11-42-12.png)  

　　配置完成之后，在任务中点击“立即构建”可以进行测试，同时可以观察任务执行时的详细日志，如果有报错则构建任务将会中止，在日志中可以看出报错的步骤及原因；如果没有报错，则将会按照既定的步骤在k8s集群中部署hellworld的应用，同时可以通过网页“http://k8s-node-ip:32180/helloWorld”访问到。  

### 2.2.2 流水线（pipeline）  
　　在上面“自由风格的软件项目”中，可以看出，所有的任务步骤都是通过图形界面进行定义的，好处是配置较为直观、简单；但是缺点也很明显，任务的配置项无法进行版本的控制，也无法做到快速的跨Jenkins环境迁移。而使用流水线（Pipeline）方式的任务，是将运行任务的节点、执行的步骤等全部配置到Jenkinsfile文件中，Jenkinsfile可以放到代码仓库中实现版本的管理。在新建任务的时候，只需配置代码仓库，并选择Jenkinsfile即可；之后执行构建的时候将自动依照Jenkinsfile中定义好的步骤执行操作。  

*  **新建任务**  
　　新建一个任务（Job），类型选择“流水线（Pipeline）”。  

*  **配置任务触发器**  
　　在“Build Triggers”中配置触发该构建任务的条件，可以配置为定时构建（轮询SCM也是定时构建），或者配置为自动触发构建。配置为Gitlab自动触发构建时，需要安装gitlab webhook插件，并需要将此处的 URL 配置到gitlab项目中的webhook中（gitlab中设置的路径为 进入项目→“设置”→“集成”，然后填写url，添加webhook，测试成功返回HTTP200即可）。  
![2019-04-09-15-59-48](http://img.zzl.yuandingsoft.com/blog/2019-04-09-15-59-48.png)  

*  **配置流水线**  
　　在“Pipeline”中配置本任务的流水线，因为流水线的配置文件Jenkinsfile文件是放在代码仓库中的，因此这里需要配置代码仓库的地址，并选择相应的Jenkinsfile配置文件。  
　　下面是各个配置项的选择和填写方法：  
　　**Definition**：定义，确定本次任务的流水线配置文件的方式，一种是“Pipeline Script”，即在这里直接填写Jenkinsfile的内容；一种是“Pipeline Script from SCM”，即表明Jenkinsfile是来自于代码仓库的。因为我们的Jenkinsfile是放在代码仓库中管理的，所以选择“Pipeline Script from SCM”。  
　　**SCM**：源代码管理，表明代码仓库的类型，比如Git、SVN等等。  
　　**Repository URL**：代码仓库地址，填写源代码的代码仓库地址，注意：执行构建任务的Jenkins-slave节点必须能够访问并从该地址下载代码。  
　　**Credentials**：访问代码仓库的凭据，即登录代码仓库的用户名和密码。  
　　**Branch to Build**：Git代码仓库的分支，即构建时拉取代码的分支，默认是“Master”。  
　　**Script Path**：流水线脚本的路径，即Jenkinsfile的路径，该路径是相对于源代码的根目录的，假设配置文件名称为Jenkinsfile，且在源代码的最顶层，那么此处可以直接填写Jenkinsfile即可。  
![2019-04-09-16-01-54](http://img.zzl.yuandingsoft.com/blog/2019-04-09-16-01-54.png)  
　　**Pipeline Syntax**： Pipeline脚本的语法及代码片段生成器，如果不会写Jenkinsfile的具体内容，可以通过代码生成器将图形界面的配置生成代码片段，再将代码片段写入Jenkinsfile中即可。如下图，选择一个需要执行的步骤，比如“部署到K8S集群”，然后填写k8s集群的kubeconfig配置和最终要执行部署的Template.yaml配置文件，点击“Generate Pipeline Script”即可生成相关的代码片段。  
![2019-04-09-16-29-59](http://img.zzl.yuandingsoft.com/blog/2019-04-09-16-29-59.png)  

*  **Jenkinsfile的编写**  
　　以上为在Jenkins中配置“流水线（Pipeline）”任务的方法，这种方法需要用到定义任务执行步骤的Jenkinsfile，Jenkinsfile是的语法是基于Groovy语言的，分为脚本式和声明式两种脚本，我们用到的是声明式的脚本，其中定义了执行任务的Jenkins-slave节点的配置，以及执行构建任务的具体步骤和执行每个步骤所用的Jenkins-slave节点。具体的Jenkinsfile写法可以参考[官方文档](https://jenkins.io/doc/book/pipeline/)。  
　　下面是本次构建过程中用到的Jenkinsfile文件的内容，其中的agent就是要执行构建任务的Jenkins-slave节点，stages是需要执行的构建步骤。需要注意的是agent需要事先在Jenkins的系统管理中进行配置，详情参考[业务容器化改造的方案——在k8s中部署Jenkins持续集成工具](http://www.chilone.tk/2019/04/09/Practice-of-Business-Containerization-Transformation-(5)/)：  

```groovy  
pipeline {  
//声明执行构建任务的 agent，指明该agnet以POD的形式运行在名称为“k8s”的kubernetes集群中，agent的标签是“jenkins-salve-pod”  
  agent {  
    kubernetes {  
      cloud 'k8s'  
      label 'jenkins-slave-pod'  
      defaultContainer 'jnlp'  
      //agent pod的具体yaml配置，除了默认的jnlp容器之外，还定义了两个容器：maven和docker。  
      //maven容器具备mvn环境，可以编译java项目。  
      //docker容器具备docker环境，可以挂在宿主机的docker.sock，形成docker in docker环境，可以构建和上传docker镜像。  
      yaml """  
apiVersion: v1  
kind: Pod  
metadata:  
  labels:  
    app: jenkins-slave  
spec:  
  securityContext:  
    runAsUser: 0  
    fsGroup: 0  
  hostAliases:  
  - ip: 192.168.51.203  
    hostnames:  
    - harbor.yuandingit.com  
    - rancher.yuandingit.com  
  containers:  
  - name: maven  
    image: harbor.yuandingit.com/cicd/maven:3.6-jdk-13-alpine  
    imagePullPolicy: IfNotPresent  
    command: ["sleep","3600"]  
  - name: docker  
    image: harbor.yuandingit.com/cicd/docker:17.03.2-ce  
    imagePullPolicy: IfNotPresent  
    command: ["sleep","3600"]  
    volumeMounts:  
    - name: jenkinsdocker  
      mountPath: /var/run/docker.sock  
  volumes:  
  - name: jenkinsdocker  
    hostPath:  
      path: /var/run/docker.sock  
      type: Socket  
  serviceAccount: jenkins-admin  
  imagePullSecrets:  
  - name: harbor-yuandingit-com  
    """  
    }  
  }  

// 定义流水线具体执行的阶段（stage）和步骤（step），这里定义了五个阶段：“Clone Code”、“Replace Variable”，“Maven Compile”，“Build Images”，“Deploy To K8s”。  
// 每个阶段中定义了需要执行的步骤，每个步骤中定义了执行步骤的容器（container），以及具体执行的命令和内容。  
  stages {  
    stage('Clone Code') {  
      steps {  
        container('docker') {  
          git credentialsId: 'gitlab-zzl', url: 'http://gitlab.cicd/group-java/helloworld.git'  
        }  
      }  
    }  
    stage('Replace Variable') {  
      steps {  
        container('docker') {  
          sh 'sed -i s/"BUILD_ID"/"${BUILD_ID}"/g Template.yaml'  
        }  
      }  
    }  
    stage('Maven Compile') {  
      steps {  
        container('maven') {  
          sh 'mvn clean package'  
        }  
      }  
    }  
    stage('Build Images') {  
      steps {  
        container('docker') {  
          withDockerRegistry(credentialsId: 'harbor-admin', url: 'http://harbor.yuandingit.com') {  
            sh 'docker build -t harbor.yuandingit.com/test/helloworld:v1.0.$BUILD_ID .'  
            sh 'docker push harbor.yuandingit.com/test/helloworld:v1.0.$BUILD_ID'  
          }  
        }  
      }  
    }  
    stage('Deploy To K8s') {  
      steps {  
        container('docker') {  
          kubernetesDeploy configs: 'Template.yaml', kubeConfig: [path: ''], kubeconfigId: '51.200-kubeconfig', secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']  
        }  
      }  
    }  
  }  
}  
```  
　　配置完成之后，在任务中点击“立即构建”可以进行测试，同时可以观察任务执行时的详细日志，如果有报错则构建任务将会中止，在日志中可以看出报错的步骤及原因；如果没有报错，则将会按照既定的步骤在k8s集群中部署hellworld的应用，同时可以通过网页“http://k8s-node-ip:32180/helloWorld”访问到。  

# 三、总结  
　　上面介绍了利用Jenkins构建CI/CD流水线的两种方式：“自由风格的软件项目”和“流水线”，并通过实际的java项目案例进行了详细的配置介绍。  
　　下面对二者的特点进行对比分析，在使用时可根据实际的需要选择：  

对比点  | 自由风格的软件项目 | 流水线  
--- | --- | ---  
配置便捷度  | 图形界面配置，方便简单，但配置步骤较多。 | 脚本文件配置，需编写Jenkinsfile，技术要求高，但配置步骤极少。  
同Jenkins迁移  | 可复制任务，同时任务中的步骤可随意修改。 | 可复制任务，但任务中的步骤相同（因为是一个Jenkinsfile）；建议是配置不同的代码源/代码分支。  
跨Jenkins迁移 | 需要重新配置任务及其中的所有步骤。 | 只需重新配置任务，不用配置步骤。  
版本控制  | 无版本控制 | 通过代码仓库实现对Jenkinsfile的版本控制。  
Jenkins-slave支持 | 支持基于K8S的Jenkins-save，但构建任务只能在一个容器（默认的容器jnlp）中运行，可能需要定制容器镜像。 | 支持基于K8S的Jenkins-slave，构建任务的步骤可在不同的容器中运行，基本不需定制容器镜像。  

　　在实际应用中，针对同一个项目，如果是临时的构建测试，且构建测试的步骤多变，需要随时修改构建步骤，可以采用“自由风格的软件项目”的方式来配置CI/CD流水线。如果是长期使用的构建任务，且要求对配置进行严格的版本控制，那么可以采用“流水线”的方式来配置CI/CD流水线。  

>　　结语：我们在Kubernetes集群中[搭建Gitlab代码仓库](http://www.chilone.tk/2019/04/02/Practice-of-Business-Containerization-Transformation-(4)/)、[搭建Jenkins持续集成工具](http://www.chilone.tk/2019/04/09/Practice-of-Business-Containerization-Transformation-(5)/)，并且利用Jenkins部署CI/CD流水线，实现项目的持续集成和持续交付。相信通过这三篇文章的介绍，大家对在Kubernetes集群中部署和实现CI/CD流水线已经有了比较直观的感受，大家可以尝试按照此方法去部署自己的CI/CD流水线。  
>　　下一篇文章将以实际的项目案例，从整体上介绍电商平台如何实现容器化以及如何构建CI/CD流水线，敬请期待。  





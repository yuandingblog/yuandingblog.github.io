---  
layout: post  
title: "业务容器化改造实践（1）"  
subtitle: 业务容器化改造的思路与方案  
date: 2019-03-19  
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

>　　最近将一个电商业务系统进行了容器化改造，并在k8s平台中成功运行。在此记录我们的改造过程以及改造思路，希望在大家的业务容器化改造过程中，能够给大家带来帮助和启发。  
>　　后续将有一系列的文章来详细介绍如何进行容器化改造，需要改造哪些地方，具体是如何实施的。本篇是该系列文章的第一篇，主要介绍改造的思路和方案。  

# 一、为什么要进行业务容器化改造  
## 1.1 传统业务面临的问题  
　　世界著名大思想家斯宾塞·约翰逊曾经说过“唯一不变的是变化本身”，因此对于任何事物，我们都需要用动态的眼光去看待。在IT领域也是如此，对于一个业务系统来说，是否需要作出改变，主要取决于两个方面：一个业务的需求变化，这里的需求包含业务功能的需求、支撑更多用户访问量的需求、应用部署运维和升级的需求、业务开发管理的需求等；一个是技术的发展变化，即是否有新的技术出现能够解决之前不能解决的问题，或者能够让业务以更高效的状态运行。  
　　随着IT技术的发展，从最初的物理机发展出虚拟化技术，企业开始将自己的业务系统迁移到虚拟化平台中，后来出现了公有云、容器等最新的技术。毋庸置疑，这些新技术可以提升当前业务系统的运行效率，那么此时我们需要考虑的问题就变为如何应用这些新技术来解决业务的需求问题。  
　　从技术方面来讲，我们通常说的虚拟化技术，指的是主机层面的虚拟化技术，用于管理硬件层间的计算资源（CPU、内存）、存储资源（硬盘、存储）、网络资源等。使用虚拟化技术在一定程度上降低了运维的复杂性，提升了资源的使用率，这种虚拟化技术可视作云计算中IaaS层。传统的业务一般是放在物理机或者虚拟机上运行的，每个业务系统运行在一个或几个物理/虚拟机上，这样少量的物理机就能支撑所有的业务运行，在最初是可以满足业务运行需求的，但是随着技术发展，特别是受到互联网冲击的业务，将会面临如下的问题：  

* **系统基础组件和功能变多**  
　　由于互联网的冲击，用户的需求增多，且很快就发生变化，另外访问的流量也扩大了N倍。应用系统需要更多的功能来满足用户日益增长的需求，同时需要增加各种基础组件（如缓存、消息中间件等）来提升用户的访问体验，以及保证业务系统的业务连续性。因此不仅应用的功能增加了，而且系统架构也发生了很大的改变。  
* **应用敏捷开发管理困难**  
　　尤其是在互联网应用系统中，用户需求的变化也非常迅速的，原先一周一次更新，现在可能要一天一次，甚至一天多次更新，因此需要研发部门缩短软件的交付周期，给其带来了非常大的挑战。  
* **系统部署运维与升级复杂**  
　　应用系统规模越来越庞大，架构越来越复杂，使得应用的安装、部署和更新也相对比较复杂；运维人员需要处理大量的安装、部署和更新工作，工作量大大提高。因此运维所需投入的人力、物力、财力等成本都成倍增加。  
* **硬件资源分配调度缓慢**  
　　在硬件资源层面，物理机资源调度极其缓慢；即使是使用虚拟机，也需要手工部署应用，不仅耗时长，而且也无法保证开发、测试、生产环境的一致性，难以满足应用快速上线、弹性伸缩等需求。  

　　那么这些问题该如何解决呢？可以引入容器技术，使用容器云可以从本质上解决传统业务面临的问题，将业务进行容器化改造，从而更加灵活、高效、快速的满足业务系统需求。  

## 1.2 业务容器化的价值  
* **提供统一标准的交付环境**  
　　由于容器采用镜像的方式实现运行环境的标准化，只需要将应用程序及其运行所依赖的环境统一打包到镜像中，即可实现一次封装、到处运行的效果。因此可以屏蔽不同环境下的环境配置和安装部署等复杂过程。  
* **提升开发交付效率**  
　　容器的标准交付理念，支持更好的与CI/CD文化融合，开发提交代码后，编译、打包、发布、部署等过程都自动完成，提升开发交付的效率，可从技术手段上保证项目管理方式和管理理念的有效落地。  
* **资源整合提升利用率**  
　　容器可以运行在多种平台中，物理机、虚拟机、公有云等资源都可以运行容器，使用容器可以帮助企业实现对不同资源的统一管理。容器技术本质上是一种操作系统虚拟化技术，容器间共享操作系统的内核进程和内核资源，通常一个容器中只运行一个或一组进程，从而有效节省操作系统级资源开销，容器的启动速度快，占用资源少，通过容器密度的提升可以更好的利用资源。  
* **管理更加简单**  
　　使用容器技术，只需简单配置，就可以替代以往大量的部署、更新工作，而且所有的更新都是动态滚动更新的，也支持回滚操作，因此可以实现高效的自动化管理。  
* **生态系统完善**  
　　使用容器技术，配套的有专业的资源调度编排平台、企业级的镜像仓库、企业级的应用商店等功能，而且日志收集分析、应用监控等功能也都应有尽有。  

# 二、业务容器化改造的思路与方案  
## 2.1 业务容器化改造的思路  
　　为什么说容器技术可以从本质上解决传统业务面临的问题？  
　　我们在详细分析传统业务面临的问题之后，可以发现是由于业务的快速变化和访问量的增加，导致传统的架构无法满足需求。为了保证每个业务功能都能够长期稳定的提供服务，至少要做到两点：一点是当业务功能模块发生变化时，不会对其他业务模块造成影响，另外一点时业务本身能够应对访问量的高峰。也就是说，要对业务功能进行拆分，使其微服务化，这样各个功能模块相对独立，且不会因其他模块更新或故障而受到影响；另外就是业务模块自身要具备快速的伸缩能力，在访问高峰时使用更多的副本节点一同来提供服务，高峰过去之后，收缩副本节点数以节省资源。  
　　业务微服务化拆分之后，原先单独的业务系统变成了大量的微服务模块，每个模块可能又有大量的更新需求，使用传统人工的方式，必定带来大量的工作量，同时也难以控制升级发布是按照既定的规范进行的，管理者难以管理和监控发布情况。因此从管理的角度来看，需要采用自动化的发布方式来解决这个问题，同时将DevOps思想融入项目管理中。  
　　另外一点是，业务微服务化拆分以后，拥有大量的微服务模块，再加上大量的副本，运行微服务模块需要更多的资源，同时各个模块需要运行在什么主机上，各自之间的亲和、互斥关系该怎么处理，这些问题依靠人工是无法解决的。那么该如何运行微服务呢？如果选择物理机来运行，需要事先投入大量的资源。如果选择虚拟机来运行，每个微服务使用一个虚拟机，将会有大量的虚拟机需要管理，运维人员的工作量将非常的巨大。
　　因此我们需要一个具备完善资源调度、编排、故障隔离、高弹性的平台来运行微服务，而容器平台恰恰都具备这些特性。经实践证明，容器化是运行微服务的最佳载体。  
　　总结起来，业务容器化改造的思路主要是三个方面：**一是业务的微服务化拆分，二是业务发布流程的自动化，三是构建容器云管理平台**。  
## 2.2 业务容器化改造的方案  
　　业务容器化改造的方案与思路也是一一对应，即：进行业务微服务拆分、构建自动的CI/CD发布流水线、搭建容器管理平台。由此可以明确看出，改造后的系统需要以下的各种组件：  
* **容器云管理平台**  
　　目前最流行的容器技术是Docker，基于Docker并具备完善编排调度、故障隔离、弹性伸缩、滚动升级的容器云管理平台是Kubernetes（简称K8S）。原生K8S的管理控制主要依赖于命令行，图形化的管理方式并不友好，可以选用第三方的开源产品来管理K8S集群。  
　　Rancher是一款开源的企业级的容器云管理平台。通过Rancher，企业再也不用自己使用一系列的开源软件从头搭建容器服务平台，可以直接在Rancher中去创建K8S集群或者管理已有的K8S集群。目前支持创建自定义的集群，或者以阿里云、亚马逊云、VMware虚拟机等资源作为计算节点的集群，或者是管理Google容器云、AWS容器云、Azure容器云等公有云的K8S集群。  
* **镜像仓库**  
　　业务以容器方式运行时，每个容器对应的都有自己的Docker镜像，因此需要一个专门的镜像仓库来存储和管理Docker镜像。在互联网上国外的有DockerHub，国内的各大公有云厂商也提供免费的镜像仓库服务。在企业内部，可以使用一些可以进行私有部署的软件，比如开源的Harbor。  
　　Harbor是用于存储和分发Docker镜像的企业级Registry服务器，通过添加一些企业必需的功能特性，例如安全、标识和管理等，扩展了开源Docker Distribution。作为一个企业级私有Registry服务器，Harbor提供了更好的性能和安全，提升用户使用Registry构建和运行环境传输镜像的效率。Harbor支持安装在多个Registry节点的镜像资源复制，镜像全部保存在私有Registry中， 确保数据和知识产权在公司内部网络中管控。另外，Harbor也提供了高级的安全特性，诸如用户管理，访问控制和活动审计等。  
* **代码仓库**  
　　持续集成（CI）是一个开发的实践，开发人员需要定期集成代码到共享的存储库，这个存储库就是代码仓库。代码仓库可以实现项目代码的版本管理，同时在构建CI/CD流水线的时候代码仓库也是必须用到的一个组件。常用的代码管理工具有SVN和Git，SVN对应的服务器端是SVN Server，Git对应的服务器端有Gitlab、Github等。在企业内部，可以使用开源的Gitlab作为私有的代码仓库。  

* **CI/CD工具**  
　　常见的CI/CD工具有Jenkins、Gitlab等，二者都是开源的，使用最为广泛的是Jenkins，其中拥有丰富的插件，支持各种语言的项目在其上进行持续集成和持续交付。并且支持与Gitlab、SVN等代码仓库进行自动触发的配置，支持Docker、K8S插件，使用Jenkins的流水线，可编译源程序代码、将代码与依赖环境打包成镜像，并在k8s集群中进行部署。  
 
 
　　总结起来，在本次的业务容器化改造的方案中，所用到的软件工具如下，供参考：

序号 | 项目 | 工具
---------|----------|---------
 1 | 容器平台 | Rancher
 2 | 镜像仓库 | Harbor
 3 | 代码仓库 | Gitlab / SVN
 4 | CI/CD工具 | Jenkins

　　具体这些方案该如何设计呢？下篇文章见。  


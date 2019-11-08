---
layout: post
title: "Kubernetes：基于jenkins的CI/CD"
author: "zhangshun"
header-img: "img/background/25.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

##### 一、在Kubernetes 安装 Jenkins优点

目前很多公司采用Jenkins集群搭建复合需求的CI/CD流程，但是会存在一些问题<br>
- 主Master发生单点故障时，整个流程都不可用
- 每个Slave的环境配置不一样，来完成不同语言的编译打包，但是这些差异化的配置导致管理起来不方便，维护麻烦
- 资源分配不均衡，有的slave要运行的job出现排队等待，而有的salve处于空闲状态
- 资源有浪费，每台slave可能是物理机或者虚拟机，当slave处于空闲状态时，也不能完全释放掉资源
正因为上面的问题，我们需要采用一种更高效可靠的方式来完成这个CI/CD流程，而Docker虚拟化容器技能很好的解决这个痛点，又特别是在Kubernetes集群环境下面能够更好来解决上面的问题
![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/流程图.png)

如上图我们可以看到Jenkins master和Jenkins slave以Pod形式运行在Kubernetes集群的Node上，Master运行在其中一个节点，并且将其配置数据存储到一个volume上去，slave运行在各个节点上，但是它的状态并不是一直处于运行状态，它会按照需求动态的创建并自动删除

**这种方式流程大致为:** 当Jenkins Master接受到Build请求后，会根据配置的Label动态创建一个运行在Pod中的Jenkins Slave并注册到Master上，当运行完Job后，这个Slave会被注销并且这个Pod也会自动删除，恢复到最初的状态(这个策略可以设置)
- **服务高可用**，当Jenkins Master出现故障时，Kubernetes会自动创建一个新的Jenkins Master容器，并且将Volume分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用的作用
- **动态伸缩**，合理使用资源，每次运行Job时，会自动创建一个Jenkins Slave，Job完成后，Slave自动注销并删除容器，资源自动释放，并且Kubernetes会根据每个资源的使用情况，动态分配slave到空闲的节点上创建，降低出现因某节点资源利用率高，降低出现因某节点利用率高出现排队的情况
- **扩展性好**，当Kubernetes集群的资源严重不足导致Job排队等待时，可以很容器的添加一个Kubernetes Node到集群，从而实现扩展

##### 二、Kubernetes 安装Jenkins

**前提准备：**<br>
1.安装harbor镜像仓库<br>
2.jenkins-master需要volume存储，安装nfs<br>
3.使用nginx-ingress暴露jenkins的Web UI
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
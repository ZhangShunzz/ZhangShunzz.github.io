---
layout: post
title: "Kubernetes——Dashboard认证及分级授权"
subtitle: "Kubernetes Dashboard就是k8s集群的webui，集合了所有命令行可以操作的所有命令。"
author: "zhangshun"
header-img: "img/background/15.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

#### 部署

github地址：https://github.com/kubernetes/dashboard<br>
部署：<br>`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml`

将service改为NodePort<br>

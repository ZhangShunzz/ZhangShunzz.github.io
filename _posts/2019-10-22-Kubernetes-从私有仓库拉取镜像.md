---
layout: post
title: "Kubernetes：从私有仓库拉取镜像"
author: "zhangshun"
header-img: "img/background/8.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### 前提条件

- kubernetes集群
- harbor镜像仓库，本文harbor部署在kubernetes集群外
- 在node上修改/etc/docker/dameon.json，可以将harbor地址添加信任，无需https

### docker-compose方式安装harbor

下载安装包： `wget https://storage.googleapis.com/harbor-releases/harbor-offline-installer-v1.5.2.tgz`

解压： `tar zxf harbor-offline-installer-v1.5.2.tgz`


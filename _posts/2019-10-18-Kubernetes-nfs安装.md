---
layout: post
title: "Kubernetes：nfs安装"
subtitle: "Kubernetes中经常会用到存储类StorageClass，使用nfs存储卷非常简便(非生产环境)"
author: "zhangshun"
header-img: "img/background/14.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### 创建 NFS 服务器

NFS是网络文件系统(Network File System), 它允许系统将本地目录和文件共享给网络上的其他系统。通过 NFS，用户和应用程序可以访问远程系统上的文件，就象它们是本地文件一样。<br>
**安装**
NFS需要nfs-utils和rpcbind两个包, 但安装nfs-utils时会一起安装上rpcbind：<br>
`yum install nfs-utils`<br>
**编辑exports文件**

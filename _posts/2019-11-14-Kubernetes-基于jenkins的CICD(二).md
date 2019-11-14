---
layout: post
title: "Kubernetes：基于jenkins的CI/CD(二)"
author: "zhangshun"
header-img: "img/background/24.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

##### 一、 增加回滚功能

之前一篇文章介绍了[在kubernetes中进行简单的CICD](https://blog.zs-fighting.cn/2019/11/08/Kubernetes-%E5%9F%BA%E4%BA%8Ejenkins%E7%9A%84CICD(%E4%B8%80)/),但是没有回滚的功能，下面增加下回滚的功能

之前文章是在构建中的时候选择部署的环境，这次我们在构建前选择部署环境

创建一个pipeline风格的任务，在构建时选择参数化构建过程，这样我们就可以给任务传递一些参数

![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/参数化构建01.png)
![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/参数化构建02.png)
---
layout: post
title: "Kubernetes-RBAC以及认证方式"
subtitle: "基于角色的权限访问控制（Role-Based Access Control）作为传统访问控制（自主访问，强制访问）的有前景的代替受到广泛的关注。"
author: "zhangshun"
header-img: "img/background/14.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

#### Service Account


Service Account为Pod中的进程和外部用户提供身份信息。所有的kubernetes集群中账户分为两类，Kubernetes管理的serviceaccount(服务账户)和useraccount（用户账户）。<br>
比如说：dashboard以pod身份运行，需要设置一个ServiceAccount，并授予较大的权限。

![](/img/in-post/2019-09-05-Kubernetes-RBAC以及认证方式/ServiceAccount.png)

```
创建Service Account：
	命令行:
		kubectl create serviceaccount mysa --dry-run -o yaml
	yaml:
		apiVersion: v1
        kind: ServiceAccount
        metadata:
          creationTimestamp: null
          name: test
```

查看Service Account：`kubectl get serviceaccount --all-namespaces -o wide`<br>


#### 认证流程

1、认证<br>
2、授权检查<br>
3、准入控制（间接访问其他资源时)

认证方式：<br>
1、token<br>
2、ssl：kubectl跟api server交互时，双向认证

授权检查：<br>
1、RBAC：基于角色Role的授权检查
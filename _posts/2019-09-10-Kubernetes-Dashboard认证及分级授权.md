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
部署：<br>
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

将service改为NodePort
```shell
kubectl patch svc kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}' -n kube-system
```

#### 认证方式

Dashboard登陆时的两种认证方式:token、config<br>
认证时的账号必须为ServiceAccount：被dashboard pod拿来由kubernetes进行认证；

一、token
1. 查看dashboard的pod名称:
**kubectl get pods -n kube-system**
2. 查看dashboard pod使用的ServiceAccount:
**kubectl get pods kubernetes-dashboard-57df4db6b-9d6m5 -n kube-system -o yaml**
3. 查看dashboard pod使用的ServiceAccount的token名称:
**kubectl get secret -n kube-system**
4. 获取登陆dashboard的token信息:
**kubectl describe secret kubernetes-dashboard-token-4w656 -n kube-system**

二、config
需要创建一个dashboard config并导入<br>
1. 新建一个kubectl 集群并保存在config文件：
**kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/ssl/ca.pem --server="https://192.168.0.222:6443" --embed-certs=true --kubeconfig=/tmp/def-ns-admin.conf**
查看config文件的配置:
**kubectl config view --kubeconfig=/tmp/def-ns-admin.conf**
2. 新建一个ServiceAccount，并获取其token信息:
**kubectl create seriveaccount kubernetes-dashboard -n kube-system**
**kubectl describe secret kubernetes-dashboard-token-4w656 -n kube-system**
3. 新建kubectl config的用户:
**kubectl config set-credentials def-ns-admin --token=$TOKEN_INFO --kubeconfig=/tmp/def-ns-admin.conf**
4. 新建context:
**kubectl config set-context def-ns-admin@kubernetes --cluster=kubernetes --user=def-ns-admin --kubeconfig=/tmp/def-ns-admin.conf**
5. 设置当前context:
**kubectl config use-context def-ns-admin@kubernetes --kubeconfig=/tmp/def-ns-admin.conf**
6. 导出def-ns-admin.conf，并在登陆dashboard时导入即可

#### 如何只通过dashboard管理特定的namespace(default)？

`RoloBinding`可以将角色中定义的权限授予用户或用户组，`RoleBinding`包含一组权限列表(subjects)，权限列表中包含有不同形式的待授予权限资源类型(users、groups、service accounts)，`RoleBinding`适用于某个命名空间内授权，而 `ClusterRoleBinding`适用于集群范围内的授权。

`RoleBinding`同样可以引用`ClusterRole`来对当前 namespace 内用户、用户组或 ServiceAccount 进行授权，这种操作允许集群管理员在整个集群内定义一些通用的 ClusterRole，然后在不同的 namespace 中使用 RoleBinding 来引用

1、在对应的名称空间下，创建一个专有的ServiceAccount。<br>
`kubectl create serviceaccount def-ns-admin -n default`<br>
2、创建一个新的RoleBinding，将ClusterRole绑定到RoleBinding上并赋予ServiceAccount，**RoleBinding限制dashboard管理的namespace**。<br>
`kubectl create rolebinding def-ns-admin --clusterrole=admin --serviceaccount=default:def-ns-admin`<br>
3、查看token名称。<br>
`kubectl get secret -n default`<br>
4、查看token数据，并使用该token登陆。<br>
`kubectl describe secret def-ns-admin-token-d5f2d -n default`
#### patch打补丁

可以先定义好yaml文件，使用patch覆盖之前的配置<br>
```
kubectl patch clusterrole cluster-reader --type merge -p "`cat /tmp/patch.yaml`"
```
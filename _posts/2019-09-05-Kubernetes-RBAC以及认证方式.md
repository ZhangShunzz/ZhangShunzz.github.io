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
每生成一个ServiceAccount默认自带一个secret，使用secret与APIServer通信，secret里面包含认证信息token,通过`kubectl describe secret $SECRETNAME`可以查看到token。<br>
`Pod.spec.serviceAccountName: admin`，通过yaml文件来配置Pod的ServiceAccount。

#### kubectl是如何认证的

kubectl组件包括：clusters(集群)，users(用户)，contexts(上下文：表示哪个用户使用哪个集群)，current-context(当前使用的上下文)<br>
kubectl也是一个k8s传统对象，查看`kubectl的config：kubectl config view`<br>
![](/img/in-post/2019-09-05-Kubernetes-RBAC以及认证方式/Kubectl.png)

```
新建一个kubectl认证：
    1、签发新的证书：
        (umask 077;openssl genrsa -out zhangshun.key 2048)    #创建新的证书私钥
        openssl req -new -key zhangshun.key -out zhangshun.csr -subj "/CN=zhangshunzz"    #生成新的申请，subj必须与新的kubectl user相同
        openssl x509 -req -in zhangshun.csr -CA ../ssl/ca.pem -CAkey ../ssl/ca-key.pem -CAcreateserial -out zhangshun.crt -days 365    #使用kubernetes集群的ca证书签署
        openssl x509 -in zhangshun.crt -text -noout    #查看新证书的信息
    2、创建新的cluster：
        kubectl config set-cluster mycluster --server="https://192.168.0.222:6443" --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true    #--server：表示apiserver，--certificate-authority：表示连接apiserver的证书，--embed-certs：是否隐藏证书信息
    3、创建新的user(zhangshunzz)：
        kubectl config set-credentials zhangshunzz --client-certificate=/etc/kubernetes/test-ssl/zhangshun.crt --client-key=/etc/kubernetes/test-ssl/zhangshun.key --embed-certs=true    #--client-certificate：客户端证书，--client-key：客户端私钥
    4、创建contexts(上下文)：
        kubectl config set-context zhangshunzz@mycluster --cluster=kubernetes --user=zhangshunzz
    5、使用新的context：
        kubectl config use-context zhangshunzz@mycluster
```

#### 认证流程

1、认证<br>
2、授权检查<br>
3、准入控制（间接访问其他资源时)

认证方式：<br>
1、token<br>
2、ssl：kubectl跟api server交互时，双向认证

授权检查：<br>
1、RBAC：基于角色Role的授权检查

```
客户端-->API server
  user：username、uid
  group：
  extra：
  API
  Request path
    /apis/apps/v1/namespaces/default/deployments/myapp-deploy
    curl http://127.0.0.1:8888/apis/apps/v1/namespaces/default/deployments/nginx
  HTTP request verb:
    get,post,put,delete
  API request verb:
    get,list,create,update,patch,watch,proxy,redirect,delete
  Resource: pods,nodes,service
  Subresource:
  Namespace
  API group
```

当创建一个Pod时，默认在Volumes中挂载了default-token-xxxx的secret，里面包含连接APIServer的token，用于在当前名称空间(default)下的Pod连接APIServer并获取Pod自身相关属性。

#### 授权插件：RBAC

基于角色的权限访问控制（Role-Based Access Control）作为传统访问控制（自主访问，强制访问）的有前景的代替受到广泛的关注。<br>

RBAC包括Role、RoleBinding、ClusterRole、ClusterRoleBinding<br>

在Role中设置对资源(Pod、Service、Deployment等)的各种权限(Get、List、Watch、Delete等)，并通过RoleBinding绑定到用户资源(User、Group、ServiceAccount等)

**RoleBinding限制在名称空间级别，ClusterRoleBinding属于集群级别、可以操作所有名称空间。**<br>
**Role、ClusterRoleBinding绑定在哪个，就限制在哪个级别**

可以使用一个ClusterRole来绑定多个RoleBinding，也是限制在namespace级别下的，绑定哪个namespace就可以访问哪个，这样就不用每一个namespace下单独创建一个Role了。

创建Role<br>
```
命令行：
        kubectl create role pods-reader --verb=get,list,watch --resource=pods --dry-run -o yaml 
yaml:
		apiVersion: rbac.authorization.k8s.io/v1
		kind: Role
		metadata:
  			name: pods-reader
			namespace: default
		rules:
		- apiGroups:
  			- ""
  			resources:
  			- pods
			verbs:
			- get
			- list
			- watch
```

创建RoleBinding(即可通过zhuangshunzz用户使用kubectl访问当前名称空间下的pods,但是无法访问其他名称空间)<br>
```

命令行：
        kubectl create rolebinding zhangshun-read-pods --role=pods-reader --user=zhangshunzz --dry-run -o yaml
        kubectl create rolebinding zhangshun-read-pods --clusterrole=cluster-reader --user=zhangshunzz --dry-run -o yaml    #使用RoleBinding来绑定ClusterRole
yaml:
		apiVersion: rbac.authorization.k8s.io/v1
		kind: RoleBinding
		metadata:
		  name: zhangshun-read-pods
		  namespace: kube-system    #绑定在哪个名称空间，就在哪个名称空间生效!!
		roleRef:
		  apiGroup: rbac.authorization.k8s.io
		  kind: Role
  		  name: pods-reader
		subjects:
		- apiGroup: rbac.authorization.k8s.io
  		  kind: User
  		  name: zhangshunzz

```

创建ClusterRole<br>

```

命令行：
        kubectl create clusterrole cluster-reader --verb=get,list,watch --resource=pods --dry-run -o yaml
yaml:
		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRole
		metadata:
		  name: cluster-reader
		rules:
		- apiGroups:
		  - ""
		  resources:
		  - pods
		  verbs:
		  - get
		  - list
		  - watch

```

创建ClusterRoleBinding(集群中所有的pods资源都可以访问)<br>

```

命令行：
        kubectl create clusterrolebinding zhangshun-read-all-pods --cluster-role=cluster-reader --user=zhangshunzz --dry-run -o yaml
yaml：
		apiVersion: rbac.authorization.k8s.io/v1beta1
		kind: ClusterRoleBinding
		metadata:
		  creationTimestamp: null
		  name: zhangshun-read-all-pods
		roleRef:
		  apiGroup: rbac.authorization.k8s.io
		  kind: ClusterRole
		  name: cluster-reader
		subjects:
		- apiGroup: rbac.authorization.k8s.io
		  kind: User
		  name: zhangshunzz

```
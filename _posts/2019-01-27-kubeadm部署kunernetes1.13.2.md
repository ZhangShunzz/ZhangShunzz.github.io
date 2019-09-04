---
layout: post
title:  kubeadm部署kubernetes1.13.2
date:   2019-01-27 09:55:00 +0800
categories: 技术
tag: Kubernetes
---

1、配置解析

```
[root@master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.0.8	master
192.168.0.9	node1
192.168.0.10	node2
```

2、禁用防火墙，安装iptables,  并安装ntp时间同步

```
systemctl stop firewalld

systemctl disable firewalld

setenforce 0

iptables -F

iptables -t nat -F

iptables -I FORWARD -s 0.0.0.0/0 -d 0.0.0.0/0 -j ACCEPT  

yum -y install ntp

ntpdate pool.ntp.org

systemctl start ntpd

systemctl enable ntpd
```

3、修改内核参数,关闭selinux

```
vim /etc/sysctl.conf

net.ipv4.ip_forward=1

net.bridge.bridge-nf-call-ip6tables = 1

net.bridge.bridge-nf-call-iptables = 1

net.bridge.bridge-nf-call-arptables = 1

vm.swappiness=0

关闭swap

swapoff -a

vim /etc/selinux/config

SELINUX=disabled

sysctl -p
```

4、安装docker,kubernetes组件，更新yum源

```
wget -P /etc/yum.repos.d  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

vim /etc/yum.repos.d/kubernetes.repo

[kubernetes]

name=Kubernetes Repo

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/

gpgcheck=1

gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

enabled=1

yum clean all && yum makecache && yum update
 
yum install docker-ce-18.06.1.ce  -y

systemctl start docker

systemctl enable docker

systemctl status docker

yum -y install kubeadm-1.13.2 kubectl-1.13.2 kubelet-1.13.2 kubernetes-cni-0.6.0 

systemctl start kubelet

systemctl enable kubelet.service

```

5、kubernetes需要的镜像

```
[root@master ~]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-controller-manager   v1.13.2             b9027a78d94c        2 weeks ago         146MB
k8s.gcr.io/kube-proxy                v1.13.2             01cfa56edcfc        2 weeks ago         80.3MB
k8s.gcr.io/kube-apiserver            v1.13.2             177db4b8e93a        2 weeks ago         181MB
k8s.gcr.io/kube-scheduler            v1.13.2             3193be46e0b3        2 weeks ago         79.6MB
k8s.gcr.io/coredns                   1.2.6               f59dcacceff4        2 months ago        40MB
k8s.gcr.io/etcd                      3.2.24              3cab8e1b9802        4 months ago        220MB
quay.io/coreos/flannel               v0.10.0-amd64       f0fad859c909        12 months ago       44.6MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        13 months ago       742kB
```

6、初始化master

```
kubeadm init \

  --kubernetes-version=v1.13.2 \

  --pod-network-cidr=10.244.0.0/16 \

  --apiserver-advertise-address=192.168.0.8

把token复制保存下来，后面添加Node节点需要使用

kubeadm join 192.168.0.8:6443 --token 588yx1.z8xigkck03tvvqb3 --discovery-token-ca-cert-hash sha256:3316ceefa90b85592f1acdecdc2fbb858415c9c4cb35c3fdf27d31b694894237
```

7、在master继续执行以下步骤：

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf

如果安装失败，需要重装时。可以使用如下命令来清理环境:

kubeadm reset
```

8、安装flannel作为Pod网络插件

```
[root@master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

9、node端镜像

```
[root@node1 ~]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy    v1.13.2             01cfa56edcfc        2 weeks ago         80.3MB
k8s.gcr.io/coredns       1.2.6               f59dcacceff4        2 months ago        40MB
k8s.gcr.io/pause         3.1                 da86e6ba6ca1        13 months ago       742kB
```

10、node安装组件,并添加集群

```
yum -y install kubeadm-1.13.2 kubectl-1.13.2 kubelet-1.13.2 kubernetes-cni-0.6.0

systemctl start kubelet

systemctl enable kubelet

systemctl status kubelet

systemctl status kubelet

[root@node1 ~]# kubeadm join 192.168.0.8:6443 --token 588yx1.z8xigkck03tvvqb3 --discovery-token-ca-cert-hash sha256:3316ceefa90b85592f1acdecdc2fbb858415c9c4cb35c3fdf27d31b694894237
```

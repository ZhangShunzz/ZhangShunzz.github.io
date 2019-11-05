---
layout: post
title: "Kubernetes：DNS的安装部署测试"
author: "zhangshun"
header-img: "img/background/26.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### kubectl dns 的安装

##### 1.1 在官网下载配置文件

官网：`https://github.com/kubernetes/kubernetes/`

具体路径是cluster/addons/dns/kube-dns

可能版本不一样，路径略有不同<br>
该路径下有三个相似的配置文件：<br>
- kube-dns.yaml.base  
- kube-dns.yaml.in  
- kube-dns.yaml.sed <br>
在此，我们使用kube-dns.yaml.sed配置文件作为模板；

需要提前准备镜像：
- k8s.gcr.io/k8s-dns-kube-dns:1.14.13
- k8s.gcr.io/k8s-dns-dnsmasq-nanny:1.14.13
- k8s.gcr.io/k8s-dns-sidecar:1.14.13

##### 1.2 需要修改其中的三个属性

`mv kube-dns.yaml.sed kube-dns.yaml`<br>
`sed -i 's/$DNS_SERVER_IP/10.10.10.2/g' kube-dns.yaml`<br>
`sed -i 's/$DNS_DOMAIN/cluster.local/g' kube-dns.yaml`<br>
`sed -i 's/$DNS_MEMORY_LIMIT/1000Mi/g' kube-dns.yaml`<br>

##### 1.3 创建dns服务

**kubectl create -f kube-dns.yaml**


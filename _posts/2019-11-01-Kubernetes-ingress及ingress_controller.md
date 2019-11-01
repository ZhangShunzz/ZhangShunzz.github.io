---
layout: post
title: "Kubernetes：ingress及ingress controller"
author: "zhangshun"
header-img: "img/background/28.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### Ingress Controller
定义Ingress并注入到Ingress Controlle中

ingress-nginx-controller是一个运行nginx服务的pod

请求先经过ingress-nginx的service，然后将请求交给ingress-nginx-controller，ingress-nginx-controller通过ingress中的配置，将流量转发到对应的Headless Service，再由Headless Service转发到pod中
![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/ingress请求流程图.png)
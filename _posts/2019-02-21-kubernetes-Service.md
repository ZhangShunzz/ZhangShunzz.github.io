---
layout: post
title:  k8s Service
date:   2019-02-21 21:43:00 +0800
categories: 技术
tag: Kubernetes
---

### Service

Service从逻辑上代表了一组Pod，具体是哪些Pod则是由label来挑选的。Service有自己的IP，而且这个IP是不变的。客户端只需要访问Service的IP，Kubernetes则负责建立和维护Service与Pod的映射关系

Service的Cluster-IP由Kubernetes节点上的iptables规则管理的，也可以说是**Service逻辑上的抽象层**

Kubernetes提供了多种类型的Service，默认是ClusterIP

- ClusterIP

Service通过Cluster内部的IP对外提供服务，只有Cluster内的节点和Pod可访问，这是默认的Service类型

- NodePort

Service通过Cluster节点的静态端口对外提供服务。Cluster外部可以通过<NodeIP>:<NodePort>访问Service

- LoadBalancer

Service利用cloud  provider特有的load  balancer对外提供服务，cloudprovider负责将load balancer的流量导向Service。目前支持的cloud provider有GCP、AWS、Azur等

```
apiVersion: v1  
kind: Service
metadata:
  name: httpd-svc
  namespace: kube-public  ③
spec:
  type: NodePort  ④
  selector:
    run: httpd  ①
  ports:  ②
  - protocol: TCP  
    nodePort: 30000  
    port: 8080
    targetPort: 80

①selector指明挑选那些label为run: httpd的Pod作为Service的后端
②将Node的30000端口映射到Service的8080端口，将Service的8080端口映射到Pod的80端口，使用TCP协议
③通过namespace: kube-public指定资源所属的namespace
④Service的类型，ClusterIP类型只能提供内部访问，NodePort类型可以对外提供服务
```
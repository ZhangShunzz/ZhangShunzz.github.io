---
layout: post
title: "Kubernetes：Prometheus Operator手动安装"
author: "zhangshun"
header-img: "img/background/19.jpg"
header-mask: 0.2
tags:
  - Kubernetes
  - Prometheus
---

### 一、prometheus-operator 介绍和功能
**prometheus-operator 介绍**

---

当今Cloud Native概念流行，对于容器、服务、节点以及集群的监控变得越来越重要。

Prometheus 作为 Kubernetes 监控的事实标准，有着强大的功能和良好的生态。但是它不支持分布式，不支持数据导入、导出，不支持通过 API 修改监控目标和报警规则，所以在使用它时，通常需要写脚本和代码来简化操作。

Prometheus Operator 为监控 Kubernetes service、deployment、daemonsets 和 Prometheus 实例的管理提供了简单的定义等，简化在 Kubernetes 上部署、管理和运行 Prometheus 和 Alertmanager 集群。

**prometheus-operator 功能**

---

1. `创建/销毁`:在 Kubernetes namespace 中更加容易地启动一个 Prometheues 实例，一个特定应用程序或者团队可以更容易使用 Prometheus Operator。
2. `便捷配置`:通过 Kubernetes 资源配置 Prometheus 的基本信息，比如版本、存储、副本集等。
3. `通过标签标记目标服务`:基于常见的 Kubernetes label 查询自动生成监控目标配置；不需要学习 Prometheus 特定的配置语言。

### 二、下载 prometheus-operator 配置

**下载官方prometheus-operator v0.29.0版本代码，官方把所有文件都放在一起，这里我分类下**

```shell
git clone https://github.com/coreos/prometheus-operator.git -b release-0.29
cd contrib/kube-prometheus/manifests/

# 新建对应的服务目录
$ mkdir -p operator node-exporter alertmanager grafana kube-state-metrics prometheus serviceMonitor adapter

# 把对应的服务配置文件移动到相应的服务目录
$ mv *-serviceMonitor* serviceMonitor/
$ mv 0prometheus-operator* operator/
$ mv grafana-* grafana/
$ mv kube-state-metrics-* kube-state-metrics/
$ mv alertmanager-* alertmanager/
$ mv node-exporter-* node-exporter/
$ mv prometheus-adapter* adapter/
$ mv prometheus-* prometheus/

# 新创建了两个目录，存放钉钉配置和其它配置
mkdir other dingtalk-hook

# 二进制部署k8s管理组件和新版本kubeadm部署的都会发现在prometheus server的页面上发现kube-controller和kube-schedule的target为0/0。
cat>>other/kube-controller-manager-svc-ep.yaml<<EOF
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: kube-system
subsets:
- addresses:
  - ip: 172.16.3.9   # master ip 地址
  ports:
  - name: http-metrics
    port: 10252
    protocol: TCP
EOF

cat>>other/kube-scheduler-svc-ep.yaml<<EOF
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
subsets:
- addresses:
  - ip: 172.16.3.9 # master ip 地址
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP
EOF

cat>>other/prometheus-additional.yaml<<EOF
- job_name: 'test'
  # Override the global default and scrape targets from this job every 5 seconds.
  scrape_interval: 5m
  metrics_path: '/metrics'
  static_configs:
    - targets: ['test.example.com']
    labels:
      group: 'dev'
EOF
```

### 三、部署operator

**默认镜像下载需要翻墙，下面是提供我个人的dockerhub地址**

---

|  官方镜像地址	   | 个人 DockerHub 地址  |
|  ----  | ----  |
| quay.io/coreos/configmap-reload:v0.0.1	  | zhangshunzz/configmap-reload:v0.0.1 |
| quay.io/coreos/k8s-prometheus-adapter-amd64:v0.4.1	  | zhangshunzz/k8s-prometheus-adapter-amd64:v0.4.1 |
| quay.io/coreos/kube-rbac-proxy:v0.4.1	  | zhangshunzz/kube-rbac-proxy:v0.4.1 |
| quay.io/coreos/kube-state-metrics:v1.5.0	  | zhangshunzz/kube-state-metrics:v1.5.0 |
| quay.io/prometheus/prometheus:v2.5.0	  | zhangshunzz/prometheus:v2.5.0 |
| quay.io/prometheus/node-exporter:v0.17.0	  | zhangshunzz/node-exporter:v0.17.0 |
| quay.io/coreos/prometheus-operator:v0.29.0	  | zhangshunzz/prometheus-operator:v0.29.0 |
| quay.io/coreos/prometheus-config-reloader:v0.29.0	  | zhangshunzz/prometheus-config-reloader:v0.29.0 |
| quay.io/prometheus/prometheus:v2.7.2	  | zhangshunzz/prometheus:v2.7.2 |
| quay.io/prometheus/alertmanager:v0.16.1	  | zhangshunzz/alertmanager:v0.16.1 |
| k8s.gcr.io/addon-resizer:1.8.4	  | zhangshunzz/addon-resizer:1.8.4 |

**首先创建namespace monitoring**

---

`kubectl apply -f 00namespace-namespace.yaml`

**部署operator**
---

`kubectl apply -f operator/`

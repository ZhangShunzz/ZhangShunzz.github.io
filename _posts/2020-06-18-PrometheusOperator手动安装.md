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

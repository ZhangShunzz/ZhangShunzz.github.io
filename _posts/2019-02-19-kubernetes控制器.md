---
layout: post
title:  k8s 控制器
date:   2019-02-19 13:41:00 +0800
categories: 技术
tag: Kubernetes
---

* content
{:toc}


### Deployment

- **运行Deployment**

```
kubectl run nginx-deployment --image=nginx:1.7.9 --replicas=2

通过kubectl get deployment命令查看nginx-deployment的状态，输出显示两个副本正常运行

接下来我们用kubectl describe deployment了解更详细的信息

用kubectl describe replicaset查看详细信息
```

![deployment]({{ '/styles/images/k8s/k8s_deployment过程.png' | prepend: site.baseurl  }})

- **伸缩**

伸缩是指在线增加或减少Pod的副本数

修改demo.yaml中replicas的数量，就可以增加或者减少副本数量


- **用label控制Pod的位置**

默认配置下，Scheduler会将Pod调度到所有可用的Node。不过有些情况我们希望将Pod部署到指定的Node，比如将有大量磁盘I/O的Pod部署到配置了SSD的Node

Kubernetes是通过label来实现这个功能的

label是key-value对，各种资源都可以设置label，灵活添加各种自定义属性

比如执行如下命令标注k8s-node1是配置了SSD的节点

`kubectl label node k8s-node1 disktype=ssd`

然后通过kubectl get node --show-labels查看节点的label

有了disktype这个自定义label，接下来就可以指定将Pod部署到k8s-node1

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zzz
spec:
  replicas: 7
  template:
    metadata:
      labels:
        app: web_server
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
      nodeSelector:
        disktype: ssd
```

要删除node上的label，执行如下命令：

`kubectl label node k8s-node1 disktype-`

### DaemonSet

**Deployment部署的副本Pod会分布在各个Node上，每个Node都可能运行好几个副本。DaemonSet的不同之处在于：每个Node上最多只能运行一个副本。**

Kubernetes自己就在用DaemonSet运行系统组件

执行`kubectl get daemonset --namespace=kube-system`返回系统所用的daemonset

kube-flannel-ds和kube-proxy分别负责在每个节点上运行flannel和kube-proxy组件

- **运行自己的DaemonSet**

![daemonset]({{ '/styles/images/k8s/k8s_daemonset.png' | prepend: site.baseurl  }})

① 直接使用Host的网络

② 设置容器启动命令

③ 通过Volume将Host路径/proc、/sys和/映射到容器中

执行`kubectl apply -f demo.yaml`

node-exporter-daemonset部署成功，k8s-node1和k8s-node2上分别运行了一个node exporter Pod

### Job or CronJob

容器按照持续运行的时间可分为两类：服务类容器和工作类容器

服务类容器通常持续提供服务，需要一直运行，比如HTTP  Server、Daemon等。

工作类容器则是一次性任务，比如批处理程序，完成后容器就退出

Kubernetes的Deployment、ReplicaSet和DaemonSet都用于管理服务类容器；对于工作类容器，我们使用Job。

- **Job**

```
apiVersion: batch/v1  ①
kind: Job  ②
metadata:
  name: myjob
spec:
  completions: 10  ⑤
  parallelism: 2  ④
  template:
    metadata:
      name: myjob
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo","hello k8s job! "]
      restartPolicy: Never  ③

①batch/v1是当前Job的apiVersion
②指明当前资源的类型为Job
③restartPolicy指定什么情况下需要重启容器。对于Job，只能设置为Never或者OnFailure，OnFailure指pod启动失败后会尝试重启。对于其他controller（比如Deployment），可以设置为Always
④paraallelism可以设置同时运行多个Pod，提高Job的执行效率
⑤completions设置job的运行总数，每次运行两个Pod，直到总共有10个Pod成功完成
```

- **CronJob**

```
apiVersion: batch/v1beta1  ①
kind: CronJob  ②
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"  ③
  jobTemplate:  ④
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo","hello k8s cronjob!"]
          restartPolicy: OnFailure

① batch/v1beta1是当前CronJob的apiVersion
②指明当前资源的类型为CronJob
③ schedule指定什么时候运行Job，其格式与Linux cron一致。这里*/1 * * * *的含义是每一分钟启动一次
④ jobTemplate定义Job的模板，格式与前面的Job一致
```

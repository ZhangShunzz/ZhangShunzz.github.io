---
layout: post
title: "Kubernetes：ingress及ingress controller"
author: "zhangshun"
header-img: "img/background/28.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### ~~一段复制的nginx-ingress介绍~~

nginx-ingress 和 traefik 都是比如热门的 ingress-controller，作为反向代理将外部流量导入集群内部，将 Kubernetes 内部的 Service 暴露给外部，在 Ingress 对象中通过域名匹配 Service，这样就可以直接通过域名访问到集群内部的服务了。相对于 traefik 来说，nginx-ingress 性能更加优秀，但是配置比 traefik 要稍微复杂一点，当然功能也要强大一些，支持的功能多许多，今天为大家介绍下 nginx-ingress 在 Kubernetes 中的安装使用。

### Ingress Controller
定义Ingress并注入到Ingress Controlle中

nginx-ingress-controller是一个运行nginx服务的pod

请求先经过ingress-nginx的service，然后将请求交给nginx-ingress-controller，nginx-ingress-controller通过ingress中的配置，将流量转发到对应的Headless Service，再由Headless Service转发到pod中
![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/ingress请求流程图.png)

### ingress安装

命令：`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml`<br>
镜像：`quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1`

需要额外创建一个service，用来接受外部流量，这里使用nodeport模式，生产环境下可以用k8s外的loadblance代理到该service
```
[root@master ingress-nginx]# cat ingress-svc-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: NodePort
  selector:
    app: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
    nodePort: 30443
```

**安装时遇到的报错**
1、报错：
```
[root@master ingress-nginx]# kubectl get all -n ingress-nginx
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   0/1     0            0           15s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-5456b94498   1         0         0       15s
[root@master ingress-nginx]# kubectl describe replicaset nginx-ingress-controller-5456b94498 -n ingress-nginx
Error creating: pods "nginx-ingress-controller-5456b94498-pgqfz" is forbidden: SecurityContext.RunAsUser is forbidden
```
原因：apiserver开启了SecurityContextDeny准入控制器，将拒绝任何试图设置某些升级的SecurityContext字段的pod<br>
解决办法：修改apiserver的配置文件
```
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction
SecurityContextDeny 不enable就行
```
2、报错：
```
[root@master ingress-nginx]# kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-5456b94498-d58jl   0/1     Pending   0          3m1s
[root@master ingress-nginx]# kubectl logs nginx-ingress-controller-5456b94498-d58jl -n ingress-nginx
[root@master ingress-nginx]# kubectl describe pods nginx-ingress-controller-5456b94498-d58jl -n ingress-nginx
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  3m13s (x3 over 3m25s)  default-scheduler  0/2 nodes are available: 2 node(s) didn't match node selector.
```
原因：查看mandatory.yaml发现,nodeselector有配置标签选择器。
解决办法：node节点添加标签
`kubectl label nodes 192.168.0.223 kubernetes.io/os=linux`

### DashBoard查看Ingress状态

![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/dashboard-ingress-nginx.png)

### 测试
创建后端service跟pod
```
apiVersion: v1
kind: Service
metadata:
  name: web-test
  namespace: int
spec:
  type: ClusterIP
  selector:
    app: web-test
  ports:
  - name: http
    targetPort: 8080
    port: 8080
  clusterIP: None
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: web-test
  name: web-test
  namespace: int
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-test
  template:
    metadata:
      labels:
        app: web-test
    spec:
      containers:
      - image: 192.168.0.109/zzc_int/tomcat-web-test:0.0.1
        imagePullPolicy: IfNotPresent
        name: web-test
        ports:
        - containerPort: 8080
          protocol: TCP
      restartPolicy: Always
      schedulerName: default-scheduler
      imagePullSecrets:
      - name: harbor-key
```
创建ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-test-ingress
  namespace: int
  annotations:
    kubernetes.io/ingress.class: "nginx"    //表示使用nginx-ingress
spec:
  rules:
  - host: web-test.zs.com    //虚拟主机
    http:
      paths:
      - path:    //url
        backend:    //后端对应的Service
          serviceName: web-test
          servicePort: 8080
```

![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/http_test_01.png)
![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/http_test_02.png)

### https测试

创建secret：
`kubectl create secret tls web-test-ingress-secret --cert=intellicre.crt --key=intellicredit.cn.key -n int`

创建ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-test-https
  namespace: int
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:    //哪些虚拟主机使用https
    - web-test.intellicredit.cn
    secretName: web-test-ingress-secret    //证书secret
  rules:
  - host: web-test.intellicredit.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: web-test
          servicePort: 8080
```

![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/https_test_01.png)
![](/img/in-post/2019-11-01-Kubernetes-ingress及ingress_controller/https_test_02.png)
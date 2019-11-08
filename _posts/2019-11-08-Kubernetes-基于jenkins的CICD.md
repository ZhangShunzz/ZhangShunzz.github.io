---
layout: post
title: "Kubernetes：基于jenkins的CI/CD"
author: "zhangshun"
header-img: "img/background/25.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

##### 一、在Kubernetes 安装 Jenkins优点

目前很多公司采用Jenkins集群搭建复合需求的CI/CD流程，但是会存在一些问题<br>
- 主Master发生单点故障时，整个流程都不可用
- 每个Slave的环境配置不一样，来完成不同语言的编译打包，但是这些差异化的配置导致管理起来不方便，维护麻烦
- 资源分配不均衡，有的slave要运行的job出现排队等待，而有的salve处于空闲状态
- 资源有浪费，每台slave可能是物理机或者虚拟机，当slave处于空闲状态时，也不能完全释放掉资源
正因为上面的问题，我们需要采用一种更高效可靠的方式来完成这个CI/CD流程，而Docker虚拟化容器技能很好的解决这个痛点，又特别是在Kubernetes集群环境下面能够更好来解决上面的问题
![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/流程图.png)

如上图我们可以看到Jenkins master和Jenkins slave以Pod形式运行在Kubernetes集群的Node上，Master运行在其中一个节点，并且将其配置数据存储到一个volume上去，slave运行在各个节点上，但是它的状态并不是一直处于运行状态，它会按照需求动态的创建并自动删除

**这种方式流程大致为:** 当Jenkins Master接受到Build请求后，会根据配置的Label动态创建一个运行在Pod中的Jenkins Slave并注册到Master上，当运行完Job后，这个Slave会被注销并且这个Pod也会自动删除，恢复到最初的状态(这个策略可以设置)
- **服务高可用**，当Jenkins Master出现故障时，Kubernetes会自动创建一个新的Jenkins Master容器，并且将Volume分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用的作用
- **动态伸缩**，合理使用资源，每次运行Job时，会自动创建一个Jenkins Slave，Job完成后，Slave自动注销并删除容器，资源自动释放，并且Kubernetes会根据每个资源的使用情况，动态分配slave到空闲的节点上创建，降低出现因某节点资源利用率高，降低出现因某节点利用率高出现排队的情况
- **扩展性好**，当Kubernetes集群的资源严重不足导致Job排队等待时，可以很容器的添加一个Kubernetes Node到集群，从而实现扩展

##### 二、Kubernetes 安装Jenkins

**前提准备：**<br>
1.[安装harbor镜像仓库](https://blog.zs-fighting.cn/2019/10/22/Kubernetes-%E4%BB%8E%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93%E6%8B%89%E5%8F%96%E9%95%9C%E5%83%8F/)<br>
2.[jenkins-master需要volume存储，安装nfs](https://blog.zs-fighting.cn/2019/10/18/Kubernetes-nfs%E5%AE%89%E8%A3%85/)<br>
3.[使用nginx-ingress暴露jenkins的Web UI](https://blog.zs-fighting.cn/2019/11/01/Kubernetes-ingress%E5%8F%8Aingress_controller/)<br>
**安装Jenkins：**<br>
1.创建一个命名空
```
kubectl create namespace jenkins
```
2.准备镜像拉取时的secret、https访问时的secret
```
[root@master jenkins]# cat jenkins_harbor_secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-key
  namespace: jenkins
data:
  .dockerconfigjson: {base64 -w 0 ~/.docker/config.json}
type:
  kubernetes.io/dockerconfigjson

[root@master jenkins]# kubectl create secret tls jenkins-cert --cert=certs/intellicre.crt --key=certs/intellicredit.cn.key -n jenkins
```
`~/.docker/config.json`为docker login的认证文件，base64加密后生成密钥<br>
`certs/intellicre.crt`、`certs/intellicredit.cn.key`分别是ssl认证时的证书跟密钥<br>
3.准备pv/pvc持久化存储数据<br>
我们将容器的 /var/jenkins_home 目录挂载到了一个名为 opspvc 的 PVC 对象上面，所以我们同样还得提前创建一个对应的 PVC 对象，我们可以使用 StorageClass 对象来自动创建。
```
cat >/opt/jenkins/jenkins_pv.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: opspv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    server: 192.168.0.222
    path: /data/kubernetes/jenkins
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: opspvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
EOF
```
需要修改/data/kubernetes/jenkins的权限，`chmod 777 -R /data/kubernetes/jenkins`
4.准备ServiceAccount，赋予权限<br>
这里还需要使用到一个拥有相关权限的 serviceAccount：jenkins，我们这里只是给 jenkins 赋予了一些必要的权限
```
cat jenkins_rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
```
5.准备jenkins-master Deployment
```
cat jenkins_deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins         #deployment名称
  namespace: jenkins      #命名空间
spec:
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 10     #优雅停止pod
      serviceAccount: jenkins               #前面创建的服务账户
      containers:
      - name: jenkins
        image: 192.168.0.109/jenkins/jenkins:0.0.1               #镜像版本
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080                #外部访问端口
          name: web
          protocol: TCP
        - containerPort: 50000              #jenkins slave发现端口
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60          #容器初始化完成后，等待60秒进行探针检查
          timeoutSeconds: 5
          failureThreshold: 12          #当Pod成功启动且检查失败时，Kubernetes将在放弃之前尝试failureThreshold次。放弃生存检查意味着重新启动Pod。而放弃就绪检查，Pod将被标记为未就绪。默认为3.最小值为1
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:                       #需要将jenkins_home目录挂载出来
        - name: jenkinshome
          subPath: jenkins
          mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: opspvc             #这里将上面创建的pv关联到pvc
      imagePullSecrets:
      - name: harbor-key				#拉取镜像使用的secret
```

我们镜像使用的是：jenkins/jenkins:lts

6.准备svc跟ingress<br>
现在还缺少一个svc跟ingress，因为我们虽然现在jenkins已经在内部可以访问，但是我们在外面是无法访问的。接下来我们创建一个svc跟ingress
```
[root@master jenkins]# cat jenkins_svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: ClusterIP
  clusterIP: None
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent

[root@master jenkins]# cat jenkins_ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - jenkins.intellicredit.cn
    secretName: jenkins-cert
  rules:
  - host: jenkins.intellicredit.cn
    http:
      paths:
      - path:
        backend:
          serviceName: jenkins
          servicePort: 8080
```
7.创建
```
kubectl apply -f jenkins_pv.yaml
kubectl apply -f jenkins_harbor_secret.yaml
kubectl apply -f jenkins_rbac.yaml
kubectl apply -f jenkins_svc.yaml
kubectl apply -f jenkins_ingress.yaml
kubectl apply -f jenkins_deployment.yaml
```
##### 三、 Web UI安装jenkins 插件
将系统默认推荐的插件安装即可，这里安装步骤忽略...<br>
**安装jenkins遇到的问题**<br>
1.**安装成功后，访问https://jenkins.intellicredit.cn:30443  浏览器界面一直显示：**<br>
**Please wait while Jenkins is getting ready to work …**<br>
**Your browser will reload automatically when Jenkins is ready.**<br>
**解法方法：**进入jenkins的工作目录，打开hudson.model.UpdateCenter.xml，将 http://updates.jenkins-ci.org/update-center.json 修改成 http://mirror.xmission.com/jenkins/updates/update-center.json<br>
2.**访问https://jenkins.intellicredit.cn:30443  浏览器界面一直显示：**<br>
**“该jenkins实例似乎已离线”**<br>
**解法方法**：进入jenkins的工作目录，修改${JENKINS_HOME}/updates/default.json，将www.google.com修改为www.baidu.com


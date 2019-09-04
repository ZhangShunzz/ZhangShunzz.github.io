---
layout: post
title:  k8s yaml文件
date:   2019-01-31 13:41:00 +0800
categories: 技术
tag: Kubernetes
---

### Pod资源
```
apiVersion: v1       
kind: Pod       
metadata:       
  name: string       
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string      
  annotations:       #自定义注释列表
    - name: string    
spec:         
  containers:      
  - name: string     
    image: string    
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名    
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称   
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称    
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string       
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string    
    hostNetwork:false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）   
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string 
        items:     
        - key: string      
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string       
          path: string
```
### 控制器（controller）

- **Deployment**

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zzz
spec:
  revisionHistoryLimit: 10  ①
  strategy:  ②
    rollingUpdate:
      maxSurge: 35%
      maxUnavailable: 35%
  replicas: 7
  template:
    metadata:
      labels:
        app: web_server
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
      nodeSelector:  ③
        disktype: ssd

①在滚动升级时记录10个版本信息，以便回滚
②maxSurge控制滚动更新过程中副本总数超过DESIRED的上限，maxUnavailable不可用的副本相占DESIRED的最大比例（滚动升级失败）
③节点选择器
```

- **Daeomnset**

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

### Service

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

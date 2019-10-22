---
layout: post
title: "Kubernetes：从私有仓库拉取镜像"
author: "zhangshun"
header-img: "img/background/8.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### 前提条件

- kubernetes集群
- harbor镜像仓库，本文harbor部署在kubernetes集群外
- 在node上修改/etc/docker/dameon.json，可以将harbor地址添加信任，无需https

### docker-compose方式安装harbor

下载安装包： `wget https://storage.googleapis.com/harbor-releases/harbor-offline-installer-v1.5.2.tgz`

解压： `tar zxf harbor-offline-installer-v1.5.2.tgz`

修改harbor配置文件：
```
vim harbor.cfg
max_job_workers = 2    #多少个进程处理任务，一般与cpu核数一致
hostname = 192.168.0.109
harbor_admin_password = 123456    #harbor管理员密码
db_password = root123    #mysql管理员密码
```

安装：`./install.sh`（需提前安装docker-ce跟docker-compose）<br>
yum -y install docker-ce<br>
yum -y install docker-compose<br>

使用http协议，客户端需要将harbor地址加入/etc/docker/daemon.json
```
vim /etc/docker/daemon.json
{
  ...
  "insecure-registries": ["192.168.0.109"]
  ...
}
```

### 为k8s集群创建Secret

有两种方式创建Secret<br>
1. kubectl命令行创建
当pod从私用仓库拉取镜像时，k8s集群使用类型为docker-registry的Secret来提供身份认证，创建一个名为harbor-kry的Secret，执行如下命令:
```
kubectl -n int create secret docker-registry harbor-key \
--docker-server=192.168.0.109 \
--docker-username=admin \
--docker-password=<your-pword> \
--docker-email=admin@example.com
```
2. 写到yaml文件中
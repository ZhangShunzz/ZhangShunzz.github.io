---
layout: post
title: 自动化运维工具saltstack(一)
date: 2019-08-08 09:13:20
categories: 技术
tags: saltstack
---

### 安装

###### Master端安装：

```
SaltStack@Master: rpm -vih http://mirrors.zju.edu.cn/epel/6/i386/epel-release-6-8.noarch.rpm
SaltStack@Master: yum install salt-master -y
SaltStack@Master: service salt-master start
```

##### Minon端安装：

```
SaltStack@Minion: rpm -vih http://mirrors.zju.edu.cn/epel/6/i386/epel-release-6-8.noarch.rpm
SaltStack@Minion: yum install salt-minion -y
SaltStack@Minion: sed -i 's/#master: salt/master: IPADDRESS/g' /etc/salt/minion #IPADDRESS
SaltStack@Minion: service salt-minion start
```

### Master配置文件

- max_open_files:可以根据Master将Minion数量进行适当的调整
- timeout:可以根据Master和Minion的网络状况适当调整
- auto_accept和autosign_file:在大规模部署Minion的时候可以设置自动签证
- interface:端口监听地址
- ipv6:是否开启ipv6地址监听
- worker_threads:saltstack管理线程数目
- file_roots:设置saltstack的文件根路径

### Minon配置文件

- master:设置master地址
- id:设置minon id

### 
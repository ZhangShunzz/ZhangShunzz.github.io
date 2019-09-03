---
layout: post
title:  Kvm：通过 libvirt 远程管理虚拟机
date:   2018-11-19 10:50:00 +0800
categories: 技术
tag: kvm
---


1、通过qemu+ssh方式

2、通过qemu+tcp方式

主控端需要安装相关工具包：

	#yum groupinstall "Virtualization" 
	#yum install libvirt libvirt-python python-virtinst virt-viewer

### 1、通过qemu+ssh方式

通过qemu+ssh连接方式比较简单，只需node5能用ssh远程访问node4即可

命令如下：

	#virsh -c qemu+ssh://root@192.168.1.166/system

如果2个节点设置了互信，免密钥登录，可直接执行virsh相关命令

	[root@node5 ~]# virsh -c qemu+ssh://root@192.168.1.166/system list
	 Id    名称                         状态
	----------------------------------------------------
	 3     vm01                           running

### 2、 通过qemu+tcp方式

被控端上：

修改/etc/sysconfig/libvirtd,开启以下2个配置项：

	[root@node4 ~]# egrep -v "^#|^$" /etc/sysconfig/libvirtd
	LIBVIRTD_CONFIG=/etc/libvirt/libvirtd.conf
	LIBVIRTD_ARGS="--listen

修改配置文件

	vim /etc/libvirt/libvirtd.conf
	listen_tls = 0
	listen_tcp = 1
	tcp_port = "16509"
	listen_addr = "0.0.0.0"
	auth_tcp = "none"

重启libvirtd并查看监听的端口

	# /etc/init.d/libvirtd restart
	# netstat -anltp|grep 16509
	tcp   0      0 0.0.0.0:16509    0.0.0.0:*      LISTEN      28843/libvirtd
	主控端上远程访问（需要确保可以访问被控端的16509 tcp端口）：
	[root@node5 ~]# virsh -c qemu+tcp://192.168.1.166/system list
	 Id    名称                         状态
	----------------------------------------------------
	 3     vm01                           running

主控端上远程访问（需要确保可以访问被控端的16509 tcp端口）：

	[root@node5 ~]# virsh -c qemu+tcp://192.168.1.166/system list
	 Id    名称                         状态
	----------------------------------------------------
	 3     vm01                           running
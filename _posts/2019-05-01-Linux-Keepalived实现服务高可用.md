---
layout: post
title: "Linux：Keepalived实现服务高可用"
author: "zhangshun"
header-img: "img/background/19.jpg"
header-mask: 0.2
tags:
  - Linux
---

### 1、keepalived是什么？
Keepalived软件起初是专为LVS负载均衡软件设计的，用来管理并监控LVS集群系统中各个服务节点的状态，后来又加入了可以实现高可用的VRRP功能。因此，Keepalived除了能够管理LVS软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL等）的高可用解决方案软件。

Keepalived软件主要是通过VRRP协议实现高可用功能的。VRRP是Virtual Router RedundancyProtocol(虚拟路由器冗余协议）的缩写，VRRP出现的目的就是为了解决静态路由单点故障问题的，它能够保证当个别节点宕机时，整个网络可以不间断地运行。

**所以，Keepalived 一方面具有配置管理LVS的功能，同时还具有对LVS下面节点进行健康检查的功能，另一方面也可实现系统网络服务的高可用功能。**

### 2、keepalived服务的三个重要功能

- 管理LVS负载均衡软件
- 实现LVS集群节点的健康检查中
- 作为系统网络服务的高可用性（failover）

### 3、Keepalived高可用故障切换转移原理

Keepalived高可用服务对之间的故障切换转移，是通过 VRRP (Virtual Router Redundancy Protocol ,虚拟路由器冗余协议）来实现的。

在 Keepalived服务正常工作时，主 Master节点会不断地向备节点发送（多播的方式）心跳消息，用以告诉备Backup节点自己还活看，当主 Master节点发生故障时，就无法发送心跳消息，备节点也就因此无法继续检测到来自主 Master节点的心跳了，于是调用自身的接管程序，接管主Master节点的 IP资源及服务。而当主 Master节点恢复时，备Backup节点又会释放主节点故障时自身接管的IP资源及服务，恢复到原来的备用角色。

那么，什么是VRRP呢？

VRRP ,全称 Virtual Router Redundancy Protocol ,中文名为虚拟路由冗余协议 ，VRRP的出现就是为了解决静态踣甶的单点故障问题，VRRP是通过一种竞选机制来将路由的任务交给某台VRRP路由器的。

### 4、keepalived软件使用

**4.1、第一个里程碑 keepalived软件安装**

`yum install keepalived -y`

```
/etc/keepalived
/etc/keepalived/keepalived.conf     #keepalived服务主配置文件
/etc/rc.d/init.d/keepalived         #服务启动脚本
/etc/sysconfig/keepalived
/usr/bin/genhash
/usr/libexec/keepalived
/usr/sbin/keepalived
```
**4.2、配置文件说明**

主节点配置文件
```
! Configuration File for keepalived

vrrp_script chk_nginx_port {		###定义脚本
        script "/etc/keepalived/sh/nginx_keepalived.sh"
        interval 3		###执行监控脚本的间隔时间
}
global_defs {		###全局配置
   router_id master			###定义路由标识信息，相同局域网唯一
}

vrrp_instance VI_1 {		###定义实例
    state MASTER		###状态参数 master/backup 只是说明
    nopreempt		###非抢占模式，避免无用的切换
    interface eth0		###虚IP地址放置的网卡位置
    virtual_router_id 254		###同一实例组要一致，同一个集群id一致
    priority 100		###优先级决定是主还是备    越大越优先
    advert_int 1		###主备通讯时间间隔
    authentication {		###认证
        auth_type PASS
        auth_pass 1111
    }
    track_script {		###调用脚本
        chk_nginx_port
    }
    virtual_ipaddress {			###虚拟路由IP地址
        192.168.0.174 dev eth0 label eth0:0			####以别名的方式设置
    }
}
```
备节点配置文件
```
! Configuration File for keepalived

vrrp_script chk_nginx_port {		###定义脚本
        script "/etc/keepalived/sh/nginx_keepalived.sh"
        interval 3		###执行监控脚本的间隔时间
}
global_defs {		###全局配置
   router_id backup			###定义路由标识信息，相同局域网唯一
}

vrrp_instance VI_1 {		###定义实例
    state BACKUP		###状态参数 master/backup 只是说明
    nopreempt		###非抢占模式，避免无用的切换
    interface eth0		###虚IP地址放置的网卡位置
    virtual_router_id 254		###同一实例组要一致，同一个集群id一致
    priority 99		###优先级决定是主还是备    越大越优先
    advert_int 1		###主备通讯时间间隔
    authentication {		###认证
        auth_type PASS
        auth_pass 1111
    }
    track_script {		###调用脚本
        chk_nginx_port
    }
    virtual_ipaddress {			###虚拟路由IP地址
        192.168.0.174 dev eth0 label eth0:0			####以别名的方式设置
    }
}
```

附带检测脚本nginx_keepalived.sh
```
#!/bin/bash
netstat -antpul|grep nginx &> /dev/null
if [ $? -ne 0 ];then
        /usr/local/nginx/sbin/nginx
        netstat -antpul|grep nginx &> /dev/null
        if [ $? -ne 0 ];then
                systemctl stop keepalived
        fi
fi
```


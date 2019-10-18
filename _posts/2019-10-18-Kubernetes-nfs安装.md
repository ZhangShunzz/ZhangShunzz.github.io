---
layout: post
title: "Kubernetes：nfs安装"
subtitle: "Kubernetes中经常会用到存储类StorageClass，使用nfs存储卷非常简便(非生产环境)"
author: "zhangshun"
header-img: "img/background/7.jpg"
header-mask: 0.2
tags:
  - Kubernetes
---

### 创建 NFS 服务器

NFS是网络文件系统(Network File System), 它允许系统将本地目录和文件共享给网络上的其他系统。通过 NFS，用户和应用程序可以访问远程系统上的文件，就象它们是本地文件一样。<br>
**安装**<br>
NFS需要nfs-utils和rpcbind两个包, 但安装nfs-utils时会一起安装上rpcbind：<br>
`yum install nfs-utils`<br>
**编辑exports文件**<br>
编辑`/etc/exports`文件添加共享目录，每个目录的设置独占一行，每行共三部分, 前边是目录, 后边是客户机, 括号中是配置参数. 编写格式如下：<br>
`NFS共享目录路径 客户机IP或者名称(参数1,参数2,...,参数n)`<br>
例如：
```
# example: /home/nfs 192.168.64.134(rw,sync,fsid=0)  192.168.64.135(rw,sync,fsid=0)   
# 第一部分: /home/nfs, 本地要共享出去的目录。
# 第二部分: 192.168.64.0/24 ，允许访问的主机，可以是一个IP：192.168.64.134，也可以是一个IP段：192.168.64.0/24. "*"表示所有
# 第三部分:
#     rw表示可读写，ro只读；
#     sync ：同步模式，内存中数据时时写入磁盘；async ：不同步，把内存中数据定期写入磁盘中；
#     no_root_squash ：加上这个选项后，root用户就会对共享的目录拥有至高的权限控制，就像是对本机的目录操作一样。不安全，不建议使用；root_squash：和上面的选项对应，root用户对共享目录的权限不高，只有普通用户的权限，即限制了root；all_squash：不管使用NFS的用户是谁，他的身份都会被限定成为一个指定的普通用户身份；
#     anonuid/anongid ：要和root_squash 以及all_squash一同使用，用于指定使用NFS的用户限定后的uid和gid，前提是本机的/etc/passwd中存在这个uid和gid。
#     fsid=0表示将/home/nfs整个目录包装成根目录

/home/nfs *(rw,sync,no_root_squash)
```
**设置开机自启动**
```
systemctl enable rpcbind.service
systemctl enable nfs-server.service
```
**启动**
```
service nfs-server start
service rpcbind start
```
**查看nfs运行**
```
[root@localhost ~]# rpcinfo -p
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100024    1   udp  47378  status
    100024    1   tcp  34772  status
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  58178  nlockmgr
    100021    3   udp  58178  nlockmgr
    100021    4   udp  58178  nlockmgr
    100021    1   tcp  42391  nlockmgr
    100021    3   tcp  42391  nlockmgr
    100021    4   tcp  42391  nlockmgr
```
**关闭防火墙**<br>
`systemctl stop firewalld.service`

### 客户端<br>

客户端不需要启动服务, 只下载nfs工具:<br>
`yum install nfs-utils`<br>
**客户端验证**
```
[root@localhost ~]# showmount -e 192.168.64.133
Export list for 192.168.64.133:
/home/nfs *
```
**客户端挂载**<br>
使用 mount 命令将NFS服务器的/home/nfs挂载到客户端的/kubernetes目录。可以在客户端终端输入如下命令：<br>
`mount 192.168.64.133:/home/nfs /kubernetes`<br>
挂载点 /kubernetes 目录必须已经存在, 且在 /kubernetes 目录中没有文件或子目录。

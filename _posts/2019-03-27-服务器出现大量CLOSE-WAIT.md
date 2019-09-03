---
layout: post
title:  Linux服务器出现大量CLOSE_WAIT
date:   2019-03-27 13:27:00 +0800
categories: 技术
tag: 故障
---

### 背景

之前接手了前同事维护的一个网站服务，下午收到通知说是网站无法访问，测试访问现象：长时间无响应之后报错502，502报错是后端处理不了请求，架构是nginx+tomcat，登陆排查

- catalina.out无明显报错
- 系统日志message没有报错
- jstack命令不存在所以看不到jvm里线程的状态
- 内存、cpu、io、磁盘空间都是正常
- ping服务器也是正常，网络没问题
- `netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`查看tcp连接状态发现出现大量CLOSE_WAIT


![大量CLOSE_WAIT](/2019-03-27-服务器出现大量CLOSE-WAIT/CLOSE_WAIT02.png)

### CLOSE_WAIT

所谓 CLOSE_WAIT，借用某位大牛的话来说应该倒过来叫做 WAIT_CLOSE，也就是说「等待关闭」，**TCP关闭连接时的图例：**

![TCP关闭连接](/2019-03-27-服务器出现大量CLOSE-WAIT/CLOSE_WAIT01.png)


首先主动关闭的一方发出 FIN 包，被动关闭的一方响应 ACK 包，此时，被动关闭的一方就进入了 CLOSE_WAIT 状态。如果一切正常，稍后被动关闭的一方也会发出 FIN 包，然后迁移到 LAST_ACK 状态。

通常，CLOSE_WAIT 状态在服务器停留时间很短，如果你发现大量的 CLOSE_WAIT 状态，那么就意味着被动关闭的一方没有及时发出 FIN 包，一般有如下几种可能：

1. 程序问题：如果代码层面忘记了 close 相应的 socket 连接，那么自然不会发出 FIN 包，从而导致 CLOSE_WAIT 累积；或者代码不严谨，出现死循环之类的问题，导致即便后面写了 close 也永远执行不到。
2. 响应太慢或者超时设置过小：如果连接双方不和谐，一方不耐烦直接 timeout，另一方却还在忙于耗时逻辑，就会导致 close 被延后。响应太慢是首要问题，不过换个角度看，也可能是 timeout 设置过小。
3. BACKLOG 太大：此处的 backlog 不是 syn backlog，而是 accept 的 backlog，如果 backlog 太大的话，设想突然遭遇大访问量的话，即便响应速度不慢，也可能出现来不及消费的情况，导致多余的请求还在队列里就被对方关闭了。

### 解决办法

##### 看到服务器之前并没有做过优化，决定先优化下再观察一段时间

1、优化文件打开数

调整系统最大文件打开数

 - 临时生效：ulimit -n 65535
 - 永久生效：

```
cat /etc/security/limits.conf
* soft nofile 32768 #限制单个进程最大文件句柄数
* hard nofile 65536 #限制单个进程最大文件句柄数
```
 
 - 修改整个系统最大文件句柄数：

```
vim /etc/sysctl.conf
fs.file-max=655350 #限制整个系统最大文件句柄数
```

2、优化tomcat配置

```
    <Connector port="80" protocol="HTTP/1.1"
               connectionTimeout="20000"
               maxConnections="3000"
               maxThreads="400"
               minSpareThreads="200"
               redirectPort="8443"
               enableLookups="false" 
               acceptCount="100" 
               maxPostSize="10485760" 
               compression="on" 
               disableUploadTimeout="true" 
               compressionMinSize="2048" 
               acceptorThreadCount="8" 
               compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript" 
               URIEncoding="utf-8" />
```

1. connnectionTimeout:网络连接超时，单位：毫秒。设置为0表示永不超时，这样设置有隐患的。通常可设置为20000毫秒
2. maxConnections:当Tomcat接收的连接数达到maxConnections时，Acceptor线程不会读取accept队列中的连接；这时accept队列中的线程会一直阻塞着，直到Tomcat接收的连接数小于maxConnections
3. maxThreads:Tomcat线程池最多能起的线程数
4. minSpareThreads:Tomcat初始化的线程池大小或者说Tomcat线程池最少会有这么多线程
5. enableLookups:是否允许DNS查询,为了提高处理能力，应设置为false
6. acceptCount:指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，就是被排队的请求数，超过这个数的请求将拒绝连接。
7. maxPostSize:对post请求参数大小做出限制
8. compression：打开压缩功能
9. disableUploadTimeout：对POST请求发送数据超时使用其他参数来设置，这样在发送数据的过程中最大可以等待的时间间隔就不再由connectionTimeout决定，而是由connectionUploadTimeout决定
10. compressionMinSize：启用压缩的输出内容大小，默认为2KB
11. acceptorThreadCount：接收Socket连接的线程数。默认值是1，这个值不需要太大，最大值与CPU核心数一样就行了，没有必要太大
12. compressableMimeType：压缩的类型

3、调整内核参数

```
net.ipv4.ip_local_port_range = 1024 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_window_scaling = 0
net.ipv4.tcp_sack = 0
net.core.netdev_max_backlog = 30000
net.ipv4.tcp_no_metrics_save = 1
net.core.somaxconn = 22144
net.ipv4.tcp_syncookies = 0
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
vm.overcommit_memory = 1
fs.file-max = 2000000
fs.nr_open = 2000000
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward=1
```

1. net.ipv4.ip_local_port_range:定义了网络连接可用作其源（本地）端口的最小和最大端口
2. net.core.rmem_max:表示接受套接字缓冲区大小的最大值
3. net.core.wmem_max:表示发送套接字缓冲区大小的最大值
4. net.ipv4.tcp_rmem：自动调优所使用的接收缓冲区的值
5. net.ipv4.tcp_wmem：自动调优所使用的发送缓冲区的值
6. net.ipv4.tcp_fin_timeout：对于本端断开的socket连接，TCP保持在FIN_WAIT_2状态的时间
7. net.ipv4.tcp_tw_recycle：启用TIME-WAIT状态sockets的快速回收，这个选项不推荐启用。在NAT(Network Address Translation)网络下，会导致大量的TCP连接建立错误
8. net.ipv4.tcp_timestamps：不再检查socket时间戳
9. net.ipv4.tcp_sack：启用有选择的应答（1表示启用），通过有选择地应答乱序接收到的报文来提高性能，让发送者只发送丢失的报文段，（对于广域网通信来说）这个选项应该启用，但是会增加对CPU的占用
10. net.core.netdev_max_backlog：在每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
11. net.ipv4.tcp_no_metrics_save：TCP 默认缓存了很多连接指标，如果设置的话，TCP 在关闭的时候不缓存这些指标

---
layout: post
title: "Linux：网络丢包以及ethtool工具使用"
author: "zhangshun"
header-img: "img/background/20.jpg"
header-mask: 0.2
tags:
  - Linux
---

### ifconfig命令

```
root@test-01:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:16:3e:05:34:06  
          inet addr:10.10.10.10  Bcast:10.10.10.10  Mask:255.255.252.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3769730720 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4085641704 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1295103764780 (1.2 TB)  TX bytes:2205272029564 (2.2 TB)

eth1      Link encap:Ethernet  HWaddr 00:16:3e:05:a0:b4  
          inet addr:139.139.139.139  Bcast:139.139.139.139  Mask:255.255.252.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:109670697 errors:0 dropped:0 overruns:0 frame:0
          TX packets:80474358 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:12142422828 (12.1 GB)  TX bytes:13172798780 (13.1 GB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:9847 errors:0 dropped:0 overruns:0 frame:0
          TX packets:9847 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2798284 (2.7 MB)  TX bytes:2798284 (2.7 MB)
```

- HWaddr：网卡的物理地址
- Bcast：广播地址
- UP：代表网卡开启状态
- RUNNING：代表网卡的网线被接上
- MULTICAST：支持组播
- MTU:1500：最大传输单元，1500字节
- RX/TX
  - packets：发送的数据包
  - **errors**：错误数据包个数，包括too-long-frames错误，Ring Buffer溢出错误，crc校验错误，帧同步错误，fifo overruns以及missed pkg等等
  - **dropped**：表示数据包已经进入了Ring Buffer，但是由于内存不够等系统原因，导致在拷贝到内存的过程中被丢弃。
  - **overruns**：overruns的增大意味着数据包没到Ring Buffer就被网卡物理层给丢弃了，而CPU无法及时的处理中断是造成Ring Buffer满的原因之一
- collisions：如果你看到大量的冲突(Collisions)，那么这很有可能是网络的传输介质出了问题，例如网线不通或hub损坏。


### ethtool工具

ethtool命令用于获取以太网卡的配置信息，或者修改这些配置。这个命令比较复杂，功能特别多。

**一、ethtool eno1：查看网卡基础信息**

```
Settings for eno1:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Half 1000baseT/Full 
	Supported pause frame use: No
	Supports auto-negotiation: Yes
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Half 1000baseT/Full 
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: Yes		###自动协商开启
	Link partner advertised link modes:  10baseT/Half 10baseT/Full 
	                                     100baseT/Half 100baseT/Full 
	                                     1000baseT/Full 
	Link partner advertised pause frame use: Symmetric Receive-only
	Link partner advertised auto-negotiation: Yes
	Speed: 1000Mb/s			###网卡速度
	Duplex: Full			###网卡工作模式
	Port: Twisted Pair
	PHYAD: 1
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off
	Supports Wake-on: g
	Wake-on: d
	Current message level: 0x000000ff (255)
			       drv probe link timer ifdown ifup rx_err tx_err
	Link detected: yes		###该网卡已激活
```

**二、ethtool -g eno1:查看Ring Buffer队列大小**

```
Ring parameters for eno1:
Pre-set maximums:
RX:		2047
RX Mini:	0
RX Jumbo:	0
TX:		511
Current hardware settings:
RX:		200
RX Mini:	0
RX Jumbo:	0
TX:		511
```
看到RX和TX最大是2047和511，当前值为200和511。**队列越大丢包的可能越小，但数据延迟会增加**

**ethtool -G eno1 rx 1024:调整 Ring Buffer 队列的权重**



**三、ethtool -S eno1:收到的数据包统计**

RX就是收到数据，TX是发出数据。还会展示NIC每个队列收发消息情况。**其中比较关键的是带有drop字样的统计和fifo_errors的统计，可以使用ethtool -S eno1|egrep 'error|drop'进行统计**

`cat /proc/net/dev`也可以统计数据包的信息，不过这个统计比较难看

**四、ethtool -i eno1:查看网卡驱动**

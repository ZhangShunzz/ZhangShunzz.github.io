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


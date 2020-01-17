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
- RX
  - aa
  - aa
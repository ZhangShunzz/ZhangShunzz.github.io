---
layout: post
title: "Linux：网卡状态以及ethtool工具"
author: "zhangshun"
header-img: "img/background/20.jpg"
header-mask: 0.2
tags:
  - Linux
---

### ifconfig命令

```
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
```

- aaa
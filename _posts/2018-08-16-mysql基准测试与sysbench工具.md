---
layout: post
title:  mysql基准测试与sysbench工具
date:   2018-08-16 11:34:00 +0800
categories: 技术
tag: mysql
---


一、基准测试简介 
---

**1、什么是基准测试**

数据库的基准测试是对数据库的性能指标进行定量的、可复现的、可对比的测试。

**基准测试与压力测试**

基准测试可以理解为针对系统的一种压力测试。但基准测试不关心业务逻辑,更加简单、直接、易于测试,数据可以由工具生成,不要求真实;而压力测试一般考虑业务逻辑(如购物车业务),要求真实的数据。

**2、基准测试的作用**

对于多数Web应用,整个系统的瓶颈在于数据库;原因很简单:Web应用中的其他因素,例如网络带宽、负载均衡节点、应用服务器(包括CPU、内存、硬盘灯、连接数等)、缓存,都很容易通过水平的扩展(俗称加机器)来实现性能的提高。而对于MySQL,由于数据一致性的要求,无法通过增加机器来分散向数据库写数据带来的压力;虽然可以通过前置缓存(Redis等)、读写分离、分库分表来减轻压力,但是与系统其它组件的水平扩展相比,受到了太多的限制。

而对数据库的基准测试的作用,就是分析在当前的配置下(包括硬件配置、OS、数据库设置等),数据库的性能表现,从而找出MySQL的性能阈值,并根据实际系统的要求调整配置。

**3、基准测试的指标**

常见的数据库指标包括:

- TPS/QPS:衡量吞吐量
- 响应时间:包括平均响应时间、最小响应时间、最大响应时间、时间百分比等,其中时间百分比参考意义较大,如前95%的请求的最大响应时间。
- 并发量:同时处理的查询请求的数量。


二、sysbench
---

**1、sysbench安装（ubuntu系统）**

	curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh |sudo bash
	sudo apt -y install sysbench

**2、oltp测试**

	find / -name oltp.lua
	sysbench --test=tests/db/oltp.lua(根据具体位置填写) --mysql-host=$hostip --mysql-port=3306 --mysql-user=$user --mysql-password=$password --mysql-db=$databasename --oltp-test-mode=complex --oltp-tables-count=10 --oltp-table-size=10000000 --rand-init=on  prepare
	sysbench tests/db/oltp.lua(根据具体位置填写) --mysql-host=$hostip --mysql-port=3306 --mysql-user=$user --mysql-password=$password --mysql-db=$databasename --oltp_tables_count=10 --oltp-table-size=10000000 --num-threads=10 --oltp-read-only=off --report-interval=10 --rand-type=uniform --max-time=1800  --max-requests=0 --percentile=99 run >> /root/sysbench.log

参数详解：

--test=tests/db/oltp.lua 指定测试的脚本

--oltp-test-mode=complex 执行模式,包括simple、nontrx和complex,默认是complex，complex模式下测试最全面,会测试增删改查,而且会使用事务。可以根据自己的需要选择测试模式。

--oltp-tables-count=10 测试的表数量,根据实际情况选择

--oltp-table-size 测试的表的大小

--rand-init=on 表示每个测试表都是用随机数据来填充的

--max-time run多少时间

--max-requests=0 表示总请求数为 0，因为上面已经定义了总执行时长，所以总请求数可以设定为 0；也可以只设定总请求数，不设定最大执行时长

--percentile 表示设定采样比例，默认是 95%，即丢弃1%的长请求，在剩余的99%里取最大值，即：模拟 对10个表并发OLTP测试，每个表1000万行记录，持续压测时间为 1小时。

--oltp-read-only=off 表示不要进行只读测试，也就是会采用读写混合模式测试

--report-interval=10 表示每10秒输出一次测试进度报告

prepare、run和cleanup,顾名思义,prepare是为测试提前准备数据,run是执行正式的测试,cleanup是在测试完成后对数据库进行清理。

**3、测试结果**

	sysbench 0.5:  multi-threaded system evaluation benchmark
	 
	Running the test with following options:
	Number of threads: 256
	Report intermediate results every 10 second(s)
	Random number generator seed is 0 and will be ignored
	 
	--线程启动
	Threads started!
	 
	--每10秒钟报告一次测试结果，tps、每秒读、每秒写、99%以上的响应时长统计
	[  10s] threads: 256, tps: 524.19, reads/s: 7697.05, writes/s: 2143.56, response time: 1879.46ms (99%)
	[  20s] threads: 256, tps: 96.50, reads/s: 1351.01, writes/s: 373.30, response time: 9853.49ms (99%)
	[  30s] threads: 256, tps: 235.50, reads/s: 3297.01, writes/s: 946.90, response time: 2150.47ms (99%)
	[  40s] threads: 256, tps: 115.50, reads/s: 1617.00, writes/s: 491.40, response time: 4562.75ms (99%)
	[  50s] threads: 256, tps: 262.10, reads/s: 3669.41, writes/s: 1016.10, response time: 2049.90ms (99%)
	[  60s] threads: 256, tps: 121.50, reads/s: 1701.00, writes/s: 499.10, response time: 3666.03ms (99%)
	[  70s] threads: 256, tps: 201.40, reads/s: 2735.10, writes/s: 769.50, response time: 3867.82ms (99%)
	[  80s] threads: 256, tps: 204.70, reads/s: 2950.29, writes/s: 838.10, response time: 2724.99ms (99%)
	[  90s] threads: 256, tps: 118.40, reads/s: 1657.61, writes/s: 490.00, response time: 3835.53ms (99%)
 
 
	OLTP test statistics:
	    queries performed:
	        read:                            8823206    -- 读总数
	        write:                           2520916    -- 写总数
	        other:                           1260458    -- 其他操作总数(SELECT、INSERT、UPDATE、DELETE之外的操作，例如COMMIT等)
	        total:                           12604580    -- 全部总数 查询总数及qps
	    transactions:                        630229 (174.94 per sec.)    -- 总事务数(每秒事务数) 事务总数及tps
	    deadlocks:                           0      (0.00 per sec.)        -- 发生死锁总数
	    read/write requests:                 11344122 (3148.86 per sec.)    -- 读写总数(每秒读写次数)
	    other operations:                    1260458 (349.87 per sec.)    -- 其他操作总数(每秒其他操作次数)
	 
	 
	General statistics:        -- 一些统计结果        
	    total time:                          3602.6152s    -- 总耗时
	    total number of events:              630229        -- 共发生多少事务数
	    total time taken by event execution: 921887.7227s    -- 所有事务耗时相加(不考虑并行因素)
	    response time:                    -- 响应时间
	         min:                                  6.52ms    -- 最小耗时
	         avg:                               1462.78ms    -- 平均耗时
	         max:                               9918.51ms    -- 最长耗时
	         approx.  99 percentile:            3265.01ms    -- 超过99%平均耗时
	 
	 
	Threads fairness:                -- 线程的稳定性
	    events (avg/stddev):           2461.8320/34.60      -- 事件(平均值/偏差)
	    execution time (avg/stddev):   3601.1239/0.63    -- 执行时间(平均值/偏差)

**4、清理测试数据**

```
sysbench --test=tests/db/oltp.lua(根据具体位置填写) --mysql-host=databasehost --mysql-port=3306 --mysql-user=user --mysql-password=password --mysql-db=databasename --oltp-test-mode=complex --oltp-tables-count=10 --oltp-table-size=10000000 --rand-init=on  cleanup
```

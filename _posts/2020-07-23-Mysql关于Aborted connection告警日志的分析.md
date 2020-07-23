---
layout: post
title: "Mysql：关于Aborted connection告警日志的分析"
author: "zhangshun"
header-img: "img/background/17.jpg"
header-mask: 0.2
tags:
  - Mysql
---

**前言： **

有时候，连接MySQL的会话经常会异常退出，错误日志里会看到"Got an error reading communication packets"类型的告警。本篇文章我们一起来讨论下该错误可能的原因以及如何来规避。

**1.状态变量Aborted_clients和Aborted_connects**


首先我们来了解下Aborted_clients和Aborted_connects这两个状态变量的含义，当出现会话异常退出时，这两个状态值会有变化。根据官方文档描述，总结如下：


![](/img/in-post/2020-07-23-Mysql关于Aborted connection告警日志的分析/mysql官方解释.png)

造成Aborted_connects状态变量增加的可能原因：
- 客户端试图访问数据库，但没有数据库的权限。
- 客户端使用了错误的密码。
- 连接包不包含正确的信息。
- 获取一个连接包需要的时间超过connect_timeout秒。

![](/img/in-post/2020-07-23-Mysql关于Aborted connection告警日志的分析/mysql官方解释-01.png)

造成Aborted_clients状态变量增加的可能原因：
- 程序退出前，客户机程序没有调用mysql_close()。
- 客户端睡眠时间超过了wait_timeout或interactive_timeout参数的秒数。
- 客户端程序在数据传输过程中突然终止。

简单来说即：数据库会话未能正常连接到数据库，会造成Aborted_connects变量增加。数据库会话已正常连接到数据库但未能正常退出，会造成Aborted_clients变量增加。

**2.Got an error reading communication packets原因分析**

会话异常退出一般会造成Aborted connection告警，即我们可以通过Aborted_clients状态变量的变化来反映出是否存在异常会话，那么出现“Got an error reading communication packets” 类似告警的原因就很明了了，查询相关资料，总结出造成Aborted connection告警的可能原因如下：
1. 会话链接未正常关闭，程序没有调用mysql_close()。
2. 睡眠时间超过wait_timeout或interactive_timeout参数的秒数。
3. 查询数据包大小超过max_allowed_packet数值，造成链接中断。
4. 其他网络或者硬件层面的问题。

**3.问题避免与总结**

其实Aborted connection告警是很难避免的，error log里或多或少会有少量Aborted connection信息，这种情况是可以忽略的，但是当你的error log里频繁出现Aborted connection告警，这时候就应该注意了，可能会对业务产生较大的影响。下面列举出几点避免错误的建议

1. 建议业务操作结束后，应用程序逻辑会正确关闭连接，以短连接替代长连接。
2. 检查以确保max_allowed_packet的值足够高，并且客户端没有收到“数据包太大”消息。
3. 确保客户端应用程序不中止连接，例如，如果PHP设置了max_execution_time为5秒，增加connect_timeout并不会起到作用，因为PHP会kill脚本。其他程序语言和环境也有类似的安全选项。
4. 确保事务提交（begin和commit）都正确提交以保证一旦应用程序完成以后留下的连接是处于干净的状态。
5. 检查是否启用了skip-name-resolve，检查主机根据其IP地址而不是其主机名进行身份验证。
6. 尝试增加MySQL的net_read_timeout和net_write_timeout值，看看是否减少了错误的数量
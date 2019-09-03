---
layout: post
title:  Mysql GTID
date:   2019-01-25 09:55:00 +0800
categories: 技术
tag: mysql
---

### 什么是GTID Replication

从 MySQL 5.6.5 开始新增了一种基于 GTID 的复制方式。通过 GTID 保证了每个在主库上提交的事务在集群中有一个唯一的ID。这种方式强化了数据库的主备一致性，故障恢复以及容错能力。

在原来基于二进制日志的复制中，从库需要告知主库要从哪个偏移量进行增量同步，如果指定错误会造成数据的遗漏，从而造成数据的不一致。借助GTID，在发生主备切换的情况下，MySQL的其它从库可以自动在新主库上找到正确的复制位置，这大大简化了复杂复制拓扑下集群的维护，也减少了人为设置复制位置发生误操作的风险。另外，基于GTID的复制可以忽略已经执行过的事务，减少了数据发生不一致的风险。

**什么是GTID**

GTID (Global Transaction ID) 是对于一个已提交事务的编号，并且是一个全局唯一的编号。 GTID 实际上 是由 UUID+TID 组成的。其中 UUID 是一个 MySQL 实例的唯一标识。TID 代表了该实例上已经提交的事务数量，并且随着事务提交单调递增。下面是一个GTID的具体形式：

`3E11FA47-71CA-11E1-9E33-C80AA9429562:23`

一组连续的事务可以用 - 连接的事务序号范围表示。例如：

`e6954592-8dba-11e6-af0e-fa163e1cf111:1-5`

GTID 集合可以包含来自多个 MySQL 实例的事务，它们之间用逗号分隔。如果来自同一 MySQL 实例的事务序号有多个范围区间，各组范围之间用冒号分隔。例如：

`e6954592-8dba-11e6-af0e-fa163e1cf111:1-5:11-18,`
`e6954592-8dba-11e6-af0e-fa163e1cf3f2:1-27`

**GTID的作用**

GTID 的使用不单单是用单独的标识符替换旧的二进制日志文件和位置，它也采用了新的复制协议。旧的协议往往简单直接，即：首先从服务器上在一个特定的偏移量位置连接到主服务器上一个给定的二进制日志文件，然后主服务器再从给定的连接点开始发送所有的事件。

新协议稍有不同：支持以全局统一事务 ID (GTID) 为基础的复制。当在主库上提交事务或者被从库应用时，可以定位和追踪每一个事务。GTID 复制是全部以事务为基础，使得检查主从一致性变得非常简单。如果所有主库上提交的事务也同样提交到从库上，一致性就得到了保证。

GTID 相关操作：默认情况下将一个事务记录进二进制文件时，首先记录它的 GTID，而且 GTID 和事务相关信息一并要发送给从服务器，由从服务器在本地应用认证，但是绝对不会改变原来的事务 ID 号。因此在 GTID 的架构上就算有了N层架构，复制是N级架构，事务 ID 依然不会改变，有效的保证了数据的完整和安全性。

GTID功能的具体归纳主要有以下两点：

  - 根据 GTID 可以知道事务最初是在哪个实例上提交的
  - GTID 的存在方便了 Replication 的 Failover
  
**如何产生GTID**

GTID 的生成受 `gtid_next` 控制。 在主服务器上`gtid_next` 是默认的 `AUTOMATIC`，即在每次事务提交时自动生成新的 GTID 。它从当前已执行的 GTID 集合(即 `gtid_executed` )中，找一个大于 0 的未使用的最小值作为下个事务 GTID 。同时在 Binlog 的实际的更新事务事件前面插入一条 `set gtid_next` 事件。

这里以一条 `insert` 语句来看看 `binlog` 中记录`GTID` 的生成过程：

![GTID1]({{ '/styles/images/gtid/GTID1.png' | prepend: site.baseurl  }})

在从库上回放主库的 Binlog 时，先执行 `SET @@SESSION.GTID_NEXT`语句，然后再执行 insert 语句，确保在主和备上这条 insert 对应于相同的 GTID。

**配置GTID**

```
[mysqld]
#config for gtid start
gtid-mode = ON
log-slave-updates = ON
enforce-gtid-consistency = ON
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
relay_log=/data/mysql/relay_10.0.18.89
#config for gtid end
```

**查看主库与从库的GTID是否开启**

```
mysql> show variables like "%gtid%";

+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)


mysql> show variables like '%gtid_next%';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| gtid_next     | AUTOMATIC |
+---------------+-----------+
1 row in set (0.00 sec)
```

**检查从服务器状态**

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.2.210
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 977
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 1190
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: master1
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table: master1.%
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 977
              Relay_Log_Space: 1391
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: f75ae43f-3f5e-11e7-9b98-001c4297532a
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: f75ae43f-3f5e-11e7-9b98-001c4297532a:1-4
            Executed_Gtid_Set: 2c55f623-4fea-11e7-82c7-001c4283459b:1,
f75ae43f-3f5e-11e7-9b98-001c4297532a:1-4
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

可以看到 IO 和 SQL 线程都为 YES ，另外 `retrieved_Gtid_Set` 接收了4个事务，`Executed_Gtid_Set` 执行了4个事务。

**如何修复GTID复制错误**

在基于 GTID 的复制拓扑中，要想修复从库的 SQL 线程错误，过去的 `SQL_SLAVE_SKIP_COUNTER` 方式不再适用。需要通过设置 `gtid_next` 或 `gtid_purged` 来完成，当然前提是已经确保主从数据一致，仅仅需要跳过复制错误让复制继续下去。

在从库上执行以下SQL：

```
mysql> stop slave;
Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_next='f75ae43f-3f5e-11e7-9b98-001c4297532a:20';
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_next='AUTOMATIC';
Query OK, 0 rows affected (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected (0.02 sec)
```

其中 `gtid_next` 就是跳过某个执行事务，设置 `gtid_next` 的方法一次只能跳过一个事务，要批量的跳过事务可以通过设置 `gtid_purged` 完成。假设下面的场景：

```
mysql> reset master;
Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_purged='f75ae43f-3f5e-11e7-9b98-001c4297532a:1-13:20';
Query OK, 0 rows affected (0.00 sec)
```

**使用xtrbackup 备份恢复GTID**

1、查看slave已经执行的事务

`cat xtrabackup_binlog_info`

`mysql1-bin.000015	892002595	e719a305-cb6b-11e8-841c-525400dbd857:1-9519670`

2、查看slave已执行的gtid是否为空，如果不为空，需要执行reset MASTER进行清理，否则无法设置gtid。

```
mysql> show master status \G;
*************************** 1. row ***************************
             File: bin.000001
         Position: 154
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: e719a305-cb6b-11e8-841c-525400dbd857:1-106016597
1 row in set (0.00 sec)
```

3、执行reset master

4、执行GTID_PURGED,与xtrabackup_binlog_info的事务一致

```
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
SET @@GLOBAL.GTID_PURGED='c9c73c70-c089-11e7-8544-00163e0ad76e:1-9519670';
SET @@SESSION.SQL_LOG_BIN = @MYSQLDUMP_TEMP_LOG_BIN;
```

5、change master

`change master to master_host='192.168.2.71', master_port=3306, master_user='repl', master_password='REPLsafe!@#$71', MASTER_AUTO_POSITION = 1;`

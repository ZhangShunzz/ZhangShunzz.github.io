---
layout: post
title:  Xtrbackup 备份 Mysql
date:   2018-08-22 17:13:00 +0800
categories: 技术
tag: [Mysql,Xtrbackup]
---


mysqldump备份方式是采用逻辑备份，但是它最大的缺陷就是备份和恢复速度慢对于一个小于50G的数据库而言，这个速度还是能接受的，但如果数据库非常大，那再使用mysqldump备份就不太适合了。

xtrabackup是一种物理备份工具，通过协议连接到mysql服务端，然后读取并复制innodb底层的"数据块"，完成所谓的"物理备份"。

支持对innodb进行热备、增量备份、差量备份

Xtrabackup是由percona提供的mysql数据库备份工具，特点：

(1)备份过程快速、可靠；

(2)备份过程不会打断正在执行的事务；

(3)能够基于压缩等功能节约磁盘空间和流量；

(4)自动实现备份检验；

(5)还原速度快。

**1、安装**

官网：https://www.percona.com/doc/percona-xtrabackup/LATEST/installation/yum_repo.html

ubuntu：

wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb

dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb

apt-get update

apt-get install percona-xtrabackup-24

centos:

yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm

yum install -y percona-xtrabackup-24.x86_64

**2、备份**

（1）完全备份

`innobackupex --user=<用户名> --password=<密码> /path/to/backup`

在备份的同时，备份数据会在备份目录下创建一个以当前日期时间为名字的目录存放备份文件

注：各文件说明

xtrabackup_checkpoints：备份类型（如完全或增量）、备份状态（如是否已经为prepared状态）和LSN(日志序列号)范围信息；

每个InnoDB页(通常为16k大小)都会包含一个日志序列号，即LSN。LSN是整个数据库系统的系统版本号，每个页面相关的LSN能够表明此页面最近是如何发生改变的。

xtrabackup_binlog_info：mysql服务器当前正在使用的二进制日志文件及至备份这一刻为止二进制日志事件的位置。

xtrabackup_binlog_pos_innodb：二进制日志文件及用于InnoDB或XtraDB表的二进制日志文件的当前position。

xtrabackup_binary：备份中用到的xtrabackup的可执行文件；

backup-my.cnf：备份命令用到的配置选项信息；

（2）准备(prepare)一个完全备份

`innobackupex --apply-log <完全备份的目录名称>`

一般情况下，在备份完成后，数据尚且不能用于恢复操作，因为备份的数据中可能会包含尚未提交的事务或已经提交但尚未同步至数据文件中的事务。因此，此时数据文件仍处理不一致状态。“准备”的主要作用正是通过回滚未提交的事务及同步已经提交的事务至数据文件也使得数据文件处于一致性状态。在准备（prepare）过程结束后，InnoDB表数据已经前滚到整个备份结束的点，而不是回滚到xtrabackup刚开始时的点。

（3）还原数据库

`innobackupex --defaults-file=/etc/my.cnf --copy-back /opt/mysqlbackup/full/2016-09-12_11-29-55/ （<完全备份目录>）`

**注意，在实现恢复时，需要关闭MySQL，并且删除/mydata/data下的所有东西**

如果执行成功，其输出信息的最后几行通常如下：
innobackupex: Starting to copy InnoDB log files
innobackupex: in '/backup/2016-03-03_11-21-31'
innobackupex: back to original InnoDB log directory '/mydata/data'
innobackupex: Finished copying back files.

160303 11:38:02 innobackupex: completed OK!

确保如上信息的最后一行出现"innobackupex: completed ok!"

`当数据恢复到数据目录后，还需要确保所有数据文件的所属均为正确的用户，如mysql；否则，在启动mysqld前还需要事先修改数据文件的所属`

`chown -R mysql.mysql /mydata/data`

（4）使用innobackupex进行增量备份

要实现第一次增量备份，可以使用下面的命令：

#innobackupex --incremental /backup --incremental-basedir=<完全备份目录>

其中，BASEDIR指的是完全备份所在的目录，此命令执行结束后，innobackupex命令会在/backup目录中创建一个以新的时间命名的目录及存放所有的增量备份数据。另外，在执行过增量备份后再一次进行增量备份时，其--incremental-basedir应该指向上一次增量备份所在的目录

需要注意的是，增量备份仅能应用于InnoDB或xtrDB表，对于MyISAM表而言，执行增量备份其实进行的是完全备份。

准备增量备份与准备完全备份有着一些不同，尤其要注意的是：
1)需要在每个备份(包括完全和增量备份上)，将已经提交的事务进行重放。重放后，所有的备份数据将合并到完全备份上
2)基于所有的备份将未提交的事务进行回滚

于是，操作就变成了：
#innobackupex --apply-log --redo-only BASEDIR

接着执行：
#innobackupex --apply-log --redo-only BASEDIR --incremental-dir=INCREMENTAL-DIR-1

而后是第二个增量备份：
#innobackupex --apply-log --redo-only BASEDIR --incremental-dir=INCREMENTAL-DIR-2

恢复时，直接使用第1次的完全备份即可
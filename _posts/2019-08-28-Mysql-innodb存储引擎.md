---
layout: post
title: "Mysql——innodb存储引擎"
subtitle: "InnoDB，是MySQL的数据库引擎之一，现为MySQL的默认存储引擎"
author: "zhangshun"
header-img: "img/post-bg-farewell-flash.jpg"
header-mask: 0.2
tags:
  - Mysql
---

#### Innodb介绍
---

InnoDB，是MySQL的数据库引擎之一，现为MySQL的默认存储引擎，为MySQL AB发布binary的标准之一。

事务型数据库的首选引擎，支持ACID事务，支持行级锁定。InnoDB是为处理巨大数据量时的最大性能设计。InnoDB存储引擎完全与MySQL服务器整合，InnoDB存储引擎为在主内存中缓存数据和索引而维持它自己的缓冲池。InnoDB存储它的表&索引在一个表空间中，表空间可以包含数个文件（或原始磁盘分区）。

#### Innodb：数据的存储
---

在 InnoDB 存储引擎中，所有的数据都被逻辑地存放在表空间中，表空间（tablespace）是存储引擎中最高的存储逻辑单位，在表空间的下面又包括段（segment）、区（extent）、页（page）

![数据存储单位](/img/in-post/2019-08-28-Mysql-innodb存储引擎/数据存储单位.png)

同一个数据库实例的所有表空间都有相同的页大小；默认情况下，表空间中的页大小都为 16KB，当然也可以通过改变 innodb_page_size 选项对默认大小进行修改，需要注意的是不同的页大小最终也会导致区大小的不同：

![数据存储对照表](/img/in-post/2019-08-28-Mysql-innodb存储引擎/数据存储对照表.png)

**如何存储表**

Mysql使用Innodb存储表时，会将`表的定义`和`数据索引`等信息分开存储，其中前者存储在.frm文件中，后者存储在.ibd文件中

.frm
无论在 MySQL 中选择了哪个存储引擎，所有的 MySQL 表都会在硬盘上创建一个 .frm 文件用来描述表的格式或者说定义；.frm 文件的格式在不同的平台上都是相同的。

.ibd
InnoDB 中用于存储数据的文件总共有两个部分，一是系统表空间文件，包括 ibdata1、ibdata2 等文件，其中存储了 InnoDB 系统信息和用户数据库表数据和索引，是所有表公用的。
当打开 innodb_file_per_table 选项时，.ibd 文件就是每一个表独有的表空间，文件存储了当前表的数据和相关的索引数据。

与现有的大多数存储引擎一样，InnoDB 使用页作为磁盘管理的最小单位；数据在 InnoDB 存储引擎中都是按行存储的，每个 16KB 大小的页中可以存放 2-200 行的记录。

**Innodb页面存储格式**

![页面存储格式](/img/in-post/2019-08-28-Mysql-innodb存储引擎/页面存储格式.png)

Fil Header/Fil Trailer 关心的是记录页的头信息
Page Header:记录页面的控制信息，共占150字节，包括页的左右兄弟页面指针、页的空间使用情况等
Infimum:记录的是比该页中任何主键都小的值
Supermum:记录的是比该页中任何主键都大的值
User Records:是整个页面中真正存放行记录的部分
Free Space:未分配空间，指页面未使用的存储空间，随着页面不断使用，未分配空间将会越来越小
Page Directory:页面最后部分，占8个字节，主要存储页面的校验信息

B+ 树在查找对应的记录时，并不会直接从树中找出对应的行记录，它只能获取记录所在的页，将整个页加载到内存中，再通过 Page Directory 中存储的稀疏索引和 n_owned、next_record 属性取出对应的记录，不过因为这一操作是在内存中进行的，所以通常会忽略这部分查找的耗时。

#### Innodb：索引的实现

有几个索引就有几个B+树。

聚簇索引的叶子节点为磁盘上的真实数据。非聚簇索引的叶子节点还是索引，指向聚簇索引B+树。

当多加一个索引时，就会多生成一颗非聚簇索引树。在做插入操作的时候，需要同时维护这几颗树的变化！因此，如果索引太多，插入性能就会下降！

**主键是用自增还是UUID?**

肯定答自增啊。innodb 中的主键是聚簇索引。如果主键是自增的，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。如果不是自增主键，那么可能会在中间插入，就会引发页的分裂，产生很多表碎片！

索引是数据库中非常非常重要的概念，它是存储引擎能够快速定位记录的秘密武器，对于提升数据库的性能、减轻数据库服务器的负担有着非常重要的作用；索引优化是对查询性能优化的最有效手段，它能够轻松地将查询的性能提高几个数量级。

**聚簇索引和辅助索引**

数据库中的 B+ 树索引可以分为聚簇索引（clustered index）和辅助索引（secondary index），它们之间的最大区别就是，聚簇索引中存放着一条行记录的全部信息，而辅助索引中只包含索引列和一个用于查找对应行记录的『书签』。

在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。

![聚集索引](/img/in-post/2019-08-28-Mysql-innodb存储引擎/聚集索引.png)

可以看到叶节点包含了完整的数据记录。这种索引叫做聚簇索引。因为InnoDB的数据文件本身要按主键聚集，所以InnoDB要求表必须有主键（MyISAM可以没有），如果没有显式指定，则MySQL系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形。

辅助索引，InnoDB的所有辅助索引都引用主键作为data域。

![辅助索引](/img/in-post/2019-08-28-Mysql-innodb存储引擎/辅助索引.png)


聚簇索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

如下图：

![辅助索引转聚集索引](/img/in-post/2019-08-28-Mysql-innodb存储引擎/辅助索引转聚集索引.png)

#### Innodb：事务与隔离级别

事务遵循了包括原子性在内的ACID四大特性：原子性、一致性、隔离性和持久性

**几种隔离级别**

事务的隔离性是数据库处理数据的几大基础之一，而隔离级别其实就是提供给用户用于在性能和可靠性做出选择和权衡的配置项。Innodb遵循了SQL:1992标准中的四种隔离级别：READ UNCOMMITED、READ COMMITED、REPEATABLE READ和SERIALIZABLE

`READ UNCOMMITED`:使用查询语句不会加锁，可能会读到未提交的事务
`READ COMMITED`:只对记录加记录锁，而不会在记录之间加间隙锁，所以允许新的记录插入到被锁定记录的附近，所以再多次使用查询语句时，可能得到不同的结果
`REPEATABLE READ`:多次读取同一范围的数据会返回第一次查询的快照，不会返回不同的数据行，但是可能发生幻读
`SERIALIZABLE`:Innodb隐式地将全部的查询语句加上共享锁，解决了幻读的问题

(1)**脏读:在一个事务中，读取了其他事务未提交的数据**

当事务级别为READ UNCOMMITED时，我们在SESSION 2中插入的`未提交`数据在SESSION 1中是可以访问的

![脏读](/img/in-post/2019-08-28-Mysql-innodb存储引擎/脏读.png)

(2)**不可重复读：在一个事务中，同一行记录被访问了两次却得到了不同的结果**

当事务的隔离级别时READ COMMITED时，虽然解决了脏读的问题，但是如果在SESSION 1先查询了一行数据，在这之后SESSION 2中修改了同一行数据并且提交了修改，在这时，如果SESSION 1中再次使用相同的查询语句，就会发现两次查询的结果不一样

![不可重复读](/img/in-post/2019-08-28-Mysql-innodb存储引擎/不可重复读.png)

(3)**幻读：在一个事务中，同一个范围内的记录被读取时，其他事务向这个范围添加了记录**

重新开启了两个会话SESSION 1和SESSION 2，在SESSION 1中我们查询全表的信息，没有得到任何记录；在SESSION 2中向表中插入一条数据并提交；由于REPEATABLE READ的原因，再次查询全表的数据时，我们获得的仍然是空集，但是在向表中插入同样的数据却出现了错误

![幻读](/img/in-post/2019-08-28-Mysql-innodb存储引擎/幻读.png)
这种现象在数据库中就被称为幻读，虽然我们使用查询语句得到了一个空的集合，但是插入数据时却得到了错误，好像之前的查询是幻觉一样

#### Innodb：事务日志

Innodb使用undo、redo log来保证事务原子性、一致性和持久性，同时采用`预写日志`方式将随机写入变成顺序追加写入，提升事务性能。

(1)**Undo log**
`Undo log`主要是为了实现事务的原子性，在Mysql数据库Innodb存储引擎中，还用`Undo log`来实现多版本并发控制(简称：MVCC)。
`Undo log`的原理很简单，就是记录事务变更前的状态，为了满足事务的原子性，在操作任何数据之前，首先将数据备份到一个地方，也就是`Undo log`，然后进行数据的修改。如果出现了错误或者用户执行了ROLLBACK语句，系统可以利用`Undo log`中的备份将数据恢复到事务开始之前的状态。

`undo log`是为回滚而用，具体内容就是copy事务前的数据内容（行）到innodb_buffer_pool中的undo buffer（或者叫undo page），再合适的时间把undo buffer中的内容刷新到磁盘。undo buffer跟redo buffer一样，也是环形缓冲，但当缓冲满的时候，undo buffer中的内容也会刷新到磁盘；并且innodb_purge_threads后台线程会清空undo页、清理"deleted"page，Innodb将`Undo log`看作数据，因此记录`Undo log`的操作也会记录到`redo log`中。这样`undo log`就可以像数据一样缓存起来。

(2)**Redo log**
记录事务将要变更后的状态。事务提交时，只要将`redo log`持久化即可，数据可在内存中变更。当系统崩溃时，虽然数据没有落盘，但是`redo log`已持久化，系统可以根据`redo log`的内容，将所有数据恢复到最新的状态。

注意是先写redo buffer，然后再修改buffer cachez中的页，因为修改是以页为单位的，所以先写redo才能保证一个大事务commit的时候，redo已经刷新的差不多了。反过来说，如果是先写buffer cache中的页，然后再写redo buffer，就可能会有很多的redo需要写，因为一个页可能有很多行数据；而很多行数据产生的redo也可能比较多，那么commit的时候，就会有很多redo需要写。

和`Undo log`相反，`Redo log`记录的是新数据的备份。在事务提交前，只要将`Redo log`持久化即可，不需要将数据持久化，当系统崩溃时，虽然数据没有持久化，但是`Redo log`已经持久化。系统可以根据`Redo log`的内容，将所有数据恢复到最新的状态。
需要注意的是，事务过程中，先把redo写进redo log buffer中，然后Mysql后台进程page cleaner thread适当的去刷新redo到`redo log`中永久保存。

(3)**checkpoint**
随着时间的积累，`redo log`会变的越来越大，如果每次都从第一条记录开始恢复，恢复的过程就会很慢。为了减少恢复的时间，就引入了`checkpoint`机制。定期将`databuffer`的内容刷新到磁盘`datafile`内，然后清除`checkpoint`之前的`redo log`。

(4)**自动恢复**
Innodb通过加载最新的快照，然后重放最近的`checkpoint`点之后的`redo log`事务(包括未提交和回滚了的)，再通过`undo log`回滚那些未提交的事务，来完成数据恢复。需要注意的地方是，`undo log`其实也是行数据，对其写操作也会记录到`redo log`内，即`undo log`也是通过`redo log`来保证持久化的。

#### Innodb：事务执行流程

![事务执行流程](/img/in-post/2019-08-28-Mysql-innodb存储引擎/事务流程.png)

针对`update user set name='zhangsan' where id=5;`

1）事务开始；
2）对id=5的这条数据上排他锁，并且给id=5两边的临近范围加gap锁，防止别的事务insert新数据；
3）将需要修改的数据页PIN到innodb_buffer_cache中；
4）记录id=5的数据到undo log；
5）记录修改id=5的信息到redo log；
6）修改id=5的name='zhangsan'；
7）刷新innodb_buffer_cache中脏数据到底层磁盘，这个过程跟commit无关；
8）commit，触发page cleaner线程，把redo从redo buffer cache中刷新到底层磁盘，并且刷新innodb_buffer_cache中脏数据到底层磁盘；
9）记录binlog
10）事务结束

#### Innodb：缓存池

![innodb缓存池](/img/in-post/2019-08-28-Mysql-innodb存储引擎/innodb缓存池.png)

用预读的方式将数据页加载到innodb_buffer_pool中

`预读`：磁盘读写，并不是按需读取，而是按页读取，一次至少读一页数据（一般是16K），如果未来要读取的数据就在页中，就能够省去后续的磁盘IO，提高效率。

**预读为什么有效？**
数据访问，通常都遵循“集中读写”的原则，使用一些数据，大概率会使用附近的数据，这就是所谓的“局部性原理”，它表明提前加载是有效的，确实能够减少磁盘IO。

**InnoDB是以什么算法，来管理这些缓冲页呢？**
最容易想到的，就是LRU(Least recently used)。

![LRU](/img/in-post/2019-08-28-Mysql-innodb存储引擎/LRU.png)

影响Innodb_buffer_pool的三个比较重要的参数
```
mysql> show variables like '%innodb_buffer_pool_size%';
+-------------------------+-------------+
| Variable_name           | Value       |
+-------------------------+-------------+
| innodb_buffer_pool_size | 11811160064 |
+-------------------------+-------------+
1 row in set (0.00 sec)

mysql> show variables like '%innodb_old_blocks_pct%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
1 row in set (0.00 sec)

mysql> show variables like '%innodb_old_blocks_time%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
1 row in set (0.00 sec)
```

`innodb_buffer_pool_size`:配置缓冲池的大小，在内存允许的情况下，DBA往往会建议调大这个参数，越多数据和索引放到内存里，数据库的性能会越好。
`innodb_old_blocks_pct`:老生代占整个LRU链长度的比例，默认是37，即整个LRU中新生代与老生代长度比例是63:37。
`innodb_old_blocks_time`:老生代停留时间窗口，单位是毫秒，默认是1000，即同时满足“被访问”与“在老生代停留时间超过1秒”两个条件，才会被插入到新生代头部。

**innodb_buffer_pool总结**
1）缓冲池(buffer pool)是一种常见的降低磁盘访问的机制；
2）缓冲池通常以页(page)为单位缓存数据；
3）缓冲池的常见管理算法是LRU，memcache，OS，InnoDB都使用了这种算法；
4）InnoDB对普通LRU进行了优化：
- 将缓冲池分为老生代和新生代，入缓冲池的页，优先进入老生代，页被访问，才进入新生代，以解决预读失效的问题
- 页被访问，且在老生代停留时间超过配置阈值的，才进入新生代，以解决批量数据访问，大量热数据淘汰的问题

#### 参考

[缓冲池(buffer pool)，这次彻底懂了！！！](https://juejin.im/post/5d11a79ee51d4555e372a624)
[MySQL innodb引擎的事务执行过程](https://www.linuxidc.com/Linux/2018-04/152080.htm#)
[『浅入浅出』MySQL 和 InnoDB](https://draveness.me/mysql-innodb)
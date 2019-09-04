---
layout: post
title:  Mysql增量备份的一种方法 selectintooutfile与loaddatainfile
date:   2018-10-22 17:51:00 +0800
categories: 技术
tag: Mysql
---


select into outfile用法
---
MySQL中，可以使用SELECT...INTO OUTFILE语句将表的内容导出为一个文本文件。

    SELECT [列名] FROM table [WHERE 语句]
    		INTO OUTFILE '目标文件' [OPTION];

“OPTION”参数为可选参数选项，其可能的取值有：

- FIELDS TERMINATED BY '字符串'：设置字符串为字段之间的分隔符，可以为单个或多个字符。默认值是“\t”。

- FIELDS ENCLOSED BY '字符'：设置字符来括住字段的值，只能为单个字符。默认情况下不使用任何符号。

- FIELDS OPTIONALLY ENCLOSED BY '字符'：设置字符来括住CHAR、VARCHAR和TEXT等字符型字段。默认情况下不使用任何符号。
FIELDS ESCAPED BY '字符'：设置转义字符，只能为单个字符。默认值为“\”。

- LINES STARTING BY '字符串'：设置每行数据开头的字符，可以为单个或多个字符。默认情况下不使用任何字符。

- LINES TERMINATED BY '字符串'：设置每行数据结尾的字符，可以为单个或多个字符。默认值是“\n”。

**举个栗子：**

    select * from raptor.loan where DATE_FORMAT(create_at,'%Y-%m-%d')="2018-10-21" or DATE_FORMAT(update_at,'%Y-%m-%d')="2018-10-21" or repaired=1 into outfile "/tmp/raptor_loan_incre_2018-10-21.csv" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';

将raptor.loan表2018-10-21日的增量数据导出到excel表，字符串为字段之间的分隔符 | ，每行数据结尾的字符 \r\n回车

load data infile用法
---

可以将select into outfile 导出的文本文件导入数据库

load data infile语句从一个文本文件中以很高的速度读入一个表中。使用这个命令之前，mysqld进程（服务）必须已经在运行。为了安全原因，当读取位于服务器上的文本文件时，文件必须处于数据库目录或可被所有人读取。

    LOAD DATA INFILE "/path/to/file" INTO TABLE table_name;
    注意：如果导出时用到了FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n'语句，那么LOAD时也要加上同样的分隔限制语句。还要注意编码问题。

**举个栗子：**

    LOAD DATA INFILE '/tmp/raptor_loan_track_incre_2018-10-21.csv' REPLACE INTO TABLE raptor.loan_track FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';

将文件/tmp/raptor_loan_track_incre_2018-10-21.csv导入raptor.loan_track表

数据库差量数据备份脚本
---

```
#!/bin/bash
date2=$(date -d yesterday +%Y-%m-%d)
sql1="select * from raptor.loan where DATE_FORMAT(create_at,'%Y-%m-%d')=\"${date2}\" or DATE_FORMAT(update_at,'%Y-%m-%d')=\"${date2}\" or repaired=1 into outfile \"/tmp/raptor_loan_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql2="select * from raptor.loan_track where DATE_FORMAT(create_at,'%Y-%m-%d')=\"${date2}\" or DATE_FORMAT(update_at,'%Y-%m-%d')=\"${date2}\" or repaired=1 into outfile \"/tmp/raptor_loan_track_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql3="select * from loan_extend where DATE_FORMAT(create_at,'%Y-%m-%d')=\"${date2}\" into outfile \"/tmp/raptor_loan_extend_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql4="select * from tenants into outfile \"/tmp/raptor_tenants_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql5="select * from user_privilege into outfile \"/tmp/raptor_user_privilege_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql6="select * from users into outfile \"/tmp/raptor_users_incre_${date2}.csv\" FIELDS TERMINATED BY '|' LINES TERMINATED BY '\r\n';"
sql7="update loan_track set repaired=0 where repaired=1;"
sql8="update loan set repaired=0 where repaired=1;"
#sql9="select count(1) from raptor.loan where DATE_FORMAT(create_at,'%Y-%m-%d')=\"${date2}\" or DATE_FORMAT(update_at,'%Y-%m-%d')=\"${date2}\" or repaired=1"
execute_sql(){
    /usr/local/mysql/bin/mysql -u xxxxxxxxx -p'xxxxxxxxxxxxxxxx'  $DB -e "${1}"
    if [ -f /tmp/${2}_incre_${date2}.csv ]
        then
            scp -P xxxxx /tmp/${2}_incre_${date2}.csv hadoop_ftp@xx.xx.xx.xx:/data/sftp_docker/hadoop_ftp/raptor_repair/
            if [ $? == 0 ]
                then
                    cd /tmp/
                    rm ${2}_incre_${date2}.csv
            fi
    fi
}
/usr/local/mysql/bin/mysql -u xxxxxxxx -p'xxxxxxxxxxxxxxx'  raptor -e "${sql9}" > /root/sh/count.log
execute_sql "${sql1}" "raptor_loan"
execute_sql "${sql2}" "raptor_loan_track"
execute_sql "${sql3}" "raptor_loan_extend"
execute_sql "${sql4}" "raptor_tenants"
execute_sql "${sql5}" "raptor_user_privilege"
execute_sql "${sql6}" "raptor_users"
execute_sql "${sql7}"
execute_sql "${sql8}"
```

**注意：**

在操作导入的时候发现会出现主键冲突报错，原因是有两张表是全量备份的，再导入时可以使用参数REPLACE，MySQL会把相同的先干掉，再插入新的值。replace关键词控制对现有的唯一键记录的重复的处理。如果你指定replace，新行将代替有相同的唯一键值的现有行
---
layout: post
title:  python脚本实现mysql增量备份 
date:   2018-12-17 14:30:00 +0800
categories: 技术
tag: [python,mysql]
---


**mydb.py**

```
#!/usr/bin/env python
#coding=utf-8
import MySQLdb as mdb
from Crypto.Cipher import AES
 
#算法
key = '0x7jz75nmrjx5k52lcqpybm12b1frbmn'
iv='1234567812345678'
BS = 16
pad   = lambda s: s + (BS - len(s) % BS) * '\0'
 
#加密
def encrypt(text):
    aes_obj = AES.new(key, AES.MODE_CBC,iv)
    buf=aes_obj.encrypt(pad(text)).encode('hex')
    return buf
#解密
def decrypt(text):
    aes_obj = AES.new(key, AES.MODE_CBC,iv)
    buf=aes_obj.decrypt(text.decode('hex'))
    return buf
#生成insert语句,从mysql自带的information_schema库中获取字段
def genSchema(host,user,passwd,table,*keys):
    dbname = "information_schema"
    con = None
    sqlSel = "select column_name from columns where table_name='"+table+ "'"
    con = mdb.connect(host=host,user=user,passwd=passwd,db=dbname,charset="utf8", unix_socket="/data/mysql/data/mysql.sock");
    cur = con.cursor()
    cur.execute(sqlSel)
    rows = cur.fetchall()
    insert =""
    update =""
    for row in rows:
        txtRow = ""
        insert = insert+ row[0] +","
        if(row[0] == keys[0] or row[0] == keys[1]):
            continue
        else:
            update = update +row[0]+"=values("+row[0]+"),"
    fieldsIn = insert.rstrip(",")
    fieldsUpdate = update.rstrip(",")
    if con:
        con.close()
    sqlPre = "insert into " + table + " (" + fieldsIn + ")  values ("
    sqlLast = ") on duplicate key update " + fieldsUpdate
    f=open('struc%s.txt'%table,'w')
    f.write(sqlPre+"\n")
    f.write(sqlLast+"\n")
    f.close()
#获取insert语句
def getSchema(table):
    f=open('struc%s.txt'%table)
    index = 0
    sqlInsert = ""
    sqlUpdate = ""
    while True:
        line = f.readline()
        if len(line) ==0:break
        if(index == 0):
            sqlInsert =line.rstrip("\n")
        if(index == 1):
            sqlUpdate = line.rstrip("\n")
        index = index+1
    return (sqlInsert, sqlUpdate)
```

**syncData.py**

```
#!/usr/bin/env python
#coding=utf-8
import os
import sys
import time
import types
import MySQLdb as mdb
import ConfigParser
from mydb import genSchema
from mydb import getSchema
from mydb import encrypt
from mydb import decrypt
 
reload(sys)
sys.setdefaultencoding('utf8')
 
if __name__ == '__main__':
    if (len(sys.argv)<2):
        print sys.argv[0],' export|import  start end'
        sys.exit(1)
method=sys.argv[1]
 
start = ""
end   = ""
myday = '2018-01-01'
if(len(sys.argv)==4):
    start = sys.argv[2] + " 00:00:00"
    end   = sys.argv[3] + " 23:59:59"
else:
    myday= time.strftime('%Y-%m-%d',time.localtime(time.time()-24*3600))
    #myday = '2018-12-10'
    start = myday+" 00:00:00"
    end   = myday+" 23:59:59"
 
#加载config.ini
config = ConfigParser.ConfigParser()
config.readfp(open("config.ini","rb"))
 
fromHost = config.get('cloud_blacklist',"host")
fromUser = config.get('cloud_blacklist',"user")
fromPass = decrypt(config.get('cloud_blacklist',"pass")).replace("\x00","")
toHost = config.get('boxing_blacklist',"host")
toUser = config.get('boxing_blacklist',"user")
toPass = decrypt(config.get('boxing_blacklist',"pass")).replace("\x00","")
 
dbname = "blacklist"
con = None
Blacklistbackup = {
    'black_record':['id','uuid'],
    'temp_black_record':['id']
}
#从阿里云导出
if(method == 'export'):
    for key in Blacklistbackup:
        try:
            Lenth = len(Blacklistbackup[key])
            if Lenth == 2:
                genSchema(fromHost,fromUser,fromPass,key,Blacklistbackup[key][0],Blacklistbackup[key][1])
            elif Lenth == 1:
                genSchema(fromHost,fromUser,fromPass,key,Blacklistbackup[key][0],key,Blacklistbackup[key][0])  
            con = mdb.connect(host=fromHost,user=fromUser,passwd=fromPass,db=dbname,charset="utf8", unix_socket="/data/mysql/data/mysql.sock");
            cur = con.cursor()
            sqlSel = "SELECT * from %s where update_at between '"%key+start+"' and '"+end+ "'"
            print sqlSel
            cur.execute(sqlSel)
            rows = cur.fetchall()
            f=open('data%s.txt'%key,'w')
            indexNum = 0
            for row in rows:
                txtRow = ""
                for field in row:
                    if(type(field) is types.UnicodeType):
                        field1=field.replace(chr(10), "")
                        txtRow += field1+"|||"
                    elif(type(field) is not types.StringType):
                        txtRow += str(field)+"|||"
                    else:
                        txtRow += field+"|||"
                line=txtRow.rstrip("|||")
                f.write(line+"\n")
                indexNum = indexNum+1
            f.close()
            print "export %s totals="%key+str(indexNum)
        except Exception as e:
            print e
        finally:
            if con:con.close()
 
#导入博兴kvm
if(method == 'import'):
    for key in Blacklistbackup:
        try:
            (preSql,lastSql) = getSchema(key)
            con = mdb.connect(host=toHost,user=toUser,passwd=toPass,db=dbname,charset="utf8",unix_socket="/data/mysql/data/mysql.sock");
            cur = con.cursor()
            f=open('data%s.txt'%key)
            index = 0
            uuid=""
            while True:
                line=f.readline().rstrip("\n")
                if len(line) ==0:break
                array = line.split("|||")
                values=""
                for aa in array:
                    if(aa.isdigit()):
                        values += str(aa)+","
                    elif(aa=='None'):
                        values += "NULL,"
                    else:
                        values += "'"+aa.replace("'","\\\'")+"',"
                uuid=array[1]
                myvalues=values.rstrip(',')
                sqlInsert = preSql + myvalues + lastSql
                index=index+1
                cur.execute(sqlInsert)
                if(index % 100 == 0):
                    con.commit()
            con.commit()
            f.close()
            print "totals %s import="%key+str(index)
        except Exception as e:
            print e
            print uuid
        finally:
            if con:
                con.close()
```

**mail.py**

```
#!/usr/bin/env python
#coding=utf-8
from email.mime.text import MIMEText
import smtplib
import os
from mydb import decrypt
 
def mail():
    mail_host = 'smtp.intellicredit.cn'
    mail_user = 'xxxxxxx@intellicredit.cn'
    mail_pass = decrypt('加密后的密码').replace("\x00","")
    sender = 'xxxxxxx@intellicredit.cn'
    receivers = ['zhangshun@intellicredit.cn']
    temp_black_record = os.popen('cat sync.log |grep temp_black_record|tail -3').read() #黑名单temp_black_record表增量备份
    black_record = os.popen('cat sync.log |grep black_record|grep -v temp_black_record|tail -3').read() #黑名单black_record表增量备份
    message = MIMEText('%s%s'%(temp_black_record,black_record),'plain','utf-8')
    message['Subject'] = '黑名单数据增量备份'
    message['From'] = sender
    message['To'] = receivers[0]
    try:
        smtpObj = smtplib.SMTP()
        smtpObj.connect(mail_host,25)
        smtpObj.login(mail_user,mail_pass)
        smtpObj.sendmail(sender,receivers,message.as_string())
        smtpObj.quit()
    except Exception as e:
        print(e)
if __name__ == '__main__':<br>　　mail()
```

**config.ini**

```
[cloud_blacklist]
host=1.1.1.1
user=user
pass=加密后的passwd
 
[boxing_blacklist]
host=1.1.1.1
user=user
pass=加密后的passwd
 
[general]
mailUser=xxxxxxx@intellicredit.cn
mailPass=加密后的passwd
mailHost=smtp.intellicredit.cn
[security]
mailUser=xxxxxxx@intellicredit.cn
mailPass=加密后的passwd
mailHost=smtp.intellicredit.cn
```
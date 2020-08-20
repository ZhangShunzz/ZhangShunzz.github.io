---
layout: post
title: "ELK：elastiflow监控网络"
author: "zhangshun"
header-img: "img/background/16.jpg"
header-mask: 0.2
tags:
  - ELK
---

### 应用场景

分析链路下的实时主机的流量交互

### 部署

安装ELK，这里选择7.8.1版本,下载安装包上传至服务器中
```
elasticsearch-7.8.1-x86_64.rpm
logstash-7.8.1.rpm
kibana-7.8.1-x86_64.rpm
```

使用 rpm 安装
```
rpm -ivh elasticsearch-7.3.2-x86_64.rpm logstash-7.3.2.rpm kibana-7.3.2-x86_64.rpm
```
重新加载系统服务并开机自启动

```
systemctl daemon-reload
systemctl enable elasticsearch.service
systemctl enable kibana.service
systemctl enable logstash.service
```
**安装elasticsearch**

修改文件句柄数
```
ulimit -n 65536
vim /etc/security/limits.conf
*　　soft　　nofile　　65536

*　　hard　　nofile　　65536
```

创建es的数据目录
```
mkdir -p /data/elk_data
chown elasticsearch:elasticsearch /data/elk_data
```

配置传输层TLS/SSL加密传输
```
/usr/share/elasticsearch/bin/elasticsearch-certutil ca
ENTER ENTER
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
ENTER ENTER ENTER

mkdir /etc/elasticsearch/certs 
cp /usr/share/elasticsearch/elastic-* /etc/elasticsearch/certs
chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```
运行 Elasticsearch 的密码配置工具，为各种内置用户生成随机的密码。
```
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
```
修改es配置文件
```
vim /etc/elasticsearch/elasticsearch.yml
cluster.name: my-elk-culster
node.name: node1
path.data: /data/elk_data
path.logs: /var/log/elasticsearch/
bootstrap.memory_lock: false
network.host: 0.0.0.0
http.port: 9200
cluster.initial_master_nodes: ["192.168.32.100:9300"]
discovery.zen.ping.unicast.hosts: ["192.168.32.100"]
xpack.security.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
indices.query.bool.max_clause_count: 8192
search.max_buckets: 100000
```

启动elasticsearch
```
systemctl start elasticsearch
```
**安装kibana**


修改kibana配置文件
```
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://127.0.0.1:9200"]
kibana.index: ".kibana"
i18n.locale: "zh-CN"
xpack.security.enabled: true
elasticsearch.username: "kibana_system"
elasticsearch.password: "ChMu94Wyh6zRAfRlRyJM"
```

启动kibana

```
systemctl start kibana
```

**安装elastiflow**

修改logstash的jvm配置
```
vim /etc/logstash/jvm.options
-Xms4g
-Xmx4g
```

添加和更新所需的Logstash插件
```
/usr/share/logstash/bin/logstash-plugin install logstash-codec-sflow
/usr/share/logstash/bin/logstash-plugin update logstash-codec-netflow
/usr/share/logstash/bin/logstash-plugin update logstash-input-udp
/usr/share/logstash/bin/logstash-plugin update logstash-input-tcp
/usr/share/logstash/bin/logstash-plugin update logstash-filter-dns
/usr/share/logstash/bin/logstash-plugin update logstash-filter-geoip
/usr/share/logstash/bin/logstash-plugin update logstash-filter-translate
```

下载第三方elastiflow
```
cd /tmp/
git clone https://github.com/robcowart/elastiflow.git
```

将elastiflow文件夹复制到相关目录,并修改elastiflow.conf
```
// 处理数据所需要的配置文件
cp -a elastiflow-master/logstash/elastiflow /etc/logstash

mkdir -p /etc/systemd/system/logstash.service.d/
// 运行elastflow时所需要的环境变量
cp -a elastiflow-master/logstash.service.d/elastiflow.conf /etc/systemd/system/logstash.service.d/

修改 /etc/systemd/system/logstash.service.d/elastiflow.conf中的es集群认证信息
$ELASTIFLOW_ES_USER
$ELASTIFLOW_ES_PASSWD
```

配置logstash pipeline文件
```
vim /etc/logstash/pipelines.yml
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"

- pipeline.id: elastiflow
  path.config: "/etc/logstash/elastiflow/conf.d/*.conf"
```

执行系统脚本
```
/usr/share/logstash/bin/system-install
```

启动logstash服务
```
systemctl daemon-reload
systemctl enable logstash
systemctl start logstash
```

将elastiflow对应的kibana模板文件导入到kibana，elastiflow/kibana/elastiflow.kibana.7.5.x.ndjson
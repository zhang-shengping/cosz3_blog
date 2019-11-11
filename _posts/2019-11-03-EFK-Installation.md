---
title: EFK installation
categories:
- elasticsearch
tags:
- EFK
- self-note
---

![front](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-installation/EFK-logging.jpg)

#  E*K 安装部署

## 准备（每个 ELK 节点）

```bash
sudo yum -y install java-openjdk-devel java-openjdk

# for Elasticsearch 7.x
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

# import GPG key
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# Clear and update your YUM package index.
sudo yum clean all
sudo yum makecache
```

## 安装 Elasticsearch

```bash
# install elasticsearch
sudo yum -y install elasticsearch

# check installation
[root@elasticsearch ~]# rpm -qi elasticsearch
Name        : elasticsearch
Epoch       : 0
Version     : 7.4.1
Release     : 1
Architecture: x86_64
Install Date: Wed Oct 23 20:24:35 2019
Group       : Application/Internet
Size        : 488087057
License     : Elastic License
Signature   : RSA/SHA512, Tue Oct 22 12:56:44 2019, Key ID d27d666cd88e42b4
Source RPM  : elasticsearch-7.4.1-1-src.rpm
Build Date  : Tue Oct 22 10:32:03 2019
Build Host  : packer-virtualbox-iso-1559162487
Relocations : /usr
Packager    : Elasticsearch
Vendor      : Elasticsearch
URL         : https://www.elastic.co/
Summary     : Distributed RESTful search engine built for the cloud
Description :
Reference documentation can be found at
https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
and the 'Elasticsearch: The Definitive Guide' book can be found at
https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html

# curl loopback ip with port 9200
[root@elasticsearch ~]# curl http://127.0.0.1:9200
{
  "name" : "elasticsearch.pdsea.f5net.com",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "2Y3lIzC0RkGmaiOhJNJuow",
  "version" : {
    "number" : "7.4.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "fc0eeb6e2c25915d63d871d344e3d0b45ea0ea1e",
    "build_date" : "2019-10-22T17:16:35.176724Z",
    "build_snapshot" : false,
    "lucene_version" : "8.2.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

# 配置 elasticsearch 监听本地端口
[root@elasticsearch ~]# vim /etc/elasticsearch/elasticsearch.ym
network.host: 0.0.0.0
discovery.seed_hosts: ["127.0.0.1"]

# 测试环境关闭防火墙
[root@elasticsearch ~]# systemctl stop firewalld.service

# 外网 curl elasticserach
➜  ~ curl http://10.145.64.127:9200
{
  "name" : "elasticsearch.pdsea.f5net.com",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "2Y3lIzC0RkGmaiOhJNJuow",
  "version" : {
    "number" : "7.4.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "fc0eeb6e2c25915d63d871d344e3d0b45ea0ea1e",
    "build_date" : "2019-10-22T17:16:35.176724Z",
    "build_snapshot" : false,
    "lucene_version" : "8.2.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## 安装 Kibana

```bash
sudo yum -y install kibana

sudo vim /etc/kibana/kibana.yml
server.host: "0.0.0.0"
server.name: "kibana.pdsea.f5net.com"
elasticsearch.hosts: ["http://10.145.64.127:9200"]

# 关闭测试环境 kibana 节点防火墙
systemctl stop firewalld.service

# 开启 kibana 服务
sudo systemctl enable --now kibana

# 使用 browser 访问测试
http://10.145.67.51:5601/
```

## 安装 Logstash

```bash
sudo yum -y install logstash

# 关闭测试环境节点防火墙
systemctl stop firewalld.service
```

[Logstash configuration maunal](https://www.elastic.co/guide/en/logstash/current/index.html)

## ELK reference

[ELK stack documentation](https://www.elastic.co/guide/index.html)

[Resources and Training](https://www.elastic.co/learn)

## 安装 Fluentd 准备工作

### setup NTP server





### 增加文件描述符最大数量

```bash
# 查看当前系统的 Liunx 文件描述符最大数量
$ ulimit -n
65535
```

如果是 1024，需要设置文件 `/etc/security/limits.conf`，增加文件描述符数量。改完需要重启。

```bash
root soft nofile 65536
root hard nofile 65536
* soft nofile 65536
* hard nofile 65536
```

### 优化内核网络参数

配置文件 `/etc/sysctl.conf`  如下，提高网络性能。

```bash
net.core.somaxconn = 1024
net.core.netdev_max_backlog = 5000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_wmem = 4096 12582912 16777216
net.ipv4.tcp_rmem = 4096 12582912 16777216
net.ipv4.tcp_max_syn_backlog = 8096
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 10240 65535
```

性能调优 refer to [How Netflix Tunes EC2 Instances for Performance](https://www.slideshare.net/brendangregg/how-netflix-tunes-ec2-instances-for-performance)。

### [CentOS7 安装 Fluentd](https://docs.fluentd.org/v/0.12/articles/install-by-rpm)

```bash
[root@fluentd ~]# wget https://toolbelt.treasuredata.com/sh/install-redhat-td-agent2.sh

[root@fluentd ~]# bash install-redhat-td-agent2.sh

# install-redhat-td-agent2.sh 中内容。

[root@fluentd ~]# cat install-redhat-td-agent2.sh
echo "=============================="
echo " td-agent Installation Script "
echo "=============================="
echo "This script requires superuser access to install rpm packages."
echo "You will be prompted for your password by sudo."

# clear any previous sudo permission
sudo -k

# run inside sudo
sudo sh <<SCRIPT

  # add GPG key
  rpm --import https://packages.treasuredata.com/GPG-KEY-td-agent

  # add treasure data repository to yum
  cat >/etc/yum.repos.d/td.repo <<'EOF';
[treasuredata]
name=TreasureData
baseurl=http://packages.treasuredata.com/2/redhat/\$releasever/\$basearch
gpgcheck=1
gpgkey=https://packages.treasuredata.com/GPG-KEY-td-agent
EOF

  # update your sources
  yum check-update

  # install the toolbelt
  yes | yum install -y td-agent

SCRIPT

# message
echo ""
echo "Installation completed. Happy Logging!"
echo ""


# 关闭测试环境节点防火墙
[root@fluentd ~]systemctl stop firewalld.service

# 开启 fluentd 服务
[root@fluentd ~]# sudo /etc/init.d/td-agent start
Starting td-agent (via systemctl):                         [  OK  ]
[root@fluentd ~]# sudo /etc/init.d/td-agent status
td-agent is running                                        [  OK  ]

# td-agent 支持的命令
$ sudo /etc/init.d/td-agent start
$ sudo /etc/init.d/td-agent stop
$ sudo /etc/init.d/td-agent restart
$ sudo /etc/init.d/td-agent status

# td-agent 配置文件路径
/etc/td-agent/td-agent.conf

# 简单更改默认配置，绑定到 0.0.0.0 IP 上。
<match td.*.*>
  @type tdlog
  apikey YOUR_API_KEY

  auto_create_table
  buffer_type file
  buffer_path /var/log/td-agent/buffer/td

  <secondary>
    @type file
    path /var/log/td-agent/failed_records
  </secondary>
</match>

## match tag=debug.** and dump to console
<match debug.**>
  @type stdout
</match>

####
## Source descriptions:
##

## built-in TCP input
## @see http://docs.fluentd.org/articles/in_forward
<source>
  @type forward
</source>

## built-in UNIX socket input
#<source>
#  @type unix
#</source>

# HTTP input
# POST http://localhost:8888/<tag>?json=<json>
# POST http://localhost:8888/td.myapp.login?json={"user"%3A"me"}
# @see http://docs.fluentd.org/articles/in_http
<source>
  @type http
  bind 0.0.0.0
  port 8888
</source>

## live debugging agent
<source>
  @type debug_agent
  bind 127.0.0.1
  port 24230
</source>


# 测试 td-agent 运行。默认配置 td-agent 接受 HTTP 请求，重定向到 /var/log/td-agent/td-agent.log 中。
[root@kibana ~] curl -X POST -d 'json={"json":"message"}' http://10.145.67.249:8888/debug.test
```

**[Fluentd configuration reference](https://docs.fluentd.org/v/0.12/configuration/config-file)**



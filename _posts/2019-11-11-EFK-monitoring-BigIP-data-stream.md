---
title: EFK 监控 F5 BigIP 数据流量
categories:
- F5
tags:
- F5
- self-note
---

![front](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/front.jpg)

# EFK 监控 F5 BigIP 流量测试

## [EFK 测试环境搭建](https://zhang-shengping.github.io/cosz3_blog/elasticsearch/2019/11/03/EFK-Installation/)

## [F5 BigIP 测试环境搭建](https://zhang-shengping.github.io/cosz3_blog/f5/2019/11/11/F5-BigIP-Test-Env/)

## 数据源测试脚本

这里使用 Python 发送多个异步 http get requests。

```python
[root@asm-replay network-tester]# cat http-test.py
import eventlet
eventlet.monkey_patch()

import requests
import random

USER_AGENTS = (
    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_7_0; en-US) AppleWebKit/534.21 (KHTML, like Gecko) Chrome/11.0.678.0 Safari/534.21",
    "Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)",
    "Mozilla/5.0 (Windows; U; Windows NT 5.0; en-US; rv:0.9.2) Gecko/20020508 Netscape6/6.1",
    "Mozilla/5.0 (X11;U; Linux i686; en-GB; rv:1.9.1) Gecko/20090624 Ubuntu/9.04 (jaunty) Firefox/3.5",
    "Opera/9.80 (X11; U; Linux i686; en-US; rv:1.9.2.3) Presto/2.2.15 Version/10.10"
)

# 这里将 domain 对应的 BigIP listener IP 配置在了 /etc/hosts 中
DOMAIN = "http://nginx-website.com"

SOURCE = ['.'.join((str(random.randint(1,254)) for _ in range(4))) for _ in range(100)]

def fetch(num):
    spoof_src = random.choice(SOURCE)
    user_agent = random.choice(USER_AGENTS)
    # 将 spoof source ip 放在 X-Forwarded-For 头部。
    headers = {'X-Forwarded-For':spoof_src, 'User-Agent':user_agent}

    r = requests.get(DOMAIN, headers = headers, timeout=20)
    return r

pool = eventlet.GreenPool(1000)
for r in range(500):
    pool.spawn(fetch, r)
pool.waitall()
```

## iRule 测试脚本

```tcl
when RULE_INIT {
    set static::general_remote_syslog_publisher "/Common/publisher-remote-syslog"
}

when CLIENT_ACCEPTED {
    set hsl [HSL::open -publisher $static::general_remote_syslog_publisher]
}

when HTTP_REQUEST {
    # 使用 iRule 将 X-Forwarded-For spoof IP 取出作为 source IP。
    set client_ip [HTTP::header X-Forwarded-For]
    set host [HTTP::header host]
    set user_agent [HTTP::header user-agent]
    set cookie [HTTP::header cookie]
}

when HTTP_RESPONSE {
    set server_ip [IP::server_addr]
    set resp_status [HTTP::status]

    HSL::send $hsl "client-ip: $client_ip\
    host: $host\
    user-agent: $user_agent\
    cookie: $cookie\
    server-ip: $server_ip\
    resp-status: $resp_status"
}
```

[iRule remote logging 简介](https://zhang-shengping.github.io/cosz3_blog/f5/2019/11/03/F5-Remote-Logging/)

## Fluentd 测试配置

```xml
[root@fluentd ~]# cat /etc/td-agent/td-agent.conf
<source>
  @type udp
  tag mytag # required
  # format json
  format /[^ ]* (?<timstamp>.*) (?<bigip-info>.* .* .* .* .*) [^ ]*: (?<client-ip>.*) [^ ]*: (?<host>.*) [^ ]*: (?<user-agent>(.* \(.*\))|(.*)) [^ ]*: (?<cookie>.*) [^ ]*: (?<server-ip>.*) [^ ]*: (?<status>.*$)/
  port 20001
  bind 0.0.0.0
</source>

<filter mytag>
  @type geoip
  geoip_lookup_keys client-ip
  backend_library geoip2_c

  <record>
    city            ${city.names.en["client-ip"]}
    geo             ${location.latitude["client-ip"]},${location.longitude["client-ip"]}
    country         ${country.iso_code["client-ip"]}
    country_name    ${country.names.en["client-ip"]}
    postal_code     ${postal.code["client-ip"]}
    region_code     ${subdivisions.0.iso_code["client-ip"]}
    region_name     ${subdivisions.0.names.en["client-ip"]}
  </record>
</filter>

# <match mytag>
  # @type stdout
# </match>

<match mytag>
  @type elasticsearch
  host 10.145.64.127
  port 9200
  logstash_format ture
  flush_interval 5s
  # index_name logstash-f5-log
</match>
```

这里通过正则表达式对源数据格式化后，再使用 `filter` 对 client-ip 数据生成对应的地理坐标。

这里生成地理坐标使用到了 Fluentd 的 `geo-ip` 插件。

插件安装：

```bash
[root@fluentd ~] sudo yum groupinstall "Development Tools"
[root@fluentd ~] /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-geoip
```

## Elasticsearch Kibana 收集/展示/监控

### 收集数据

**在 Kibana Develop Tool 中创建 Elasticserach 对应的 index 和 mapping**

```bash
PUT /fluentd
PUT /fluentd/_mapping
{
  "properties": {
    "timstamp": {
      "type": "date"
    },
    "bigip-info": {
      "type": "text"
    },
    "client-ip": {
      "type": "ip"
    },
    "host": {
      "type": "text"
    },
    "user-agent": {
      "type": "keyword"
    },
    "cookie": {
      "type": "keyword"
    },
    "server-ip": {
      "type": "ip"
    },
    "status": {
      "type": "text"
    },
    "city": {
      "type": "text"
    },
    "geo": {
      "type": "geo_point"
    },
    "country": {
      "type": "text"
    },
    "country_name": {
      "type": "text"
    },
    "postal_code": {
      "type": "text"
    },
    "region_code": {
      "type": "text"
    },
    "region_name": {
      "type": "text"
    }
  }
}
```



**在数据源节点运行 python 脚本发送 http 请求给 BigIP listener**

```bash
[root@asm-replay ~]# python http-test.py
```



**在 Kibana 上创建 index pattern**

这里 fluentd 发送过来的数据默认使用的是 `fluentd` index。

![fluentd index pattern](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/fluentd-index-pattern.jpg)



**在 Discover 中查看数据**

将 Kibana 的时间轴调整到数据包对应的 `timestamp` 上。

![Kibana Discover data](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/kibana-discover-data.jpg)



### 展示数据

**产生地理坐标地图**

在 `Visualize` 中 `create new visualization` ，选择 `Coordinate Map`。

![map-0](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/coordinate-map-0.png)



在 `Coordinate Map` 中设置图中的 `Buckets`。

![map-1](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/coordinate-map-1.png)

Kibana 有多种多样展示数据的方式，`Coordinate Map` 只是其中一种简单的展示方式，还存在更多有趣的展示和计算方式去探索。

### 监控数据

**enable x-pack 插件**

这里数据监控使用到了elasticsearch 和 kibana 的 `x-pack` 插件。在 elasticsearch 和 kibana 7.4.\* 版本中 `x-pack` 插件已经自带。

```bash
[root@elasticsearch ~]# /usr/share/elasticsearch/bin/elasticsearch-plugin install x-pack
ERROR: this distribution of Elasticsearch contains X-Pack by default

[root@kibana ~] /usr/share/kibana/bin/kibana-plugin install x-pack --allow-root
Plugin installation was unsuccessful due to error "Kibana now contains X-Pack by default, there is no longer any need to install it as it is already present."
```

使用 `x-pack` 的 `watcher` 功能，需要`license` 或者 enable `30 days trial`。

这里开启 `30 天试用`：

![x-pack-30-days-trail](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/x-pack-30-days-trail.jpg)



**设置 `watcher`**

这里设置的 watcher 是`每一分钟如果收集到的 fluentd 数据超过 100 条就发送 webhook 到一台 nginx 服务器`，这里的 `nginx-website.com` domain name 对应的 IP 地址设置在了 `kibana` 这台机器的 `/etc/hosts` 中。

![kibana-watcher](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/kibana-watcher.jpg)



多次运行之前的 python `http-test` 脚本，出发一次 `alter`，查看状态：

![watcher-firing](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/watcher-firing.jpg)

点击 `Firing` 可以查看到跟多的信息，比如 `webhook` 的返回值。

![watcher-firing-2](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/watcher-firing-2.jpg)

## Reference

**iRule**

[Getting Started with iRules: Logging & Comments](https://devcentral.f5.com/s/articles/getting-started-with-irules-logging-comments-20406)

[iRules Home](https://clouddocs.f5.com/api/irules/)

**Fluentd**

[Config File Syntax](https://docs.fluentd.org/configuration/config-file)

[ElasticSearch 自幹訪客流量分析地圖]([https://blog.toright.com/posts/5668/elasticsearch-%E8%87%AA%E5%B9%B9%E8%A8%AA%E5%AE%A2%E6%B5%81%E9%87%8F%E5%88%86%E6%9E%90%E5%9C%B0%E5%9C%96.html](https://blog.toright.com/posts/5668/elasticsearch-自幹訪客流量分析地圖.html))

**Elasticsearh**

[elasticsearch 7.4.\* 创建 index](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)

[elasticserach data type](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-data-types.html)

[Elastic Stack Features (formerly X-Pack) Alternatives](https://sematext.com/blog/x-pack-alternatives/)

[How to change the field type of data loaded in Kibana](https://discuss.elastic.co/t/how-to-change-the-field-type-of-data-loaded-in-kibana/136909)

**Kibana**

[kibana 7.4.\* doc](https://www.elastic.co/guide/en/kibana/current/index.html)

[kibana: 加载示例数据](https://www.elastic.co/guide/cn/kibana/current/index.html)


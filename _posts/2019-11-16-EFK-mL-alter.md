---
title: EFK ML 监控报警初探
categories:
- Elasticsearch
tags:
- F5
- self-note
---

![front](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/efk-monitor-bigip/front.jpg)

# Elasticsearch ML（Machine Learning） 测试

## 环境准备和配置

### 标准数据源

首先我们得产生一些有一定规律的标准数据。这里使用我之前写的 python 脚本加上 linux 的 `crontab`.

```python
[root@asm-replay ~]# cat /root/network-tester/http-test.py
import eventlet
eventlet.monkey_patch()

import requests
import random
import datetime

USER_AGENTS = (
    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_7_0; en-US) AppleWebKit/534.21 (KHTML, like Gecko) Chrome/11.0.678.0 Safari/534.21",
    "Mozilla/5.0 (Windows; U; MSIE 9.0; Windows NT 9.0; en-US)",
    "Mozilla/5.0 (Windows; U; Windows NT 5.0; en-US; rv:0.9.2) Gecko/20020508 Netscape6/6.1",
    "Mozilla/5.0 (X11;U; Linux i686; en-GB; rv:1.9.1) Gecko/20090624 Ubuntu/9.04 (jaunty) Firefox/3.5",
    "Opera/9.80 (X11; U; Linux i686; en-US; rv:1.9.2.3) Presto/2.2.15 Version/10.10"
)

DOMAIN = "http://nginx-website.com"
# DOMAIN = "http://ng1.com"
SOURCE = ['.'.join((str(random.randint(1,254)) for _ in range(4))) for _ in range(100)]

def fetch(num):
    spoof_src = random.choice(SOURCE)
    user_agent = random.choice(USER_AGENTS)
    headers = {'X-Forwarded-For':spoof_src, 'User-Agent':user_agent}

    r = requests.get(DOMAIN, headers = headers, timeout=20)
    return r

pool = eventlet.GreenPool(1000)
for r in range(600):
    pool.spawn(fetch, r)
pool.waitall()
print "finished %s"%datetime.datetime.now()
```

创建 bash 脚本

```bash
[root@asm-replay ~]# cat /root/network-tester/http-test.sh
#!/usr/bin/bash

/usr/bin/python  /root/network-tester/http-test.py >> /root/network-tester/request.log
```

使用命令编辑 crontab

```bash
[root@asm-replay ~]# crontab -u root -e

# 在 crontab 中添加配置，每 2 分钟 run 一次 http-test.sh
*/2 * * * * /root/network-tester/http-test.sh

# 保存退出后 crontab 会自动加载，可以使用一下命令查看
[root@asm-replay ~]# crontab -u root -l
*/2 * * * * /root/network-tester/http-test.sh
```

以上工作完成后， linux crontab 会每 2 分钟对 `http://nginx-website.com` host 进行 600 次随机访问，这些访问的 log 将被之前的 EFK 收集。

等候一晚时间，会在 elasticsearch 端产生大量的数据。

```bash
[root@asm-replay ~]# head network-tester/request.log

finished 2019-11-13 01:54:04.212286
finished 2019-11-13 01:56:03.841707
finished 2019-11-13 01:58:04.451095
finished 2019-11-13 02:00:06.136037
...
finished 2019-11-14 00:24:04.128633
finished 2019-11-14 00:26:05.744777
finished 2019-11-14 00:28:04.157556
```

### 异常数据源

模拟异常数据，这里使用了一个简单的Python DOS Tool [GoldenEye](https://github.com/jseidl/GoldenEye)，它使用多进程的方式，对指定某个 web host 在短时间类发送大量的 http request 将服务器带宽占满，同时也占用 server 的处理资源。

```bash
[root@asm-replay GoldenEye]# git clone https://github.com/jseidl/GoldenEye.git
[root@asm-replay ~]# cd GoldenEye/
[root@asm-replay GoldenEye]# ./goldeneye.py http://nginx-website.com
```



## 创建 Single metric ML 和 Alter

Elasticsearch 7.4.\* 的 ML 和 Watcher(Alter) 功能只有在开启 x-pack 的使用功能后才可以使用。Elasticsearch 7.4.\* 和 Kibana 7.4.\* 都自带 x-pack plugin，不需要再安装。

**创建 ML**

![ml-create](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create.jpg)

这里选择 fluentd index。

![选择 index](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-1.jpg)

选择比较简单的机器学习

![ml-create-2](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-2.jpg)

选择数据范围，这里选了全部数据范围

![ml-create-3](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-3.jpeg)

这里的 `Bucket span` 选择 `15m` (15 分钟，每 15 分钟取个 count 值用来建模)

![ml-create-4](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-4.jpg)

给这个 ML job 起一个 unique 名称。

![ml-create-5](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-5.jpg)

ML 会自动对建模的数据量进行验证

![ml-create-6](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-6.png)

创建 job

![ml-create-7](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-7.jpg)

创建 ML job 后，我们可以看到之前 DOS Tool 产生的异常值，这时我们可以在去创建实时数据观察。

![ml-create-8](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-8.jpg)

在创建实时数据观察后，我们可以创建 watch 来配置 alter 功能，这里我们选择捕获一天中的异常。这样比较容易触发 alter。点击保存。

![ml-create-9](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-9.jpg)

这里我们需要修改一下 watcher 的配置文件（alter 名称是 `ml-<ml job 名称>`），让watcher对低级别的异常数据触发 alter。

![ml-create-10](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-10.jpg)

## 触发 Alter

创建好以上 `ML` 和 `watcher` 后，我们可以去观察是否有报警存在。下图中可以看到报警被触发。

![ml-create-11](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-11.jpg)

报警的 action 是在 Elasticserach 的 log 文件中打印 log。

![ml-create-12](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-12.jpg)

查看 Elasticsearch 的 log 文件中成功打印出了 alter logging。

![ml-create-13](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/ml-create-13.jpg)



## 创建 Population ML

`Population ML` 在种群之间进行比较某个 metric，找出异常与其他种群成员的异常值和对象。

选择 elasticsearch 中 fluentd index 中的全量数据作为模型的 training data。

![elk-ml-popluation-1](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-1.jpg)

选择 population 字段，同时选择做 aggregation 的 metric。
![elk-ml-popluation](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation.jpg)

选择完 population 字段和 aggregation metric 后，也可以配置 `Multi-metric ML` 中的 `Split data` 选项，这里我选择的是 server ip，将数据按照 server ip的维度分开展示分析。

![elk-ml-popluation-2](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-2.jpg)

给 `ML job` 起一个 unique 名称。

![elk-ml-popluation-3](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-3.jpg)

Elasticsearch 对数据进行验证。

![elk-ml-popluation-4](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-4.jpg)

创建 `ML job` 。

![elk-ml-popluation-5](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-5.jpg)

![elk-ml-population-6](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-population-6.jpg)

创建完 `ML job` 后，我们可以 `view results`。在 results 中我们可以针对 `client-ip` 和 `server-ip` 两种视角观察 `anomalies`。并且可以发现 IP `210.175.240.6` 相对于其他种群成员，在某个时间段属于严重异常。

![elk-ml-popluation-7](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-7.jpg)



![elk-ml-popluation-8](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/EFK-ML-Alter/elk-ml-popluation-8.jpg)

这里就没有赘述如何去创建 `watch`, `Popluation ML` 创建 `watch` 的方式和 `Single metric ML` 创建 `watch` 的方式相同。

# Reference

[Elasticsearch 6.7: Getting started with machine learning](https://www.elastic.co/guide/en/elastic-stack-overview/6.7/ml-getting-started.html)

[youtube demo](https://www.youtube.com/watch?v=wJVgh5knV4E)

[github examples](https://github.com/elastic/examples)

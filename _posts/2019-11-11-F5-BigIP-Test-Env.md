---
title: F5 BigIP test Env Setup
categories:
- F5
tags:
- F5
- self-note
---

![front](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/front.png)

# F5 VIO 环境 BigIP 测试准备

## F5 BigIP agent 准备

![bigip](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/bigip.jpg)

## 启动 Nginx servers

![nignx](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/nignx.jpg)

## CentOS7 在安装 Nginx

```bash
# 在 /etc/yum.repos.d/nginx.repo 文件中配置 Nginx repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1

[root@nginx-1 ~] sudo yum install epel-release
[root@nginx-1 ~] sudo yum clean all
[root@nginx-1 ~] sudo yum makecache

[root@nginx-1 ~] sudo yum install nginx

[root@nginx-1 ~] sudo systemctl start nginx

# 测试环境关闭防火墙
[root@nginx-1 ~] systemctl stop firewalld.service
```

## F5 网络端口配置

**查看 F5 BigIP 设备的网络 interface，在 VIO 上找到对应的 Port**

如下：

![BigIP interface](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/bigip-interface.jpeg)

F5 BigIP 设备的 1.1 interface 在 VIO 云平台上对应的 Port（MAC 和 IP address 都可以查看到 ）。

![bigip port on VIO](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/bigip-vio-port.jpeg)

**BigIP 设备上创建 Vlan，将 interface 和某个 vlan 绑定**

![BigIP vlan](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/bigip-vlan.jpeg)

**将 vlan 和 selfip （VIO 对应 Port 的 IP 地址） 绑定**

![BigIP selfip](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/bigip-selfip.jpeg)

**在同网段机器上测试 ping selfip**

![ping-test](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/ping-test.jpeg)

## F5 负载均衡相关资源配置

**创建 VIP，<font color=Crimson>VIP IP 地址需要和 client 可达</font>**

![111571988543_.pic](/Users/pzhang/Library/Containers/com.tencent.xinWeChat/Data/Library/Application Support/com.tencent.xinWeChat/2.0b4.0.9/d2ecb0183e6aef31fd67e17414c085e3/Message/MessageTemp/9e20f478899dc29eb19741386f9343c8/Image/111571988543_.pic.jpg)

 **在 VIP 上挂载 listener，<font color=Crimson>注意这里的 listener snat 必须是automap，这样 member 包的返回包才能到达 listener</font>**

![listener](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/listener.jpg)

**在 Listener 上挂载 Pool 上**

![pool](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/pool.jpg)

**在 Pool 中加上 Members（Member 的 IP 即是 Nginx servers 的 IP）<font color=Crimson>bigIP 也需要和 Nginx server 可达</font>**

![members](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/bigip-test-env/members.jpg)

**测试 loadbalancer 资源部署情况**

<font color=Crimson>在一台可以访问到 VIP 的机器上给 VIP 配置一个 hostname</font>

```bash
[root@elasticsearch ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.10.1.37  nginx-website.com
10.10.1.128  website.com
```

通过 hostname 访问 VIP，查看是否能得到 Nginx 返回值（如下）

```
[root@elasticsearch ~]# curl -v http://website.com:8888
* About to connect() to website.com port 8888 (#0)
*   Trying 10.10.1.128...
* Connected to website.com (10.10.1.128) port 8888 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: website.com:8888
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.16.1
< Date: Fri, 25 Oct 2019 07:47:15 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Tue, 13 Aug 2019 15:04:31 GMT
< Connection: keep-alive
< ETag: "5d52d17f-264"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host website.com left intact
```

**Debug F5 loadbalancer 的互通性主要看两个：1. client 是否能连通 VIP，2. VIP 是否能连通 member**

****

以上是在 VIO 中搭建一个 F5 BigIP 简单测试环境的方法。

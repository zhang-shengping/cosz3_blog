---
title: BIGIP-kick-off
categroies:
- 部署
tags:
- F5
---

![](https://i.ytimg.com/vi/5lDLvHdSeGU/maxresdefault.jpg)

这里记载下 F5 VE VMware 的安装过程：

1. 首先登陆这个网站注册一个 90 天的序列号 https://f5.com/products/trials/product-trials

2. F5 会发送注册成功邮件

   ```tex
   You're ready to start your free 90-day trial.

   Thank you for requesting a trial. Below are the registration keys you'll need to get started. These keys expire 90 days from today if not activated.
   BIG-IP Registration key(s):
   XXXX-SYCHL-DVMJR-MEJWK-TZDXIHP (LTM, Per App VE, 3 Instances)
   XXXX-NWNSF-ZMLJQ-WFHUI-LPMCKNG (Service Scaling VE)
   BIG-IQ Registration key(s):
   XXXX-KXMSWB-MVO-UHTDOXJ-LVMWTQJ (BIG-IQ Console Node)
   XXXX-AFZGRY-QHU-CQFEBIS-SHDMNVE (BIG-IQ Data Collection Device)
   Evaluation duration: 90 days
   Contact: XXXX@XXXX
   Before you begin, please review the documentation and download the appropriate software for your environment. https://devcentral.f5.com/articles/getting-started-with-big-ip-ve-trial-22469
   Thank you,
   F5 Networks, Inc.
   ```

3. 登陆邮件中提供的链接：https://devcentral.f5.com/articles/getting-started-with-big-ip-ve-trial-22469 下载 ova 镜像。

4. 将镜像导入到 VMware 中

5. 可能访问不到管理界面的 webpage，这里需要设置一下 VMware 虚拟网卡为桥接。

6. 登陆上 F5 的管理网页，activate 下 liesence, 如果第一 （LTM, Per App VE, 3 Instances）key  用不了，可以把每一个 key 都是试下（总有一个可以的，我 activate 用的是 Service Scaling VE）。

这里就不提 Utility 怎么配置了，可以参考 F5 的官方文档。






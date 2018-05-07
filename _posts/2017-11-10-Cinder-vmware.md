---
title: Cinder 对接 Vmware
categories:
- 部署
tags:
- cinder
- vmware
---

![useful image]({{ site.url }}/cosz3_blog/assets/images/blog_pictures/storage-picture.jpg)


今天搭了个测试环境，记录下 cinder 对接 vmware 过程。

#### Openstack Cinder 中的服务

**cinder-api**:  和用户交互，接受 API 请求

**cinder-scheduler**:  调度服务，用来选择最优的存储的节点来创建卷

**cinder-volume**: 与块存储服务和例如``cinder-scheduler``的进程进行直接交互。它也可以与这些进程通过一个消息队列进行交互。[``](https://docs.openstack.org/mitaka/zh_CN/install-guide-obs/common/get_started_block_storage.html#id1)cinder-volume``服务响应送到块存储服务的读写请求来维持状态。它也可以和多种存储提供者在驱动架构下进行交互。

**cinder-backup**:  [``](https://docs.openstack.org/mitaka/zh_CN/install-guide-obs/common/get_started_block_storage.html#id1)cinder-backup``服务提供任何种类备份卷到一个备份存储提供者。就像``cinder-volume``服务，它与多种存储提供者在驱动架构下进行交互。

**消息列队**：在块存储的进程之间路由信息。

#### Openstack Cinder 对接 Vmware

**Cinder 配置文件**：/etc/cinder/cinder.conf

```shell
[DEFAULT]
# 在默认情况下使用的 volume type
default_volume_type = lvmdriver-1
# 应用多种 backends
enabled_backends = lvmdriver-1,vmware

# 配置 vmware 存储
[vmware]
volume_driver = cinder.volume.drivers.vmware.vmdk.VMwareVcVmdkDriver
vmware_host_password = Yangtao123.
vmware_host_username = administrator@os.com
vmware_host_ip = 172.28.9.181
vmware_insecure = True
# backend 名称
volume_backend_name = vmware

# 配置 lvm 存储
[lvmdriver-1]
image_volume_cache_enabled = True
volume_clear = zero
lvm_type = auto
iscsi_helper = lioadm
volume_group = stack-volumes-lvmdriver-1
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name = lvmdriver-1
```

**Vcenter 给 datastore 添加标签**：

1. 在 home 中 找到 Tags, 添加 标签 和 category（任意名称）

   ![](https://ws1.sinaimg.cn/large/006tKfTcly1fmvacwydo8j31kw0t6taz.jpg)

2. 将 tag 和 datastore 做关联

   ![](https://ws1.sinaimg.cn/large/006tKfTcly1fmvallxfs0j319q0ugwh2.jpg)

3. 回到主页 home，选择 Policies and Profiles。添加 VM Storage Policies，将 tag 和 策略 policy 坐关联，添加 tag based policy

   ![](https://ws2.sinaimg.cn/large/006tKfTcly1fmvapa0tmsj30mm0l83zv.jpg)

   ​

   ![](https://ws3.sinaimg.cn/large/006tKfTcly1fmvasxa8ovj31kw0yrtcq.jpg)

   ![](https://ws3.sinaimg.cn/large/006tKfTcly1fmvau53xtaj31kw0uwjvv.jpg)

**Cinder Volume 创建**：

```shell
# 创建 volume type
openstack volume type create vmware

# 给 volume type 打 tag，注意这里 storage_profile 既是在 vmware 中设置的策略名称 Policies and Profile
# ！！！！ 根据官方文档，可以加上 thick。但是实际测试中，需要把 property vmware:vmdk_type=thick 去掉，否则会报错。
# openstack volume type set --property vmware:vmdk_type=thick --property vmware:storage_profile=cinder-storage-policy THICK_VOLUME
openstack volume type set --property vmware:storage_profile=cinder-storage-policy VOLUME

# 逻辑上创建一个 vmware 的 volume
openstack volume create --size 1 --type vmware
```

> ”另外，M版对接6.5，创建云硬盘，不会出现volume开头的虚机，存储里也看不到volume-id文件夹，只有挂载后才会出现，注意！！“

---
title: Introduction of Nova Placment
categories:
- Nova
tags:
- nova
- nova-placement
---
[Openstack Placement](https://docs.openstack.org/nova/latest/user/placement.html) 是用来统计 Openstack 中 hypervisor 上不同资源总量，使用量，剩余量，和资源类型的服务。Openstack Placemenet 服务以 wsgi REST API 的方式暴露给用户，默认端口是 8778。

## Openstack Placement 中一些基本术语简单解释：

**resource class**: 表示不同的资源如 CPU， memory，PCI 等等，在数据库中以一个固定的数字对应。

相关数据模型：

```shell
MariaDB [nova_api]> desc resource_classes;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| name       | varchar(255) | NO   | UNI | NULL    |                |
| created_at | datetime     | YES  |     | NULL    |                |
| updated_at | datetime     | YES  |     | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

**resource provider**：表示一个资源（CPU/memory/PCI，volume，IP  Pool 等）的提供单位。

相关数据模型：

```shell
MariaDB [nova_api]> desc resource_providers;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| created_at | datetime     | YES  |     | NULL    |                |
| updated_at | datetime     | YES  |     | NULL    |                |
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| uuid       | varchar(36)  | NO   | UNI | NULL    |                |
| name       | varchar(200) | YES  | UNI | NULL    |                |
| generation | int(11)      | YES  |     | NULL    |                |
| can_host   | int(11)      | YES  |     | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

**inventories**：表示现有不同资源的总量，这些资源可以来自不同的 resource provider。

相关数据模型：

```shell
MariaDB [nova_api]> desc inventories;
+----------------------+----------+------+-----+---------+----------------+
| Field                | Type     | Null | Key | Default | Extra          |
+----------------------+----------+------+-----+---------+----------------+
| created_at           | datetime | YES  |     | NULL    |                |
| updated_at           | datetime | YES  |     | NULL    |                |
| id                   | int(11)  | NO   | PRI | NULL    | auto_increment |
| resource_provider_id | int(11)  | NO   | MUL | NULL    |                |
| resource_class_id    | int(11)  | NO   | MUL | NULL    |                |
| total                | int(11)  | NO   |     | NULL    |                |
| reserved             | int(11)  | NO   |     | NULL    |                |
| min_unit             | int(11)  | NO   |     | NULL    |                |
| max_unit             | int(11)  | NO   |     | NULL    |                |
| step_size            | int(11)  | NO   |     | NULL    |                |
| allocation_ratio     | float    | NO   |     | NULL    |                |
+----------------------+----------+------+-----+---------+----------------+
```

**resource alloactions**：被分配走的资源。

相关数据模型：

```shell
MariaDB [nova_api]> desc allocations;
+----------------------+-------------+------+-----+---------+----------------+
| Field                | Type        | Null | Key | Default | Extra          |
+----------------------+-------------+------+-----+---------+----------------+
| created_at           | datetime    | YES  |     | NULL    |                |
| updated_at           | datetime    | YES  |     | NULL    |                |
| id                   | int(11)     | NO   | PRI | NULL    | auto_increment |
| resource_provider_id | int(11)     | NO   | MUL | NULL    |                |
| consumer_id          | varchar(36) | NO   | MUL | NULL    |                |
| resource_class_id    | int(11)     | NO   | MUL | NULL    |                |
| used                 | int(11)     | NO   |     | NULL    |                |
+----------------------+-------------+------+-----+---------+----------------+
```

**resource provider aggregates**：和 nova aggreate 相关的 resource provider。

相关数据模型：

```shell
MariaDB [nova_api]> desc resource_provider_aggregates;
+----------------------+----------+------+-----+---------+-------+
| Field                | Type     | Null | Key | Default | Extra |
+----------------------+----------+------+-----+---------+-------+
| created_at           | datetime | YES  |     | NULL    |       |
| updated_at           | datetime | YES  |     | NULL    |       |
| resource_provider_id | int(11)  | NO   | PRI | NULL    |       |
| aggregate_id         | int(11)  | NO   | PRI | NULL    |       |
+----------------------+----------+------+-----+---------+-------+

MariaDB [nova_api]> desc aggregates;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| created_at | datetime     | YES  |     | NULL    |                |
| updated_at | datetime     | YES  |     | NULL    |                |
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| uuid       | varchar(36)  | YES  | MUL | NULL    |                |
| name       | varchar(255) | YES  | UNI | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

**compute nodes**： 控制 hypervisor 的 compute node，nova 调用 compute node 创建虚拟机。

相关数据模型（在 nova 数据库中，而不是 nova_api）：

```
MariaDB [nova]> desc compute_nodes;
+-----------------------+--------------+------+-----+---------+----------------+
| Field                 | Type         | Null | Key | Default | Extra          |
+-----------------------+--------------+------+-----+---------+----------------+
| created_at            | datetime     | YES  |     | NULL    |                |
| updated_at            | datetime     | YES  |     | NULL    |                |
| deleted_at            | datetime     | YES  |     | NULL    |                |
| id                    | int(11)      | NO   | PRI | NULL    | auto_increment |
| service_id            | int(11)      | YES  |     | NULL    |                |
| vcpus                 | int(11)      | NO   |     | NULL    |                |
| memory_mb             | int(11)      | NO   |     | NULL    |                |
| local_gb              | int(11)      | NO   |     | NULL    |                |
| vcpus_used            | int(11)      | NO   |     | NULL    |                |
| memory_mb_used        | int(11)      | NO   |     | NULL    |                |
| local_gb_used         | int(11)      | NO   |     | NULL    |                |
| hypervisor_type       | mediumtext   | NO   |     | NULL    |                |
| hypervisor_version    | int(11)      | NO   |     | NULL    |                |
| cpu_info              | mediumtext   | NO   |     | NULL    |                |
| disk_available_least  | int(11)      | YES  |     | NULL    |                |
| free_ram_mb           | int(11)      | YES  |     | NULL    |                |
| free_disk_gb          | int(11)      | YES  |     | NULL    |                |
| current_workload      | int(11)      | YES  |     | NULL    |                |
| running_vms           | int(11)      | YES  |     | NULL    |                |
| hypervisor_hostname   | varchar(255) | YES  |     | NULL    |                |
| deleted               | int(11)      | YES  |     | NULL    |                |
| host_ip               | varchar(39)  | YES  |     | NULL    |                |
| supported_instances   | text         | YES  |     | NULL    |                |
| pci_stats             | text         | YES  |     | NULL    |                |
| metrics               | text         | YES  |     | NULL    |                |
| extra_resources       | text         | YES  |     | NULL    |                |
| stats                 | text         | YES  |     | NULL    |                |
| numa_topology         | text         | YES  |     | NULL    |                |
| host                  | varchar(255) | YES  | MUL | NULL    |                |
| ram_allocation_ratio  | float        | YES  |     | NULL    |                |
| cpu_allocation_ratio  | float        | YES  |     | NULL    |                |
| uuid                  | varchar(36)  | YES  | UNI | NULL    |                |
| disk_allocation_ratio | float        | YES  |     | NULL    |                |
| mapped                | int(11)      | YES  |     | NULL    |                |
+-----------------------+--------------+------+-----+---------+----------------+
```

 

跟详细的解释可以参考 [Placement API References](https://docs.openstack.org/nova/latest/user/placement.html#references)

## Openstack Placement 服务解决问题： 

### 为什么会有 placemenet 服务：

#### Nova 存在的问题：

在 Openstack 资源可以分成两类：

* 每个 compute node 上的资源，如  CPU, memory, PCI devices and local ephemeral disk
* 多个 compute node 共享资源，如共享存储 Ceph 和 NFS 

Nova 在统计这些资源的时候会直接将所有 compute 节点上的资源总和一起统计。这样会导致共享资源会被重复统计。

举个例子：

环境：有 20 个 compute 节点（每个节点上一个 KVM hypervisor），公用一个 ceph pool vms，每个 pool 会有 3 个 replication。

```shell
[root@172e18e211e39 ~]# ceph osd pool ls
rbd
images
volumes
vms
backups
[root@172e18e211e39 ~]# ceph osd pool get vms size
size: 3
[root@172e18e211e39 ~]# ceph df
GLOBAL:
    SIZE     AVAIL     RAW USED     %RAW USED
    218T      214T        3661G          1.64
POOLS:
    NAME        ID     USED      %USED     MAX AVAIL     OBJECTS
    rbd         0          0         0        73043G           0
    images      1       833G      1.13        73043G      213401
    volumes     2       380G      0.52        73043G       99249
    vms         3          0         0        73043G           2
    backups     4      4491M         0        73043G        1158
```

找到一个 hypervisor 查看状态，看见可用 `local_gb` = ceph 中 vms MAX AVAIL /  replication 数量 223452。说明一个节点上可用的 volume 资源是 223452 GB。这里并没有看出什么问题。

```shell
[root@172e18e211e99 ~]# openstack hypervisor show 172e18e211e22
-------------------------------------------------------------------------------------+
| Field                | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
+----------------------+------------------------------------------------------------------
| aggregates           | [u'd_test']                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| cpu_info             | {"vendor": "Intel", "model": "IvyBridge", "arch": "x86_64", "features": ["pge", "avx", "vmx", "clflush", "sep", "syscall", "vme", "dtes64", "tsc", "sse", "xsave", "xsaveopt", "erms", "xtpr", "cmov", "smep", "ssse3", "est", "pat", "monitor", "smx", "pcid", "lm", "msr", "nx", "fxsr", "tm", "sse4.1", "pae", "sse4.2", "pclmuldq", "acpi", "tsc-deadline", "popcnt", "mmx", "osxsave", "cx8", "mce", "de", "tm2", "ht", "dca", "pni", "pdcm", "mca", "pdpe1gb", "apic", "fsgsbase", "f16c", "pse", "ds", "invtsc", "lahf_lm", "aes", "sse2", "ss", "ds_cpl", "arat", "pbe", "fpu", "cx16", "pse36", "mtrr", "rdrand", "rdtscp", "x2apic"], "topology": {"cores": 8, "cells": 2, "threads": 2, "sockets": 1}} |
| current_workload     | 0                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| disk_available_least | 219795                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| free_disk_gb         | 223352                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| free_ram_mb          | 20451                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| host_ip              | 172.18.211.22                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| host_time            | 13:10:42                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| hypervisor_hostname  | 172e18e211e22                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| hypervisor_type      | QEMU                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| hypervisor_version   | 2006000                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| id                   | 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| load_average         | 20.02, 20.06, 20.07                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| local_gb             | 223452                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| local_gb_used        | 100                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| memory_mb            | 131043                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| memory_mb_used       | 110592                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| running_vms          | 5                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| service_host         | 172e18e211e22                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| service_id           | 97                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| state                | up                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| status               | enabled                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| uptime               | 3 days,  4:00                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| users                | 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| vcpus                | 28                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| vcpus_used           | 24                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
+----------------------+------------------------------------------------------------------
```

但是，当统计 20 个 compute 节点上的可用 volume 资源量时，会发现。volume 资源总量 = compute 节点数 * 每个节点上的 volume 资源。这样资源被扩大了 20 倍，local_gb 4469040。

```
[root@172e18e211e99 ~]# openstack hypervisor stats show
+----------------------+---------+
| Field                | Value   |
+----------------------+---------+
| count                | 20      |
| current_workload     | 0       |
| disk_available_least | 4395900 |
| free_disk_gb         | 4467030 |
| free_ram_mb          | 313724  |
| local_gb             | 4469040 |
| local_gb_used        | 16234   |
| memory_mb            | 2620860 |
| memory_mb_used       | 1880797 |
| running_vms          | 197     |
| vcpus                | 560     |
| vcpus_used           | 565     |
+----------------------+---------+
```

为了解决类似上面例子中出现的问题。Placement 将被加入Nova 组件。

#### 如何解决 Nova 共享资源的问题

Openstack 项目使用 placement 来规划资源，记录统计资源。这里用 Openstack 官方的一个 [Scenario](https://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/generic-resource-pools.html#scenario-1-shared-disk-storage-used-for-vm-disk-images) 例子解释。

环境：有一堆 compute node 使用 100TB 的 NFS 共享存储，其中有 NFS 中有 1TB 被拥有于 Openstack 之外的存储。

1. 假设第 1 行机架上的 6 到 10 号机器需要使用共享存储，那么先将这 5 台机器放到同一个 aggreate 中。

   ```shell
   AGG_UUID=`openstack aggregate create r1rck0610`
   # for all compute nodes in the system that are in racks 6-10 in row 1...
   openstack aggregate add host $AGG_UUID $HOSTNAME
   ```

2. 创建一个 NFS 存储的 resource provider，同时和 aggreate 做关联。（如果是在 compute local 资源的情况下 resource provider 时和某个 compute_node uuid 做关联）

   ```shell
   RP_UUID=`openstack resource-provider create "/mnt/nfs/row1racks0610/" \
       --aggregate-uuid=$AGG_UUID`
   ```

3. 云部署人员配置 resource provider 的资源量。

   ```shell
   openstack resource-provider set inventory $RP_UUID \
       --resource-class=DISK_GB \
       --total=100000 --reserved=1000 \
       --min-unit=50 --max-unit=10000 --step-size=10 \
       --allocation-ratio=1.0
       
   # --reserved 表示被用于其他存储的 NFS
   # 当有创建虚拟机的 request 时，那么存储资源检查公式是：((TOTAL - RESERVED) * ALLOCATION_RATIO) - USED >= REQUESTED_AMOUNT 
   ```

依照上面方法，创建 vm 时 vloume 资源已经被人工规划好，而不是由 compute node 主动获取，而导致共享资源被放大 compute node 个数倍。

## Placement API

在 Openstack 中 [Microversions](https://developer.openstack.org/api-guide/compute/microversions.html) 的概念。使用 Microversions，可以基于每个 request 的方式来触发 service 新的功能。在 Placement 服务中出发新的功能可以用 header。

```shell
OpenStack-API-Version: placement 1.10
```

```shell
# request
curl -g -i -X GET http://172.28.8.202:8778/placement -H "OpenStack-API-Version: placement 1.10" -H "Connection: keep-alive" -H "User-Agent: nova-schedulerkeystoneauth1/3.1.0python-requests/2.11.1CPython/2.7.5" -H "X-Auth-Token: gAAAAABauinIhVegGV_Pr9lCIh0KRZ7MIQ7dM9o6YsULt_cTn4zPWaNF_iqafCn_CeEUi9HXTdQKfyp4soRtgUrASBsbqeWakzb6x2XI9VO2jFlcjLvlcPcq1aIJ4zTImf7LSXaaiYD8H1V_NyZ7ZI3_704Emjnkc1uHC_wBRwLJXf3osdda4Sk"

# response
HTTP/1.1 200 OK
Date: Tue, 27 Mar 2018 11:26:04 GMT
Server: Apache/2.4.6 (CentOS)
OpenStack-API-Version: placement 1.10
vary: OpenStack-API-Version,Accept-Encoding
x-openstack-request-id: req-9d6b4b2c-39e0-49ef-8cfc-7e6429200a9d
Content-Length: 75
Keep-Alive: timeout=15, max=100
Connection: Keep-Alive
Content-Type: application/json

{"versions": [{"min_version": "1.0", "max_version": "1.10", "id": "v1.0"}]}
```

```shell
# request
curl -g -i -X GET http://172.28.8.202:8778/placement/resource_providers -H "OpenStack-API-Version: placement 1.10" -H "Connection: keep-alive" -H "User-Agent: nova-schedulerkeystoneauth1/3.1.0python-requests/2.11.1CPython/2.7.5" -H "X-Auth-Token: gAAAAABauinIhVegGV_Pr9lCIh0KRZ7MIQ7dM9o6YsULt_cTn4zPWaNF_iqafCn_CeEUi9HXTdQKfyp4soRtgUrASBsbqeWakzb6x2XI9VO2jFlcjLvlcPcq1aIJ4zTImf7LSXaaiYD8H1V_NyZ7ZI3_704Emjnkc1uHC_wBRwLJXf3osdda4Sk"

# response
HTTP/1.1 200 OK
Date: Tue, 27 Mar 2018 11:32:03 GMT
Server: Apache/2.4.6 (CentOS)
OpenStack-API-Version: placement 1.10
vary: OpenStack-API-Version,Accept-Encoding
x-openstack-request-id: req-acbd106d-9355-4c01-9f87-5b6e7ee14de4
Content-Length: 691
Keep-Alive: timeout=15, max=100
Connection: Keep-Alive
Content-Type: application/json

{
    "resource_providers": [
        {
            "generation": 458,
            "uuid": "95045799-12ca-4e62-8397-b23f0d71ad18",
            "links": [
                {
                    "href": "/placement/resource_providers/95045799-12ca-4e62-8397-b23f0d71ad18",
                    "rel": "self"
                },
                {
                    "href": "/placement/resource_providers/95045799-12ca-4e62-8397-b23f0d71ad18/inventories",
                    "rel": "inventories"
                },
                {
                    "href": "/placement/resource_providers/95045799-12ca-4e62-8397-b23f0d71ad18/usages",
                    "rel": "usages"
                },
                {
                    "href": "/placement/resource_providers/95045799-12ca-4e62-8397-b23f0d71ad18/aggregates",
                    "rel": "aggregates"
                },
                {
                    "href": "/placement/resource_providers/95045799-12ca-4e62-8397-b23f0d71ad18/traits",
                    "rel": "traits"
                }
            ],
            "name": "domain-c7.b52faa6d-71d0-4a31-bf8e-0fab03c5ca7f"
        }
    ]
}
```



Placement 提供了以下这些 API 操作接口：

- API Versions
- Rerource Provider
- Resource Providers
- Resource Class
- Resource Classes
- Resource Provider Inventory
- Resource Provider Inventories
- Resource Provider Aggregates
- Traits
- Resource Provider Traits
- Allocations
- Resource Provider Alllocations
- Usages
- Resource Provider Usages
- Allocation Candidates

如果前端需要调用 placement 显示资源可以参考 [Placement API Doc](https://developer.openstack.org/api-ref/placement/#resource-classes) 。

## Placement Client

Placement client 也提供了一个 python client，可以通过 `pip install osc-placement` 安装。

安装后可以使用 `openstack` 命令直接和 Placement service 交互：

```shell
[root@yjfnode4 ~]# openstack help resource
Command "resource" matches:
  resource class create
  resource class delete
  resource class list
  resource class show
  resource provider aggregate list
  resource provider aggregate set
  resource provider allocation delete
  resource provider allocation set
  resource provider allocation show
  resource provider create
  resource provider delete
  resource provider inventory class set
  resource provider inventory delete
  resource provider inventory list
  resource provider inventory set
  resource provider inventory show
  resource provider list
  resource provider set
  resource provider show
  resource provider usage show
  
  [root@yjfnode4 ~]# openstack resource provider list
+--------------------------------------+------------------------------------------------+------------+
| uuid                                 | name                                           | generation |
+--------------------------------------+------------------------------------------------+------------+
| 95045799-12ca-4e62-8397-b23f0d71ad18 | domain-c7.b52faa6d-71d0-4a31-bf8e-0fab03c5ca7f |        458 |
+--------------------------------------+------------------------------------------------+------------+
```

更多命令可以参考：[osc-placement](https://docs.openstack.org/osc-placement/latest/readme.html) , [osc-placement’s documentation](https://docs.openstack.org/osc-placement/latest/index.html) 

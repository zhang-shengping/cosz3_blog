---
title: Openstack Nova update_avaiable_resource
categories:
- Nova
tags:
- code
- nova
- self-note
---


## update_available_resource 的作用

`update_available_resource` Nova compute **定时**上报资源的方法，他的主要上报对象是 Nova placement 服务，同时也会直接更新数据库中的 `compute node table`。那么 `update_availalbe_resource` 上报了哪些资源？上报的资源大体可分为两类：1 compute service node 的资源，2. hypervisor node（host）资源。

#### 概念

* compute service node 资源：Nova compute service 可以类比 VMware 的 vCenter，他可以直接管理一个 hypervisor 或者多个 hypervisor。在一个 compute service node 下的资源则是当前 node 下管理的所有 hypervisor 资源的总和。
* hyperviosr node 资源：是针对单个 hypervisor 来说的资源（如 KVM 中的一个 hypervisor），Openstack 将 vCenter 上的一个 cluster 资源看成一个 hypervisor node 的资源。每个 compute service node 通过计算每个 Hypervisor 节点上的资源量来统计当前compute service node 上剩余的资源量。
* host：表识一个 compute service node，host name 可以通过 nova.conf 配置文件更改，如果没有更改则是和 compute service node 所在主机名称一样。（多个 compute service node 可以配置一样的 hostname）
* hypervisor hostname：标识一个 hypervisor node，在 Openstack 中 vCenter 的 hypervisor node hostname 则视为一个 cluster 的 domain name，在 KVM 中通过 libert driver 的 getInfo 方法获得。（多个 hypervisor 不可以被配置一个的 hypervisor name，会导致统计的 hypervisor 资源统总量小于真实的 hypervisor 资源的总量）
* host 和 hypervisor 总是要做关联。没有 host 的 hypervisor 是不存在的。

## update_available_resource 代码分析

Nova code 里面会有两个 update_available_resource 方法，一个是在 compute/manager.py 和 compute/resource_tracker.py 文件中，前者会调用后者。

`update_available_resource`在 Nova compute pre_start_hook 中第一次触发，这次出发是主动调用。在 Nova compute 启动之前上报一次资源状态。

```python
# 在 nova/compute/manager.py 文件中
def pre_start_hook(self):
    """After the service is initialized, but before we fully bring
    the service up by listening on RPC queues, make sure to update
    our available resources (and indirectly our available nodes).
    """
    # startup 为 True 时表示第一次上报，log 会提示 ignore warnings 信息
    self.update_available_resource(nova.context.get_admin_context(),
                                   startup=True)
```

### update_avaiable_resource 时如何定义的，其中大体有四部：

* `self._get_compute_nodes_in_db` : 用 compute service node 的 host 寻找 nova db compute_nodes 中对应的 compute node。
* `self.driver.get_available_nodes`：使用 virt driver 去找到当前 compute service node 下管理的 hypervisor node。
* `self.update_available_resource_for_node`：将 compute service node host 和 hypervisor node hostname 在 compute_nodes 表中做关联，同时update placement 服务。 **注意这个方法是针对每一个 hypervisor node，而不是 compute node**
* `self.scheduler_client.reportclient.delete_resource_provider`: 把 compute service node 和 hypervisor node 关联有错误的 compute service node 从 compute_nodes 表中清理掉，同时也清理掉这些 hypervisor 在 placement 服务中的记录。

```python
    # 在 nova/compute/manager.py 文件中
    @periodic_task.periodic_task(spacing=CONF.update_resources_interval)
    def update_available_resource(self, context, startup=False):
        """See driver.get_available_resource()

        Periodic process that keeps that the compute host's understanding of
        resource availability and usage in sync with the underlying hypervisor.

        :param context: security context
        :param startup: True if this is being called when the nova-compute
            service is starting, False otherwise.
        """
        # 这里的 compute_nodes 可以理解为 compute service node
        # 通过 self.host （配置文件中可以配置）获取在 nova compute node table
        # 中的获取 compute node 信息。
        compute_nodes_in_db = self._get_compute_nodes_in_db(context,
                                                            use_slave=True,
                                                            startup=startup)
        try:
            # 这里的 node 指的是被 compute service node manage 的 hypervisor
            # node （hypervisor hostname）
            nodenames = set(self.driver.get_available_nodes())
        except exception.VirtDriverNotReady:
            LOG.warning("Virt driver is not ready.")
            return
        # 这里去更新 compute_node table 和 resource provider table
        # 这里的 nodename 是 compute service  manage 的 hypervisor
        # 在这里会去创建 resource provider，resource inventory，resource class，resource
        # provider 等。
        # 注意这个方法是针对每一个 hypervisor node，而不是 compute node
        for nodename in nodenames:
            self.update_available_resource_for_node(context, nodename)

        # 这里的意思是找到了多个 compute nodes，但是其中有些 compute nodes 并不是
        # 管理当前 driver 控制的 hypervisor。说明 compute nodes 和已经不存在的 hypervisor node
        # 做了关联，这个是没有用的。就算找到这个 compute nodes 去创建虚拟机，hypervisor node 不存在
        # 是创建不了虚拟机的。
        # Delete orphan compute node not reported by driver but still in db
        for cn in compute_nodes_in_db:
            if cn.hypervisor_hostname not in nodenames:
                LOG.info("Deleting orphan compute node %(id)s "
                         "hypervisor host is %(hh)s, "
                         "nodes are %(nodes)s",
                         {'id': cn.id, 'hh': cn.hypervisor_hostname,
                          'nodes': nodenames})
                cn.destroy()
                # Delete the corresponding resource provider in placement,
                # along with any associated allocations and inventory.
                # TODO(cdent): Move use of reportclient into resource tracker.
                self.scheduler_client.reportclient.delete_resource_provider(
                    context, cn, cascade=True)
```

### `self.update_available_resource_for_node` 的定义

`self.update_available_resource_for_node`: 这个方法是 `update_available_resource` 更新 resource 的主题所在。**注意这个方法是针对每一个 hypervisor node，而不是 compute node**

从代码来看，这里使用到了 resource tracker 对象（**注意在创建resource tracker 对象时，compute manager 将自己的self.host 传给了 resource tracker 对象的 self.host**）。直接去调用resource tracker 的 `update_available_resource`。 nodename 参数也是从 drvier 那里得到的 hypervisor hostname set 集合。


```python
    def update_available_resource_for_node(self, context, nodename):
        # 这个compute service node 将他的 host 传入 rt
        rt = self._get_resource_tracker()
        try:
            rt.update_available_resource(context, nodename)
        except exception.ComputeHostNotFound:
            # NOTE(comstud): We can get to this case if a node was
            # marked 'deleted' in the DB and then re-added with a
            # different auto-increment id. The cached resource
            # tracker tried to update a deleted record and failed.
            # Don't add this resource tracker to the new dict, so
            # that this will resolve itself on the next run.
            LOG.info("Compute node '%s' not found in "
                     "update_available_resource.", nodename)
            # TODO(jaypipes): Yes, this is inefficient to throw away all of the
            # compute nodes to force a rebuild, but this is only temporary
            # until Ironic baremetal node resource providers are tracked
            # properly in the report client and this is a tiny edge case
            # anyway.
            self._resource_tracker = None
            return
        except Exception:
            LOG.exception("Error updating resources for node %(node)s.",
                          {'node': nodename})
```

### resource tracker 中的`update_available_resource`

```python
    def update_available_resource(self, context, nodename):
        """Override in-memory calculations of compute node resource usage based
        on data audited from the hypervisor layer.

        Add in resource claims in progress to account for operations that have
        declared a need for resources, but not necessarily retrieved them from
        the hypervisor layer yet.

        :param nodename: Temporary parameter representing the Ironic resource
                         node. This parameter will be removed once Ironic
                         baremetal resource nodes are handled like any other
                         resource in the system.
        """
        LOG.debug("Auditing locally available compute resources for "
                  "%(host)s (node: %(node)s)",
                 {'node': nodename,
                  'host': self.host})
        # 这里用每一个 nodename， 通过driver 来获取 每个 hypervisor node 上的resource
        resources = self.driver.get_available_resource(nodename)
        # NOTE(jaypipes): The resources['hypervisor_hostname'] field now
        # contains a non-None value, even for non-Ironic nova-compute hosts. It
        # is this value that will be populated in the compute_nodes table.
        # 将对应 resource 和 管理这个 resource的 compute service 节点的 IP 做关联
        resources['host_ip'] = CONF.my_ip

        # We want the 'cpu_info' to be None from the POV of the
        # virt driver, but the DB requires it to be non-null so
        # just force it to empty string
        if "cpu_info" not in resources or resources["cpu_info"] is None:
            resources["cpu_info"] = ''

        self._verify_resources(resources)

        self._report_hypervisor_resource_view(resources)
        # available resource,
        self._update_available_resource(context, resources)
```

resource tracker 中的`update_available_resource`主要用 hypervisor node hostname 来统计 hypervisor node 上的资源量。

* `self.driver.get_available_resource`：通过使用 nodename （hypervisor node hostname） 获取 hypervisor node 上 resource 总量
* `resources['host_ip'] = CONF.my_ip`：这一步直接将 hyperviosr node 和 compute service node **通过 IP 作起了关联**。
* `_update_available_resource`：通过以上步骤的到了一个没有经过任何处理和记录的 raw resource，此方法用来记录和细化处理得到的 raw resource。

### `_update_available_resource` 解析

这片文章只是在与关心 resource update 的流程，所以对 resource 的计算部分暂时不提。

在这里和 resource update 有关的函数：

* `self._init_compute_node`: 这里主要是 compute node，将 compute node 以 hypervisor hostname ：compute node 方式缓存在 self.compute_nodes 字典中。同时会初始化下 placement 服务。**其实看 `self._init_compute_node`代码可以发现也会去调用一次 `self._update`方法，这个方法在 `self._init_compute_node`存粹是为了初始化 placement 服务**。
* `self._update`：此处的 update 和 init compute node 中的 update 是完全不同的，此处的 update 是 compute node 计算过使用量后，针对 placement 服务的 update，而不是去初始化 placement。

```python
    # 这里第一次创建 compute node 在 compute_node数据库中
    @utils.synchronized(COMPUTE_RESOURCE_SEMAPHORE)
    def _update_available_resource(self, context, resources):

        # initialize the compute node object, creating it
        # if it does not already exist.
        self._init_compute_node(context, resources)

        nodename = resources['hypervisor_hostname']

        # if we could not init the compute node the tracker will be
        # disabled and we should quit now
        if self.disabled(nodename):
            return

        # Grab all instances assigned to this node:
        instances = objects.InstanceList.get_by_host_and_node(
            context, self.host, nodename,
            expected_attrs=['system_metadata',
                            'numa_topology',
                            'flavor', 'migration_context'])

        # Now calculate usage based on instance utilization:
        self._update_usage_from_instances(context, instances, nodename)

        # Grab all in-progress migrations:
        migrations = objects.MigrationList.get_in_progress_by_host_and_node(
                context, self.host, nodename)

        self._pair_instances_to_migrations(migrations, instances)
        self._update_usage_from_migrations(context, migrations, nodename)

        self._remove_deleted_instances_allocations(
            context, self.compute_nodes[nodename], migrations)

        # Detect and account for orphaned instances that may exist on the
        # hypervisor, but are not in the DB:
        orphans = self._find_orphaned_instances()
        self._update_usage_from_orphans(orphans, nodename)

        cn = self.compute_nodes[nodename]

        # NOTE(yjiang5): Because pci device tracker status is not cleared in
        # this periodic task, and also because the resource tracker is not
        # notified when instances are deleted, we need remove all usages
        # from deleted instances.
        self.pci_tracker.clean_usage(instances, migrations, orphans)
        dev_pools_obj = self.pci_tracker.stats.to_device_pools_obj()
        cn.pci_device_pools = dev_pools_obj

        self._report_final_resource_view(nodename)

        metrics = self._get_host_metrics(context, nodename)
        # TODO(pmurray): metrics should not be a json string in ComputeNode,
        # but it is. This should be changed in ComputeNode
        cn.metrics = jsonutils.dumps(metrics)

        # update the compute_node
        self._update(context, cn)
        LOG.debug('Compute_service record updated for %(host)s:%(node)s',
                  {'host': self.host, 'node': nodename})
```

### `_init_compute_node`

* 从 resource 中可以获得 hypervisor_hostname，赋值给 nodename
* 用 nodename 去获取  self.compute_nodes 中缓存的 compute node 信息，如果获取得到，给 compute node 和 placement 做更新。
* 用 `_get_compute_node` 方法去数据库 compute node table  中（但是数据库不在本地）获取 compute node。**注意这里使用的索引是 host 和 nodename，即是 compute service node name 和 hypervisor host node name（这就说明如果多个 computer service host 都有相同的 host name，同时管理的 hypervisor node name 也相同是，那么 compute_nodes 表中只会对一条记录不断更新）**。如果获取得到，给 compute node 和 placement 做更新。
* 如果缓存和数据库中都获取不到那么只有在 compute node table 中重新创建 compute node，最后更新 placemnet 服务。

**这里为什么每次 placement 都要做更新，个人觉得只要第二次和第三次做更新即可，因为第一次在缓存中找到的话，说明 compute service node 没有被重启过，resource 应该没什么变化。但是如果一个 compute 管理多个 hypervisor 的话，这里更新的东西可能是别的 会不会和 *pci_tracker* 有关？？？**

```python
    def _init_compute_node(self, context, resources):
        """Initialize the compute node if it does not already exist.

        The resource tracker will be inoperable if compute_node
        is not defined. The compute_node will remain undefined if
        we fail to create it or if there is no associated service
        registered.

        If this method has to create a compute node it needs initial
        values - these come from resources.

        :param context: security context
        :param resources: initial values
        """
        nodename = resources['hypervisor_hostname']

        # if there is already a compute node just use resources
        # to initialize
        if nodename in self.compute_nodes:
            cn = self.compute_nodes[nodename]
            # 同时也会把 host_ip 给更改掉
            self._copy_resources(cn, resources)
            self._setup_pci_tracker(context, cn, resources)
            self._update(context, cn)
            return
        # 关键是在这里，通过 hypervisor host name
        # 和 compute service host 节点信息，去 compute node 表中
        # 寻找 compute node。如果有多个 compute service host 连接一个
        # 同一个 hypervisor 那么他们的 hypervisor host name 必然是相同的
        # 同时如果多个 compute service node host name 也是相同的，
        # 那么找到的是同一个 在 compute node 表中的记录。

        # 依次类推，那resource provider 记录的也是一个 hostname
        # 和 hypervisor name 相同的一个 compute service。这样就算是挂掉一个
        # compute service node，在下次上报时会被没有挂掉的 compute service node
        # 刷新 IP。

        # shenping: 虽然是多个 compute service node，但是这样，在 compute node
        # table 和 resource  provider table 中只有一条记录。
        # now try to get the compute node record from the
        # database. If we get one we use resources to initialize
        cn = self._get_compute_node(context, nodename)
        if cn:
            self.compute_nodes[nodename] = cn
            # 同时也会把 host_ip 给更改掉
            self._copy_resources(cn, resources)
            self._setup_pci_tracker(context, cn, resources)
            self._update(context, cn)
            return

        if self._check_for_nodes_rebalance(context, resources, nodename):
            return

        # 如果没有找到，则会去在 compute_node table 中创建这个 object
        # there was no local copy and none in the database
        # so we need to create a new compute node. This needs
        # to be initialized with resource values.
        cn = objects.ComputeNode(context)
        cn.host = self.host
        # 同时也会把 host_ip 给更改掉
        self._copy_resources(cn, resources)
        self.compute_nodes[nodename] = cn
        cn.create()
        LOG.info('Compute node record created for '
                 '%(host)s:%(node)s with uuid: %(uuid)s',
                 {'host': self.host, 'node': nodename, 'uuid': cn.uuid})

        self._setup_pci_tracker(context, cn, resources)
        # 使用 compute node 的信息去更新
        # resource provider inventory table（一个 hypervisor 总资源 table）
        self._update(context, cn)
```

### `self._update`

`self._update`方法其实是调用了 scheduler client，用 scheduler report 去上报 placement 服务。

**注意这里将 compute_node.uuid 传入了 set_inventory_for_provider，而 compute_node.uuid 是来自 cn.create() 时，同时 compute_node.hypervisor_hostname 也被传入 set_inventory_for_provider, 这时 placement 中的 resource provider 信息和 compute node 通过这两个 id 做了联系。 **

```python
    def _update(self, context, compute_node):
        """Update partial stats locally and populate them to Scheduler."""
        # 比较 updated_at 时间，如果和 cache 的 old compute node updated_at 时间
        # 不一样的话就 compute_node.save()
        if self._resource_change(compute_node):
            # If the compute_node's resource changed, update to DB.
            # NOTE(jianghuaw): Once we completely move to use get_inventory()
            # for all resource provider's inv data. We can remove this check.
            # At the moment we still need this check and save compute_node.
            compute_node.save()

        # NOTE(jianghuaw): Some resources(e.g. VGPU) are not saved in the
        # object of compute_node; instead the inventory data for these
        # resource is reported by driver's get_inventory(). So even there
        # is no resource change for compute_node as above, we need proceed
        # to get inventory and use scheduler_client interfaces to update
        # inventory to placement. It's scheduler_client's responsibility to
        # ensure the update request to placement only happens when inventory
        # is changed.
        nodename = compute_node.hypervisor_hostname
        # Persist the stats to the Scheduler
        try:
            # 从 driver 获得 inventory
            inv_data = self.driver.get_inventory(nodename)
            _normalize_inventory_from_cn_obj(inv_data, compute_node)
            # 使用 scheduler client 上报信息，可以回去placement 服务创建
            # resource provider，resource inventory，resource class 等。
            # 这里传入了 compute_node.uuid， compute_node.hypervisor_hostname
            self.scheduler_client.set_inventory_for_provider(
                context,
                compute_node.uuid,
                compute_node.hypervisor_hostname,
                inv_data,
            )
        except NotImplementedError:
            # Eventually all virt drivers will return an inventory dict in the
            # format that the placement API expects and we'll be able to remove
            # this code branch
            self.scheduler_client.update_compute_node(context, compute_node)

        try:
            traits = self.driver.get_traits(nodename)
        except NotImplementedError:
            pass
        else:
            # NOTE(mgoddard): set_traits_for_provider does not refresh the
            # provider tree in the report client, so we rely on the above call
            # to set_inventory_for_provider or update_compute_node to ensure
            # that the resource provider exists in the tree and has had its
            # cached traits refreshed.
            self.reportclient.set_traits_for_provider(
                context, compute_node.uuid, traits)

        # 和热插拔有关
        if self.pci_tracker:
            self.pci_tracker.save(context)
```

### scheduler report 服务

```python
    def set_inventory_for_provider(self, context, rp_uuid, rp_name, inv_data,
                                   parent_provider_uuid=None):
        """Given the UUID of a provider, set the inventory records for the
        provider to the supplied dict of resources.

        :param context: The security context
        :param rp_uuid: UUID of the resource provider to set inventory for
        :param rp_name: Name of the resource provider in case we need to create
                        a record for it in the placement API
        :param inv_data: Dict, keyed by resource class name, of inventory data
                         to set against the provider
        :param parent_provider_uuid:
                If the provider is not a root, this is required, and represents
                the UUID of the immediate parent, which is a provider for which
                this method has already been invoked.

        :raises: exc.InvalidResourceClass if a supplied custom resource class
                 name does not meet the placement API's format requirements.
        """
        self._ensure_resource_provider(
            context, rp_uuid, rp_name,
            parent_provider_uuid=parent_provider_uuid)

        # Auto-create custom resource classes coming from a virt driver
        self._ensure_resource_classes(context, set(inv_data))

        # NOTE(efried): Do not use the DELETE API introduced in microversion
        # 1.5, even if the new inventory is empty.  It provides no way of
        # sending the generation down, so no way to trigger/detect a conflict
        # if an out-of-band update occurs between when we GET the latest and
        # when we invoke the DELETE.  See bug #1746374.
        self._update_inventory(context, rp_uuid, inv_data)
```

以下三个方法和 placement 服务相关，待续...
* `_ensure_resource_provider`
* `_ensure_resource_classes`
* `_update_inventory`

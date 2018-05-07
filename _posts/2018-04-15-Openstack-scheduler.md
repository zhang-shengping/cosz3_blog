---
title: Openstack select destinations (P 版)
categories:
- Nova
tags:
- nova
- code
- self-note
---

![useful image]({{ site.url }}/cosz3_blog/assets/images/blog_pictures/Filters+and+weight+hosts.jpg)


以下是关于一些 P 版 Openstack scheduler 筛选 hosts（compute）代码的分析记录。以后版本筛选 host 方法会有些变化，仅供参考记录。  

## nova scheduler manager

scheduler manager 主要包含了 Nova scheduler 服务的启动，初始化模块，和一些暴露给外面服务 （Nova conductor…）等模块。是对 Nova scheduler driver 的一个封装。

```python
class SchedulerManager(manager.Manager):
    """Chooses a host to run instances on."""

    target = messaging.Target(version='4.4')

    _sentinel = object()

    def __init__(self, scheduler_driver=None, *args, **kwargs):
        client = scheduler_client.SchedulerClient()
        self.placement_client = client.reportclient
        if not scheduler_driver:
            # 初始化方法中使用配置文件中的 CONF.scheduler.driver
            # 默认使用 nova/scheduler/filter_scheduler.py
            scheduler_driver = CONF.scheduler.driver
        self.driver = driver.DriverManager(
                "nova.scheduler.driver",
                scheduler_driver,
                invoke_on_load=True).driver
        super(SchedulerManager, self).__init__(service_name='scheduler',
                                               *args, **kwargs)
 
   #......
    # 暴露出来用于调度的方法
    @messaging.expected_exceptions(exception.NoValidHost)
    def select_destinations(self, ctxt,
                            request_spec=None, filter_properties=None,
                            spec_obj=_sentinel, instance_uuids=None):
        #.....
        # 使用到了 placement 服务做初步选取 provider
        if self.driver.USES_ALLOCATION_CANDIDATES:
            res = self.placement_client.get_allocation_candidates(resources)
        #......
        # 使用了 filter driver 本身的服务
        # import pdb; pdb.set_trace()
        dests = self.driver.select_destinations(ctxt, spec_obj, instance_uuids,
            alloc_reqs_by_rp_uuid, provider_summaries)
```

### 从 placement 服务中输出的参数

```python
([{
	u 'allocations': [{
		u 'resource_provider': {
			u 'uuid': u 'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5'
		},
		u 'resources': {
			u 'VCPU': 1,
			u 'MEMORY_MB': 1024
		}
	}]
}], {
	u 'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5': {
		u 'resources': {
			u 'VCPU': {
				u 'used': 13,
				u 'capacity': 1024
			},
			u 'MEMORY_MB': {
				u 'used': 23552,
				u 'capacity': 367914
			}
		}
	}
})

# alloc_reqs:
[{
	u 'allocations': [{
		u 'resource_provider': {
			u 'uuid': u 'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5'
		},
		u 'resources': {
			u 'VCPU': 1,
			u 'MEMORY_MB': 1024
		}
	}]
}]

# provider_summaries:
{
	u 'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5': {
		u 'resources': {
			u 'VCPU': {
				u 'used': 13,
				u 'capacity': 1024
			},
			u 'MEMORY_MB': {
				u 'used': 23552,
				u 'capacity': 367914
			}
		}
	}
}

# alloc_reqs_by_rp_uuid 把多个 resource allocations 以 uuid：[xxx]
defaultdict(<type 'list'>, {u'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5': [{u'allocations': [{u'resource_provider': {u'uuid': u'ccafbbcb-b44f-4c19-b71d-f098ec0f78c5'}, u'resources': {u'VCPU': 1, u'MEMORY_MB': 1024}}]}]})

```

## FilterScheduler (在 Scheduler 中维护了一个 HostManager)

### 寻找 host (使用 HostManager)

Scheduler manager 调用 FilterScheduler 中的 select_denstination。

```python
    # filter scheduler 中 select_denstination 方法主要部分
    def _schedule(self, context, spec_obj, instance_uuids,
            alloc_reqs_by_rp_uuid, provider_summaries):
```



```python
        # Find our local list of acceptable hosts by repeatedly
        # filtering and weighing our options. Each time we choose a
        # host, we virtually consume resources on it so subsequent
        # selections can adjust accordingly.

        # Note: remember, we are using an iterator here. So only
        # traverse this list once. This can bite you if the hosts
        # are being scanned in a filter or weighing function.
        
        # 1
        # 这个方法主要在于确认 通过 resource provider uuid 寻找到的 compute node
        # 有对应的 nova-compute service 启动。
        # 同时也维护了一个 self.host_state_map, 如下
        """  
                host = compute.host
                node = compute.hypervisor_hostname
                state_key = (host, node)
                host_state = self.host_state_map.get(state_key)
                if not host_state:
                    host_state = self.host_state_cls(host, node,
                                                     cell_uuid,
                                                     compute=compute)
                    self.host_state_map[state_key] = host_state
                # We force to update the aggregates info each time a
                # new request comes in, because some changes on the
                # aggregates could have been happening after setting
                # this field for the first time
                host_state.update(compute,
                                  dict(service),
                                  self._get_aggregates_info(host),
                                  self._get_instance_info(context, compute))
         """
        hosts = self._get_all_host_states(elevated, spec_obj,
            provider_summaries)
        
        # .....
        for num in range(num_instances):
            # 2
            #
            hosts = self._get_sorted_hosts(spec_obj, hosts, num)
```

#### _get_all_host_states 方法解析

这个方法主要在于确认 通过 resource provider uuid 寻找到的 compute node，有对应的 nova-compute service 启动。

为了程序的快捷维护添加了一个缓存：`self.host_state_map`

filterscheduler 使用 resource provider summaries 中的 resource provider uuid，先确认 compute 所以在的 cell， 然后根据 resource provider uuid 从 objects.ComputeNodeList 中找到对应 compute node 信息，从 servicelist 中选择 service 信息。

```python
# 通过 resource provider uuid 找到的 compute nodes 信息。
(Pdb) p compute_nodes
defaultdict(<type 'list'>, {'00000000-0000-0000-0000-000000000000': [], '06653bee-65a2-4cc8-8067-0a31b54196b2': [ComputeNode(cpu_allocation_ratio=16.0,cpu_info='',created_at=2018-01-16T11:08:35Z,current_workload=0,deleted=False,deleted_at=None,disk_allocation_ratio=1.0,disk_available_least=None,free_disk_gb=199,free_ram_mb=224799,host='172-18-211-194',host_ip=172.18.211.194,hypervisor_hostname='domain-c7.cdb6f267-8fd1-462c-a51e-77c648762cb0',hypervisor_type='VMware vCenter Server',hypervisor_version=6000000,id=2,local_gb=199,local_gb_used=0,mapped=1,memory_mb=245791,memory_mb_used=20992,metrics='[]',numa_topology=None,pci_device_pools=PciDevicePoolList,ram_allocation_ratio=1.5,running_vms=5,service_id=None,stats={io_workload='0',num_instances='5',num_os_type_None='2',num_os_type_linux='3',num_proj_fcd49087941f4f02a80f419a436a3c83='5',num_task_None='5',num_vm_active='4',num_vm_stopped='1'},supported_hv_specs=[HVSpec,HVSpec],updated_at=2018-04-16T07:20:56Z,uuid=ccafbbcb-b44f-4c19-b71d-f098ec0f78c5,vcpus=64,vcpus_used=10)]})

# 注意这里找到了两个 nova-compute。
(Pdb) p services
{u'aaaa': Service(availability_zone=<?>,binary='nova-compute',compute_node=<?>,created_at=2018-04-12T13:25:31Z,deleted=False,deleted_at=None,disabled=False,disabled_reason=None,forced_down=False,host='aaaa',id=8,last_seen_up=2018-04-12T13:34:53Z,report_count=56,topic='compute',updated_at=2018-04-12T13:34:53Z,uuid=eade4702-078d-4d52-a436-673e949bf5af,version=22),
 
u'172-18-211-194': Service(availability_zone=<?>,binary='nova-compute',compute_node=<?>,created_at=2018-01-16T08:27:43Z,deleted=False,deleted_at=None,disabled=False,disabled_reason=None,forced_down=False,host='172-18-211-194',id=7,last_seen_up=2018-04-16T07:21:26Z,report_count=707862,topic='compute',updated_at=2018-04-16T07:21:26Z,uuid=493c6c3a-5397-431c-b991-0d42230ef32d,version=22)}

# Nova 通过 resource provider uuid 找到 compute node， 再通过 computed host name 找到对应的 service。
/usr/lib/python2.7/site-packages/nova/scheduler/host_manager.py(671)_get_host_states()
669  	        for cell_uuid, computes in compute_nodes.items():
670  	            for compute in computes:
671  ->	                service = services.get(compute.host)

# 找到对应的 service 
(Pdb) p service
Service(availability_zone=<?>,binary='nova-compute',compute_node=<?>,created_at=2018-01-16T08:27:43Z,deleted=False,deleted_at=None,disabled=False,disabled_reason=None,forced_down=False,host='172-18-211-194',id=7,last_seen_up=2018-04-16T07:21:26Z,report_count=707862,topic='compute',updated_at=2018-04-16T07:21:26Z,uuid=493c6c3a-5397-431c-b991-0d42230ef32d,version=22)
```

#### def _schedule 中 def _get_sorted_hosts 方法解析 （filter，weight 功能）

使用封装的 HostManager 进行 filter 和 weight 工作。

```python
    def _get_sorted_hosts(self, spec_obj, host_states, index):
        """Returns a list of HostState objects that match the required
        scheduling constraints for the request spec object and have been sorted
        according to the weighers.
        """
        
        # 进行 filter
        # 使用 HostManager 对象中定义的 self.filter_handler = filters.HostFilterHandler()
        # 和 self.enabled_filters = self._choose_host_filters(self._load_filters()),
        # 来过滤 hosts.
        # 同时实现了 ingore hosts，force hosts，force nodes,spec_obj.requested_destination 
        # 等功能。
        # 此方法针对多个 hosts（compute node），多个 filters（enabled_filters）和
        # 一个 instance 进行筛选。将筛选过后的 host 交给 wiegher 处理
        filtered_hosts = self.host_manager.get_filtered_hosts(host_states,
            spec_obj, index)

        LOG.debug("Filtered %(hosts)s", {'hosts': filtered_hosts})

        if not filtered_hosts:
            return []
        
        # 对 filters 筛选过后的 hosts 进行 weight 处理
        # 在做 weight 时需要注意：
        # 1. 使用了
        weighed_hosts = self.host_manager.get_weighed_hosts(filtered_hosts,
            spec_obj)
        # Strip off the WeighedHost wrapper class...
        weighed_hosts = [h.obj for h in weighed_hosts]

        LOG.debug("Weighed %(hosts)s", {'hosts': weighed_hosts})

        # We randomize the first element in the returned list to alleviate
        # congestion where the same host is consistently selected among
        # numerous potential hosts for similar request specs.
        host_subset_size = CONF.filter_scheduler.host_subset_size
        if host_subset_size < len(weighed_hosts):
            weighed_subset = weighed_hosts[0:host_subset_size]
        else:
            weighed_subset = weighed_hosts
        chosen_host = random.choice(weighed_subset)
        weighed_hosts.remove(chosen_host)
        return [chosen_host] + weighed_hosts
```



**HostManager 对象中两个比较重要的方法**

注意这两个方法都是在 `_get_sorted_hosts` 这个方法中被定义，所以一下两个方法都是**针对一个个 instance spec_obj 来执行的**，一个 instance 经过 filter 过后筛选出 hosts，再将这些 hosts 进行 weight。

```python
# 首先可以看下 HostManager 中的属性
	# HostManager 中进行 filter 的地方
    def get_filtered_hosts(self, hosts, spec_obj, index=0):
        """Filter hosts and return only ones passing all filters."""
        
        #....
        
        # 这里需要注意 HostManager 中的几个属性
        # 1. self.filter_handler = filters.HostFilterHandler()
        # 2. self.enabled_filters = self._choose_host_filters(self._load_filters())
        return self.filter_handler.get_filtered_objects(self.enabled_filters,
                hosts, spec_obj, index)
    
    # ...
    # HostManager 中进行 weight 的地方
    def get_weighed_hosts(self, hosts, spec_obj):
        """Weigh the hosts."""
        
        # 这里需要注意 HostManager 中的几个属性
        # 1. weigher_classes = self.weight_handler.get_matching_classes(CONF.filter_scheduler.weight_classes)
        # 2. self.weighers = [cls() for cls in weigher_classes]
        # 3. self.weight_handler = weights.HostWeightHandler()
        return self.weight_handler.get_weighed_objects(self.weighers,
                hosts, spec_obj)
```

**针对 get_filtered_objects 主要功能等解析**

```python
class BaseFilterHandler(loadables.BaseLoader):
    """Base class to handle loading filter classes.

    This class should be subclassed where one needs to use filters.
    """
  
    # 注意这个方法被用在一个 for num in range(num_instances) 循环中，针对了每个 instance 做
    # 筛选
    def get_filtered_objects(self, filters, objs, spec_obj, index=0):
        
        # objs 指的是 host 经过上一步 provider resource 选取，并经过 service 
        # 验证过后的所有 host objects
        # spec_obj 是当前处理的 instance obj。其中之一
        # index 的是这个 instance 在 instance list（如果是批量创建）中的 index 索引。
        list_objs = list(objs)
        LOG.debug("Starting with %d host(s)", len(list_objs))
        # Track the hosts as they are removed. The 'full_filter_results' list
        # contains the host/nodename info for every host that passes each
        # filter, while the 'part_filter_results' list just tracks the number
        # removed by each filter, unless the filter returns zero hosts, in
        # which case it records the host/nodename for the last batch that was
        # removed. Since the full_filter_results can be very large, it is only
        # recorded if the LOG level is set to debug.
        part_filter_results = []
        full_filter_results = []
        log_msg = "%(cls_name)s: (start: %(start)s, end: %(end)s)"
        
        # fitlers 包含了所有 enabled_filters = RetryFilter,AvailabilityZoneFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
       
        for filter_ in filters:
            # 这里的 if 语句判断解释如下：
            #     
            # Set to true in a subclass if a filter only needs to be run once
    		# for each request rather than for each instance
    		# run_filter_once_per_request = False
            # 如果 run_filter_once_per_request 是 False，那么 filter 就只会在 nova-condutor
            # 请求后 run 一次，即使对第一个 index 为 0 的instance run 一次。
            if filter_.run_filter_for_index(index):
                cls_name = filter_.__class__.__name__
                start_count = len(list_objs)
                
                # 这里是 filter 真正在做 filter 的地方。每个 filter 会去实现 filter_all 方法。
                # list_objs 针对所有 hosts 和 一个 instance 进行 filter
                # **********************************************
                objs = filter_.filter_all(list_objs, spec_obj)
                # **********************************************
                
                if objs is None:
                    LOG.debug("Filter %s says to stop filtering", cls_name)
                    return
                
                # 注意着里，将筛选后的 list_objs 替换。
                # list_objs 可能会缩短。不符合的被 filter 掉。
                # **********************************************
                list_objs = list(objs)
                # **********************************************
                
                end_count = len(list_objs)
                part_filter_results.append(log_msg % {"cls_name": cls_name,
                        "start": start_count, "end": end_count})
                if list_objs:
                    remaining = [(getattr(obj, "host", obj),
                                  getattr(obj, "nodename", ""))
                                 for obj in list_objs]
                    full_filter_results.append((cls_name, remaining))
                else:
                    LOG.info(_LI("Filter %s returned 0 hosts"), cls_name)
                    full_filter_results.append((cls_name, None))
                    break
                LOG.debug("Filter %(cls_name)s returned "
                          "%(obj_len)d host(s)",
                          {'cls_name': cls_name, 'obj_len': len(list_objs)})
            ...
            # 这里将 list_objs 返回。
            return list_objs
```



**针对 get_weighed_objects 功能解析 **

```python
class BaseWeightHandler(loadables.BaseLoader):
    object_class = WeighedObject

    def get_weighed_objects(self, weighers, obj_list, weighing_properties):
        
        # weighers: weight_classes = nova.scheduler.weights.all_weighers
        # obj_list: 经过 filter 过后的 obj_list（这里是针对一个 instance）
        # weighing_properties: spec_obj
        """Return a sorted (descending), normalized list of WeighedObjects."""
        weighed_objs = [self.object_class(obj, 0.0) for obj in obj_list]

        if len(weighed_objs) <= 1:
            return weighed_objs

        for weigher in weighers:
            weights = weigher.weigh_objects(weighed_objs, weighing_properties)

            # Normalize the weights
            weights = normalize(weights,
                                minval=weigher.minval,
                                maxval=weigher.maxval)

            for i, weight in enumerate(weights):
                obj = weighed_objs[i]
                obj.weight += weigher.weight_multiplier() * weight
# (Pdb) p chosen_host
# (172-18-211-194, domain-c7.cdb6f267-8fd1-462c-a51e-77c648762cb0) ram: 224793MB disk: 203776MB io_ops: 0 instances: 5
# (Pdb) p  weighed_hosts
# []
# (Pdb) p [chosen_host] + weighed_hosts
# 返回值：[(172-18-211-194, domain-c7.cdb6f267-8fd1-462c-a51e-77c648762cb0) ram: 224793MB disk: 203776MB io_ops: 0 instances: 5]
        return sorted(weighed_objs, key=lambda x: x.weight, reverse=True)
    
    
```



#### def _schedule 中 filter 和 weight 过后的处理

```python
    def _schedule(self, context, spec_obj, instance_uuids,
            alloc_reqs_by_rp_uuid, provider_summaries):
        """Returns a list of hosts that meet the required specs, ordered by
        their fitness.

        These hosts will have already had their resources claimed in Placement.

        :param context: The RequestContext object
        :param spec_obj: The RequestSpec object
        :param instance_uuids: List of instance UUIDs to place or move.
        :param alloc_reqs_by_rp_uuid: Optional dict, keyed by resource provider
                                      UUID, of the allocation requests that may
                                      be used to claim resources against
                                      matched hosts. If None, indicates either
                                      the placement API wasn't reachable or
                                      that there were no allocation requests
                                      returned by the placement API. If the
                                      latter, the provider_summaries will be an
                                      empty dict, not None.
        :param provider_summaries: Optional dict, keyed by resource provider
                                   UUID, of information that will be used by
                                   the filters/weighers in selecting matching
                                   hosts for a request. If None, indicates that
                                   the scheduler driver should grab all compute
                                   node information locally and that the
                                   Placement API is not used. If an empty dict,
                                   indicates the Placement API returned no
                                   potential matches for the requested
                                   resources.
        """
        elevated = context.elevated()

        # Find our local list of acceptable hosts by repeatedly
        # filtering and weighing our options. Each time we choose a
        # host, we virtually consume resources on it so subsequent
        # selections can adjust accordingly.

        # Note: remember, we are using an iterator here. So only
        # traverse this list once. This can bite you if the hosts
        # are being scanned in a filter or weighing function.
        hosts = self._get_all_host_states(elevated, spec_obj,
            provider_summaries)
            
        # .......
        # 省略部分代码
        # .......
        
			else:
                instance_uuid = instance_uuids[num]

                # Attempt to claim the resources against one or more resource
                # providers, looping over the sorted list of possible hosts
                # looking for an allocation request that contains that host's
                # resource provider UUID
                claimed_host = None
                # 返回多个 hosts
                # 遍历返回的 hosts，根据 host.uuid 在 placmenet 返回的 allocation 中寻找
                # 对应的 allocation 记录。
                # 找到第一个后便 claim resource ，然后跳出 for host in hosts 循环
                for host in hosts:
                    cn_uuid = host.uuid
                    if cn_uuid not in alloc_reqs_by_rp_uuid:
                        LOG.debug("Found host state %s that wasn't in "
                                  "allocation requests. Skipping.", cn_uuid)
                        continue

                    alloc_reqs = alloc_reqs_by_rp_uuid[cn_uuid]
                    # 在 placement 服务中 claim resource
                    # return self.placement_client.claim_resources(instance_uuid,
                    # alloc_req, project_id, user_id)
                    # allocation request 真正在这里使用。
                    if self._claim_resources(elevated, spec_obj, instance_uuid,
                            alloc_reqs):
                        claimed_host = host
                        # 找到第一个后就跳出循环
                        break

                if claimed_host is None:
                    # We weren't able to claim resources in the placement API
                    # for any of the sorted hosts identified. So, clean up any
                    # successfully-claimed resources for prior instances in
                    # this request and return an empty list which will cause
                    # select_destinations() to raise NoValidHost
                    LOG.debug("Unable to successfully claim against any host.")
                    self._cleanup_allocations(claimed_instance_uuids)
                    return []

                claimed_instance_uuids.append(instance_uuid)

            LOG.debug("Selected host: %(host)s", {'host': claimed_host})
            # 将 claim 后的 host 加入 selected_hosts 列表
            selected_hosts.append(claimed_host)

            # 消费 resource, 这里从 request 中消费掉
            # Now consume the resources so the filter/weights will change for
            # the next instance.
            claimed_host.consume_from_request(spec_obj)
            if spec_obj.instance_group is not None:
                spec_obj.instance_group.hosts.append(claimed_host.host)
                # hosts has to be not part of the updates when saving
                spec_obj.instance_group.obj_reset_changes(['hosts'])
        
        # 从上面可以看出 selected_hosts 貌似没任何顺序，但是 append 就是顺序。
        # selected_hosts.append(claimed_host) 顺序对应了 instance 列表的顺序。
        # 在一下方法中比较 hosts 和 instances 列表长度是否一致
        # def select_destinations(self, context, spec_obj, instance_uuids,
        #    alloc_reqs_by_rp_uuid, provider_summaries):
        # 可参考下段代码
        
        return selected_hosts
```

**选取的 hosts  和 instance 如何映射**

用列表的 index 一一映射。

```python
        # 选取 selected_hosts
        selected_hosts = self._schedule(context, spec_obj, instance_uuids,
            alloc_reqs_by_rp_uuid, provider_summaries)

        # 比较 selected_hosts长度和 instance 长度是否一致
        #（可以看出一个 instance 又一个对应的 hosts，一一映射关系从列表顺序可以看出）
        # Couldn't fulfill the request_spec
        if len(selected_hosts) < num_instances:
            # NOTE(Rui Chen): If multiple creates failed, set the updated time
            # of selected HostState to None so that these HostStates are
            # refreshed according to database in next schedule, and release
            # the resource consumed by instance in the process of selecting
            # host.
            for host in selected_hosts:
                host.updated = None

            # Log the details but don't put those into the reason since
            # we don't want to give away too much information about our
            # actual environment.
            LOG.debug('There are %(hosts)d hosts available but '
                      '%(num_instances)d instances requested to build.',
                      {'hosts': len(selected_hosts),
                       'num_instances': num_instances})

            reason = _('There are not enough hosts available.')
            raise exception.NoValidHost(reason=reason)
```

以上记录了下 Openstack P 版的 scheduler manager， filter_scheduler，host manager 筛选 host 的大致过程，这里并没有对每个 filter 进行分析。以后再针对每个 filter 进行记录。

以下记录些针对 filter 的介绍：

https://cloudarchitectmusings.com/2013/06/26/openstack-for-vmware-admins-nova-compute-with-vsphere-part-2/

https://platform9.com/blog/virtual-machine-placement-openstack/


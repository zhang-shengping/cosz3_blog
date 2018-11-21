---
title: Neutron Ml2 plugins (M) note 1
categroies:
- Neutron
tags:
- code
- self-note
---

这里记载下在学习 Neutron （Mitaka version）时候的一些 Notes，可能也包含一些代码。

Ml2 是 Neutron server 的 core plugin， Ml2Plugin 定义在 /opt/stack/neutron/neutron/plugins/ml2/plugin.py 中。

Ml2Plugin 的描述：

  Implement the Neutron L2 abstractions using modules.

  Ml2Plugin is a Neutron plugin based on separately extensible sets
  of network types and mechanisms for connecting to networks of
  those types. The network types and mechanisms are implemented as
  drivers loaded via Python entry points. Networks can be made up of
  multiple segments (not yet fully implemented).


从描述中可以看书 Ml2Plugin 主要管理的是三种 drviers （type driver， mechanism driver 和 extension driver。 定义在 /etc/neutron/plugins/ml2/ml2_conf.ini 中）

1. type driver 和 extension driver： 主要是给 neutron server 提供 API 接口 （server 端必须配置）
2. mechanism driver： 主要定义了 hooks，用来配置相应的网络设备（如 ovs）（agent 端必须配置）


Ml2 core plugin 支持的 services 种类：
```
_supported_extension_aliases = ["provider", "external-net", "binding",
                                "quotas", "security-group", "agent",
                                "dhcp_agent_scheduler",
                                "multi-provider", "allowed-address-pairs",
                                "extra_dhcp_opt", "subnet_allocation",
                                "net-mtu", "vlan-transparent",
                                "address-scope",
                                "availability_zone",
                                "network_availability_zone",
                                "default-subnetpools"]
```
Ml2Plugin 通过继承不同的类来实现了支持不同的 services。 Ml2Plugin 类中也暴露了许多相关 service 的接口，供 neutron server 使用。


看下 Ml2Plugin 初始化都做了些什么：
```
def __init__(self):
    # First load drivers, then initialize DB, then initialize drivers
    self.type_manager = managers.TypeManager()
    self.extension_manager = managers.ExtensionManager()
    self.mechanism_manager = managers.MechanismManager()
    super(Ml2Plugin, self).__init__()
    self.type_manager.initialize()
    self.extension_manager.initialize()
    self.mechanism_manager.initialize()
    self._setup_dhcp()
    self._start_rpc_notifiers()
    self.add_agent_status_check(self.agent_health_check)
    self._verify_service_plugins_requirements()
    LOG.info(_LI("Modular L2 Plugin initialization complete"))
```
1. 实例化了 TypeManager（用来加载和管理不同种类的 type_driver 的对象）
2. 实例化了 ExtensionManager（用来加载和管理不同种类的 extension_driver 的对象）
3. 实例化了 MechanismManager（用来加载和管理不同种类的 mechanise_driver 的对象）
4. 初始化所有继承的父类
5. 初始化 type driver
6. 初始化 extension driver
7. 初始化 mechanism driver
8. 设置 dhcp 服务定时检测服务 (注意这里的检测服务都是从数据库中检查状态，不同于 Telemetry 的检测方式)，如果 neutron.conf 中设置了 allow_automatic_dhcp_failover 为 true，则会将网络配置从 down 掉的 dhcp agent 迁移到没有 down 掉的 dhcp agent 上。（到 Neutron DHCP 服务相关内容时，再详细记录 `self.add_agent_status_check(self.remove_networks_from_down_agents)`）
9. _start_rpc_notifiers 创建 openvswitch agent 和 dhcp 服务的 rpc client 端。
10. 设置 openvswitch agent 服务定时检测服务 (注意这里的检测服务都是从数据库中检查状态，不同于 Telemetry 的检测方式),（这里相关内容，在 openvswitch agent notes 中再添加。）
11.检查自定义的 service_plugins 是否需要 Ml2 中 其他的 requirements。

## Ml2Plugin agent 中的 Message Queues

Neutron 的大部分 Plugins 中都会包含三个主要的作用：1.创建，管理，调用 RPC 2.消费 RPC（?)，3.给 wsgi 服务提供相应网络服务的接口，

下面会围绕着三点来对 Ml2Plugin 进行分析。

### 创建 RPC client：

从上面的 `__init__` 方法中包含了 RPC client 创建的函数：`self._start_rpc_notifiers()`.
```
@log_helpers.log_method_call
def _start_rpc_notifiers(self):
    """Initialize RPC notifiers for agents."""
    self.notifier = rpc.AgentNotifierApi(topics.AGENT)
    self.agent_notifiers[const.AGENT_TYPE_DHCP] = (
        dhcp_rpc_agent_api.DhcpAgentNotifyAPI()
    )
```
这个方法主要是用来创建两类的 RPC client， 一是 openvswitch agent 的 client， 二是 dhcp service 的 RPC client。

AgentNotifierApi 是在 plugins/ml2/plugin.py 中定义的. AgentNotifierApi 还继承了几个类，这几个类中也定义了 RPC client 相关的 topic 和需要调用的方法，这里 AgentNotifierApi 中的 *** topic 参数在 neutron/common/topics.py 中定义，值为 `AGENT = q-agent-notifier`。

#### AgentNotifierApi Target 中的 topic 是 `q-agent-notifier` 这个值说明了：
  1. 当使用 topic 方式的 RPC 时，在 target 中没有定义 exchange 的情况下，message 将发送给 Neutron 默认的 control exchange `neutron`，且 topic `q-agent-notifier` 指定的是 routing_key, 即将 message 送给通过 `q-agent-notifier` routing key 绑定在 `neutron` exchange 的 queue 中。这里有个问题 topic 的 queue 是如何绑定到 exchange 上的 declare_topic_consumer ？？？
  2. 当使用 fanout 方式 RPC 时，target 中定义的 exchange 将不起作用， exchange name 将定义为 topic 名 + `_fanout`, 因为是 fanout 模式所以也不存在 routing_key（topic）。message 将发送到所有和 exchange 相连的 queues 中。（虽然使用 rabbitmqctl list_bindings 可以列出来 fanout exchange 连接 queue 的 binding，但是这些 binding 在 message 传送过程中不会被使用。因为 fanout message 中不包含 routing_key，所以不可能有 routing_key 和 binding 之间的 match 动作。） 这里有个问题 fanout 的 queue 是如何创建和绑定上的 declare_fanout_consumer ？？？？
  3. 从上面两点可以简单总结 neutron server 像 agent 发送消息一般有两类：topic，fanout。会发送的绑定在 neutron exchange 或者以 topic + `fanout` 为名的 exchange 上。如果是 topic 类型 binding key 以 q-agent-notifier 开头的。

```
# 父类中也定义了类似 network_delete 的其他网络功能的接口
class AgentNotifierApi(dvr_rpc.DVRAgentRpcApiMixin,
                       sg_rpc.SecurityGroupAgentRpcApiMixin,
                       type_tunnel.TunnelAgentRpcApiMixin):
    """Agent side of the openvswitch rpc API.

    API version history:
        1.0 - Initial version.
        1.1 - Added get_active_networks_info, create_dhcp_port,
              update_dhcp_port, and removed get_dhcp_port methods.
        1.4 - Added network_update
    """

    def __init__(self, topic):
        self.topic = topic
        # 创建新的 topic
        self.topic_network_delete = topics.get_topic_name(topic,
                                                          topics.NETWORK,
                                                          topics.DELETE)
        self.topic_port_update = topics.get_topic_name(topic,
                                                       topics.PORT,
                                                       topics.UPDATE)
        self.topic_port_delete = topics.get_topic_name(topic,
                                                       topics.PORT,
                                                       topics.DELETE)
        self.topic_network_update = topics.get_topic_name(topic,
                                                          topics.NETWORK,
                                                          topics.UPDATE)

        target = oslo_messaging.Target(topic=topic, version='1.0')
        self.client = n_rpc.get_client(target)

    def network_delete(self, context, network_id):
        # 发送 message 前跟换 target 的 topic
        cctxt = self.client.prepare(topic=self.topic_network_delete,
                                    fanout=True)
        cctxt.cast(context, 'network_delete', network_id=network_id)

    def port_update(self, context, port, network_type, segmentation_id,
                    physical_network):
        cctxt = self.client.prepare(topic=self.topic_port_update,
                                    fanout=True)
        cctxt.cast(context, 'port_update', port=port,
                   network_type=network_type, segmentation_id=segmentation_id,
                   physical_network=physical_network)

    def port_delete(self, context, port_id):
        cctxt = self.client.prepare(topic=self.topic_port_delete,
                                    fanout=True)
        cctxt.cast(context, 'port_delete', port_id=port_id)

    def network_update(self, context, network):
        cctxt = self.client.prepare(topic=self.topic_network_update,
                                    fanout=True, version='1.4')
        cctxt.cast(context, 'network_update', network=network)
```

#### DhcpAgentNotifyAPI Target 中的 topic 是 `dhcp_agent` 这个值说明了：
可以类比 AgentNotifierApi Target 的分析。

```
class DhcpAgentNotifyAPI(object):
    """API for plugin to notify DHCP agent.

    This class implements the client side of an rpc interface.  The server side
    is neutron.agent.dhcp_agent.DhcpAgent.  For more information about changing
    rpc interfaces, please see doc/source/devref/rpc_api.rst.
    """
    # It seems dhcp agent does not support bulk operation
    VALID_RESOURCES = ['network', 'subnet', 'port']
    VALID_METHOD_NAMES = ['network.create.end',
                          'network.update.end',
                          'network.delete.end',
                          'subnet.create.end',
                          'subnet.update.end',
                          'subnet.delete.end',
                          'port.create.end',
                          'port.update.end',
                          'port.delete.end']

    def __init__(self, topic=topics.DHCP_AGENT, plugin=None):
        self._plugin = plugin
        target = oslo_messaging.Target(topic=topic, version='1.0')
        self.client = n_rpc.get_client(target)
        # register callbacks for router interface changes
        registry.subscribe(self._after_router_interface_created,
                           resources.ROUTER_INTERFACE, events.AFTER_CREATE)
        registry.subscribe(self._after_router_interface_deleted,
                           resources.ROUTER_INTERFACE, events.AFTER_DELETE)
```


### 创建 RPC consumer：

```
@log_helpers.log_method_call
def start_rpc_listeners(self):
    """Start the RPC loop to let the plugin communicate with agents."""
    self._setup_rpc()
    self.topic = topics.PLUGIN
    self.conn = n_rpc.create_connection()
    self.conn.create_consumer(self.topic, self.endpoints, fanout=False)
    self.conn.create_consumer(
        topics.SERVER_RESOURCE_VERSIONS,
        [resources_rpc.ResourcesPushToServerRpcCallback()],
        fanout=True)
    # process state reports despite dedicated rpc workers
    self.conn.create_consumer(topics.REPORTS,
                              [agents_db.AgentExtRpcCallback()],
                              fanout=False)
    return self.conn.consume_in_threads()

def start_rpc_state_reports_listener(self):
    self.conn_reports = n_rpc.create_connection(new=True)
    self.conn_reports.create_consumer(topics.REPORTS,
                                      [agents_db.AgentExtRpcCallback()],
                                      fanout=False)
    return self.conn_reports.consume_in_threads()
```

plugin 是一个 rpc client `_start_rpc_notifiers` 也是一个 rpc consumer `start_rpc_listeners` (neutron server start listeners)？

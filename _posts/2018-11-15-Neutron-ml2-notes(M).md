---
title: Neutron plugins 和 services (M) note 0
categroies:
- Neutron
tags:
- code
- self-note
---

这里记载下在学习 Neutron （Mitaka version）时候的一些 Notes，可能也包含一些代码。

Neutron API server 的启动方式有两种，一种是使用 legacy 的 wsgi eventlet， 还有一种是使用 wsgi_pecan 的方式，
这两种方式都会调用 neutron/service.py 下面的 `start_plugin_workers` 方法来启动 plugins（core plugins (默认是 Ml2 plugin) 和 service plugins）。

Neutron 的 Ml2 plugins 和 service plugins 的管理器是 neutron/manager.py 中 `NeutronManager`（singleton）class。
NeutronManager 主要功能是解析配置文件，主要提供加载 plugins 的方法如 _load_service_plugins。

（NOTE：plugins vs service：这里要说明下 plugins 和 services 的关系，一个 plugins 可能提供多种 service，如果 Ml2 core plugin。）

`_load_service_plugins` 方法加载了三种 plugins：
  1. core Ml2 plugin （如果有些网络 service 已经被 core plugin 支持，那么会使用 core plugin， `_load_services_from_core_plugin`）
  2. 默认加载的一些 service plugins `DEFAULT_SERVICE_PLUGINS` 常量中定义的 （auto_allocate, tag, timestamp_core, network_ip_availability）
  3. 加载配置 neutron.conf 配置文中定义的 service_plugins.

这些 plugins 中有些（如 service_plugins）会使用直接加载的方式，也有的会使用 entry_points 中的 namespace `neutron.service_plugins` 加载方式加载到内存。加载后的 plugins 实例都会暴露出来相应的方法接口给对映的 wsgi 服务。

最终对映 service 的名称（`plugin.plugin_type`）和 plugin 的实例会被写入 `self.service_plugins` 字典中。

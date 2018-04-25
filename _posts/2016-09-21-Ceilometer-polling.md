---
title: Ceilometer polling
categories:
- Ceilometer
tags:
- code
- self-note
- ceilometer
---
Ceilometer 有很多的queue，有时候我会比较困惑哪些这些queue和这些服务之间的关系。所以借此文自己梳理一下这些关系，以备后患。

Ceilometer 主要分为三个组件 ceilometer-polling，ceilometer-notification-agent 和 ceilometer-collector。三个组件分别负责采数据，处理数据，分发数据。下面就一个个梳理下这些服务和各个Queue的关系。



# Ceilometer-polling：

## 代码部分：

显示polling服务在agent/manager.py文件下，被封装成了一个PollingTask类。

Polling Task 类定义了一个poll_and_notify方法。这个方法完成了拉取数据的主要功能： 1. 探索资源， 2. 拉取数据，3.分发数据到queue上。

这里直接看它如何发数据：

```python
   ...

    if self._batch:
        sample_batch.append(sample_dict)
    else:
                # 单个处理
        self._send_notification([sample_dict])

if sample_batch:
        # 批处理
    self._send_notification(sample_batch)

   ...

# 批处理和单个处理掉的都是一个方法
def _send_notification(self, samples):
    self.manager.notifier.sample(
        self.manager.context.to_dict(),
        'telemetry.polling',
        {'samples': samples}
    )

# _send_notification 封装 oslo_messaging的Notifier
# 定义在了Manager对象的notifier属性上。
self.notifier = oslo_messaging.Notifier(
    messaging.get_transport(),
    driver=cfg.CONF.publisher_notifier.telemetry_driver,
    publisher_id="ceilometer.api")


----------------------------------------------------------------

# 这里我们关心的是这个samples到底notifier到哪里去了。
# 但是这里单从Notifier的构造方式上是看不出来的。
# 原因是Notifier 声明topic和exchange的地方都不那么明显。
# 这里我们就要去 Notifier定义的地方去看看了
# 在Oslo_messaging 下有 notifier.py 文件里面定义了Notifier类，不是很长。

        ...
class Notifier(object):
    """Send notification messages.
        ...
   def __init__(self, transport, publisher_id=None,
                 driver=None, topic=None,
                 serializer=None, retry=None):
    ...
                # oslo_messaging这里借了transport中的conf的配置项用用。
                # 即是conf中包涵了咱们配置文件的很多内容。
        transport.conf.register_opts(_notifier_opts)
        self.transport = transport
        self.publisher_id = publisher_id
        self.retry = retry

                # 这里应该是我们常用的messageV2的格式了。
        self._driver_names = ([driver] if driver is not None
                              else transport.conf.notification_driver)

                # 这里topic指定了我们要将message发送给哪个topic
                # 这里看到先会看我们Notifier里面有没有定义，
        # 如果没有定义那么我们就用 配置文件中的topic
                # 通常情况下配置文件中的topic是 “notification_topics = notifications“
                # 我刚刚问了下我旁边的同学，我们可以grep到的queue是什么名字
                # 分别是 “notification.sample” 和 “notification.info”
                # 这里为什么有一个sample和一个info呢？
                # 因为，ceilometer-polling调用Notifier的sample方法去发送消息
                # 默认情况下 oslo_messaging 会建立 topic+ . + method 格式的queue去接受相应的notification。
                # 那么 “notification.sample” 就是polling发给notification-agent的那个queue。
                # “notification.info” 很多openstack service 会向这个queue上发notification消息。
        self._topics = ([topic] if topic is not None
                        else transport.conf.notification_topics)

        self._serializer = serializer or msg_serializer.NoOpSerializer()
        self._driver_mgr = named.NamedExtensionManager(
            'oslo.messaging.notify.drivers',
            names=self._driver_names,
            invoke_on_load=True,
            invoke_args=[transport.conf],
            invoke_kwds={
                'topics': self._topics,
                'transport': self.transport,
            }
        )
    _marker = object()



# ------------------------- 其实这里还有个东西没有看，那就是exchange定义，Notifier的exchange定义比它的topic定义还要隐蔽 (如果只是观察queue的话，一般exchange不是那么重要) ------------------------------


# polling也是调用的对象是Notifier去做send任务，所有东西都被Notifier给封装了，但是notifier也没有明显的定义exchange。

# 借上面的Notifier对象继续
# 这里找到了topic到位置。
self._topics = ([topic] if topic is not None
                        else transport.conf.notification_topics)

        self._serializer = serializer or msg_serializer.NoOpSerializer()

               # 这里调用的插件是notification的插件
              # 这里找的是 ”oslo.messaging.notify.drivers“ namespace 下的 self._driver_names
        # 通常情况下就是 messagev2
        self._driver_mgr = named.NamedExtensionManager(
            'oslo.messaging.notify.drivers',
            names=self._driver_names,
            invoke_on_load=True,
            invoke_args=[transport.conf],
            invoke_kwds={
                'topics': self._topics,
                'transport': self.transport,
            }
        )

# 在messagev2类被定义在oslo_messaging/notify/_impl_messaging.py 文件中，继承于MessagingDriver。

class MessagingDriver(notifier._Driver):

    def __init__(self, conf, topics, transport, version=1.0):
        super(MessagingDriver, self).__init__(conf, topics, transport)
        self.version = version
    def notify(self, ctxt, message, priority, retry):
        priority = priority.lower()
        for topic in self.topics:
            target = oslo_messaging.Target(topic='%s.%s' % (topic, priority))
            try:

                             # 最终polling的Notifier会来调用 notify 方法中的 self.transport._send_notification 方法。
                             # 而这里的self.transport对象既是我们先前调用ceilometer/agent/manager.py self.notifier 对象用get_transport方法传入的transport对象。
                self.transport._send_notification(target, ctxt, message,
                                                  version=self.version,
                                                  retry=retry)

            except Exception:
                LOG.exception("Could not send notification to %(topic)s. "
                              "Payload=%(message)s",
                              dict(topic=topic, message=message))

class MessagingV2Driver(MessagingDriver):
    "Send notifications using the 2.0 message format."
    def __init__(self, conf, **kwargs):
        super(MessagingV2Driver, self).__init__(conf, version=2.0, **kwargs)

---------------------------------------------------------------------------------------------------------------------------------------------------

# 看看transport.send_notification 是做什么的吧。到 oslo_messaging的transport.py 文件下可以看到_send_notification又封装了一层

    ...

    def _send_notification(self, target, ctxt, message, version, retry=None):
        if not target.topic:
            raise exceptions.InvalidTarget('A topic is required to send',
                                           target)

               # 这里的driver就是get_transport方法中oslo_messaging 通过URL 解析出来的rabbit driver。
          # transport对象既是封装了底层driver的上层对象。
        self._driver.send_notification(target, ctxt, message, version,
                                       retry=retry)


   ...

-------------------------------------------------------------------------

# 那就看下rabbit是如何发送notifications的。在 oslo_messaging/_drivers/impl_rabbit.py的RabbitDriver是找不到 send_notification方法的。
# 因为RabbitDriver只是个包装，它包装了amqpdriver.AMQPDriverBase 这个对象，在同样的目录下可以找到 amqpdriver.py

    # AMQPDriverBase 对象中有 send_notification 有包装了 self._send
    def send_notification(self, target, ctxt, message, version, retry=None):
        return self._send(target, ctxt, message,
                          envelope=(version == 2.0), notify=True, retry=retry)

# 在self._send 中我们可以看到如下
        try:
                   # 得到连接
            with self._get_connection(rpc_amqp.PURPOSE_SEND) as conn:
                             notify 是false
                if notify:
                    conn.notify_send(self._get_exchange(target),
                                     target.topic, msg, retry=retry)
                               # 直接发送到fanout exchange
                elif target.fanout:
                    conn.fanout_send(target.topic, msg, retry=retry)
                else:
                    topic = target.topic
                    if target.server:
                        topic = '%s.%s' % (target.topic, target.server)

                                     # 根据topic发送到某个exchange。
                    conn.topic_send(exchange_name=self._get_exchange(target),
                                    topic=topic, msg=msg, timeout=timeout,
                                    retry=retry)

# 再看下这个方法。如果设置了target则从target中取 exchange，如果没有则取transport中default的exchange
    def _get_exchange(self, target):
        return target.exchange or self._default_exchange

# 那么transport中default的exchange是什么。
# 就还得看回get_transport方法。
    kwargs = dict(default_exchange=conf.control_exchange,
                  allowed_remote_exmods=allowed_remote_exmods)
# 其实并不是这样。看exchange
# 而是应该去prepare_service 方法中有set_transport_defaults 将control_exchange设置成了’ceilometer’ 而不是openstack。
# 从这里我们会有两个方面去看到transport层，一是底层用的什么driver，二是底层定义的是什么exchange。
```

## 图解：

![](https://ws4.sinaimg.cn/large/006tKfTcly1fqowdlovyyj313r0l73yt.jpg)



# Ceilometer-notification-agent (承前启后，拉取和分发):

## notification-agent 是做什么的

notification-agent 主要用来拉取 notification bus 上的所有信息，比如说nova，neutron, ceilometer, cinder, glance, keystone, heat, swift 等等的信息。

拉取来信息后，notification会根据 pipeline.yaml 的定义来处理 sample 和 info 信息。

如果在 ceilometer.conf 文件中定义了store_events=True, 那么notifications-agent 会根据 event_pipeline.yaml 上的数据将event 类的数据储存。

所以说如果想不存储和接受 event 数据，我们可以设置 store_events = False。其实这样说也收集event数据。因为ceilometer 有个遗留问题，是将event 类信息以meter类信息存储。

因此，如果想不存储任何 event 信息。那就设置 store_event = False 同时 disable_non_metric_meters=true

```python
# WARNING: Ceilometer historically offered the ability to store events
# as meters. This usage is NOT advised as it can flood the metering
# database and cause performance degradation. (boolean value)
disable_non_metric_meters = true
```

## notification-agent 拉取信息

这里要说明两个问题, 一是notification-agent怎么拉取信息， 二是拉取哪里的信息。

首先notification-agent怎么拉数据,  肯定是创建了notification listener，那么在哪里创建的。

```python
# 在ceilometer/notification.py 文件下有定义_configure_main_queue_listeners 方法
    def _configure_main_queue_listeners(self, pipe_manager,
                                        event_pipe_manager):

             # 去加载在ceilometer.notification 命名空间下的插件

        notification_manager = self._get_notifications_manager(pipe_manager)

        ...

        endpoints = []

        ...

        targets = []
        for ext in notification_manager:
            handler = ext.obj

                ...

            # NOTE(gordc): this could be a set check but oslo_messaging issue
            # https://bugs.launchpad.net/oslo.messaging/+bug/1398511
            # This ensures we don't create multiple duplicate consumers.
            for new_tar in handler.get_targets(cfg.CONF):
                if new_tar not in targets:
                    targets.append(new_tar)
            endpoints.append(handler)
        urls = cfg.CONF.notification.messaging_urls or [None]
        for url in urls:
            transport = messaging.get_transport(url)
            listener = messaging.get_notification_listener(
                transport, targets, endpoints)
            listener.start()
            self.listeners.append(listener)

meter = ceilometer.meter.notifications:ProcessMeterNotifications
_sample = ceilometer.telemetry.notifications:TelemetryIpc
http://www.cnblogs.com/gsk99/archive/2010/12/13/1904541.html
```

但是更具上面信息还是看不出，notification agent  是去哪里拉去信息的。其实，这些信息都是由各个插件定义的，先看一下所有notifications插件的基类NotificationBase。

```python
# 在agent/plugin_base.py 下可以找到所有notifications的基类NotificationBase
class NotificationBase(PluginBase):
    """Base class for plugins that support the notification API."""

     # 初始化
    def __init__(self, manager):
        super(NotificationBase, self).__init__()
        # NOTE(gordc): this is filter rule used by oslo.messaging to dispatch
        # messages to an endpoint.
        # 用于筛选
        if self.event_types:
            self.filter_rule = oslo_messaging.NotificationFilter(
                event_type='|'.join(self.event_types))
        self.manager = manager


     # 这里返回topics
     # 可以看到这里放返回的topic是定义在conf文件种notification_topics
    # 一般默认情况下是 notifications
    @staticmethod
    def get_notification_topics(conf):
        if 'notification_topics' in conf:
            return conf.notification_topics
        return conf.oslo_messaging_notifications.topics

     # 不同插件可能有不通的值，用来标识不通的信息。
    @abc.abstractproperty
    def event_types(self):
        """Return a sequence of strings.
        Strings are defining the event types to be given to this plugin.
        """

    # 不同插件所绑定的exchange和topic都是不同的，这里返回插件绑定的exchange
    @abc.abstractmethod
    def get_targets(self, conf):
        """Return a sequence of oslo.messaging.Target.
        Sequence is defining the exchange and topics to be connected for this
        plugin.
        :param conf: Configuration.
        """

     # 处理notification信息。
    @abc.abstractmethod
    def process_notification(self, message):
        """Return a sequence of Counter instances for the given message.
        :param message: Message to process.
        """

     # 接受MQ上info类信息。即 notification.info queue上的信息。
    def info(self, ctxt, publisher_id, event_type, payload, metadata):
        """RPC endpoint for notification messages at info level
        When another service sends a notification over the message
        bus, this method receives it.
        :param ctxt: oslo.messaging context
        :param publisher_id: publisher of the notification
        :param event_type: type of notification
        :param payload: notification payload
        :param metadata: metadata about the notification
        """
        notification = messaging.convert_to_old_notification_format(
            'info', ctxt, publisher_id, event_type, payload, metadata)
        self.to_samples_and_publish(context.get_admin_context(), notification)

     # 接受MQ上sample类信息。即 notification.sample queue上的信息。
    def sample(self, ctxt, publisher_id, event_type, payload, metadata):
        """RPC endpoint for notification messages at sample level
        When another service sends a notification over the message
        bus at sample priority, this method receives it.
        :param ctxt: oslo.messaging context
        :param publisher_id: publisher of the notification
        :param event_type: type of notification
        :param payload: notification payload
        :param metadata: metadata about the notification
        """
        notification = messaging.convert_to_old_notification_format(
            'sample', ctxt, publisher_id, event_type, payload, metadata)
        self.to_samples_and_publish(context.get_admin_context(), notification)

     # 分发信息的方法。
    def to_samples_and_publish(self, context, notification):
        """Return samples produced by *process_notification*.
        Samples produced for the given notification.
        :param context: Execution context from the service or RPC call
        :param notification: The notification to process.
        """
        with self.manager.publisher(context) as p:
            p(list(self.process_notification(notification)))

# 由NotificationBase这个类看出。它定义了所有的关键信息。
# 1. get_targets 和 get_notification_topics 方法确定了是哪个scope内哪个topic上的信息。
# 2. info 和 sample 则是确定了去拉去哪种类型的信息。
# 3. to_samples_and_publish 定义了信息拉取到后如何处理信息。

*******
# 现在结合上面代码 和 在_configure_main_queue_listeners 中的 messaging.get_notification_listener
# messaging.get_notification_listener 中的 targets 将会定义两个值exchange 和 topic 用来确定 queue的位置，这就会用到每个插件的get_tragets 和 get_notification_topics 的返回值。
# messaging.get_notification_listener 中的 endpoints 定义接受什么样的信息，这里就会找对映的处理信息的方法，如每个插件的info 和 sample方法。
******


Publisher's name is plugin name in setup.cfg
```

## notification-agent 分发信息

```python
# 从上面代码可以追溯notification是通过此方法分发数据的。
    def to_samples_and_publish(self, context, notification):
        """Return samples produced by *process_notification*.
        Samples produced for the given notification.
        :param context: Execution context from the service or RPC call
        :param notification: The notification to process.
        """
        with self.manager.publisher(context) as p:
            p(list(self.process_notification(notification)))

# 跟踪代码可以得到 pipeline.py 文件中的 class PublishContext(object) 类会被用到
class PublishContext(object):
 ...

       # 这里返回的时没个pipelinet对象中的publish_data方法的value
    def __enter__(self):
        def p(data):
            for p in self.pipelines:
                p.publish_data(self.context, data)
        return p

# publish_data 被EventPipeline 和SamplePipeline 对象实现。并且去调用对应Sink对象中的publish_events 和publish_samples.
# 如下
    def publish_data(self, ctxt, samples):
        if not isinstance(samples, list):
            samples = [samples]
        supported = [s for s in samples if self.source.support_meter(s.name)
                     and self._validate_volume(s)]
        # 在这
        self.sink.publish_samples(ctxt, supported)


# Sink 和Pipeline类似有两个子类 EventSink SampleSink。
# 这两个字类实现了 根据插件实现了不同的 publisher
# 如下
class SampleSink(Sink):
      # 插件在ceilometer.publisher 下
    NAMESPACE = 'ceilometer.publisher'
      ...

         def publish_samples(self, ctxt, samples):
        self._publish_samples(0, ctxt, samples)

     ....


---------------------------------------------------

# 最长用的插件是notifier = ceilometer.publisher.messaging:SampleNotifierPublisher
# 可以去publisher/messaging.py文件下看下它的实现

class NotifierPublisher(MessagingPublisher):
    def __init__(self, parsed_url, default_topic):
        super(NotifierPublisher, self).__init__(parsed_url)
        options = urlparse.parse_qs(parsed_url.query)
        topic = options.get('topic', [default_topic])[-1]
        self.notifier = oslo_messaging.Notifier(
            messaging.get_transport(),
            driver=cfg.CONF.publisher_notifier.telemetry_driver,
            publisher_id='telemetry.publisher.%s' % cfg.CONF.host,
            topic=topic,
            retry=self.retry
        )

       # 定义发送什么样的信息。event和metering统一成为sample数据。
    def _send(self, context, event_type, data):
        try:
            self.notifier.sample(context.to_dict(), event_type=event_type,
                                 payload=data)
        except oslo_messaging.MessageDeliveryFailure as e:
            raise_delivery_failure(e)

# 继承于上面的 NotifierPublisher， 同时设置topic 为publisher_notifier.metering_topic
# 则默认topic是 metering_topic = metering
class SampleNotifierPublisher(NotifierPublisher):
    def __init__(self, parsed_url):
        super(SampleNotifierPublisher, self).__init__(
            parsed_url, cfg.CONF.publisher_notifier.metering_topic)

# 继承于上面的 NotifierPublisher， 同时设置topic 为publisher_notifier.event_topic
# 则默认topic是 event_topic = event
class EventNotifierPublisher(NotifierPublisher):
    def __init__(self, parsed_url):
        super(EventNotifierPublisher, self).__init__(
            parsed_url, cfg.CONF.publisher_notifier.event_topic)

****************
# 由上知 notifications-agent的数据发送到了metering.sample 和 event.sample queue 上。
*****************
```

## 小结一下

polling -----> notification.sample -------> notification -------> metering.sample

service -----> [notification.info](http://notification.info) -----------> notification -------> event.sample


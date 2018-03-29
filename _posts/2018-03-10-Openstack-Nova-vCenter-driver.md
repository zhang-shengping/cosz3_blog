---
title: Openstack Nova 管理 VMware vCenter 接口
categories:
- Nova
tags:
- nova
- vCenter
---

Nova 提供了管理 vmware 虚拟机的方法，主要是通过 Nova 的 vmware driver 调用 vCenter webservice API 来实现。这片文章主要记录下 Nova virt driver vmwareapi 是如何实现对 vCenter 的管理。
## WSDL + SOAP

WSDL (Web Services Definition Language):

WSDL 是用于描述 Web Service 的一种 XML 表示法。WSDL 定义告诉客户如何编写 Web Service 请求，并且描述了由 Web Service 提供程序提供的接口。

https://wsvc.cdiscount.com/MarketplaceAPIService.svc?wsdl

SOAP (Simple Object Access Protocol ):

SOAP 是一个基于 XML 格式文件的协议，通常用 HTTP 协议来发送给 server 端，来实现远程方法调用。

SOAP reference: https://www.tutorialspoint.com/soap/soap_quick_guide.htm

## vSphere Object Model

Vsphere  通过以下三种类型 (type) 对象来操作 VCenter：

1. Managed Object：服务端的管理对象。
2. Managed Object Reference：客户端对服务端一个管理对象的索引。
3. Data Object ：描述管理对象中的数据，定义客户端和服务端之间传输数据的格式。

### Managed Object

Managed Object 是 VCenter Server 中所有管理对象的基类，在 Managed Objcet 上 VCenter 扩展了许多不同的管理对象。

“The VMware vSpheremanagement object model is a complex system of data structures designed toprovision, manage, monitor, and control the life-cycle of all components thatcan possibly comprise virtual infrastructure.”

不同的 managed object 用来管理 vsphere 的不同部分。

refer: https://www.vmware.com/support/developer/converter-sdk/conv60_apireference/index-mo_types.html

#### Managed object 主要分两类：

* Managed objects that extendthe [ManagedEntity](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ManagedEntity.html) managed object type,and thus, are components that comprise the inventory of virtualcomponents. For example, instances of host systems ([HostSystem](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.HostSystem.html)), virtual machines ([VirtualMachine](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.VirtualMachine.html)), and datastores ([Datastore](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.Datastore.html)) are inventoried objects,and are referred to generically as managed entities.  (用来管理虚拟组件 [inventory](https://172.28.8.247/vsphere-client/?csp))
* Managed objects that provideservices for the entire system. Managed objects in this category enablemanaging performance ([PerformanceManager](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.PerformanceManager.html)), managing licenses forVMware products (LicenseManager), and managing virtual storage ([VirtualDiskManager](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.VirtualDiskManager.html)). These managed objectsare the service interfaces for the virtual infrastructure management components. (用来管理 vsphere 系统的)



* configurations service：绿色
* administrative/management services：蓝色
* inventory：橘色
* entry point to a vCenter Server or ESX(i) host ：红色

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fptn7ok79ej30oo0cqwfj.jpg)



#### Managed object  操作

每个 Managed object 可以包含 properties 和 operation。

* Properties 在 server 端对映 一个 managed object 的 attributes.
* Operation 在 server 端对映一个 managed object 的 method.

在 client 端指定一个 managed object 的 operation 和 properties，通过 SOAP 定义的 xml 使用 HTTP 发送给 server 端，相当于调用了server 端对映 managed object 的方法。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fptnc7fuwtj30eg06at8w.jpg)

### MOR [ManagedObjectReference](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vmodl.ManagedObjectReference.html)

a [dataobject](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/do-types-landing.html) type that provides a reference to server-side objects.

MOR  是 server side  object 在 client 端的一个引用。通过一个 mor 可以得到对映的 managed object。

Client 端可以通过 reference 得到一个 managed object 的 properties 和 operation。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fptneblp18j307002bmwz.jpg)

### Data Object

Data Object  规范对接了 client 和 server 端传输的数据格式。

当 client 端对一个 server 端的 Managed Object 进行操作时，client 端会对 managed object 的操作 operation 和 properties 发送给 server 端。

这些 operation 和 properties 都是 Data Object。

Data Object 的类型可以有很多：

* Managed  object (代表 server side 的一个 managed object)
* String
* Integer
* Enumeration
* List
* …

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fptnkmpmu1j30mw08fmxk.jpg)

### [ServiceInstance](https://code.vmware.com/apis/42/vsphere)

The [ServiceInstance](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ServiceInstance.html) managed object is thesingleton root object of the inventory on both VCenter servers and servers runningon standalone host agents. The server creates the [ServiceInstance](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ServiceInstance.html) automatically, andalso automatically creates the various manager entities that provide servicesin the virtual environment.

ServiceInstance 是一个单例， VCenter 或者 ESXI web service 服务起来时自动创建，ServiceInstance 自动提供其他 managed object 入口。

A vSphere API clientapplication begins by connecting to a server and obtaining a reference to the [ServiceInstance](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ServiceInstance.html). The client can then usethe [RetrieveServiceContent](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ServiceInstance.html) method to gain access to the various vSphere manager entities and to theroot folder of the inventory.

Refer： https://172.28.8.247/mob

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fptnlnq8mcj30n80f03yx.jpg)

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fptnm4idpaj30k70f03zf.jpg)

#### Inventory 中可以添加哪些类型的 managed object

When you create managed objects, the server adds them to the inventory. The inventory of managed objects includes instances the following object types:

[ServiceInstance](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ServiceInstance.html) -- Root of theinventory; created by vSphere.

[Datacenter](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.Datacenter.html) -- A container thatrepresents a virtual data center. It contains hosts, network entities, virtualmachines and virtual applications, and datastores.

[Folder](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.Folder.html) -- A container usedfor hierarchical organization of the inventory.

[VirtualMachine](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.VirtualMachine.html) -- A virtual machine.

[VirtualApp](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.VirtualApp.html) -- A virtualapplication.

[ComputeResource](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ComputeResource.html) -- A compute resource(either a cluster or a stand-alone host).

[ResourcePool](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.ResourcePool.html) -- A subset ofresources provided by a ComputeResource.

[HostSystem](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.HostSystem.html) -- A single host (ESXServer or VMware Server).

[Network](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.Network.html) -- A networkavailable to either hosts or virtual machines.

[DistributedVirtualSwitch](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.DistributedVirtualSwitch.html) -- A distributedvirtual switch.

[DistributedVirtualPortgroup](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.dvs.DistributedVirtualPortgroup.html) -- A distributedvirtual port group.

[Datastore](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/98d63b35-d822-47fe-a87a-ddefd469df06/8212891f-77f8-4d27-ab3b-9e2fa52e5355/doc/vim.Datastore.html) --Platform-independent, host-independent storage for virtual machine files.

## Openstack Nova + oslo_vmware

### Oslo_vmware：

VCenter API  的 SDK (python language 基于 python suds module), 主要负责建立 vmware API session， 解析 wsdl 文件和managed object，使用 morf 获取 managed object 的 property，发送更改 managed object property 。

nova.virt.vmwareapi:

基于 oslo_vmware，主要负责获取 vm 相关信息，组装 method 参数和调用 vm 操作相关的 method。

**oslo_vmware 在用户层面的使用**：

1. 建立 session 连接
2. 用 session 和 VCenter 交互

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fptnp10t1tj30fv07pwep.jpg)

refer： [https](https://docs.openstack.org/oslo.vmware/latest/user/usage.html)[://](https://docs.openstack.org/oslo.vmware/latest/user/usage.html)[docs.openstack.org/oslo.vmware/latest/user/usage.html](https://docs.openstack.org/oslo.vmware/latest/user/usage.html)

**Nova vmwareapi 在用户层面的使用与 oslo_vmware 一样**：

1. 建立 session 连接

2. 用 session 和 VCenter 交互，交互内容被细化为 vm 相关的操作。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fptnqoh0npj30f207rjri.jpg)

**相关类的 MUL 图**：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fptnr57hzqj30hq0f0jrs.jpg)

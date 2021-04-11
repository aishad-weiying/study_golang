# OpenStack介绍
openstack 是（infrastructure as a service，基础设置即服务）IAAS 架构的实现，OpenStack 是一个由 NASA（美国国家航空航天局）和 Rackspace 合作研发并发起的，以 Apache 许可证授权的自由软件和开放源代码项目。

OpenStack 是一个开源的云计算管理平台项目，由几个主要的组件组合起来完成具体工作。
OpenStack 支持几乎所有类型的云环境，项目目标是提供实施简单、可大规模扩展、丰富、标准统一的云计算管理平台。OpenStack 通过各种互补的服务提供了基础设施即服务（IaaS）的解决方案，每个服务提供 API 以进行集成。


OpenStack 是一个旨在为公共及私有云的建设与管理提供软件的开源项目。它的社区拥有超过130家企业及1350位开发者，这些机构与个人都将OpenStack作为基础设施即服务（IaaS）资源的通用前端。OpenStack 项目的首要任务是简化云的部署过程并为其带来良好的可扩展性。本文希望通过提供必要的指导信息，帮助大家利用 OpenStack 前端来设置及管理自己的公共云或私有云。

OpenStack 云计算平台，帮助服务商和企业内部实现类似于 Amazon EC2 和 S3 的云基础架构服务(Infrastructure as a Service, IaaS)。OpenStack 包含两个主要模块：Nova和 Swift，前者是 NASA 开发的虚拟服务器部署和业务计算模块；后者是 Rackspace 开发的分布式云存储模块，两者可以一起用，也可以分开单独用。OpenStack 除了有 Rackspace 和 NASA 的大力支持外，还有包括 Dell、Citrix、 Cisco、 Canonical 等重量级公司的贡献和支持，发展速度非常快，有取代另一个业界领先开源云平台 Eucalyptus 的态势。

# OpenStack的历史版本
当前最新的是S 版，每半年更新一次新版本，已经从 A-P，从 G 版以后国内的使用用户越来越多，OpenStack 遵循一个一年两次的开发及发布的周期，在春末提供一个发布，秋季第二个版本。使用版本的代号按按字母顺序排列，目前，Stein 版本是最新稳定版本。

Alpha：是内部测试版,一般不向外部发布，通常只在软件开发者内部交流，该版本软件的 Bug较多，需要继续修改。

Dev：在软件开发中多用于开发软件的代号，相比于 beta 版本，dev 版本可能出现的更早，甚至还没有发布。这也就意味着，dev 版本的软件通常比 beta 版本的软件更不稳定

Beta：也是测试版，这个阶段的版本会一直加入新的功能。在 Alpha 版之后推出。

RC：(Release Candidate) 就是发行候选版本,RC 版不会再加入新的功能了，主要着重于除
错。

GA:General Availability,正式发布的版本。

Release:该版本意味“最终版本”，在前面版本的一系列测试版之后，终归会有一个正式版本，
是最终交付用户使用的一个版本。该版本有时也称为标准版。

> 由于目前企业内常用的版本为Ocata版本，所以下面我将针对Ocata版本进行介绍和搭建
> 官方文档：https://docs.openstack.org/ocata/zh_CN/install-guide-rdo/index.html

## OpenStack各组件的功能
OpenStack通过Noav调用KVM/XEN/VMWARE等虚拟化技术创建的虚拟机，及OpenStack是一个管理平台框架，支持众多的虚拟化管理，cinder存储支持GlusterFS、ISCSI、MFS等存储技术给虚拟机使用，及OpenStack不会绑定一个应用，而是兼容众多的相关技术
[table id=1 /]
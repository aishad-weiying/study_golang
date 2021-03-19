# kubernetes 概述
尽管 kubernetes 面世才不过几年,但是,其已经称为容器编排领域事实上的标准,也行在未来的几年中 kubernetes 无论在私有云,公有云或者混合云,kubernetes 都将成为一个任何应用,任何环境提供的容器管理框架

kubernetes v1.0 版本与 2015 年 7 月 21 日发布,紧随其后,Google 与 Linux 基金会组建了云原生计算基金会(CNCF)

#### 云原生的意义

1. 基础设施的一致性和可靠性

- 容器镜像
- 自包含
- 可漂移

2. 简单可预测的部署与运维

- 自描述,自运维
- 流程自动化
- 容器水平扩展
- 可快速复制的管控系统与支撑组件

# kubernetes 的特性
kubernetes 是一种在一组主机上运行和协同容器化应用程序的系统,旨在提供可预测性,可扩展性与高可用性的方法来完全管理容器化应用程序和服务的生命周期的平台

1. 自动装箱(容器的调度)
构建于容器之上,基于资源依赖及其它约束,自动完成容器的部署,并不会影响其可用性,并通过调度机制混合关键性应用和非关键型应用的工作负载与同一节点,以提升资源的利用率

2. 自我修复(自愈)
支持容器故障后自动重启,节点故障后重新调度新的容器,以及其他可用节点的健康状态检查失败后关闭容器并重新创建等自我修复的功能

3. 水平扩展
支持通过简单的系统命令或UI 手动的水平扩展,以及基于 CPU 等资源的使用率自动的水平扩展机制

4. 服务发现和负载均衡
kubernetes 通过其附加的组件之一 KubeDNS(或者 CoreDNS)为系统内置服务发现功能,它会为每个 service 配置 DNS 名称,并允许此集群内的客户端直接使用这个 DNS 名称发起访问请求,而 service 通过 iptables 或者 IPVS 内建了负载均衡的机制

5. 自动发布和回滚
kubernetes 支持应用程序的灰度发布,支持发生故障后的回滚

6. 密钥和配置管理

7. 存储编排
kubernetes 支持 pod 对象按照需要自动挂载不同类型的存储系统,这包括节点本地存储,公有云服务上的云存储,以及网络存储

8. 批量处理执行

## kubernetes 的术语

1. master
master 节点是整个集群的网关和中枢,负责为用户和客户端暴露 API,跟踪其他服务器的健康状态,以最优的方式调度工作负载,以及编排其他组件执行通信任务等功能,是用户与客户端之间的核心联络点,可以部署过个 master 节点来实现高可用

2. node 节点
node 节点是 kubernetes 集群的工作节点,负载接收来自 master 节点的工作指令,响应的创建或者删除 pod 对象,以及调整网络规划,合理的路由和流量转发等

在用户要将应用程序部署在上面的时候,master 节点会根据调度算法,为其自动的调度到指定的 node 节点上运行

3. pod
kubernetes 中最小的调度单元,kubernetes 并不是直接运行容器,而是是用一个抽象的资源对象来封装一个或者多个容器,这个抽象的资源对象就是 pod

同一个 pod 中的容器共享网络民初空间可存储资源,这些容器可以使用 lo 端口直接通信,但是又在 mount,user 或者 PID 等名称空间进行了隔离

3. label
label 是将资源进行分类的标识符,资源标签其实就是一个键值型数据,标签宗旨在对象(如 Pod)辨识性的属性,一个对象可以拥有多个标签,同一个标签也可以同时被多个对象所拥有

4. selector
全称为 Label Selector(标签选择器),是一种根据 label 来过滤符合条件的资源对象的机制

用户通常使用标签对资源对象进行分类,而后使用标签选择器挑选出他们

5. pod 控制器
尽管 pod 是 k8s 的最小调度单位,但是用户并不会直接部署或者管理 pod 对象,而是借助控制器 controller 对其进行管理

用于工作负载的控制器是一种管理 pod 声明周期的资源抽象,它们是 kubernetes 上的一类对象,而非单个资源对象,包括 ReplicationController,ReplicaSet,Deployment,StatfulSet 和 job 等

6. servcie
建立在一组 pod 上的资源抽象,它们通过标签选择器选定一组 pod 对象,为这些 pod 提供统一的访问入口(ip 或者 DNS 名称),访问请求到达 service 的时候,会自动的被均衡到 pod 上

service 实际上就是一个四层的代理服务

7. 存储卷
volume 是独立于容器文件系统之外的存储空间,常用如扩展容器的存储空间或者提供持久化的存储

- 临时卷和本地卷:都位于node 本地,一旦 pod 被调度到其他的 node ,这种类型的存储卷就无法访问到

- 网络卷:通过网络共享给 pod 使用,一般用于访问共享的数据,或者持久化数据

8. name 和 namespace
name 是 k8s 集群中资源对象的标识符,它的作用域通常是名称空间 namespace,在同一个 namespace 中,同一类型的资源对象的名称必须具有唯一性,名称空间通常用户实现租户或者项目的资源隔离,从而实现逻辑分组

创建的 pod 和 service 等资源对象的时候,都属于名称空间级别,未指定的时候,都属于默认的名称空间 default

9. annotation(注释)
常用于各种非标识型元数据(metadata)附加到对象上,主要目的是方便工具或者用户的阅读及查找等

10. Ingress
类似 service 为 pod 提供外部访问的通道

## k8s 的集群组件

#### Master 组件

1. API Server
复制输出 RESTful 风格的 kubernetes API,它是发往所有集群的 REST 操作命令的接入点,并负责接收,校验并相应所有的请求,结果状态被持久的保存在 ectd 中

API Server 是整个集群的网关

2. etcd
是 CoreOS 基于 Raft 协议开发的分布式键值存储,可用于服务发现,共享配置以及一致性保障,etcd 是独立的服务组件,并不隶属于 kubernetes 集群自身,一般是以集群的方式部署

3. controller-manager
集群级别的大多数功能都是由几个被称为控制器的进程执行的,这几个进程被集成在 kube-controller-manager 守护进程中,由控制器完成的功能主要包括生命周期功能和 API 业务逻辑

- 生命周期功能: 包括 namespace 创建和生命周期,event 垃圾回收,pod 终止相关的垃圾回收,级联垃圾回收以及 node 垃圾回收等

- API 业务逻辑:例如 replicaSet 执行的 pod 扩展等

4. schedule
调度器,API server 确认 pod 对象创建的请求之后,变需要由 schedule 根据集群内各个节点的资源可用状态,以及要运行的容器的资源需求做出调度策略,kubernetes 也支持用户自定义调度器

#### node 组件

1. 核心代理程序 kubelet
是运行在工作节点上的守护进程,从 API server 接收到关于 pod对象的配置信息后,确保满足它的期望状态,kubelet 会在 API server 上注册当前工作节点,定期向 master 汇报节点的资源使用情况,并通过 cAdvisor 监控容器和节点的资源占用情况

2. container runtime
每个 node 都负责提供一个容器运行时环境,它负责下载镜像并运行容器,kubelet 并未固定了链接至某容器的运行时环境,而是以插件的方式载入配置的配置,这种方式清晰的定义了各个组件的边界,目前 kubernetes 支持的容器运行时环境至少包括:docker,RKT,cri-o 和 fraki 等

3. kube-proxy
每个节点都需要运行一个守护进程kube-proxy,它能够按需要为 service 资源对象生成iptables 或者 IPVS 规则,从而捕获访问 service 的 ClusterIP 的流量并为其正确的转发至后端的 pod 对象

##### 核心组件

1. CoreDNS
在 kubernetes 集群中调度运行提供 DNS 服务的 pod,统一集群中的其他 pod 可以使用此 DNS 服务解决主机名

kubernetes 在 1.11 版本之后从 kubeDNS 改用 CoreDNS

2. dashboard
kubernetes 集群的全部功能都要基于 Web UI,来管理集群中的应用程序或者集群自身

3. Heapster
容器和节点的性能监控与分析系统,它收集并解析多种指标数据,如资源利用率,生命周期等,从 1.1.2 版本起,其功能已然由 Prometheus 结合其他的组件取代

4. Ingress Controller

## kubernetes 的网络模型
kubernetes 中主要存在四种类型的通信

1. 同一个 pod 中的不同容器之间的通信

2. 各个 pod 彼此间的通信

3. pod 与 service 的通信

4. 集群外部流量同 service 的通信

kubernetes 为 pod 和 service 资源对象分别使用了各自的专用网络,pod 的网络由 kubernetes 的网络插件配置实现,而 service 的网络由 kubernetes 集群指定,kubernetes 的网络模型需要外部插件实现,而实现网络模型的时候,任何插件都需要满足以下条件:

- 所有 pod 间均可以不经 nat 机制直接通信

- 所有节点均可以不经过 nat 直接与所有的容器通信

- 所有的 pod 对象都在同一平面网络中,而且可以使用 pod 自身的地址直接通信

> pod 的 ip 地址实际上存储在与某个网卡(可以是虚拟设备)上,而 service 的地址是一个虚拟的 ip 地址,没有任何的网络接口配置此地址,它是由 kube-proxy 借助 iptables 或者 IPVS 规则重新定向到本地的端口再将其调度至后端的 pod 对象

总体来说一个 kubernetes 集群中,应该包含了三个网络环境

- 各个主机(master, node, etcd)所属的网络

- 专用于 pod 资源对象的网络

- 专用于 service 对象的网络

## kubernetes 上的核心对象

1. pod
pod 资源对象是一种集合了一个到多个应用容器,存储资源,专用ip以及支撑容器运行的其他逻辑组件,不过 pod 对象中的进程均是运行在彼此隔离的容器中的,并于各个容器间共享两种关键资源

- 网络:每个 pod 对象都会被分配一个集群内专用的 ip 地址,同一个 pod 中的所有容器共享 pod 的网络和 UTS 名称空间,因此这些容器间通信的时候,可以基于本地回环接口 lo 进行,而 pod 与外部的其他组件通信需要借助 service 资源对象的 clusterIP 以及相应的端口完成

- 存储卷:用户可以为 pod 对象设置一组存储卷资源,这些资源可以共享给其内部的所有容器使用,从而完成数据的共享,存储卷还是保证在容器被重启或者重建之后,数据不会丢失

2. controller
Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自 动化修复流程，确保集群始终处于预期的工作状态。

3. service 
尽管 pod 拥有 ip 地址,但是很难保证 pod 在重启或者重建后 ip 地址不会发生变化,如果发生变化的话,就会导致服务不可访问,于是,service 资源被用于在被访问的 pod 对象中添加一个有这固定 ip 地址的中间层,客户端向此地址发起访问请求后,由相关的 service 资源调度不能改代理至后端的pod 对象

service Ip 是一种虚拟 ip,也称为 cluster ip,它专用于集群内通信,pod client 的访问请求可以 service 的 cluster ip 作为目标地址,但是集群网络属于私有网络地址,他们仅在集群内部可达,讲集群外部的访问流量引入集群内部的常用方法就是通过节点网络进行,实现方法就是通过工作节点的 ip 地址和端口(nodePort)接入请求并将其代理至后端的 pod 对象的 ip 以及其监听的 ip 地址

简答的来说 service 有三种类型

- 仅用于集群内部通信的 cluster ip

- 接入机器外部请求的 nodeport 类型,它工作于每个节点的主机 ip 之上

- LoadBalancer 类型,可以把集群外部的请求负载到多个 node 的主机的 nodePort 上
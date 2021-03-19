kubernetes  系统的 API server基于 HTTP/HTTPS 接收并相应客户端的请求,它提供了一种基于资源的RESTful风格的编程接口将集群的各个组件都抽象成为了标准的REST,并支持通过标准的HTTP方法以及JSON为数据序列化方案进行资源管理操作

## kubernetes 的资源对象
一句资源的主要功能作为分类标准,kubernetes 的API对象大致分为:

- 工作负载
- 发现和负载均衡
- 配置和存储
- 集群
- 元数据

### 工作负载类型的资源
pod 是工作负载型资源的基础资源,它负责运行容器,并为其解决环境依赖,例如:存储卷,配置信息和密钥数据等

应用程序主要分为无状态和有状态两种,ReplicationController , ReplicaSet和Deployment负责管理无状态的应用,StatefulSet 负责管理有状态的应用

- ReplicationController:用于确保每个pod的副本数能够在任何时候满足目标数量,在新的版本中被deployment和replicaSet取代
- ReplicaSet : 新一代的ReplicationController,与之不同的是replicaSet支持更多类型的选择器
- Deployment:用于管理无状态的持久化应用,例如HTTP服务器,它用于为pod和ReplicaSet提供声明式的更新,是构架在ReplicaSet之上的更高级的控制器
- StatefulSet:用于管理有状态的应用,如dashboard服务程序,与deployment不同的是,StatefulSet会为每个pod创建一个独有的持久性的标识,并会确保各个pod之间的顺序性
- DaemonSet:用于确保每个节点都运行了某pod的一个副本,新增的节点一样会被添加此类pod,在节点移除的时候,此类pod会被回收,往往用于运行寄去存储的守护进程或者日志收集进程
- Job:用于管理运行完成之后即可终止的应用

### 发现和负载均衡
pod资源可能会因为任何意外的故障而被重建,于是就需要有固定的可被访问的方式,此外,pod资源仅在集群内部可见,如果是集群外部的请求要访问pod内部的资源,同样需要将其暴露到集群之外,并且为访问同一种pod的流量进行负载均衡,在集群中service和Endpoint资源来提供此类功能

### 配置和存储
docker容器的分层联合挂载的方式,决定了不宜在容器那日不存储持久化的数据,于是就需要通过引入挂载外部存储卷的方式来完成数据的持久化,kubernetes为此设计了Volume资源来解决,它支持众多类型的存储设备或者存储系统,如ClusterFS,CEPH RDB和Flocker等

此外,基于镜像构建容器应用的时候,其配置信息与镜像制作的时候配入,从而为不同的环境定制配置就变得较为困难,docker通过环境变量作为解决方案,但是一旦植入的话,无法在运行时修改,ConfigMap 资源能够以环境变量或者存储卷的方式来接入到pod资源中,并且被多个同类的pod共享引用,从而实现一次修改,多处生效

### 集群级别的资源
pod,deployment,service和ConfigMap都属于名称空间级别的资源,下面的资源都属于集群级别的资源,只能又管理员进行操作

- Namespace : 资源对象名称的作用范围,绝大多数的资源都隶属于某个namespace,默认为default
- Node:kubernetes的工作接待你,其标识符在集群中必须唯一
- Role:名称空间级别的由规则组成的权限集合,可被RoleBinding引用
- ClusterRole:Cluster级别的由规则组成的权限集合
- RoleBinding:将role中的许可绑定在一个或者一组用户之上

### 元数据类型资源
此类资源对象用于为集群内部的其他资源配置其行为或者特性

## 资源在API中的组织形式

kubernetes 通常利用标准的RESTful术语来描述API概念

- 资源类型:是指在URL中使用的名称,如pod,namespace和servic等,其URL格式为"GROUP/Version/Resouce"
- 所有的资源类型都对应一个JSON表示格式,称为种类(kind),客户端创建对象必须以JSON格式提交信息
- 隶属于同一种类资源类型的对象,组成的列表称为集合(collection),如PodList
- 某种类型的单个实例称为资源或对象


kind 代表着资源对象所属的类型,又可以分为以下三种:

-  对象类:如namespace,deployment和service
-  列表类:通常表示同意类型的资源集合
-  简单类:常用语在对象上执行某种特殊操作

简单的来说,名称空间级别的每一个资源都能在API的URL 路径表示都可以简单的抽象为"/apis/group/version/namespaces/namespace/kind",如下:
```bash
root@master1:~# kubectl get --raw /apis/apps/v1/namespaces/default/deployments | jq .
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "selfLink": "/apis/apps/v1/namespaces/default/deployments",
    "resourceVersion": "71223"
  },
  "items": []
}

```

## 访问kubernetes 的REST API

以编程的方式来访问kubernetes的REST API 有助于了解其流式话的管理机制,最为常见的一种就是通过curl作为HTTP客户端,通过API server 在集群上操作资源对象模拟请求

但是,通过kubeadm部署的集群,默认仅支持HTTPS的访问接口,那么我们需要先通过kubectl proxy 在本地主机上为API server启动一个代理网关,由它支持HTTP进行通信

##### 在本地启动 API server 的代理网关

```bash
root@master1:~# kubectl proxy --port=8080
Starting to serve on 127.0.0.1:8080

```

#### 使用curl发送请求
```bash
root@master1:~# curl 127.0.0.1:8080/apis/apps/v1/namespaces/default/deployments
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "selfLink": "/apis/apps/v1/namespaces/default/deployments",
    "resourceVersion": "72673"
  },
  "items": []

```

#### 请求集群上的namespaceList 资源对象
```bash
root@master1:~# curl 127.0.0.1:8080/api/v1/namespaces
{
  "kind": "NamespaceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces",
    "resourceVersion": "72862"
  },
  "items": [
    {
      "metadata": {
        "name": "default",
        "selfLink": "/api/v1/namespaces/default",
        "uid": "a2571c9b-0e07-45cc-bd36-83decf3987c1",
        "resourceVersion": "146",
        "creationTimestamp": "2020-06-02T13:56:19Z"
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }
    },
....

# 还可以对上面删除的信息进行过滤,比如只显示所有的namespace名称

root@master1:~# curl 127.0.0.1:8080/api/v1/namespaces | jq .items[].metadata.name
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1762  100  1762    0     0   860k      0 --:--:-- --:--:-- --:--:--  860k
"default"
"kube-node-lease"
"kube-public"
"kube-system"

```

#### 查看指定名称空间资源对象的信息
```bash
root@master1:~# curl 127.0.0.1:8080/api/v1/namespaces/kube-system
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "kube-system",
    "selfLink": "/api/v1/namespaces/kube-system",
    "uid": "76f5ed40-2ca8-4ba5-afa5-ac0e8a42fc41",
    "resourceVersion": "5",
    "creationTimestamp": "2020-06-02T13:56:18Z"
  },
  "spec": {
    "finalizers": [
      "kubernetes"
    ]
  },
  "status": {
    "phase": "Active"
  }

```

## 对象类资源格式

上面的命令的相应中展示了kubernetes大多数资源对象的配置格式,它是一个json序列化的数据结构,主要具有kind,apiVersion,metadata,spec和status五个字段

- kind:代表着资源对象所属的类型
- apiVersion:所使用的api的版本
- metadata:为资源提供元数据信息
- spec:用于定义资源的用户期望状态
- status:记录活动对象的当前状态信息

### metadata嵌套字段
metadata用于描述对象的属性信息,其内嵌多个字段用于定义元数据

必选字段:
- namespace: 指定当前对象所属的名称空间,默认为default
- name:设定当前对象的名称,在其所属的名称空间的同意类型中必须唯一
- uid:当前对象的唯一标识符,通常由kubernetes自身维护

可选字段:
- labels:用于设定标识当前字段的标签
- annotations:非标识型键值数据,用来作为挑选条件,用户labels的补充
- resouceVersion:当前对象的内部版本标识符,用于让客户端确认对象是否发生变化
- generation:用于表示当前对象目标状态的代码
- creation Timestamp:当前对象创建的日期的时间戳
- deletion Timestamp:当前对象删除日期的时间戳

用户通过资源清单创建资源的时候,通常仅需要给出必要的字段,可选的字段可以根据情况选择,对于为给出的字段,则由一系列的finalizer自动填充

## 资源配置清单文件格式

首先可以根据资源类型查看帮助信息
```bash
root@master1:~# kubectl explain deployment
KIND:     Deployment
VERSION:  apps/v1

DESCRIPTION:
     Deployment enables declarative updates for Pods and ReplicaSets.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind	<string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata	<Object>
     Standard object metadata.

   spec	<Object>
     Specification of the desired behavior of the Deployment.

   status	<Object>
     Most recently observed status of the Deployment.

```

可以查看对应字段下面的二级字段和三级字段
```bash
root@master1:~# kubectl explain deployment.spec.replicas
KIND:     Deployment
VERSION:  apps/v1

FIELD:    replicas <integer>

DESCRIPTION:
     Number of desired pods. This is a pointer to distinguish between explicit
     zero and not specified. Defaults to 1.
```

### 根据现有的资源创建清单
虽然 kubectl explain 命令可以帮助我们创建资源清单,但是通过现有的模板进行修改才是我们常用的方式
```bash
root@master1:~# kubectl get deployment myapp -o yaml --export > deploy_demo.yaml

root@master1:~# cat deploy_demo.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: null
  generation: 1
  labels:
    run: myapp
  name: myapp
  selfLink: /apis/apps/v1/namespaces/default/deployments/myapp
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: myapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: myapp
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: myapp
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}

```

资源清单本质上是一个json或者yaml格式的文本文件,由资源对象的配置信息组成,支持使用git等进行版本控制,用户可以以资源清单为基础,在kubernetes集群上陈述式或声明式的对资源进行管理
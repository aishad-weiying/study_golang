## 管理pod对象的容器
一个pod对象中至少应该存在一个容器,因此,containers 字段是定义pod时的spec字段必须的嵌套字段,用于为pod指定创建的容器列表,进行容器配置是,name为必选字段,用于指定容器的名称,image为可选字段,以方便更高级别的管理类资源deployment等能覆盖此字段
```bash
name: 容器名称
image: 镜像名称
```

### 镜像机器获取策略
启动容器时,容器引擎首先与本地查找指定的镜像,如果不存在就需要到指定的镜像仓库下载镜像

kubernetes 系统支持用户自定义镜像文件的获取策略,例如在网络资源较为紧张的情况下,可以禁止从镜像仓库获取镜像文件,容器的 imagePullPolicy 字段用来指定镜像的获取策略

- Always: 镜像标签为 laster 或者镜像不存在的时候,会自动的到镜像仓库获取镜像文件
- IfNotPresent:仅当本地镜像缺失的时候才会到镜像仓库获取镜像文件
- Never:禁止从镜像仓库下载镜像,即仅使用本地镜像文件

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:laster
    imagePullPolicy: Always
```

上面的资源清单中,定义创建容器时使用的镜像为nginx:laster,那么每次创建资源的时候,总是会到镜像仓库获取最新版本的镜像文件

标签为laster的镜像,默认的策略就是Always,而其他标签的镜像默认为IfNotPresent

### 暴露端口
在docker中,使用默认网络的容器化应用需要通过NAT机制才能将其暴露到外部网络中,才能被其他节点上的容器客户端所访问,然而,kubernetes中的.pod的ip地址处于同一个网络平面中,无论是否为这些容器暴露的端口,都可以被其他节点的容器客户端访问

容器pods字段是值是一个列表,由一个到多个的端口对象组成,常用的嵌套字段有如下几个:
- containerPort:非必选字段,指定在pod对象的ip地址上暴露的容器端口
- name: 当前端口的名词,且在当前pod 中必须唯一,因此可以被service资源调用
- protocol:端口的相关协议,tcp或者udp,默认为tcp

```bash
apiVersion: v1
kind: Pod
metadata:
  name: pod-example
spec:
  contianers:
  - name: myapp
    image: ikubernetes/myapp:v1
    ports:
    - name: http
      contianerPort: 80
      protocol: TCP
```

上述资源清单中,定义了名为pod-example的pod资源,指定了要暴露在容器上的端口为TCP 80 端口,并将其命名为http

然而pod对象的ip地址仅在集群内可达,它们无法直接影响来自集群之外的请求,一个简单的解决方案就是通过其所在的Node节点的ip地址和端口将其暴露在外部

- hostPort: 主机端口,它将接收到的请求通过NAT机制转发至由contianerPort字段指定的容器端口
- hostIP:主机端口要绑定的ip地址,默认为0.0.0.0,考虑到pod具体被调度到集群中的哪个节点是不确定的,那么这个字段一般都是使用默认值

### 自定义运行的容器化应用
由docker 镜像启动容器时,运行的应用程序在相应的DockerFile中由entrypoint指令进行定义,传递给程序的参数则是通过cmd指令指定,entrypoint不存在的时候,可由cmd同时传递给程序及其参数

##### 获取指定镜像中定义的启动命令和参数

```bash
root@master1:~# docker inspect nginx:latest -f {{.Config.Cmd}}
[nginx -g daemon off;]


root@master1:~# docker inspect nginx:latest -f {{.Config.Entrypoint}}
[/docker-entrypoint.sh]
```

容器的command字段能够指定不同于镜像默认运行的应用程序,args可以给command传递参数
```bash
apiVersion: v1
kind: Pod
metadata:
  name: pod-example
spec:
  contianers:
  - name: myapp
    image: apline
    command: [ "/bin/sh" ]
    args: [ "-c","while ture;do sleep 30;done" ]
```

上述的资源清单中,将默认的启动命令/bin/sh改为了 /bin/sh -c while ture;do sleep 30;done

> 如果在资源清单中定义了command和args,那么会将镜像默认的启动命令和参数覆盖掉

### 环境变量
向pod中的容器对象传递环境变量数据的方式有两种:
- env
- envFrom

#### env
通过环境变量配置容器化的应用的时候,需要在容器的配置字段来嵌套使用env字段,它的值是一个由环境变量构成的列表,通常包含name和value

- name:环境变量的变量名称
- value: 传递给环境变量的值,通过`$(var_name)`引用.逃逸格式为`\$\$(var_name)`,默认为空

```bash
apiVersion: v1
kind: Pod
metadata:
  name: pod_example
spec:
  contianers:
  - name: myapp
    image: alpine:laster
    env:
    - name: HOSTNAME
      value: localhost.example.com
    - name: LOG_LEVEL
      vlaue: info
```

>  无论这些环境变量是否会被使用到,都会被直接注入到shell 环境中,可以使用printenv 查看注入的环境变量

```bash
root@master1:~# kubectl exec myapp-564fc884f-nmjcx printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=myapp-564fc884f-nmjcx
MYAPP_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.20.0.1
MYAPP_PORT=tcp://10.20.210.54:80
MYAPP_PORT_80_TCP=tcp://10.20.210.54:80
KUBERNETES_PORT=tcp://10.20.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
MYAPP_SERVICE_PORT=80
MYAPP_SERVICE_HOST=10.20.210.54
MYAPP_PORT_80_TCP_PROTO=tcp
MYAPP_PORT_80_TCP_ADDR=10.20.210.54
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.20.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_SERVICE_HOST=10.20.0.1
NGINX_VERSION=1.19.0
NJS_VERSION=0.4.1
PKG_RELEASE=1~buster
HOME=/root
```

#### 共享节点的网络名称空间

同一个pod对象的各个容器均运行于一个独立的,隔离的网络名称空间中,共享同一个网络协议栈以及相关的网络设备
```bash
root@master1:~# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE    IP                NODE      NOMINATED NODE   READINESS GATES
default       client-6dc66c487b-dszqn           1/1     Running   1          108m   10.10.2.32        node2     <none>           <none>
default       myapp-564fc884f-nmjcx             1/1     Running   1          24h    10.10.3.40        node3     <none>           <none>
default       myapp-564fc884f-q9x79             1/1     Running   1          24h    10.10.1.16        node1     <none>           <none>
kube-system   coredns-67c766df46-8wzbg          1/1     Running   11         8d     10.10.0.15        master1   <none>           <none>
kube-system   coredns-67c766df46-htrrd          1/1     Running   11         8d     10.10.0.14        master1   <none> 
```

但是系统中也有一些pod对象是直接运行在所在节点的网络名称空间的,来执行系统级别的管理任务,例如用kubeadm部署的kubernetes集群中的kube-proxy,kube-apiserver,kube-controller-manager,kube-flannel,kube-scheduler
```bash
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE    IP                NODE      NOMINATED NODE   READINESS GATES
kube-system   etcd-master1                      1/1     Running   5          8d     192.168.100.111   master1   <none>           <none>
kube-system   kube-apiserver-master1            1/1     Running   5          8d     192.168.100.111   master1   <none>           <none>
kube-system   kube-controller-manager-master1   1/1     Running   5          8d     192.168.100.111   master1   <none>           <none>
kube-system   kube-flannel-ds-amd64-bz2zm       1/1     Running   14         8d     192.168.100.116   node1     <none>           <none>
kube-system   kube-flannel-ds-amd64-jnhzs       1/1     Running   15         8d     192.168.100.117   node2     <none>           <none>
kube-system   kube-flannel-ds-amd64-rzcxx       1/1     Running   27         8d     192.168.100.118   node3     <none>           <none>
kube-system   kube-flannel-ds-amd64-xjmfg       1/1     Running   5          8d     192.168.100.111   master1   <none>           <none>
kube-system   kube-proxy-2c2kb                  1/1     Running   12         8d     192.168.100.116   node1     <none>           <none>
kube-system   kube-proxy-b7wq8                  1/1     Running   13         8d     192.168.100.117   node2     <none>           <none>
kube-system   kube-proxy-sdzl5                  1/1     Running   5          8d     192.168.100.111   master1   <none>           <none>
kube-system   kube-proxy-wm2x9                  1/1     Running   24         8d     192.168.100.118   node3     <none>           <none>
kube-system   kube-scheduler-master1            1/1     Running   5          8d     192.168.100.111   master1   <none>
```


##### 自建的pod使用节点的名称空间
需要在资源清单的容器字段使用 hostNetwork: true

```bash
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  contianers:
  - name: myapp
    image: alpine:laster
  hostNetwork: true
```

> 另外,在资源清单中,还可以分别使用 hostPID和hostIPC 来共享工作节点的PID和IPC名称空间

### 设置pod的安全上下文

pod的安全上下文用于设定pod或容器的权限和访问控制功能,其常用的属性包括下面几个方面

- 基于UID,GID控制访问对象的权限
- 以特权或非特权的方式运行
- 通过Linux Capabilities 为其提供部分特权
- 基于Seccomp过滤进程的系统调用
- 基于Selinux 的安全标签
- 是否能够进行权限升级

pod对象的安全上下文定义在spec.securityContext字段中,而容器的安全上下文则定义在spec.contianers.securityContext 字段中

```bash
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  contianers:
  - name: myapp
    image: alpine:laster
    command: ["/bin/sh","-c","sleep 360000"]
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: flase
```

> 以uid为1000的非特权用户运行容器,并禁止权限升级


## 标签
标签能够附加在kubernetes 的任何资源之上,简单的来说,标签就是键值类型的数据,能够在创建资源的时候指定,也能随时按需指定,而后即可使用标签选择器进行匹配从而完成资源的挑选

### 管理资源标签

创建资源的时候,可以直接在metadata中嵌套使用 lables 字段,来定义要附加的标签

```bash
root@master1:~# vim deploy_demo.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: pod-with-label
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1

# 应用这个清单
root@master1:~# kubectl apply -f deploy_demo.yaml
# 查看标签信息
root@master1:~# kubectl get pods --show-labels
NAME             READY   STATUS    RESTARTS   AGE    LABELS
pod-with-label   1/1     Running   0          117s   run=test
```

当标签较多的时候,使用-L keys1,keys2...可以指定显示有着特定键值的标签信息
```bash
root@master1:~# kubectl get pods -L run
NAME             READY   STATUS    RESTARTS   AGE     RUN
pod-with-label   1/1     Running   0          4m11s   test

```

1. 使用kubectl 命令为运行中的pod添加标签
```bash
root@master1:~# kubectl label pods/pod-with-label env=status
pod/pod-with-label labeled
root@master1:~# kubectl get pods --show-labels
NAME             READY   STATUS    RESTARTS   AGE     LABELS
pod-with-label   1/1     Running   0          6m49s   env=status,run=test
```

2. 对于已经存在了键值的标签,要使用 --overwrite 选项强制覆盖原有的键值
```bash
root@master1:~# kubectl label pods/pod-with-label env=stop --overwrite
pod/pod-with-label labeled
root@master1:~# kubectl get pods --show-labels
NAME             READY   STATUS    RESTARTS   AGE    LABELS
pod-with-label   1/1     Running   0          9m7s   env=stop,run=test
```

## 标签选择器

标签选择器用于表达标签的选择标准或者规则,目前有两种标签选择器,一个是基于等值关系,另一种是基于集合关系,使用标签选择器的时候,遵循以下规则

1. 使用多个标签选择器之间的逻辑关系为"与"
2. 使用空的标签选择器意味着每个资源都会被选中
3. 空的标签选择器将无法挑选出任何资源

- 等值关系
"="和"=="都表示相等,"!="表示不相等

- 集合关系
  - key in (value1,value2....): 指定的键的值在给定的值的列表中即可满足条件
  - key notin (value1,value2....): 指定的键的值不在给定的值的列表中即可满足条件
  - key : 存在此键名标签的资源
  - !key : 所有不存在此键名标签的资源

> 为了避免shell解析 "!",那么当使用感叹号的时候,需要使用单引号将此类表达式包裹起来

```bash
root@master1:~# kubectl get pods --all-namespaces -l 'k8s-app in (kube-dns,etcd)'
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-67c766df46-8wzbg   1/1     Running   14         35d
kube-system   coredns-67c766df46-htrrd   1/1     Running   14         35d
root@master1:~# 
```

此外,kubenetes的诸多资源对象必须以标签选择器的方式关联到pod对象,例如service,deployment和replicaSet 类型的资源等,它们在spec字段中嵌套使用嵌套的"selector"字段,通过"matchLabels"来指定标签选择器,有的甚至使用matchExpressions 来构造复杂的标签选择机制
```bash
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - (key: tier,operator:In,values:[cache])
```

  

  

  

  

  

  
## pod 控制器

master的各个组件中,API server 仅负责将资源存储到ectd中,并将其变动通知给各个相关的客户端程序,List-Watch 是kubernetes 实现的核心机制之一,在资源对象发生变动的时候,由APIserver负责将变动写入etcd中,并通过水平触发的机制,主动通知给相关的客户端程序,以确保各个客户端程序不错过任何一个事件

### pod 模板资源

PodTemplate 是kubernetes API 常用的资源类型,用于为控制器指定自动创建资源对象的时候所指定的配置信息,因为要内置于控制器中使用,所以pod 模板的配置不需要apiVersion和kind字段,但是其他的字段基本上一致

```bash
root@master1:~# kubectl explain replicaset.spec.template
KIND:     ReplicaSet
VERSION:  apps/v1

RESOURCE: template <Object>

DESCRIPTION:
     Template is the object that describes the pod that will be created if
     insufficient replicas are detected. More info:
     https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller#pod-template

     PodTemplateSpec describes the data a pod should have when created from a
     template

FIELDS:
   metadata	<Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec	<Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

```

下面我们来创建一个 pod 模块资源
```bash
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    dev: test
  labels:
    dev: test
    ower: weiying
  name: replica-test
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-tomcat
  template:
    metadata:
      labels:
        app: myapp-tomcat
    spec:
      containers:
      - name: test
        image: ikubernetes/myapp:v1
        ports:
        - name: tomcat
          containerPort: 808
```

## replicaSet 控制器

ReplicaSet 简称 RS,是 pod 控制器类型的一种实现,用于确保 pod 对象副本数在任一时刻都能精确的满足期望的数量,replicaSet 控制器资源启动后,会查找集群中匹配器标签选择器的pod资源对象,当前活动的对象的数据量与期望值是否一致,多则删除,少就通过pod模板创建

作用

- 确保 pod 资源对象的数量与期望值相匹配
- 保证pod的健康运行,如果检测到工作节点故障的话,会在可用的工作节点按照 pod 模板重新创建出 pod 资源对象
- 弹性伸缩

#### 创建replicaSet

```bash
# 可以查看帮助信息
root@master1:~# kubectl explain replicaset
```

RS 对象的资源清单也分为 apiVersion,kind,metadata接spec,而status字段为只读字段,它的spec字段一般套用一下选项

- replicas: 期望的副本数量
- selector:当前控制器匹配pod对象副本的标签选择器,支持一下两张机制
  - matchLabels: 通过直接指定键值对来指定标签选择器 
  -  matchExpressions: 基于表达式来指定标签选择器列表
- template: 用于补足 pod 资源对象的模板
- minReadySeconds: 对于新创建的pod ,在启动多久之后,如果其容器为发生崩溃等异常情况即被视为就绪

```bash
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    dev: test
  labels:
    dev: test
    ower: weiying
  name: replica-test
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-tomcat
  template:
    metadata:
      labels:
        app: myapp-tomcat
    spec:
      containers:
      - name: test
        image: ikubernetes/myapp:v1
        ports:
        - name: tomcat
          containerPort: 8080
```

通过上面模板创建的replicaSet之后,如果机器中有标签为 app: myapp-tomcat 的pod对象的话,这些pod就会归属到这个replicaSet的管控之下,如果没有的话,那么replicaSet会自动根据期望的数量和模板创建出标签带有app: myapp-tomcat的pod资源对象
```bash
root@master1:~# kubectl get replicasets -o wide 
NAME           DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replica-test   3         3         3       28m   test         ikubernetes/myapp:v2   app=myapp-tomcat
```

  1. 当pod的数量缺少的时候,replicaSet控制器对象会自动的创建出期望的数量
  2. 当pod数量过多的时候,replicaSet也会自动删除pod

##### 查看pod资源对象的变动的相关事件

```bash
root@master1:~# kubectl describe replicasets/replica-test
Name:         replica-test
Namespace:    default
Selector:     app=myapp-tomcat
Labels:       dev=test
              ower=weiying
Annotations:  dev: test
              kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{"dev":"test"},"labels":{"dev":"test","ower":"weiying"},"name":"repl...
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=myapp-tomcat
  Containers:
   test:
    Image:        ikubernetes/myapp:v2
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  37m    replicaset-controller  Created pod: replica-test-gsqnf
  Normal  SuccessfulCreate  37m    replicaset-controller  Created pod: replica-test-kw8cb
  Normal  SuccessfulCreate  37m    replicaset-controller  Created pod: replica-test-4nslh
  Normal  SuccessfulCreate  6m16s  replicaset-controller  Created pod: replica-test-x7vr8
  Normal  SuccessfulDelete  12s    replicaset-controller  Deleted pod: pod-with-label

```

通过打印出replicaSet对象的详细状态,我们在event字段就能看出pod字段对象的创建于删除

### 更新 replicaSet 控制器

  在日常中,代码总是会更新的,我们可能总是会去修改pod资源使用的镜像,或者是动态的伸缩pod的数量,这个时候就需要对replicaSet对象进行修改

##### 更改pod 的模板:相当于升级应用

更新pod模板中的镜像
```bash
    template:
    metadata:
      labels:
        app: myapp-tomcat
    spec:
      containers:
      - name: test
        image: ikubernetes/myapp:v2
        ports:
        - name: tomcat
          containerPort: 8080

```

将pod 模板中的镜像从v1版本升级到v2版本,更改之后使用一下命令进行更新replicaSet
```bash
kubectl apply -f replicaset.yaml
# 或者
kubectl replace -f replicaset.yaml
```

不过对于正在运行的pod来说,使用的镜像仍然是更新之前的镜像
```bash
 kubectl get pods -l app=myapp-tomcat -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name                 Image
replica-test-gsqnf   ikubernetes/myapp:v1
replica-test-kw8cb   ikubernetes/myapp:v1
replica-test-x7vr8   ikubernetes/myapp:v2
```

对于这些仍旧使用的是老版本的镜像的pod来说,要手动将其删除重建

- 一次性删除所有
- 分批次删除,先删除一部分,等待pod副本数补充上来之后再删除其他的

##### 扩容和缩容

1. 直接指定副本数量
```bash
kubectl scale replicasets/replica-test --replicas=5
```

2. 在现有的副本数量符合一定的标准的时候,才执行扩容或者缩容
```bash
root@master1:~# kubectl scale replicasets/replica-test --current-replicas=4 --replicas=3
error: Expected replicas to be 4, was 5

# 使用--current-replicas=n 执行需要满足的副本数量
# 因为指定的是4个,但是实际上有5个,那么不会进行缩容的操作
```

3. 删除replicaSet
```bash
kubectl delete replicasets/replcase-name
# 执行上面的命令的时候,删除replicaSet的时候会将其管控的pod一并删除
# 可以使用 --cascade=false 取消级联
kubectl delelte replicasets/replica-name --cascade=false
```
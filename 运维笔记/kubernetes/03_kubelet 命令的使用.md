# kubectl 命令的使用
kubernetes api 是集群管理其各种资源的统一入口,它提供了一个 RESTful 风格的 CRUD(create,read,update和delete)接口,用于查询和修改集群的状态,并将集群的状态存储在 etcd 中

kubectl 的核心功能就是通过 API server 操作 kubernetes 的各种资源对象

## 创建资源对象

- 创建 deployment 控制器对象
创建名为 nginx-deploy 的 pod 对象
```bash
kubectl run nginx-deploy --image=nginx:1.12 --replicas=2

kubectl run 的其它参数:
	--replicas=n: 指定创建的pod的副本数量,这个选项在 1.18版本之后不生效
	--port=n :指明容器要暴露的端口
	--dry-run :这个选项表示,仅在测试运行后就退出,1.18版本之后为 --dry-run=client
	-l,--label: 为创建的 pod 对象指定标签
	--record=true/false : 是否就爱那个当前创建的命令保存至对象的注释中
	--save-config=true/false : 是否将当前对象的配置信息保存至注释信息中
	--restart=Never : 创建不受控制器管控的自主式 pod 对象
```

- 创建 service 资源对象
创建名为 nginx-svc 的 service 资源对象
```bash
kubectl expose deployment/nginx --name=nginx-svc --port=80
```

用户也可以根据事先定义好的资源清单创建资源对象,例如事先定义好了 deployment 对象的 nginx-deploy 文件 和 service 对象的 nginx-svc 的文件
```bash
kubectl create -f nginx-deploy.yaml -f nginx-svc.yaml
```

甚至还可以将创建交给 kubectl 执行定义,用户只需要声明期望状态,这种方式称为声明式对象配置
```bash
kubectl apply -f nginx-deploy.yaml -f nginx-svc.yaml
```

## 查看资源对象


1. 查看所有的 namespace
kubernetes 系统中的大部分资源都隶属于某个 namespace 对象,缺省的名称空间为 default,若需要执行名称空间,需要在创建的时候使用 -n 或者 --namespace 来指定 namespace
```bash
root@master1:~# kubectl get namespaces
NAME              STATUS   AGE
default           Active   35h
kube-node-lease   Active   35h
kube-public       Active   35h
kube-system       Active   35h
```

2. 查看某个 namespace 中拥有指定标签的 pod 对象 
例如: 查看 kube-system 名称空间中有 k8s-app 标签的 pod 对象
```bash
root@master1:~# kubectl get pods -l k8s-app -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-546565776c-8prc9   1/1     Running   1          35h
coredns-546565776c-bpdqp   1/1     Running   1          35h
kube-proxy-68mcq           1/1     Running   1          35h
kube-proxy-g5d45           1/1     Running   1          35h
kube-proxy-hhklt           1/1     Running   1          24h
kube-proxy-nknf8           1/1     Running   1          35h
```

3. 查看 deployment 控制器对象
```bash
kubectl get deployments --all-namespaces
NAMESPACE     NAME      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   coredns   2/2     2            2           2d11h
```

上面输出的各个字段的说明;

- namespace : 所在的名称空间

- name : 资源对象的名称

- ready : 处于就绪态的副本数量

- UP-TO-DATE :更新到最新版本的pod对象的副本数量

- AVAILABLE: 当前处于可用状态的pod 的副本数量

- AGE : pod 的存在时长


4. 查看资源对象的详细信息
每个资源对象都包含这用户期望额状态(spec)和现有的状态(status)


```bash
kubectl get pods -l component=kube-apiserver -o yaml -n kube-system

# -o yaml 表示以 yaml 的格式输出,还可以指定为 json 或者 wide格式

kubectl describe pods -l component=kube-apiserver -n kube-system
# kubectl discribe 还能显示与当前对象相关的其他资源对象,如 event 或者 controller
```

上面的两个查看资源对象详细信息的命令,还支持查指定名称的资源对象的详细信息
```bash
kubectl get pods/kube-apiserver-master1 -n kube-system -o yaml
# 或者: kubectl get pods kube-apiserver-master1 -n kube-system -o yaml

kubectl describe/pods kube-apiserver-master1 -n kube-system
# 或者: kubectl describe pods kube-apiserver-master1 -n kube-system
```

```bash
root@master1:~# kubectl describe pods myapp 
Name:         myapp
Namespace:    default
Priority:     0
Node:         node3/192.168.100.118
Start Time:   Thu, 28 May 2020 22:27:05 +0800
Labels:       run=myapp
Annotations:  <none>
Status:       Running
IP:           10.10.3.2
IPs:
  IP:  10.10.3.2
Containers:
  myapp:
    Container ID:   docker://4003d7f0fdb26120052949c95f26180a207194d1505f23765fe4852b613a60fd
    Image:          ikubernetes/myapp:v1
    Image ID:       docker-pullable://ikubernetes/myapp@sha256:9c3dc30b5219788b2b8a4b065f548b922a34479577befb54b03330999d30d513
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 28 May 2020 22:27:30 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-6r672 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-6r672:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-6r672
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/myapp to node3
  Normal  Pulling    29m        kubelet, node3     Pulling image "ikubernetes/myapp:v1"
  Normal  Pulled     29m        kubelet, node3     Successfully pulled image "ikubernetes/myapp:v1"
  Normal  Created    29m        kubelet, node3     Created container myapp
  Normal  Started    29m        kubelet, node3     Started container myapp
```

> 上面的输出中一般来说 status和event是重点的关注字段

5. 显示容器中的日志信息
通常一个容器中仅会有一个进程及其子进程,此进程作为 PID 为 1 的进程接收并处理管理消息,同时将日志输出到终端中,而无需保存在文件中,因为查看容器的日志信息,一般要到其控制上查看
```bash
kubectl logs kube-apiserver-master1 -n kube-system
```

> 使用 -f 选项还能持续的查看日志的输出,如果pod 中有多个容器,可以使用 -c 指定容器查看


6. 在容器中执行命令
容器的隔离性,使得对其内部信息的获取变得不再直观,而当要获取容器内部的特性的时候,需要使用 kubectl exec 命令

kubectl exec POD_NAME COMMADN

```bash
kubectl exec kube-apiserver-master1 -n kube-system -- ls -l
```

> 注意: 如果 pod 中存在多个容器,需要使用 -c 选项指定容器再运行

进入到容器的方法
```bash
kubectl exec -it kube-apiserver-master1 -- sh
```

7. 删除资源对象
使命已经完成的或者存在错误的资源对象可以使用 kubectl delete 命令删除,但是,对于受控于控制器的对象来说,删除之后可能会重新创建

```bash
# 删除指定的pod
kubectl delete pods net-test2

# 删除指定标签的pod
kubectl delete pods -l app=monitor -n kube-system

# 删除指定名称空间中的所有pod
kubectl delete pods --all -n default
```

> 此外,可能有些资源对象,支持优雅删除机制,它们有着默认的删除宽限时间,不过可以在命令中私用 --grace-period 或者 --now 选项来覆盖默认的删除时间

## 部署 service 对象
为 myapp 创建的pod对象使用"NodePort"类型的服务暴露到集群之外
```bash
kubectl expose deployments/myapp --type="NodePort" --port=80 --name=mservice

	--type: 指定创建的service 类型
	--port:指定要暴露的容器的端口
```

查看创建的 service 对象
```bash
root@master1:~# kubectl get services -o wide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE     SELECTOR
kubernetes   ClusterIP   10.20.0.1     <none>        443/TCP        4d18h   <none>
mservice     NodePort    10.20.54.36   <none>        80:32673/TCP   5m17s   run=myapp
```

上面的输出中,其中 PORT字段表明,集群中各个工作节点会捕获发往本机的目标端口为32673的流量,将其代理至当前service对象的80端口,集群之外的用户,可以使用任意一个节点的这个32673端口,都能范文到service对象上的服务

##### 集群内部的机器访问

启动一个 busybox pod 用来测试
```bash
kubectl run client --image=busybox --restart=Never -it -- /bin/sh
```

进入到容器中,访问mservice这个service的80端口
```bash
root@master1:~# kubectl exec -it busybox-dd6bb7cf6-ds42z  sh

/ # wget -O - -q http://mservice.default:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

集群内部的主机,直接可以通过访问service对象的名称加上对应的端口,就能访问对于那个的deployment 总对应的服务,其中.default表示所在的namespace

> 创建service 的时候,service对象的名称以及ClusterIP会由CordDNS插件动态添加至名称解析库中
>

##### 查看 service 对象的详细描述信息
```bash
root@master1:~# kubectl describe services mservice
Name:                     mservice
Namespace:                default
Labels:                   run=myapp
Annotations:              <none>
Selector:                 run=myapp
Type:                     NodePort
IP:                       10.20.54.36
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32673/TCP
Endpoints:                10.10.1.8:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

- Selector: 表示当前可用的标签选择器,用来选择关联的pod 对象
- Port : 暴露的端口,即当前service 对象用于接收并提供请求的端口
- TargetPort : 容器中用于暴露的目标端口,由service port路由至此的端口
- NodePort : 当前service对象的 NodePort ,这个参数是否存在有效值,与TYPE类型有关
- Endpoints : 后端的ip地址与端口,即当前service对象代理的pod的ip和端口
- Session Affinity : 是否启用会话粘性
- External Traffic Policy: 外部流量的调度策略


## 使用kubectl命令实现扩容和缩减

在创建deployment对象的时候,使用--replicas可以指定该对象创建或者管理的pod的数量,使用 kubectl scale 命令可以实现对其的扩容和缩减的操作

1. 首先创建测试使用的deployment
```bash
root@master1:~# kubectl run scaling --image=alpine --replicas=3 -l scales=3 sleep 360000

root@master1:~# kubectl get pods -l scales=3 -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
scaling-64579b9d99-5gtcw   1/1     Running   0          38s   10.10.3.33   node3   <none>           <none>
scaling-64579b9d99-8qkqm   1/1     Running   0          38s   10.10.1.10   node1   <none>           <none>
scaling-64579b9d99-r6z98   1/1     Running   0          38s   10.10.2.26   node2   <none>           <none>

```

2. 扩容刚刚创建的pod
```bash
root@master1:~# kubectl scale deployments/scaling --replicas=5
deployment.apps/scaling scaled

root@master1:~# kubectl get pods -o wide -l scales=3
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
scaling-64579b9d99-4qx2w   1/1     Running   0          34s     10.10.1.12   node1   <none>           <none>
scaling-64579b9d99-5gtcw   1/1     Running   0          6m50s   10.10.3.33   node3   <none>           <none>
scaling-64579b9d99-8qkqm   1/1     Running   0          6m50s   10.10.1.10   node1   <none>           <none>
scaling-64579b9d99-h8wn4   1/1     Running   0          34s     10.10.3.34   node3   <none>           <none>
scaling-64579b9d99-r6z98   1/1     Running   0          6m50s   10.10.2.26   node2   <none>           <none>

```

3. 缩减pod
```bash
root@master1:~# kubectl scale deployments/scaling --replicas=2
deployment.apps/scaling scaled

root@master1:~# kubectl get pods -o wide -l scales=3
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
scaling-64579b9d99-8qkqm   1/1     Running   0          8m24s   10.10.1.10   node1   <none>           <none>
scaling-64579b9d99-r6z98   1/1     Running   0          8m24s   10.10.2.26   node2   <none>           <none>

```

4. 查看刚才经过扩容和缩减的pod的详细信息
```bash
root@master1:~# kubectl describe deployments/scaling
Name:                   scaling
Namespace:              default
CreationTimestamp:      Sun, 07 Jun 2020 16:51:49 +0800
Labels:                 scales=3
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               scales=3
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable

```

从上面的输出中,我们可以看出pod副本数的变化,都有对应的记录

5. 扩容之前myapp pod,查看service的变化
```bash
root@master1:~# kubectl scale deployments/myapp --replicas=3
root@master1:~# kubectl describe services/mservice
Name:                     mservice
Namespace:                default
Labels:                   run=myapp
Annotations:              <none>
Selector:                 run=myapp
Type:                     NodePort
IP:                       10.20.54.36
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32673/TCP
Endpoints:                10.10.1.8:80,10.10.2.27:80,10.10.3.35:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

```

我们扩容的两个节点也会对应的用于service的Endpoints 
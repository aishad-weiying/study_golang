## Deployment 控制器
Deployment 是kubectl 控制器的又一种实现,它构建如replicaSet 控制器之上,可为 Pod 和 ReplicaSet 资源提供声明试更新

Deployment 控制器资源的主要职责同样是为了保证 Pod 资源的健康运行,其大部分功能均可通过调用 ReplicaSet 控制器来实现,同时还添加了部分特性

- 时间和状态查看: 必要时可以查看Deployment 对象升级的详细进度和状态
- 回滚:升级操作完成之后发现问题的时候,支使用回滚机制将应用返回到前一个或者由用户指定的历史记录中的版本之上
- 版本记录: 对Deployment 对象的每次操作都予以保存,以供可后续执行的回滚操作使用
- 暂停和启动: 对于每一次升级,都能够随时暂停和启动
- 多种自动更新方案: 一是 Recreate ,即重建更新机制,全面停止,删除旧的pod后,用新版本替代,另一个是RollingUpdate,即滚动升级机制,逐步替换旧有的pod更新至新的版本

### 创建Deployment
Deployment 是标准的kubernetes API 资源,它构建与replicaSet 资源之上,于是其spec字段中就嵌套使用了replicaSet 控制器支持的replicas,selector,template和minReadySeconds

```bash
cat deploy_demo.yaml 
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: myapp-deploy  ## 控制器的名称
spec:
  replicas: 3        ## replicas 的数量
  selector:          ## 当前replicasSet匹配pod对象副本的标签选择器
    matchLabels:
      app: myapp
  template:          ## 用于补足pod副本数的时候使用的模板
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        ports:
        - containerPort: 80
          name: http

```

应用上面Deployment资源
```bash
kubectl apply -f deploy_demo.yaml --record

# --record参数可以记录命令
```

查看创建的Deployment 资源
```bash
kubectl get deployments myapp-deploy
```

查看该Deployment 资源创建的pod资源
```bash
kubectl get pods -L app=myapp --show-labels
NAME                            READY   STATUS    RESTARTS   AGE   APP=MYAPP   LABELS
myapp-deploy-7866747958-dccnt   1/1     Running   0          13m               app=myapp,pod-template-hash=7866747958
myapp-deploy-7866747958-j7fvj   1/1     Running   0          13m               app=myapp,pod-template-hash=7866747958
myapp-deploy-7866747958-wds6l   1/1     Running   0          13m               app=myapp,pod-template-hash=7866747958
```

## Deployment的更新策略

Deployment 更新的时候,只需要用户在指定的pod模板中,更改需要变动的内容,例如使用的镜像等,剩下的步骤交给Deployment控制器自动完成即可

Deployment 控制器详细信息中包含了其更新策略的相关配置信息,如: myapp-deploy 控制器资源中的StrategyType和RollingUpdateStrateg字段等

```bash
 kubectl describe deployments myapp-deploy
Name:                   myapp-deploy
Namespace:              default
CreationTimestamp:      Tue, 11 Aug 2020 21:29:39 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"kubernetes.io/change-cause":"kubectl apply --filename=deploy_demo....
                        kubernetes.io/change-cause: kubectl apply --filename=deploy_demo.yaml --record=true
Selector:               app=myapp
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
  Containers:
   myapp:
    Image:        ikubernetes/myapp:v1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-deploy-7866747958 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set myapp-deploy-7866747958 to 3
```

Deployment 控制器支持两种更新策略: 滚动更新(rolling update)和重新创建(recreate),默认为滚动更新

- 重新创建(recreate):创建类似 replicaSet 中的第一种更新方式,即首先删除现有的pod对象,然后由控制器基于新的模板创建出新的pod资源对象,这种方式通常应用于新旧版本不兼容的情况下
- 滚动更新(rolling update): 是默认的更新策略,它在删除一部分旧的pod资源的同时,补充创建一部分新版本的pod对象进行应用升级,其优势是升级期间,容器中提供的服务不会中断,但是要求应用能够同时兼容旧版本和新版本


Deployment 控制器的滚动更新操作并非在同一个replicaSet 控制器对象下进行,而是分别置于两个replicaSet控制器中,旧的控制器的pod对象数量不断的减少的同时,新的控制器的pod对象的数量不断增加,直到旧的pod对象不再拥有pod对象,而新的控制器的副本数量变得完全符合期望的数量为止.

滚动更新时,应用升级期间还要确保可用的pod对象数量不低于某阈值,以确保可以持续处理客户端的请求,变动的方式和pod对象的数量范围将通过`deployment.spec.strategy.rollingUpdate.maxSurge`和`deployment.spec.strategy.rollingUpdate.maxUnavailable` 两个属性协同进行定义

- `maxSurge`: 指定在升级期间存在的总pod对象数量最多可以超出期望值的数量,其值可以是0或者正整数,也可以是一个期望值的百分比: 如果期望值是3,当前设置的值为1,那么表示pod的总数量不能超过4个
- `maxUnavailable`: 升级期间正常可以使用的pod的副本数量(包括新旧版本)最多不能低于期望值的个数,其值可以使0或者正整数,,也可以是一个期望值的百分比,默认值为1,该值意味着如果期望值为3,则升级期间至少要有两个pod对象正处于正常提供服务的状态

> `maxSurge`和`maxUnavailable` 属性的值不可以同时设置为0,否则pod对象的副本数量在符合用户期望的数量后,无法做出合理变动以进行滚动更新操作

配置的时候,用户还可以使用Deployment控制器`deployment.spec.minReadySeconds`属性来控制应用升级的速度,新旧更替过程中,新创建的pod对象一旦成功响应就绪探针就会被视为可用,而后即可开始下一轮的替换操作,而`deployment.spec.minReadySeconds`能够定义在新的pod创建多久之后才会被视为就绪状态,再次期间更新操作会被阻塞

因此,它可以用来让kubernetes 在每次创建出来pod资源后都要等上一段时间后才开始下一轮的更替,这个时间长度的理想值是等到pod对象中的应用已经可以接受并处理请求流量,实际上精心设计的等待时长和就绪性探测能让kubernetes系统规避一部分因程序BUG而导致的升级故障

Deployment 控制器也支持用户保留其滚动更新历史中旧的replicaSet对象版本,这赋予了控制器进行应用回滚的能力,用户可以按需回滚到指定的历史版本,控制器可以保存的历史版本数量由`deployment.spec.revisionHistoryLimit` 属性进行定义,当然也只有保存于reversion 历史中的replicaSet 版本可用于回滚

> 为了保存版本升级的历史,需要在创建Deployment 对象时的命令中使用 --record 选项用来保存历史

尽管滚动更新以节约资源著称,但是也存在一些劣势,直接改动现有的环境,会使系统引入不确定性风险,而且升级过程中出现问题后,执行回滚操作也会较为缓慢,有鉴于此,金丝雀部署可能是较为理想的实现方式,当然,如果不考虑系统资源的可用性,那么传统的蓝绿不是也是不错的选择


## 升级Deployment

修改pod模板相关的配置参数,便能完成Deployment控制器更新资源

为了方便观察更新操作,我们首先将之前的Deployment控制器设置minReadySeconds参数
```bash
kubectl patch deployments myapp-deploy -p '{"spec": {"minReadySeconds": 5}}'
```

patch 命令的补丁形式为JSON格式,以`-p`选项指定,如果想要更改容器使用的镜像也可以使用类似的命令
```bash
 kubectl patch deployments myapp-deploy -p '{"spec": {"containers": ["name": "myapp","image": "ikubernetes/myapp:v2"]}}'
 # 更改使用的镜像的话,还是使用下面的命令更加的方便
  kubectl set image
```

> 修改Deployment控制器的minReadySeconds,replicas和strategy等字段的值,并不会触发pod资源的更新操作,因此它们不属于模板的内嵌字段,对现有的pod不会产生任何的影响

下面我们来讲之前创建的Deployment的镜像改为v2版本
```bash

```

使用` kubectl rollout status`打印滚动更新过程中的状态信息
```bash
kubectl rollout status deployments myapp-deploy
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
deployment "myapp-deploy" successfully rolled out
```

或者使用`kubectl get deployments`监控其更新过程中pod对象的变化信息

```bash
kubectl get deployments myapp-deploy --watch
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   4/3     3            3           24h
myapp-deploy   4/3     3            4           24h
myapp-deploy   3/3     3            3           24h
```

滚动更新的时候,Deployment控制器会自动的创建一个新的replicasSet控制器对象来管控新版本的pod对象,升级完成之后,旧版的replicasSet会保存在历史版本中,但是此前管控的pod对象会被删除
```bash
kubectl get replicasets -l app=myapp
NAME                      DESIRED   CURRENT   READY   AGE
myapp-deploy-6ff4cd9c8c   3         3         3       7m24s
myapp-deploy-7866747958   0         0         0       24h
```

新创建的pod的名称也会使用新的replicasSet控制器的名称作为前缀
```bash
kubectl get pod -l app=myapp
NAME                            READY   STATUS    RESTARTS   AGE
myapp-deploy-6ff4cd9c8c-6r47k   1/1     Running   0          8m20s
myapp-deploy-6ff4cd9c8c-ngljs   1/1     Running   0          8m27s
myapp-deploy-6ff4cd9c8c-s8sdl   1/1     Running   0          8m14s
```

## 金丝雀发布

为了尽可能的降低对现有系统及其容器的影响,金丝雀发布过程中通常采用`先添加在删除`并且可用的pod资源对象总数不低于期望值的方式进行

为了能够清晰的了解过程,我们首先将Deployment的maxSurge属性的值设置为1,并将maxUnavailable的属性设置为0
```bash
kubectl patch deployments myapp-deploy -p '{"spec": {"strategy": {"rollingUpdate": {"maxSurge": 1,"maxUnavailable": 0}}}}'
```

下面我们来启动myapp-deploy控制器的更新过程,在修改相应容器的镜像版本之后立即暂停更新进度,它会在启动第一批新版本pod对象的创建操作后转为暂停的状态
```bash
kubectl set image deployments myapp-deploy myapp=ikubernetes/myapp:v3 && kubectl rollout pause deployments myapp-deploy

deployment.apps/myapp-deploy image updated
deployment.apps/myapp-deploy paused

```

> 这里之所以能够暂停成功,是因为之前我们将`minReadySeconds`的值设置为了5秒

通过其状态信息,我们可以看到,在创建一个新版本的pod资源之后滚动更新操作`暂停`
```bash
kubectl rollout status deployments myapp-deploy 
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
```

查看暂停之后的pod数量
```bash
kubectl get deployments myapp-deploy --watch
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   4/3     1            4           24h
```

> 从上面我们可以看出,旧版本的pod依旧在运行,新创建了1个pod,总数量为4个,不超过我们设置的数量

查看replicaSet的状态
```bash
kubectl get replicasets 
NAME                      DESIRED   CURRENT   READY   AGE
myapp-deploy-6ff4cd9c8c   3         3         3       40m
myapp-deploy-b6c4c7568    1         1         1       5m29s
myapp-deploy-7866747958   0         0         0       24h
```

> 我们可以看到同时存在新旧两个版本的replicaSet

通常在更新暂停之后,我们会将一些用户的访问信息引入一部分到新的pod上测试,测试没有问题的,可以使用`kubectl rollout resume`命令继续此前的更新操作
```bash
kubectl rollout resume deployments myapp-deploy ]

deployment.apps/myapp-deploy resumed
```

## 回滚Deployment控制器下的应用发布
若是因为各种原因导致滚动更新不能正常进行,则应该立即将应用回滚到之前的版本

查看现在的版本
```bash
kubectl describe deployments myapp-deploy 
Name:                   myapp-deploy
Namespace:              default
CreationTimestamp:      Tue, 11 Aug 2020 21:29:39 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 3
```

回滚到之前的版本
```bash
kubectl rollout undo deployments myapp-deploy 

deployment.apps/myapp-deploy rolled back
```
查看回滚的过程
```bash
kubectl rollout status deployments myapp-deploy 
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
deployment "myapp-deploy" successfully rolled out
```

查看所有的历史版本
```bash
kubectl rollout history deployments myapp-deploy 
deployment.apps/myapp-deploy 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=deploy_demo.yaml --record=true
3         kubectl apply --filename=deploy_demo.yaml --record=true
4         kubectl apply --filename=deploy_demo.yaml --record=tru
```

回滚到指定的版本
```bash
kubectl rollout undo deployments myapp-deploy --to-revision=1
```

> 回滚操作中,其中revision记录中的信息会发生变动,回滚操作会被当做一次滚动更新最佳到历史记录中,而被回滚的条目会被删除,如果此前的滚动更更新处于暂停的状态,需要先将pod模板该回到之前的版本,然后继续更新,否则会一直处于暂停状态,无法回滚

## 扩容和缩容
通过修改`spec.replicas`即可立即修改Deployment控制器中的pod资源的副本数量,他将试试作用于控制器并直接修改生效
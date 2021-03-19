## DaemonSet 控制器
DaemonSet 是pod控制器的又一种实现，用于在集群中的全部节点上同时运行一份指定的pod资源副本，后续加入到集群中的工作节点也会自动的创建一份对应的pod资源，当从集群中移除节点的时候，此类pod对象也会被自动回收，无需重建，管理员也可以使用节点选择器或者是节点标签在指点的节点的运行指定的pod对象

DaemonSet 是一种特殊的控制器,它有特定的应用场景,通常用于执行系统级操作任务的应用,如下:

- 运行集群存储的守护进程,如在各个节点上运行glusterd或者ceph
- 在各个节点上运行日志收集的守护进程, 如fluentd或者logstach
- 在各个节点上运行监控系统的各个代理守护进程,如Prometheus Node Export,colletcd,Datadog agent等

只有将Pod对象运行于固定的几个节点并且需要先于其它的pod启动的时候,才有必要使用DaemonSet控制器,否则应该使用Deployment 控制器

### 创建DaemonSet控制器

DaemonSet 控制器的资源清单的 spec 字段中嵌套使用的字段与pod控制器类似,但是不支持使用replicas,但是template是必选字段
```bash
# kubectl explain daemonset.spec
KIND:     DaemonSet
VERSION:  apps/v1

RESOURCE: spec <Object>

DESCRIPTION:
     The desired behavior of this daemon set. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

     DaemonSetSpec is the specification of a daemon set.

FIELDS:
   minReadySeconds	<integer>
     The minimum number of seconds for which a newly created DaemonSet pod
     should be ready without any of its container crashing, for it to be
     considered available. Defaults to 0 (pod will be considered available as
     soon as it is ready).

   revisionHistoryLimit	<integer>
     The number of old history to retain to allow rollback. This is a pointer to
     distinguish between explicit zero and not specified. Defaults to 10.

   selector	<Object> -required-
     A label query over pods that are managed by the daemon set. Must match in
     order to be controlled. It must match the pod template's labels. More info:
     https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors

   template	<Object> -required-
     An object that describes the pod that will be created. The DaemonSet will
     create exactly one copy of this pod on every node that matches the
     template's node selector (or on every node if no node selector is
     specified). More info:
     https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller#pod-template

   updateStrategy	<Object>
     An update strategy to replace existing DaemonSet pods with new pods.

```

下面是一份简单的资源清单文件,它将在每个节点上运行filebeat进程,用来收集容器的相关日志
```bash
vim filebeat-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: filebeat
  name: filebeat-ds
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
      name: filebeat
    spec:
      containers:
      - name: filebeat
        image: ikubernetes/filebeat:5.6.5-alpine
        env:
        - name: REDIS_HOST
          value: db.ilinux.io:6379
        - name: LOG_LEVEL
          value: info

```

通过上面的资源清单创建对应的DaemonSet 资源
```bash
# kubectl apply -f filebeat-ds.yaml 
daemonset.apps/filebeat-ds created
# 查看创建的pod资源
# kubectl get pods -l app=filebeat -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
filebeat-ds-k7nc4   1/1     Running   0          38s   10.10.3.55   node3   <none>           <none>
filebeat-ds-r4r42   1/1     Running   0          38s   10.10.2.48   node2   <none>           <none>
filebeat-ds-rpmfp   1/1     Running   0          38s   10.10.1.34   node1   <none>           <none>
# 查看创建的DaemonSet控制器
kubectl get daemonsets.apps -o wide -l app=filebeat
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS   IMAGES                              SELECTOR
filebeat-ds   3         3         3       3            3           <none>          2m43s   filebeat     ikubernetes/filebeat:5.6.5-alpine   app=filebeat
```

与其他的资源相同,DaemonSet控制器也可以通过`kubectl describe`来查看详细信息
```bash
 kubectl describe daemonsets.apps filebeat-ds 
Name:           filebeat-ds
Selector:       app=filebeat
Node-Selector:  <none>
Labels:         app=filebeat
Annotations:    deprecated.daemonset.template.generation: 1
                kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"apps/v1","kind":"DaemonSet","metadata":{"annotations":{},"labels":{"app":"filebeat"},"name":"filebeat-ds","namespace":"defa...
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=filebeat
  Containers:
   filebeat:
    Image:      ikubernetes/filebeat:5.6.5-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:
      REDIS_HOST:  db.ilinux.io:6379
      LOG_LEVEL:   info
    Mounts:        <none>
  Volumes:         <none>
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  4m5s  daemonset-controller  Created pod: filebeat-ds-k7nc4
  Normal  SuccessfulCreate  4m5s  daemonset-controller  Created pod: filebeat-ds-rpmfp
  Normal  SuccessfulCreate  4m5s  daemonset-controller  Created pod: filebeat-ds-r4r42

```

- Node-Selector: 为空表示需要运行在集群的每个节点上面
- Desired Number of Nodes Scheduled: 3 当前集群有三个节点,那么期望的pod副本数为3

> 对于只在特定的节点才运行的DaemonSet 控制器的pod资源来说,只需要在pod模板的spec字段中嵌套使用nodeSelector字段即可选择指定的节点`kubectl explain daemonset.spec.template.spec.nodeSelector`

### 更新 DaemonSet 对象
DaemonSet 更新的相关配置在`daemonset.spec.updateStrategy`嵌套字段中使用,目前支持滚动更新和删除时更新两种更新策略,滚动更新为默认的更新策略

例如: 更新上面DaemonSet控制器的pod版本
```bash
kubectl set image daemonsets filebeat-ds filebeat=ikubernetes/filebeat:5.6.6-alpine
```

查看更新的详细信息
```bash
# kubectl describe daemonsets.apps filebeat-ds
Name:           filebeat-ds
Selector:       app=filebeat
Node-Selector:  <none>
Labels:         app=filebeat
Annotations:    deprecated.daemonset.template.generation: 2
                kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"apps/v1","kind":"DaemonSet","metadata":{"annotations":{},"labels":{"app":"filebeat"},"name":"filebeat-ds","namespace":"defa...
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 2
Number of Nodes Misscheduled: 0
Pods Status:  2 Running / 1 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=filebeat
  Containers:
   filebeat:
    Image:      ikubernetes/filebeat:5.6.6-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:
      REDIS_HOST:  db.ilinux.io:6379
      LOG_LEVEL:   info
    Mounts:        <none>
  Volumes:         <none>
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  22m   daemonset-controller  Created pod: filebeat-ds-k7nc4
  Normal  SuccessfulCreate  22m   daemonset-controller  Created pod: filebeat-ds-rpmfp
  Normal  SuccessfulCreate  22m   daemonset-controller  Created pod: filebeat-ds-r4r42
  Normal  SuccessfulDelete  67s   daemonset-controller  Deleted pod: filebeat-ds-k7nc4
  Normal  SuccessfulCreate  61s   daemonset-controller  Created pod: filebeat-ds-v45kb
  Normal  SuccessfulDelete  49s   daemonset-controller  Deleted pod: filebeat-ds-rpmfp
  Normal  SuccessfulCreate  43s   daemonset-controller  Created pod: filebeat-ds-4tzkc
  Normal  SuccessfulDelete  12s   daemonset-controller  Deleted pod: filebeat-ds-r4r42
  Normal  SuccessfulCreate  2s    daemonset-controller  Created pod: filebeat-ds-xv2fq
```

从上面可以看出,默认的滚动更新策略是一次性删除节点上的pod资源,然后在将其创建爱出来,然后在开始操作另一个节点上的资源

```bash
kubectl rollout history daemonsets filebeat-ds
daemonset.apps/filebeat-ds 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

从上面我们可以看出,对DaemonSet控制器的操作也会记录在版本历史中,因此我们也可以对DaemonSet进行升级,回滚,金丝雀发布等等

## Job 控制器

与DaemonSet和Deployment控制器管理的守护进程类的服务不同的是,Job控制器用于调配pod对象运行一次性任务,容器中的进程在正常运行结束后,不会对其重启,而是将pod对象置于`Complete`状态,如果容器中的进程因为错误而终止,那么需要配置是否重启,未运行完成的pod对象因为其所在的节点故障而终止的会被重新调度

实践中,有的作业任务可能需要运行不止一次,用户可以配置它们以串行或者并行的方式运行,总结起来就是,这种类型的job控制器对象有两种:

- 单工作队列的串行式job:即以多个一次性的作业方式串行执行多次作业,直至满足期望的次数,这种方式也可以理解为并行度为1的作业执行方式,在某一个时刻进存在一个 pod资源对象
- 多工作队列的并行式job: 这种方式可以设置工作队列数,即作业数,每个队列仅负责运行一个作业,也可以用有限的工作队列运行较多的作业,即工作队列数少于总的作业数,相当于运行多个串行作业队列

job控制器常用于管理那些运行一段时间就可以完成的任务,例如计算或者备份操作

### 创建Job控制器

job 控制器的spec字段内嵌的template字段为必要字段,它的使用方式与Deployment等控制器并无太大的差别

job控制器会为其pod对象自动添加`job-name=JOB_NAME`和`controller-uid=UID`的标签,并使用标签选择器完成对controller-uid标签的关联

需要注意的是 Job 控制器位于API群组的 `batch/v1` 内
```bash
# vim job-example.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
spec:
  template:
    spec: 
      containers:
      - name: myjob
        image: alpine
        command: ["/bin/sh","-c","sleep 120"]
      restartPolicy: Never

```

> restartPolicy : 字段默认为 always

根据上面的清单创建Job控制器,查看相关的任务状态
```bash
# kubectl apply -f job-example.yaml 
job.batch/job-example created

#kubectl get jobs.batch job-example 
NAME          COMPLETIONS   DURATION   AGE
job-example   0/1           14s        14s
```

查看详细的信息
```bash
# kubectl describe jobs.batch job-example 
Name:           job-example
Namespace:      default
Selector:       controller-uid=d570b4b3-a14f-4a26-8800-db575920a7ba
Labels:         controller-uid=d570b4b3-a14f-4a26-8800-db575920a7ba
                job-name=job-example
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"job-example","namespace":"default"},"spec":{"template":{"spec":...
Parallelism:    1
Completions:    1
Start Time:     Sat, 15 Aug 2020 11:21:58 +0800
Completed At:   Sat, 15 Aug 2020 11:24:20 +0800
Duration:       2m22s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=d570b4b3-a14f-4a26-8800-db575920a7ba
           job-name=job-example
  Containers:
   myjob:
    Image:      alpine
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sh
      -c
      sleep 120
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  3m33s  job-controller  Created pod: job-example-w8cdh

```

运行两分钟后状态变为`completed`
```bash
# kubectl get pod job-example-w8cdh 
NAME                READY   STATUS      RESTARTS   AGE
job-example-w8cdh   0/1     Completed   0          2m49s

```

### 并行式job

将并行度属性`job.spec.parallelism`的值设置为1,并设置并行总任务数`job.spec.completions`能够让job控制器以串行的方式运行多个任务
```bash
# vim job-multi.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: job-multi
spec:
  completions: 5
  template:
    spec:
      containers:
      - name: myjob
        image: alpine
        command: ["/bin/sh","-c","sleep 20"]
      restartPolicy: OnFailur
```

将其创建并查看pod的变动
```bash
# kubectl get pods -l job-name=job-multi --watch
NAME              READY   STATUS    RESTARTS   AGE
job-multi-8fqkj   0/1     Pending   0          0s
job-multi-8fqkj   0/1     Pending   0          0s
job-multi-8fqkj   0/1     ContainerCreating   0          0s
job-multi-8fqkj   1/1     Running             0          16s
job-multi-8fqkj   0/1     Completed           0          37s
job-multi-fzdgk   0/1     Pending             0          0s
job-multi-fzdgk   0/1     Pending             0          0s
job-multi-fzdgk   0/1     ContainerCreating   0          0s
job-multi-fzdgk   1/1     Running             0          22s
job-multi-fzdgk   0/1     Completed           0          42s
job-multi-kt6l9   0/1     Pending             0          0s
job-multi-kt6l9   0/1     Pending             0          0s
job-multi-kt6l9   0/1     ContainerCreating   0          0s
job-multi-kt6l9   1/1     Running             0          18s
job-multi-kt6l9   0/1     Completed           0          38s
job-multi-nnf9q   0/1     Pending             0          0s
job-multi-nnf9q   0/1     Pending             0          0s
job-multi-nnf9q   0/1     ContainerCreating   0          0s
job-multi-nnf9q   1/1     Running             0          17s
job-multi-nnf9q   0/1     Completed           0          37s
job-multi-s7fvt   0/1     Pending             0          0s
job-multi-s7fvt   0/1     Pending             0          0s
job-multi-s7fvt   0/1     ContainerCreating   0          0s
job-multi-s7fvt   1/1     Running             0          18s
job-multi-s7fvt   0/1     Completed           0          37s
```

> 从上面的输出中,我们可以看出,当我们设置`job.spec.parallelism`的值为1,串行的执行5个任务的时候,只有当一个任务执行完毕之后才会创建下一个pod执行下一个任务

那么我将`job.spec.parallelism`的值设置为2,还是执行5个任务,查看详细的情况
```bash
vim job-multi2.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: job-multi2
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
      - name: myjob
        image: alpine
        command: ["/bin/sh","-c","sleep 20"]
      restartPolicy: OnFailure
```

创建之后,查看pod的变动情况
```bash
# kubectl get pods -l job-name=job-multi2 --watch
NAME               READY   STATUS    RESTARTS   AGE
job-multi2-crncg   0/1     Pending   0          0s
job-multi2-9jppb   0/1     Pending   0          0s
job-multi2-crncg   0/1     Pending   0          0s
job-multi2-9jppb   0/1     Pending   0          0s
job-multi2-9jppb   0/1     ContainerCreating   0          0s
job-multi2-crncg   0/1     ContainerCreating   0          0s
job-multi2-9jppb   1/1     Running             0          17s
job-multi2-crncg   1/1     Running             0          17s
job-multi2-9jppb   0/1     Completed           0          37s
job-multi2-mln7t   0/1     Pending             0          0s
job-multi2-crncg   0/1     Completed           0          37s
job-multi2-mln7t   0/1     Pending             0          0s
job-multi2-mln7t   0/1     ContainerCreating   0          0s
job-multi2-b6hh2   0/1     Pending             0          0s
job-multi2-b6hh2   0/1     Pending             0          0s
job-multi2-b6hh2   0/1     ContainerCreating   0          0s
job-multi2-mln7t   1/1     Running             0          17s
job-multi2-b6hh2   1/1     Running             0          17s
job-multi2-mln7t   0/1     Completed           0          37s
job-multi2-vngrf   0/1     Pending             0          0s
job-multi2-b6hh2   0/1     Completed           0          37s
job-multi2-vngrf   0/1     Pending             0          0s
job-multi2-vngrf   0/1     ContainerCreating   0          0s
job-multi2-vngrf   1/1     Running             0          17s
job-multi2-vngrf   0/1     Completed           0          37s
```

> 根据上面的输出可以看出,这次是每次启动两个pod来执行任务

### job 扩容
`job.spec.parallelism` 定义的并行度表示同时运行的pod的对象数,此属性支持运行时调整从而改变其队列总数,实现扩容和缩容

```bash
# kubectl scale jobs.batch job-multi2 --replicas=n
```

### 删除job
job控制器再其pod运行完毕之后,将不再占用系统资源,用户金额按需保留或使用资源删除命令将其删除,不过,如果某job控制器的容器应用总是无法正常结束运行,而其`restartPolicy`字段又定义为了自动重启,则可能一直处于不停的重启和错误的循环过程中,为了防止这种情况,job控制器提供了两个属性来抑制这种情况的发生: 

- `job.spec.activeDeadlineSeconds`: job的deadline(最后期限),用于为其制定活动的时间长度,超出此时长后的作业将被终止
- `jobs.spec.backoffLimit`: 将作业标记为失败状态之前的重试次数,默认为6

## CronJob 控制器

CronJob 控制器用户管理Job 控制器资源的运行时间,Job 控制器定义的作业任务在其控制器资源创建之后便会立即生效,但是CronJob可以类似于Linux的Crontab计划任务的方式,控制其运行的时间点以及重复运行的方式,具体如下:

- 在未来的某时间点运行作业一次
- 在指定的时间点重复运行作业

CronJob 对象支持使用的时间格式类似crontab,略有不同的是CronJob控制器在指定的时间点时,`?`和`*`的意义相同,都表示任何可用的有效值

### 创建CronJob控制器

crontab控制器的spec字段可以嵌套使用以下:

- jobTemplate:job控制器模板,用于为CronJob控制器生成job对象,为必选字段
- schedule: Cron格式的作用调度运行时间点,为必选字段
- concurrencyPolicy: 并发执行策略,可用的值有`Allow`允许,`Forbid`禁止和`Replace`替换,用于定义前一次作业运行尚未完成时是否以及如何运行后一次作业
- failedJobsHistoryLimit : 为失败的任务执行保留的历史记录,默认为1
- successfulJobsHistoryLimit : 为成功的任务保留的历史记录,默认为3
- startingDeadlineSeconds: 因为各种原因缺乏执行作业的时间而导致的启动作业错误的超时时长,会被计入错误历史记录
- suspend: 是否挂起后续的任务执行,默认为false,对运行中的作业不会产生影响

下面是一个简单的资源清单文件,每隔两分钟运行一次由jobTemplate定义的任务

```bash
# vim cronjob-example.yaml

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-example
  labels:
    app: mycrontab
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    metadata:
      labels:
        app: mycrontab-jobs
    spec:
      parallelism: 2
      template:
        spec:
          containers:
          - name: myjob
            image: alpine
            command:
            - /bin/sh
            - -c
            - date;echo Hello from the kubernetes cluster;sleep 10
          restartPolicy: OnFailure

```


启动并且查看状态
```bash
# kubectl apply -f cronjob-example.yaml
cronjob.batch/cronjob-example created
root@master1:~# kubectl get cronjobs.batch cronjob-example 
NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-example   */2 * * * *   False     0        <none>          15s
```

- SCHEDULE: 表示调度的时间点
- SUSPEND:表示后续任务是否处于挂起状态,即暂停任务的调度和运行
- ACTIVE表示活动状态的job对象数量
- LAST SCHEDULE : 表示上一次调度运行至此刻的时长


### CronJob的控制机制

CronJob 控制器是更高级别的资源控制器,它以job控制器资源为管控对象,并借助于它管理pod资源对象

```bash
# kubectl get jobs -l app=mycrontab-jobs
NAME                         COMPLETIONS   DURATION   AGE
cronjob-example-1597481520   2/1 of 2      27s        4m39s
cronjob-example-1597481640   2/1 of 2      27s        2m39s
cronjob-example-1597481760   2/1 of 2      15s        39s
```

上面的命令只能列出三条,因为默认的`successfulJobsHistoryLimit`值为3

如果作业重复执行时指定的时间点接近,而作业的时长跨过了其两次执行的时间长度,那么就会出现两个job同时存在的情形,有些job对象可能会存在无法或者不能同时运行的情况,这个时候就要通过`concurrencyPolicy`指定并发执行的策略:

- Allow: 默认,表示允许前后job,甚至属于同一个cronjob的更多job同时运行
- Forbid: 表示用于禁止前后两个job同时运行,入过前一个尚未结束,那么后一个不会启动
- Replace: 用于让后一个job取代前一个job,即终止前一个并启动后一个
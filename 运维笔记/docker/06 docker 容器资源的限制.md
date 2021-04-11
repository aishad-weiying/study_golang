默认情况下,容器没有资源限定,可以使用系统所有的资源,docker 通过docker run 配置容器的内存, cpu, 磁盘io使用量

其中许多的功能都要求您的内核支持linux功能,要检查支持,可以使用 docker info 命令,如果内核中禁用了某些功能,您可能会在输出的结尾看到警告信息,比如WARNING: No swap limit support


## 内存限制
对于Linux 主机,如果没有足够的内存来执行重要的系统任务,将会抛出 OOM 或者 Out of memory exception(内存溢出,内存泄露或者内存异常 ),随后系统开始杀死进程释放内存,每个进程都有可能会被kill掉,包括dockerd和其他的应用程序,如果重要的系统进程被kill,会导致整个系统宕机

产生 OOM 异常的时候,docker 尝试通过调整docker守护程序上的OOM优先级来减轻这些风险,以便它比系统上的其他进程更不可能被杀死

### 限制容器对内存的访问

docker 可以强制执行硬性内存限制,即只允许容器使用给定的内存大小,docker 也可以执行非硬性的内存限制,即容器可以尽可能多的使用内存,除非检测到主机上的内存不够用了


#### 内存限制参数

- -m or memory= : 容量可以使用的最大内存量,如果设置了这个选项,则允许的最小值为4m
- --memory-swap : 容器可以使用的交换分区的大小,要在设置物理内存限制的前提才能设置交换分区的限制
- --memoty-swappiness : 设置容器使用交换分区的倾向性,范围为0-100,0表示不用,100表示能用就用
- --kernel-memory : 容器可以使用的最大内核内存量,最小为4m,由于内核内存与用户内存隔离,因此无法与用户空间内存直接交换,因此内核内存不足的容器可能会阻塞宿主机资源,对其他的容器产生影响
- --menory-reservation:允许指定小于 --memory的软限制,当docker检测到主机上的争用或内存不足的时候,会激活该限制,如果使用了这个参数,必须将其设置为低于--memory才能使其优先,但是这个参数为软限制
- --oom-kill-disable : 默认情况下,发生oom时,kernel 会杀死容器内进程,但是可以使用这个参数来禁止 oom 发生在指定的容器上,这个参数仅在设置 -m 选线的容器上才会生效,如果没有设置-m选项,产生oom的时候,主机为了释放资源还是会杀死系统进程的

#### 交换分区限制(k8s在1.8版本之后就禁用了swap,了解即可)
- --memo-swap : 只有在设置了-m参数之后,才会有意义,当容器使用了超过-m指定的限制时,超出部分的内存置到硬盘上,但是使用swap会导致应用程序的性能降低
- --memory-swap：值为正数， 那么--memory和--memory-swap都必须要设置，--memory-swap表示你能使用的内存和swap分区大小的总和，例如： --memory=300m, --memory-swap=1g, 那么该容器能够使用 300m 内存和 700m swap，即--memory是实际物理内存大小值不变，而实际的计算方式为(--memory-swap)-(--memory)=容器可用swap
- --memory-swap：如果设置为0，则忽略该设置，并将该值视为未设置，即未设置交换分区。
- --memory-swap：如果等于--memory的值，并且--memory设置为正整数，容器无权访问swap即也没有设置交换分区
- --memory-swap：如果设置为unset，如果宿主机开启了swap，则实际容器的swap值为2x( --memory)，即两倍于物理内存大小，但是并不准确。
- --memory-swap：如果设置为-1，如果宿主机开启了swap，则容器可以使用主机上swap的最大空间。

### 容器内存软限制

```bash
# 1. 下载用于测试使用的镜像
docker pull lorel/docker-stress-ng

# 软限制参数
docker run -it --rm --oom-kill-disable --memory 128m --memory-reservation 64m lorel/docker-stress-ng --vm 2 --vm-bytes 256m

--vm 指定要启动都少个工作进程
--vm-bytes 指定每个进程使用的内存

# 查看软限制大小
cat /sys/fs/cgroup/memory/docker/容器id/memory.soft_limit_in_bytes 
67108864

# 使用 docker stats 查看docker 镜像占用的资源
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
b0838de66282        angry_keldysh       0.00%               126MiB / 128MiB     98.42%              1.17kB / 0B         0B / 0B             5

```

> 从上面的结果中我们能看出,设置了软限制为64M,但是实际的使用中是可以突破这个限制的

### 容器内存硬限制
还是使用上面的镜像测试
```bash
root@client:~# docker run -it --rm --oom-kill-disable --memory 128m lorel/docker-stress-ng --vm 2 --vm-bytes 256m

# 使用 docker stats 查看使用情况
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
9af4ab05cdf3        focused_greider     0.00%               126MiB / 128MiB     98.42%              766B / 0B           0B / 0B             5

# 查看docker 容器硬限制的文件
 cat /sys/fs/cgroup/memory/docker/容器id/memory.limit_in_bytes 
134217728

# 这个值可以在此文件中通过echo命令更改
echo "268435456" > /sys/fs/cgroup/memory/docker/容器id/memory.limit_in_bytes

# docker stats查看资源的使用情况
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
9af4ab05cdf3        focused_greider     0.00%               254MiB / 256MiB     99.21%              1.12kB / 0B         0B / 0B             5

# 注意: 只能利用这个硬限制的文件来调大内存,但是不能缩减内存
```

> 通过上面我们可以看出,即使启动容器的时候,我们指定了超出硬限制的内存,但是实际上还是不能超过硬限制

swap 的限制和内存限制类似,因为后续的k8s中,禁用了swap,那么这里就不做演示

## CPU 限制
一个宿主机有几十个核心的cpu,但是有成百上千的进程,那么这些该去怎么执行?

- 实时优先级 : 0-99
- 非实时优先级(nice):-20-19.对应100-139的进程优先级

Linux kernel 进程的调度基于CFS 完全公平调度

- CPU 密集型的场景: 优先级越低越好,计算密集型任务的特点就是进行大量的运算,消耗cpu 资源
- IO 密集型的场景:优先级值越高,设计到的网络,磁盘IO的任务都是IO密集型任务,这类任务的特点就是CPU消耗很少,任务的大部分时间都在等待IO操作完成(因为IO操作的速度远远低于CPU和内存的速度)

##### 磁盘调度算法
```bash
cat /sys/block/sda/queue/scheduler 
noop deadline [cfq] 
# 如果是固态盘的话建议使用 noop
```
默认情况下,每个容器对主机CPU周期的访问权限是不受限制的,但是我们可以设置各种约束来限制给定容器访问主机的CPU周期,大多数用户使用的是默认的CFS调度,在docker 1.13 版本之后,还可以配置实时优先级

- --cpus= : 指定容器可以使用多少CPU资源,例如,如果主机有两个CPU,并且设置了 --cpus=1.5 ,那么这个容器最多只能访问1.5个CPU,这相当于设置--cpu-period =100000和--cpu-quota=150000
- -cpu-period :设置CPU CFS调度程序周期，它与--cpu-quota一起使用，默认为100微妙，范围从 100ms~1s，即[1000, 1000000]
- --cpu-quota :在容器上添加CPU CFS配额，也就是cpu-quota / cpu-period的值，通常使用--cpus设置此值
- --cpuset-cpus  主要用于指定容器运行的CPU编号，也就是我们所谓的绑核
```bash
cat /sys/fs/cgroup/cpuset/docker/容器id/cpuset.cpus
0-1

```

- --cpuset-mem  设置使用哪个cpu的内存，仅对 非统一内存访问(NUMA)架构有效top
- --cpu-shares   主要用于cfs中调度的相对权重，,cpushare值越高的，将会分得更多的时间片，默认的时间片1024，最大262144

### 测试CPU限制

1. 不指定cpu限定会默认全部占满
```bash
docker run -it --rm lorel/docker-stress-ng --cpu 2 --vm 2

# 使用 docker stats 查看资源使用情况

CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
9118c0c3d2af        lucid_kepler        198.97%             532.7MiB / 1.861GiB   27.95%              1.05kB / 0B         0B / 0B             7

```

2. 限制容器cpu使用
```bash
root@client:~# docker run -it --rm --cpus 1 lorel/docker-stress-ng --cpu 2 --vm 2

# 查看资源的使用情况
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
787c51374da3        peaceful_feistel    99.88%              532.8MiB / 1.861GiB   27.96%              906B / 0B           0B / 0B             7
```

3. 将容器运行在指定的cpu资源
```bash
 docker run -it --rm --cpus 1 lorel/docker-stress-ng --cpu 2 --vm 2
# 不指定的时候,cpu的使用率
%Cpu0  : 51.2 us,  0.0 sy,  0.0 ni, 48.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 49.7 us,  0.0 sy,  0.0 ni, 50.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

# 可以看出是均匀的使用两个CPU的

# 限制容器运行在指定的CPU上面
docker run -it --rm --cpus 1 --cpuset-cpus 1 lorel/docker-stress-ng --cpu 2 --vm 2

查看资源的使用情况
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni, 94.6 id,  0.0 wa,  0.0 hi,  5.4 si,  0.0 st
%Cpu1  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

4. cpu-shares 的值
启动两个容器,一个 cpu-shares指定为1000 ,一个指定为500 ,查看资源的使用率
```bash
CONTAINER ID        NAME                 CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
ac10f7c834cb        stupefied_kapitsa    133.05%             532.7MiB / 1.861GiB   27.95%              906B / 0B           0B / 0B             7
81b8a57b774b        optimistic_mestorf   65.89%              532.7MiB / 1.861GiB   27.95%              586B / 0B           8.19kB / 0B         7
```

可以看出 --cpu-shares值为1000的为--cpu-shares值为500的cpu得到使用率的两倍

动态修改 CPU shares 的值
``` bash
cat /sys/fs/cgroup/cpu/docker/容器id/cpu.shares 
500

cat /sys/fs/cgroup/cpu/docker/容器id/cpu.shares 
1000
```
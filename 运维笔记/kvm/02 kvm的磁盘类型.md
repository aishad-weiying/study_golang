# 虚拟机磁盘格式：

## raw：指定多大就创建多大，直接占用指定大小的空间
目前来看，KVM和XEN默认的格式好像还是这个格式，

- 优点：

1. 因为其原始，有很多原生的特性，例如直接挂载也是一件简单的事情
2. 支持转换成其它格式的虚拟机镜像对裸露的它来说还是很简单的（如果其它格式需要转换，有时候还是需要它做为中间格式）
3. 空间不足的时候可以支持在原盘上追加空间
```bash
	dd if=/dev/zero of=zeros.raw bs=1024k count=4096	#（先创建4G的空间）
	cat foresight.img zeros.raw > new-foresight.img		#（追加到原有的镜像之后）
```

- 缺点：
	如果你要把整块磁盘都拿走的话得全盘拿了（copy镜像的时候）,会比较消耗网络带宽和I/O。

## cow
曾经qemu的写时拷贝的镜像格式，目前由于历史遗留原因不支持窗口模式。从某种意义上来说是个弃婴，还没得它成熟就死在腹中，后来被qcow格式所取代。

## qcow
一代的qemu的cow格式，刚刚出现的时候有比较好的特性，但其性能和raw格式对比还是有很大的差距，目前已经被新版本的qcow2取代。

## qcow2
qcow2，是openstack默认也是比较推荐的格式，将差异保存在一个文件，文件比较小而且做快照也比较小，空间是动态增长的：现在比较主流的一种虚拟化镜像格式，经过一代的优化，目前qcow2的性能上接近raw裸格式的性能

- 特点
qcow2的snapshot，可以在镜像上做N多个快照：更小的存储空间，即使是不支持holes的文件系统也可以（这下du -h和ls -lh看到的就一样了）,支持多个snapshot，对历史snapshot进行管理,支持zlib的磁盘压缩,支持AES的加密

## vmdk:VMware的格式

## vdi:VirtualBox的格式

## 磁盘格式的转换：
### raw转换为qcow2：
此步骤使用qemu-img工具实现，如果机器上没有，可以通过rpm或yum进行安装，包名为qemu-img。
qemu-img是专门虚拟磁盘映像文件的qemu命令行工具。
```bash
[ root@localhost ~]# qemu-img convert -f raw centos.img -O qcow2 centos.qcow2
	参数说明：convert   将磁盘文件转换为指定格式的文件
                     -f   指定需要转换文件的文件格式
                    -O  指定要转换的目标格式
     转换完成后，将新生产一个目标映像文件，原文件仍保存。
```

### qcow2转换为raw：
```bash
[ root@localhost ~]# qemu-img convert -O qcow2 my.raw myqow.qcow
```

### VMDK转换为qcow2:
```bash
[ root@localhost ~]# qemu-img convert -f vmdk -O qcow2 xxx.vmdk    xxx.img
```
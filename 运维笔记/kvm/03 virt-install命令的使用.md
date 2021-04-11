## virt-install命令的使用
usage: virt-install --name NAME --ram RAM STORAGE INSTALL [options]使用指定安装介质新建虚拟机。
```bash
- optional arguments:
-h, --help 		show this help message and exit
--version 		show program's version number and exit
--connect 		URI 使用 libvirt URI 连接到 hypervisor

通用选项:
	-n NAME, --name NAME 	客户端事件名称
	--memory MEMORY 	配置虚拟机内存分配。例如：
			--memory 1024 (in MiB)
			--memory 512,maxmemory=1024

	--vcpus VCPUS 为虚拟机配置的 vcpus 数。例如：
			--vcpus 5
			--vcpus 5,maxcpus=10,cpuset=1-4,6,8
			--vcpus sockets=2,cores=4,threads=2,

	--cpu CPU CPU 型号及功能。例如：
			--cpu coreduo,+x2apic
			--cpu host

	--metadata METADATA 配置虚拟机元数据。例如：
			--metadata name=foo,title="My pretty title",uuid=...
			--metadata description="My nice long description"
安装方法选项:
	--cdrom CDROM 光驱安装介质
	-l LOCATION, --location LOCATION
			安装源(例如：nfs：host:/path、http://host/path、ftp://host/path)

	--pxe 使用 PXE 协议从网络引导
	--import 在磁盘映像中构建虚拟机
	--livecd 将光驱介质视为 Live CD
	-x EXTRA_ARGS, --extra-args EXTRA_ARGS
			附加到使用 --location 引导的内核的参数
	--initrd-inject INITRD_INJECT
			使用 --location 为 initrd 的 root
			添加给定文件
	--os-variant DISTRO_VARIANT
			在其中安装 OS 变体的虚拟机，比如 'fedora18'、'rhel6'、'winxp' 等等。
	--boot BOOT 配置虚拟机引导设置。例如：
			--boot hd,cdrom,menu=on
			--boot init=/sbin/init (for containers)
	--idmap IDMAP 为 LXC 容器启用用户名称空间。例如：
			--idmap uid_start=0,uid_target=1000,uid_count=10
设备选项:
	--disk DISK 使用不同选项指定存储。例如：
			--disk size=10 (new 10GiB image in default location)
			--disk /my/existing/disk,cache=none
			--disk device=cdrom,bus=scsi
			--disk=?
	-w NETWORK, --network NETWORK
		配置虚拟机网络接口。例如：
			--network bridge=mybr0
			--network network=my_libvirt_virtual_net
			--network network=mynet,model=virtio,mac=00:11...
			--network none
			--network help
	--graphics GRAPHICS 配置虚拟机显示设置。例如：
			--graphics vnc
			--graphics spice,port=5901,tlsport=5902
			--graphics none
			--graphics vnc,password=foobar,port=5910,keymap=ja
	--controller CONTROLLER
			配置虚拟机控制程序设备。例如：
			--controller type=usb,model=ich9-ehci1
	--input INPUT 配置虚拟机输入设备。例如：
			--input tablet
			--input keyboard,bus=usb
	--serial SERIAL 配置虚拟机串口设备
	--parallel PARALLEL 配置虚拟机并口设备
	--channel CHANNEL 配置虚拟机沟通频道
	--console CONSOLE 配置虚拟机与主机之间的文本控制台连接
	--hostdev HOSTDEV 将物理 USB/PCI/etc 主机设备配置为与虚拟机共享
	--filesystem FILESYSTEM
				将主机目录传递给虚拟机。例如：
					--filesystem /my/source/dir,/dir/in/guest
					--filesystem template_name,/,type=template
	--sound [SOUND] 配置虚拟机声音设备模拟
	--watchdog WATCHDOG 配置虚拟机 watchdog 设备
	--video VIDEO 配置虚拟机视频硬件。
	--smartcard SMARTCARD
			配置虚拟机智能卡设备。例如：
				--smartcard mode=passthrough
	--redirdev REDIRDEV 配置虚拟机重定向设备。例如：
			--redirdev usb,type=tcp,server=192.168.1.1:4000
	--memballoon MEMBALLOON
			配置虚拟机 memballoon 设备。例如：
				--memballoon model=virtio
	--tpm TPM 配置虚拟机 TPM 设备。例如：
			--tpm /dev/tpm
	--rng RNG 配置虚拟机 RNG 设备。例如：
			--rng /dev/random
	--panic PANIC 配置虚拟机 panic 设备。例如：
			--panic default
虚拟机配置选项:
	--security SECURITY 设定域安全驱动器配置。
	--numatune NUMATUNE 为域进程调整 NUMA 策略。
	--memtune MEMTUNE 为域进程调整内粗策略。
	--blkiotune BLKIOTUNE
			为域进程调整 blkio 策略。
	--memorybacking MEMORYBACKING
			为域进程设置内存后备策略。例如：
				--memorybacking hugepages=on
	--features FEATURES 设置域 <features> XML。例如：
			--features acpi=off
			--features apic=on,eoi=on
	--clock CLOCK 设置域 <clock> XML。例如：
			--clock offset=localtime,rtc_tickpolicy=catchup
	--pm PM 配置 VM 电源管理功能
	--events EVENTS 配置 VM 生命周期管理策略
	--resource RESOURCE 配置 VM 资源分区（cgroups）
虚拟化平台选项:
	-v, --hvm 客户端应该是一个全虚拟客户端
	-p, --paravirt 这个客户端一个是一个半虚拟客户端
	--container 这台虚拟机需要一个容器客户端
	--virt-type HV_TYPE 要使用的管理程序名称(kvm、qemu、xen等等)
	--arch ARCH 模拟的 CPU 构架
	--machine MACHINE 要模拟的机器类型

其它选项:
	--autostart 引导主机时自动启动域。
	--wait WAIT 等待安装完成的分钟数。
	--noautoconsole 不要自动尝试连接到客户端控制台
	--noreboot 完成安装后不要引导虚拟机。
	--print-xml [XMLONLY]		输出所生成域 XML，而不是创建虚拟机。
	--dry-run 完成安装步骤，但不要创建设备或者定义虚拟机。
	--check CHECK 启用或禁用验证检查。例如：
				--check path_in_use=off
				--check all=off
	-q, --quiet 禁止无错误输出
	-d, --debug 输入故障排除信息
使用 '--option=?' 或者 '--option help' 查看可用子选项

```


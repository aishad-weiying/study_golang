# LVS的NAT模型和DR模型的实现
## LVS集群的设计要注意的问题：
1. 是否需要会话保持
2. 是否需要共享存储
	共享存储：NAS， SAN， DS（分布式存储）
	数据同步：

3. 一个ipvs主机可以同时定义多个Cluster Service（TCP， UDP， AH， ESP， AH_ESP, SCTP）
4. 一个Cluster Service上只要应该有一个RS
5. 定义时致多名lvs的类型和lvs使用的调度算法
## ipvsadm的使用
	管理集群服务
	管理服务上的RS

- 程序包：ipvsadm
	Unit File: ipvsadm.service
	主程序：/usr/sbin/ipvsadm
	规则保存工具：/usr/sbin/ipvsadm-save
	规则重载工具：/usr/sbin/ipvsadm-restore
	配置文件：/etc/sysconfig/ipvsadm-config
### ipvsadm命令的使用
- 核心功能：
	集群服务管理：增、删、改
	集群服务的RS管理：增、删、改
	查看
1. 集群管理：增、改、删

- 增、改：
	ipvsadm -A|E -t|u|f service-address [-s scheduler] [-p [timeout]]

- 删除：
	ipvsadm -D -t|u|f service-address
- service-address：
	-t: TCP协议的端口，VIP:TCP_PORT
	-u: UDP协议的端口，VIP:UDP_PORT
	-f：firewall MARK，防火墙标记，一个数字
- 选项：
	-s  scheduler：指定集群的调度算法，默认为wlc

2. 管理集群上的RS：增、改、删

- 增、改：
	ipvsadm -a|e -t|u|f service-address -r server-address [-g|i|m] [-w weight]

- 删除：
	ipvsadm -d -t|u|f service-address -r server-address

- server-address：RS服务器的地址
	rip[:port] 如省略port，不作端口映射

- 选项：
	lvs类型：
	-g: gateway, dr类型，默认
	-i: ipip, tun类型
	-m: masquerade, nat类型
	-w weight：权重

3. 通用

- 清空定义的所有内容：ipvsadm –C

- 清空计数器：ipvsadm -Z [-t|u|f service-address]

- 查看：ipvsadm -L|l [options]
	--numeric, -n：以数字形式输出地址和端口号
	--exact：扩展信息，精确值
	--connection，-c：当前IPVS连接输出
	--stats：统计信息
	--rate ：输出速率信息

4. 保存及重载规则

- 保存：建议保存至/etc/sysconfig/ipvsadm
	ipvsadm-save > /PATH/TO/IPVSADM_FILE
	ipvsadm -S > /PATH/TO/IPVSADM_FILE
	systemctl stop ipvsadm.service

- 重载：
	ipvsadm-restore < /PATH/FROM/IPVSADM_FILE
	ipvsadm -R < /PATH/FROM/IPVSADM_FILE
	systemctl restart ipvsadm.service

## LVS-NAT的实现
- 设计要点：
	(1) RIP与DIP在同一IP网络, RIP的网关要指向DIP
	(2) 支持端口映射
	(3) Director要打开核心转发功能

	[![](http://aishad.top/wordpress/wp-content/uploads/2019/05/lvs-nat的实现.png)](http://aishad.top/wordpress/wp-content/uploads/2019/05/lvs-nat的实现.png)

- 实验步骤：
1. VS服务器开启端口转发：
	vim /etc/sysctl.conf
		net.ipv4.ip_forward = 1
	sysctl -p

2. VS服务器增加集群：
	ipvsadm -A -t 172.22.45.131:80 -s rr
	查看：ipvsadm -L -n

3. VS服务器添加RS：
	ipvsadm -a -t 172.22.45.131:80 -r 192.168.45.132:8080 -m
	ipvsadm -a -t 172.22.45.131:80 -r 192.168.45.133:8080 -m

4. 保存规则：
	 ipvsadm -S > /etc/sysconfig/ipvsadm

5. 重载规则：
	 ipvsadm -R < /sysconfig/ipvsadm

## LVS-DR的实现
	DR模型中各个主机均要配置VIP，解决地址冲突的方式有三种：
	（1）在前端网关做静态绑定
	（2）在各RS使用arptables(让RS拒绝对arp请求的VIP请求做响应)
		arptables -A IN -d $VIP -j DROP
		arptables -A OUT -s $VIP -j mangle –mangle-ip-s $RIP
	（3）在各RS修改内核参数，来限制arp响应和通告的级别
		/proc/sys/net/ipv4/conf/all/arp_ignore：是否响应arp请求
		0：默认值，表示可使用本地任意接口上配置的任意地址进行响应
		1：仅在请求的目标IP配置在本地主机的接收到请求报文的接口上时，才给予响应
		/proc/sys/net/ipv4/conf/all/arp_announce：是否接收，记录来自别的主机的arp通告，或者通告给别人
		0：默认值，把本机所有接口的所有信息向每个接口的网络进行通告
		1：尽量避免将接口信息向非直连网络进行通告
		2：必须避免将接口信息向非本网络进行通告

- 设计要点：
	（1）RS服务器采用双IP桥接网络，一个是DIP，一个是VIP
	（2）Web服务器采用和DIP相同网段和RS链接
	（3）每个Web服务器配置VIP
	（4）每个Web服务器可以出外网

[![](http://aishad.top/wordpress/wp-content/uploads/2019/05/LVS-DR实现.png)](http://aishad.top/wordpress/wp-content/uploads/2019/05/LVS-DR实现.png)

- 实验步骤：
1. VS配置	
	ifconfig eth0:1 172.22.45.133 netmask 255.255.255.255 broadcast 172.22.45.133 up（vip仅用来做调度器的目标地址，掩码可以设置子为32为，广播地址指向自己）
	route add -host 172.22.45.133 dev eth0:1(要求从那个网络接口进来的请求报文，就必须从哪个接口出去,VS可以不配置)

2. RS配置(两台RS服务器都做相同的操作)
	echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
	echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
	echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
	ifconfig lo:1 172.22.45.133 netmask 255.255.255.255 broadcast 172.22.45.133 up
	route add -host 172.22.45.133 dev lo:1

3. 添加集群服务
	ipvsadm -A 172.22.45.133:80 -s wrr
	ipvsadm -a -t 172.22.45.133:80 -r 172.22.45.134 -g -w 1
	ipvsadm -a -t 172.22.45.133:80 -r 172.22.45.135 -g -w 2

### DR模型VS和RS配置时可使用脚本完成：
```bash
RS的配置脚本
#!/bin/bash
vip=10.0.0.100
mask='255.255.255.255'
dev=lo:1
case $1 in
start)
	echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
	echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
	echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
	ifconfig $dev $vip netmask $mask #broadcast $vip up
	#route add -host $vip dev $dev
	;;
stop)
	ifconfig $dev down
	echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
	echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
	echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
	;;
*) 
	echo "Usage: $(basename $0) start|stop"
	exit 1
;;
esac

VS的配置脚本
#!/bin/bash
vip='10.0.0.100'
iface='lo:1'
mask='255.255.255.255'
port='80'
rs1='192.168.0.101'
rs2='192.168.0.102'
scheduler='wrr'
type='-g'
case $1 in
start)
	ifconfig $iface $vip netmask $mask #broadcast $vip up
	iptables -F
	ipvsadm -A -t ${vip}:${port} -s $scheduler
	ipvsadm -a -t ${vip}:${port} -r ${rs1} $type -w 1
	ipvsadm -a -t ${vip}:${port} -r ${rs2} $type -w 1
	;;
stop)
	ipvsadm -C
	ifconfig $iface down
	;;
*)
	echo "Usage $(basename $0) start|stop“
	exit 1
esac
```
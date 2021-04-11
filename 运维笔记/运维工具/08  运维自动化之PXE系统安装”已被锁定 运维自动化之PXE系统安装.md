# 网络的配置方式
1. 静态指定
2. 动态获取: bootp:boot protocol MAC与IP一一静态对应
	dhcp:增强的bootp，动态，引用了租约的概念

# DHCP
	动态主机配置协议
	局域网协议，UDP协议


- 主要用途
	用于内部网络和网络服务供应商自动分配IP地址给用户
	用于内部网络管理员作为对所有电脑作集中管理的手段

- 使用场景
	自动化安装系统
	解决IPV4资源不足问题

## DHCP的报文
客户端：68/udp
服务端：67/udp
报文是通过广播获取

1. DHCP DISCOVER：客户端发送请求服务器
2. DHCP OFFER ：服务器应答客户端
3. DHCP REQUEST：客户端发送确定信息服务器
4. DHCP ACK ：服务器发送确认信息到客户端
5. DHCP NAK：服务器到客户端,通知用户无法分配合适的IP地址
6. DHCP DECLINE ：客户端到服务器，指示地址已被使用
7. DHCP RELEASE：客户端到服务器，放弃网络地址和取消剩余的租约时间
8. DHCP INFORM：客户端到服务器, 客户端如果需要从DHCP服务器端获取更为详细的配置信息，则发送Inform报文向服务器进行请求，极少用到

DHCP的续租：通过单播方式通知服务端

- 50% ：租赁时间达到50%时来续租，刚向DHCP服务器发向新的DHCPREQUEST请求。如果dhcp服务没有拒绝的理由，则回应DHCPACK信息。当DHCP客户端收到该应答信息后，就重新开始新的租用周期

- 87.5%：如果之前DHCP Server没有回应续租请求，等到租约期的7/8时，主机会再发送一次广播请求

### DHCP服务器的实现

1. 安装
```bash
[ root@localhost ~]# yum -y install dhcp

Dhcp Server
	二进制程序：/usr/sbin/dhcpd
		配置文件：
			/etc/dhcp/dhcpd.conf --> /etc/rc.d/init.d/dhcpd
			/etc/dhcp/dhcpd6.conf--> /etc/rc.d/init.d/dhcpd6
			/usr/sbin/dhcrelay
				/etc/rc.d/init.d/dhcrelay
			dhcp server:67/udp
			dhcp client: 68/udp
			dhcpv6 client:546/udp
Dhcp client
		dhclient
		自动获取的IP信息：/var/lib/dhclient
```

2. 编辑配置文件
```bash
[ root@localhost ~]# vim /etc/dhcp/dhcpd.conf
# 定义搜索域的域名，例如在ping www 的时候，会自动在后面加上指定的搜索域域名
option domain-name "aishad.top";
#定义DNS服务器
option domain-name-servers 192.168.55.1, 8.8.8.8;
# 默认租约期限，单位秒
default-lease-time 600;
#最大租约期限
max-lease-time 7200;
#日志记录
log-facility local7;
#全局配置

# 定义子网，定义为其分配地址的网络
subnet 192.168.55.0 netmask 255.255.255.0 {
  range 192.168.55.2 192.168.55.200;  #地址范围
#  option domain-name-servers 192.168.55.1;
#  option domain-name "";
  option routers 192.168.55.1; 网关
#  option broadcast-address 10.5.5.31;
#  default-lease-time 600;
#  max-lease-time 7200;
}

[ root@localhost ~]# systemctl start dhcpd.service
#监听了udp的67端口
```

3. 查看租约信息
```bash
[ root@localhost ~]# cat /var/lib/dhcpd/dhcpd.leases
# The format of this file is documented in the dhcpd.leases(5) manual page.
# This lease file was written by isc-dhcp-4.2.5

server-duid "\000\001\000\001$\253<\207\000\014)\177*\307";

lease 192.168.55.2 {
  starts 0 2019/06/30 09:36:10;
  ends 0 2019/06/30 09:46:10;
  cltt 0 2019/06/30 09:36:10;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 00:0c:29:47:ce:09;
}

```

4. dhcp为指定的服务器分配固定地址
```bash
host name {
	hardware ethernet 指定mac地址
	fixed-address ip地址(不能使用地址池范文内的地址)
}
```
## 在dhcp配置文件中指定引导文件
客户端在获取地址的时候，要通知客户端TFTP服务器的地址，和TFTP上引导文件的位置
```bash
[ root@localhost ~]# vim /etc/dhcp/dhcpd.conf
subnet 192.168.55.0 netmask 255.255.255.0 {
  range 192.168.55.2 192.168.55.200;  #地址范围
#  option domain-name-servers 192.168.55.1;
#  option domain-name "";
  option routers 192.168.55.1; 网关
#  option broadcast-address 10.5.5.31;
#  default-lease-time 600;
#  max-lease-time 7200;
	next-server 192.168.55.1; #TFTP服务器地址
	filename "pxelinux.0"; #指明引导文件名称
}
```

# TFTP服务器
Trivial File Transfer Protocol ，是一种用于传输文件的简单高级协议，是文件传输协议（FTP）的简化版本。用来传输比文件传输协议（FTP）更易于使用但功能较少的文件

- TFTP和FTP的区别
```bash
1、安全性区别
	FTP支持登录安全，具有适当的身份验证和加密协议，在建立连接期间需要与FTP身份验证通信
	TFTP是一种开放协议，缺乏安全性，没有加密机制，与TFTP通信时不需要认证
2、传输层协议的区别
	FTP使用TCP作为传输层协议，TFTP使用UDP作为传输层协议
3、使用端口的区别
	FTP使用2个端口：TCP端口21，是个侦听端口；TCP端口20或更高TCP端口1024以上用于源连接
	TFTP仅使用一个具有停止和等待模式的端口：端口69/udp
4、RFC的区别
	FTP是基于RFC 959文档，带有其他RFC涵盖安全措施；TFTP基于RFC 1350文档
5、执行命令的区别
	FTP有许多可以执行的命令（get，put，ls，dir，lcd）并且可以列出目录等
	TFTP只有5个指令可以执行（rrq，wrq，data，ack，error）
```

- 安装和使用
```bash
[ root@localhost mnt]# yum -y install tftp-server.x86_64
[ root@localhost mnt]# systemctl start tftp.service 
# 监听udp的69端口
[ root@localhost mnt]# ss -unl
State      Recv-Q Send-Q Local Address:Port                Peer Address:Port              
UNCONN     0      0                  *:67                             *:*                  
UNCONN     0      0                 :::69                            :::*    
```

- 准备pxelinux.0文件
```bash
[ root@localhost tftpboot]# yum install syslinux
[ root@localhost tftpboot]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/pxelinux.0

#是光盘启动后的安装图形界面，也属于SYSLINUX项目，menu.c32版本是纯文本的菜单
[ root@localhost ~]# cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
```

- 准备系统安装需要的文件
```bash
[ root@localhost ~]# mkdir /var/lib/tftpboot/pxelinux.cfg/
# vmlinuz是内核映像initrd.img是ramfs
[ root@localhost ~]# cp /mnt/images/pxeboot/initrd.img /mnt/images/pxeboot/vmlinuz /var/lib/tftpboot/

#isolinux.bin的配置文件，当光盘启动后（即运行isolinux.bin），会自动去找isolinux.cfg文件
[ root@localhost ~]# cp /mnt/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default

[ root@localhost tftpboot]# tree /var/lib/tftpboot/
/var/lib/tftpboot/
├── initrd.img
├── menu.c32
├── pxelinux.0
├── pxelinux.cfg
│   └── default
└── vmlinuz
```

### 准备启动菜单
```bash
[ root@localhost ~]# vim /var/lib/tftpboot/pxelinux.cfg/default
default menu.c32
timeout 600
 
 
menu title PXE INSTALL MENU
 
 
label auto
  menu label Auto Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img ks=http://192.168.55.1/ks/ks7-mini.cfg net.ifnames=0 biosdevname=0
 
label manual
  menu label Manual install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img  inst.repo=http://192.168.55.1/centos/7 net.ifnames=0 biosdevname=0
 
label local
  menu default
  menu label ^Boot from local drive
  localboot 0xffff
```
### 准备全新的操作系统，安装系统测试
> 可到upload.aishad.top获取kickstart文件
# Centos最小化安装系统后的初始化操作

```bash
#!/bin/bash

#设置ps变量
cat >> /etc/profile.d/ps.sh <<EOF
PS1="\[\e[1;32m\][ \[\e[1;33m\]\u\[\e[32m\]@\h\[\e[1;32m\] \W\[\e[1;32m\]]\[\e[0m\]\[\e[1;31m\]\\\$\[\e[0m\] "
EOF

echo 'HISTTIMEFORMAT= "%F %T "' >> /etc/profile

rm -rf /etc/yum.repos.d/*
#配置阿里yum源
cat > /etc/yum.repos.d/base.repo <<EOF
[aliyun]
name=aliyun base
baseurl=https://mirrors.aliyun.com/centos/7/os/x86_64/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/centos/7/os/x86_64/RPM-GPG-KEY-CentOS-7
 
[epel]
name=epel
baseurl=https://mirrors.aliyun.com/epel/7/x86_64/
enabeld=1
gpgcheck=0
 
#[cdrom]
#name=cdrom
#baseurl=file:///mnt/cdom
#enabled=1
#gpgcheck=1
#gpgkey=file:///mnt/cdrom/RPM-GPG-KEY-CentOS-7
EOF

yum clean all &> /dev/null
yum makecache &> /dev/null && echo "yum reops Set to complete"
yum -y remove libvirt-daemon &> /dev/null


sed -i -r 's/.*(StrictHostKeyChecking).*/\1 no/'  /etc/ssh/ssh_config
sed -i -r 's/.*(GSSAPIAuthentication).*/\1 no/' /etc/ssh/sshd_config
sed -i -r 's/.*(UseDNS).*/\1 no/' /etc/ssh/sshd_config

yum -y install bash-completion &> /dev/null

#sed -i -r 's@^(GRUB_CMDLINE_LINUX=\".*)(\")$@\1 net.ifnames=0\2@' /etc/default/grub
#grub2-mkconfig -o /etc/grub2.cfg


sed -i -r 's/(^SELINUX=).*/\1disabled/' /etc/sysconfig/selinux
sed -i -r 's/(^SELINUX=).*/\1disabled/' /etc/selinux/config 
setenforce 0
systemctl stop firewalld.service &> /dev/null
systemctl disable firewalld.service &> /dev/null

systemctl stop NetworkManager &> /dev/null
systemctl disable NetworkManager &> /dev/null


yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute  bridge-utils -y

echo "Initialization successful"

exec bash

```

## 常用的服务器系统参数
```bash
[ root@openstack1 ~]# grep '^[a-z]' /etc/sysctl.conf 
net.ipv4.conf.default.rp_filter = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 687194767360
kernel.shmall = 4294967296
net.ipv4.tcp_mem = 786432 1048576 1572864
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_sack = 1
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 20480
net.core.optmem_max = 81920
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_max_tw_buckets = 20000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_timestamps = 1 #?
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.ip_local_port_range = 10001    65000
vm.overcommit_memory = 0
vm.swappiness = 10
```

## 系统常用的PAM相关的参数
```bash
[ root@openstack1 ~]# grep -v '^#' /etc/security/limits.conf 

*                soft    core               unlimited
*                hard    core               unlimited
*	             soft    nproc              1000000
*	             hard    nproc              1000000
*	             soft    nofile             1000000
*                hard    nofile             1000000
*                soft    memlock            32000
*                hard    memlock            32000
*                soft    msgqueue           8192000
*                hard    msgqueue           8192000
```

# Ubuntu最小化安装后的操作
```bash
1. 更改网卡的命名方式为eth0
root@k8s-master1:~# vim /etc/default/gru
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"

root@k8s-master1:~# update-grub

2. 更改ip地址 18.04
root@k8s-master1:~# vim /etc/netplan/01-netcfg.yaml 

network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.7.200/21]
      gateway4: 192.168.0.254
      nameservers:
              addresses: [223.5.5.5]
# 16.04 更改ip地址
root@ubuntu:~# vim /etc/network/interfaces
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 172.19.36.152
netmask 255.255.255.0
gateway 172.19.36.1
dns 172.19.10.2

3.重启并测试
root@k8s-master1:~# netplan apply

4. 更改主机名

5. 配置apt源
root@k8s-master1:~# vim /etc/apt/sources.list

deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

root@k8s-master1:~# apt update

6. 安装常用命令

root@k8s-master1:~# apt purge ufw lxd lxd-client lxcfs lxc-common

root@k8s-master1:~# apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate gcc openssh-server iotop unzip zip 
```

# Ubuntu安装中文语言包
```bash
# 安装中文语言包
root@zabbix-server:~# apt install language-pack-zh*

#配置相关的环境变量
root@zabbix-server:~# vim /etc/environment 
# 添加下面两行
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN:zh:en_US:en"

# 重新配置本地配置，选择语言为zh_CN.UTF-8
root@zabbix-server:~# dpkg-reconfigure locales
```

# 其他设置
```bash
设置重启网卡后不删除静态路由:
/etc/sysconfig/static-routes : (没有static-routes的话就手动建立一个这样的文件)
any net 192.168.3.0/24 gw 192.168.3.254
any net 10.250.228.128 netmask 255.255.255.192 gw 10.250.228.129


取消vim自动缩进功能：
vim ~/.vimrc
set paste
```
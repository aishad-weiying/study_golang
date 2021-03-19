# kubernetes基础环境的搭建
k8s基础集群环境主要是运行kubernetes管理端服务以及node节点上的服务部署及使用。

## 单机master部署
![](images/fa3b5228534c1571bcd2bac953d616d0.png)

## 多master 部署
![](images/ede573106d739f3b4b0d4aefa558977a.png)

安装部署服务器统计
[table id=5 /]

## 基础环境的准备
```bash
ubuntu 安装后的准备工作

1. 更改网卡的命名方式为eth0
root@k8s-master1:~# vim /etc/default/gru
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"

root@k8s-master1:~# update-grub

2. 更改ip地址
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


7. 安装docker
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce
```

## 部署k8s的高可用环境

### 一：在ha1节点安装keepalived+haproxy

1. 安装并配置keepalived
```bash
root@k8s-ha1:~# apt install keepalived 

root@k8s-ha1:~# cp /usr/share/doc/keepalived/samples/keepalived.conf.sample /etc/keepalived/keepalived.conf
root@k8s-ha1:~# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    interface eth0
    virtual_router_id 50
    nopreempt
    priority 100
    advert_int 1
    virtual_ipaddress {
        192.168.7.248 dev eth0 label eth0:0
    }
}

root@k8s-ha1:~# systemctl start keepalived.service
root@k8s-ha1:~# systemctl enable keepalived.service
```

2. 安装并配置haproxy代理k8s api server
api端口：192.168.7.248：6443 #需要配置在负载均衡上实现反向代理，dashboard的端口为8443
```bash
root@k8s-ha1:~# apt install haproxy

root@k8s-master1:~# vim /etc/haproxy/haproxy.cfg
listen k8s-api-server-6443
  bind 192.168.7.248:6443
  mode tcp
  server 192.168.7.101 192.168.7.101:6443 check inter 2000 fall 3 rise 5

root@k8s-ha1:~# systemctl start haproxy
root@k8s-ha1:~# systemctl enable haproxy
```

## harbor节点安装并配置harbor https
```bash

root@k8s-harbor1:~# apt install docker-ce docker-compose

# 下载harbor离线安装包
root@k8s-harbor1:~# wget http://upload.aishad.top/harbor-offline-installer-v1.7.5.tgz -O /usr/local/src/harbor-offline-installer-v1.7.5.tgz
# 配置harbor
root@k8s-harbor1:/usr/local/src# tar -xvf harbor-offline-installer-v1.7.5.tgz
root@k8s-harbor1:/usr/local/src# ln -sv /usr/local/src/harbor /usr/local
root@k8s-harbor1:/usr/local/src# cd /usr/local/harbor
# 生成harbor要使用的ca证书
root@k8s-harbor1:/usr/local/harbor# mkdir certs/
root@k8s-harbor1:/usr/local/harbor#  openssl genrsa -out /usr/local/harbor/certs/harbor-ca.key
Generating RSA private key, 2048 bit long modulus (2 primes)
.....................+++++
...............................+++++
e is 65537 (0x010001)

root@k8s-harbor1:/usr/local/harbor# touch /root/.rnd
root@k8s-harbor1:/usr/local/harbor# openssl req -x509 -new -nodes -key /usr/local/harbor/certs/harbor-ca.key -subj "/CN=k8s-harbor1.weiying.net" -days 7120 -out /usr/local/harbor/certs/harborca.crt

#配置harbor
root@k8s-harbor1:/usr/local/harbor# vim harbor.cfg
hostname = k8s-harbor1.weiying.net
ui_url_protocol = https
ssl_cert = /usr/local/harbor/certs/harborca.crt
ssl_cert_key = /usr/local/harbor/certs/harbor-ca.key
harbor_admin_password = 123456

#安装harbor
root@k8s-harbor1:/usr/local/harbor# ./install.sh

# Windows修改hosts文件  192.168.7.103 k8s-harbor1.weiying.net
# 访问测试，新建baseimages项目
```

2. 在master和node节点上同步harbor证书
harbor证书主要是给master和node节点到harbor上拉取或上传镜像使用的，需要提前将证书文件分发到各个master和node节点上，这里暂时只上传到master1节点，后期从master节点批量分发
```bash
root@k8s-master1:~# mkdir /etc/docker/certs.d/k8s-harbor1.weiying.net -p
# 目录名称和证书域名相同

root@k8s-master1:~# scp 192.168.7.103:/usr/local/harbor/certs/harborca.crt /etc/docker/certs.d/k8s-harbor1.weiying.net/
# 添加hosts解析文件
root@k8s-master1:~# vim /etc/hosts

127.0.0.1 localhost
192.168.7.103 k8s-harbor1.weiying.net

# 重启docker测试
root@k8s-master1:~# systemctl restart docker
# 测试登录
root@k8s-master1:~# docker login k8s-harbor1.weiying.net
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

#测试上传镜像
root@k8s-master1:~# docker pull alpine
root@k8s-master1:~# docker tag alpine:latest k8s-harbor1.weiying.net/baseimages/alpine:v1
root@k8s-master1:~# docker push k8s-harbor1.weiying.net/baseimages/alpine:v1 
# 上传成功

```

# 部署kubernetes
具体参考：https://github.com/easzlab/kubeasz

## 使用ansible部署k8s

### ansible基础环境准备

1. 各个需要通过ansible部署的节点安装python2.7
```bash
apt-get install python2.7
ln -s /usr/bin/python2.7 /usr/bin/python

```

2. master节点安装ansible
在两个master节点都安装，这样两个节点都能操作
```bash
root@k8s-master1:~# apt-get  install ansible
```

3. 通过脚本做各个节点的免秘钥登录，并分发harbor证书，以及其他配置
```bash
root@k8s-master1:~# apt install  sshpass
# master 节点先生成密钥
root@k8s-master1:~# ssh-keygen 
root@k8s-master1:~# cat scp.sh 
#!/bin/bash
#目标主机列表
IP="
192.168.7.101
192.168.7.102
192.168.7.103
192.168.7.104
192.168.7.105
192.168.7.106
192.168.7.107
192.168.7.108
192.168.7.109
192.168.7.110
192.168.7.111
"
for node in ${IP};do
	sshpass -p admin123 ssh-copy-id ${node} -o StrictHostKeyChecking=no
	if [ $? -eq 0 ];then
		echo "${node} 秘钥copy完成"
		echo "${node} 秘钥copy完成,准备环境初始化....."
		ssh ${node} "mkdir /etc/docker/certs.d/k8s-harbor1.weiying.net -p"
		echo "Harbor 证书目录创建成功!"
		scp /etc/docker/certs.d/k8s-harbor1.weiying.net/harborca.crt ${node}:/etc/docker/certs.d/k8s-harbor1.weiying.net/harborca.crt
		echo "Harbor 证书拷贝成功!"
		scp /etc/hosts ${node}:/etc/hosts
		echo "host 文件拷贝完成"
		scp -r /root/.docker ${node}:/root/
		echo "Harbor 认证文件拷贝完成!"
		scp -r /etc/resolv.conf ${node}:/etc/
	else
		echo "${node} 秘钥copy失败"
	fi
done

root@k8s-master1:~# bash scp.sh
```
4. 到node节点测试能从harbor下载镜像
```bash
root@k8s-node1:~# docker login k8s-harbor1.weiying.net
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
root@k8s-node1:~# docker pull k8s-harbor1.weiying.net/baseimages/alpine:v1

# 测试成功
```

### 克隆ansible安装k8s项目
```bash
root@k8s-master1:~# git clone -b 0.6.1 https://github.com/easzlab/kubeasz.git
```

1. 准备ansible hosts文件
```bash
root@k8s-master1:~# mv /etc/ansible/hosts /opt/
root@k8s-master1:~# mv kubeasz/* /etc/ansible/
root@k8s-master1:~# cd /etc/ansible/
root@k8s-master1:/etc/ansible# cp example/hosts.m-masters.example ./hosts
root@k8s-master1:/etc/ansible# vim hosts
# 集群部署节点：一般为运行ansible 脚本的节点
# 变量 NTP_ENABLED (=yes/no) 设置集群是否安装 chrony 时间同步
[deploy]
192.168.7.101 NTP_ENABLED=no

# etcd集群请提供如下NODE_NAME，注意etcd集群必须是1,3,5,7...奇数个节点
[etcd]
192.168.7.105 NODE_NAME=etcd1
192.168.7.106 NODE_NAME=etcd2
192.168.7.107 NODE_NAME=etcd3

[new-etcd] # 预留组，后续添加etcd节点使用
#192.168.1.x NODE_NAME=etcdx

[kube-master]
192.168.7.101
#192.168.1.2

[new-master] # 预留组，后续添加master节点使用
#192.168.1.5

[kube-node]
192.168.7.110
#192.168.1.4

[new-node] # 预留组，后续添加node节点使用
#192.168.1.xx

# 参数 NEW_INSTALL：yes表示新建，no表示使用已有harbor服务器
# 如果不使用域名，可以设置 HARBOR_DOMAIN=""
[harbor]
#192.168.1.8 HARBOR_DOMAIN="harbor.yourdomain.com" NEW_INSTALL=no

# 负载均衡(目前已支持多于2节点，一般2节点就够了) 安装 haproxy+keepalived
#[lb]
#192.168.1.1 LB_ROLE=backup
#192.168.1.2 LB_ROLE=master

#【可选】外部负载均衡，用于自有环境负载转发 NodePort 暴露的服务等
#[ex-lb]
#192.168.1.6 LB_ROLE=backup EX_VIP=192.168.1.250
#192.168.1.7 LB_ROLE=master EX_VIP=192.168.1.250

[all:vars]
# ---------集群主要参数---------------
#集群部署模式：allinone, single-master, multi-master
DEPLOY_MODE=multi-master

#集群主版本号，目前支持: v1.8, v1.9, v1.10，v1.11, v1.12, v1.13
K8S_VER="v1.13"

# 集群 MASTER IP即 LB节点VIP地址，为区别与默认apiserver端口，设置VIP监听的服务端口8443
# 公有云上请使用云负载均衡内网地址和监听端口
MASTER_IP="192.168.7.248"
KUBE_APISERVER="https://{{ MASTER_IP }}:6443"

# 集群网络插件，目前支持calico, flannel, kube-router, cilium
CLUSTER_NETWORK="calico"

# 服务网段 (Service CIDR），注意不要与内网已有网段冲突
SERVICE_CIDR="10.50.0.0/16"

# POD 网段 (Cluster CIDR），注意不要与内网已有网段冲突
CLUSTER_CIDR="172.50.0.0/16"

# 服务端口范围 (NodePort Range)
NODE_PORT_RANGE="30000-60000"

# kubernetes 服务 IP (预分配，一般是 SERVICE_CIDR 中第一个IP)
CLUSTER_KUBERNETES_SVC_IP="10.50.0.1"

# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_DNS_SVC_IP="10.50.254.254"

# 集群 DNS 域名
CLUSTER_DNS_DOMAIN="weiying.local."

# 集群basic auth 使用的用户名和密码
BASIC_AUTH_USER="admin"
BASIC_AUTH_PASS="123456"

# ---------附加参数--------------------
#默认二进制文件目录
bin_dir="/usr/bin"

#证书目录
ca_dir="/etc/kubernetes/ssl"

#部署目录，即 ansible 工作目录，建议不要修改
base_dir="/etc/ansible"
```

2. 准备二进制文件
hosts文件中指定的集群环境为v1.13，那么需要准备k8s 1.13的二进制文件
```bash
root@k8s-master1:/etc/ansible/bin# pwd
/etc/ansible/bin
root@k8s-master1:/etc/ansible/bin# ls
k8s.1-13-5.tar.gz  readme.md
root@k8s-master1:/etc/ansible/bin# tar -xf k8s.1-13-5.tar.gz 
root@k8s-master1:/etc/ansible/bin# mv bin/* .
root@k8s-master1:/etc/ansible/bin# ls
bin             containerd       docker-init   host-local               kube-proxy
bridge          containerd-shim  docker-proxy  k8s.1-13-5.tar.gz        kube-scheduler
calicoctl       ctr              etcd          kube-apiserver           loopback
cfssl           docker           etcdctl       kube-controller-manager  portmap
cfssl-certinfo  docker-compose   flannel       kubectl                  readme.md
cfssljson       dockerd          helm          kubelet 
```

3. 初始化环境
```bash
root@k8s-master1:/etc/ansible# pwd
/etc/ansible
root@k8s-master1:/etc/ansible# ansible-playbook 01.prepare.yml

# 可以查看/etc/ansible/roles/prepare/tasks/main.yml，了解01.prepare 完成的任务
# 可以再/etc/ansible/roles/prepare/templates/ 目录下面的文件做响应的修改，包括内核参数、ulimits或者加在的模块，改了之后重新初始化

```

4. 部署etcd集群
可选更改启动脚本路径，实例替换etcd版本为3.2.24
```bash
root@k8s-master1:~# wget https://github.com/etcd-io/etcd/releases/download/v3.2.24/etcd-v3.2.24-linux-amd64.tar.gz
root@k8s-master1:~# tar -xvf etcd-v3.2.24-linux-amd64.tar.gz
root@k8s-master1:~# mv /etc/ansible/bin/etcd* /opt/
root@k8s-master1:~# cp etcd-v3.2.24-linux-amd64/etcd* /etc/ansible/bin/
root@k8s-master1:/etc/ansible# ansible-playbook 02.etcd.yml
```
也可以使用默认的3.3.10
```bash
root@k8s-master1:/etc/ansible# ansible-playbook 02.etcd.yml
```
各etcd服务器验证etcd服务
```bash
root@k8s-etcd1:~# export  NODE_IPS="192.168.7.105 192.168.7.106 192.168.7.107"

root@k8s-etcd1:~# for ip in ${NODE_IPS}; do ETCDCTL_API=3 /usr/bin/etcdctl --endpoints=https://${ip}:2379 --cacert=/etc/kubernetes/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem endpoint health; done
https://192.168.7.105:2379 is healthy: successfully committed proposal: took = 4.149904ms
https://192.168.7.106:2379 is healthy: successfully committed proposal: took = 3.164164ms
https://192.168.7.107:2379 is healthy: successfully committed proposal: took = 17.017976ms
```

5. 部署docker
可选更改启动脚本路径，但是docker已经提前安装，因此不需要重新执行
```bash
root@k8s-master1:/etc/ansible# ansible-playbook 03.docker.yml
```

6. 部署master
可选更改启动脚本路径
```bash
root@k8s-master1:/etc/ansible# ansible-playbook 04.kube-master.yml
```

7. 部署node
node节点必须安装docker
```bash
root@k8s-master1:/etc/ansible#  vim roles/kube-node/defaults/main.yml
SANDBOX_IMAGE: "mirrorgooglecontainers/pause-amd64:3.1"  #基础容器镜像，先下载下来，因为后期会经常使用，到公网上下载比较慢，下载后上传到本地harbor仓库，然后更改基础镜像地址
root@k8s-master1:/etc/ansible# docker pull mirrorgooglecontainers/pause-amd64:3.1
root@k8s-master1:/etc/ansible# docker tag mirrorgooglecontainers/pause-amd64:3.1 k8s-harbor1.weiying.net/baseimages/pause-amd64:3.1
root@k8s-master1:/etc/ansible# docker push k8s-harbor1.weiying.net/baseimages/pause-amd64:3.1

root@k8s-master1:/etc/ansible#  vim roles/kube-node/defaults/main.yml
SANDBOX_IMAGE: "k8s-harbor1.weiying.net/baseimages/pause-amd64:3.1"

# 部署node
root@k8s-master1:/etc/ansible# ansible-playbook 05.kube-node.yml 
```

8. 部署网络服务calico
可选更改calico服务启动脚本路径，csr证书信息
```bash
# k8s1-13-5版本支持使用calico3.3.6，下载calico3.3.6的镜像，使用默认的3.4.0也可以
root@k8s-master1:~# wget https://github.com/projectcalico/calico/releases/download/v3.3.6/release-v3.3.6.tgz

# 更改配置文件时间3.3.6
root@k8s-master1:/etc/ansible# vim roles/calico/defaults/main.yml
# 更新支持calico 版本: [v3.2.x] [v3.3.x] [v3.4.x]
calico_ver: "v3.3.6"
root@k8s-master1:~# tar -xvf  release-v3.3.6.tgz
root@k8s-master1:~# ls release-v3.3.6/images/
calico-cni.tar  calico-kube-controllers.tar  calico-node.tar  calico-typha.tar

#导入calico node、calico cni、calico-kube-controllers镜像到本地，并上传到harbor
root@k8s-master1:~/release-v3.3.6/images# docker tag calico/node:v3.3.6 k8s-harbor1.weiying.net/baseimages/calico-node:v3.3.6
root@k8s-master1:~/release-v3.3.6/images# docker push k8s-harbor1.weiying.net/baseimages/calico-node:v3.3.6

root@k8s-master1:~/release-v3.3.6/images# docker load -i calico-cni.tar 
root@k8s-master1:~/release-v3.3.6/images# docker tag calico/cni:v3.3.6  k8s-harbor1.weiying.net/baseimages/calico-cni:v3.3.6
root@k8s-master1:~/release-v3.3.6/images# docker push k8s-harbor1.weiying.net/baseimages/calico-cni:v3.3.6

root@k8s-master1:~/release-v3.3.6/images# docker load -i calico-kube-controllers.tar 
root@k8s-master1:~/release-v3.3.6/images# docker tag calico/kube-controllers:v3.3.6 k8s-harbor1.weiying.net/baseimages/calico-kube-controllers
root@k8s-master1:~/release-v3.3.6/images# docker push k8s-harbor1.weiying.net/baseimages/calico-kube-controllers


# 更改calico3.3的配置文件，该文件中定义了calico是什么运行的，指定对应的网络插件镜像的位置
root@k8s-master1:/etc/ansible# vim roles/calico/templates/calico-v3.3.yaml.j2
- name: calico-node
          image: k8s-harbor1.weiying.net/baseimages/calico-node:v3.3.6

- name: install-cni
          image: k8s-harbor1.weiying.net/baseimages/calico-cni:v3.3.6

- name: calico-kube-controllers
          image: k8s-harbor1.weiying.net/baseimages/calico-kube-controllers

# 替换3.3.6版本的calicoctl文件
root@k8s-master1:~# mv /etc/ansible/bin/calicoctl /opt/
root@k8s-master1:~# cp release-v3.3.6/bin/calicoctl /etc/ansible/bin/calicoctl 

# 部署calico
root@k8s-master1:/etc/ansible# ansible-playbook 06.network.yml
```
验证calico
```bash
root@k8s-master1:/etc/ansible# calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.7.110 | node-to-node mesh | up    | 08:57:24 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

9. 添加node节点
```bash
# 已经安装了docker，不需要安装docker，注释掉
root@k8s-master1:/etc/ansible# vim 20.addnode.yml
- docker # 将这一行注释或者删除

# 在hosts文件中添加node节点
root@k8s-master1:/etc/ansible# vim hosts
[new-node] # 预留组，后续添加node节点使用
192.168.7.111

# 添加node节点
root@k8s-master1:/etc/ansible# ansible-playbook 20.addnode.yml

# 查看node信息
root@k8s-master1:/etc/ansible# calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.7.110 | node-to-node mesh | up    | 08:57:24 | Established |
| 192.168.7.111 | node-to-node mesh | up    | 09:05:55 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

root@k8s-master1:/etc/ansible# kubectl get node
NAME            STATUS                     ROLES    AGE   VERSION
192.168.7.101   Ready,SchedulingDisabled   master   59m   v1.13.5
192.168.7.110   Ready                      node     45m   v1.13.5
192.168.7.111   Ready                      node     84s   v1.13.5
```

10. 添加master
```bash
# 已经安装了docker，不需要安装docker，注释掉
root@k8s-master1:/etc/ansible# vim 21.addmaster.yml
- docker # 将这一行注释或者删除

# 在hosts文件中添加master节点
root@k8s-master1:/etc/ansible# vim hosts
[new-master] # 预留组，后续添加master节点使用
192.168.7.102

# 添加master
root@k8s-master1:/etc/ansible# ansible-playbook 21.addmaster.yml

# 验证节点信息
root@k8s-master1:/etc/ansible# calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.7.110 | node-to-node mesh | up    | 08:57:23 | Established |
| 192.168.7.111 | node-to-node mesh | up    | 09:05:54 | Established |
| 192.168.7.102 | node-to-node mesh | up    | 09:15:14 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

root@k8s-master1:/etc/ansible# kubectl get node
NAME            STATUS                     ROLES    AGE    VERSION
192.168.7.101   Ready,SchedulingDisabled   master   69m    v1.13.5
192.168.7.102   Ready,SchedulingDisabled   master   2m8s   v1.13.5
192.168.7.110   Ready                      node     55m    v1.13.5
192.168.7.111   Ready                      node     11m    v1.13.5
```
配置haproxy代理两个master
```bash
root@k8s-ha1:~# vim /etc/haproxy/haproxy.cfg 
listen k8s-api-server-6443
  bind 192.168.7.248:6443
  mode tcp
  server 192.168.7.101 192.168.7.101:6443 check inter 2000 fall 3 rise 5
  server 192.168.7.102 192.168.7.102:6443 check inter 2000 fall 3 rise 5

root@k8s-ha1:~# systemctl restart haproxy.service 
```

### 验证不同node节点的容器的网络能够通信
```bash
# 启动4个pod测试
root@k8s-master1:/etc/ansible# kubectl run net-test1 --image=alpine --replicas=4 sleep 360000
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/net-test1 created

root@k8s-master1:/etc/ansible# kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
net-test1-68898c85b7-224j6   1/1     Running   0          47s
net-test1-68898c85b7-kwjm5   1/1     Running   0          47s
net-test1-68898c85b7-lm9sx   1/1     Running   0          47s
net-test1-68898c85b7-nwrl9   1/1     Running   0          47s

# 测试
root@k8s-master1:/etc/ansible# kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
net-test1-68898c85b7-224j6   1/1     Running   0          93s   172.50.120.2     192.168.7.110   <none>           <none>
net-test1-68898c85b7-kwjm5   1/1     Running   0          93s   172.50.170.194   192.168.7.111   <none>           <none>
net-test1-68898c85b7-lm9sx   1/1     Running   0          93s   172.50.170.193   192.168.7.111   <none>           <none>
net-test1-68898c85b7-nwrl9   1/1     Running   0          93s   172.50.120.1     192.168.7.110   <none>           <none>

# 使用node1上的pod测试是否与node2上的pod能够通信
root@k8s-master1:/etc/ansible# kubectl exec -it net-test1-68898c85b7-224j6 sh
/ # ping 172.50.170.194
PING 172.50.170.194 (172.50.170.194): 56 data bytes
64 bytes from 172.50.170.194: seq=0 ttl=62 time=8.444 ms
64 bytes from 172.50.170.194: seq=1 ttl=62 time=0.524 ms
64 bytes from 172.50.170.194: seq=2 ttl=62 time=0.459 ms
^C
--- 172.50.170.194 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.459/3.142/8.444 ms

# 测试成功
```

## k8s dashboard 部署
部署kubernetes的web管理界面dashboard

1. 导入dashboard镜像并上传至本地harbor服务器
```bash
root@k8s-master1:~# mkdir /etc/ansible/manifests/dashboard/1.10.1
root@k8s-master1:~# cd /etc/ansible/manifests/dashboard/1.10.1
# 下载镜像
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1

# 打tag号并上传到harbor
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1 k8s-harbor1.weiying.net/baseimages/kubernetes-dashboard-amd64:v1.10.1

root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# docker push k8s-harbor1.weiying.net/baseimages/kubernetes-dashboard-amd64:v1.10.1
```

2. 修改yaml文件中的dashboard镜像地址为本地harbor地址
```bash
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# cp ../*.yaml .
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# ls
admin-user-sa-rbac.yaml    read-user-sa-rbac.yaml  ui-read-rbac.yaml
kubernetes-dashboard.yaml  ui-admin-rbac.yaml

root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# vim kubernetes-dashboard.yaml
    spec:
      containers:
      - name: kubernetes-dashboard
        #image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
        image: k8s-harbor1.weiying.net/baseimages/kubernetes-dashboard-amd64:v1.10.1
```

3. 创建服务
```bash
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# kubectl apply -f .
```

4. 验证dashboard启动完成
```bash
root@k8s-master1:/etc/ansible# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-58f9cc8c94-9sjpx   1/1     Running   0          56m
calico-node-4fdg8                          2/2     Running   0          38m
calico-node-dd7d5                          2/2     Running   0          56m
calico-node-v6vmc                          2/2     Running   0          47m
calico-node-wqkc4                          2/2     Running   0          56m
kubernetes-dashboard-6784f98d57-h29m9      1/1     Running   0          77s

root@k8s-master1:/etc/ansible# kubectl get svc -n kube-system
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.50.107.207   <none>        443:48664/TCP   106s

# 获取dashboard的访问地址
root@k8s-master1:/etc/ansible# kubectl cluster-info
Kubernetes master is running at https://192.168.7.248:6443
kubernetes-dashboard is running at https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
登录访问测试
![](images/59fad42f442076b229fc27fa1db71762.png)
![](images/753cde06bc3c1418b2190f841fd308cf.png)
获取令牌
```bash
root@k8s-master1:/etc/ansible# kubectl -n kube-system get secret | grep admin-user
admin-user-token-4dztg                kubernetes.io/service-account-token   3      6m31s
root@k8s-master1:/etc/ansible#  kubectl -n kube-system describe secret admin-user-token-4dztg
Name:         admin-user-token-4dztg
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 6b53cb6c-a3c1-11e9-9a72-000c29d1bdd3

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1346 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTRkenRnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YjUzY2I2Yy1hM2MxLTExZTktOWE3Mi0wMDBjMjlkMWJkZDMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.oTxcRoakmwIG745vQ1w4yc0BuFkQjbogONBaNd_IWyq8h6t7zwnpInpPK2reacySV36baZooRpPzWlZ_Vi2JwonTe7tJr_3_b1x9JbYfweOTeUnGvriAVeqRrewhutBuh7hoZbEtCWkAYfpBepkLGYjhkgtAQ01Hwuk3mdDuzq1yTNNvTp5jgJLTZbHe9oZqBVWfNKziG1_i_3O86qoKLboXILOPf9_mDnhxUB01LQh7T5PIUntoGVR2yYdTqJKsR0MvGAOanHM_wi1S4rkT_nd-IWquu-yKnZlQZUggx6bOV12jbppu7qpgKv6XxrrK8HdNgup_UUK_zSUWw-5JPA
```
输入令牌登录
![](images/ab4be4e0a26f87a40bcacf2213718ad0.png)

#### 使用Kubeconfig登录

1. 设置token的登录会话保持时间
```bash
root@k8s-master1:~# vim  /etc/ansible/manifests/dashboard/1.10.1/kubernetes-dashboard.yaml
spec:
      containers:
      - name: kubernetes-dashboard
        #image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
        image: k8s-harbor1.weiying.net/baseimages/kubernetes-dashboard-amd64:v1.10.1
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          - --token-ttl=43200 # 添加超时时间

root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# kubectl apply -f .
```
> 如果创建服务的时候使用的是kubectl create -f .命令，那么改完配置后需要删除后创建(删除命令kubectl delete -f . )会导致登录token发生变化，要重新获取,使用apply创建的服务更改配置文件后，重新apply只会应用更改的部分

web界面查看更改成功
![](images/fe7cf4ed14edf5f5f090af29c372b752.png)

2. 设置Kubeconfig登录
制作Kubeconfig文件
```bash
root@k8s-master1:~# cp /root/.kube/config /opt/kubeconfig
# 在文件最后一行加入token信息，前面有四个空格,token 和密钥之间也有一个空格
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTRkenRnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YjUzY2I2Yy1hM2MxLTExZTktOWE3Mi0wMDBjMjlkMWJkZDMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.oTxcRoakmwIG745vQ1w4yc0BuFkQjbogONBaNd_IWyq8h6t7zwnpInpPK2reacySV36baZooRpPzWlZ_Vi2JwonTe7tJr_3_b1x9JbYfweOTeUnGvriAVeqRrewhutBuh7hoZbEtCWkAYfpBepkLGYjhkgtAQ01Hwuk3mdDuzq1yTNNvTp5jgJLTZbHe9oZqBVWfNKziG1_i_3O86qoKLboXILOPf9_mDnhxUB01LQh7T5PIUntoGVR2yYdTqJKsR0MvGAOanHM_wi1S4rkT_nd-IWquu-yKnZlQZUggx6bOV12jbppu7qpgKv6XxrrK8HdNgup_UUK_zSUWw-5JPA
# 将这个文件导出，使用这个文件登录
```
![](images/6cfe8cd3d2a4c8a78ec4bace41f587b3.png)

## DNS服务
目前常用的dns组件有kube-dns和coredns两个

### kube-dns
kube-dns：提供service name域名的解析
dns-dnsmasq：提供DNS缓存，降低kubedns负载，提高性能
dns-sidecar：定期检查kubedns和dnsmasq的健康状态

1. 部署kube-dns
```bash
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# mkdir /etc/ansible/manifests/dns
root@k8s-master1:/etc/ansible/manifests/dashboard/1.10.1# cd /etc/ansible/manifests/dns
root@k8s-master1:/etc/ansible/manifests/dns# mkdir kube-dns
root@k8s-master1:/etc/ansible/manifests/dns# mkdir coredns
root@k8s-master1:/etc/ansible/manifests/dns# cd kube-dns/

# 上传需要的文件 可到http://upload.aishad.top/kube-dns/kube-dns.zip，下载后解压到kube-dns目录即可
```
2. 导入镜像并上传至本地harbor
```bash
# 导入busybox镜像
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker load -i busybox-online.tar.gz
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker tag quay.io/prometheus/busybox:latest k8s-harbor1.weiying.net/baseimages/busybox:latest
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker push  k8s-harbor1.weiying.net/baseimages/busybox:latest
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# vim busybox.yaml
- image: k8s-harbor1.weiying.net/baseimages/busybox:latest
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# kubectl apply -f busybox.yaml 
pod/busybox created

# 修改配置文件
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# vim kube-dns.yaml
clusterIP: 10.50.254.254  # 这个ip地址为在hosts文件中指定的dns
- --domain=weiying.local. # 这个域名为在hosts文件中指定的域名
- --server=/weiying.local/127.0.0.1#10053
- --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.weiying.local,5,SRV
- --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.weiying.local,5,SRV

# 导入k8s-dns-kube-dns-amd64镜像
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker load -i k8s-dns-kube-dns-amd64_1.14.13.tar.gz
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker tag gcr.io/google-containers/k8s-dns-kube-dns-amd64:1.14.13 k8s-harbor1.weiying.net/baseimages/k8s-dns-kube-dns-amd64:1.14.13
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker push k8s-harbor1.weiying.net/baseimages/k8s-dns-kube-dns-amd64:1.14.13

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# vim kube-dns.yaml
 - name: kubedns
        image: k8s-harbor1.weiying.net/baseimages/k8s-dns-kube-dns-amd64:1.14.13

# 导入k8s-dns-dnsmasq-nanny-amd64镜像
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker load -i k8s-dns-dnsmasq-nanny-amd64_1.14.13.tar.gz
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker tag gcr.io/google-containers/k8s-dns-dnsmasq-nanny-amd64:1.14.13 k8s-harbor1.weiying.net/baseimages/k8s-dns-dnsmasq-nanny-amd64:1.14.13
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker push k8s-harbor1.weiying.net/baseimages/k8s-dns-dnsmasq-nanny-amd64:1.14.13

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# vim kube-dns.yaml
- name: dnsmasq
        image: k8s-harbor1.weiying.net/baseimages/k8s-dns-dnsmasq-nanny-amd64:1.14.13

# 导入k8s-dns-sidecar-amd64镜像
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker load -i k8s-dns-sidecar-amd64_1.14.13.tar.gz
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker tag gcr.io/google-containers/k8s-dns-sidecar-amd64:1.14.13 k8s-harbor1.weiying.net/baseimages/k8s-dns-sidecar-amd64:1.14.13
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# docker push k8s-harbor1.weiying.net/baseimages/k8s-dns-sidecar-amd64:1.14.13

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# vim kube-dns.yaml
 name: sidecar
        image: k8s-harbor1.weiying.net/baseimages/k8s-dns-sidecar-amd64:1.14.13

```
3. 创建服务
```bash
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns# kubectl apply -f kube-dns.yaml
```

4. 查看pod状态
```bash
root@k8s-master1:/etc/ansible# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-58f9cc8c94-9sjpx   1/1     Running   0          113m
calico-node-4fdg8                          2/2     Running   0          95m
calico-node-dd7d5                          2/2     Running   0          113m
calico-node-v6vmc                          2/2     Running   0          104m
calico-node-wqkc4                          2/2     Running   0          113m
kube-dns-84856d555c-d64q6                  3/3     Running   0          2m9s
kubernetes-dashboard-6784f98d57-h29m9      1/1     Running   0          58m
```

5. 解析测试
```bash
root@k8s-master1:/etc/ansible# kubectl exec busybos nslookup kubernetes
Error from server (NotFound): pods "busybos" not found
root@k8s-master1:/etc/ansible# kubectl exec busybox nslookup kubernetes
Server:    10.50.254.254
Address 1: 10.50.254.254 kube-dns.kube-system.svc.weiying.local

Name:      kubernetes
Address 1: 10.50.0.1 kubernetes.default.svc.weiying.local
     #域名的组成部分   server的名称.namespace的名称.svc.后缀.
```

6. web界面查看
![](images/6fd80fa7ee4b0a1e1a220097b600c701.png)

## 监控组件heapster
heapster：数据采集
influxdb：数据存储
grafana：web展示

1. 导入相应的镜像并更改yaml中的镜像地址
```bash
# 导入heapster并上传
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker load -i heapster-amd64_v1.5.1.tar
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker tag gcr.io/google-containers/heapster-amd64:v1.5.1 k8s-harbor1.weiying.net/baseimages/heapster-amd64:v1.5.1
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker push k8s-harbor1.weiying.net/baseimages/heapster-amd64:v1.5.1

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# vim heapster.yaml 
image: k8s-harbor1.weiying.net/baseimages/heapster-amd64:v1.5.1

# 导入grafana并上传
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker load -i heapster-grafana-amd64-v4.4.3.tar 
Loaded image ID: sha256:8cb3de219af7bdf0b3ae66439aecccf94cebabb230171fa4b24d66d4a786f4f7
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker tag 8cb3de219af7bdf0b3 k8s-harbor1.weiying.net/baseimages/heapster-grafana-amd64-v4.4.3
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker push k8s-harbor1.weiying.net/baseimages/heapster-grafana-amd64-v4.4.3

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# vim grafana.yaml
image: k8s-harbor1.weiying.net/baseimages/heapster-grafana-amd64-v4.4.3


# 导入influxdb镜像并上传
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker load -i heapster-influxdb-amd64_v1.3.3.tar 
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker tag gcr.io/google-containers/heapster-influxdb-amd64:v1.3.3 k8s-harbor1.weiying.net/baseimages/heapster-influxdb-amd64:v1.3.3
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# docker push k8s-harbor1.weiying.net/baseimages/heapster-influxdb-amd64:v1.3.3

root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# vim influxdb.yaml
image: k8s-harbor1.weiying.net/baseimages/heapster-influxdb-amd64:v1.3.3
```
2. 创建服务
```bash
root@k8s-master1:/etc/ansible/manifests/dns/kube-dns/heapster# kubectl create -f.
```

3. 查看grafana的界面
```bash
root@k8s-master1:~# kubectl cluster-info
Kubernetes master is running at https://192.168.7.248:6443
KubeDNS is running at https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
monitoring-grafana is running at https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://192.168.7.248:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
![](images/f38b082ba721ba1c491ed9b0840d8d88.png)

4. dashboard查看
![](images/222ba5e6b88d45c0de91419d1f7e4d8f.png)
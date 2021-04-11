# 网络服务neutron
neutron 是 openstack 的网络组件，是 OpenStack 的网络服务，网络服务提供网络，子网以及路由这些对象的抽象概念。每个抽象概念都有自己的功能，可以模拟对应的物理设备：网络包括子网，路由在不同的子网和网络间进行路由转发。

对于任意一个给定的网络都必须包含至少一个外部网络。不像其他的网络那样，外部网络不仅仅是一个定义的虚拟网络。相反，它代表了一种OpenStack安装之外的能从物理的，外部的网络访问的视图。外部网络上的IP地址可供外部网络上的任意的物理设备所访问

外部网络之外，任何 Networking 设置拥有一个或多个内部网络。这些软件定义的网络直接连接到虚拟机。仅仅在给定网络上的虚拟机，或那些在通过接口连接到相近路由的子网上的虚拟机，能直接访问连接到那个网络上的虚拟机。

如果外部网络想要访问实例或者相反实例想要访问外部网络，那么网络之间的路由就是必要的了。每一个路由都配有一个网关用于连接到外部网络，以及一个或多个连接到内部网络的接口。就像一个物理路由一样，子网可以访问同一个路由上其他子网中的机器，并且机器也可以访问路由的网关访问外部网络。

另外，你可以将外部网络的IP地址分配给内部网络的端口。不管什么时候一旦有连接连接到子网，那个连接被称作端口。你可以给实例的端口分配外部网络的IP地址。通过这种方式，外部网络上的实体可以访问实例.

- 网络
在显示的网络环境中我们使用交换机将多个计算机连接起来从而形成了网络，而在neutron 的环境里，网络的功能也是将多个不同的云主机连接起来。

- 子网
是现实的网络环境下可以将一个网络划分成多个逻辑上的子网络，从而实现网络隔离，在 neutron 里面子网也是属于网络。

- 端口
计算机连接交换机通过网线连，而网线插在交换机的不同端口，在 neutron 里面端口属于子网，即每个云主机的子网都会对应到一个端口。

- 路由器
用于连接不通的网络或者子网。

## 网络类型

提供者网络：虚拟机桥接到物理机，并且虚拟机必须和物理机在同一个网络范围内。(比较常用，要做好网络规划，否则如果ip地址不够用的话，将不能创建虚拟机)

自服务网络：可以自己创建网络，最终会通过虚拟路由器连接外网(用这种情况，太多的虚拟机要出去的话，虚拟路由器会是瓶颈)

## 消息队列
大多数的OpenStack Networking安装都会用到，用于在neutron-server和各种各样的代理进程间路由信息。也为某些特定的插件扮演数据库的角色，以存储网络状态

OpenStack网络主要和OpenStack计算交互，以提供网络连接到它的实例

网络服务同样支持安全组。安全组允许管理员在安全组中定义防火墙规则。一个实例可以属于一个或多个安全组，网络为这个实例配置这些安全组中的规则，阻止或者开启端口，端口范围或者通信类型。

每一个Networking使用的插件都有其自有的概念。虽然对操作VNI和OpenStack环境不是至关重要的，但理解这些概念能帮助你设置Networking。所有的Networking安装使用了一个核心插件和一个安全组插件(或仅是空操作安全组插件)。另外，防火墙即服务(FWaaS)和负载均衡即服务(LBaaS)插件是可用的。

# 部署neutron网络服务

## 为neutron创建数据库并授权
```bash
MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.036 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'admin123';
Query OK, 0 rows affected (0.926 sec)
```

### 在控制端创建 neutron 用户并设置密码为 neutron 
```bahs
[ root@openstack1 ~]# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | 9cfc38258eae490aae898f778e798c5d |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 对neutron用户授权
将 neutron 用户授权为 service 项目的 admi 权限
```bash
[ root@openstack1 ~]# openstack role add --project service --user neutron admin
```

## 控制端创建neutron服务并注册
- 创建 neutron 服务实体
```bash
[ root@openstack1 ~]# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | a086c72b50eb4e61949c251810c1cd99 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

- 创建网络服务API端点
```bash
[ root@openstack1 ~]# openstack endpoint create --region RegionOne network public http://192.168.7.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 630eb556df1842ebaffe181f8c71e690 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a086c72b50eb4e61949c251810c1cd99 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.7.248:9696        |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne network internal http://192.168.7.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3fcaab4656374a029d9e6bf65381df29 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a086c72b50eb4e61949c251810c1cd99 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.7.248:9696        |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne network admin http://192.168.7.248:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fe1aae6c74d1401bab91fa15ff1577b0 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a086c72b50eb4e61949c251810c1cd99 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://192.168.7.248:9696        |
+--------------+----------------------------------+
```

## 配置HAProxy
```bash
listen  openstack_neutron
 bind 192.168.7.248:9696
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:9696  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:9696  check inter 3000 fall 2 rise 5 backup
```
# 配置网络选项
可以部署自服务网络或者提供者网络

## 部署提供者网络

### 控制端安装 neutron
```bash
[ root@openstack1 ~]# yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
```

### 编辑/etc/neutron/neutron.conf 配置文件
```bash
#在 [database] 部分，配置数据库访问：
[database]
connection = mysql+pymysql://neutron:admin123@192.168.7.248/neutron

[ root@openstack1 ~]# grep -n "^[a-Z\[]"  /etc/neutron/neutron.conf
1:[DEFAULT]
#启用ML2插件并禁用其他插件
2:core_plugin = ml2
3:service_plugins =
#配置RabbitMQ消息队列访问权限
4:transport_url = rabbit://openstack:admin123@192.168.2.101
5:auth_strategy = keystone
#配置网络服务来通知计算节点的网络拓扑变化
6:notify_nova_on_port_status_changes = true
7:notify_nova_on_port_data_changes = true

737 [database]
738 connection = mysql+pymysql://neutron:admin123@172.20.45.199/neutron

852:[keystone_authtoken]
853:auth_uri = http://192.168.7.248:5000
854:auth_url = http://192.168.7.248:35357
855:memcached_servers = 192.168.7.248:11211
856:auth_type = password
857:project_domain_name = default
858:user_domain_name = default
859:project_name = service
860:username = neutron
861:password = neutron 

#配置网络服务来通知计算节点的网络拓扑变化
1081:[nova]
1082:auth_url = http://192.168.7.248:35357
1083:auth_type = password
1084:project_domain_name = default
1085:user_domain_name = default
1086:region_name = RegionOne
1087:project_name = service
1088:username = nova
1089:password = nova

1188:[oslo_concurrency]
1203:lock_path = $state_path/lock
```
#### 配置 Modular Layer 2 (ML2) 插件(创建桥接网卡)
ML2 插件使用 Linuxbridge 机制来为实例创建 layer－2 虚拟网络基础设施

编辑/etc/neutron/plugins/ml2/ml2_conf.ini 配置文件
```bash
[ root@openstack1 ~]# grep -n "^[a-Z\[]"  /etc/neutron/plugins/ml2/ml2_conf.ini
1:[DEFAULT]
113:[ml2]
#启用flat和VLAN网络
115:type_drivers = flat,vlan
#禁用私有网络
116:tenant_network_types =
#启用Linuxbridge机制
117:mechanism_drivers = linuxbridge
#启用端口安全扩展驱动
118:extension_drivers = port_security
#配置桥接的网络名称，就是openstack网络要桥接到哪个物理网卡
166:[ml2_type_flat]
168:flat_networks = internal
#启用 ipset 增加安全组的方便性
#ipset：iptables的一个扩展，它允许创建同时匹配整个IP地址“集（集）”的防火墙规则。这些地址集存在于已进行了索引的数据结构中，从而可以提高效率，特别是在有大量规则的系统中。
237:[securitygroup]
238:enable_ipset = true
```

#### 配置Linuxbridge代理
Linuxbridge代理为实例建立layer－2虚拟网络并且处理安全组规则。

编辑 /etc/neutron/plugins/ml2/linuxbridge_agent.ini 配置文件
```bash
[ root@openstack1 ~]# grep -n "^[a-Z\[]"  /etc/neutron/plugins/ml2/linuxbridge_agent.ini
#将公共虚拟网络和公共物理网络接口对应起来
144:[linux_bridge]
145:physical_interface_mappings = internal:br0
# internal 是个网络名称，要与在/etc/neutron/plugins/ml2/ml2_conf.ini配置文件中flat_networks字段定义的名称相同
#br0为底层的物理公共网络接口

#启用安全组并配置 Linux 桥接 iptables 防火墙驱动，开启之后会经过安全组的审核，如果不是设置在安全组中的规则，直接拒绝，这种方式性能比较差
162:[securitygroup]
163:enable_security_group = true
164:firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver #防火墙的驱动文件
#禁止VXLAN覆盖网络
183:[vxlan]
184:enable_vxlan = false
```

#### 配置DHCP代理
该DHCP代理为虚拟网络提供DHCP服务

编辑/etc/neutron/dhcp_agent.ini配置文件
```bash
[ root@openstack1 ~]# cat /etc/neutron/dhcp_agent.ini
#配置Linuxbridge驱动接口，DHCP驱动并启用隔离元数据，这样在公共网络上的实例就可以通过网络来访问元数据
[DEFAULT]
interface_driver = linuxbridge #桥接
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq #dhcp驱动文件
enable_isolated_metadata = true #开启元数据
```

## 配置元数据代理
要访问元数据的地址和密码
编辑/etc/neutron/metadata_agent.ini文件

```bash
#配置元数据主机以及共享密码
[DEFAULT]
nova_metadata_ip = 192.168.7.248
metadata_proxy_shared_secret = 20190621
```
## 配置 nova 调用 neutron
```bash
[ root@openstack1 ~]# vim /etc/nova/nova.conf
[neutron]
url = http://192.168.7.248:9696
auth_url = http://192.168.7.248:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = 20190621
```

### 创建软连接
网络服务初始化脚本需要一个超链接 /etc/neutron/plugin.ini 指向ML2插件配置文件/etc/neutron/plugins/ml2/ml2_conf.ini
```bash
[ root@openstack1 ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

## 初始化数据库
```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

### 启 nova API 服务
```bash
[ root@openstack1 ~]# systemctl restart openstack-nova-api.service
```

### 配置 haproxy 代理
```bash
listen  openstack_nova_api
 bind 192.168.7.248:8775
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:8775  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:8775  check inter 3000 fall 2 rise 5 backup
```

## 启动 neutron 服务并设置为开机启动
```bash
[ root@openstack1 ~]# systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service

[ root@openstack1 ~]# systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

### 验证 neutron 控制端是否注册成功
```bash
[ root@openstack1 ~]# neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host       | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
| 4421bde7-e2fe-479b-b788-cd14a8f5a76b | Metadata agent     | openstack1 |                   | :-)   | True           | neutron-metadata-agent    |
| 7e1d4a5c-37d7-449a-b30a-028d222764f8 | DHCP agent         | openstack1 | nova              | :-)   | True           | neutron-dhcp-agent        |
| 997a5a4b-f7d8-4c6e-b389-af8cbfcf1ebd | Linux bridge agent | openstack1 |                   | :-)   | True           | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
```

### neutron 控制端重启脚本
```bash
#!/bin/bash
systemctl restart openstack-nova-api.service neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

# 部署neutron计算节点

计算节点安装组件
```bash
[ root@node1 ~]# yum install openstack-neutron-linuxbridge ebtables ipset -y
```

## 编辑/etc/neutron/neutron.conf 配置文件
```bash
[ root@node1 ~]# grep -n "^[a-Z\[]" /etc/neutron/neutron.conf
[ root@node1 ~]# grep -n "^[a-Z\[]" /etc/neutron/neutron.conf
1:[DEFAULT]
2:transport_url = rabbit://openstack:admin123@192.168.2.101
3:auth_strategy = keystone

847:[keystone_authtoken]
848:auth_uri = http://192.168.7.248:5000
849:auth_url = http://192.168.7.248:35357
850:memcached_servers = 192.168.7.248:11211
851:auth_type = password
852:project_domain_name = default
853:user_domain_name = default
854:project_name = service
855:username = neutron
856:password = neutron

1175:[oslo_concurrency]
1190:lock_path = /var/lib/neutron/tmp
```

### 提供者网络配置

#### 配置Linuxbridge代理

编辑 /etc/neutron/plugins/ml2/linuxbridge_agent.ini 配置文件
```bash
[ root@node1 ~]# grep -n "^[a-Z\[]" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
144:[linux_bridge]
146:physical_interface_mappings = internal:br0 #internal为在控制端固定写好的，后面的网卡设备根据服务器情况写
162:[securitygroup]
163:enable_security_group = true
164:firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
183:[vxlan]
184:enable_vxlan = false
```

### 配置 nova  /etc/nova/nova.conf调用使用网络
在[neutron] 部分，配置访问参数：
```bash
[neutron]
url = http://192.168.7.248:9696
auth_url = http://192.168.7.248:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```
#### 重启 nova 服务
```bash
[ root@node1 ~]# systemctl restart openstack-nova-compute.service
```


#### 启动 neutron 并设置为开机启动
```bash
[ root@node1 ~]# systemctl enable neutron-linuxbridge-agent.service
[ root@node1 ~]# systemctl start neutron-linuxbridge-agent.service
```

## neutron 控制端验证计算节点是否注册成功
```bash
[ root@openstack1 ~]# neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host       | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
| 28a1d454-a02d-4602-bcd6-f8360aa08ff9 | Linux bridge agent | node2      |                   | :-)   | True           | neutron-linuxbridge-agent |
| 4421bde7-e2fe-479b-b788-cd14a8f5a76b | Metadata agent     | openstack1 |                   | :-)   | True           | neutron-metadata-agent    |
| 44fd9a24-a171-490f-8c51-ae9ff9f9db44 | Linux bridge agent | node1      |                   | :-)   | True           | neutron-linuxbridge-agent |
| 7e1d4a5c-37d7-449a-b30a-028d222764f8 | DHCP agent         | openstack1 | nova              | :-)   | True           | neutron-dhcp-agent        |
| 997a5a4b-f7d8-4c6e-b389-af8cbfcf1ebd | Linux bridge agent | openstack1 |                   | :-)   | True           | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+----------------+---------------------------+
```

# 继续查看下一篇博客：[仪表盘的安装](http://aishad.top/wordpress/?p=378 "仪表盘的安装")
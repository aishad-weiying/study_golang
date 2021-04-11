# 部署nova 计算节点
计算服务支持几种不同的 hypervisors。为了简单起见，这个配置在计算节点上使用 :term:`KVM <kernel-based VM (KVM)>`扩展的:term:`QEMU <Quick EMUlator (QEMU)>`作为hypervisor，支持虚拟机的硬件加速。在旧的硬件上，这个配置使用通用的QEMU作为hypervisor。

## 安装nova计算服务
```bash
[ root@node1 ~]# yum install openstack-nova-compute
```

### 修改配置文件/etc/nova/nova.conf
```bash
[ root@node2 ~]# grep -n "^[a-Z\[]" /etc/nova/nova.conf
1:[DEFAULT]
2:enabled_apis = osapi_compute,metadata
3:transport_url = rabbit://openstack:admin123@192.168.2.101
4:use_neutron = True
5:firewall_driver = nova.virt.firewall.NoopFirewallDriver

3074:[api]
3090:auth_strategy=keystone

4942:[glance]
4943:api_servers = http://192.168.7.248:9292

5602:[keystone_authtoken]
5604:auth_uri = http://192.168.7.248:5000
5605:auth_url = http://192.168.7.248:35357
5606:memcached_servers = 192.168.7.248:11211
5607:auth_type = password
5608:project_domain_name = default
5609:user_domain_name = default
5610:project_name = service
5611:username = nova
5612:password = nova

7303:[oslo_concurrency]
7318:lock_path=/var/lib/nova/tmp

8149:[placement]
8150:os_region_name = RegionOne
8151:project_domain_name = Default
8152:project_name = service
8153:auth_type = password
8154:user_domain_name = Default
8155:auth_url = http://192.168.7.248:35357/v3
8156:username = placement
8157:password = placement

9737:[vnc]
9738:enabled = True
9739:vncserver_listen = 0.0.0.0
9740:vncserver_proxyclient_address = 192.168.3.101
9741:novncproxy_base_url = http://192.168.7.248:6080/vnc_auto.html #如果此处是vip的地址，那么haproxy要代理6080的端口
```

#### 确认计算节点是否支持硬件加速
```bash
[ root@node2 ~]# egrep -c '(vmx|svm)' /proc/cpuinfo

#如果这个命令返回了 one or greater 的值，那么你的计算节点支持硬件加速且不需要额外的配置。

#如果这个命令返回了 zero 值，那么你的计算节点不支持硬件加速。你必须配置 libvirt 来使用 QEMU 去代替 KVM

#在 /etc/nova/nova.conf 文件的 [libvirt] 区域做出如下的编辑：

	#[libvirt]
	# ...
	#virt_type = qemu
```

### 启动 nova 计算服务并设置为开机启动
```bash
[ root@node2 ~]# systemctl enable libvirtd.service openstack-nova-compute.service

[ root@node2 ~]# systemctl start libvirtd.service openstack-nova-compute.service
```
## 在控制端验证
确认数据库中是否存在计算主机
```bash
openstack hypervisor list
+----+---------------------+-----------------+---------------+-------+
| ID | Hypervisor Hostname | Hypervisor Type | Host IP       | State |
+----+---------------------+-----------------+---------------+-------+
|  1 | node1               | QEMU            | 172.20.45.110 | up    |
|  2 | node2               | QEMU            | 172.20.45.111 | up    |
+----+---------------------+-----------------+---------------+-------+
```
### 添加计算节点到 cell 数据库
```bash
#控制节点主动发现计算节点
[ root@openstack1 ~]#  su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': 4dc76d88-522c-4540-95ce-51f046a90f2e
Found 2 computes in cell: 4dc76d88-522c-4540-95ce-51f046a90f2e
Checking host mapping for compute host 'node2': ccecabd0-5a44-404c-b19d-1f00cbf106c6
Creating host mapping for compute host 'node2': ccecabd0-5a44-404c-b19d-1f00cbf106c6
Checking host mapping for compute host 'node1': 253a8054-47e7-429e-9603-da35285a88f0
Creating host mapping for compute host 'node1': 253a8054-47e7-429e-9603-da35285a88f0

[ root@openstack1 ~]#  openstack hypervisor list
+----+---------------------+-----------------+---------------+-------+
| ID | Hypervisor Hostname | Hypervisor Type | Host IP       | State |
+----+---------------------+-----------------+---------------+-------+
|  1 | node2               | QEMU            | 192.168.3.101 | up    |
|  2 | node1               | QEMU            | 192.168.3.100 | up    |
+----+---------------------+-----------------+---------------+-------+
```

### 定期主动发现：慎用
```bash
vim /etc/nova/nova.conf
[scheduler]
discover_hosts_in_cells_interval=300 # 单位秒
```

### 验证计算节点
```bash
[ root@openstack1 ~]# nova host-list
+------------+-------------+----------+
| host_name  | service     | zone     |
+------------+-------------+----------+
| openstack1 | consoleauth | internal |
| openstack1 | conductor   | internal |
| openstack1 | scheduler   | internal |
| openstack2 | consoleauth | internal |
| openstack2 | conductor   | internal |
| openstack2 | scheduler   | internal |
| node2      | compute     | nova     |
| node1      | compute     | nova     |
+------------+-------------+----------+

#列出服务组件，以验证是否成功启动并注册了每个进程
[ root@openstack1 ~]# nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | openstack1 | internal | enabled | up    | 2019-06-21T07:31:18.000000 | -               |
| 2  | nova-conductor   | openstack1 | internal | enabled | up    | 2019-06-21T07:31:21.000000 | -               |
| 3  | nova-scheduler   | openstack1 | internal | enabled | up    | 2019-06-21T07:31:23.000000 | -               |
| 7  | nova-consoleauth | openstack2 | internal | enabled | up    | 2019-06-21T07:31:18.000000 | -               |
| 8  | nova-conductor   | openstack2 | internal | enabled | up    | 2019-06-21T07:31:24.000000 | -               |
| 9  | nova-scheduler   | openstack2 | internal | enabled | up    | 2019-06-21T07:31:17.000000 | -               |
| 10 | nova-compute     | node2      | nova     | enabled | up    | 2019-06-21T07:31:16.000000 | -               |
| 11 | nova-compute     | node1      | nova     | enabled | up    | 2019-06-21T07:31:20.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+

[ root@openstack1 ~]#  nova image-list
WARNING: Command image-list is deprecated and will be removed after Nova 15.0.0 is released. Use python-glanceclient or openstackclient instead
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 1e58f3fb-8cfb-4406-894c-3c83a111f0b2 | cirros-0.3.5 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+

[ root@openstack1 ~]#  nova-status upgrade check
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+

#列出服务组件是否成功注册，计算服务，包含控制端
[ root@openstack1 ~]# openstack compute service list

+----+------------------+------------+----------+---------+-------+----------------------+
| ID | Binary           | Host       | Zone     | Status  | State | Updated At           |
+----+------------------+------------+----------+---------+-------+----------------------+
|  1 | nova-consoleauth | openstack1 | internal | enabled | up    | 2019-06-21T07:33:48. |
|    |                  |            |          |         |       | 000000               |
|  2 | nova-conductor   | openstack1 | internal | enabled | up    | 2019-06-21T07:33:51. |
|    |                  |            |          |         |       | 000000               |
|  3 | nova-scheduler   | openstack1 | internal | enabled | up    | 2019-06-21T07:33:53. |
|    |                  |            |          |         |       | 000000               |
|  7 | nova-consoleauth | openstack2 | internal | enabled | up    | 2019-06-21T07:33:48. |
|    |                  |            |          |         |       | 000000               |
|  8 | nova-conductor   | openstack2 | internal | enabled | up    | 2019-06-21T07:33:54. |
|    |                  |            |          |         |       | 000000               |
|  9 | nova-scheduler   | openstack2 | internal | enabled | up    | 2019-06-21T07:33:47. |
|    |                  |            |          |         |       | 000000               |
| 10 | nova-compute     | node2      | nova     | enabled | up    | 2019-06-21T07:33:46. |
|    |                  |            |          |         |       | 000000               |
| 11 | nova-compute     | node1      | nova     | enabled | up    | 2019-06-21T07:33:50. |
|    |                  |            |          |         |       | 000000               |
+----+------------------+------------+----------+---------+-------+----------------------+
#每个控制端启动三个服务： nova-consoleauth：做认证的服务 nova-conductor：连接数据库  nova-scheduler：用来做调度的
#计算节点只启动一个服务：nova-compute ：启动虚拟机


#检查 cells 和 placement API 是否工作正常：
[ root@openstack1 ~]#  nova-status upgrade check
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+

#列出 keystone 服务中的端点，以验证 keystone 的连通性。
[ root@openstack1 ~]# openstack catalog list
+-----------+-----------+--------------------------------------------+
| Name      | Type      | Endpoints                                  |
+-----------+-----------+--------------------------------------------+
| glance    | image     | RegionOne                                  |
|           |           |   internal: http://192.168.7.248:9292      |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.7.248:9292        |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.7.248:9292         |
|           |           |                                            |
| placement | placement | RegionOne                                  |
|           |           |   admin: http://192.168.7.248:8778         |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.7.248:8778        |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.7.248:8778      |
|           |           |                                            |
| keystone  | identity  | RegionOne                                  |
|           |           |   admin: http://192.168.7.248:35357/v3     |
|           |           | RegionOne                                  |
|           |           |   public: http://192.168.7.248:5000/v3     |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.7.248:5000/v3   |
|           |           |                                            |
| nova      | compute   | RegionOne                                  |
|           |           |   public: http://192.168.7.248:8774/v2.1   |
|           |           | RegionOne                                  |
|           |           |   internal: http://192.168.7.248:8774/v2.1 |
|           |           | RegionOne                                  |
|           |           |   admin: http://192.168.7.248:8774/v2.1    |
|           |           |                                            |
+-----------+-----------+--------------------------------------------+
```

## 请继续查看下一篇博客：[网络服务neutron的部署](http://aishad.top/wordpress/?p=377 "网络服务neutron的部署")
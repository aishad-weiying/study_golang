# Nova
nova 是 openstack 最早的组件之一，nova 分为控制节点和计算节点，计算节点通过 nova computer 进行虚拟机创建，通过 libvirt 调用 kvm 创建虚拟机，nova 之间通信通过 rabbitMQ队列进行通信，其组件和功能如下：
API：负责接收和响应外部请求。
Scheduler：负责调度虚拟机所在的物理机。拿到一个来自队列请求虚拟机实例，然后决定那台计算服务器主机来运行它
Conductor：计算节点访问数据库的中间件。调解nova-compute服务和数据库之间的交互。它消除了对nova-compute服务所做的云数据库的直接访问
Consoleauth：用于控制台的授权认证。
Novncproxy：VNC 代理，用于显示虚拟机操作终端。

- Nova-API 的功能
Nova-api 组件实现了 restful API 的功能，接收和响应来自最终用户的计算 API 请求，接收外部的请求并通过 message queue 将请求发动给其他服务组件，同时也兼容 EC2 API，所以也可以使用 EC2 的管理工具对 nova 进行日常管理。

- nova scheduler
nova scheduler 模块在 openstack 中的作用是决策虚拟机创建在哪个主机（计算节点）上。决策一个虚拟机应该调度到某物理节点，需要分为两个步骤：

1. 过滤（filter），过滤出可以创建虚拟机的主机
[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/nova1.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/nova1.png)

2. 计算权值（weight），根据权重大进行分配，默认根据资源可用空间进行权重排序
[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/nova2.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/nova2.png)

# 安装并配置控制节点
控制节点部署在openstack的管理端

安装nova控制端
```bash
 [ root@openstack1 ~]# yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api -y
```

## 数据库服务器创建并授权
```bash
CREATE DATABASE nova_api;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'admin123';

CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'admin123';

CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'admin123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'admin123';
```

## 创建 nova 用户供nova服务使用：
创建 nova 用户并设置密码为 nova
```bash
[ root@openstack1 ~]# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | 6ef143bb68b243d9bd18b181873253d4 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 对nova用户授权
将 nova 用户授权为 service 项目的 admin 权限
```bash
[ root@openstack1 ~]# openstack role add --project service --user nova admin
```

## 创建 nova 服务实体
```bash
[ root@openstack1 ~]# openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | d80fdd0fb74445bca79c26c53bcd3f0c |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```
- 创建nova服务的 API 端点
```bash
[ root@openstack1 ~]# openstack endpoint create --region RegionOne compute public http://192.168.7.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 36b0454631f74edabe63568ad6db2f75 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d80fdd0fb74445bca79c26c53bcd3f0c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.7.248:8774/v2.1   |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne compute internal http://192.168.7.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b84b11134e3a43b19a6ab59b383aaaab |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d80fdd0fb74445bca79c26c53bcd3f0c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.7.248:8774/v2.1   |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne compute admin http://192.168.7.248:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e3c4751bc071470999c4418bc1d2117a |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | d80fdd0fb74445bca79c26c53bcd3f0c |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://192.168.7.248:8774/v2.1   |
+--------------+----------------------------------+
```

### 创建 placement 用户并授权
Placement 用户密码设置为 placement
```bash
[ root@openstack1 ~]# openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | c4ad157170bd428da5c8990008d75682 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 对 placement 用户授权
将 placement 用户授权为 service 项目的 admin 权限
```bash
[ root@openstack1 ~]#  openstack role add --project service --user placement admin
```

#### 创建 placement 服务实体
```bash
[ root@openstack1 ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 99068eba367043dc8f63bf4c41a16cf1 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

- 创建placement服务的 API 端点
```bash
[ root@openstack1 ~]# openstack endpoint create --region RegionOne placement public http://192.168.7.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7a74dab5212a46ac9bf831b10d871a28 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 99068eba367043dc8f63bf4c41a16cf1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.7.248:8778        |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne placement internal http://192.168.7.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e0e79641ca0146b88985d998329ad017 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 99068eba367043dc8f63bf4c41a16cf1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.7.248:8778        |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne placement admin http://192.168.7.248:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0ee184b69baa4d169b5623e9d3d1b99d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 99068eba367043dc8f63bf4c41a16cf1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://192.168.7.248:8778        |
+--------------+----------------------------------+
```
## 配置HAproxy代理8874和8878
```bash
listen  openstack_nova_port_8774
 bind 192.168.7.248:8774
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:8774  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:8774  check inter 3000 fall 2 rise 5 backup

listen  openstack_nova_port_8778
 bind 192.168.7.248:8778
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:8778  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:8778  check inter 3000 fall 2 rise 5 backup
```
> 另一个要配置为backup，防止两个rabbitMQ做同步的时候，因为网络问题造成延迟，导致另一台读不到数据


## 编辑/etc/nova/nova.conf配置文件
```bash
[ root@openstack1 ~]# grep -n "^[a-Z\[]" /etc/nova/nova.conf
1:[DEFAULT]
2:enabled_apis = osapi_compute,metadata  #只启用计算和元数据API
#配置RabbitMQ消息队列访问权限,不支持使用负载均衡做代理
3:transport_url = rabbit://openstack:admin123@192.168.2.101
4:rpc_backend=rabbit
#启用网络服务支持
5:use_neutron = True
6:firewall_driver = nova.virt.firewall.NoopFirewallDriver


#使用keystone认证
3073:[api]
3074:auth_strategy = keystone

#配置api数据库的连接
3372:[api_database]
3386:connection = mysql+pymysql://nova:admin123@192.168.7.248/nova_api

#配置nova数据库的连接
4375:[database]
4377:connection = mysql+pymysql://nova:admin123@192.168.7.248/nova

#配置镜像服务 API 的位置
4944:[glance]
4945:api_servers = http://192.168.7.248:9292

#使用keystone认证时的选项
5604:[keystone_authtoken]
5605:auth_uri = http://192.168.7.248:5000
5606:auth_url = http://192.168.7.248:35357
5607:memcached_servers = 192.168.7.248:11211
5608:auth_type = password
5609:project_domain_name = default
5610:user_domain_name = default
5611:project_name = service
5612:username = nova
5613:password = nova

#配置锁路径
7305:[oslo_concurrency]
7320:lock_path=/var/lib/nova/tmp
#配置锁路径，在同一个任务中把一个操作以文件的方式放到指定路径，在操作没处理完之前加锁

#
8151:[placement]
8152:os_region_name = RegionOne
8153:project_domain_name = Default
8154:project_name = service
8155:auth_type = password
8156:user_domain_name = Default
8157:auth_url = http://192.168.7.248:35357/v3
8158:username = placement
8159:password = placement

#配置VNC代理使用控制节点的管理接口IP地址，监听本机的ip地址,或者0.0.0.0
9739:[vnc]
9740:enabled = true
9741:vncserver_listen = 192.168.1.100
9742:vncserver_proxyclient_address = 192.168.1.100

```

### 配置 apache 允许访问 placement API
```bash
[ root@openstack1 ~]# vim /etc/httpd/conf.d/00-nova-placement-api.conf
# 添加在最下方
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
[ root@openstack1 ~]# systemctl restart httpd
```

### 数据库初始化
```bash
# 初始化nova_api数据库
[ root@openstack1 ~]# su -s /bin/sh -c "nova-manage api_db sync" nova

#初始化nova cell0 数据库
[ root@openstack1 ~]# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

# 初始化nova cell1 数据库
[ root@openstack1 ~]# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

#初始化nova数据库
[ root@openstack1 ~]# su -s /bin/sh -c "nova-manage db sync" nova
```

### 验证 nova cell0 和 nova cell1 是否正常注册
```bash
[ root@openstack1 ~]#  nova-manage cell_v2 list_cells
+-------+--------------------------------------+
|  Name |                 UUID                 |
+-------+--------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |
| cell1 | 4dc76d88-522c-4540-95ce-51f046a90f2e |
+-------+--------------------------------------+
```

### 启动并将 nova 服务设置为开机启动
```bash
[ root@openstack1 ~]# systemctl enable openstack-nova-api.service   openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service

[ root@openstack1 ~]# systemctl start openstack-nova-api.service   openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## 验证nova端
```bash
[ root@openstack1 ~]# nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | openstack1 | internal | enabled | up    | 2019-06-21T06:13:45.000000 | -               |
| 2  | nova-conductor   | openstack1 | internal | enabled | up    | 2019-06-21T06:13:45.000000 | -               |
| 3  | nova-scheduler   | openstack1 | internal | enabled | up    | 2019-06-21T06:13:45.000000 | -               |
| 7  | nova-consoleauth | openstack2 | internal | enabled | up    | 2019-06-21T06:13:48.000000 | -               |
| 8  | nova-conductor   | openstack2 | internal | enabled | up    | 2019-06-21T06:13:48.000000 | -               |
| 9  | nova-scheduler   | openstack2 | internal | enabled | up    | 2019-06-21T06:13:48.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
```

## 查看rabbitMQ连接情况
[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/rabbit_nova.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/rabbit_nova.png)

## 请继续查看下一篇博客：[openstack nova计算节点的配置](http://aishad.top/wordpress/?p=376 "openstack nova计算节点的配置")
# 镜像服务glance
Glance 是 OpenStack 镜像服务组件，glance 服务默认监听在 9292 端口，其接收 REST API 请求，然后通过其他模块（glance-registry 及 image store）来完成诸如镜像的获取、上传、删除等操作，Glance 提供 restful API 可以查询虚拟机镜像的 metadata，并且可以获得镜像，通过Glance，虚拟机镜像可以被存储到多种存储上，比如简单的文件存储或者对象存储（比如OpenStack 中 swift 项目）是在创建虚拟机的时候，需要先把镜像上传到 glance，对镜像的列出镜像、删除镜像和上传镜像都是通过 glance 进行理，glance 有两个主要的服务，一个是glace-api 接收镜像的删除上传和读取，一个是 glance-Registry。

- glance-Registry
负责与 mysql 数据交互，用于存储或获取镜像的元数据（metadata），提供镜像元数据相关的 REST 接口，通过 glance-registry 可以向数据库中写入或获取镜像的各种数据，glance-registyr 监听的端口是 9191，glance 数据库中有两张表，一张是 glance 表，一张是 imane property 表，image 表保存了镜像格式、大小等信息，image property 表保存了镜像的定制化信息。

- image store
是一个存储的接口层，通过这个接口 glance 可以获取镜像，image store 支持的存储有 Amazon 的 S3、openstack 本身的 swift、还有 ceph、glusterFS、sheepdog 等分布式存储，image store 是镜像保存与读取的接口，但是它只是一个接口，具体的实现需要外部的支持，glance 不需要配置消息队列，但是需要配置数据库和 keystone。

> 注：要保证镜像服务上传的镜像的高可用，可以实现将这些镜像保存到后端共享的存储中，然后对后端的存储做高可用

# 创建glance服务

- 管理端安装glance
```bash
[ root@openstack1 ~]# yum install -y openstack-glance
```

## 在数据库服务器创建glance数据库并授权
```bash
MariaDB [(none)]>  create database glance;

MariaDB [(none)]> grant all on glance.* to 'glance'@'%' identified by 'admin123';
```
### 创建 glance 用户：如果已经创建跳过此步骤
创建 glance 密码用户并设置密码为 glance
```bash
[ root@openstack1 ~]# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | c33540c93bd74143a354eab7d23dab44 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 对 glance 用户授权
把 glance 和 neutron 用户添加到 service 项目并授予 admin 角色
```bash
[ root@openstack1 ~]# openstack role add --project service --user glance admin
```
### 创建glance服务实体
```bash
[ root@openstack1 ~]# openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 7a2f49e374ac4a6ea9105299a8fade68 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

- 创建镜像服务的 API 端点:9292端口是用来和用户操作做对接的
```bash
[ root@openstack1 ~]# openstack endpoint create --region RegionOne image public http://192.168.7.248:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ac5f48153da04691808b0fb44f80f212 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7a2f49e374ac4a6ea9105299a8fade68 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.7.248:9292        |
+--------------+----------------------------------+

[ root@openstack1 ~]# openstack endpoint create --region RegionOne image internal http://192.168.7.248:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8fa524f85d504250b72e464a2f3c37cc |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7a2f49e374ac4a6ea9105299a8fade68 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.7.248:9292        |
+--------------+----------------------------------+

[ root@openstack1 ~]#  openstack endpoint create --region RegionOne image admin http://192.168.7.248:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e6ac0200deab45fda182236392652333 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7a2f49e374ac4a6ea9105299a8fade68 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://192.168.7.248:9292        |
+--------------+----------------------------------+
```

## 在管理端配置glance

- 编辑/etc/glance/glance-api.conf配置文件
```bash
[ root@openstack1 ~]#  grep -n "^[a-Z\[]" /etc/glance/glance-api.conf
# 配置数据库访问
1:[DEFAULT]
1798:[database]
1800:connection = mysql+pymysql://glance:admin123@192.168.7.248/glance

#配置本地文件系统存储和镜像文件位置
1915:[glance_store]
1917:stores = file,http
1918:default_store = file #镜像以文件的方式保存
1919:filesystem_store_datadir = /var/lib/glance/images/
#镜像的存放位置，该目录会自动创建，可以将后端的存储挂载到这个目录

#配置认证服务访问
3285:[keystone_authtoken]
3287:auth_uri = http://192.168.7.248:5000
3288:auth_url = http://192.168.7.248:35357
3289:memcached_servers = 192.168.7.248:11211
3290:auth_type = password
3291:project_domain_name = default
3292:user_domain_name = default
3293:project_name = service
3294:username = glance  #glance认证的账号，该账号用来上传镜像
3295:password = glance  #glance认证时密码

4246:[paste_deploy]
4271:flavor = keystone
```

- 编辑配置文件 glance-registry.conf
```bash
[ root@openstack1 ~]#  grep -n "^[a-Z\[]" /etc/glance/glance-registry.conf

1088:[database]
1089:connection = mysql+pymysql://glance:admin123@192.168.7.248/glance

1204:[keystone_authtoken]
1205:auth_uri = http://192.168.7.248:5000
1206:auth_url = http://192.168.7.248:35357
1207:memcached_servers = 192.168.7.248:11211
1208:auth_type = password
1209:project_domain_name = default
1210:user_domain_name = default
1211:project_name = service
1212:username = glance
1213:password = glance

2135:[paste_deploy]
2160:flavor = keystone

```

### 配置 haproxy 代理 glance
```bash
listen  openstack_glabce_api
 bind 192.168.7.248:9292
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:9292  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:9292  check inter 3000 fall 2 rise 5 backup

listen  openstack_glabce
 bind 192.168.7.248:9191
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:9191  check inter 3000 fall 2 rise 5
 server 192.168.1.101  192.168.1.101:9191  check inter 3000 fall 2 rise 5 backup
```
### 初始化数据库
```bash
[ root@openstack1 ~]# su -s /bin/sh -c "glance-manage db_sync" glance
```

### 启动 glance 并设置为开机启动
```bash
[ root@openstack1 ~]# systemctl enable openstack-glance-api.service openstack-glance-registry.service
[ root@openstack1 ~]# systemctl start openstack-glance-api.service openstack-glance-registry.service
# 验证9191和9292端口打开
```
### 配置后端存储共享目录到镜像目录
因为内存不足，所以就在mysql服务器开启nfs共享
```bahs
# 配置共享目录
[ root@mysql ~]# cat /etc/exports
/data/iso *(rw,async)
# 给恭喜目前权限，因为上传目录的时候，使用glance用户上传，将目录的属主和属组改为glance
[ root@mysql ~]# chown 161.161 /data/iso -R
[ root@mysql ~]# ll /data/
total 0
drwxr-xr-x 2 161 161 6 Jun 21 20:28 iso
[ root@mysql ~]# ll /data/ -d
drwxr-xr-x 3 root root 16 Jun 21 20:28 /data/
```
- openstack管理端设置挂载
```bahs
[ root@openstack1 ~]# cat /etc/fstab 
192.168.2.100:/data/iso /var/lib/glance/images nfs defaults,_netdev 0 0
[ root@openstack1 ~]# mount -a
```


### 测试 glance 上传镜像
```bash
#在 glance 下载一个 0.3.4 版本的测试镜像
[ root@openstack1 ~]# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

[ root@openstack1 ~]# openstack image create "cirros-0.3.5" --file /root/cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2019-06-21T03:12:51Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/514f3560-2dd4-473e-b52a-cc24f0d23e73/file |
| id               | 514f3560-2dd4-473e-b52a-cc24f0d23e73                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros-0.3.5                                         |
| owner            | 0476bec7404e498185d6d9886f719b2b                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2019-06-21T03:12:58Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```
> 会在指定的镜像目录生成以镜像id命名的文件
```bash
[ root@openstack1 ~]# ll /var/lib/glance/images/
total 12980
-rw-r----- 1 glance glance 13287936 Jun 21 11:12 514f3560-2dd4-473e-b52a-cc24f0d23e73
```
### 验证镜像
```bahs
[ root@openstack1 ~]# glance image-list
+--------------------------------------+--------------+
| ID                                   | Name         |
+--------------------------------------+--------------+
| 514f3560-2dd4-473e-b52a-cc24f0d23e73 | cirros-0.3.5 |
+--------------------------------------+--------------+
[ root@openstack1 ~]# openstack image list
+--------------------------------------+--------------+--------+
| ID                                   | Name         | Status |
+--------------------------------------+--------------+--------+
| 514f3560-2dd4-473e-b52a-cc24f0d23e73 | cirros-0.3.5 | active |
+--------------------------------------+--------------+--------+
#查看指定镜像的详细信息
[ root@openstack1 ~]# openstack image show cirros-0.3.5
```

## 继续查看下一篇博客：[openstack noav控制端部署](http://aishad.top/wordpress/?p=371 "openstack noav控制端部署")
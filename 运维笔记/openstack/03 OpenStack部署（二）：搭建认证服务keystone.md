# Keystone简介

Keystone 中主要涉及到如下几个概念：User、Tenant、Role、Token:
	User：使用 openstack 的用户。
	Tenant：租户,可以理解为一个人、项目或者组织拥有的资源的合集。在一个租户中可以拥有很多个用户，这些用户可以根据权限的划分使用租户中的资源。
	Role：角色，用于分配操作的权限。角色可以和用户做绑定，使得该用户获得角色对应的操作权限。
	Token：指的是一串比特值或者字符串，用来作为访问资源的记号。Token 中含有可访问资源的范围和有效时间。
- 虚拟机创建的过程
[![](images/ks.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/ks.png)

keystone默认监听的端口是5000和35357

## keystone数据库配置

### 为keystonr创建数据库
```bash
MariaDB [(none)]> create database keystone;
MariaDB [(none)]> grant all on keystone.* to 'keystone'@'%' identified by 'admin123';
```

- 验证数据库：
	验证可以从 openstack 控制端使用 keystone 访问数据库
```bash
# 通过负载均衡使用keystone连接数据库
[ root@openstack1 ~]# mysql -ukeystone -padmin123 -h192.168.7.248
```

## 安装keystone
```bash
#openstack-keystone 是 keystone 服务，http 是 web 服务，mod_wsgi 是 python 的通用网关
[ root@openstack1 ~]#  yum install -y openstack-keystone httpd mod_wsgi python-memcached
```

- 配置keyston的配置文件
```bash
#生成临时的token，可以绕过keystone，直接以管理员的token身份登录到OpenStack
[ root@openstack1 ~]# openssl rand -hex 10
a6998cd3e82cc4646388

[ root@openstack1 ~]# vim /etc/keystone/keystone.conf
18 admin_token = a6998cd3e82cc4646388
714 connection = mysql+pymysql://keystone:admin123@192.168.7.248/keystone
#此处的ip地址为vip的地址，也可以使用域名，只要本地能解析即可
2832 provider = fernet #Fernet UUID令牌的提供者
```

- 初始化数据库
```bash
#会在keystone数据库创建默认表等操作
[ root@openstack1 ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

- 初始化证书并验证
```bash
[ root@openstack1 ~]# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
[ root@openstack1 ~]#  keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
[ root@openstack1 ~]#  ll /etc/keystone/fernet-keys/
total 8
-rw-------. 1 keystone keystone 44 Jun 19 22:16 0
-rw-------. 1 keystone keystone 44 Jun 19 22:16 1
```

- 配置appache代理pyhton
附带wsgi文件的下载地址：[下载wsgi文件](http://upload.aishad.top/wsgi-keystone.conf "下载wsgi文件")
```bash
[ root@openstack1 ~]# ln -sv /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/wsgi-keystone.conf

[ root@openstack1 ~]# systemctl start httpd
[ root@openstack1 ~]# systemctl enable httpd
#监听了35357和5000端口
```

- 创建域、用户、项目和角色
```bash
#通过 admin 的 token 设置环境变量进行操作,跳过keystone的认证，获取admin的权限
[ root@openstack1 ~]# export OS_TOKEN=a6998cd3e82cc4646388
[ root@openstack1 ~]# export OS_URL=http://192.168.1.100:35357/v3
[ root@openstack1 ~]# export OS_IDENTITY_API_VERSION=3
```

- 创建默认域

> 一定要在上一步设置完成环境变量的前提下方可操作成功，否则会提示未认证。
>  命令格式为：openstack domain create --description "描述信息" 域名

```bash
[ root@openstack1 ~]# openstack domain create --description "Default Domain" default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Default Domain                   |
| enabled     | True                             |
| id          | f80d8184b3464c70aa08273a7ee0e5fc |
| name        | default                          |
+-------------+----------------------------------+
#删除默认域
#openstack domain delete f80d8184b3464c70aa08273a7ee0e5fc
```

- 创建一个admin项目

>命令格式为 openstack project --domain 域 --description "描述" 项目名
```bash
[ root@openstack1 ~]# openstack project create --domain default --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled     | True                             |
| id          | 0476bec7404e498185d6d9886f719b2b |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | f80d8184b3464c70aa08273a7ee0e5fc |
+-------------+----------------------------------+
```

- 创建 admin 用户并设置密码为 admin
```bash
[ root@openstack1 ~]# openstack user create --domain default --password-prompt admin
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | e86ce7b180e24ca883d17236347c35bf |
| name                | admin                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 创建 admin 角色
一个项目里面可以有多个角色，目前角色只能创建在/etc/keystone/policy.json 文件中定义好的角色
```bash
[ root@openstack1 ~]# openstack role create admin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | f2fcfd388b224141a29db83ac62315a8 |
| name      | admin                            |
+-----------+----------------------------------+
```

- 给 admin 用户授权
将 admin 用户授予 admin 项目的 admin 角色，即给 admin 项目添加一个用户叫 admin，并将其添加至 admin 角色，角色是权限的一种集合
```bash
[ root@openstack1 ~]# openstack role add --project admin --user admin admin
```

- 创建 demo 项目
```bash
[ root@openstack1 ~]# openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled     | True                             |
| id          | 6c410cb5b00849f8815177b73ae94876 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | f80d8184b3464c70aa08273a7ee0e5fc |
+-------------+----------------------------------+
```

- 创建 demo 用户并设置密码为 demo
```bash
[ root@openstack1 ~]# openstack user create --domain default --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled             | True                             |
| id                  | fca496e90289494f8bbcaebed93b5ceb |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 创建一个 user 角色
```bash
[ root@openstack1 ~]# openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 184859aec29a4b81b444dac30c93f1f9 |
| name      | user                             |
+-----------+----------------------------------+
```

- 把 demo 用户添加到 demo 项目
然后赋予 user 权限
```bash
[ root@openstack1 ~]# openstack role add --project demo --user demo user
```

- 创建一个 service 项目
各服务之间与 keystone 进行访问和认证，service 用于给服务创建用户,service项目会记录openstack各个服务之间的访问方式
```bash
[ root@openstack1 ~]# openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | f80d8184b3464c70aa08273a7ee0e5fc |
| enabled     | True                             |
| id          | 08a1746b80fd4a4b92de16f0365c244f |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | f80d8184b3464c70aa08273a7ee0e5fc |
+-------------+----------------------------------+
```
## 服务注册
将 keystone 服务地址注册到 openstack

- 创建一个 keystone 认证服务：
```bash
[ root@openstack1 ~]# openstack service list #查看当前的服务
# 将 keystone 服务地址注册到 openstack
[ root@openstack1 ~]# openstack service create --name keystone --description "OpenStack Identity" identity #identity认证类型
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | cb3388f4727f49c49c4c3eb1208e5f60 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
[ root@openstack1 ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| cb3388f4727f49c49c4c3eb1208e5f60 | keystone | identity |
+----------------------------------+----------+----------+
```

- 创建 endpoint
如果创建错误或多创建了，就要全部删除再重新注册，因为你不知道哪一个是对的哪一个是错的，所以只能全部删除然后重新注册，注册的IP地址写keepalived的VIP，稍后配置haproxy：
实体一共有三种：public、admin和internal
```bash
# 公共端点
[ root@openstack1 ~]# openstack endpoint create --region RegionOne identity public http://192.168.7.248:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 40056ad3e96f4ea5b3b6987271b08e8b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cb3388f4727f49c49c4c3eb1208e5f60 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.7.248:5000/v3     |
+--------------+----------------------------------+
#私有端点
[ root@openstack1 ~]# openstack endpoint create --region RegionOne identity internal http://192.168.7.248:5000/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6382d3e12e8c4b8d886756516fdec899 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cb3388f4727f49c49c4c3eb1208e5f60 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.7.248:5000/v3     |
+--------------+----------------------------------+
#管理端点
[ root@openstack1 ~]# openstack endpoint create --region RegionOne identity admin http://192.168.7.248:35357/v3
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1e7c504f141148b9a809fbc0c48578c1 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | cb3388f4727f49c49c4c3eb1208e5f60 |
| service_name | keystone                         |
| service_type | identity                         |
| url          | http://192.168.7.248:35357/v3    |
+--------------+----------------------------------+
#验证三个端点
[ root@openstack1 ~]# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                           |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------+
| 1e7c504f141148b9a809fbc0c48578c1 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.7.248:35357/v3 |
| 40056ad3e96f4ea5b3b6987271b08e8b | RegionOne | keystone     | identity     | True    | public    | http://192.168.7.248:5000/v3  |
| 6382d3e12e8c4b8d886756516fdec899 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.7.248:5000/v3  |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------+
```

### 配置HAProxy的转发：
```bash
root@lb1:~# vim /etc/haproxy/haproxy.cfg 
listen  openstack_keystone_public_url
 bind 192.168.7.248:5000
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:5000  check inter 3000 fall 2 rise 5

listen  openstack_keystone_admin_url
 bind 192.168.7.248:35357
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:35357  check inter 3000 fall 2 rise 5
# 重启服务
root@lb1:~# systemctl restart haproxy.service
#测试
[ root@mysql ~]$ telnet 192.168.7.248 35357
Trying 192.168.7.248...
Connected to 192.168.7.248.
Escape character is '^]'.
^C
Connection closed by foreign host.
[ root@mysql ~]$ 
[ root@mysql ~]$ telnet 192.168.7.248 5000
Trying 192.168.7.248...
Connected to 192.168.7.248.
Escape character is '^]'.
^C
Connection closed by foreign host.
```

## 测试 keystone 是否可以做用户验证
验证 admin 用户，密码 admin123，新打开一个窗口并进行以下操作
```bash
[ root@openstack1 ~]# export OS_IDENTITY_API_VERSION=3
[ root@openstack1 ~]# openstack --os-auth-url http://192.168.7.248:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: 
+------------+-----------------------------------------------------------------------------+
| Field      | Value                                                                       |
+------------+-----------------------------------------------------------------------------+
| expires    | 2019-06-19T16:20:32+0000                                                    |
| id         | gAAAAABdClLCCpuF6x0VjQ2UVipmRB-1m3PIGXdZ006PlxWa-OX_K7lAixpCptpTP9Ypb4AUFMh |
|            | QkTibu6rLoacbhb_mW27b4ZJBy2pVrQpLPTiWQpbCB3GAMYunrnO61TEN1_MeLV0Ozs5jBGdMKg |
|            | xf4oLL9pz0f71humdDAAZ-YLoAr9Yc7nU                                           |
| project_id | 0476bec7404e498185d6d9886f719b2b                                            |
| user_id    | e86ce7b180e24ca883d17236347c35bf                                            |
+------------+-----------------------------------------------------------------------------+
```
验证 demo 用户，密码为 demo：
```bash
[ root@openstack1 ~]# export OS_IDENTITY_API_VERSION=3
[ root@openstack1 ~]# openstack --os-auth-url http://192.168.7.248:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
+------------+-----------------------------------------------------------------------------+
| Field      | Value                                                                       |
+------------+-----------------------------------------------------------------------------+
| expires    | 2019-06-19T16:22:48+0000                                                    |
| id         | gAAAAABdClNJBqQ7eQF5S6oEHKtS5_t0ahskONCLqgHdSR3CzTtO0ean_qn2cxonmXg05ctrs8O |
|            | TIjTKs6Q6Ka8C31OolSnfZsHgMUG7_0gzeqB8Zx6gDri-                               |
|            | i4xjbl_aIjmDBMXhaLYKwismLFEjf4-hRya593vZGhbwB7HzkgDeFh1dzRP5xaY             |
| project_id | 6c410cb5b00849f8815177b73ae94876                                            |
| user_id    | fca496e90289494f8bbcaebed93b5ceb                                            |
+------------+-----------------------------------------------------------------------------+
```

## 使用脚本设置环境变量
admin用户的脚本
```bash
[ root@openstack1 ~]# cat admin-ocata.sh 
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin123
export OS_AUTH_URL=http://192.168.7.248:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
# 生成环境变量
[ root@openstack1 ~]# source admin-ocata.sh
# 重新用admin获取，不再需要输入密码
[ root@openstack1 ~]# openstack --os-auth-url http://192.168.7.248:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
```
demo用户的脚本
```bash
[ root@openstack1 ~]# cat demo-ocata.sh 
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.7.248:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```


### 创建 glance 用户：后面创建的glance服务会用到
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

### 创建 nova 用户：后面创建的nove服务会用到
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

### 创建 neutron 用户并设置密码为 neutron 后面创建的neutron服务会用到
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
## 部署openstack 管理端2：实现管理端的高可用

- 在openstack2安装openstack服务和keystone服务
```bash
[ root@openstack2 ~]# yum install -y python-openstackclient
[ root@openstack2 ~]# yum install -y openstack-keystone httpd mod_wsgi python-memcached
[ root@openstack2 ~]# yum install -y mariadb python2-PyMySQL
```

- 将openstack1的keystone服务的配置文件打包发送给openstack2
```bash
[ root@openstack1 keystone]# tar zcvf keystone.tar.gz ./*
[ root@openstack1 keystone]# scp keystone.tar.gz 192.168.1.101:/etc/keystone/
# 拷贝wsgi的配置文件
[ root@openstack1 keystone]# scp /usr/share/keystone/wsgi-keystone.conf 192.168.1.101:/etc/httpd/conf.d/
```

- 启动服务
```bash
#启动httpd服务，监听5000和35357端口
[ root@openstack2 keystone]# systemctl start httpd
```

- 配置HAProxy
```bash
listen  openstack_keystone_public_url
 bind 192.168.7.248:5000
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:5000  check inter 3000 fall 2 rise 5
 server 192.168.1.100  192.168.1.101:5000  check inter 3000 fall 2 rise 5 backup

listen  openstack_keystone_admin_url
 bind 192.168.7.248:35357
 mode tcp
 log global
 balance source
 server 192.168.1.100  192.168.1.100:35357  check inter 3000 fall 2 rise 5
 server 192.168.1.100  192.168.1.101:35357  check inter 3000 fall 2 rise 5 backup
```

## 请继续查看下一篇博客：[openstack glance服务的安装](http://aishad.top/wordpress/?p=370 "openstack glance服务的安装")
# 企业级自动化运维工具Ansible 特性
模块化：调用特定的模块，完成特定任务
有Paramiko、PyYAML、Jinja2(模板语言)三个关键模块
支持自定义模块
基于python语言实现
部署简单，基于python和ssh，agentless
安全，基于OpenSSL
支持playbook编排任务
幂等性，一个任务执行1遍和执行n遍效果一样，不因重复执行带来意外情况
无需代理，不依赖PKI（无需ssl）
可使用任何编程语言写模块
YAML格式，编排任务，支持丰富的数据结构
交强大的多层解决方案

## ansible主要组成部分
Ansible PLAYBOOKS：任务剧本（任务集），编排定义ansible任务集的配置文件，有ansible顺序依次执行，通常是json格式的YML文件

INVENTORY：ansible管理主机的清单/etc/ansible/hosts

MODULES：ansible执行命令的功能模块，多数为内置的核心模块，也可以自定义

PLUGINS：模块功能的补充，如连接类型插件、循环插件、变量插件、过滤插件等，该功能不常用

API：供第三方程序调用的应用程序编程接口

Ansible：组合INVENTORY、API、MODEULES、PLUGINS的绿框，可以理解为是ansible命令工具，其为核心执行工具

1. ansible命令的执行来源
user：普通用户，即system administrator
cmdb（配置管理数据库）API调用
PUBLIC/PRIVATE CLOUD API调用
USER-> Ansible Playbook -> Ansibile

2. 利用ansible实现管理方式
ad-hoc：即ansible命令，主要用具临时命令使用场景
ansible-playboos：主要用于长期规划好的，大型项目的场景，需要由前期的规划过程

3. ansible-playbook（剧本）执行过程
将已有编排好的任务集写入ansible-playbook
通过ansible-playbook命令分析任务集至逐条ansible命令，按预定规则逐条执行

4. ansible主要操作对象
hosts：主机
networking：网络设备

5. 注意事项
	执行ansible的主机一般称为主控端，中控，master或堡垒机
	主控端Python版本需要2.6或以上
	被控端Python版本小于2.4需要安装python-simplejson
	被控端如开启SELinux需要安装libselinux-python
	windows不能做为主控端

## 安装ansible
1. rpm包安装：EPEL源
```bash
yum install ansibel
```

2. 编译安装
```bash
yum -y install python-jinja2 PyYAML python-paramiko python-babel python-crypto
tar xf ansible-1.5.4.tar.gz
cd ansible-1.5.4
python setup.py build
python setup.py install
mkdir /etc/ansible
cp -r examples/* /etc/ansible
```

3. git方式
```bash
git clone git://github.com/ansible/ansible.git --recursive
cd ./ansible
source ./hacking/env-setup
```

4. pip安装：pip是安装Python包的管理器，类似yum
```bash
yum install python-pip python-devel
yum install gcc glibc-devel zibl-devel rpm-bulid openssl-devel
pip install --upgrade pip
pip install ansible --upgrade

```

### ansible 的相关配置文件
1. 配置文件

/etc/ansible/ansible.cfg主配置文件，配置ansible工作特性
/etc/ansible/hosts:主机清单，定义管理的主机列表
/etc/ansible/roles：存放角色的目录

2. 程序
/usr/bin/ansible 主程序，临时命令执行工具
/usr/bin/ansible-doc 查看配置文档，模块功能查看工具
/usr/bin/ansible-galaxy 下载/上传优秀代码或Roles模块的官网平台
/usr/bin/ansible-playbook 定制自动化任务，编排剧本工具
/usr/bin/ansible-pull 远程执行命令的工具
/usr/bin/ansible-vault 文件加密工具
/usr/bin/ansible-console 基于Console界面与用户交互的执行工具

### ansible的主机清单inventory
ansible的主要功用在于批量主机操作，为了便捷的使用其中的部分主机，可以再inventory file中将其分组命名

默认的invebtory file为/etc/ansible/hosts

inventory file可以有多个，且也可以通过Dynamic Inventory来动态生成

1. /etc/ansible/hosts文件格式
inventory文件遵循INI文件风格，中括号中的字符为组名，可以将同一个主机同时归并到多个不同的组中，此外，当如若目标主机使用了非默认的ssh端口，还可以在主机名称之后使用冒号加端口号来标明
```bash
ntp.magedu.com
[webservers]
www1.magedu.com:2222
www2.magedu.com
[dbservers]
db1.magedu.com
db2.magedu.com
db3.magedu.com
```
如果主机名称遵循相似的命令模式，还可以使用列表的方式表示各个主机
```bash
[websrvs]
www[01:100].example.com
[dbsrvs]
db-[a:f].example.com
```

## ansible 配置文件
Ansible 配置文件/etc/ansible/ansible.cfg （一般保持默认）

1. defaults字段
invebtory = /etc/ansible/hosts : 主机列表配置文件
library = /usr/share/my_modules/ ： 库文件存放目录
remote_tmp = $HOME/.ansible/tmp ： 临时py命令文件存放在远程主机目录
local_tmp = $HOME/.ansible/tmp : 本机的临时命令执行目录
forks = 5 ： 默认并发数
sudo_user =root ：默认sudo用户
ask_sudo_pass =True ：每次执行ansible命令时是否询问ssh密码
ask_pass = True
remote_port =22
host_key_checking = False # 检查对应服务器的host_key，建议取消注释
log_path=/var/log/ansible.log #日志文件
module_name = command #默认模块

### ansible系列命令
ansible ansible-doc ansible-playbook ansible-vault ansible-console ansible-galaxy ansible-pull

1. ansible-doc：显示模块帮助
```bash
-a ：显示所有的模块文档
-l，--list：列出可用模块
-s，--snippet：显示指定模块的playbook
```

示例
```bash
ansible-doc -l 列出所有模块
ansible-doc ping 查看指定模块帮助用法
ansible-doc -s ping 查看指定模块帮助用法
```

2. ansible
ansible 通过ssh实现配置管理、应用部署、任务执行等功能，建议配置ansible端能基于密钥认证的方式联系各个被管理节点
```bash
--version ：显示版本
-m module ：指定模块，默认为command
-v  ：详细过程，-vv -vvv更详细
--list-host ：显示主机列表，课简写--list
-k，--ask-pass ： 提示输入ssh连接密码，默认Key验证
-C，--check  ： 检查，并不执行
-T，--timeout=TIMEOUT ：执行命令的超时时间，默认10s
-u，--uses=REMOTE_USER ：执行远程执行的用户
-b，--become ：代替旧版的sudo切换
--become-user=username ： 指定sudo的runas用户，默认为root
-K，--ask-become-pass ： 提示输入sudo是的口令
```

3. ansible的host-pattern
匹配主机的列表
```bash
ALL: 表示所有inventory中的所有主机
	ansible all -m ping
通配符：*
	ansible "*" -m ping
	ansible 192.168.5.* -m ping
	ansible "*srvs" -m ping
或关系：
	ansible "websrvs:appsrvs" -m ping
	ansible "192.168.5.101:192.168.5.102" -m ping
逻辑与：
	ansible "websrvs:&dbsrvs" –m ping
		在websrvs组并且在dbsrvs组中的主机
逻辑非：
	ansible 'websrvs:!dbsrvs' –m ping
		在websrvs组，但不在dbsrvs组中的主机
		注意：此处为单引号
综合逻辑：
	ansible 'websrvs:dbsrvs:&appsrvs:!ftpsrvs' –m ping
正则表达式：
	ansible "websrvs:&dbsrvs" –m ping
	ansible "~(web|db).*\.magedu\.com" –m ping
```

4. ansible-galaxy
连接 https://galaxy.ansible.com 下载相应的roles
```bash
# 列出所有已安装的galaxy
ansible-galaxy list
# 安装galaxy
ansible-galaxy install geerlingguy.redis
# 删除galaxy
ansible-galaxy remove geerlingguy.redis
```

5. ansible-pull
推送命令至远程，效率无限提升，对运维要求较高
```bash
ansible-playbook
执行playbook
示例：ansible-playbook hello.yml（注意缩进问题）
	cat hello.yml
	#hello world yml file
		- hosts: webservers    指定要执行的主机
		  remote_user: root	   指定执行的用户
		  tasks:
	    - name: hello world
	      command: /usr/bin/wall hello world
```

6. ansible-vault
功能：管理加密解密yml文件
```bash
ansible-vault [create|decrypt|edit|encrypt|rekey|view]
ansible-vault encrypt hello.yml 加密(对称加密)
ansible-vault decrypt hello.yml 解密
ansible-vault view hello.yml 查看
ansible-vault edit hello.yml 编辑加密文件
ansible-vault rekey hello.yml 修改口令
ansible-vault create new.yml 创建新文件
```

7. Ansible-console：2.0+新增，可交互执行命令，支持tab
```bash
root@test (2)[f:10] $
执行用户@当前操作的主机组 (当前组的主机数量)[f:并发数]$
	设置并发数： forks n 例如： forks 10
	切换组： cd 主机组 例如： cd web
	列出当前组主机列表： list
	列出所有的内置命令： ?或help
示例：
	root@all (2)[f:5]$ list
	root@all (2)[f:5]$ cd appsrvs
	root@appsrvs (2)[f:5]$ list
	root@appsrvs (2)[f:5]$ yum name=httpd state=present
	root@appsrvs (2)[f:5]$ service name=httpd state=started
```

## ansible命令的执行过程
1. 加载自己的配置文件，默认/etc/ansible/ansible.cfg
2. 加载自己对应的模块配置，如command
3. 通过ansible将模块或命令生成对应的临时py文件，并将该文件传输至远程服务器的对应执行用户$HOME/.ansible/tmp/ansible-tmp-数字/XXX.PY文件
4. 给文件+x 并执行
5. 执行并返回结果
6. 删除临时py文件并退出

### ansible的执行状态
绿色：执行成功并且不需要做改变的操作
黄色：执行成功并且对目标主机做变更
红色：执行失败

### ansible使用示例
1. 以普通用户执行ping命令存活检测
```bash
[ root@localhost ~]# ansible all -m ping -u wei -k
SSH password: 
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
192.168.5.102 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

2. 以wei 用户sudo至root执行ping存活检测
```bash
[ root@localhost ~]# ansible all -m ping -u wei -k -b
SSH password:
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

3. 以 wei用户sudo到weiying用户执行ping存活检测
```bash
[ root@localhost ~]# ansible webservers -m ping -u wei -k -b --become-user=weiying
SSH password:
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

4. 以wei用户sudi值root用户执行ls
```bash
[ root@localhost ~]# ansible webservers -m command -u wei -a 'ls /root' -b --become-user=root -k -K
SSH password: 
BECOME password[defaults to SSH password]: 
192.168.5.101 | CHANGED | rc=0 >>
aa.sh
anaconda-ks.cfg
```
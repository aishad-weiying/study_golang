# handlers和notify结合使用触发条件

Handlers：是task列表，这些task与前述的task并没有本质上的不同,用于当关注的资源发生变化时，才会采取一定的操作

Notify此action可用于在每个play的最后被触发，这样可避免多次有改变发生时每次都执行指定的操作，仅在所有的变化发生完成后一次性地执行指定操作。在notify中列出的操作称为handler，也即notify中调用handler中定义的操作

## 以安装并配置httpd为例
```bash
[ root@localhost ~]# vim httpd.yaml

---

- hosts: all
  tasks:
  - name: "安装apache"
    yum: name=httpd
  - name: "复制文件"
    copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/
    notify: restart httpd  # 此处为指定的handlers的name
	# 只有当notify所在的name发生变化后才会执行执行的handlers中的任务
  - name: "启动apache，并设置开机自启动"
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart httpd
      service: name=httpd state=restarted
```

执行playbook验证
```bash
[ root@localhost ~]# ansible-playbook httpd.yaml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [安装apache] ****************************************************************************
ok: [192.168.5.101]
ok: [192.168.5.102]

TASK [复制文件] ********************************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

TASK [启动apache，并设置开机自启动] *******************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

RUNNING HANDLER [restart httpd] ************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
### notify触发多个handlers的执行
```bash
[ root@localhost ~]# vim httpd.yaml 

---

- hosts: all
  tasks:
  - name: "安装apache"
    yum: name=httpd
  - name: "复制文件"
    copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/
    notify:
      - restart httpd
      - check httpd process
  - name: "启动apache，并设置开机自启动"
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart httpd
      service: name=httpd state=restarted
    - name: check httpd process
      shell: killall -0 httpd && echo $?
```
执行playbook验证
```bash
[ root@localhost ~]# ansible-playbook httpd.yaml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.101]
ok: [192.168.5.102]

TASK [安装apache] ****************************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [复制文件] ********************************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

TASK [启动apache，并设置开机自启动] *******************************************************************
ok: [192.168.5.101]
ok: [192.168.5.102]

RUNNING HANDLER [restart httpd] ************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

RUNNING HANDLER [check httpd process] ******************************************************
changed: [192.168.5.101]
changed: [192.168.5.102]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=6    changed=3    unreachable=0    failed=0    skipped=0
```
# playbook中tags的使用
标签的作用是可以挑选性的执行playbook，而不是全部执行

每个不同的动作可以拥有相同的标签

```bash
[ root@localhost ~]# vim httpd.yaml 

---

- hosts: all
  tasks:
  - name: "安装apache"
    yum: name=httpd
    tags: install
  - name: "复制文件"
    copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/
    tags: copy
    notify:
      - restart httpd
      - check httpd process
  - name: "启动apache，并设置开机自启动"
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart httpd
      service: name=httpd state=restarted
    - name: check httpd process
      shell: killall -0 httpd && echo $?
```

挑选tags执行
```bash
# 指定多个标签是同逗号分隔标签 ansible-playbook -t install,cpoy httpd.yaml
[ root@localhost ~]# ansible-playbook -t install httpd.yaml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [安装apache] ****************************************************************************
changed: [192.168.5.101]
changed: [192.168.5.102]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

# playbook中变量的使用

变量名：仅能由字母、数字和下划线组成，且只能以字母开头

## 在playbook中使用变量

1. 通过{{ variable_name }} 调用变量，且变量名前后必须有空格，有时用"{{ variable_name }}"才生效

2. ansible-playbook test.yml -e "hosts=www user=linux"


## 变量来源

1. setup模块：获取所有主机上的各种详细信息，将其保存在变量中
```bash
#查看所有的变量
ansible all -m setup
ansible all -m setup -a 'filter="关键词或正则表达式"'
# 查看主机名：
[ root@localhost ~]# ansible all -m setup -a 'filter="ansible_hostname"'
192.168.5.102 | SUCCESS => {
    "ansible_facts": {
        "ansible_hostname": "localhost", 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "ansible_hostname": "node1", 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}

# 查看内存信息
[ root@localhost ~]# ansible all -m setup -a 'filter="ansible_memtotal_mb"'
192.168.5.102 | SUCCESS => {
    "ansible_facts": {
        "ansible_memtotal_mb": 1819, 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "ansible_memtotal_mb": 1819, 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}

# 查看系统版本
[ root@localhost ~]# ansible all -m setup -a 'filter="ansible_distribution_major_version"'
192.168.5.102 | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution_major_version": "7", 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution_major_version": "7", 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}
```

在ploybook中调用setup变量
```bash
# 使用当前的主机名创建文件
[ root@localhost ~]# vim var.yml
- hosts: all
  remote_user: root

  tasks:
    - name: create file
      file: name=~/{{ ansible_hostname }} state=touch

# 执行
ansible-playbook var.yml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [create file] *************************************************************************
changed: [192.168.5.101]
changed: [192.168.5.102]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

2. 在主机清单中定义变量

普通变量：变量名=值（可以对单个主机定义变量，放在单个主机后面）
```bash
[ root@localhost ~]# vim /etc/ansible/hosts
[webservers]
192.168.5.101 http_port=8080 maxRequestsPerChild=808

[dbservers]
192.168.5.102 http_port=8081 maxRequestsPerChild=909
# playbook中调用
[ root@localhost ~]# vim var.yml 

- hosts: all
  remote_user: root

  tasks:
    - name: create file
      file: name=~/{{ ansible_hostname }}{{ http_port }}{{ maxRequestsPerChild }} state=touch
```

公共变量：组变量是指赋予给指定组内所有主机上的在playbook中可用的变量
```bash
[ root@localhost ~]# vim /etc/ansible/hosts
[webservers:vars] # 针对webservers组中的所有主机都生效
test='_'
[webservers]
192.168.5.101 http_port=8080 maxRequestsPerChild=808
192.168.5.102 http_port=8081 maxRequestsPerChild=909
# 调用
[ root@localhost ~]# vim var.yml 

- hosts: all
  remote_user: root

  tasks:
    - name: create file
      file: name=~/{{ ansible_hostname }}{{ test }}{{ http_port }}{{ test }}{{ maxRequestsPerChild }} state=touch
```

> 普通变量的优先级大于公共变量

3. 命令行定义变量（优先级最高）
```bash
[ root@localhost ~]# vim var.yml 

- hosts: all
  remote_user: root

  tasks:
    - name: create file
      file: name=~/{{ ansible_hostname }}{{ test }}{{ http_port }}{{ test }}{{ maxRequestsPerChild }} state=touch

# 执行
[ root@localhost ~]# ansible-playbook -e http_port=9527 var.yml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [create file] *************************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

4. 在playbook中定义变量
```bash
- hosts: all
  remote_user: root

  vars:
    - http_port: 8000
    - maxRequestsPerChild: 999
  tasks:
    - name: create file
      file: name=~/{{ ansible_hostname }}{{ test }}{{ http_port }}{{ test }}{{ maxRequestsPerChild }} state=touch
```

5. 定义独立的yaml变量文件
```bash
[ root@localhost ~]# vim vars.yml
var1: httpd
var2: nginx
```
playbook文件中调用
```bash
[ root@localhost ~]# vim var.yml 

- hosts: all
  remote_user: root

  vars_files:
    - vars.yml
  tasks:
    - name: create file1
      file: name=~/{{ var1 }} state=touch
    - name: create file2
      file: name=~/{{ var2 }} state=touch
```

6：定义在role中

> 变量优先级：命令行>yaml变量文件中的变量>playbook>主机清单铺变量>主机清单公共变量
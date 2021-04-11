# 模板template
```bash
文本文件，嵌套有脚本（使用模板编程语言编写）
Jinja2语言，使用字面量，有下面形式
	字符串：使用单引号或双引号
	数字：整数，浮点数
	列表：[item1, item2, ...]
	元组：(item1, item2, ...)
	字典：{key1:value1, key2:value2, ...}
	布尔型：true/false
算术运算：+, -, *, /, //, %, **
比较操作：==, !=, >, >=, <, <=
逻辑运算：and，or，not
流表达式：For，If，When
```

template功能：根据模块文件动态生成对应的配置文件
template文件必须存放于templates目录下，且命名为 .j2 结尾
yaml/yml 文件需和templates目录平级，目录结构如下：
```bash
[ root@localhost http]# tree 
.
├── nginx.yml
└── templates
    └── nginx.conf.j2
```
## 以安装并配置nginx为例
1. 准备yml文件
```bash
vim nginx.yml

- hosts: all
  remote_user: root

  tasks:
    - name: "安装nginx"
      yum: name=nginx
    - name: "复制模板文件"
      template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
      notify: restart nginx
    - name: "启动apache，并设置开机自启动"
      service: name=nginx state=started enabled=yes
  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
```

2. 准备nginx.conf.j2的nginx配置文件
```bash
vim templates/nginx.conf.j2
# 根据cpu的数量设置进程数
worker_processes  {{ ansible_processor_vcpus*2 }};
# 调用在hosts文件中定义的变量
listen       {{ http_port }};
```

3. 执行后查看进程数是否为cpu个数的两倍
```bash
 root@node1 ~]# ps -ef | grep nginx
root      10942      1  0 18:37 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     10943  10942  0 18:37 ?        00:00:00 nginx: worker process
nginx     10944  10942  0 18:37 ?        00:00:00 nginx: worker process
nginx     10945  10942  0 18:37 ?        00:00:00 nginx: worker process
nginx     10946  10942  0 18:37 ?        00:00:00 nginx: worker process
root      10957   4507  0 18:37 pts/0    00:00:00 grep --color=auto nginx
```

4. 验证端口号
```bash
[ root@localhost http]# ansible all -m shell -a 'ss -tnlp| grep nginx'
192.168.5.101 | CHANGED | rc=0 >>
LISTEN     0      128          *:8080                     *:*                   users:(("nginx",pid=4867,fd=6),("nginx",pid=4866,fd=6),("nginx",pid=4865,fd=6),("nginx",pid=4864,fd=6),("nginx",pid=4863,fd=6),("nginx",pid=4862,fd=6),("nginx",pid=4861,fd=6))

192.168.5.102 | CHANGED | rc=0 >>
LISTEN     0      128          *:8081                     *:*                   users:(("nginx",pid=4885,fd=6),("nginx",pid=4884,fd=6),("nginx",pid=4883,fd=6),("nginx",pid=4882,fd=6),("nginx",pid=4881,fd=6),("nginx",pid=4880,fd=6),("nginx",pid=4879,fd=6))
```

# when 条件测试
条件测试:如果需要根据变量、facts或此前任务的执行结果来做为某task执行与否的前提时要用到条件测试,通过when语句实现，在task中使用，jinja2的语法格式

when语句：在task后添加when子句即可使用条件测试；when语句支持Jinja2表达式语法

示例：
```bash
tasks:
  - name: "shutdown RedHat flavored systems"
    command: /sbin/shutdown -h now
    when: ansible_os_family == "RedHat
```

## 使用when条件判断，判断不同的系统版本复制不同的模板
```bash
tasks:
  - name: install conf file to centos7
    template: src=nginx.conf.c7.j2 dest=/etc/nginx/nginx.conf
    when: ansible_distribution_major_version == "7"
  - name: install conf file to centos6
    template: src=nginx.conf.c6.j2 dest=/etc/nginx/nginx.conf
    when: ansible_distribution_major_version == "6"
```

根据不同版本的操作系统执行不同的命令
```bash
- hosts: all
  remote_user: root

  tasks:
    - name: add group nginx
      tags: group
      group: name=nginx state=present
    - name: add user nginx
      tags: user
      user: name=nginx state=present group=nginx
    - name: install nginx
      yum: name=nginx state=present
    - name: restart nginx
      shell: systemctl restart nginx
      when: ansible_distribution_major_version == "7"
    - name: restart nginx2
      shell: service nginx restart
      when: ansible_distribution_major_version == "6"
```

# 迭代：with_items
当有需要重复性执行的任务时，可以使用迭代机制

对迭代项的引用，固定变量名为”item“
要在task中使用with_items给定要迭代的元素列表
列表格式：字符串、字典

示例：
```bash
# 创建两个用户
- name: add user testuser1
 user: name=testuser1 state=present groups=wheel
- name: add user testuser2
 user: name=testuser2 state=present groups=wheel
# 使用迭代的方式创建
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
    - testuser1
    - testuser2
```
将多个文件进行copy到被控端
```bash
- hosts: testsrv
  remote_user: root

  tasks
    - name: Create rsyncd config
	  copy: src={{ item }} dest=/etc/{{ item }}
       with_items:
         - rsyncd.secrets
         - rsyncd.conf
    - name: install some packages
      yum: name={{ item }} state=present
      with_items:
        - nginx
        - memcached
        - php-fpm
```

## 迭代嵌套子变量
```bash
- hosts：websrvs
  remote_user: root
  tasks:
   - name: add some groups
     group: name={{ item }} state=present
     with_items:
       - group1
       - group2
       - group3
   - name: add some users
     user: name={{ item.name }} group={{ item.group }} state=present
     with_items:
       - { name: 'user1', group: 'group1' }
       - { name: 'user2', group: 'group2' }
       - { name: 'user3', group: 'group3' }
```

# Playbook中template for if

for循环的使用方法：
```bash
{% for 变量 in 变量列表 %}
# 调用变量
变量.变量列表中的变量
{% endfor %} # 结束
```

例如：创建下列各式的配置文件
```bash
servser{
	listen 80;
	server name www.a.com;
	root /data/webappa;
}
servser{
	listen 81;
	server name www.b.com;
	root /data/webappc;
}
servser{
	listen 82;
	server name www.c.com;
	root /data/webappc;
}

```

1. 在yml文件中定义变量
```bash
[ root@localhost http]# vim for_creat_vhost.yml 

- hosts: all
  remote_user: root
  vars:
    vhosts:
      - web1:
        port: 81
        na: www.a.com
        dir: /data/webappa
      - web2:
        port: 82
        na: www.b.com
        dir: /data/webappb
      - web1:
        port: 83
        dir: /data/webappc
  tasks:
    - name: templates
      template: src=vhosts.j2 dest=~/test.conf
```

2. template模板中使用for循环调用变量
```bash
[ root@localhost http]# vim templates/vhosts.j2

{% for i in vhosts %}

server{
    listen {{ i.port }};
    {% i.na is defined %}  # 如果i.na没有定义，那么servername这行就不会生成
        servername {{ i.na }};
    {% endfor %}
    root {{ i.dir }}
}

{% endfor %}

```

3. 执行playbook并验证结果
```bash
[ root@localhost http]# ansible-playbook for_creat_vhost.yml 

PLAY [all] *********************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [192.168.5.102]
ok: [192.168.5.101]

TASK [templates] ***************************************************************************
changed: [192.168.5.102]
changed: [192.168.5.101]

PLAY RECAP *********************************************************************************
192.168.5.101              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.5.102              : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

# 验证
[ root@localhost http]# ansible all -m shell -a "cat ~/test.conf"
192.168.5.102 | CHANGED | rc=0 >>

server{
    listen 81;
        servername www.a.com;
        root /data/webappa;
}


server{
    listen 82;
        servername www.b.com;
        root /data/webappb;
}


server{
    listen 83;
        root /data/webappc;   # 因为没有定义web3的na变量，那那么就不会生成servername行
}
```
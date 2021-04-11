# ansible 配置免秘钥登录
ansible 的主控节点要与ansible 的被控制节点之间配置免秘钥登录，否则只能在执行ansible命令的时候加-k参数
```bash
[ root@localhost ~]# ansible all -m ping -k
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
> 这种方式的话要保证所有的主机的密码都是相同的才可以

配置免秘钥登录
```bash
ssh-keygen
ssh-copy-id 192.168.5.101
ssh-copy-id 192.168.5.102
```


# ansible 的常用模块

1. command ：在远程主机执行命令，默认模块。可忽略-m选项
```bash
ansible webservers -m command -a 'cat /etc/shadow'
```
> 注意：此命令不支持 $VARNAME < > | ; & 等，可用shell模块实现

2. shell：与command相似，用shell执行命令
```bash
ansible all -m shell -a 'echo $PATH > /root/path.txt'
192.168.5.101 | CHANGED | rc=0 >>

192.168.5.102 | CHANGED | rc=0 >>
```
> 该模块会调用shell命令，但是，例如ansible all -m shell -a "awk -F: '$3>=1000{print $1}' /etc/passwd" 这种复杂的命令执行的时候也会报错，只能是写到脚本中，将脚本copy到各个节点，然后ansible 执行，再把结果从远程拉取到本地
```bash
[ root@localhost ~]# ansible webservers -m shell -a "awk -F: '$3>=1000{print $1}' /etc/passwd"
192.168.5.101 | FAILED | rc=1 >>
awk: cmd. line:1: >=1000{print }
awk: cmd. line:1: ^ syntax errornon-zero return code

[ root@localhost ~]# ansible webservers -m shell -a "cat ~/test.sh"
192.168.5.101 | CHANGED | rc=0 >>
awk -F: '$3>=1000{print $1}' /etc/passwd

[ root@localhost ~]# ansible webservers -m shell -a "bash ~/test.sh"
192.168.5.101 | CHANGED | rc=0 >>
wei
weiying

```

3. script：在远程脚本运行ansible 服务器上的脚本
```bash
[ root@localhost ~]# cat test.sh 
#!/bin/bash
awk -F: '$3>=1000{print $1}' /etc/passwd

[ root@localhost ~]# ansible webservers -m script -a "/root/test.sh"
192.168.5.101 | CHANGED => {
    "changed": true, 
    "rc": 0, 
    "stderr": "Shared connection to 192.168.5.101 closed.\r\n", 
    "stderr_lines": [
        "Shared connection to 192.168.5.101 closed."
    ], 
    "stdout": "wei\r\nweiying\r\n", 
    "stdout_lines": [
        "wei", 
        "weiying"
    ]
}
```

4. copy：把主控端的文件拷贝到远程主机，可对目录操作
拷贝本地指定文件到被控制端的服务器，并指定文件的权限、所有者和所属组，如目标存在，默认覆盖，可以执行先备份文件
```bash
[ root@localhost ~]# ansible webservers -m copy -a "src=/etc/ansible/hosts dest=/root/ansible_master owner=wei group=wei mode=600 backup=yes"
192.168.5.101 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "a6f3b4c9d161cf5ff68e2662cadcd88af8660163", 
    "dest": "/root/ansible_master", 
    "gid": 1000, 
    "group": "wei", 
    "mode": "0600", 
    "owner": "wei", 
    "path": "/root/ansible_master", 
    "size": 1053, 
    "state": "file", 
    "uid": 1000
}
```

5. fetch：从远程主机提取文件至主控制端，与copy相反，只能对文件操作，要是对目录操作的话需要先将目录打包
```bash
 root@localhost ~]# ansible webservers -m fetch -a "src=/root/test.sh dest=/root/test2.sh" 
192.168.5.101 | CHANGED => {
    "changed": true, 
    "checksum": "043b6b51330d99c08b6e47b3f904c8031ba4673f", 
    "dest": "/root/test2.sh/192.168.5.101/root/test.sh", 
    "md5sum": "3e8783c443f04c349ac8abea217c52a3", 
    "remote_checksum": "043b6b51330d99c08b6e47b3f904c8031ba4673f", 
    "remote_md5sum": null
}

ls test2.sh/192.168.5.101/root/test.sh
test2.sh/192.168.5.101/root/test.sh
```
6. file：设置文件属性
```bash
[ root@localhost ~]# ansible webservers -m shell -a "ls -l test.sh"
192.168.5.101 | CHANGED | rc=0 >>
-rw-r--r-- 1 root root 41 Jul 17 22:29 test.sh

[ root@localhost ~]# ansible webservers -m file -a "path=/root/test.sh owner=wei mode=755"
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "gid": 0, 
    "group": "root", 
    "mode": "0755", 
    "owner": "wei", 
    "path": "/root/test.sh", 
    "size": 41, 
    "state": "file", 
    "uid": 1000
}
```
创建软连接：
```bash
[ root@localhost ~]# ansible webservers -m file -a "src=/root/test.sh dest=/root/test-link state=link"
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "dest": "/root/test-link", 
    "gid": 0, 
    "group": "root", 
    "mode": "0777", 
    "owner": "root", 
    "size": 13, 
    "src": "/root/test.sh", 
    "state": "link", 
    "uid": 0
}

[ root@localhost ~]# ansible webservers -m shell -a "ls -l /root/test-link"
192.168.5.101 | CHANGED | rc=0 >>
lrwxrwxrwx 1 root root 13 Jul 17 23:10 /root/test-link -> /root/test.sh

# 软连接为link，硬链接为hard
```

7. hostname：管理主机名
```bash
[ root@localhost ~]# ansible webservers -m shell -a "hostname"
192.168.5.101 | CHANGED | rc=0 >>
localhost.localdomain

[ root@localhost ~]# ansible webservers -m hostname -a "name=node1"
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "ansible_domain": "", 
        "ansible_fqdn": "node1", 
        "ansible_hostname": "node1", 
        "ansible_nodename": "node1", 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "name": "node1"
}
[ root@localhost ~]# ansible webservers -m shell -a "hostname"
192.168.5.101 | CHANGED | rc=0 >>
node1
```

8. cron：计划任务
支持的时间：minute、hour、day、month、weekday
```bash
# 创建任务，每五分钟执行一次指定的脚本或命令
[ root@localhost ~]# ansible webservers -m cron -a "minute=*/5 job='/usr/bin/cat /etc/fstab &/dev/null' name=test"
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "envs": [], 
    "jobs": [
        "test"
    ]
}
[ root@localhost ~]# ansible webservers -m shell -a "crontab -l"
192.168.5.101 | CHANGED | rc=0 >>
#Ansible: test
*/5 * * * * /usr/bin/cat /etc/fstab &/dev/null

# 删除任务
[ root@localhost ~]# ansible webservers -m cron -a "state=absent name=test"
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "envs": [], 
    "jobs": []
}
[ root@localhost ~]# ansible webservers -m shell -a "crontab -l"
192.168.5.101 | CHANGED | rc=0 >>

```

9. yum：管理包
```bash
# 安装
ansible webservers -m yum -a "name=httpd state=present"

# 删除
ansible webservers -m yum -a "name=httpd state=absent"
```

10. service：管理服务
```bash
# 启动服务
ansible webservers -m service -a "name=httpd state=started"
# 停止服务
ansible webservers -m service -a "name=httpd state=stopped"
# 启动服务，并设为开机自启
ansible webservers -m service -a "name=httpd state=started enabled=yes"
# 重启
ansible webservers -m service -a "name=httpd state=restarted"
# 重读配置文件
ansible webservers -m service -a "name=httpd state=reloaded"
```

11. user：用户管理
```bash
# 创建普通用户，指定UID、描述信息、删除及家目录，指定的组必须是已存在的组
ansible webservers -m user -a "name=user1 comment='test' uid=9527 home=/home/user_test group=wei"
# create_home=no 不创建用户家目录
# 创建系统用户：（默认系统用户不创建家目录）
ansible webservers -m user -a "name=sysuser system=yes home=/home/sysuser"
# 删除用户及其家目录数据
ansible webservers -m user -a "name=user1 state=absent remove=yes"
```
12. group：组管理
```bash
# 创建组
[ root@localhost ~]# ansible webservers -m group -a "name=testgroup system=yes" 
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "gid": 995, 
    "name": "testgroup", 
    "state": "present", 
    "system": true
}
# 删除组
[ root@localhost ~]# ansible webservers -m group -a "name=testgroup state=absent" 
192.168.5.101 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "name": "testgroup", 
    "state": "absent"
}
```
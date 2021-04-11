# bash的条件测试
判断某需求是否满足，需要由测试机制来实现，专用的测试表达式需要由测试命令辅助完成测试过程

- 测试表达式的返回值
	为真：返回0
	为假：返回1

- 测试命令：
	test EXPRESSION
	[ EXPRESSION ]
	[[ EXPRESSION ]]
> EXPRESSION前后必须有空白字符

## bash数值测试
-v var：变量VAR是否设置

- -gt：是否大于
- -ge：是否大于等于
- -eq：是否等于
- -ne：是否不等于
- -lt：是否小于
- -le：是否小于等于

## bash的字符测试

- =：是否等于
- >：左边的ascii码是否大于右边ascii码
- <： 左边的ascii码是否小于右边ascii码
- !=：是否不等于
- =~：左侧字符串是否能够被右侧的PATTERN所匹配
	注意: 此表达式一般用于[[ ]]中；扩展的正则表达式
- -z "STRING": 字符串是否为空，空为真，不空为假
- -n "STRING": 字符串是否不空，不空为真，空为假

>注意：用于字符串比较时的用到的操作数都应该使用引号

## bash文件测试
###存在性测试

- -a FILE：同-e
- -e FILE: 文件存在性测试，存在为真，否则为假

### 存在性及类别测试：

- -b FILE：存在且为块设备
- -c FILE：存在且为字符设备
- -d FILE：是否存在且为目录文件
- -f FILE：是否存在且为普通文件
- -h FILE 或 -L FILE：存在且为符号链接文件
- -p FILE：是否存在且为命名管道文件
- -S FILE：是否存在且为套接字文件

### 文件权限测试

- -r FILE：是否存在且可读
- -w FILE: 是否存在且可写
- -x FILE: 是否存在且可执行

### 文件特殊权限测试

- -u FILE：是否存在且拥有suid权限
- -g FILE：是否存在且拥有sgid权限
- -k FILE：是否存在且拥有sticky权限

### 文件属性测试
- -s FILE: 是否存在且非空
- -t fd: fd 文件描述符是否在某终端已经打开
- -N FILE：文件自从上一次被读取之后是否被修改过
- -O FILE：当前有效用户是否为文件属主
- -G FILE：当前有效用户是否为文件属组
- FILE1 -ef FILE2: FILE1是否是FILE2的硬链接
- FILE1 -nt FILE2: FILE1是否新于FILE2（mtime）
- FILE1 -ot FILE2: FILE1是否旧于FILE2

# bash组合测试条件

- 第一种：必须使用测试命令进行，[[ ]] 不支持
EXPRESSION1 -a EXPRESSION2 并且
EXPRESSION1 -o EXPRESSION2 或者
! EXPRESSION 非

- 第二种
COMMAND1 && COMMAND2 并且，短路与，代表条件性的AND THEN
COMMAND1 || COMMAND2 或者，短路或，代表条件性的OR ELSE
! COMMAND 非
> 如：[ -f "$FILE" ] && [[ "$FILE" =~ .*\.sh$ ]]

## 组合条件示例
```bash
test "$A" = "$B" && echo "Strings are equal"
test "$A"-eq "$B" && echo "Integers are equal"
[ "$A" = "$B" ] && echo "Strings are equal"
[ "$A" -eq "$B" ] && echo "Integers are equal"
[ -f /bin/cat -a -x /bin/cat ] && cat /etc/fstab
[ -z "$HOSTNAME" -o $HOSTNAME "=="localhost.localdomain" ] && hostname www.weiying.com
```

### 条件性的执行操作符示例
```bash
[ root@weiying ~]# grep -q no_such_user /etc/passwd || echo 'No such user'
No such user

[ root@weiying ~]# ping -c1 -W2 station1 &> /dev/null \
> && echo "station1 is up" \
> || (echo 'station1 is unreachable'; exit 1)
station1 is unreachable
```

# 使用read命令来接受输入
使用read来把输入值分配给一个或多个shell变量
-p 指定要显示的提示
-s 静默输入，一般用于密码
-n N 指定输入的字符长度N
-d ‘字符’ 输入结束符
-t N TIMEOUT为N秒
read 从标准输入中读取值，给每个单词分配一个变量
>所有剩余单词都被分配给最后一个变量
```bash
read -p "Enter a filename: " FILE
```

### 练习
1、编写脚本 argsnum.sh，接受一个文件路径作为参数；如果参数个数小于1，则提示用户“至少应该给一个参数”，并立即退出；如果参数个数不小于1，则显示第一个参数所指向的文件中的空白行数
```bash
[ root@weiying ~]# cat argsnum.sh 
#!/bin/bash

[ $# -ge 1 ] && echo `grep '^$' "$1" | wc -l` || echo 'please input a argv'
[ root@weiying ~]# ./argsnum.sh 
please input a argv
[ root@weiying ~]# ./argsnum.sh /boot/grub2/grub.cfg
16
```
2、编写脚本 hostping.sh，接受一个主机的IPv4地址做为参数，测试是否可连通。如果能ping通，则提示用户“该IP地址可访问”；如果不可ping通，则提示用户“该IP地址不可访问”
```bash
[ root@weiying ~]# cat hostping.sh 
#!/bin/bash

read -p "please input a ipaddress: " ipadd
ping -c1 $ipadd &>/dev/null

[ $? -eq 0 ] && echo 'host is alive' || echo 'host is not alive'

[ root@weiying ~]# ./hostping.sh 
please input a ipaddress: www.baidu.com
host is alive
[ root@weiying ~]# ./hostping.sh 
please input a ipaddress: 1.1.1.1
host is alive
[ root@weiying ~]# ./hostping.sh 
please input a ipaddress: 192.168.1.1
host is not alive
```
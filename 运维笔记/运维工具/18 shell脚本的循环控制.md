# 流程中的逻辑处理
顺序执行
选择执行
循环执行

## bash中的选择执行

### if语句
1. 但分值if语句
```bash
if 判断条件;then
	当能够匹配条件时执行的语句
fi
```
2. 多分支if语句
```bash
if 条件1;then
	符合条件1时执行的命令
elif 条件2;then
	符合条件2时执行的命令
else
	以上条件都不符合时执行的mingl
fi
```

### case语句：多分支条件判断
```bash
case 变量引用 in 
patter1)
	分支1
	;;
patter2)
	分支2
	;;
*)
	默认分支
	;;
esac
#注意：此处的模式不是正则表达式中的模式，而是支持glob通配符
#	case支持glob风格的通配符：
#		*：任意长度的任意字符；
#		?：任意单个字符；
#		[]：范围内任意单个字符；
#		a|b：a或b；
```

## 循环
将某代码段重复运行多次
有进入条件和退出条件

### for循环
```bash
for 变量名 in 列表;do
	循环体
done
```
#### 执行机制
依次将列表中的元素赋值给指定的变量名，每次赋值后就执行一次循环体，直到列表中的元素耗尽，循环结束

##### 列表的生成方式

1. 直接给出列表

2. 整数列表
	{起始值....结束值}
	$(seq [start [step]] end)

3. 返回列表的命令
	$(COMMADN)

4. 使用glob，如：*.sh

5. 变量引用
	$@、$*

### for的特殊格式

- 双小括号方法：
即((…))格式，也可以用于算术运算，双小括号方法也可以使bash Shell实现C语言风格的变量操作，例如((I++))

- for循环的特殊格式
```bash
for ((控制变量初始化;条件判断表达式;控制变量的修正表达式));do
	循环体
done
```

- 控制变量初始化
仅在运行到循环代码段时执行一次

- 控制变量的修正表达式
每轮循环结束会先进行控制变量修正运算，而后再做条件判断

## for循环实例
1. 添加10个用户user1-user10，密码为8位随机字符
```bash
#!/bin/bash

for i in user{1..10};do
	useradd $i
	echo `openssl rand -base64 4` | passwd --stdin $i
done
```

2. 编写脚本，提示请输入网络地址，如192.168.0.0，判断输入的网段中主机在线状态
```bash
#!/bin/bash
read -p "please input internal ip example 192.168.0.0: " ipadd
ipnet=`echo $ipaddr|cut -d. -f1-3`
for i in $ipnet.{1..255};do
	ping -c1 $i &> /dev/null && echo "$i is alive" || echo "$i is not alive"
done
```

## while循环
```bash
while CONDITION; do
	循环体
done
```
CONDITION：循环控制条件，进入循环之前，先做一次判断，每次循环之后会再次做判断，条件为true，则执行一次循环，直到条件测试状态为false，终止循环
> CONDITION一般应该有循环控制变量，因此变量的值会在循环体中不断被修正

while循环的实例：
1. 添加10个用户user1-user10，密码为8位随机字符
```bash
#!/bin/bash

i=1
while $i<=10;do
	useradd user$i
	echo `openssl rand -hex 8` | passwd --stdin user$i
	let i++
done
```
## until循环
```bash
until CONDITION;do
	循环体
done
```
> 当条件表达式为false时进入循环，当条件表达式为true时退出循环

## 循环控制

### continue
结束本轮循环，进入下一次循环
```bash
while CONDTIITON1; do
	CMD1
	...
	if CONDITION2; then
		continue
	fi
	CMDn
	...
done
```

### break
结束break所在的循环

### shift
用于将参量列表 list 左移指定次数，默认为左移一次
> 参量列表 list 一旦被移动，最左端的那个参数就从列表中删除。while 循环遍历位置参量列表时，常用到 shift
```bash
#!/bin/bash
#step through all the positional parameters
until [ -z "$1" ]
do
 echo "$1"
 shift
done
echo 

./shfit.sh a b c d e f g h  #将会依次输出命令行的各个参数
```

### 循环控制实例
1. 每隔3秒钟到系统上获取已经登录的用户的信息；如果发现用户 hacker 登录，则将登录时间和主机记录于日志/var/log/login.log中,并退出脚本
```bash
#!/bin/bash

while true;do
	w | awk -F' ' '{print $1}'| grep hacker
	if [ $? -eq 1 ];then
		echo "hacker is login" >> /var/log/login.log
		break
	fi
	sleep 3
```

## while循环的特殊用法
遍历文件的每一行
```bash
while read line;do
	循环体
done < file_name
```
> 依次读取指定文件中的每一行，且将行赋值给变量line

## select 循环与菜单
select 循环主要用于创建菜单，按数字顺序排列的菜单项将显示在标准错误上，并显示 PS3 提示符，等待用户输入,select 经常和 case 联合使用
```bash
select variable in list;do
	循环体命令
done
```
用户输入菜单列表中的某个数字，执行相应的命令
用户输入被保存在内置变量 REPLY 中
select 是个无限循环，因此要记住用 break 命令退出循环，或用 exit 命令终止脚本。也可以按 ctrl+c 退出循环
# 使用zabbix监控memcached

1. 安装memcached
```bash
yum install  memcached
systemctl ensable  memcached.service 
systemctl enable  memcached.service
```

2. 编写脚本，获取memcached的状态
使用ncat(ubuntu为nc)命令
echo -e "stats\nquit"| ncat 127.0.0.1 "11211"
```bash
vim /etc/zabbix/zabbix_agentd.d/memcached.sh
 #!/bin/bash
[ root@node1 zabbix_agentd.d]# bash -x memcached.sh 
[ root@node1 zabbix_agentd.d]# vim memcached.sh 

#!/bin/bash
memcached_status(){
    M_PORT=$1
    M_COMMAND=$2
    echo -e "stats\nquit" | nc 127.0.0.1 "$M_PORT" | grep "STAT $M_COMMAND " | awk '{print $3}'
}

main(){
    case $1 in
        memcached_status)
            memcached_status $2 $3
        ;;
    esac
}

main $1 $2 $3

chmod +x  /etc/zabbix/zabbix_agentd.d/memcached.sh
```

3. 配置agent调用该脚本
```bash
vim /etc/zabbix/zabbix_agentd.d/agent_memcached_status.conf
UserParameter=memcached_status[*],/etc/zabbix/zabbix_agentd.d/memcached.sh $1 $2 $3

systemctl restart zabbix-agent.service 
```

4. server端验证
```bash
/apps/zabbix_server/bin/zabbix_get -s 192.168.6.103 -p 10050 -k "memcached_status["memcached_status","11211","curr_connections"]"
10
```

5. 创建模板、监控项和图像
![](images/7ee0ec6d3c8d6af67258b1705fdb27ff.png)

6. 关联主机并验证
![](images/f0d4a1b2ecedf1e7bb842ea9c1aa9ae1.png)


# zabbix监控redis
监控连接数和内存使用


1. 安装并启动redis
```bash
yum -y install redis

systemctl restart redis
```

2. 编写脚本，获取redis的状态
通过非交互的方式取值 (echo -en "INFO \r\n";sleep 1;) | nc 127.0.0.1 "6379"
```bash
[ root@node1 zabbix_agentd.d]# vim /etc/zabbix/zabbix_agentd.d/redis.sh

#!/bin/bash
redis_status(){
    R_PORT=$1
    R_COMMAND=$2
    (echo -en "INFO \r\n";sleep 1;) | nc 127.0.0.1 "$R_PORT" > /tmp/redis_"$R_PORT".tmp
    REDIS_STAT_VALUE=$(grep ""$R_COMMAND":" /tmp/redis_"$R_PORT".tmp | cut -d ':' -f2)
    echo $REDIS_STAT_VALUE
}

help(){
    echo "${0} + redis_status + PORT + COMMAND"
}
main(){
    case $1 in
        redis_status)
            redis_status $2 $3
            ;;
        *)
            help
            ;;
    esac
}

main $1 $2 $3

chmod +x /etc/zabbix/zabbix_agentd.d/redis.sh
```

3. 让agent调用该脚本
```bash
vim /etc/zabbix/zabbix_agentd.d/agent_redis_status.conf
UserParameter=redis_status[*],/etc/zabbix/zabbix_agentd.d/redis.sh $1 $2 $3

systemctl restart zabbix-agent.service 
```

4. server端测试
```bash
/apps/zabbix_server/bin/zabbix_get -s 192.168.6.103 -p 10050 -k "redis_status[redis_status,6379,rdb_last_save_time]"
1563177139
```

5. 添加监控模板、监控项和图形
![](images/8f8043563ac297cce23dde7bdb52e12a.png)

6. 关联到主机并验证
![](images/3595c1b21ad6e229d3f17e01118f978c.png)

# 邮件报警通知配置

1. 配置中创建报警媒介类型
![](images/2d06718b9b1d622a943d369905102efe.png)

2. 在用户设置中给用户添加报警媒介
![](images/8c4efcac8cfeeb0f43dd95e8a30d74ea.png)

3. 配置报警动作
![](images/6513d7c117731854d3e5d36701f1dfc4.png)
定义操作
![](images/14479dcef2a56f72378d67279c5359d9.png)
![](images/0c7a058e129ea78c854e1c85745fcebe.png)
每60s为一次操作，前三次操作每次发送一封邮件

4. 测试能否发送邮件
![](images/f261366f29043eefa029d892e783bb8d.png)
主机中关联了nginx检查模板，当关闭80端口的时候会报警，关闭80端口测试
![](images/455b867f54a6bd6f76cbed174bbf80b4.png)

5. 配置业务恢复通知
![](images/0b5d5b1891fdfd972e1e26ef92fca20f.png)
再次关闭nginx测试报警信息
![](images/41587a629e7e8a87cb1192949de69d99.png)

启动nginx测试恢复通知
![](images/5f130d661bd873f881ffe5aaf8f16289.png)
恢复时只会发送一次邮件

6. 配置nginx等服务的故障自治愈
前提：agent运行远程执行命令，特殊命令zabbix要在sudo配置文件中运行无密码执行

新建触发器动作
![](images/ec82464912145552c1a225f889f50269.png)

配置操作
![](images/74236e0f65712e032cd149c6d9bb235a.png)

配置agent
```bash
vim /etc/zabbix/zabbix_agentd.conf
EnableRemoteCommands=1   # 运行执行远程命令
UnsafeUserParameters=1   # 执行远程命令的时候允许特殊字符

systemctl restart zabbix-agent.service
```

给zabbix用户授权
```bash
vim /etc/sudoers
zabbix  ALL=(ALL)   NOPASSWD:ALL
```

关闭nginx测试能否收到邮件并重启nginx
![](images/94b10a061763536d74b71ff59a62fa0d.png)
![](images/70f4903b75c84058fc3b575fe1f23984.png)
![](images/fc3fd4970166047a522e731e9cc3837c.png)

测试成功
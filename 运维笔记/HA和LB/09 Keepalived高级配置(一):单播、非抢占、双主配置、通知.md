Keepalived高级配置(一)：单播、非抢占、双主配置、通知

## 单播配置
	keepalived默认使用组播地址进行探测，这样会导致当virtual_router_id配置相同时会导致冲突，而且这些报文在网络中是广播的，每个主机都能收到，会导致浪费大量的网络带宽，所以配置成单播地址

- 组播报文：

[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/组播.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/组播.png)

```bash
 unicast_src_ip 本机源IP
 	unicast_peer {
 		目标主机IP
	 }
	# 禁用vrrp_strict
	# master节点配置
		vrrp_instance VI_1 {
			state MASTER
			interface eth0
			virtual_router_id 45
			priority 100
			advert_int 1
			unicast_src_ip 172.20.45.131 # backup节点为172.20.45.132
				unicast_peer {
				172.20.45.132 # backup节点为172.20.45.131
			}
			authentication {
				auth_type PASS
				auth_pass admin123
			}
			virtual_ipaddress {
				172.20.45.248 dev eth0 label eth0:0
			}
		}
```
- 单播报文：

[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/单播.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/单播.png)

## Keepalivde 双主配置
	两个或以上VIP分别运行在不同的keepalived服务器，以实现服务器并行提供web访问的目的，提高服务器资源利用率。

- master节点配置
```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 45
    priority 100
    advert_int 1
    unicast_src_ip 172.20.45.132
        unicast_peer {
        172.20.45.131
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
	172.20.45.248 dev eth0 label eth0:1
    }

}
vrrp_instance VI_2 {
    state BACKUP
    interface eth0
    virtual_router_id 144
    priority 80
    advert_int 1
    unicast_src_ip 172.20.45.132
        unicast_peer {
        172.20.45.131
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
        172.20.45.249 dev eth0 label eth0:2
    }

}
```

- backup节点配置：
```bash
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 45
    priority 80
    advert_int 1
    unicast_src_ip 172.20.45.132
        unicast_peer {
        172.20.45.131
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
	172.20.45.248 dev eth0 label eth0:1
    }

}
vrrp_instance VI_2 {
    state MASTER
    interface eth0
    virtual_router_id 144
    priority 100
    advert_int 1
    unicast_src_ip 172.20.45.132
        unicast_peer {
        172.20.45.131
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
        172.20.45.249 dev eth0 label eth0:2
    }

}
```

## 非抢占:默认为抢占
	抢占：当优先级高的keepalived的服务器宕机后，vip会移动到优先级低的节点上，继续提供服务，当优先级高的服务器启动后，vip的地址会再次被抢占到优先级高的服务器上

- nopreempt #关闭VIP抢占，需要VIP state都为BACKUP，通过优先级的高低来判断vip默认的工作主机

```bash
# master节点
	vrrp_instance VI_1 {
		state BACKUP
		interface eth0
		virtual_router_id 45
		priority 100
		advert_int 1
		nopreempt
		unicast_src_ip 172.20.45.132
			unicast_peer {
			172.20.45.131
		}
		authentication {
			auth_type PASS
			auth_pass admin123
		}
		virtual_ipaddress {
		172.20.45.248 dev eth0 label eth0:1
		}

	}
# BACKUP节点
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 45
    priority 80
    advert_int 1
	nopreempt
    unicast_src_ip 172.20.45.132
        unicast_peer {
        172.20.45.131
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
        172.20.45.248 dev eth0 label eth0:1
    }

}
```

## Keepalived通知配置

- 发件人配置:
```bash
[ root@localhost ~]# cat /etc/mail.rc
	set from=2286416563@qq.com
	set smtp=smtp.qq.com
	set smtp-auth-user=2286416563@qq.com
	set smtp-auth-password=tzglhzpcrojteaia #授权码在邮箱设置中获取
	set smtp-auth=login
	set ssl-verify=ignore
#测试：
[ root@localhost ~]# 	echo "test" | mail -s "weiying" 2286416563@qq.com
```

- nopreempt：定义工作模式为非抢占模式

- preempt_delay 300：抢占式模式，节点上线后触发新选举操作的延迟时长，默认模式

- 定义通知脚本：

1. notify_master <STRING\>|<QUOTED-STRING\>：
	当前节点成为主节点时触发的脚本

2. notify_backup <STRING\>|<QUOTED-STRING\>：
	当前节点转为备节点时触发的脚本

3. notify_fault <STRING\>|<QUOTED-STRING\>：
	当前节点转为“失败”状态时触发的脚本

4. notify <STRING\>|<QUOTED-STRING\>：
	通用格式的通知触发机制，一个脚本可完成以上三种状态的转换时的通知

- Keepalived通知脚本
```bash
[root@localhost keepalived]# cat /etc/keepalived/notify.sh
#!/bin/bash
contact='2286416563@qq.com'
notify() {
mailsubject="$(hostname) to be $1, vip 转移"
mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"
echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
notify master
;;
backup)
notify backup
;;
fault)
notify fault
;;
*)
echo "Usage: $(basename $0) {master|backup|fault}"
exit 1
;;
esac
[root@localhost keepalived]# chmod  +x /etc/keepalived/notify.sh
```

- 脚本的调用方法：
	notify_master "/etc/keepalived/notify.sh master"
	notify_backup "/etc/keepalived/notify.sh backup"
	notify_fault "/etc/keepalived/notify.sh fault"
```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 45
    priority 100
    advert_int 1
    unicast_src_ip 172.20.45.131
        unicast_peer {
        172.20.45.132
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
        172.20.45.248 dev eth0 label eth0:1
    }

    notify_master "/etc/keepalived/notify.sh master"
    notify_backup "/etc/keepalived/notify.sh backup"
    notify_fault "/etc/keepalived/notify.sh fault"
}
```
- 停止keepalived服务，验证IP 切换后是否收到通知邮件：

[![](http://aishad.top/wordpress/wp-content/uploads/2019/06/邮件.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/邮件.png)

## 配置众多的keepalived
	配置keepalived两个节点的集群环境仍然是有两个keepalived都会宕机的可能，这样的话就需要配置3个或更多的keepalived

- 配置方法
```bash
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 45
    priority 70 #master为100 ，第一个BACKUP为80
    advert_int 1
    unicast_src_ip 172.20.45.133
        unicast_peer {
        172.20.45.131  # 对应的master和其他的backup也要讲新加的backup加进去
		172.20.45.132
    }
    authentication {
        auth_type PASS
        auth_pass admin123
    }
    virtual_ipaddress {
        172.20.45.248 dev eth0 label eth0:1
    }
}
```
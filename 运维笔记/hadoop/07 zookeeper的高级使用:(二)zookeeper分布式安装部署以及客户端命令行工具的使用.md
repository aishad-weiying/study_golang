# 分布式安装zookeeper

1. 修改zoo.cfg配置文件
```bash
# 增加集群的配置信息
vim zoo.cfg

server.1=192.168.1.100:2888:3888
server.2=192.168.1.101:2888:3888
server.3=192.168.1.102:2888:3888

# 配置参数说明

#server.id=ip或者主机名:port1:port2

# id:是一个数字，表示这个是第几号服务器,这个需要在集群的dataDir目录下,配置myid这个文件,在文件中标明该节点的id,Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server

# port1:这个是服务器与集群中的Leader服务器交换信息的端口

# port2:万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行选举时服务器相互通信的端口

```

2. 配置myid文件
```bash
# server1 执行
echo 1 > /usr/local/src/zookeeper/data/zookeeper_data/myid
# server2 执行
echo 2 > /usr/local/src/zookeeper/data/zookeeper_data/myid
# server3 执行
echo 3 > /usr/local/src/zookeeper/data/zookeeper_data/myid
```

3. 分别在各个节点上重新启动zookeeper
```bash
# server1

root@ubuntu:/usr/local/src/zookeeper# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/src/zookeeper/bin/../conf/zoo.cfg
Mode: follower

# server2

root@ubuntu:/usr/local/src/zookeeper# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/local/src/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
root@ubuntu:/usr/local/src/zookeeper# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/src/zookeeper/bin/../conf/zoo.cfg
Mode: leader


# server3

root@ubuntu:/usr/local/src/zookeeper# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/local/src/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
root@ubuntu:/usr/local/src/zookeeper# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/src/zookeeper/bin/../conf/zoo.cfg
Mode: follower

```


# 命令行客户端的使用

使用zookeeper的客户端工具 zkCli.sh

1. help
```bash
# 显示所有操作命令
[zk: localhost:2181(CONNECTED) 0] help
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]		# 设置路径
	ls path [watch]		# 查看路径
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path	# 删除路径
	get path [watch]	# 获取路径信息
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port    # 连接其它的节点
```

2. ls
```bash
# 查看当前znode中所包含的内容
[zk: localhost:2181(CONNECTED) 2] ls /
[zookeeper]
```

3. ls2
```bash
# 查看当前节点数据并能看到更新次数等数据
[zk: localhost:2181(CONNECTED) 3] ls2 /
[zookeeper]
cZxid = 0x0  事务创建的编号
ctime = Thu Jan 01 08:00:00 CST 1970 创建的时间
mZxid = 0x0 znode最后修改的事务编号
mtime = Thu Jan 01 08:00:00 CST 1970 znode最后修改时间
pZxid = 0x0 znode最后更新的子节点zxid
cversion = -1 znode子节点变化号，每变化一次就自增1
dataVersion = 0  znode数据变化号，数据每变化一次就自增1
aclVersion = 0   znode访问控制列表的变化号
ephemeralOwner = 0x0  如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0
dataLength = 0  znode的数据长度
numChildren = 1  znode子节点数量
```

4. create [-s] [-e] path data acl
```bash
# 创建一个普通的节点
[zk: localhost:2181(CONNECTED) 5] create /app "test app"
Created /app
[zk: localhost:2181(CONNECTED) 6] ls /
[app, zookeeper]
[zk: localhost:2181(CONNECTED) 7] ls2 /app
[]
cZxid = 0x400000002
ctime = Mon Oct 14 14:12:19 CST 2019
mZxid = 0x400000002
mtime = Mon Oct 14 14:12:19 CST 2019
pZxid = 0x400000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 8
numChildren = 0 
```

5. get 获取节点的值
```bash
[zk: localhost:2181(CONNECTED) 9] get /app 
test app
cZxid = 0x400000002
ctime = Mon Oct 14 14:12:19 CST 2019
mZxid = 0x400000002
mtime = Mon Oct 14 14:12:19 CST 2019
pZxid = 0x400000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 8
numChildren = 0
```

6. 创建短暂节点
```bash
[zk: localhost:2181(CONNECTED) 10] create -e /app-emphemeral "test emphemeral"
Created /app-emphemeral
[zk: localhost:2181(CONNECTED) 11] ls /
[app, zookeeper, app-emphemeral]
# 退出当前的客户端再次登录验证
[zk: localhost:2181(CONNECTED) 12] quit

# 重新登录
root@ubuntu:/usr/local/src/zookeeper# bin/zkCli.sh
[zk: localhost:2181(CONNECTED) 0] ls /
[app, zookeeper]
# 创建的短暂节点被删除
```

7. 创建带序号的节点
```bash
[zk: localhost:2181(CONNECTED) 1] create -s /app2 "test sequential"
Created /app20000000002
[zk: localhost:2181(CONNECTED) 2] ls /
[app, app20000000002, zookeeper]
# 在创建的节点后面自动增加编号,编号自动增加
```

8. set 修改节点中数据的值
```bash
[zk: localhost:2181(CONNECTED) 7] set /app "test2 app"
cZxid = 0x400000002
ctime = Mon Oct 14 14:12:19 CST 2019
mZxid = 0x400000008
mtime = Mon Oct 14 14:31:20 CST 2019
pZxid = 0x400000002
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
[zk: localhost:2181(CONNECTED) 8] get /app
test2 app
cZxid = 0x400000002
ctime = Mon Oct 14 14:12:19 CST 2019
mZxid = 0x400000008
mtime = Mon Oct 14 14:31:20 CST 2019
pZxid = 0x400000002
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
```

9. 节点的值变化监听,注册一次仅生效一次
```bash
# 在server3 上监听/app的数据的变化
[zk: localhost:2181(CONNECTED) 0] get /app watch

# 在 server1 上修改/app的值
[zk: localhost:2181(CONNECTED) 9] set /app "test3 app"
cZxid = 0x400000002
ctime = Mon Oct 14 14:12:19 CST 2019
mZxid = 0x40000000a
mtime = Mon Oct 14 14:34:02 CST 2019
pZxid = 0x400000002
cversion = 0
dataVersion = 2
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0

# server3上监听到数据的变化
[zk: localhost:2181(CONNECTED) 1] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/ap
```

10. 点的子节点变化监听(路径变化),注册一次仅生效一次
```bash
# 在server3 上监听/app节点上子节点的变化
[zk: localhost:2181(CONNECTED) 1] ls /app watch
[]

# 在server1 上/app节点上创建子节点
[zk: localhost:2181(CONNECTED) 10] create /app/app1 "test app-1"
Created /app/app1

# 在server3 上监听到子节点的变化
[zk: localhost:2181(CONNECTED) 2] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/app

```

11. 删除节点
```bash
[zk: localhost:2181(CONNECTED) 11] ls /
[app, app20000000002, zookeeper, app20000000003]
[zk: localhost:2181(CONNECTED) 12] delete /app20000000002
```

12. 递归删除节点
```bash
# 节点中有子节点使用delete不能删除
[zk: localhost:2181(CONNECTED) 15] ls /app
[app1]
[zk: localhost:2181(CONNECTED) 16] delete /app
Node not empty: /app
# 递归删除节点以及下面的子节点
[zk: localhost:2181(CONNECTED) 17] rmr /app

```

13. 节点的状态查看
```bash
[zk: localhost:2181(CONNECTED) 20] stat /app20000000003
cZxid = 0x400000007
ctime = Mon Oct 14 14:23:50 CST 2019
mZxid = 0x400000007
mtime = Mon Oct 14 14:23:50 CST 2019
pZxid = 0x400000007
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 15
numChildren = 0
```

## stat 结构体说明

1. cZxid:创建节点事务的zxid，标记此节点由哪个id所表示的事务创建产生的，事务id是和会话相关的

2. ctime - znode被创建的毫秒数(从1970年开始)

3. mzxid - 最后一次更新该节点的事务id

4. mtime - znode最后修改的毫秒数(从1970年开始)

5. pZxid-  该节点的子节点列表最后被更新的事务id

6. cversion - znode子节点变化号，znode子节点修改次数

7. dataversion - znode数据变化号

8. aclVersion - znode访问控制列表的变化号

9. ephemeralOwner- 如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0

10. dataLength- znode的数据长度

11. numChildren - znode子节点数量

## zookeeper的四字命令
主要用于zookeeper的监控

1. 使用stat显示连接信息
```bash
root@ubuntu:/usr/local/src/zookeeper# telnet 127.0.0.1 2181
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
stat
Zookeeper version: 3.4.14-4c25d480e66aadd371de8bd2fd8da255ac140bcf, built on 03/06/2019 16:18 GMT
Clients:
 /127.0.0.1:55462[0](queued=0,recved=1,sent=0)

Latency min/avg/max: 0/0/0
Received: 4
Sent: 3
Connections: 1
Outstanding: 0
Zxid: 0x400000011
Mode: follower
Node count: 5
Connection closed by foreign host.
```

2. 使用srvr显示server信息
```bash
root@ubuntu:/usr/local/src/zookeeper# telnet 127.0.0.1 2181
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
srvr
Zookeeper version: 3.4.14-4c25d480e66aadd371de8bd2fd8da255ac140bcf, built on 03/06/2019 16:18 GMT
Latency min/avg/max: 0/0/0
Received: 5
Sent: 4
Connections: 1
Outstanding: 0
Zxid: 0x400000011
Mode: follower
Node count: 5
Connection closed by foreign host.
```

3. 使用conf显示配置信息
```bash
root@ubuntu:/usr/local/src/zookeeper# telnet 127.0.0.1 2181
root@ubuntu:/usr/local/src/zookeeper# telnet 127.0.0.1 2181
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
conf
clientPort=2181
dataDir=/usr/local/src/zookeeper/data/zookeeper_data/version-2
dataLogDir=/usr/local/src/zookeeper/log/version-2
tickTime=2000
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
serverId=1
initLimit=10
syncLimit=5
electionAlg=3
electionPort=3888
quorumPort=2888
peerType=0
```

4. 使用cons显示当前连接的客户端
```bash
root@ubuntu:/usr/local/src/zookeeper# telnet 127.0.0.1 2181
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
cons
 /127.0.0.1:55468[0](queued=0,recved=1,sent=0)

Connection closed by foreign host.
```

5. ruok：查看server是否正常在线

6. wchs：查看服务器有哪些监听器

7. envi：显示java运行的java jdk的环境配置信息
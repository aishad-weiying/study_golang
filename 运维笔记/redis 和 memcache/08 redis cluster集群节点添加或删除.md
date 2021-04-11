# redis cluster集群节点维护
	集群运行时间长久之后，难免由于硬件故障、网络规划、业务增长等原因对已有集群进行相应的调整，比如增加 Redis node 节点、减少节点、节点迁移、更换服务器等。
	增加节点和删除节点会涉及到已有的槽位重新分配及数据迁移。

> 1.增加节点肯定会造成数据的丢失，因为要随机的将槽位分配给新加入的节点，即使使用RDB或者AOF文件，也会造成另外两个节点上的数据的丢失
> 2.删除节点，将指定master上的槽位移动到指定的master上，利用RDB或者AOF文件可以做到数据不丢失

### 集群维护之动态添加节点
	增加 Redis node 节点，需要与之前的 Redis node 版本相同、配置一致，然后启动 Redis node，6380和6379端口，实现一主一从。

1. 配置新节点为master，并开启集群模式，设置masterauth密码

2. 添加新节点到集群
	新加172.20.45.137:6379为master，172.20.45.137:6380为slave
```bash
	[ root@localhost ~]# redis-trib.rb add-node 172.20.45.137:6379 172.20.45.131:6379
	[ root@localhost ~]# redis-trib.rb check 172.20.45.131:6379
	>>> Performing Cluster Check (using node 172.20.45.131:6379)
	S: cc7df73e121192185bd31d40859103b7a0c68bdd 172.20.45.131:6379
	   slots: (0 slots) slave
	   replicates 1289987119bae49782002768a35d90c52d3f2d75
	S: 23bc01f789eb7e624dd4288ac548a63f545678cb 172.20.45.136:6379
	   slots: (0 slots) slave
	   replicates afa67c80fa1dcaec758ae54b0760d466096ac3ff
	S: af8aff4dba8f20881218ea6ea2143cd3b5b143cf 172.20.45.134:6379
	   slots: (0 slots) slave
	   replicates 325b2cabe287c2bac388eae12598ea661edea46b
	M: afa67c80fa1dcaec758ae54b0760d466096ac3ff 172.20.45.132:6379
	   slots:5461-10922 (5462 slots) master
	   1 additional replica(s)
	M: 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7 172.20.45.137:6379
	   slots: (0 slots) master   ##没有槽位也没有丛节点
	   0 additional replica(s)
	M: 1289987119bae49782002768a35d90c52d3f2d75 172.20.45.135:6379
	   slots:0-5460 (5461 slots) master
	   1 additional replica(s)
	M: 325b2cabe287c2bac388eae12598ea661edea46b 172.20.45.133:6379
	   slots:10923-16383 (5461 slots) master
	   1 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
```

> Redis 5 添加方式：redis-cli -a 123456 --cluster add-node 192.168.7.104:6379 192.168.7.101:6379

3. 添加6380的端口
	redis-trib.rb add-node 172.20.45.137:6380 172.20.45.131:6379

> redis 5 :redis-cli -a 123456 --cluster add-node 172.20.45.137:6380 172.20.45.131:6379

4. 将6380改为137的6379的slave节点
	172.20.45.137:6380> CLUSTER REPLICATE 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7(137的6379的id)

4. 分配槽位
	添加主机之后需要对添加至集群种的新主机重新分片否则其没有分片
```bash
[ root@localhost ~]# redis-trib.rb reshard 172.20.45.137:6379
>>> Performing Cluster Check (using node 172.20.45.137:6379)
M: 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7 172.20.45.137:6379
   slots:0-1364,5461-6826,10923-12287 (4096 slots) master
   1 additional replica(s)
M: 1289987119bae49782002768a35d90c52d3f2d75 172.20.45.135:6379
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
S: 4efeb88c882d3d07d2fab0bb77f3317963bdd774 172.20.45.137:6380
   slots: (0 slots) slave
   replicates 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7
M: afa67c80fa1dcaec758ae54b0760d466096ac3ff 172.20.45.132:6379
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
M: 325b2cabe287c2bac388eae12598ea661edea46b 172.20.45.133:6379
   slots:12288-16383 (4096 slots) master
   1 additional replica(s)
S: af8aff4dba8f20881218ea6ea2143cd3b5b143cf 172.20.45.134:6379
   slots: (0 slots) slave
   replicates 325b2cabe287c2bac388eae12598ea661edea46b
S: cc7df73e121192185bd31d40859103b7a0c68bdd 172.20.45.131:6379
   slots: (0 slots) slave
   replicates 1289987119bae49782002768a35d90c52d3f2d75
S: 23bc01f789eb7e624dd4288ac548a63f545678cb 172.20.45.136:6379
   slots: (0 slots) slave
   replicates afa67c80fa1dcaec758ae54b0760d466096ac3ff
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096 #分配多少个槽位 给指定的主机
What is the receiving node ID? 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7 #接收 slot 的服务器 ID，手动输入
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:all #将哪些源主机的槽位分配给 192.168.7.104:6379，all 是自动在所有的 redis node 选择划分，如果是从 redis cluster 删除主机可以使用此方式将主机上的槽位全部移动到别的 redis 主机
………………………………..
 Moving slot 6823 from 116c4c6de036fdbac5aaad25eb1a61ea262b64af
 Moving slot 6824 from 116c4c6de036fdbac5aaad25eb1a61ea262b64af
 Moving slot 6825 from 116c4c6de036fdbac5aaad25eb1a61ea262b64af
 Moving slot 6826 from 116c4c6de036fdbac5aaad25eb1a61ea262b64af
 Moving slot 10923 from 70de3821dde4701c647bd6c23b9dd3c5c9f24a62
 Moving slot 10924 from 70de3821dde4701c647bd6c23b9dd3c5c9f24a62
 Moving slot 10925 from 70de3821dde4701c647bd6c23b9dd3c5c9f24a62
 Moving slot 10926 from 70de3821dde4701c647bd6c23b9dd3c5c9f24a62
…………………………………..
 Moving slot 1364 from f4cfc5cf821c0d855016488d6fbfb62c03a14fda
Do you want to proceed with the proposed reshard plan (yes/no)? yes #确认分配

```

> redis 5 :redis-cli -a 123456 --cluster reshard 192.168.7.104:6379 

> 如果分配过程中保存摸个槽位有数据，就去删除，然后执行
> redis-trib.rb fix 172.20.45.131:6379
> 再根据已分配的槽位将剩下未分完的继续划分

5. 验证重新分配槽位之后的集群状态
```bash
[ root@localhost ~]# redis-trib.rb info 172.20.45.131:6379
172.20.45.132:6379 (afa67c80...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.137:6379 (661de21f...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.135:6379 (12899871...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.133:6379 (325b2cab...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```

### 集群维护之动态删除节点
	添加节点的时候是先添加 node 节点到集群，然后分配槽位，删除节点的操作与添加节点的操作正好相反，是先将被删除的 Redis node 上的槽位迁移到集群中的其他 Redis node 节点上，然后再将其删除。如果一个 Redis node 节点上的槽位没有被完全迁移，删除该 node 的时候会提示有数据且无法删除。

1. 迁移master的槽位到其他的master
```bahs
[ root@localhost ~]# redis-trib.rb reshard 172.20.45.131:6379
>>> Performing Cluster Check (using node 172.20.45.131:6379)
S: cc7df73e121192185bd31d40859103b7a0c68bdd 172.20.45.131:6379
   slots: (0 slots) slave
   replicates 1289987119bae49782002768a35d90c52d3f2d75
S: 23bc01f789eb7e624dd4288ac548a63f545678cb 172.20.45.136:6379
   slots: (0 slots) slave
   replicates afa67c80fa1dcaec758ae54b0760d466096ac3ff
S: af8aff4dba8f20881218ea6ea2143cd3b5b143cf 172.20.45.134:6379
   slots: (0 slots) slave
   replicates 325b2cabe287c2bac388eae12598ea661edea46b
M: afa67c80fa1dcaec758ae54b0760d466096ac3ff 172.20.45.132:6379
   slots:6827-10922 (4096 slots) master
   1 additional replica(s)
S: 661de21fa7e4f0f1ca9c23d615b7b5d875c1d5e7 172.20.45.137:6379
   slots: (0 slots) slave
   replicates 4efeb88c882d3d07d2fab0bb77f3317963bdd774
M: 1289987119bae49782002768a35d90c52d3f2d75 172.20.45.135:6379
   slots:1365-5460 (4096 slots) master
   1 additional replica(s)
M: 325b2cabe287c2bac388eae12598ea661edea46b 172.20.45.133:6379
   slots:12288-16383 (4096 slots) master
   1 additional replica(s)
M: 4efeb88c882d3d07d2fab0bb77f3317963bdd774 172.20.45.137:6380
   slots:0-1364,5461-6826,10923-12287 (4096 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096 #要移走多少个槽位
What is the receiving node ID? 1289987119bae49782002768a35d90c52d3f2d75 #接收槽位的id
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:4efeb88c882d3d07d2fab0bb77f3317963bdd774   #要从那个master移走
Source node #2:done #结束
--------------------------------------------------------------------------
 	Moving slot 12277 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12278 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12279 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12280 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12281 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12282 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12283 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12284 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12285 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12286 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
    Moving slot 12287 from 4efeb88c882d3d07d2fab0bb77f3317963bdd774
--------------------------------------------------------------------------
Do you want to proceed with the proposed reshard plan (yes/no)? yes 确认
```

> redis 5 :redis-cli -a 123456 --cluster reshard 192.168.7.104:6379 

2. 验证槽位迁移完成
```bahs
[ root@localhost ~]# redis-trib.rb info 172.20.45.131:6379
172.20.45.132:6379 (afa67c80...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.135:6379 (12899871...) -> 0 keys | 8192 slots | 2 slaves.
172.20.45.133:6379 (325b2cab...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.137:6380 (4efeb88c...) -> 0 keys | 0 slots | 0 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
```

3. 从集群删除服务器
	虽然槽位已经迁移完成，但是服务器 IP 信息还在集群当中，因此还需要将 IP 信息从集群删除
```bash
[root@localhost ~]# redis-trib.rb del-node 172.20.45.132:6379 4efeb88c882d3d07d2fab0bb77f3317963bdd774
>>> Removing node 4efeb88c882d3d07d2fab0bb77f3317963bdd774 from cluster 172.20.45.132:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
[ root@localhost ~]# redis-trib.rb info 172.20.45.131:6379
172.20.45.132:6379 (afa67c80...) -> 0 keys | 4096 slots | 1 slaves.
172.20.45.135:6379 (12899871...) -> 0 keys | 8192 slots | 2 slaves.
172.20.45.133:6379 (325b2cab...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.

```

> redis 5 :redis-cli -a 123456 --cluster del-node 192.168.7.101:6379f4cfc5cf821c0d855016488d6fbfb62c03a14fda


## 集群维护之导入现有 Redis 数据
	导入数据需要 redis cluster 不能与被导入的数据有重复的 key 名称，否则导入不成功或中断

1. 导入数据之前需要关闭各 redis 服务器的密码，包括集群中的各 node 和源 Redis server，避免认证带来的环境不一致从而无法导入，可以加参数--cluster-replace 强制替换 Redis cluster 已有的 key。
```bahs
[root@redis-s1 ~]# redis-cli -h 192.168.7.102 -p 6379 -a 123456
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
192.168.7.102:6379> CONFIG SET requirepass ""
OK
```

2. 执行数据导入
	将源 Redis server 的数据直接导入之 redis cluster。

```bahs
Redis 3/4：
[root@s1 ~]# redis-trib.rb import --from 172.18.200.107:6382 --replace 172.18.200.107:6379
Redis 5：
#redis-cli --cluster import 集群服务器 IP:PORT --cluster-from 外部 Redis node-IP:PORT --cluster-copy --
cluster-replace
[root@redis-s2 redis]# redis-cli --cluster import 192.168.7.103:6379 --cluster-from 192.168.7.101:6379
--cluster-copy
```